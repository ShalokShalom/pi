This is an attempted port of `pi` to F#, using Fable's TypeScript backend as a **round-trip verification target** rather than a runtime replacement. 

## The core strategy

Fable has an official TypeScript emit backend (`fable-ts`) that compiles F# → TypeScript instead of JavaScript. The idea is:

1. Write F# that mirrors Pi's logic
2. Compile F# → TypeScript via Fable's TS backend
3. Run Pi's existing Vitest test suite against the emitted TS
4. Diff the emitted TS against the original Pi TS to verify semantic equivalence
5. Iterate until tests pass, then the port is verified semantically.
6. Now go over the code and check for readability, conciseness, and speed (in that order).

This gives you a **falsifiable correctness criterion** and a **much** smaller codebase, that is also significantly [simpler](https://www.youtube.com/watch?v=SxdOUGdseq4) to understand as a human. 

***

## Fable TypeScript Backend Setup

Fable supports TypeScript output via the `--lang typescript` flag. The project setup looks like:

```fsharp
// YourProject.fsproj
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <Compile Include="src/Core/AgentSession.fs" />
    <Compile Include="src/Core/SessionManager.fs" />
    <!-- etc. -->
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="Fable.Core" Version="4.*" />
  </ItemGroup>
</Project>
```

```json
// .fablerc or fable args in package.json
{
  "scripts": {
    "build:fable": "dotnet fable --lang typescript --outDir fable-out/",
    "test:roundtrip": "vitest run --root fable-out/"
  }
}
```

The emitted `.ts` files land in `fable-out/` and are directly testable by Vitest — no extra transform needed.

***

## Handling Bun APIs

Since the goal is TypeScript output (not a different runtime), Bun APIs stay exactly as they are. The trick is telling Fable to **emit them as passthrough FFI** rather than trying to translate them. You do this with Fable's `import` attribute:

```fsharp
// src/Bun/BunBindings.fs
module BunBindings

open Fable.Core
open Fable.Core.JsInterop

// Passthrough — Fable emits: import { spawn } from "bun"
[<Import("spawn", from="bun")>]
let spawn: obj = jsNative

// Typed version for the subprocess result
type SpawnOptions =
    abstract cmd: string[] with get, set
    abstract stdout: string with get, set
    abstract stderr: string with get, set

[<Import("spawn", from="bun")>]
let spawnTyped: SpawnOptions -> obj = jsNative
```

Fable will emit exactly `import { spawn } from "bun"` in the output TypeScript, which is what the original Pi code uses. The Bun runtime is entirely unaffected — you're just describing the boundary to Fable's type system.

For `Bun.file`, `Bun.write`, `Bun.$` (shell), etc., the same pattern applies:

```fsharp
// Bun.file() → emits: import { file } from "bun"
[<Import("file", from="bun")>]
let bunFile: string -> obj = jsNative

// For Bun's global namespace APIs accessed as Bun.xxx
[<Global("Bun")>]
let Bun: obj = jsNative
```

***

## Structural port strategy

Given Pi's module layout, here's how to map the F# file structure to produce clean round-trip TypeScript:

```
packages/coding-agent/src/core/
  agent-session.ts          → F#: Core/AgentSession.fs
  agent-session-runtime.ts  → F#: Core/AgentSessionRuntime.fs
  session-manager.ts        → F#: Core/SessionManager.fs
  event-bus.ts              → F#: Core/EventBus.fs
  bash-executor.ts          → F#: Core/BashExecutor.fs  (thin Bun FFI)
  exec.ts                   → F#: Core/Exec.fs          (thin Bun FFI)
  tools/                    → F#: Tools/*.fs
```

The session lifecycle types from `agent-session-runtime.ts` translate almost mechanically to F# DUs and records:

```fsharp
// F# — Core/AgentSessionRuntime.fs
module AgentSessionRuntime

open Fable.Core

// The TypeScript union: "new" | "resume" | "fork" | "quit"
// becomes a proper DU — Fable emits it as a string literal union in TS
type ShutdownReason =
    | [<CompiledName("new")>]   New
    | [<CompiledName("resume")>] Resume
    | [<CompiledName("fork")>]  Fork
    | [<CompiledName("quit")>]  Quit

// The interface maps directly to an F# record
type CreateAgentSessionRuntimeResult =
    { session:              AgentSession
      services:             AgentSessionServices
      diagnostics:          AgentSessionRuntimeDiagnostic[]
      modelFallbackMessage: string option }
```

The `[<CompiledName>]` attribute is critical — it ensures Fable emits the exact string values the existing TypeScript code and tests expect, so the round-trip test can compare output.

***

## Mutable class state

`AgentSessionRuntime` has private mutable fields that get swapped during session transitions. In F# you have two clean options:

**Option A — `ref` cells** (close to the TS original, easier to verify):
```fsharp
type AgentSessionRuntime
    (initialSession:   AgentSession,
     initialServices:  AgentSessionServices,
     createRuntime:    CreateAgentSessionRuntimeFactory) =

    let mutableSession  = ref initialSession
    let mutableServices = ref initialServices

    member _.Session  = mutableSession.Value
    member _.Services = mutableServices.Value

    member _.Apply(result: CreateAgentSessionRuntimeResult) =
        mutableSession.Value  <- result.session
        mutableServices.Value <- result.services
```

Fable emits this as a class with `#session` and `#services` private fields — essentially identical to the original TS.

**Option B — `MailboxProcessor`** (more idiomatic F#, harder to verify round-trip):

Stick with Option A for the round-trip phase. Once the tests pass and correctness is confirmed, you can refactor to Option B for the "real" F# port if you want the actor model.

***

## Test wiring

Pi already has a Vitest config in `packages/coding-agent/vitest.config.ts`. The round-trip test workflow:

```
fsharp source → fable --lang typescript → fable-out/*.ts
                                              ↓
                              vitest run (pointing at fable-out/)
                                              ↓
                           diff fable-out/ vs original src/core/
```

The diff is a **structural** check, not a character-by-character one. You care that:
- All exported symbols exist with the same names
- All type shapes are compatible
- Behavior under tests is identical

You can wire this into a single `npm` script:

```json
"scripts": {
  "port:verify": "dotnet fable --lang typescript --outDir fable-out && vitest run --root fable-out && echo '✓ Round-trip passed'"
}
```

***

## Practical iteration order

Start from the **leaves** (no dependencies) and work inward:

1. `event-bus.ts` — trivial, just a typed pub/sub; perfect first target
2. `defaults.ts`, `timings.ts`, `telemetry.ts` — tiny files, fast wins
3. `bash-executor.ts` / `exec.ts` — these are the Bun API boundary; get the FFI bindings right here early
4. `session-manager.ts` — pure data manipulation, testable in isolation
5. `agent-session-runtime.ts` — the lifecycle orchestrator; once session-manager works, this follows
6. `agent-session.ts` (103 KB) — save this for last; it's the streaming LLM loop and the hardest

The main practical risk is Fable's TypeScript emit sometimes generating slightly different identifier casing or import styles than Pi's original. Using `[<CompiledName>]` attributes consistently on public API members prevents most of those mismatches. For module-level `export` compatibility, Fable's `[<ExportDefault>]` and `[<Export>]` attributes give you fine control over what the emitted `.ts` exports.

# Detailed description of the cyclic workflow for AI systems

**AI task:**
- Scaffold `pi-core.fsproj`, Fable TS backend build, `justfile`
- Port one small, complete, real module (e.g. `defaults.ts`) end-to-end
- Manually compare Fable output against original TypeScript
- Document every structural difference (identifier casing, module shape, class vs record emission)

**Gate:** If the Fable emit diverges significantly from Pi's code style, adjust `[<CompiledName>]` and `[<ExportDefault>]` attributes until the emitted TS is structurally compatible. Do not proceed until this is resolved. This spike protects every subsequent cycle.

***

### Cycle 1 — Pattern Catalogue and Protocol Schema

Two independent AI tasks that must both be completed before Cycle 2:

**Task A — Comby pattern catalogue (TypeScript-side analysis only):**

The AI uses Comby¹ against the TypeScript source to produce a structured catalogue of every recurring idiom that requires non-trivial F# treatment. Comby runs only against TypeScript here — brace-delimited, where it works correctly. Output is a human-readable document, not executable rules:

```
Pattern: private mutable class field
Comby: private :[field]: :[type];
Occurrences: 47 across 12 files
F# treatment: ref cell with [<CompiledName>]
```

**Task B — HTTP protocol schema:**

Define the complete JSON contract between the F# core and the future Kotlin UI *before* any logic is ported. This is a design document, not code. Produces `protocol.md` with all endpoint shapes, SSE event types, and session state representations. Locking this down now prevents rework in later cycles.

**Your review gate:** Read the Comby catalogue and the protocol schema. These are the two highest-leverage review artifacts in the entire project.

***

### Cycle 2 — Type System

**AI task:** Port all `interface` and `type` definitions to F# records and DUs — no function bodies.

The AI works from the Comby catalogue (which identifies all DU-candidate types) plus Serena's type graph. Every public type gets `[<CompiledName>]` annotations. The result is `Types.fs` — a single module that all later modules import.

**Verify:** `tsc --noEmit` on Fable-emitted `.d.ts` against `pi-ai`'s existing test fixtures (which have good type coverage).

**Additional verification:** Property-based tests using `fast-check` written *now* for the type constructors, before any logic is ported. These become the correctness oracle for later cycles since unit tests are sparse.

***

### Cycle 3 — `pi-ai` Package

**AI task:** Port `packages/ai` — the LLM abstraction layer.

This is the best-tested part of the codebase. The AI ports it semantically — reading the author's design rationale for provider quirks (Cerebras doesn't like `store`, Mistral uses `max_tokens`, Google doesn't support tool streaming) and expressing each as explicit F# pattern match cases with comments referencing the original intent. [mariozechner](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)

**Verify:** Run `pi-ai`'s existing extensive test suite against the Fable-emitted TypeScript. This is the one cycle where the round-trip Vitest strategy works reliably as the primary gate.

***

### Cycle 4 — Bun FFI Boundary

**AI task:** Complete `BashExecutor.fs` and `Exec.fs`.

Isolated deliberately. The AI writes typed Fable `[<Import>]` bindings for every Bun API surface used across the codebase — not just what cycle 4 needs, but all of them — so subsequent cycles never need to revisit the FFI layer.

**Verify:** You run manually. Spawn `echo hello`, assert stdout. Spawn a failing command, assert exit code, and stderr.

***

### Cycle 5 — Session Data Layer

**AI task:** `Messages.fs`, `SessionCwd.fs`, `AgentSessionServices.fs`, `SessionManager.fs`.

The AI reads `session-manager.ts` (43 KB) holistically before writing any F#. It first proposes a translation strategy — specifically, how it will handle each mutable field identified in the Comby catalogue — which you approve or adjust. Then it writes the F#.

**Verify:** Write property-based tests (fast-check) for session creation, JSONL round-trip, and branch/fork operations. These are the tests that didn't exist before — they become the permanent regression suite.

***

### Cycle 6 — Configuration and Resolution

**AI task:** `Config.fs`, `ResolveConfigValue.fs`, `ModelRegistry.fs`, `ModelResolver.fs`, `SettingsManager.fs`.

Configuration files are dense with conditional logic. The AI's advantage over a transpiler is that it can read `models.generated.ts` — the parsed OpenRouter/models.dev data — and understand *why* the registry is structured the way it is, producing F# that preserves the intent rather than just the syntax.

**Verify:** Property-based tests for config merging and model resolution.

***

### Cycle 7 — Extension System

**AI task:** `Extensions/Runner.fs`, `Extensions/Index.fs`, `Skills.fs`. Then `PackageManager.fs` split into sub-modules.

Before writing any F#, the AI uses Serena to map the full dependency graph of `package-manager.ts` and proposes a sub-module decomposition. You approve the decomposition before translation begins. The AI then ports each sub-module in dependency order.

**Docfork** (used correctly now — for library docs retrieval): feed it the Bun npm API documentation so the AI has accurate context for the package installation bindings.

***

### Cycle 8 — Agent Session Runtime

**AI task:** `AgentSessionRuntime.fs`, `Sdk.fs`, `SystemPrompt.fs`, `PromptTemplates.fs`.

The orchestration layer. The AI reads the author's philosophy ("no plan mode", "no sub-agents", "minimal system prompt") as design context alongside the code. This is where the semantic understanding of intent matters most — the F# should express *why* the session lifecycle is structured the way it is, not just *what* it does.

**Verify:** Property-based tests for session switch, fork, new session, and import-from-JSONL.

***

### Cycle 9 — Agent Session Core

**AI task:** `agent-session.ts` (103 KB), split into sub-modules.

Proposed decomposition (AI confirms or revises using Serena):
- `AgentLoop.fs` — main LLM request/response iteration
- `ToolDispatch.fs` — routing tool calls
- `Compaction.fs` — context window management  
- `StreamingResponse.fs` — SSE token delivery

At 103 KB, this may require two AI sessions. The natural split point is `StreamingResponse.fs` — self-contained enough to port independently. Delta diff between original TS and Fable-emitted TS is reviewed manually for behavioral differences that property tests cannot catch.

***

### Cycle 10 — HTTP Boundary

**AI task:** Complete `HttpServer.fs` implementing the protocol schema defined in Cycle 1.

Because the schema was locked in Cycle 1, this cycle is mechanical. The AI implements the endpoints, the SSE streaming path, and an integration test suite that starts the server and exercises every endpoint.

**Final gate:** `curl localhost:3141/sessions` returns valid JSON. SSE stream delivers tokens in the correct order. The protocol schema document and implementation are diffed to confirm no drift.

***

## Improvements Over Previous Plan

| Issue | Fix |
|---|---|
| No test coverage on `coding-agent` | Property-based tests written *alongside* each cycle rather than relying on pre-existing Vitest suite |
| Transpiler as primary driver | AI semantic porting as primary driver; Comby as pre-analysis cataloguing tool only |
| Docfork mischaracterized | Used correctly for library docs retrieval (Bun APIs, Fable docs) into AI context |
| Protocol defined too late | Locked in Cycle 1 as a design artifact before any logic is ported |
| Fable TS fidelity unvalidated | Cycle 0 spike required as hard gate before any porting begins |
| Transpiler rule set as one-time artifact | Replaced by the Comby pattern catalogue as a living reference document |

---
¹
TODO: Comby is anticipated to be the most readable, idiomatic choice. We will actually run tests, to see if an alternative, such as `ast-grep` may be working better. Also a test run without such tool would be appropriate. The consideration between these tools is considered to be an open requirement for the actual implementation of the entire project. 

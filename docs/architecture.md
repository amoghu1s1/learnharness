# Learning Harness — Architecture

Status: **the canonical architecture spec** — describes the as-built system, verified against the code by a full audit on 2026-09-18 (every file under `src/`, `app/`, `components/`, `electron/`, `scripts/`, the lint config, and the test suite; `lint` and `typecheck` clean on the audited tree). The pre-audit spec is archived at [`archive/architecture-2026-09-08.md`](archive/architecture-2026-09-08.md) — history only. Product reasoning lives in [`docs/vision.md`](vision.md); file-by-file layout lives in the README.

**What this is.** A standalone learning app (web + Electron desktop): the learner's knowledge accumulates into one persistent, cross-session knowledge graph, built from tutoring sessions in which a teacher agent authors a formal, visible dependency tree (the plan tree) and verifies understanding with structured questions. The graph is the home page; the session is the classroom; everything else serves those two.

**Design principles, in priority order:**

1. **Maximum control.** Every dependency edge is explicit and machine-checked. Nothing reaches sideways for convenience.
2. **Easy editability.** A worker can trace any behavior to one file. Layers are named by responsibility, not by framework accident.
3. **Zero technical debt.** The shape you see is the shape that runs. Deprecated paths are deleted, not parked.
4. **Crystal-clear architecture.** Two barrels as the entire public API, two composition roots, one writer per store, named seams. When a rule can be a lint rule, it is.

**The one-sentence model:** a pure domain core at the center; every external effect (fs, LLMs, browser, OS) behind an adapter at the edge; two composition roots as the only places the edges are plugged together; and every persistent store written by exactly one system.

---

## 1. Two axes, one law

Hold the architecture along two orthogonal axes:

**Static structure — the hexagon.** One pure core (`src/core`); everything else is an *adapter* (implements a core port or drives the core) or a *composition root* (the only code that wires adapters to core). Dependency arrows point **inward only** — toward purity, toward stability. Two barrels are the entire public API: `@/src/core` (browser-safe) and `@/src/server` (the server-side surface).

**Runtime ownership — five systems.** Each persistent store has exactly one owning writer-system; systems talk only through named, idempotent protocols:

| System | Owns (sole writer) | Role |
|---|---|---|
| **Classroom** — session engine | `data/chats/*.json` | the turn pipeline, prober→teacher, plan tree, questions, phases; practice sessions are a `kind: "practice"` variant |
| **Workbench** — background labor | `data/jobs/*.json` | job runner, researcher/demo-builder/visual-maker, steer delivery |
| **Library** — world model | `data/graph/*` ledgers | extraction pipeline (sole graph writer), derivation, birth-session clusters |
| **Stage** — presentation shell | nothing server-side | components/, graph canvas, docks — pure consumer via `/api/*` |
| **Machine room** — host | process + config | Electron main, standalone Next server, `config.ts`, data-dir seam, model auth (BYOK keys / desktop subscription connections) |

**The unifying law** (generates everything else): **imports point inward, writes point to append-only evidence, and everything else is derived.** Dependency flows toward purity; data flows toward evidence; no state is stored where it can be recomputed.

---

## 2. The layers and the import law

Dependency arrows point **down only**. Each layer may import the core barrel; nothing skips a boundary for convenience.

```
┌──────────────────────────────────────────────────────────────┐
│ SHELL      app/            Next.js route handlers (transport) │
│            components/     session + home UI (render only)    │
├──────────────────────────────────────────────────────────────┤
│ SERVER     src/server      composition roots (stores.ts,      │
│            chat-handler.ts) + named service seams + the      │
│            pi engine bridge. Zero business logic              │
├──────────────────────────────────────────────────────────────┤
│ AGENT      src/agent       prober + teacher loops, tool       │
│            surfaces, prompt fragments, sub-agents, the       │
│            extraction pipeline. Deps arrive via factories —  │
│            never imported from below                          │
├──────────────────────────────────────────────────────────────┤
│ CORE       src/core        pure domain: contracts (6 domains),│
│            ports, merger + derivation, coverage, question-   │
│            core, plan-tree, phases, layout, prices. No IO     │
│            (two files use node:crypto — pure                  │
│            but node-bound, hence kept out of the barrel)      │
├──────────────────────────────────────────────────────────────┤
│ STORE      src/store       fs adapters: chats · graph        │
│            ledgers · jobs (implement core ports) + two       │
│            portless settings adapters. The ONLY modules     │
│            that touch data/                                   │
└──────────────────────────────────────────────────────────────┘
```

**The import law** (`eslint.config.mjs` — the config's sole job; four `import/no-restricted-paths` zones, all `"error"`; each verified to fire; `lint` is a gate on every change):

| Zone | May import | Rationale |
|---|---|---|
| `components/**` | `@/src/core` only (the barrel) | The UI renders domain data; it never runs domain logic or touches IO |
| `app/**` | `@/src/core` + `@/src/server` (the barrels) | Route handlers are transport; they delegate to server entry points |
| `src/core/**` | nothing below itself (never `agent`/`server`/`store`) | The core is pure; adapters implement its ports, never the reverse |
| `src/agent/**` | never `server`/`store` | The agent layer receives its dependencies (ports, engines, seams) via factories/closures |

```mermaid
flowchart TB
    subgraph SHELL["SHELL — render + transport, no domain logic"]
        APP["app/ — pages + thin API route handlers"]
        COMP["components/ — session + home UI, graph canvas"]
    end

    subgraph SERVER["SERVER — src/server — composition + seams"]
        SBAR["barrel index.ts — THE server surface"]
        ROOTS["composition roots: stores.ts (storage) · chat-handler.ts (turn)"]
        SEAMS["service seams: turn-service · jobs (wake) · turn-liveness · commit-positions · observability · model-catalog · settings-body"]
        PIB["pi/ — engine bridge: record-adapter (record → pi messages) · ui-stream (pi events → SSE)"]
    end

    subgraph AGENT["AGENT — src/agent — LLM loops, deps injected"]
        TEACH["prober + teacher agents · tools · prompt fragments"]
        SUBS["sub-agents: researcher · demo-builder · visual-maker"]
        PIPE["extraction pipeline"]
    end

    subgraph CORE["CORE — src/core — pure domain, zero IO"]
        CONTRACTS["contracts/ — 6 domains: plan · graph · chat · jobs · probe · models"]
        PORTS["ports.ts — ChatStore · GraphStore · JobStore (types only)"]
        BRAIN["merger + derivation · coverage · question-core · plan-tree · phases · layout · prices · tool-parts"]
        CBAR["barrel index.ts — browser-safe exports ONLY"]
    end

    subgraph STORE["STORE — src/store — fs adapters, only data/ writers"]
        ADP["chats.ts · graph.ts · jobs.ts (implement core ports)"]
        PORTLESS["render-settings.ts (portless)"]
    end

    CFG["config.ts — leaf facts: PATHS · LEDGER_FILES · engine-construction facts (no models — model slots own those)"]

    COMP -->|"lint zone: @/src/core only"| CBAR
    APP -->|"lint zone: @/src/core + @/src/server"| CBAR
    APP -->|"lint zone: @/src/core + @/src/server"| SBAR
    SBAR --> ROOTS
    SBAR --> SEAMS
    ROOTS -->|"instantiates + passes ports"| ADP
    ROOTS -->|"factory calls, ports injected"| TEACH
    ROOTS -->|"factory calls, ports injected"| PIPE
    SEAMS --> PIB
    TEACH --> CONTRACTS
    TEACH --> BRAIN
    SUBS --> CONTRACTS
    PIPE --> BRAIN
    ADP -->|"implement"| PORTS
    ADP --> CFG
    CBAR --> CONTRACTS
    CBAR --> PORTS
    CBAR --> BRAIN
    SERVER --> CFG
    AGENT --> CFG

    CORE -.->|"✗ FORBIDDEN (lint)"| SERVER
    AGENT -.->|"✗ FORBIDDEN (lint)"| SERVER
    COMP -.->|"✗ FORBIDDEN (lint)"| SERVER
```

**Two rules are documentation-only by design.** The ESLint plugin's `except` can't express "all but one", so (a) `src/server/stores.ts` is *intended* to be the only server module importing adapters — but the server barrel also re-exports the two portless settings adapters and `observability.ts` imports `chatStore` directly; and (b) the browser-safety of the core barrel is a doc-comment convention (see §2.1). `tests/`, `scripts/`, and `electron/` are deliberately outside the law (lint-excluded); tests import freely across layers, scripts act as extra composition roots.

### 2.1 The core barrel and browser-safety

`src/core/index.ts` exports only browser-safe modules — pure logic + types, no node builtins, no fs, no network. Deliberately **not** barrel-exported (server code imports them directly): `graph.ts` and `coverage.ts` (need `node:crypto`), `clusters.ts`, `session-clusters.ts` and `layout-settle.ts` (server-only roles). Port *types* are types-only, hence safe to re-export. This split is what lets the same core serve browser (grading, layout, plan statuses, cost folding) and server (merger, derivation, coverage).

### 2.2 Contracts — one file per domain

`src/core/contracts.ts` is a barrel over six domain files, each owned outright by its stream (schema changes in a domain you don't own go through the master session):

- `contracts/plan.ts` — plan tree types + derived statuses (`derivePlanStatuses`)
- `contracts/graph.ts` — graph domain: ledgers, findings/audit shapes, snapshot, birth-session clusters
- `contracts/chat.ts` — session records, usage/money channel, settings, handoff text
- `contracts/jobs.ts` — the ack-and-steer protocol: job records, steer message builders, deterministic steer ids, usage accumulation
- `contracts/probe.ts` — probe assessment + the shared zod schema
- `contracts/models.ts` — client-safe model-catalog shapes

---

## 3. Ports, adapters, and the composition roots

Storage is **per-entity semantic ports**, never one generic interface: `ChatStore`, `GraphStore`, `JobStore` (`src/core/ports.ts` — types only). The fs adapters (`src/store/{chats,graph,jobs}.ts`) implement them and contain persistence and nothing else. One more adapter (`src/store/render-settings.ts`) is a portless free function re-exported through the server barrel — a deliberate pragmatic bypass, documented here so nobody mistakes it for the pattern.

**Idioms.** Graph evidence lives in **append-only JSONL ledgers** (corrupt-line-tolerant reads; strict shape guards drop invalid lines, never guess). Chats, jobs, positions, sources, and the settings blobs are whole-file read-modify-write JSON. All writes are synchronous, single-process (atomicity rides the Node event loop). Merge/dedup/derivation semantics are **reader-side, in core** — the store never decides.

- **`src/server/stores.ts` is THE composition root for storage**: it instantiates the adapters and hands the port objects to everything else (chat-handler, job runner, extraction pipeline, graph/positions/chats routes, turn services). It is the single place a storage decision (JSON vs SQLite, local vs remote) gets re-made.
- **`src/server/chat-handler.ts` is THE composition root for a turn**: it wires the stores, the job runner, the extraction pipeline, and the pi engine into the stream response (`setTurnRunner`/`setJobRespawn` register its seams for the wake pipeline). All logic lives behind the seams it calls.
- The agent layer is dependency-inverted throughout: `createJobRunner({ chatStore, jobStore, deliver })`, `createExtractionPipeline({ chatStore, graphStore, extractor?, auditor?, embedder })`, `createTeacherTools({ chatId, chatStore, jobStore, spawn })`, `createProberAgent`/`createTeacherAgent(engine?, tools?, …)`. Tests inject the same ports production wires — or swap an in-memory port.

The engine side is adapted too: `src/agent/models.ts` (`piEngineForSpec`) builds each role's engine from the resolved **model slots** (`settings.json` `models` section — strong/cheap/vision; each slot names a provider + model, with credentials from a BYOK key or a desktop subscription connection; a chain that ends empty refuses honestly with 503 `models_not_configured`) — and the teacher/prober conversation runs the **SESSION'S OWN record**: `POST /api/chats` stamps every newborn verbatim from the strong slot (the new-session default, chosen in setup), in-session picks write the session's record (`PUT /api/chats/:id/settings`), and the strong slot never reaches an existing session again (legacy unstamped records fall back to it at resolve time) — per-session models, owner 2026-09-28. Beyond the conversation, `src/server/pi/` bridges the two SDK worlds: `record-adapter.ts` (persisted UIMessage record → stateless pi `Message[]`, dangling tool calls get synthesized INTERRUPTED results) and `ui-stream.ts` (pi agent events → AI-SDK UIMessage SSE).

```mermaid
flowchart LR
    subgraph DRIVING["DRIVING ADAPTERS — inbound"]
        BROWSER["Browser UI — components/: question dock (addToolOutput) · graph canvas · docks"]
        HTTP["HTTP transport — app/api/*: thin route handlers"]
        NOTES["Notes front door — POST /api/graph/import"]
        ELEC["Electron main — launcher + subscription credential bridge · spawns server · LH_DATA_DIR + BYOK env"]
    end

    subgraph HEXCORE["THE CORE — src/core (pure domain)"]
        direction TB
        SESS["session brain: phases · plan-tree · question-core · pending · steer builders"]
        GBRAIN["graph brain: merger + derivation · coverage · clusters · layout"]
        PORTSBOX["PORTS (types only): ChatStore · GraphStore · JobStore"]
    end

    subgraph DRIVEN["DRIVEN ADAPTERS — outbound"]
        FS["fs stores — src/store (data/)"]
        LLM["LLM engines — pi-ai providers + subscription connections via piEngineForSpec"]
        HEAD["headless Chrome — demo verify · SVG render"]
        DEEP["Deepgram STT — token-minting route, browser-side WebSocket"]
        WEB["web tools — Google CSE + Readability/Jina (researcher)"]
    end

    BROWSER --> HTTP
    NOTES --> HTTP
    ELEC --> HTTP
    HTTP --> SESS
    HTTP --> GBRAIN
    SESS --> PORTSBOX
    GBRAIN --> PORTSBOX
    PORTSBOX --> FS
    SESS --> LLM
    GBRAIN --> LLM
    SESS --> HEAD
    SESS --> WEB
    BROWSER --> DEEP
```

---

## 4. The five systems (runtime ownership and protocols)

The systems view: each box owns its store; edges are the protocols, each with the adapter/seam that implements it.

```mermaid
flowchart TB
    LEARNER["the learner"]

    subgraph STAGE["STAGE — presentation shell (owns nothing server-side)"]
        UI["session UI · question dock · plan panel · graph canvas · activity dock · sidebar"]
    end

    subgraph CLASSROOM["CLASSROOM — session engine (owns data/chats)"]
        TURN["turn pipeline — handleChatRequest"]
        AGENTS["prober → teacher · practice agent (stateless per-turn, from the record)"]
        CHATDATA["chat records — data/chats/*.json"]
    end

    subgraph WORKBENCH["WORKBENCH — background labor (owns data/jobs)"]
        JOBRUN["job runner"]
        SUBAGENTS["researcher · demo-builder · visual-maker"]
        WAKE["wake pipeline — per-chat serialized queues on globalThis"]
    end

    subgraph LIBRARY["LIBRARY — world model (owns data/graph ledgers)"]
        EXTRACT["extraction pipeline — extractor → auditor → merger (SOLE graph writer, serial per source)"]
        DERIVE["derivation — graphSnapshot · birth-session clusters (reads, never write)"]
        GRAPHDATA["append-only ledgers — data/graph/*.jsonl"]
    end

    subgraph ROOM["MACHINE ROOM — host"]
        HOST["Electron main → standalone Next server · config.ts · LH_DATA_DIR · model auth (BYOK / subscriptions)"]
    end

    LEARNER -->|"text · answers [transport adapter]"| UI
    UI -->|"POST /api/chat [question answers ride addToolOutput]"| TURN
    TURN -->|"quiz / ask_user_question stream to browser [client-question tools, no execute]"| UI
    TURN -->|"saveMessages · persistTurn [ChatStore port]"| CHATDATA
    TURN -->|"spawn(job) — fire-and-forget ack [job tools + JobStore port]"| JOBRUN
    JOBRUN --> SUBAGENTS
    SUBAGENTS -->|"result → steer message [steer builders in core/jobs.ts]"| WAKE
    WAKE -->|"exactly-once wake turn [turnRunner seam → same handleChatRequest]"| TURN
    TURN -->|"triggerExtraction when uncoveredTokens ≥ chunkTokens [trigger seam, single-flight]"| EXTRACT
    EXTRACT -->|"reads transcript as evidence [ChatStore port]"| CHATDATA
    EXTRACT -->|"append-only ledger writes [GraphStore port — sole writer]"| GRAPHDATA
    DERIVE -->|"pure read: snapshot for home [graphSnapshot]"| GRAPHDATA
    UI -->|"GET /api/graph (read) · PUT positions (sanctioned exception) · POST import"| DERIVE
    HOST -.->|"hosts every server-side system"| TURN
```

Three server-side systems, each the sole writer of its own store, connected by named protocols: the **turn request** (Stage→Classroom), the **ack-and-steer** loop (Classroom→Workbench→Classroom), and the **extraction trigger + transcript read** (Classroom→Library). Practice is not a separate system: a `kind: "practice"` session uses the same Classroom turn pipeline with the practice agent. The Stage touches systems only through the thin HTTP transport; the teacher reaches the Library directly through the gated prober tools (`list_sessions` / `list_propositions`) rather than any injected context.

---

## 5. The session (one chat turn)

`handleChatRequest` is framework-free (plain `Request`/`Response`) with a `buildAgent` test seam so tests drive it with scripted engines. The turn, in exact order:

```mermaid
sequenceDiagram
    participant B as Browser (useChat)
    participant R as POST /api/chat
    participant H as handleChatRequest
    participant L as turn-liveness (globalThis)
    participant CS as chatStore adapter
    participant A as prober/teacher agent (pi engine)
    participant F as onFinish seams

    B->>R: messages (+ graded question output via addToolOutput)
    R->>H: handleChatRequest(req)
    H->>L: beginTurn(chatId)
    Note over H,L: already running → 409; retryingFetch waits, re-fetches<br/>the record, merges unsent client messages by id, re-POSTs
    H->>H: claimSteers — coalesce queued steers into outgoing
    H->>CS: saveMessages (write-ahead: persist even before the stream)
    H->>A: build agent from record (probe_done part in transcript ⇒ teacher)
    Note over H,A: recordToPiMessages: stateless rebuild; dangling<br/>tool calls get synthesized INTERRUPTED results
    A-->>B: SSE stream (ui-stream bridge)
    Note over B: quiz / ask_user_question have NO server execute:<br/>rendered in the dock; graded client-side via question-core
    B->>B: QuestionDock: parse → shuffle (seeded) → buildAnswer (grade)
    B->>R: continuation (sendAutomaticallyWhen fires)
    A-->>F: onFinish
    F->>CS: persistTurn (canonical, idempotent) + appendTurnUsage
    F->>F: triggerExtraction — if uncovered tokens ≥ chunkTokens, fire async
    F->>L: endTurn (ALWAYS last — the registry never locks)
```

- **Client-side question tools** (`quiz`, `ask_user_question`) have no server execute: the call streams to the browser, the learner answers in the dock, the client grades via `question-core` and returns the verdict with `addToolOutput`, which fires the continuation. A pending quiz persisted in the record is exactly what makes **resume mid-question free**. Deterministic seeded shuffling (keyed by toolCallId) keeps server and client renders identical.
- **The prober→teacher handoff**: the first turn(s) run the prober (floor/ceiling via quiz, goal via ask); `probe_done` (server-executed) files the assessment and queues a handoff steer; from the first `tool-probe_done` part in the transcript, the handler builds the teacher agent for every later turn. Phases (`core/phases.ts`) fold strictly: probe → plan → teach.
- **The formal plan tree is author-once**: the teacher authors the session's dependency map with a single schema-validated `plan_init` tool (a mermaid-subset graph, parsed deterministically with line-numbered errors, validated as a DAG). Progress is never mutated — statuses (`current`/`visited`/upcoming) are **derived** from tagged quiz calls in the transcript; the tree panel and the plan blocks render that derivation.
- **Job tools are server-executed, ack-and-steer**: `research`/`demo`/`visual` resolve immediately mid-turn with a fire-and-forget ack (the teacher keeps teaching); `spawn(job, input)` fires the background runner.
- Turn services are named seams taking explicit dependencies. Extraction contains its errors, but **canonical persistence must not fail silently**: a failed input write refuses acceptance without starting an agent; `persistTurn` throws on a failed reply write. The runner reports settlement failure before `[DONE]` and releases liveness even when normal settlement fails. Steer reconciliation uses the durable record, so a later bookkeeping failure cannot replay already-saved learner input.
- **Late attachment must still see settlement failure.** A small, memory-only notice per session precedes the settled stream terminal when the failing turn has already left the live registry. The next accepted turn, successful session deletion, or server exit clears it. Reopening seeds the existing banner from that notice and waits for ordinary learner input; dismissing the banner is not permission to automatically retry the model. No synthetic failure entry is added to the saved transcript.
- **Recovery never blindly resends an answer.** Submitted question drafts survive optimistic unmount in memory; the existing Submit action reconciles uncertain acceptance against the canonical record and running-turn state before sending again. EOF or a rejected stream read without a terminal triggers one read-only reattachment; another transport failure shows the existing error banner rather than claiming completion. Explicit completion, cancellation, and the literal-final-entry saved-error rule remain unchanged.

### 5.1 The wake pipeline (Workbench → Classroom)

**Worker completion and session delivery are separate lifecycles.** Dispatch persists the tool input on the job before spawning. On completion, the runner saves its result, terminal worker status (`done`/`error`), and `delivery: {kind: "result", status: "pending"}` in one job-record update. It then queues the native custom **steer message** without awaiting delivery. Its content is model-facing text; details carry the UI artifact. Worker resources are released and the Background agents row disappears even if an unanswered question holds the result indefinitely. Dispatch cards say “finished,” not “delivered.” Only computing workers can be aborted; abort saves an independent pending abort notice.

```mermaid
flowchart TB
    R["save worker outcome + pending delivery; release worker"] --> ENQ["one registry entry per steer ID, including wake cargo"]
    ENQ --> LIVE{"live agent and background consent?"}
    LIVE -->|yes| STEER["pi steering at native boundaries; answer run takes parked results"]
    LIVE -->|no| GATE["wait for turn-free, question-free, unpaused, consented session"]
    GATE --> RUN["idle wake through the normal turn composition"]
    RUN -->|"409"| BACKOFF["existing backoff 500→1000→2000→5000ms"]
    BACKOFF --> GATE
    RUN -->|"403"| HOLD["retain ownership until consent permits delivery"]
    RUN -->|"other failed start"| QUIET["quiet pending; next learner run or restart recovery"]
    RUN --> SAVE["canonical marker persistence, not merely HTTP 200"]
    STEER --> SAVE
    SAVE --> FIN["acknowledge delivery; worker outcome unchanged"]
    CRASH["lazy recovery + live-agent registration restore known pending jobs"] --> ENQ
```

The per-chat registry retains ownership across live handoff, idle-wake flight, and 409/403 return; repeated submissions share one entry/promise. A deterministic marker ID in the **saved conversation** completes delivery. Streaming visibility alone does not. A failed bookkeeping acknowledgement cannot replay a saved marker, and later teaching/model failure does not reopen delivery or retry the accepted turn. Failed starts leave results durably pending without a new error UI or retry timer; the in-memory entry stays parked until a learner run or process restart. Normal question/dismissal waits and consent deferrals are not failures.

Recovery reads saved job input before legacy transcript lookup. Missing research questions fail honestly without invoking the researcher. Legacy running jobs with saved results become finished workers with pending delivery; legacy terminal jobs without delivery metadata remain untouched, rather than guessing that old outcomes should be replayed. Pending result and abort notices are restored before a newly registered agent's first steering poll, even without a preceding jobs-feed read. Deleting a session discards its pending delivery obligations; only live turns and computing workers block deletion. These guarantees use the existing JSON storage and single-process delivery registry, not a transactional queue or a promise of exactly-once model execution.

Sub-agents **verify, never one-shot**: the demo-builder headlessly executes its HTML in system Chrome before delivery; the visual-maker renders each SVG and looks at it (needs a vision-capable model — the `vision` slot in settings).

---

## 6. The knowledge graph — two channels, one writer

**The extraction pipeline is the sole graph writer.** Nothing else appends to `data/graph/*.jsonl` — not the teacher, not the server, not a migration script. (The one sanctioned exception: `PUT /api/graph/positions` writes layout positions only, sanitized, with an incremental layout settle when the graph grew. Backfill scripts bypass this by design, as one-time operations.)

```mermaid
flowchart TB
    TRIG["trigger seam (turn-service): coverageOf(record, coverageLedger)<br/>— uncovered tokens ≥ graph.extractionChunkTokens (settings)?"] -->|"yes, single-flight per source"| RUN["extraction.runExtraction(chatId)"]
    IMPORT["POST /api/graph/import (notes front door)<br/>deterministic src id = sha256(path); content-fp guard skips unchanged re-imports"] -->|"importNotes: shared pipeline, per-door preambles"| RUN
    RUN --> BATCH["batchRuns — greedy token-budgeted batches of oldest uncovered TEACHER-PROSE messages"]
    EXT["EXTRACTOR (LLM) — sees ONLY the teacher-prose view (tree-blind, graph-blind)<br/>submits propositions + REASONING CHAINS (owner redesign, 2026-09-30)<br/>stored AS-IS — chains.jsonl is the truth (chains-primary, 2026-09-30)"]
    EXT --> AUD["AUDITOR (LLM, agentic loop) — sees findings ONLY, never the source;<br/>explores the graph via find / list_sessions / list_propositions tools,<br/>judges identity ONLY (link-or-create, equivalence bias) — sees propositions, never mints chains"]
    AUD --> MERGE["MERGER (deterministic, core/graph.ts) — the ONLY ledger writer:<br/>identity via validated linkTo · chain dedupe (premises+conclusion → one arg_ id) · coverage checkpoint"]
    MERGE --> SETTLE["settleGraphLayout — resting positions ledger"]
    MERGE -->|"failure at ANY stage"| NOTHING["checkpoints nothing — the next trigger re-runs"]
```

**Raw reasoning chains only** (2026-09-27 cleanup; teacher-prose view 2026-09-28): the extractor reads ONLY the session — and since the 2026-09-28 ruling, ONLY the teacher agent's prose: plain paragraphs under the session title, no learner messages, no thinking, no tool calls or results, no quiz blocks. Coverage fingerprints, slot tokens, and batch budgets all derive from that same selection (`messageViewLines` in `core/coverage.ts`), so "covered" is exactly "what the extractor saw" — a quiz answer or learner edit can never open a run, and only a teacher-prose edit re-opens its message. The old call+result inseparability bond is deleted (nothing left to keep together); batching is pure greedy fill. The shared markdown engine (`core/session-markdown.ts`) keeps the full session format for the future user-facing export and is no longer used by extraction. The extractor's deliverable is propositions + REASONING CHAINS (owner redesign, 2026-09-30): a chain is premises that JOINTLY entail a conclusion — independent arguments for the same conclusion are separate chains, conclusions are reusable by id as premises, and a bare proposition rides the propositions list alone. The wire schema carries propositions (id/name/statement/optional aliases) + chains (premises minItems 1/conclusion) — and since the chains-primary ruling (2026-09-30) the STORE carries exactly that wire shape: chains append as-is to `chains.jsonl` (`SessionFindings` IS the store shape now; `chainsToEdges` and `chainsToGraph` are deleted — no translation seam exists). **Supports edges are retired entirely** (owner, same evening): `EdgeType`, the supports draw toggle and thickness slider, and every supports code path are gone — a stored typed edge no longer exists anywhere, and every arrow on the canvas is derived from a chain. No migration, no backcompat: the pre-chains graph data was wiped by ruling. The chain laws are enforced by the error-back hook at the submit seam: every premise/conclusion id must resolve to a submitted proposition, premises are non-empty, no self-reference — a violation returns an error string and the model re-submits within its turns. The prompt is GOAL-FIRST, no rule armor (owner philosophy ruling, 2026-09-30): one shared `EXTRACTION_CORE` (the goal, the truth law, the submission mechanics) plus two one-sentence contextualizations — session and notes preambles — assembled at the call sites, so both front doors share the core by construction; the old defensive lists (DROP lists, merge-your-own-output, name walls, when-in-doubt-omit, the faithfulness paragraph) are all cut, and the old opening describing Quiz/Recall-check/Question blocks (stale since the teacher-prose ruling) is simply omitted. A `prepareArguments` heal at the pre-validation seam rescues stringified-payload submits (propositions or chains). Every node is an objective, world-facing fact. No quiz outcomes, no understanding assessments — the events ledger and `UnderstandingLevel` are deleted; the learner's relationship to a proposition is practice territory — practice sessions (§7) consume graph material as feedback-only context and never write learner states back to the graph.

**Digest-aware serialized extraction — the extractor is the assembler** (owner redesign, 2026-09-30 evening): the spine stage came and went in one day — the owner's live test showed the post-hoc assembly pass added complexity without perceptible gain, because the real problems (restatement nodes, shallow re-derivations) are mint-time issues. The replacement moves assembly INTO the extractor: when a source already yielded material, each stretch's extractor prompt carries **THE GRAPH SO FAR** (that source's merged propositions and chains, ids verbatim), and the extractor builds on it — referencing existing ids instead of restating them, leaning on established conclusions as premises. Chains may therefore span stretches: endpoints are a mixed namespace of fresh local ids OR existing `n_` ids (validated at the submit seam, error-back; the merger's job-local map passes existing ids through verbatim). The system prompt reframes the goal as uncovering the source's **core ideas** and showing how everything **derives from the foundations** — direction upward, prefer lean derivations, never re-derive what the graph already holds. Because each stretch must read every earlier merge, a source's batches/chunks run **STRICTLY IN ORDER** (serialized per source); cross-source runs stay parallel. The first stretch of a source runs bare (no digest, no extra spend).

**Chain simplification — the merger's maintenance pass** (owner, 2026-09-30 evening): the same conclusion often arrives from many angles — `[A,B]⟹X`, `[X,C]⟹Y`, and then `[A,B,C]⟹Y`, which adds nothing. At every merge, `simplifyChains` (core/graph.ts) walks the WHOLE chain ledger and drops provably redundant chains: per conclusion group, greedy in (premise count, creation time) order, a chain is redundant when a kept chain's premises strictly contain its premise set (**prefer the fewest premises**) or when every premise is covered by a kept chain's premises plus their transitive ancestry (**ancestry-cover** — `[A,B]⟹X` + `[X,C]⟹Y` covers `[A,B,C]⟹Y`, and transitively through deeper derivations). Same-(premises, conclusion) lines are one logical chain — a key drops with every provenance line. The rewrite rides the merge's audit-locked critical section (atomic `rewriteLedgers`); run stats count the drop separately (`droppedChains`) so appends and maintenance never blur. Simplification runs at merge time because a NEW chain can make an OLD one redundant — nothing is retroactively re-derived, only provable redundancy is removed.

**Per-chunk jobs, serial per source** (owner rulings 2026-09-28 + 2026-09-30 evening): every chunk (notes file) and every batch (chat coverage) is its own JOB — registry row, usage record, failure domain — but a source's jobs run STRICTLY IN ORDER (each extractor continues the source's digest; see above). Cross-source extracts stay parallel, throttled by a process-wide 3-permit extract semaphore on `globalThis` (the provider-facing rate stance; a job waiting for a permit shows `queued`). The audit→merge section remains a process-wide FIFO critical section — a mutex on `globalThis` (the same module-instance idiom as the run registries): before any auditor LLM call or graph read, a job waits (`waiting` phase) until every earlier job's merge has landed, so each auditor explores the graph as it actually stands — two concurrent audits would each link into a graph the other is about to change, the structural mint of duplicate nodes. Failure isolation: one job failing (bad chunk, degraded model output) is absorbed — its messages stay uncovered and honestly re-runnable while its siblings still merge; the run returns what actually merged. Abort is source-wide: the serial loop stops at the first AbortError, and merged jobs keep their checkpoints. The status feed shows one row per job (`file.md · 2/5` suffixes for chunked sources), and each job persists its own usage record (failed jobs bill their partial spend too).

**Division of trust** (the pipeline's core rule): the extractor is fully trusted and is the only stage that ever sees the source. The auditor never sees the transcript — it receives the findings' PROPOSITIONS ONLY (the chains never reach the auditor: they are the extractor→store deliverable and the merger's business — owner ruling, 2026-09-30) and explores the existing graph through read-only tools (`find`, `list_sessions`, `list_propositions` — statement text included, no timestamps) to reach its verdicts. VERDICTS-ONLY (owner ruling 2026-09-28): the auditor's submit is `{ links }` — identity links only (which findings propositions ARE existing nodes). The edges argument is GONE (owner, same evening): the auditor never creates relations between propositions — cross-source connection happens solely through identified equivalences, and argument structure comes from the extractor alone. It never restates the findings; the merger joins findings + verdicts, so an order of magnitude less is emitted and there is no structural way for the auditor to alter or drop extractor output. Equivalence bias (owner ruling 2026-09-27): paraphrases, rewordings, and compressions of the same claim MERGE; only genuinely distinct propositions stay apart. The deterministic merger validates and appends: link resolution, chain resolution through the job-local map (which passes existing `n_` ids through verbatim for the digest-aware extractor's mixed-namespace chains), chain dedupe by (sorted premises, conclusion) — a known argument re-asserted by another chat appends under the SAME `arg_` id (link-style: the first line's id and createdAt stand, provenance unions) — the chain-simplification maintenance pass (below) — coverage checkpoints. No stage rejects the extractor's output; simplification only removes provable redundancy. **Per-job in-memory resolution** (owner, 2026-09-29): the mappings ledger is deleted — each job resolves its own proposition ids against the graph in memory and merges independently, so no job can see or collide with another's ids (the persisted table once silently dropped same-id propositions across chunks and welded wrong edges). Cross-job and cross-run identity is the auditor's links alone, plus the coverage ledger (nothing re-extracts) and the notes content-fingerprint guard.

**Coverage, not quiz-counting** (supersedes the v1 every-N-quiz-answers trigger): each message's *view* (the compact line format `core/coverage.ts` has always hashed) is hashed into a per-chat fingerprint ledger (`coverage.jsonl`); "covered" means exactly "a run committed on this content." The prompt view is the richer markdown document — a deliberate 2026-09-27 split: re-formatting presentation never re-opens covered history. Uncovered = fingerprint diff, never dual-tracked; runs are greedy token-budgeted batches (`graph.extractionChunkTokens` — a settings knob, default 4096, floor 500, read fresh each turn) of the oldest uncovered messages — quiz-less sessions extract too. A coarse `sessions.jsonl` checkpoint still marks each merged run. The trigger seam (`turn-service.ts`) is trigger-agnostic from the pipeline's side: when a better mechanism lands (session end, idle, explicit flush), only the seam changes. Single-flight per source; the run registry powers the coverage UI and true abort.

**Two front doors, one pipeline**: chat transcripts (quiz-triggered) and raw personal notes (`POST /api/graph/import` — each file becomes a `NotesSource` playing the `chatId` role, deterministic id from the path hash; a content-fingerprint stamp makes an unchanged re-import a TRUE no-op — no model calls, no usage — while changed text re-runs the whole file, owner 2026-09-29). Notes chunk like chats (owner, 2026-09-28): large files split at line boundaries under the same token budget, and each chunk runs as its own job exactly like a chat batch — serial per source (digest order), serialized audits under the gate. No single model call carries a whole file. Both submit tools require their top-level fields (`chains`/`links` present even when empty): a payload missing them fails validation rather than silently merging structure-less nodes.

**The read channel is pure derivation.** `GET /api/graph` reads the ledgers — the home view never triggers a write. **Chains-primary (owner ruling, 2026-09-30): the stored truth is propositions (`nodes.jsonl`) + reasoning chains (`chains.jsonl`); the canvas renders a LOSSY DERIVED IMAGE of what is truly stored.** `graphSnapshot` merges the ledgers, then `deriveView` (core/graph.ts) rebuilds the view on every read: each stored chain mints an IMPLICATION NODE (id = the chain's `arg_` id, name `⟹`, statement = the chain line) plus untyped arrows premise→implication→conclusion; dangling chains (a member node since deleted) drop. This is the BIJECTION the edge store could not hold: two independent arguments for one conclusion (`A∧B⇒E`, `C∧D⇒E`) render as two distinct diamonds instead of a manufactured `A∧B∧C∧D⇒E` fan-in — one stored argument, one visual argument. Implication nodes are regular physics citizens for now (rendering specialization is deliberately future work). Clusters are the BIRTH-SESSION partition over propositions only (`birthSessionPartition`, core/session-clusters.ts — session provenance; no Louvain anywhere, whatever old comments claimed) — implication nodes are never partitioned and ride the physics unclustered. Node detail cards show name + proposition + birth sessions; the edge lists died with typed edges (owner, 2026-09-30).

**Deletion is full death, cascade-aware.** Deleting a proposition kills EVERY stored chain that references it, premise or conclusion — the derived implication node vanishes with the last of them, and positions are pruned. Deleting an implication node (any `arg_`-prefixed id — the canvas treats them as deletable nodes) deletes exactly that ONE stored argument and nothing else. Same minimal confirm dialog from the canvas right-click and the chain table's fenced Delete.

**The teacher probes, nothing is injected** (supersedes the semantic-injection experiment, killed 2026-09-23): the teacher reaches the graph on its own terms through two read-only prober tools — `list_sessions` (birth sessions, the learner's files) and `list_propositions` (the names of what a session covered) — gated by the `agent.graphTools` setting (composing with `graph.enabled`). No topic embedding, no top-K selection, no session-start steer; the agent decides what to look at, and the graph never writes into a lesson unbidden.

**Rendering**: a hand-rolled Canvas 2D renderer (`components/graph-canvas.ts` — one rAF loop, dirty-flag idle, d3-zoom camera, zoom-gated label fade, cluster hulls computed at draw time) over a d3-force layout in core (`layout.ts`, warm-started `incrementalLayout` + `settleGraphLayout`). Render settings and drag-committed positions persist server-side; the camera persists per-device in localStorage.

---

## 7. Practice — sessions, not a separate system

Practice is a **session kind**, not a subsystem. `ChatKind` is `"teach" | "practice"` (absence reads as teach); a practice record is created by `POST /api/chats` with `kind: "practice"` and lives in `data/chats/*.json` like any other chat. The same `handleChatRequest` turn pipeline runs it, but the handler builds `createPracticeAgent` instead of the prober/teacher. There is no `PracticeStore`, no `data/practice`, no `/api/practice` route, and no practice pipeline.

**The agent and its tools.** The practice prompt (`src/agent/prompts/practice/`) carries the method — practice is strengthening, not assessment: help what the learner already met come to mind and run in a new situation, with minimal hints and no certification. `src/agent/tools/practice.ts` carries the two core tools. `problem` poses one problem: its arguments hold the statement the learner sees, the demand mode, optional target proposition ids, and a `modelSolution` that stays hidden. The call ends the turn. The learner works in `PracticeDock` and submits labeled thinking traces; the agent reads whether a trace claims an answer and responds with the hint ladder or a verdict. `conclude` records the outcome, verdict, hint rung, miss class, and retest flag, then reveals the committed solution. `activeProblemOf` is the single fold that keeps at most one problem active.

**Starting a session.** Material can arrive two ways: picked propositions from the graph, or a free-form topic/problem in the learner's own words. Picks are optional; when present, `POST /api/chats` writes a `graph-context` marker as the first message, carrying the picked propositions and the chains among them. The practice agent may target propositions by id, validated against the whole graph; an untagged problem is valid when the material is not in the graph.

**No graph writes.** Practice consumes graph material as feedback-only context and never writes to the graph. `handleChatRequest` skips extraction for practice records, so a practice session produces no new propositions, chains, coverage checkpoints, or understanding levels. The session record is the deliverable.

**Surfaces.** The normal chat transcript renders practice cards (`ProblemCard`, `ConcludeCard`, `ThinkingCard`); an active problem swaps the composer for `PracticeDock`. The session list includes `kind: "practice"` rows like any other chat. There are no problem sets, no separate grade store, no partitioner/generator/grader jobs, and no practice rows in the activity feed.

The old problem-set implementation — partitioner, generators, graders, `/practice`, `data/practice` — is gone; §13 records the replacement.

---

## 8. Process boundaries (the machine room)

```mermaid
flowchart LR
    subgraph DESKTOP["Electron app (production)"]
        MAIN["Electron main (electron/main.cjs)<br/>launcher + credential/security owner<br/>origin-pinned controls · private child IPC"] -->|"spawn: ELECTRON_RUN_AS_NODE,<br/>loopback port, LH_DATA_DIR=userData/data"| NEXT["standalone Next server (child)<br/>teaching/inference logic: turns · extraction · jobs · graph"]
        MAIN <-->|"subscription status + scoped auth [inherited IPC, never refresh tokens]"| NEXT
        MAIN -->|"loadURL, sandboxed renderer"| REND["Browser renderer (components/)<br/>fetch/WebSocket to own origin only"]
        REND -->|"subscription controls [preload IPC]"| MAIN
    end
    subgraph OUTSIDE["outside the process"]
        PROV["model providers — BYOK keys (auth.json/env) or subscription connections (desktop OAuth) · never argv"]
        DG["Deepgram STT (browser-direct WebSocket)"]
        GCSE["Google CSE / Jina (researcher tools)"]
        CHROME["system Chrome (headless: demo verify · SVG render)"]
        DISK["data/ ledgers (via LH_DATA_DIR seam)"]
    end
    NEXT --> PROV
    NEXT --> GCSE
    NEXT --> CHROME
    NEXT --> DISK
    REND --> DG
    REND -->|"fetch /api/*"| NEXT
```

The Electron main is a launcher + credential/security guard, never a host for teaching or inference logic: single-instance lock, deny-by-default permissions, origin-pinned navigation, sandboxed renderer. All Next.js code runs unchanged in a child process (`output: "standalone"`), relocated by the `LH_DATA_DIR` seam (also the test seam — `data-dir-seam.test.ts` covers it, including a fresh-child-process case). The build hard-fails if repo `data/` would leak into the bundle or if prompt assets fall out of the trace (`scripts/build-server.mjs`).

---

### 8.1 Subscription connection boundary

`electron/chatgpt.cjs` retains the existing verified-account ChatGPT authentication, storage and revocation behavior. `electron/subscriptions.cjs` delegates GitHub Copilot/Grok login and refresh to the installed Pi SDK through its public provider/Models interfaces. Their single saved connections live independently in owner-only, atomically written `github-copilot-credentials.json` / `xai-credentials.json` beside the existing app credential files. In packaged desktop mode Electron main alone owns durable OAuth credentials and serialized refresh; no global Pi files or API-key credential precedence are involved.

The packaged desktop renderer gets projected status, public device user codes, prompts and controls over origin-pinned IPC. The privately launched server child receives credential-free catalogs and scoped request auth (including native headers/account endpoints), never refresh tokens. `src/agent/subscription-bridge.ts` rejects framework-only IPC and invalidates stale connection/catalog scopes. `subscription-stream.ts` dispatches directly through native providers, retaining native protocol identity on the wire and restoring logical app identity in assistant history. Persisted routes `chatgpt`, `github-copilot-subscription` and `xai-subscription` are distinct from ordinary API routes and never fall back to them.

For ordinary Next web/dev, `src/server/web-subscriptions.ts` composes these same plain-Node controllers into one lazy process-global credential owner beside configured `AUTH_FILE`. The server injects the private transport through `src/agent/subscription-runtime.ts`; the agent layer never imports the server layer. Existing scoped leases, authority revisions and foreground/background abort listeners work through that transport, without exposing an HTTP token/lease operation. `src/server/web-subscription-http.ts` exposes only same-origin status/control operations. Generic `subscriptions-changed` notifications use the existing SSE notify→fetch channel; opaque one-use browser actions redirect to provider pages without putting authorization URLs in app JSON. Browser adapters preserve the UI bridge contracts; Electron dev uses the web owner with a narrow system-browser opener. Desktop markers fail closed rather than falling back to web credentials. This is the existing shared developer-server trust model, not user authentication or DNS-rebinding protection; LAN behavior is unchanged. Only one server process may own an auth directory. See [development](development.md#subscription-connections-in-webdev) for storage, browser locality and access limitations.

`subscription-request-cycle.ts` is the shared admission/automatic-input recovery state machine; the old ChatGPT module is its compatibility export. Background consent is independent per provider, default ON, with foreground-preserving cancellation and queue ownership retained on no-dispatch. Native usage rejected before dispatch is transiently excluded from helper usage collectors, without adding persisted provenance fields. Catalogs and read-time pricing preserve the selected connection: subscription use displays compact provider labels, API dollars remain dollars, and mixed usage joins both without estimating subscription dollars or claiming detected tiers/allowances.

## 9. The foundational invariants — what ALWAYS holds

### Tier A — machine-enforced (`eslint.config.mjs`; lint fails the build)

| Law | Zone |
|---|---|
| **Imports point inward.** `src/core` never imports `agent`/`server`/`store` | zone 3 |
| **The shell sees barrels only.** `components/**` → `@/src/core`; `app/**` → `@/src/core` + `@/src/server` | zones 1–2 |
| **The agent layer never imports server or store** — every dependency arrives as a factory/closure argument | zone 4 |

### Tier B — gate-enforced elsewhere

| Law | Gate |
|---|---|
| Strict types, no silent indexing (`strict`, `noUncheckedIndexedAccess`, `verbatimModuleSyntax`) | `tsc --noEmit` |
| No session data in the bundle; prompt assets traced | `build-server.mjs` hard-fails |
| The core barrel stays browser-safe (node-bound/heavy modules deliberately excluded) | barrel doc-comment; chunk-verified when the exclusions were set |

### Tier C — conventional, verified to hold in the code today

| Law |
|---|
| **One composition root per concern** — `stores.ts` (storage), `chat-handler.ts` (a turn) |
| **One writer per ledger** — the merger is the sole graph writer; chat records written only by the turn pipeline; job records only by the runner; positions PUT is the only sanctioned graph-adjacent exception |
| **The store never decides** — adapters persist and shape-guard; merge/dedup/derivation semantics are reader-side, in core |
| **Derived, never dual-tracked** — append-only evidence; every state recomputed at read (snapshot, plan statuses, phases, costs, coverage) |
| **Idempotency via deterministic keys** — steer ids, ledger refs, coverage fingerprints, source content fingerprints, cluster content hashes; re-running anything converges |
| **LLMs judge; code mechanizes** — grading, merging, fingerprints, layout are deterministic; a failed LLM stage checkpoints nothing |
| **The persisted record is the only source of truth** — agents are stateless per-turn rebuilds; no unanswered tool call reaches a provider |
| **Failure containment** — side channels (context, extraction, usage, naming) are fire-and-forget behind try/catch seams; a side-channel failure never touches the response or the persist; the turn registry clears on every path |
| **Absence is not zero** — unreported tokens, absent prices, absent levels render as absent, never fabricated |
| **Schema-valid ⇔ parseable** — boundary zod schemas are shared verbatim by producer and consumer |

### Known deviations from the ideal (documented, deliberate or accepted)

- Two portless settings adapters re-exported through the server barrel; `observability.ts` imports `chatStore` directly. The "only `stores.ts` imports adapters" rule is documentation-only.
- `scripts/` and `electron/` are lint-exempt; scripts act as extra composition roots (and backfills are, by design, extra graph writers); `tests/` imports freely across layers.
- Small conventional-only sync points: `SIDEBAR_AUTOCLOSE` mirrored in `electron/main.cjs` ↔ `components/sidebar.tsx`; IPC channel names mirrored by comment; the jobs dir path hardcoded in the adapter rather than `config.ts`.

---

## 10. Observability and change delivery (the push model, owner ruling 2026-10-02)

Structured-logging frameworks are still deferred; the baseline is one correlation-tagged log line per event at the seams (`[chat <id>]` on turn services, `[extraction <id>]` on the pipeline, `[job <id>]` on runners, `[graph]` on reads). What has grown past log lines is the **activity feed** (`server/observability.ts` → `GET /api/activity` → the activity dock): running jobs and extraction runs, per-run usage (money channel), and true abort (`POST /api/activity/abort` — kind `job`/`extraction`). When structured logging lands, it goes behind these seams — no call sites change.

### 10.1 The events channel — how the client learns truth moved

Before 2026-10-02 the client discovered server-side change by polling: a lattice of interval polls (sidebar 2.5s, dock adaptive 2.5/10s, jobs 1/4s, graph status 2s, practice 2s, post-turn watches) whose blind spots were measured in the probe→teach handoff freeze (a server-initiated turn in a jobless session was invisible to an open view). The ruling: **push** — exactly one app-scoped SSE channel, `GET /api/events` (`core/events.ts` bus → `server/events-stream.ts` seam), held open by **one EventSource per open window** (`components/server-events.tsx`, mounted outermost in the root layout — the only file where `EventSource` exists). The client owns the connection's lifetime; the server frames `: ping` heartbeats every 25s and never closes first. **Notify→fetch**: payloads never carry data — an event says only *that* truth moved (plus a key like `chatId`); every consumer then fetches fresh from the normal endpoints. Reconnects (EventSource auto-retry) fire a **resync** hook per consumer for one immediate refetch, healing any events the gap ate.

### 10.2 Event vocabulary

| Event | Payload | Emitted at |
|---|---|---|
| `turn-started` | `{chatId}` | `beginTurn` success (after the registry write; the double-begin 409 path is silent) — every turn, client-initiated included |
| `turn-settled` | `{chatId}` | `endTurn` on a live entry — fires after `persistTurn`, so fetches read durable truth |
| `jobs-changed` | `{chatId}` | every job-store write (create/update/delete-that-removed) |
| `activity-changed` | `{}` | extraction `setPhase`/`finishExtraction` |
| `chats-changed` | `{}` | list-shape writes only: chat create/rename/delete-that-removed |

### 10.3 Emitter laws

- **Notify-after-write** — the emit always follows the successful write, so a fetch triggered by the event reads committed truth.
- **Present-entry gating** — mutations that changed nothing (missing-id update, delete of an absent row) emit nothing.
- **THE SILENCE LAW** — content writers (`saveMessages`, `appendTurnUsage`, `appendExtractionUsage`, `saveSettings`, `saveTree`) are deliberately silent: `saveMessages` would spam per turn-persist, and list-visible effects already arrive via `turn-started`/`turn-settled` (pinned in tests).
- **Reads never emit** — pure reads are side-effect-free (pinned).

The bus itself is a `globalThis`-backed listener set (the Next per-entry-module discipline, same as the turn registry), snapshot-iterated so unsubscribing mid-emit is safe, with per-listener error isolation.

### 10.4 Consumer laws

- **Refs, not closures** — handlers read `localStreamRef`/`turnInFlightRef`/`statusRef`, never stale render state.
- **The 409-race guard** — on `turn-started`, ignore when `localStreamRef.current` is held (the client's own turn just began; it owns that stream) or already attached; otherwise attach to the running turn (`GET /stream` live response).
- **No mid-attached-stream sync** — while attached, the terminal frame owns settle; a `turn-settled`-triggered `syncFromServer` would swap a stale snapshot and break in-place continuation (the one exception: error status heals by detaching).
- **~250ms trailing debounce** — bursty emitters coalesce into one fetch per window (sidebar/dock/jobs/graph all share the shape).
- **Resync on reconnect** — every consumer refetches once when the channel (re)opens.

### 10.5 What remains bounded or local, and why

- **Extraction-coverage ×2** (session menu block + ring) — bounded, run-scoped 2s reads: the ring drains **per fold**, and the events channel is per **phase**; that granularity gap is accepted by design (owner, 2026-10-02).
- **The 409-retry poll** (`session-core.ts`, 500ms→5s ramp) — a racing client's bounded backoff while a turn it tried to start is already running; not standing.
- **Dock clock ticker** (1s `setNow`) and **dictation audio meter** — local rendering, no network.
- **Mount fast-paths kept** — `initiallyInFlight` (attach on render-time snapshot), the mount-time `needsContinuation` check, and the `lh:chat-renamed`/`lh:turn-state` **local window broadcasts** (instant same-tab patching; the channel is for cross-window and server-initiated truth).

---

## 11. Testing

All deterministic, no network: engines are injected (pi-ai's **faux provider** — `createFauxCore`/`setResponses` — wrapped as the `PiEngine` production uses), `globalThis.fetch` is stubbed where transport matters, and pure logic gets in-memory port implementations. Suites either mirror production wiring with the real fs adapters (backup-and-clear of the ledgers they touch) or swap in-memory ports deliberately ("this suite never touches data/graph"). Four strata:

1. **Pure core units** — merger via real adapters, derivation, coverage, question grading, plan tree, phases, clusters, layout, tool-parts, prices, fuzzy search.
2. **Server flows** — `handleChatRequest` driven end-to-end with scripted faux engines: persistence, liveness, steer claiming, the wake pipeline (409 retry, quiz deferral, idempotency), crash recovery, the data-dir seam (including a fresh child process).
3. **Sub-agent loops** — researcher/demo-builder/visual-maker with fake headless APIs; the extraction pipeline end-to-end (audit drops, failure-checkpoints-nothing, single-flight, abort); practice sessions end-to-end (problem pose, thinking submissions, conclude/reveal, dismissal), the active-problem fold, target validation, practice argument healing, and the prompt's hint ladder.
4. **Render tests** — transcript, cards, docks, composer, sidebar, coverage, stats, and practice surfaces (practice dock, problem/conclude/thinking cards, graph-context card, view projection) via React server-side rendering (`renderToStaticMarkup`), no DOM library.

Gates on every change: `tsc` clean, all tests, `eslint` clean, `next build` clean.

---

## 12. The seam directory

Every place the architecture is designed to be cut — the "one file" a future change should touch:

| Seam | File | What it swaps |
|---|---|---|
| Storage decision (JSON→SQLite, remote) | `src/server/stores.ts` | the adapters, once |
| LLM engine / model per role | settings `models` slots + `src/agent/models.ts` (`piEngineForSpec`) | provider wiring, one place |
| Turn composition | `src/server/chat-handler.ts` | what a turn consists of |
| Extraction trigger | `src/server/turn-service.ts` (`triggerExtraction`) | when the Library runs |
| Practice agent | `src/agent/tools/practice.ts` + `src/agent/prompts/practice/` | problem/conclude behavior without touching the turn pipeline |
| Graph access for teacher | `src/agent/tools/graph.ts` (`list_sessions` / `list_propositions`) | what the teacher can probe, one gate |
| Steer delivery (wake) | `src/server/jobs.ts` + injected `deliver` | how background results re-enter lessons |
| Push events | `src/core/events.ts` (bus) + `src/server/events-stream.ts` (SSE seam) + `components/server-events.tsx` (the window's EventSource) | change notification, end to end |
| Agent construction (tests) | `handleChatRequest(req, buildAgent?)` | scripted engines, no refactor |
| Sub-agent verification | `sub-agents/headless.ts` (`headlessOverride`) | demo/SVG verification, no Chrome |
| Data location (Electron) | `LH_DATA_DIR` env (`config.ts` PATHS) | where data/ lives, nothing else |
| Desktop host | `electron/main.cjs` | launcher/security shell + desktop subscription credential bridge |

---

## 13. Decision history — superseded decisions (do not re-litigate without new evidence)

Each of these was implemented, lived in real data, and was replaced; the replacement reasoning is in `docs/vision.md`'s amendments table.

| Superseded | Replaced by | When |
|---|---|---|
| Live-emission graph construction (per-session, plan-tree-derived) | Post-hoc, tree-blind extraction pipeline | 2026-09-02 (vision §1) |
| `taught` events, misconception nodes, node descriptions/kinds | True propositions only; wrongness lives in the ledger; name+statement+aliases | 2026-09-02/08 |
| Editable plan tree (`plan_add_node`…`plan_mark_done`) | Author-once `plan_init` + derived statuses | 2026-09-08 era |
| Hotspot + 1-hop graph context | Semantic top-K under token budget, delivered as a steer | 2026-09-14 (vision-context-injection) |
| every-N-quiz-answers extraction trigger | Token-coverage batches (fingerprint ledger) | 2026-09-15 (vision-extraction-coverage) |
| sigma.js + graphology + forceAtlas2 + Louvain clustering | Hand-rolled Canvas 2D renderer, d3-force layout, embedding-based agglomerative clusters | 2026-09-13 era |
| Vercel AI SDK LLM loop (`MockLanguageModelV4` tests) | pi-ai engine (faux-provider tests), AI SDK kept as the UIMessage protocol | pi-engine-swap |
| Problem-set practice (`PracticeStore`, `/api/practice`, `practice-pipeline.ts`, `/practice` page) | Practice as a `kind: "practice"` chat session with `problem`/`conclude` tools | 2026-10-04 |

## 14. Deliberately deferred

Structured logging (§10); SQLite/remote storage (drop-in behind the ports, §3); multi-scale cluster hierarchy (Louvain's dendrogram idea, parked); the anchored calibration rubric; alternative extraction triggers beyond the coverage seam (session end, idle); retroactive bootstrap of the POC sessions (vision §1 consolidation). When any of these land, this doc changes with the commit that lands them — it describes the system, not an intention.
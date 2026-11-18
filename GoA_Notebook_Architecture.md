# GoA — ARC-AGI-3 Notebook Architecture

**Files documented**
`GoA_RTX6000_25Game_Run-7.ipynb` and `GoA_RTX6000_25Game_Run-21_3.ipynb`

**Prepared** 19 November 2026 · Group of AI (GoA)
**Method** every figure below was read directly out of the two `.ipynb` files — cell sources parsed with `json`, Python modules parsed with `ast`. Nothing here is recalled from earlier reports.

---

## 1. The headline finding

These two notebooks are **the same notebook, one with an extra layer bolted on.**

`Run-7` has 46 cells. `Run-21_3` has 36 cells. I hashed every cell source in both files and compared:

- Cells in `Run-7` that are **not** in `Run-21_3`: **10 cells — indices 20 through 29.**
- Cells in `Run-21_3` that are **not** in `Run-7`: **none.**
- After removing those 10 cells from `Run-7`, the remaining 36 cells are **byte-identical and in the same order** as `Run-21_3`.

The 10 extra cells are the **GE layer** (GE1–GE4, the grid text-encoding subsystem). Everything else — the 16 core file-writing cells (15 Python modules plus one JSON schema), the two deep-mode prompt files, all gates, the run loop — is the same code.

So there is **one architecture here, not two.** `Run-21_3` is the core agent; `Run-7` is that same core with a separate grid-encoding research layer attached.

| | Run-21_3 | Run-7 |
|---|---|---|
| Cells | 36 | 46 |
| Bytes | 563,869 | 1,069,664 |
| `%%writefile` cells | 18 | 25 |
| my_agent version | 1.0.21 | 1.0.21 |
| Notebook generation | v18.0.0 "OBJECT VISION" | v18.0.0 "OBJECT VISION" |
| GE layer | **absent** | **present** (GE1–GE4) |
| Everything else | identical | identical |

---

## 2. Execution order

Both notebooks run top to bottom in the same order. `Run-7` slots the GE layer in after the agent modules.

### 2.1 Boot phase

| Run-7 | Run-21_3 | What it does |
|---|---|---|
| 0 | 0 | Markdown header — version, certification block |
| 1 | 1 | **PREFLIGHT v8** — 271 lines. Paths, GPU (96 GB hard check), host RAM, parquet, disk. Fails fast before anything expensive. |
| 2 | 2 | **Offline installs** — 159 lines. Installs the verified wheel set from the offline bundle; builds the games cache. |
| 3 | 3 | **vLLM bootstrap v7** — 129 lines. Skip-if-installed, exact wheel matching. |

### 2.2 Module-write phase

Sixteen `%%writefile` cells lay down the core agent package at `/kaggle/working/goa/agent/` — 15 Python modules and one JSON schema. Order matters: later modules import earlier ones.

### 2.3 GE layer (Run-7 only)

`Run-7` cells 20–29. See §5.

### 2.4 Certification and launch phase

| Run-7 | Run-21_3 | What it does |
|---|---|---|
| 30 | 20 | **Certified file repair v15** — verifies each module's md5 against the pin, repairs trailing-newline drift, refuses to continue on a mismatch |
| 31 | 21 | **R-13 build check v7** — builds the single-file artifact and asserts it embeds exactly the 5 certified sources |
| 32 | 22 | **vLLM launch + runbook globals** — starts both servers, waits for health |
| 33–34 | 23–24 | Two deep-mode prompt files (`IMP2P`, `IMP2B`) |
| 35–36 | 25–26 | **Template-conformance** — sets `CONFORMANCE_PASSED` |
| 37 | 27 | **Frozen pre-flight runbook** — 855 lines, 8 checks (CS-1…CS-8) |
| 38 | 28 | **Coder-solver conformance** — 8 checks, arms `CODER_CONFORMANCE_PASSED` |
| 39 | 29 | **Frame probe v14** — proves the framework `Agent` can be loaded directly |
| 40 | 30 | **THE RUN** — the actual 25-game loop |
| 41 | 31 | Coordination summary — thinker + coder evidence |
| 42 | 32 | Backup — md5 manifest + selective zip |
| 43 | 33 | Markdown note on conformance ordering |
| 44–45 | 34–35 | md5 register (two views of the same pin set) |

---

## 3. The agent modules

Both notebooks write the same 16 core files — 15 Python modules plus one JSON schema. Sizes are characters of source; line counts are physical lines. These 16 cells total **324,210 characters**.

| # | Module | Lines | What it is |
|---|---|---|---|
| 4 | `context_manager.py` | 1,564 | 3-zone prompt assembly, overflow guards, grid diffing |
| 5 | `inference_layer.py` | 334 | Model-call layer, response parsing, call log |
| 6 | `grid_text.py` | 1,063 | 64×64 grid → text: objects, legend, regions, RLE |
| 7 | `action_pipeline.py` | 1,157 | Observation, delta packets, no-op prediction, calibration |
| 8 | `sandbox_v6.py` | 419 | AST-screened subprocess sandbox for coder output |
| 9 | `counterexample_schema.json` | 120 | Frozen schema for coder failures |
| 10 | `replay_verifier.py` | 94 | Transition buffer and replay verification |
| 11 | `cegis_taxonomy.py` | 42 | Classifies a coder failure into a repair class |
| 12 | `cegis_router.py` | 32 | Maps failure class → repair budget |
| 13 | `adversarial_critic.py` | 34 | Critiques a spec before it is executed |
| 14 | `inference_layer_v1_6.py` | 16 | v1.6 shim |
| 15 | `inference_layer_v1_6_additions.py` | 39 | v1.6 additions |
| 16 | `coder_solver.py` | 285 | The CEGIS loop: generate → run → counterexample → repair |
| 17 | `my_agent.py` | **2,419** | The agent itself — `Agent`, `MyAgent`, loop detector, calibration |
| 18 | `lora_datagen.py` | 772 | Training-data extraction from replays |
| 19 | `vllm_transport.py` | 206 | HTTP client for both vLLM servers |

`my_agent.py` is the centre of gravity: 2,419 lines, more than a third of the whole package.

### 3.1 Key classes

**`my_agent.py`** — `Agent`, `MyAgent`, `LoopDetector`, `StuckSig`, `JoltRequest`, `LocalCalibration`, `CRLFIntakeEntry`, `_FallbackAction`, `_FakeCM`, `_PromptPacket`; plus error types `AgentError`, `VersionMismatch`, `VersionMissing`, `ComponentUnavailable`.

**`context_manager.py`** — `ContextManager`, `PromptPacket`, `GridDiffEncoder`, `_Capture`; errors `ContextOverflowError`, `Zone2OverflowError`.

**`grid_text.py`** — `GridTextConverter`, `ObjectRec`; errors `GridTextError`, `RegionTooLargeError`, `DimensionMismatchError`, `InvalidGridError`, `InvalidArgumentError`.

**`action_pipeline.py`** — `ActionPipeline`, `NoOpPredictor`, `LiveCalibration`, `CalibrationRow`, `EventRecord`, `ActionResult`, `ObserveResult`, `DeltaPacket`; errors `PipelineError`, `NoOpDetected`, `DivergenceError`, `PremiseFailureError`, `CalibrationError`, `StaleObservationError`.

**`inference_layer.py`** — `InferenceLayer`, `ReasonerResult`, `WorldModelResult`, `DiagnoseResult`, `PromptPacket`.

**`coder_solver.py`** — `CoderSolver`, `AttemptCounters`, `SolveResult`, `GuardedExecutor`.

**`sandbox_v6.py`** — `SandboxViolation`, `_NumpyAliasCollector`; the public entry point is `run_candidate(code, entry, args)`.

---

## 4. Runtime architecture

### 4.1 Two models, two servers

Both notebooks launch **two separate vLLM servers on one GPU**:

| | Reasoner | Coder |
|---|---|---|
| Model | `qwen3.8-27b` (FP8) | `qwen3-coder-30b-a3b-instruct` |
| Port | 8000 | 8001 |
| Path | `/kaggle/input/datasets/gray2488/qwen38-27b-fp8` | `…/qwen3-coder-30b-a3b-instruct` |
| `--reasoning-parser` | `qwen3` | — |
| `--max-model-len` | 16,384 | 16,384 |
| GPU share | ~0.40 | ~0.42 |

`vllm_transport.py` holds one `VLLMTransport("http://127.0.0.1:8000", "http://127.0.0.1:8001")` and routes by role. Both endpoints speak the same OpenAI-compatible `/v1/chat/completions`, so the split is a routing decision, not a protocol difference.

### 4.2 Certified versions

```python
CERTIFIED_VERSIONS = {
    "context_manager":  "1.6",
    "inference_layer":  "1.6",
    "grid_text":        "1.0.8",
    "action_pipeline":  "1.0.5",
}
```

`check_certified_versions()` in `my_agent.py` asserts these at import time. A mismatch raises `VersionMismatch` rather than degrading quietly.

### 4.3 The per-step loop

`GoAMyAgent` subclasses `ma.MyAgent`, which owns the framework contract. The path is:

```
framework Agent
  └─ MyAgent.choose_action(frames, latest_frame)
       └─ Agent.framework_choose(latest_frame, prev_state)
            └─ Agent.act(frame)            ← everything happens here
```

Inside `act()`:

1. **Settle** the frame, compute the RLE anchor and checksum.
2. **Eyes** — `self._gt.extract_objects(settled, background=0)` returns `ObjectRec` records (id, colour digit/char, size, bbox, ohash, pixels). The object set fingerprint `_objset_fp` is compared to the previous step to detect change.
3. **Feed the context manager** — objects go in as structured text (`[id] color=.. rows=r0-r1 cols=c0-c1`), so the model never has to do RLE column arithmetic itself. This is the "Object Vision" change in v1.0.21.
4. **Loop detection** — `self._loop.tick(...)` with `LoopDetector` / `StuckSig` / `JoltRequest`; a detected loop raises a jolt rather than repeating a move.
5. **Decide** — `_reason_and_plan()` builds the prompt from the 3-zone context manager and calls the reasoner through `InferenceLayer`.
6. **Delegate or act** — if the plan calls for a program, `_maybe_admit_coder(task_ctx)` hands a typed spec to `CoderSolver`; otherwise the reasoner's own action is used.
7. **Dispatch** — the chosen action string becomes an engine call via `_make_framework_action(name, x, y)`.

### 4.4 The 3-zone context manager

`context_manager.py` splits the prompt into three zones with independent budgets and different lifetimes:

- **Zone 1** — the static system prompt. Set once via `set_system_prompt()`; re-sent on every call.
- **Zone 2** — durable, structured state: confirmed rules, hypotheses, goal notes. `update_zone2()` and `promote_to_zone2()` write here. Each field has a token cap; exceeding it raises `Zone2OverflowError` unless `truncate=True` is passed.
- **Zone 3** — the volatile tail: recent actions, current observation, transient notes.

A hard cascade guards total size (`ContextOverflowError`), and `GridDiffEncoder` compresses grid deltas instead of re-sending whole frames. Nine self-tests (`_t1`…`_t9`) cover reset, whole-field replacement, cap-and-warn, diff round-trip, drift guard, refresh triggers, build order, gate cascade and cascade exhaustion.

### 4.5 The coder-solver pipeline (CEGIS)

```
reasoner decides "this needs a program"
        ↓
task_ctx built (typed spec)
        ↓
build_spec(task_ctx)  →  spec text
        ↓
coder model (:8001)  →  Python source
        ↓
sandbox_v6.run_candidate(code, "next_action", {frame, history})
        ↓
board changed?  ── yes ──→  admit the program
        │
        no
        ↓
counterexample → cegis_taxonomy.classify() → cegis_router.route()
        ↓
repair attempt (up to CODER_ATTEMPTS_PER_GAME_CAP = 6) → or retire
```

`replay_verifier.py` holds the transition buffer and builds counterexamples. `adversarial_critic.critique_spec()` reviews the spec before execution.

### 4.6 The sandbox

`sandbox_v6.py` is not a soft filter. It:

- parses the candidate with `ast`,
- checks every import against `_ALLOWED_IMPORTS` (`numpy`, `math`, `itertools`, `collections`) and a second module whitelist inside the child wrapper,
- builds safe builtins with `_make_safe_builtins()`,
- runs the function in a **separate subprocess** with rlimits passed via environment variables,
- returns `{ok, result, error_class}` and never raises into the caller.

`run_candidate()` is the single entry point.

---

## 5. The GE layer (Run-7 only)

Ten cells — 20 through 29. This is a self-contained grid text-encoding subsystem that the core agent does not require.

| Run-7 cell | File | Lines | Contents |
|---|---|---|---|
| 20 | — | — | Markdown: GE1–GE4 design note (DIR/092 Rev v38) |
| 21 | — | 5 | `import os` + path setup |
| 22 | `ge1/wire.py` | 3,694 | RLE encode/decode, frame registry, evidence logging, validation |
| 23 | `ge2/ge2_adapters.py` | 2,000 | Frame/state adapters, census validation, SDK vocabulary binding |
| 24 | `ge3/ge3_selector.py` | 3,099 | Policy, cost bundles, candidate selection, dispatcher |
| 25 | `ge4/integration.py` | 932 | Runtime config, Ed25519 trust verification, upstream loading |
| 26 | `ge4/gates.py` | 332 | `GateReport` / `GateEntry` — the gate framework |
| 27 | `tests/__init__.py` | 2 | Package marker |
| 28 | `tests/test_ge4.py` | 33 | `_run_all()` — exits non-zero on failure |
| 29 | — | 92 | GE integration cell — wires the four layers together |

Key classes: `ReasonCode` / `Status` (GE1, GE2), `Policy` / `CostBundle` / `Selection` / `Dispatcher` (GE3), `DispatchResult` / `RuntimeConfig` / `CapConfig` / `AuthConfig` (GE4), `GateStatus` / `GateEntry` / `GateReport` (gates).

**Size:** the ten GE cells total **476,470 characters** — **147% of the entire core module package** (324,210 characters). The extra layer is larger than the thing it is attached to, and it is the only difference between the two files.

**Important:** `ge4/integration.py` contains `_ed25519_verify`, `_verify_trust_record` and `_verify_owner_attestation`. This layer carries cryptographic trust machinery. It is not a thin formatter.

---

## 6. Run loop and budgets

Both notebooks share the same run cell (Run-7 cell 40 / Run-21_3 cell 30):

```
arc = arc_agi.Arcade(operation_mode=OperationMode.OFFLINE)
game_ids = sorted({e.game_id.split("-")[0] for e in all_envs})
        ↓  count must equal EXPECTED_GAMES (25)
smoke: arc.make(game_ids[0]) → GoAMyAgent(...) → assert game_id set
        ↓
for i, gid in enumerate(game_ids, 1):
    if global budget exceeded: abort
    env = arc.make(gid)
    a = GoAMyAgent(card_id=..., game_id=gid, arc_env=env, ...)
    a.main()
    record minutes / levels / actions / il_calls / tokens
```

| Constant | Value | Meaning |
|---|---|---|
| `EXPECTED_GAMES` | 25 | Game count the run insists on |
| `MAX_ACTIONS` | 80 | Framework per-game action cap |
| `MAX_MINUTES_PER_GAME` | 17.0 | Per-game wall-clock cap, enforced inside `is_done()` |
| `GLOBAL_TIME_BUDGET_H` | 7.5 | Whole-run cap (Kaggle limit is 9 h) |
| `MAX_MODEL_LEN` | 16,384 | Context window per server |

### 6.1 Token and size budgets

| Constant | Value | Where |
|---|---|---|
| `REASONER_ANSWER_CAP` | 200 | inference_layer |
| `REASONER_THINKING_BUDGET` | 1,000 | inference_layer |
| `REASONER_DEEP_THINKING_BUDGET` | 2,000 | inference_layer |
| `REASONER_DEEP_MAX_TOKENS` | 6,144 | inference_layer |
| `CODER_CODE_CAP` | **2,048** | inference_layer |
| `MAX_TEXT_CHARS` | 4,000 | grid_text |
| `MAX_REGION_CELLS` | 576 | grid_text |
| `MAX_DELTA_PACKET_CHARS` | 4,000 | action_pipeline |
| `_MAX_OBS_OBJECTS` | 24 | my_agent |
| `CODER_ATTEMPTS_PER_GAME_CAP` | 6 | my_agent |
| `_MAX_DEAD_SIGS_PER_LEVEL` | 200 | my_agent |
| `GE_CAP` | 2,500 | GE layer |

`CODER_CODE_CAP = 2048` in both files. The gate in the coder-solver conformance cell pins the same value, so the two agree — there is no drift between the constant and its gate in either notebook.

---

## 7. The gate chain

Both notebooks refuse to run games if certification fails. The chain, in order:

1. **Certified file repair (v15)** — each module's md5 compared to its pin; trailing-newline drift repaired; mismatch → refuse.
2. **R-13 build check (v7)** — build the single-file artifact, extract the embedded `_SRC_<module>` blobs, assert exactly the 5 certified sources are embedded and nothing else.
3. **Template conformance** — arms `CONFORMANCE_PASSED`; a skip is recorded as *not* a pass.
4. **Frozen pre-flight runbook** — 8 checks: CS-1 schema md5, CS-2 sandbox blocks `import`, CS-3 blocks `__import__`, CS-4 benign round-trip, CS-5 inference-layer `call_log` + cap, CS-6 attempt arithmetic (worst case 18), CS-7 solver wires the real sandbox, CS-8 instance-level call log + coder params seam.
5. **Frame probe (v14)** — proves the framework `Agent` loads directly and `MyAgent.main()` exists.
6. **md5 register** — two independent views of the pin set.

The chain is fail-closed by design: a genuine code regression should stop the run *before* hours of GPU time are spent, not after.

---

## 8. What neither notebook contains

Stated plainly, because it matters for anyone reading these as production artifacts:

- **No delegation brief for either model.** Neither notebook puts a role description, an agenda, or a shared protocol into the reasoner's or the coder's prompt. The reasoner prompt is a JSON contract; the coder prompt is a typed spec. The models are not told who they are or what the other one is doing.
- **No coordination protocol.** There is no GOAL → SUCCESS TEST → RESULT → NEXT GOAL structure. The coder receives a spec and returns a program; the reasoner receives telemetry but is not instructed to treat a no-op as *its own* spec error.
- **No `delegate_spec` field.** In `Run-7` the only `task_ctx` is the world-model replay path (`"mode": "world"`, *"Reproduce every buffered transition exactly"*). There is no reasoner-decides-then-coder-implements path.
- **No human-replay wiring.** `replay_verifier.py` exists, but no replay corpus is loaded into the live agent.
- **No `effect_entries` / `rhae_rows` telemetry.** The later runs (30 Oct – 1 Nov) that actually cleared levels recorded these fields in `run_results.json`. Neither of these notebooks produces them.

That last point is the load-bearing one: **these are not the notebooks that cleared the ten levels.** They are the closest surviving ancestors of that lineage — same generation header (v18.0.0, my_agent 1.0.21), but the level-scoring runs were on 1.0.23 or later, and their notebooks were never archived.

---

## 9. Summary

One architecture, two files.

The core is a three-layer agent: **eyes** (`grid_text` turns the 64×64 board into objects and text), **context** (`context_manager` assembles a 3-zone prompt with hard budgets), and **decision** (`my_agent` drives the reasoner, optionally delegates to a sandboxed coder-solver under a CEGIS repair loop). Two vLLM servers on one GPU back it — a 27B reasoner and a 30B-A3B code model — and a fail-closed gate chain stands between the build and the first game.

`Run-21_3` is that core, complete, in 36 cells.
`Run-7` is that same core plus a 476,470-character GE1–GE4 grid-encoding and trust layer, in 46 cells.

Static AST parsing found no syntax errors in the 15 Python files written by Run-21_3 or the 22 Python files written by Run-7. The uploaded notebooks contain no saved cell outputs, so this note does not assert that the certified-byte repair, R-13 build, or coder-solver conformance gates were executed or passed in these archived copies. `CODER_CODE_CAP` matches its source gate pin in both files.

Neither is the notebook that cleared levels. That file is not in any archive.

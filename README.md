# ARC-AGI-3 Agent Core

A Kaggle notebook containing the GoA ARC-AGI-3 agent core (notebook generation v18.0.0; `my_agent.py` version 1.0.21).

## What the agent does

The notebook runs top to bottom:

1. **Preflight:** checks paths, GPU and host resources, parquet inputs, and disk capacity before expensive work begins.
2. **Offline setup:** installs the pinned wheel set from the offline bundle and prepares the games cache.
3. **Model serving:** starts two local vLLM endpoints on the same GPU and waits for both health checks.
4. **Module assembly:** writes the agent package, including context management, inference, grid-text conversion, action handling, the coder solver, replay verification, and transport.
5. **Certification:** repairs/checks certified module bytes, performs the single-file build check, runs template and coder-solver conformance, executes the frozen preflight runbook, and probes framework loading.
6. **Game loop:** opens the offline Arcade environment, checks the expected game count, performs a smoke initialization, then runs the configured game loop and records run telemetry.

At each agent step, the agent settles the frame and computes its anchor/checksum; extracts objects from the grid; feeds structured object information and recent state into the context manager; checks for loops; asks the reasoner for a plan; optionally invokes the coder-solver when a program is called for; and dispatches the selected action to the framework.

## Runtime

Two separate vLLM servers run on one GPU:

| Role | Model | Local endpoint | Context length | GPU memory share |
|---|---|---|---:|---:|
| Reasoner | `qwen3.8-27b` (FP8) | `http://127.0.0.1:8000` | 16,384 | 0.40 |
| Coder | `qwen3-coder-30b-a3b-instruct` | `http://127.0.0.1:8001` | 16,384 | 0.42 |

Both endpoints use the OpenAI-compatible chat-completions interface. The notebook's transport layer routes requests by role.

## CEGIS loop and sandbox

The reasoner may decide that a task needs a program. The agent builds a typed task specification, sends it to the coder model, and passes the returned Python candidate to `sandbox_v6.run_candidate()`. The candidate is checked for a board-state change. If it does not produce the required change, the replay verifier builds a counterexample; the CEGIS taxonomy classifies the failure; the router selects a repair budget; and the solver may request a repair or retire the candidate. The configured repair-attempt cap is 6 per game.

The sandbox parses candidate code with Python's AST, checks imports against an allowlist (including a second module check in the child wrapper), constructs restricted builtins, and executes the candidate in a separate subprocess with resource limits. Its entry point returns a structured result rather than raising candidate errors into the caller.

## Gate chain

The notebook places these checks before the game loop:

1. **Certified file repair:** compares module bytes with their MD5 pins, repairs trailing-newline drift where permitted, and refuses on a mismatch.
2. **R-13 build check:** builds the single-file artifact and checks the embedded certified source set.
3. **Template conformance:** sets the conformance flag only after the checks run; a skipped check is not treated as a pass.
4. **Frozen preflight runbook:** runs the eight CS checks covering schema integrity, sandbox restrictions, benign execution, inference call logging and caps, attempt arithmetic, solver-to-sandbox wiring, and coder-parameter wiring.
5. **Frame probe:** checks that the framework Agent can be loaded directly and that the MyAgent main loop is available.
6. **MD5 register:** provides two views of the pinned file set.

## Budget constants

| Constant | Value |
|---|---:|
| `EXPECTED_GAMES` | 25 |
| `MAX_ACTIONS` | 80 |
| `MAX_MINUTES_PER_GAME` | 17.0 |
| `GLOBAL_TIME_BUDGET_H` | 7.5 |
| `MAX_MODEL_LEN` | 16,384 |
| `REASONER_ANSWER_CAP` | 200 |
| `REASONER_THINKING_BUDGET` | 1,000 |
| `REASONER_DEEP_THINKING_BUDGET` | 2,000 |
| `REASONER_DEEP_MAX_TOKENS` | 6,144 |
| `CODER_CODE_CAP` | 2,048 |
| `CODER_ATTEMPTS_PER_GAME_CAP` | 6 |
| `MAX_TEXT_CHARS` | 4,000 |
| `MAX_REGION_CELLS` | 576 |
| `MAX_DELTA_PACKET_CHARS` | 4,000 |
| `_MAX_OBS_OBJECTS` | 24 |
| `_MAX_DEAD_SIGS_PER_LEVEL` | 200 |

## What this notebook is not

This notebook is **not** the notebook that cleared levels. The scoring runs used agent version **1.0.23 or later**; those scoring-run notebooks were never archived. This notebook contains neither a delegation brief nor a coordination protocol between the two models. It should not be described as production-ready or as a competition winner.

## Scope

This repository contains the core notebook and its architecture reference. It does not claim performance results beyond what is recorded in the notebook itself.

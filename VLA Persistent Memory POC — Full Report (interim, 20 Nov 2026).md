# VLA Persistent Memory POC — Full Report (interim, 20 Nov 2026)

Nov 20, 2026 · @Jaya Krishna

## Summary

The POC's main claim now has evidence: a VLA without memory could not re-find a moved object it had seen earlier, and an external memory that stores object positions fixed that. In the three-part tasks of point 3, the robot reached the object in 0 of 5 runs without memory and 5 of 5 with the object map. With a CLIP picture memory it reached the object in 0 of 5.

**One robot, one VLA, memory as the only variable.** Everything ran in simulation: BEHAVIOR-1K on Isaac Sim 6.0.1, the Galaxea R1Pro robot, and the π0.5 policy (Comet pt50 checkpoint) on one RTX PRO 6000 Blackwell workstation. Memory never enters the VLA. It answers "where is the object?", a scripted controller turns or drives the robot there, and the VLA takes over.

| Point | Question | Status | Headline result |
| --- | --- | --- | --- |
| 1 | Can the VLA do five simple tasks with the object in view? | Done 18 Nov | Reached the object 10/10, picked it up 4/10, finished the task 0/10 |
| 2 | Does it fail once the object is out of view, with no memory? | Done 19 Nov | Usual place: found 8/10, by learned habit. Moved to a new spot: found 0/6 indoors. Moved but in view (control): reached 5/6 |
| 3 | Do memory blocks fix it in a pass-through → old task → next task chain? | Done 20 Nov | Object reached within 60 s: no memory 0/5, picture memory 0/5, room memory 3/5, object map 5/5, true position 5/5 |
| 4 | Comparative analysis | Written here (this report) | Memory helps when it can localise the object; the cheapest useful block is a room-level map; picture memory needs the object large in view |

**What the evidence supports.**

- Without memory, π0.5 only re-finds objects where training usually put them (point 2, condition A 8/10). It does not keep anything from earlier frames.
- Moving the object to a spot it never had in training removes that shortcut: the VLA never found it out of view (point 2 B, 0/6; point 3 M0, 0/5).
- An external memory restores the step it is needed for, getting the VLA to the right object, without retraining it (point 3).
- How the memory stores things matters more than having one: object positions worked every time, rooms worked when the room's centre was near the object, and CLIP-ranked frames never did.

**What it does not show yet.** Task completion: the VLA finished none of the point 1 to 3 tasks (phase 1 had five full successes: 1 of 84 radio-checkpoint runs, 4 of 39 Comet runs), so memory has been shown to get it to the right object, not to finish a long-horizon task. The object map also uses the simulator's perfect object labels, so it is a best case.

**Heads-up.** This is an interim report, not the end of the work. Open next steps: making the VLA finish tasks (the training question), replacing the perfect labels with a real detector, a moved-after-seen probe, and more instances.

## About this report

This report covers the whole POC from 23 Oct to 20 Nov 2026 in one place, and it is a snapshot, not a closing report. Work continues, so later results will be added to the Baseline Report and the wiki first.

The POC ran in two phases:

- **Phase 1, the original eight-step POC (10–17 Nov).** It built the platform and tooling, and tested the radio task with two policies, five start conditions and three memory blocks. It closed with the [17 Nov final report](https://claude.ai/artifact/Q9zPpAnN5LYvGyoTNtSGfA).
- **Phase 2, the four points (18–20 Nov).** After a scope reset it re-ran the question on five tasks: in view (point 1), out of view without memory (point 2), and with memory in three-part tasks (point 3). Point 4, the comparison, is written for the first time here.

| Document | What it holds | Where |
| --- | --- | --- |
| This report | everything, end to end, plus point 4 | this page |
| Baseline Report | points 1–3 with claim sheets and videos | [Baseline Report](https://claude.ai/artifact/KynN2mHczShBSaYt3BS7DY) |
| Phase-1 final report | the eight-step POC, claims C1–C13 | [Final Report, 17 Nov](https://claude.ai/artifact/Q9zPpAnN5LYvGyoTNtSGfA) |
| Knowledge wiki | glossary, architecture, decisions, issues, how-to | [Knowledge Wiki](https://claude.ai/artifact/AnEyqV7epZS7j3H3ytY58k) |
| Handbook | setup, recovery, console pages, internals, troubleshooting | `Documents\VLA-Memory-POC-Handbook.md`; copy at `~/pm_console/docs/HANDBOOK.md` on wiprornd |
| Plan and runbook | the 9 Nov plan; every command of steps 1–3 | `Documents\VLA-Memory-POC-Plan.md`, `Documents\VLA-Memory-POC-Runbook.md` |

**How to read the numbers.**

- Times are simulated seconds: 30 simulation steps = 1 s. Wall-clock time is about 7–8 steps per second, so a 107 s episode takes about 7 minutes.
- "Instance 301" etc. are the benchmark's public test instances: the same house and goal, with the robot and objects placed differently.
- Run ids such as `20261119_165512_turning_on_radio_301` carry the container's clock (UTC). Host logs carry India time (UTC+5:30).
- **Reach** = the target is within 1.5 m of the robot base and inside the head-camera image. **Success** = the benchmark's own goal check. "Tooling and measures" defines every measure.
- Counts are given as "k of n" rather than percentages, because most cells hold 1–6 runs.

## Goal, constraints and how the scope changed

The question never changed: does persistent memory help a VLA in long-horizon tasks, and which kind? The scope narrowed twice, and one constraint, inference only, was lifted on 18 Nov.

**Original goal (23 Oct).** A short proof of concept comparing memory setups for long-horizon robot tasks in a simulated apartment: a VLA with no memory, memory inside the VLA, and separable memory that other models can use, with memory types such as Visual RAG, scene graphs and ConceptGraphs. It is an early step toward the user's own spatial-memory research.

**Constraints set then.**

- Inference only: download released models and run them; no training or fine-tuning.
- About two weeks from 23 Oct.
- One Ubuntu workstation: Intel Xeon 6515P and one NVIDIA RTX PRO 6000 Blackwell (96 GB). Docker containers preferred.
- Every claim backed by real runs; nothing assumed shown as done.

**The four points (18 Nov reset).**

1. Five different simple tasks with the object in view.
2. The same five out of view, without memory, to show that the VLA fails because it keeps nothing from earlier frames.
3. Two or three memory types on those tasks.
4. A comparative analysis: what persistent memory does for long-horizon tasks.

On 19 Nov two rules were added. Point 2 must not let the policy re-find objects by habit (from where training usually put them). Point 3 must use three-part tasks built from already-trained tasks: C#1 passing through a scene, C#2 an old action, C#3 a next action that needs what C#1 showed. No large vision-language models.

| Date | What happened | Why it mattered |
| --- | --- | --- |
| 23 Oct | POC defined, constraints set | two-week, inference-only budget |
| late Oct – 8 Nov | Platforms surveyed; BEHAVIOR-1K chosen; release v3.9.4 pinned for Isaac Sim 5.1 | Blackwell GPUs need Isaac Sim 5.1 or newer |
| 9 Nov | Simplified to one robot, one VLA trained for it, memory as the only variable; plan written | removed embodiment and policy as confounds |
| 10 Nov | Execution started: container, downloads; the user ran each command by hand from the runbook | hands-on setup by the user |
| 11 Nov | Isaac Sim 5.1 crashed with the 595 driver; moved to Isaac Sim 6.0.1 and BEHAVIOR main; first radio episode ran (grasped, never switched on) | platform settled; first sign that manipulation, not finding, limits success |
| 12–13 Nov | PM Console built (Streamlit app, sim server); handbook written; Comet checkpoint downloaded | runs became repeatable and scriptable |
| 13–15 Nov | Overnight pipelines: steps 3d, 4, 5, 6, 7 with both policies, then probe B | the eight-step POC done 15 Nov |
| 17 Nov | Comet language test; one success retracted (object carried over); phase-1 final report | phase 1 closed |
| 18 Nov | Scope reset to four points; Baseline Report started; point 1 done. Evening: training allowed, demos relayed, training script ready, training not started | narrower, task-level question |
| 19 Nov | Point 2 done (conditions A, B, C). Evening: memory types compared, point 3 designed and built, gate passed | memory test made habit-proof |
| 19–20 Nov | Point 3: 30 runs, 22:20 to 03:32 India time, after one restart for a scene-reset fix | point 3 data complete |
| 20 Nov | This report, with point 4 | snapshot of the whole POC |

**Who did what.** Until 13 Nov the user ran every command by hand, and I wrote the commands and checked outputs over ssh (from 11 Nov, at the user's request). From 14 Nov the user asked me to run the remaining steps, so I launched pipelines on wiprornd directly. Two things stayed with the user, because my automatic permission check blocks them: installing packages on wiprornd, and stopping the user's own running processes.

## Platform and setup

Everything runs on one workstation, wiprornd, inside one Docker container: BEHAVIOR-1K on Isaac Sim 6.0.1, the R1Pro robot, and π0.5 policies served over a WebSocket. Nothing runs on a real robot.

**Hardware and host.**

| Item | Value |
| --- | --- |
| Machine | wiprornd, reached by `ssh wiprornd` (user `jk`) |
| CPU / GPU | Intel Xeon 6515P; one NVIDIA RTX PRO 6000 Blackwell Server Edition, 96 GB |
| OS | Ubuntu 26.04, kernel 7.0.0-43; unattended upgrades on (they moved the driver once, on 11 Nov) |
| Driver | 595.99.02 (CUDA 13.2). Kept on purpose, because the user's Isaac Sim 6.1 data-collection work uses it too |
| Network | intermittent; Hugging Face's download servers are unreachable, so large downloads are relayed through this PC |
| Fallback considered | mig1: 4× RTX A6000 on driver 555, shared, no sudo; kept as a fallback only, never used |

**Container.** `persistent_memory` (image `persistent_memory:base`, OS layer only), with all GPUs, host network, 32 GB shared memory and restart unless stopped. Two volumes hold everything installed: `/home/jk/docker/persistent_memory/root_home` → `/root` and `.../workspace` → `/workspace`. So the container can be deleted and recreated without losing data; this was done once, on 11 Nov.

**Software environments.**

| Environment | Contents | Runs |
| --- | --- | --- |
| `behavior6` (conda) | Python 3.12, Isaac Sim 6.0.1, OmniGibson from BEHAVIOR-1K `main` at `b544acb`, torch 2.11 (CUDA 13), warp-lang 1.12.0, LeRobot 0.6.2, BDDL 3.7.0 | the simulator, the sim server, the official evaluator |
| openpi (uv) | Python 3.11, JAX 0.5.3, torch 2.7.1 + CUDA 12.8, transformers 5.5; wensi-ai/openpi branch `behavior` at `0cc8e35` | the radio checkpoint server; since 19 Nov also the CLIP service (the only torch here with Blackwell kernels) |
| openpi-comet (uv) | Team Comet's openpi fork, LeRobot 0.3.4, torch 2.7.1 + CUDA 12.6 | the Comet checkpoint server; the training script |
| console (venv, host) | Python 3.14, Streamlit 1.67 | the PM Console |
| `behavior` (conda) | the old Isaac Sim 5.1 environment | retired |

**Simulator and benchmark.** BEHAVIOR-1K is a household benchmark on the OmniGibson simulator. The POC uses its 2026 challenge task set: 100 tasks, each with 300 training instances that have human demonstrations and 20 public test instances (301–320). An instance fixes the robot's start pose and where the task's objects are. An episode ends at success or after 1.5 × the mean human demonstration length (radio: 3,225 steps, 107.5 s). Like the official evaluator, the sim server loads only the rooms a task needs (partial scene loading).

| Task (benchmark name) | House | Rooms loaded | Official command (abridged) |
| --- | --- | --- | --- |
| turning\_on\_radio | house\_double\_floor\_lower | living room, kitchen, corridor, garden | Turn on the radio receiver that's on the table in the living room. |
| hanging\_pictures | same | same | Pick up the poster from the kitchen countertop and hang it on one of the wall nails in the kitchen. |
| spraying\_for\_bugs | same | same | Pick up the pesticide atomizer in the garden and spray insectifuge to fully cover both potted plants in the garden. |
| attach\_a\_camera\_to\_a\_tripod | house\_double\_floor\_upper | three bedrooms | Attach the digital camera to the camera tripod in the bedroom. |
| clean\_a\_trumpet | same | three bedrooms | In the bedroom, pick up the scrub brush from the desk and scrub the cornet (trumpet) on the desk … |
| picking\_up\_trash, making\_microwave\_popcorn | various | task rooms | phase 1, step 4 only |

**Robot.** Galaxea R1Pro: a holonomic wheeled base, a torso, two arms with grippers, one head camera and two wrist cameras. The policy receives the three camera images, the robot's joint states and a text command, and returns one action per step. In BEHAVIOR's evaluator the robot is named `robot_r1`.

**Policies (VLAs).** Both are π0.5 models, a vision-language-action model from Physical Intelligence, released for this robot.

| Policy | Trained on | Weights | Used in |
| --- | --- | --- | --- |
| Radio checkpoint | human demos of `turning_on_radio` only (BEHAVIOR team's openpi release) | `/workspace/checkpoints/pi05_b1k_radio/` (12 GB) | phase 1 steps 3, 5, 6, 7, probe B |
| Comet pt50 (`pi05-b1kpt50-cs32`) | 50 BEHAVIOR-2025 tasks; 2nd place in the BEHAVIOR Challenge 2025 | `/workspace/checkpoints/openpi_comet/` (12 GB) | phase 1 steps 3e, 4, 6c, 7c; all of points 1–3 |

Comet predicts 32 steps at a time and replans (receding horizon). Its first query takes about 37 s while JAX compiles. Comet and the simulator together use about 48 GB of the 96 GB GPU.

**CLIP (point 3 only).** `openai/clip-vit-large-patch14`, 428M parameters, 1.71 GB. It scores how well a picture matches a text and generates nothing. It is served by `clip_server.py` on 127.0.0.1:8610 and takes about 33 ms per frame (six crops).

## Tooling and measures

All experiments run through tooling built for the POC, the PM Console, so that every episode follows the official evaluator's rules and leaves the same record. Memory sits in the sim server, between the simulator and the policy.

**The control loop.** On every simulator step the sim server sends the robot's three camera images, its joint state and the current command to the policy server over a WebSocket (port 8000), and gets back one action. Before the VLA takes over, a start controller in the sim server can script the robot: turn it, drive it, or move objects. That controller is also where the memory blocks live. At hand-over it asks the memory where the target is, turns or drives the robot there, and then lets the VLA act. **The VLA never sees the memory**: memory reaches it only by putting the robot in a better place.

&#91;embedded content: POC architecture · PC, host and container; the POC's own additions highlighted\]

The console starts each run over HTTP. On every step the live policy sends frames and the command to π0.5 and applies the action it returns. Before hand-over, the start controller moves the robot and asks the memory; the VLA never receives the memory itself.

| Part | Where | What it does |
| --- | --- | --- |
| PM Console | host, `~/pm_console`, Streamlit on 127.0.0.1:8510 | pages Live (load a scene, type a command, choose a start condition and memory, watch), System (container, servers, checkpoint switch), Benchmark (official evaluator), Results, Terminals (ttyd, 7681). Opened from the PC through an SSH tunnel |
| Sim server | container, `sim_server.py`, HTTP on 127.0.0.1:8600 | keeps one task loaded and runs each episode through BEHAVIOR's own `BatchedEvaluator` (same robot config, reset, timeout and goal check as the official evaluation). HTTP calls: status, frames, run, change command, stop, camera, preview, shutdown |
| Live policy | `pm_live.py` | the evaluator's WebSocket policy plus live frames, the typed command sent as `prompt`, a Stop button and the trace |
| Prompt-aware policy servers | `serve_b1k_prompt.py` (radio checkpoint), `serve_comet_prompt.py` (Comet) | pass the typed command to the model; the stock servers always use the task's own command |
| Start controller and memory blocks | `pm_memory.py` | start conditions, object relocation, the memory blocks, and a controller that rotates the robot in place or follows an A\* path on the house's free-space map at up to 0.5 m/s |
| Tracer | `pm_live.py` | every 10 steps (0.33 s): robot pose and room, what each hand holds, every task object's position, distance and whether it projects into the head camera |
| CLIP service | `clip_server.py`, 127.0.0.1:8610 | embeds frames and texts for the picture memory (point 3) |
| Plans and batches | `plans/*.json`, `tools/batch.py`, `tools/ctl.py`, `tools/pipeline*.sh` | a plan lists scenes and runs; the batch runs them one after another, loads each scene in turn, labels each run `<plan> · <key> · r<k>` and skips runs already done. Pipelines chain batches in a tmux session |
| Analysis | `tools/analyze.py`, `p2_compare.py`, `p3_compare.py` | tables per run and per condition from each run's record and trace |
| Evidence builder | `container/evidence.py` + a spec | per claim: keyframe sheets (third-person view on top; wrist and head cameras below), both videos and an index |

**What a run leaves behind.** Each run gets a folder `/workspace/outputs/console/runs/<run id>/` holding:

- `run.json`: the request, the start controller's events with step numbers, the memory's answer and statistics, and the evaluator's result;
- `trace.jsonl`: the trace sampled every 10 steps;
- `videos/`: the evaluator's robot-camera video (wrist cameras left, head camera right) and a third-person `follow.mp4`.

**Start conditions.**

| Code | Before the VLA takes over | Used in |
| --- | --- | --- |
| default | the benchmark's start pose | phase 1; point 1 diagnostics |
| face (V0) | turned in place to face the target (with `face_with`, the middle of several task objects) | phase 1, point 1, point 2 C |
| away (V1) | turned to face away; the target was never seen | phase 1 |
| look\_away (V2) | faces the target 2 s, then turns 180° with base commands | phase 1, point 2 A and B |
| tour (V3) | visits every loaded room and turns 360° in each (teleported between rooms), then starts in the room farthest from the target | phase 1 step 6 |
| probe B | as V2, but the target is moved to the floor beside the robot while it faces away | phase 1 |
| relocate | before the look, the target is moved to free floor at least 2 m from every place it had in the 300 training instances | point 2 B and C |
| chain | C#1 drive past the C#3 object, C#2 old task, C#3 next task | point 3 |

**Measures.**

| Measure | Definition |
| --- | --- |
| Success, q-score | the benchmark's goal check; q = share of goal conditions met |
| Reach | target within 1.5 m of the robot base and inside the head-camera image (projection only, no occlusion test), from the trace |
| Time to reach | seconds from a reference to Reach: hand-over, end of the turn, placement after the tour, or the C#3 command switch. Scripted turning and driving after the reference is counted |
| Reach within T s | runs that reach within T seconds of the reference |
| Picked up (grasp) | the object is held by the robot's grasp at any trace sample |
| Path, rooms, end room | from the trace |
| Memory answer error | distance on the floor between the memory's answer and the target's true position when asked |
| Memory size, write and query time | from the memory block's own statistics |

**Validity checks.**

- A run is flagged when its target starts more than 0.15 m from where it starts in other runs of the same instance (catches objects carried over from the previous episode). The check works only across several runs of an instance, so it is run over all batches of a policy together.
- Grasps are released before each episode (since 14 Nov). In point 3 the whole scene is also restored before each run (since 19 Nov, 22:20).
- Point 3 counts a run only if C#1 really showed the object and the object was out of view when the command switched.

## Phase 1: the eight-step POC (10–17 Nov)

Phase 1 showed, on the radio task, that π0.5 keeps no memory and that an external memory more than halves the search for an object out of view. It also showed that a memory which is not updated is worse than none. Full detail, claim sheets and run ids are in the [17 Nov final report](https://claude.ai/artifact/Q9zPpAnN5LYvGyoTNtSGfA); this section summarises it.

| Step | What it did | Result |
| --- | --- | --- |
| 1 | Container with persistent volumes | done 10 Nov, recreated 11 Nov |
| 2 | Simulator, robot assets, checkpoints | done 11 Nov on Isaac Sim 6.0.1; first launch 284 s (shader compile) |
| 3a–3c | Pipeline checks; first radio episode, instance 301 | ran 3,225 steps to the timeout: the robot drove to the radio, grasped it and held it, never switched it on |
| 3d | Radio checkpoint, 7 commands × 2, instance 301 | ignores the command: every run drove about 1.9 m to the radio, including "Go to the kitchen" and "do nothing" |
| 3e, 3f | Comet, same 7 commands × 2, instance 302; radio checkpoint control | Comet followed "Go to the kitchen" (2/2, 90% of the episode in the kitchen) but went to the radio for the other six, including "do nothing". The control ignored all three commands, so it is the policy, not the instance |
| 4 | Comet on five tasks, instances 301–302 | 8 of 9 clean episodes reached the object, 4 grasped it, 2 succeeded (picking up trash) |
| 5 | Start conditions without memory, radio checkpoint | V0 2.8 s, V1 12.8 s, V2 13.3 s: having seen the radio did not help |
| 6 | House tour, then start in the far room | with memory 10/10 reached the radio in about 11 s; without, 2 of 6 within 60 s |
| 7 | V2 with memory | radio checkpoint 13.3 → 5.2 s (oracle) and 5.3 s (object map); Comet 16.8 → 7.2 s |
| probe B | Radio moved while the robot faced away | oracle 3.9 s; no memory 12.8 s; stale object map 41.0 s, 0 of 5 within 20 s |
| 8 | Conclusions | final report, 17 Nov |

**Findings.**

- **No memory inside the VLA.** A robot that saw the radio 2 s before turning away needed as long to find it (13.3 s) as one that never saw it (12.8 s).
- **External memory more than halves the search,** with the same memory code for both policies (property A). Reach within 10 s rose from 0 of 10 runs to 21 of 21 with the radio checkpoint, and from 1 of 5 to 4 of 5 with Comet. Of the oracle's 5.2 s, 3.3 s is the scripted turn back.
- **Across rooms, memory decides whether the robot gets there.** After the tour, 10 of 10 runs with memory reached the radio in about 11 s. Without memory, 2 of 6 did within 60 s, and the radio checkpoint never got closer than 4.7 m. Most of the 11 s is the scripted drive; the VLA did the last metre.
- **A stale memory misleads (property B fails).** The object map had only ever seen the radio on the table, so after the move it sent the robot back to the empty table, 2.1 m off.
- **Success is limited by manipulation, not finding.** Full successes: 1 of 84 radio-checkpoint runs, 4 of 39 Comet runs (two radio, two trash).

**One success retracted.** On 14 Nov a Comet radio run on instance 302 switched the radio on in 916 steps and was reported as the first success. On 17 Nov the start-position check, run over all batches together, showed that it had started with the radio carried over from the previous episode, on the floor 1.1 m from the robot. It is excluded. A second carried-over run, in step 7, had already been excluded and re-run on 14 Nov, after grasp release before each episode was added.

**Memory properties A–E** (set out in the plan before any run).

| Property | Result in phase 1 |
| --- | --- |
| A. Planner-agnostic: one memory serves any policy | holds: no code changed between the two policies |
| B. Incremental update: follows changes in the world | fails for the object map as built (probe B) |
| C. View-consistent identity | not measured: needs a second radio |
| D. Explicit spatial relations | not measured: needs two radios to tell apart |
| E. Compact retrieval | holds: at most 284 bytes, one 3-D point per answer |

**Run counts.** 123 valid episodes: steps 3d (14), 3e (14), 3f (3), 4 (9 valid of 10), 5 (15), 7 and 7b (25 valid of 26), smoke tests (3), 6 (9), 7c (10), 6c (6) and probe B (15). Ten Comet episodes that failed at step 0 in the first step-4 attempt, and 8 console tests from 11–13 Nov, are not counted.

### Evidence (phase 1)

Each claim sheet shows one claim's runs as rows of keyframes: the third-person view on top, the robot cameras below (wrist cameras left; head camera right, which is what the VLA sees). Videos are the third-person view at real speed. Every file is in `Documents\VLA-Memory-POC-Evidence`, one folder per claim.

**C1 · The radio checkpoint ignores the command.** Instance 301, four of the seven commands, including "Go to the kitchen" and "do nothing": every run drives about 1.9 m to the radio.

&#91;image: Claim sheet C1: radio checkpoint, four commands, instance 301\]

&#91;image: Video C1: radio checkpoint told Go to the kitchen\]

*Video: told "Go to the kitchen" (run 20261113\_200951\_turning\_on\_radio\_301), the robot drives to the radio instead and reaches it at 6.7 s.*

**C13 · Comet follows "Go to the kitchen", not "do nothing".** Instance 302: Comet with the official command, "Go to the kitchen" and "do nothing", then the radio checkpoint with "Go to the kitchen" as the control.

&#91;image: Claim sheet C13: Comet language test and the radio-checkpoint control, instance 302\]

&#91;image: Video C13: Comet told Go to the kitchen\]

*Video: Comet told "Go to the kitchen" (20261117\_061256) leaves the living room at about 10 s and spends the episode in the kitchen.*

**C2 · The VLA keeps no memory.** Radio checkpoint, instance 302, no memory: V0 (facing the radio) reached it 3.3 s after hand-over, V1 (never saw it) 17.0 s, V2 (saw it 2 s, then turned away) 14.2 s.

&#91;image: Claim sheet C2: V0, V1 and V2 without memory, instance 302\]

&#91;image: Video C2: V2 without memory, instance 302\]

*Video: V2 without memory (20261113\_221129). The robot saw the radio, turned away, and searches as if it had never seen it.*

**C3 · Memory turns the robot back to the radio.** Instance 302, V2: no memory 14.2 s after hand-over; the oracle 1.8 s and the object map 2.5 s, after the controller has turned the robot back.

&#91;image: Claim sheet C3: V2 with no memory, the oracle and the object map, instance 302\]

&#91;image: Video C3: V2 with the object map, instance 302\]

*Video: V2 with the object map (20261113\_234653). The controller turns the robot to the remembered radio, then the VLA reaches it.*

**C10 · The same memory code works with a second policy (property A).** Comet, instance 302, V2: 29.5 s without memory, 9.2 s with the oracle.

&#91;image: Claim sheet C10: Comet V2 without memory and with the oracle, instance 302\]

**C8 · Across rooms, memory decides whether the robot gets there.** Radio checkpoint after the house tour, instance 302: without memory it never reaches the radio; with the object map it is driven back and reaches it 0.3 s after hand-over.

&#91;image: Claim sheet C8: tour without memory and with the object map, instance 302\]

&#91;image: Video C8: tour without memory\]

*Video: tour without memory (20261114\_084419). Placed in the room farthest from the radio, the robot never reaches it.*

&#91;image: Video C8: tour with the object map\]

*Video: tour with the object map (20261114\_085839). The controller drives the robot back along a planned path; the VLA does the last metre.*

**C9 · With memory, Comet completes the task after the tour.** Instance 303: without memory Comet reaches the radio only after 74 s and does not switch it on; with the object map it is driven back and switches it on (2,153 steps).

&#91;image: Claim sheet C9: Comet after the tour, without memory and with the object map, instance 303\]

&#91;image: Video C9: Comet, tour with the object map, success\]

*Video: 20261114\_081015. The tour, the drive back, and Comet switching the radio on.*

**C12 · A memory that is not updated misleads (probe B).** Instance 302: the radio is moved to the floor while the robot faces away. After hand-over the oracle reaches it in 1.1 s, no memory in 5.2 s, and the stale object map in 43.5 s, because it sends the robot back to the empty table.

&#91;image: Claim sheet C12: probe B with no memory, the oracle and the stale object map, instance 302\]

&#91;image: Video C12: probe B with the stale object map\]

*Video: stale object map (20261114\_160246). The robot is turned back to the empty table, then has to search the room.*

**C7 · Comet fails at grasping and placing, not at finding (step 4).** Trash pickup on instance 302 succeeds; hanging pictures reaches the poster but never grasps it; the camera-on-tripod run barely moves.

&#91;image: Claim sheet C7: Comet on three of the five step-4 tasks\]

&#91;image: Video C7: Comet picking up trash, success\]

*Video: 20261114\_060046\_picking\_up\_trash\_302, one of Comet's two step-4 successes.*

**C4, C5, C11 · Successes and a near miss.** C4: the radio checkpoint grasps the radio but never switches it on (20261113\_232522). C5: Comet switches the radio on from a clean start, told "I would like to listen to some music" (20261117\_055845, 3,198 steps). C11: the radio checkpoint's only success, V2 with the object map (20261114\_092751).

&#91;image: Claim sheet C4: the radio checkpoint grasps but does not switch on\]

&#91;image: Claim sheet C5: Comet's clean success and a grasp without switch-on\]

&#91;image: Claim sheet C11: the radio checkpoint's only success\]

**C6 · Why two runs are excluded.** Both started with the radio carried over from the previous episode.

&#91;image: Claim sheet C6: the two excluded runs\]

&#91;image: Video C6: the retracted Comet run\]

*Video: the retracted run 20261114\_041915\_turning\_on\_radio\_302. The episode starts with the radio already on the floor 1.1 m from the robot.*

## Point 1: five tasks with the object in view (18 Nov)

With every object in view, Comet reached the first object in 10 of 10 episodes and picked it up in 4 of 10, but completed none of the five tasks (goal score 0 in all ten). Its limit is manipulation, not finding objects, so points 2 to 4 measure reaching and picking up, not task success.

**Design.** One multi-task checkpoint (Comet pt50) runs five different tasks with their official commands, on public test instances 301 and 302. Before the first frame the robot is turned in place to face the task objects: the middle of them when they fit within 70°, else the first one. Batch `p1_inview`, pipeline 9, 18 Nov, 13:22–15:50 India time.

**Why these five tasks.** They are the five with the shortest demonstrations among Comet's 50 training tasks that need only one or two actions on one or two objects, and they cover different rooms and actions: press, pick and hang, pick and spray, pick and attach, pick a tool and scrub. Comet's tasks use only three houses. The single-floor house was left out because its shortest task needs six skills and six objects.

| Task | House · room | Action | First object reached (s after the first VLA step, 301 / 302) | Picked up | Success, goal score |
| --- | --- | --- | --- | --- | --- |
| Turning on the radio | lower · living room | press a button | 2/2 (4.3, 6.6) | stood at the radio, never pressed: 0/2 | 0/2, 0 |
| Hanging pictures | lower · kitchen | pick up, hang | 2/2 (at start, 11.6) | poster 2/2, lifted 0.33 and 0.55 m, never hung | 0/2, 0 |
| Spraying for bugs | lower · garden | pick up, spray | 2/2 (17.0, at start) | sprayer 2/2, lifted 0.65 and 0.36 m, never sprayed | 0/2, 0 |
| Attaching a camera to a tripod | upper · bedroom | pick up, attach | 2/2 (5.3, at start) | camera 0/2 | 0/2, 0 |
| Cleaning a trumpet | upper · bedroom | pick up a tool, scrub | 2/2 (3.3, 5.3) | brush 0/2 | 0/2, 0 |
| All |  |  | 10/10 within 17 s | 4/10 | 0/10 |

**Is the checkpoint broken, or the test instances unusual?** Two checks say neither fully explains it.

- From the unchanged benchmark start, in phase 1 step 4, Comet completed picking up trash 2 of 2 times. It also failed hanging pictures, the tripod and popcorn there (0 of 6), so the in-place turn alone is not the cause.
- On 18 Nov evening the same five tasks ran on training instances 2 and 3, which have human demonstrations (pipeline 10, benchmark start). Result: 0 of 10 successes (one spraying run reached goal score 0.5), first object picked up 6 of 10 (radio lifted rather than pressed, poster 2/2, sprayer 2/2, brush 1/2, tripod 0/2). The same failure on instances it was trained on means instance novelty is not the cause.

Two possible causes remain untested: Comet was trained on renders from Isaac Sim 5.1 while this setup uses 6.0.1, and manipulation is simply the weak part of this checkpoint. Evidence: claim sheets P1a–P1e and ten videos in the [Baseline Report](https://claude.ai/artifact/KynN2mHczShBSaYt3BS7DY); pack `Documents\VLA-Memory-POC-Baseline-Evidence` (387 MB).

### Evidence (point 1)

Claim sheets P1a–P1e show both instances of each task at the start, when the object was reached, at the grasp and at the end. Videos combine the third-person view (top) with the robot cameras (bottom), instance 301, at real speed. Files: `Documents\VLA-Memory-POC-Baseline-Evidence`.

**P1a · Radio: reached, never switched on.** Reached at 4.3 s and 6.6 s; the robot stands at the radio and never presses the button.

&#91;image: Claim sheet P1a: turning on the radio, both instances\]

&#91;image: Video P1a: turning on the radio, instance 301\]

*Video: 20261118\_100539\_turning\_on\_radio\_301, 107.5 s.*

**P1b · Poster: picked up, never hung.** Lifted 0.33 m and 0.55 m and held to the end.

&#91;image: Claim sheet P1b: hanging pictures, both instances\]

&#91;image: Video P1b: hanging pictures, instance 301\]

*Video: 20261118\_075557\_hanging\_pictures\_301, 119 s.*

**P1c · Sprayer: picked up, never sprayed.**

&#91;image: Claim sheet P1c: spraying for bugs, both instances\]

&#91;image: Video P1c: spraying for bugs, instance 301\]

*Video: 20261118\_091913\_spraying\_for\_bugs\_301, 324 s.*

**P1d · Camera: reached, never grasped.**

&#91;image: Claim sheet P1d: attaching a camera to a tripod, both instances\]

&#91;image: Video P1d: attaching a camera to a tripod, instance 301\]

*Video: 20261118\_081422\_attach\_a\_camera\_to\_a\_tripod\_301, 196 s.*

**P1e · Brush: the robot goes to the desk, never picks the brush up.**

&#91;image: Claim sheet P1e: cleaning a trumpet, both instances\]

&#91;image: Video P1e: cleaning a trumpet, instance 301\]

*Video: 20261118\_084218\_clean\_a\_trumpet\_301, 265 s.*

## Point 2: out of view, no memory (19 Nov)

Without memory, the VLA got back to an object it had lost sight of only where training usually put it: 8 of 10 runs in the usual place, 0 of 6 indoors once the object was moved somewhere new. With the moved object in view, it reached it in 5 of 6 runs, so what is missing in the moved case is memory.

**Design.** Each episode starts facing the task objects, as in point 1. A start script holds still for 2 s while the head camera sees the target, turns the robot 180° in place, and hands over to the VLA at step 165 (5.5 s). At hand-over the target is out of the head camera in every run, and there is no memory. Same checkpoint, official commands, instances 301 and 302, and point 1's first object as the target.

**Why the object is moved (the "no cheating" rule).** The test instances put each target 0–0.21 m from where it stood in the 300 training instances of its task. A policy that learned "the poster is on the countertop" finds it there without remembering anything it saw. So condition B moves the target, before the look, to free floor at least 2 m from all 300 training places. The spot is picked automatically: in the task's rooms, 1.5–4 m from the robot, in clear floor view of it, at least 1 m from other task objects. If no spot passes, the margin drops to 1 m and the run records that.

| Condition | Target | At hand-over | Runs |
| --- | --- | --- | --- |
| A: usual place, out of view | where the instance puts it | out of view | 10 |
| B: moved, out of view | moved to free floor ≥ 2 m from every training place | out of view | 10 (8 count) |
| C: moved, in view (control) | same spot as B, no turn | in view | 8 indoor (6 count) |

**Why the control.** The official commands name the usual place ("from the kitchen countertop", "on the table in the living room", "from the desk"), and Comet follows location words (phase 1, claim C13). If the robot goes to the moved object when it can see it, its failure in B is the missing memory, not the command.

| Task | A: usual place | B: moved, out of view | C: moved, in view |
| --- | --- | --- | --- |
| Radio | reached 2/2 (15.8 s, 20.5 s) | 0/2; moved 3.75 and 3.24 m; ended 3.23 and 2.01 m from it | 2/2 (4.0 s, 3.0 s), picked up on 302 |
| Poster | 2/2 (9.8 s, 19.8 s), picked up on 302 | 0/2; moved 2.85 and 6.66 m; ended 2.04 and 6.59 m from it | 1/2 (3.0 s); on 302 went to the countertop instead |
| Sprayer (garden) | 2/2 (40.2 s, 103.5 s), picked up 2/2 | 2/2 but late (160.2 s, 112.8 s) after 41 and 54 m of wandering; never picked up | not run |
| Camera (tripod) | 0/2; robot moved only 0.7 and 0.3 m | impossible: no free spot away from training | impossible |
| Brush | 2/2 (16.2 s, 15.5 s), picked up on 302 | 0/2; moved 3.25 and 2.73 m; ended 2.31 and 2.12 m from it | 2/2 (17.3 s, 9.0 s) |
| All | 8/10 | 0/6 indoor; 2/2 in the garden | 5/6 |

Times are seconds after hand-over (A and B at 5.5 s, C at the first step). No run completed its task; the goal score was 0 everywhere except one A spraying run (0.5).

**Where the robot went instead.** In all six indoor B runs it never got closer to the moved object than at hand-over (1.5–3.7 m) and ended 0.3–2.9 m farther from it. In all six it moved toward the object's usual place, by 0.2–2.9 m. Clearest case, poster 302: the robot crossed the kitchen from 3.74 m to 0.85 m of the countertop while the poster lay 3.67 m behind it.

**Caveats.**

- **Tripod:** the camera's 300 training placements cover the robot's bedroom, so no free spot there is even 1 m from all of them (18 cells pass, all against walls or furniture). B and C were impossible; six relocation attempts failed and are not counted. A also failed: the robot barely moved.
- **Garden:** training placements of the sprayer cover nearly all of it, so the moved sprayer is only 2.0–2.5 m from the nearest one. It was found, after the usual place and a long walk, and never picked up.
- **Command vs habit:** on poster 302 in C the robot walked to the countertop with the poster in view, so that one run does not separate memory from command. In B and C the radio was moved into the kitchen and poster 302 into the living room, rooms the commands do not name; C still reached the radio 2/2.
- **A code issue found on 19 Nov evening:** the relocation's "task rooms only" filter was inactive (see Problems). A check of all 14 point-2 relocations showed every one landed in a room the task loads, so no result changes.

Batches `p2_outview` (A), `p2_moved` and `p2_moved_tripod` (B), `p2_moved_inview` (C), pipelines 11–14. Evidence: claim sheets P2a–P2e, the tripod map and 11 videos in the Baseline Report; pack `Documents\VLA-Memory-POC-Point2-Evidence` (953 MB, 24 combined videos).

### Evidence (point 2)

Claim sheets P2a–P2e show condition A in rows 1–2, B in rows 3–4 and C in rows 5–6. Each row has keyframes at the start, while the robot sees the target, at hand-over, when it reaches the target or the target's usual place, and at the end; a red line gives where the target was moved. Videos: third-person view on top, robot cameras below. Files: `Documents\VLA-Memory-POC-Point2-Evidence`.

**P2a · Radio: found in its usual place, never once moved, reached when in view.**

&#91;image: Claim sheet P2a: turning on the radio, conditions A, B and C\]

&#91;image: Video A: radio in its usual place, out of view, instance 301\]

*Video A, usual place (20261119\_073142): the robot turns back to the table and reaches the radio 15.8 s after hand-over.*

&#91;image: Video B: radio moved, out of view, instance 301\]

*Video B, radio moved 3.75 m (20261119\_074651): it is never seen again; the robot stays near the table.*

&#91;image: Video C: radio moved, in view, instance 301\]

*Video C, the same moved radio in view (20261119\_115503): the robot walks to it and reaches it at 4.0 s.*

**P2b · Poster: the robot goes to the countertop, where training put the poster.**

&#91;image: Claim sheet P2b: hanging pictures, conditions A, B and C\]

&#91;image: Video B: poster moved, out of view, instance 302\]

*Video B, poster moved 6.66 m (20261119\_085852): the robot crosses the kitchen to the countertop while the poster lies 3.67 m behind it.*

&#91;image: Video C: poster moved, in view, instance 302\]

*Video C, the one control miss (20261119\_114409): the poster is in view 3.67 m away, and the robot still walks to the countertop.*

**P2c · Sprayer in the garden: found only after a long search.**

&#91;image: Claim sheet P2c: spraying for bugs, conditions A and B\]

&#91;image: Video B: sprayer moved, out of view, instance 301\]

*Video B (20261119\_080426): the robot first goes to the sprayer's usual place, walks 41 m and finds the moved sprayer at 160 s; it never picks it up.*

**P2d · Camera: not found even in its usual place, and it cannot be moved.**

&#91;image: Claim sheet P2d: attaching a camera to a tripod, condition A\]

&#91;image: Map: the camera's 300 training placements cover the robot's bedroom\]

*Map: the bedroom from above. The camera's 300 training placements (blue) cover the free floor, so no relocation spot exists.*

&#91;image: Video A: camera in its usual place, out of view, instance 301\]

*Video A (20261119\_055434): the robot turns away from the camera and barely moves for the rest of the episode.*

**P2e · Brush: found on the desk, never once moved, reached when in view.**

&#91;image: Claim sheet P2e: cleaning a trumpet, conditions A, B and C\]

&#91;image: Video B: brush moved, out of view, instance 301\]

*Video B, brush moved 3.25 m (20261119\_093341): the robot goes to the desk and never sees the brush again.*

## Point 3: memory on three-part tasks (19–20 Nov)

When the memory knew where the C#3 object was, the robot reached it in every valid run: object map 5 of 5, true position 5 of 5. Without memory, and with CLIP picture memory, it never did (0 of 5 each); room memory got there 3 of 5, one of them only because its route passed the object. The VLA itself did little after the command switched: it never grasped the C#3 object, and without memory it did not head for it at all.

&#91;embedded content: plan p3\_main, 19–20 Nov 2026 · 25 valid runs: radio → poster on instances 301–302, poster → radio on 301–303\]

### Design (fixed before the runs)

One episode is one continuous run in one house, in three parts:

1. **C#1, passing through.** A scripted controller drives the robot from its benchmark start to a pass point 1.5–2.5 m from the C#3 object, turns it to face the object, holds 2 s, and drives back to the start facing the C#2 object. The VLA is not involved. C#1 took 7.5–14.7 s.
2. **C#2, old action.** Comet does the loaded task with its official command until it grasps that task's object (plus 1 s) or 45 s pass. C#2 took 9.7–45 s.
3. **C#3, next action.** The robot lets go of what it holds, and the command switches to the other task's official command. The memory is asked where that task's object is, the controller drives there, and Comet takes over. The episode ends 90 s after the switch.

**No habit shortcut.** The C#3 object is moved before C#1 to free floor at least 2 m from every place it had in its task's 300 training instances. It goes in its usual room first, then a fallback room, at least 3 m from the C#2 area, out of floor view of it where possible. The pass point must be indoors and reachable by a route at most 2.5× the straight line plus 3 m. Only C#1 shows where the object is.

**Chains.** Both in the lower house, whose three tasks load the same four rooms. Each task's scene holds only its own objects, so the C#3 object is added from the other task's scene (same model and size).

- Radio → poster: scene `turning_on_radio` plus the poster. C#2 at the radio in the living room; C#3 poster moved in the kitchen.
- Poster → radio: scene `hanging_pictures` plus the radio. C#2 at the poster in the kitchen; C#3 radio moved to the living room (301), corridor (302) or kitchen (303, the fallback).

**Memory blocks.** Each sees the whole episode and does not know which object C#3 will ask for.

|  | Block | Writes | Answers at the switch | Robot is driven to |
| --- | --- | --- | --- | --- |
| M0 | none | – | – | stays where C#2 ended |
| M3 | picture memory | every 10 steps: the head-camera frame's CLIP embeddings (full image, centre, four quarters) and the camera's position and heading | the frame best matching "a photo of a poster" (or radio), scored against four neutral texts | where that frame was taken, same heading |
| M6 | room memory | every 10 steps: the room of each object the camera sees (simulator labels, ray-cast visibility) | the room of the last-seen object of the asked category | that room's central free spot |
| M4 | object map | every 10 steps: position, first and last sighting of every object the camera sees | the last-seen object of the asked category | 1 m from it, facing it |
| M5 | oracle | – | true position | 1 m from it, facing it |

**Measures and rules.** Main measure: the C#3 object reached within 60 s of the switch. A run counts only if C#1 really showed the object and it was out of view at the switch. Pre-set rule: a block helps if it beats no memory by at least 3 of 6 runs; the result counts against memory if no memory does about as well.

**Gate (19 Nov, radio → poster, 301).** Oracle: reached 7.6 s after the switch. No memory: never closer than 4.5 m. The chain worked, so the full plan started.

### Results (25 valid runs)

| Memory | Answer error (median; range) | Time to reach (median) | Closest approach (median) | C#3 grasp | Memory size | Query time | Write time per episode |
| --- | --- | --- | --- | --- | --- | --- | --- |
| M0 none | – | never | 4.7 m | 0/5 | – | – | – |
| M3 picture | 5.0 m (3.0–7.3) | never | 4.1 m | 0/5 | 3.2–4.2 MB | 0.02–0.13 s | 10–13 s |
| M6 room | 1.3 m (0.2–4.6) | 5.7 s (3 runs within 60 s) | 0.6 m | 0/5 | 0.7–0.9 KB | 0.01 s | 47–59 s |
| M4 object map | 0 m in all 5 | 7.0 s | 0.95 m | 0/5 | 5.2–6.9 KB | under 1 ms | 43–54 s |
| M5 oracle | 0 m | 7.7 s | 0.84 m | 0/5 | – | – | – |

Times to reach run from the switch and include the controller's drive (3.3–12.7 s for M4 and M5; longer in poster → radio, where the radio was 6.7–7.1 m away in two instances). Write time is wall-clock time spent on memory writes over a roughly 10-minute episode. For M4 and M6 it is the visibility check over about 260 scene objects.

**What the VLA did after the switch.** This qualifies the result.

- After hand-over Comet moved a median 0.3 m. It moved more than 1 m in 4 of 25 runs.
- Without memory it did not search: its distance to the C#3 object's usual place stayed 5.2–6.6 m. In point 2 it did walk to the usual place, but there the task was loaded from the start.
- In 8 of 25 runs it picked up the C#2 object after being told the new task.
- One clear exception: room memory, poster → radio, 303. Dropped at the kitchen's centre 4.9 m from the radio, Comet walked 5.9 m and reached it at 63 s.

So in point 3 the memory and its controller did the getting there; the VLA did at most the last approach (closest 0.3–1.0 m) and grasped nothing. This matches phase 1's language test: in a scene, Comet falls back to the task it was trained on there.

**Why the picture memory failed.** Every answer pointed at a C#2 frame. In an offline re-scoring of one run, the ten C#1 frames with the poster in view ranked 126–156 of 159 for "a photo of a poster": the poster was small and seen from 2.7 m. The top frame showed the red radio in the gripper over a grey rug.

&#91;image: Left: a C#1 frame with the poster in view. Right: CLIP's top match for a poster\]

*Left: a C#1 head-camera frame with the moved poster in view, ranked 136 of 159 for "a photo of a poster". Right: CLIP's top match, a C#2 frame of the red radio in the gripper over a grey rug (run 20261119\_165512, re-scored offline).*

**Why room memory is in between.** It works when the room's centre is near the object: 0.2 and 0.8 m from the poster in the small kitchen (reached at 5.7 and 7.7 s). In poster → radio 301 the living room's centre was 2.4 m from the radio, but the drive there ran past the radio: it counted as reached at 3.3 s, and the robot then pushed it along for 2 m. That success came from the route, not from the room answer. In the corridor (302) the centre was 1.3 m away and the radio never came into view. In the 303 kitchen it was 4.6 m away, the case where Comet walked.

**Why instance 303 of radio → poster does not count.** The poster went to the same spot as in 301 but settled behind a dining chair. The ray check failed for all 60 look steps, and the object map never stored it, so C#1 did not show it. Under the rule set before the runs, its five runs are excluded. The oracle still reached it; object and room memory had no answer.

&#91;image: Instance 303: the poster behind a dining chair during the C#1 look\]

*Instance 303 during the C#1 look (run 20261119\_191638). Left, the head camera: no poster visible. Right, the third-person view: the poster stands behind the chair.*

**C#2 does not depend on memory.** The memory is passive during C#2. The C#2 grasp happened in 10 of 25 valid runs, spread over all five conditions, which is Comet's own variation from identical starts.

Runs: plan `p3_main` (30 runs, 22:20 to 03:32 India time), gate `p3_gate` (2). Plan `p3_chain` (3 finished, 1 stopped) was discarded after the scene-reset problem (see Problems). Analysis: `tools/p3_compare.py` → `logs/p3_compare.json` on wiprornd. The point-3 evidence pack (6 claims, 30 runs, 930 MB) is in \`Documents\\VLA-Memory-POC-Point3-Evidence\`; the evidence below comes from it.

### Evidence (point 3)

Claim sheets P3a–P3f show the five memory conditions of one chain and instance as rows: M0, M3, M6, M4, M5. Keyframes: the C#1 sighting, the start of C#2, the C#3 command switch, the reach, the memory's hand-over and the end; a red line gives where the C#3 object was moved. Videos: third-person view on top, robot cameras below, real speed.

**P3a · Radio → poster, 301: only memories that localise the poster get the robot to it.** M0 stays in the living room (closest 4.7 m); M3's answer is 5.0 m off; M6, M4 and M5 reach the poster at 7.7, 7.0 and 7.7 s.

&#91;image: Claim sheet P3a: radio then poster, instance 301, five memory conditions\]

&#91;image: Video P3a: no memory\]

*Video M0 (20261119\_172744): after the switch to the poster command, the robot stays at the radio and picks it up again.*

&#91;image: Video P3a: picture memory\]

*Video M3 (20261119\_165512): CLIP's best match for "a photo of a poster" is a frame taken at the radio, so the memory sends the robot where it already is.*

&#91;image: Video P3a: object map\]

*Video M4 (20261119\_170553): the object map's answer is exact. The controller drives the robot into the kitchen to 1 m from the poster; the VLA takes over but does not grasp it.*

**P3b · Radio → poster, 302: the same pattern on a second instance.**

&#91;image: Claim sheet P3b: radio then poster, instance 302\]

**P3c · Poster → radio, 301: room memory reaches the radio only because its route passes it.**

&#91;image: Claim sheet P3c: poster then radio, instance 301\]

&#91;image: Video P3c: room memory, the route passes the radio\]

*Video M6 (20261119\_194544): on the way to the living room's centre the robot passes the radio and pushes it along for 2 m.*

**P3d · Poster → radio, 302: the radio in the corridor is found only with an exact position.**

&#91;image: Claim sheet P3d: poster then radio, instance 302\]

&#91;image: Video P3d: object map, radio in the corridor\]

*Video M4 (20261119\_202845): the controller drives about 7 m from the kitchen into the corridor; the radio is reached 12.5 s after the switch.*

&#91;image: Video P3d: room memory, radio in the corridor\]

*Video M6 (20261119\_203836): driven to the corridor's centre, 1.3 m from the radio, the robot stays there and the radio never comes into the head camera.*

**P3e · Poster → radio, 303: the one run where the VLA searched by itself.**

&#91;image: Claim sheet P3e: poster then radio, instance 303\]

&#91;image: Video P3e: room memory, the VLA walks to the radio\]

*Video M6 (20261119\_212907): dropped at the kitchen's centre 4.9 m from the radio, Comet walks 5.9 m and reaches it 63.3 s after the switch.*

**P3f · Radio → poster, 303 (excluded): C#1 never showed the poster.**

&#91;image: Claim sheet P3f: radio then poster, instance 303, excluded\]

## Point 4: comparative analysis

Across all runs, the VLA needed memory as soon as an object left its view and could not be found by habit. The memory that worked was the one that stores where objects are, not the one that stores the most. Today that memory helps only through a controller that moves the robot: the VLA neither reads it nor finishes the tasks.

### 1. The same question across all experiments

One line of evidence runs from "in view" to "out of view with memory". Each row is a different experiment, so the measures differ slightly; the direction is the same in every one.

| Situation | Experiment | Got to the object | Notes |
| --- | --- | --- | --- |
| Object in view | point 1 (5 tasks) | 10 of 10 | picked up 4 of 10, completed 0 |
| Moved object, in view | point 2 C | 5 of 6 | the one miss walked to the usual place |
| Out of view, usual place | point 2 A | 8 of 10 | found by habit, not by memory |
| Out of view, never seen | phase 1 V1, radio | as slow as V2 below | V1 12.8 s vs V2 13.3 s |
| Out of view, seen 2 s earlier, no memory | phase 1 V2, radio | 0 of 10 within 10 s | seeing it first did not help |
| Out of view, moved, no memory | point 2 B | 0 of 6 indoors | went to the usual place instead |
| Three-part task, no memory | point 3 M0 | 0 of 5 | did not even head for the usual place |
| Out of view, seen, with memory | phase 1 V2 + M5 or M4 | 21 of 21 within 10 s | radio checkpoint |
| Another room, with memory | phase 1 tour | 10 of 10 (2 of 6 without) | about 11 s, mostly scripted drive |
| Three-part task, with memory | point 3 M4 and M5 | 5 of 5 each | M6 3 of 5; M3 0 of 5 |
| Moved after the memory saw it | phase 1 probe B, stale M4 | 0 of 5 within 20 s | slower than no memory (41 s vs 12.8 s) |

### 2. The memory types side by side

|  | M0 none | M3 picture memory | M6 room memory | M4 object map | M5 oracle |
| --- | --- | --- | --- | --- | --- |
| Stores | nothing | frames + camera pose, CLIP embeddings | object → room | object → 3-D position, last seen | nothing (reads the simulator) |
| Perception assumed | – | real camera images | perfect labels + ray-cast visibility | perfect labels + ray-cast visibility | perfect |
| Answer | – | a viewpoint (where a picture was taken) | a room's centre | a point | a point |
| Answer error, point 3 | – | 3.0–7.3 m | 0.2–4.6 m | 0 m | 0 m |
| Reached, point 3 | 0/5 | 0/5 | 3/5, one by the route passing it (4/5 by 90 s) | 5/5 | 5/5 |
| Size per episode | – | 3.2–4.2 MB | under 1 KB | 5–7 KB | – |
| Query time | – | 0.02–0.13 s | 0.01 s | under 1 ms | – |
| A moved object (property B) | – | not tested; keeps old frames, so likely stale | not tested; same writes as M4 | fails (probe B) | always current |
| Same code for two policies (property A) | – | not tested | not tested | holds (phase 1) | holds (phase 1) |
| Realistic today? | yes | yes | no: needs a detector | no: needs a detector | no |

### 3. What the comparison says

1. **Memory is needed, and habit hides the need.** Out of view, the VLA re-found objects only where training put them (8 of 10) and never once they were moved (0 of 6, then 0 of 5). A test that leaves objects in their usual places would wrongly conclude that the VLA copes without memory.
2. **Localisation beats richness.** The picture memory stored the most (about 4 MB) and found nothing. The object map stored 6 KB and found everything. What matters is whether the answer is a place the robot can be taken to.
3. **How precise depends on what the VLA does after hand-over.** Comet did at most the last metre (closest 0.3–1.0 m). So a room-level answer worked when the room's centre was within about a metre of the object (2 runs); a third success came from the route passing the object. With a VLA that searches by itself, a coarse room memory might be enough: the one run where Comet walked 5.9 m came from room memory.
4. **Perception is the open problem.** Both memories that worked used the simulator's perfect labels. The only memory working from real camera images failed because CLIP could not pick out a small, distant poster among 159 frames. A realistic object map needs an object detector in front of it, which is the next test.
5. **A memory must be kept current.** When the radio moved behind the robot, the object map's old answer cost more time than no memory (41 s against 12.8 s). Every block here only adds sightings; none notices that an object has gone.
6. **The channel is the gap.** Memory reached the VLA only by moving the robot. After a command switch the VLA mostly stayed put (median 0.3 m) and in 8 of 25 runs went back to the old task's object. A long-horizon system needs a task-level layer that turns memory into goals, and perhaps a VLA trained to take a goal or a memory as input.
7. **Manipulation, not memory, limits completion.** Completed tasks: 0 of 10 in view (point 1), 0 in points 2–3, and 5 in phase 1 (1 of 84 radio-checkpoint runs, 4 of 39 Comet runs). Memory can get the VLA to the object; it cannot make it finish the task.

### 4. Conclusion for the persistent-memory design

What the evidence points to, stated as design requirements:

- **Object-centric and metric at its core:** store objects with positions, as M4 did; that is the only form that answered correctly every time.
- **A coarse room layer on top:** cheap (under 1 KB) and enough when the VLA can search a room.
- **Fed by real perception:** an open-vocabulary detector on the camera frames, with images kept as evidence for re-identification rather than as the retrieval key.
- **Updated, not only appended:** record when a remembered place is seen empty, and re-ask the memory.
- **Delivered through a planner today:** the controller hand-over works; a VLA that consumes the memory directly needs training.

**What the evidence does not show yet.**

- That memory improves task completion.
- How M3 and M6 behave when objects move.
- Any memory working from real perception.
- Results beyond 2 houses, 7 tasks and 1–5 runs per cell. No statistical tests were run; the effects shown are large (0 of 5 against 5 of 5), the small ones are not resolved.

## Decisions and trade-offs

Most decisions traded realism or breadth for a clean, finishable test within the time. Each row says what was given up.

### Platform

| Decision | Chosen over | Why | What it cost |
| --- | --- | --- | --- |
| BEHAVIOR-1K with the R1Pro robot | Isaac Lab-Arena with G1 and GR00T N1.6 (kept as fallback) | released π0.5 checkpoints for this robot on household tasks, an official evaluator, and task instances that record where objects were in training | slow scenes (3–5 min to load, 7–8 steps/s); tight coupling to Isaac Sim versions |
| One robot, one VLA, memory as the only variable (9 Nov) | comparing robots and VLAs | isolates the memory effect | memories built into other VLAs could only be cited from papers, not run |
| Keep NVIDIA driver 595 | install the R580 driver that Isaac Sim 5.1 needed | the user's Isaac Sim 6.1 data-collection work uses 595; NVIDIA validated Isaac Sim 6.0.1 on 595 | moved off the simulator version the checkpoints were trained on (render shift, untested) |
| Isaac Sim 6.0.1 | 6.1, which the user asked for | BEHAVIOR main accepts only 6.0.1 | two Isaac Sim versions on one machine |
| BEHAVIOR main pinned at `b544acb` | release v3.9.4 | only main runs on Isaac Sim 6 | Comet's code expects older BEHAVIOR modules: stand-in modules written for the server and the training loader |
| The user ran every command by hand until 13 Nov | full automation from the start | the user wanted hands-on experience of the setup | slower start; then automated pipelines from 14 Nov to finish in time |
| A Streamlit console over an SSH tunnel; live view from frames and videos | WebRTC streaming of Isaac Sim | streaming stayed black after two attempts; the console works and records everything | no smooth live view; videos are watched afterwards |
| Load only the task's rooms, as the official evaluator does | the full house | identical to the benchmark's conditions | the tripod object could not be moved; some routes run through few rooms |

### Experiment design

| Decision | Chosen over | Why | What it cost |
| --- | --- | --- | --- |
| Memory reaches the VLA through a scripted controller | memory as a VLA input | inference only; no released VLA takes a memory | measured gains are memory plus controller, not a VLA using memory |
| Simulator labels for the object map and room memory | an object detector | test how the memory is organised, not detection errors | their results are a best case |
| Oracle first, as a gate and a ceiling | comparing memories directly | proves the hand-over channel works before any comparison | one extra condition per experiment |
| Reach as the main measure | task success | success was 0 in view (point 1), so it cannot separate conditions | the claim is "gets to the object", not "does the task" |
| Move the object at least 2 m from all 300 training places (point 2 B, point 3) | leaving objects where the instance puts them | the user's rule: no re-finding by habit | objects on the floor, outside the manipulation training data; impossible for the tripod, weak in the garden |
| A moved-but-in-view control (point 2 C) | trusting B alone | the official commands name the usual place; C shows the robot does go to a moved object it can see | 8 extra runs |
| Official commands kept | location-free commands | comparable with the benchmark and training | one control run (poster 302) stays ambiguous; a location-free re-run was not done |
| Comet for all of phase 2 | task specialists | one checkpoint for every task and both chains; the radio checkpoint ignores language anyway | Comet falls back to a scene's own task |
| Five tasks with the shortest demos and 1–2 actions | harder or more varied tasks | the best chance of success in view | two houses only |
| Point 3 chains in the lower house, the other task's object added | mixing houses | both tasks need one scene; the upper house's tripod object could not be moved | two chains, one house |
| M4, M3 (CLIP) and M6 for point 3 | keyframes + VLM, captioned Visual RAG, ConceptGraphs with an LLM, trained memories (19 types compared) | no large VLMs (the user's rule); simple; three different ways of storing | no realistic object detector tested yet |
| CLIP ViT-L/14 | ViT-B/16 | weights available as safetensors on the official main branch; better on small objects | 1.7 GB download relayed through the PC |
| CLIP scoring rule fixed before any run (neutral texts subtracted) | tuning after seeing results | avoids fitting the rule to the outcome | the rule stayed even though it failed |
| Picture memory answers a viewpoint | locating the object in the image | no depth or box needed | relies on the VLA to find the object from there |
| Object and room memory asked by category | asked by the object's exact identity | a realistic query ("a poster") | could pick the wrong object of that category (did not happen) |
| Release held objects at the C#3 switch | keep them in hand | Comet's training never starts a task holding something | the dropped C#2 object stays close, and Comet picked it up in 8 of 25 runs |
| Hidden spot nearest to the C#2 area | the farthest spot | the fairest case for no memory: a short search | spots repeat across instances (301 and 303 used the same one) |
| Validity rules set before the runs | judging runs afterwards | no cherry-picking | instance 303 of radio → poster lost all 5 runs |
| Re-run all of point 3 after the scene-reset fix | keeping 3 finished runs | equal conditions for every memory | about 1.2 h of runs discarded |
| 2–3 instances, one run each | repeats and more instances | time: a chain run takes about 10 min | 1–5 runs per cell; counts, no statistics |
| Training allowed (18 Nov), but replay a human demo first | 15k-step fine-tuning at once | if demos do not replay in Isaac Sim 6.0.1, fine-tuning on them cannot work | training not started yet |

**The time budget shaped the scope.** The original plan was about two weeks from 23 Oct; set-up alone took 10–12 Nov, mainly the driver and simulator change. What was cut or deferred:

- the vision-language-model memories (keyframes + VLM, captioned Visual RAG);
- probes for identity and relations (properties C and D);
- a driven, full-house tour;
- location-free commands;
- repeats per condition;
- the point-3 evidence pack (built afterwards, on 20 Nov);
- training.

## Problems hit and how each was fixed

Thirty problems came up; none is open. Four changed results or the run plan: the driver crash (simulator version), the carried-over object (one success retracted), the inactive room filter, and the pushed chair (point 3 re-run).

| Date | Problem | Fix |
| --- | --- | --- |
| 20 Nov | In instance 303 of radio → poster the moved poster settled behind a dining chair and was never visible in C#1 | excluded by the validity rule set before the runs (5 runs) |
| 19 Nov | The poster's reported "moved" distance read 0 m after the first run, because each relocation measures from the object's last pose | the analysis uses the distance from the 300 training placements instead |
| 19 Nov | Saving the clean scene as a dictionary returned raw tensors, not JSON | store the simulator's own JSON string |
| 19 Nov | A chair the robot pushed in one run stayed for the next and covered the poster: the evaluator restores only the task objects | snapshot the whole scene after loading, restore it before every run; plan `p3_chain` discarded, re-run as `p3_main` |
| 19 Nov | The CLIP service never started: its own start check matched its command line | start it with a detached `docker exec`; `tools/clip_start.sh` |
| 19 Nov | CLIP failed with "no kernel image": the Comet environment's torch (CUDA 12.6) has no Blackwell kernels | run CLIP in the openpi environment (torch with CUDA 12.8, sm\_120) |
| 19 Nov | Hugging Face's download servers unreachable from wiprornd; CLIP ViT-B/16 has no safetensors on its main branch | ViT-L/14 downloaded on this PC, checksum matched, copied over |
| 19 Nov | OmniGibson's `world_to_map` flips the point order of a batch, so the relocation's room filter was scrambled: the poster went into the garden and the C#1 route ran 21 m round a closed glass garden door, where the robot stuck | index rooms by hand, keep the pass point indoors, reject routes longer than 2.5× straight + 3 m; all 14 point-2 relocations checked and unaffected |
| 19 Nov | Point-2 control sheets said "moved before the look" though C has no look | fixed the evidence builder's text |
| 19 Nov | The tripod camera could not be moved: its 300 training placements cover the robot's bedroom; spots ≥ 1 m away exist only in other bedrooms, behind walls | reported as impossible; tripod B and C excluded (6 attempts) |
| 18 Nov | Comet's training loader needs BEHAVIOR v3.7.2 utilities that main removed | a shim folder of v3.7.2 files and stubs; the data check passed (4,044,581 samples) |
| 18 Nov | wiprornd reached only Google domains; the human demos are on Hugging Face | relayed through this PC: 22 GB at 1–4 MB/s in, about 100 MB/s out; 20 GB verified on wiprornd |
| 17 Nov | A reported Comet success (916 steps) had started with the radio carried over from the previous episode; a per-batch check cannot flag an instance with one run | start-position check run across all batches of a policy; the success retracted |
| 15 Nov | Run records showed a stale policy label (the sim server's start-up label) | record the policy server that answered each run |
| 14 Nov | A tour test planned a path to a point on the table and found none | plan between the nearest free cells |
| 14 Nov | An object held at the end of an episode followed the hand through the reset (a radio in mid-air) | release every grasp before each episode; the run was repeated |
| 14 Nov | Comet's server imports a BEHAVIOR module that main renamed; its environment has no OmniGibson | a stand-in module plus the openpi BEHAVIOR fork's WebSocket server |
| 14 Nov | The first Comet attempt failed at step 0 in 10 episodes (input batch dimension) | fixed in the Comet server; those 10 not counted |
| 13 Nov | The Claude Doc handbook stayed empty when the docs connector dropped | handbook kept as a Markdown file |
| 13 Nov | Choosing a command preset looked like changing the task; the scene stayed loaded | the Live page split into 1 · Scene and 2 · Command |
| 12 Nov | The stock policy server ignores the typed command | prompt-aware policy servers |
| 11 Nov | `og.shutdown()` ends the process, so the evaluator's summary never prints | read success from the result files |
| 11 Nov | WebRTC streaming stayed black in two attempts | parked; the console shows frames and records videos |
| 11 Nov | Partial scene loading is hard-coded in the evaluator | a full-scene option in the sim server |
| 11 Nov | The openpi robot config reads `robot::…` keys; the evaluator names the robot `robot_r1` | `poc.py` registers a renamed copy, `poc/R1Pro` |
| 11 Nov | `torch-cluster` failed to build during setup | ignored: optional, point clouds only |
| 11 Nov | Isaac Sim 5.1 crashed at the first rendered frame with driver 595 on Blackwell (a known 595-branch issue); unattended upgrades had moved the driver overnight | moved to Isaac Sim 6.0.1 and BEHAVIOR main; driver kept |
| 10 Nov | Internet at first reached only pypi.nvidia.com and drive.google.com | worked later the same day; connectivity is intermittent, so check DNS first |
| early Nov | `import omnigibson` failed: warp is only in an optional extra | `pip install warp-lang==1.12.0` (still needed on main) |
| early Nov | Release v3.9.3-post2 failed to install (its LeRobot branch needs Python 3.12, Isaac Sim 5.1 needs 3.11) | used v3.9.4, which pins LeRobot |

**Pictures behind two of the problems.**

&#91;image: Lower house map: the first gate run's garden detour\]

*The lower house from above, y up (rooms: 3 corridor, 5 garden, 6 kitchen, 7 living room; red = free floor the robot cannot reach from the start). With the room filter scrambled, the poster ("poster\_new") and the pass point were placed in the garden, and the C#1 route ran out through the kitchen's garden door, where the robot stuck ("stuck"). Blue dots: the poster's 300 training placements on the counter.*

&#91;image: Pushed chair: the head camera in two consecutive runs\]

&#91;image: Pushed chair: the third-person view in two consecutive runs\]

*The same C#1 moment in two consecutive runs before the scene-reset fix (left 20261119\_162702, right 20261119\_163712). A chair pushed in the earlier run covers the poster in the later one.*

## Limitations

The findings hold for this setting only: simulation, two houses, seven tasks, two π0.5 checkpoints, and memory delivered by a scripted controller. The main effects are large (0 of 5 against 5 of 5), but small effects cannot be resolved with these sample sizes.

**Scope and samples.**

- Simulation only, one workstation, no real robot.
- 1–5 runs per cell: two instances per task in points 1–2, two or three per chain in point 3, one run per condition. No statistical tests.
- Comet varies between runs from identical starts: the C#2 grasp happened in 10 of 25 valid point-3 runs, spread over every condition. Single runs can therefore differ by chance.
- Point 3 covers one house, two chains and four rooms. Radio → poster has only two valid instances, because instance 303 was excluded, and instances 301 and 303 used the same poster spot.

**What is and is not measured.**

- Task success was 0 throughout phase 2, so every claim is about getting to the object, not doing the task.
- Memory reaches the VLA only through a controller that turns or drives the robot. In point 3 that controller did most of the work, so the gains are those of memory plus controller.
- The VLA never saw the memory; nothing here tests a VLA that reads one.

**Measurement.**

- Reach counts an object inside the head-camera image even if something hides it (no occlusion test), at 1.5 m, sampled every 10 steps (0.33 s).
- The object map and room memory use the simulator's object labels and a ray cast to the object's centre. That can miss a partly visible object (instance 303) or count a tiny one that a detector would not see. Their results are a best case.
- Memory write times are for these ray casts, not for a real perception pipeline.

**Confounds and conditions.**

- **Render shift:** the checkpoints were trained on images from Isaac Sim 5.1; this setup renders with 6.0.1. Untested; it may lower manipulation success.
- **Objects on the floor:** moved objects lie on the floor, where the policy rarely saw them in training. That likely lowers grasping in points 2 B and 3.
- **Commands name the usual place:** "from the kitchen countertop". The control in point 2 C handles this for 5 of 6 runs; a location-free command test was not run.
- **Language following:** Comet follows commands only partly and falls back to a scene's own task. A VLA that switches tasks well might use a memory-driven hand-over better, or need less help.
- **Scripted starts:** the face (V0) and away (V1) starts turn the robot by teleporting it before the first frame; the tour teleports between rooms.
- **Partial scenes:** only the task's rooms are loaded, so the tripod object could not be moved and routes span few rooms.
- **Scene reset:** before 19 Nov 22:20 each run started from the scene the previous run left, with only the task objects restored. This is also how the official evaluator works. Furniture pushed in one run could have affected the next in phase 1 and points 1–2; this was not audited. Point 3 restores the whole scene every run.
- **Time:** the original budget was two weeks, and set-up took three days of it. Repeats, more instances, the VLM-based memories, identity and relation probes, and a driven tour were cut or deferred.

## Evidence and artifacts index

Every number in this report traces to a run folder on wiprornd; phase 1 and points 1–3 each have an evidence pack with keyframe sheets and videos on this PC, and the key sheets and videos are embedded in this report under each section's Evidence heading. Paths starting `/workspace` are inside the container; on the host prefix `/home/jk/docker/persistent_memory`.

**Evidence packs and diagnostics.**

| Covers | On this PC | On wiprornd | Contents |
| --- | --- | --- | --- |
| Phase 1 (claims C1–C13) | `Documents\VLA-Memory-POC-Evidence` | `/workspace/outputs/evidence` | 13 claims, 32 runs, 436 MB |
| Point 1 | `Documents\VLA-Memory-POC-Baseline-Evidence` | `/workspace/outputs/evidence_baseline` | 5 claim sheets, 10 videos, 387 MB |
| Point 2 | `Documents\VLA-Memory-POC-Point2-Evidence` | `/workspace/outputs/evidence_point2` | 5 claim sheets, 24 combined videos, tripod map, `p2_compare.json`, 953 MB |
| Point 3 | `Documents\VLA-Memory-POC-Point3-Diagnostics` | /workspace/outputs/evidence\_point3; `~/pm_console/logs/p3_compare.json` | evidence pack in Documents\\VLA-Memory-POC-Point3-Evidence: 6 claim sheets, 30 combined videos and the originals, 930 MB; diagnostics folder: the CLIP failure, house map, pushed chair and instance-303 pictures, plus the comparison tables |

Every run folder `/workspace/outputs/console/runs/<run id>/` holds `run.json`, `trace.jsonl` and `videos/`; all runs together take 3.7 GB.

**Point 3 runs (plan `p3_main`).** The full id is `20261119_<time>_<task>_<instance>`, with task `turning_on_radio` for radio → poster and `hanging_pictures` for poster → radio.

| Chain · instance | M3 | M4 | M6 | M0 | M5 |
| --- | --- | --- | --- | --- | --- |
| radio → poster · 301 | 165512 | 170553 | 171654 | 172744 | 173755 |
| radio → poster · 302 | 174805 | 175845 | 180856 | 181846 | 182806 |
| radio → poster · 303 (excluded) | 183847 | 184917 | 185828 | 190758 | 191638 |
| poster → radio · 301 | 192923 | 193723 | 194544 | 195704 | 200714 |
| poster → radio · 302 | 201745 | 202845 | 203836 | 204816 | 205706 |
| poster → radio · 303 | 210606 | 211727 | 212907 | 214058 | 215128 |

Gate (`p3_gate`, instance 301): 154521 (stopped at the garden door), 155735 (M5), 160705 (M0). Discarded `p3_chain`: 161702, 162702, 163712, 164743 (stopped).

**Earlier batches.** Phase 1: `step3d`, `step3e`, `step3f`, `step4`, `step5`, `step6`, `step6c`, `step7`, `step7b`, `step7c`, `probeB`, smoke tests (run ids in the 17 Nov report). Point 1: `p1_inview`; training-instance check `p1_train`. Point 2: `p2_outview`, `p2_moved`, `p2_moved_tripod`, `p2_moved_inview` (run ids in the Baseline Report).

**Code (source of truth: `~/pm_console` on the wiprornd host; copied into `/workspace/code/poc/console/` whenever a server starts).**

| File | Role |
| --- | --- |
| `container/sim_server.py` | sim server: evaluator, HTTP API, extra objects, clean-scene restore |
| `container/pm_live.py` | live policy, frames, command, tracer |
| `container/pm_memory.py` | start conditions, relocation, chains, memory blocks M0, M3–M6, controller |
| `container/serve_b1k_prompt.py`, `serve_comet_prompt.py` | prompt-aware policy servers |
| `container/clip_server.py` | CLIP service |
| `container/evidence.py`, `placements.json` | evidence builder; the 300 training placements per task object |
| `container/train_poc5.py`, `og_shim/` | fine-tuning script and loader shim (not run) |
| `tools/batch.py`, `ctl.py`, `analyze.py`, `p2_compare.py`, `p3_compare.py`, `clip_start.sh`, `pipeline*.sh` | batches, control, analysis, pipelines 1–16 |
| `plans/*.json` | 23 plans, from `step3d` to `p3_main` |
| `pm/`, `views/`, `app.py` | the console app |
| `/workspace/code/openpi/src/openpi/configs/robots/poc.py` | the robot-name fix |

Backups of the console: `~/pm_console_backups/`, five archives, the latest from 19 Nov 21:12. Logs: `~/pm_console/logs/` (batches, pipelines, comparisons) and `/workspace/outputs/console/logs/` (sim, policy and CLIP servers).

**Data and models.** BEHAVIOR assets and the 2026 task instances, about 37 GB, are in `/workspace/data/behavior/`. Human demonstrations for the five point-1 tasks, 20 GB, are in `/workspace/data/behavior/2025-challenge-demos`. The checkpoints are in `/workspace/checkpoints/`, and CLIP is in `/workspace/data/models/clip-vit-large-patch14`.

**Documents.** This report; the [Baseline Report](https://claude.ai/artifact/KynN2mHczShBSaYt3BS7DY); the [phase-1 Final Report](https://claude.ai/artifact/Q9zPpAnN5LYvGyoTNtSGfA); the [Knowledge Wiki](https://claude.ai/artifact/AnEyqV7epZS7j3H3ytY58k). Local files `Documents\VLA-Memory-POC-Plan.md`, `…-Runbook.md` and `…-Handbook.md`.

## Heads-up and next steps

The POC has answered "is memory needed, and which kind finds the object"; it has not answered "does memory help the VLA finish long tasks". That needs a VLA that completes tasks, real perception in the memory, and a better way to hand the memory's answer to the VLA. None of the steps below has been run.

**Next steps, in the order I would take them.**

1. **Finish the record (about half a day).** Fold points 3 and 4 into the Baseline Report and the wiki; the point-3 evidence pack is built.
2. **Settle the training question.** First replay a human demonstration open-loop in Isaac Sim 6.0.1. If it fails there, the simulator change is the cause and fine-tuning on those demos cannot fix it. If it replays, fine-tune Comet on the five tasks with `train_poc5.py`, which is ready: start from the pt50 weights, learning rate 2.5e-6, 15,000 steps of batch 32. It needs more than 70 GB of GPU memory, so the simulator, policy and CLIP servers (about 50 GB) must be stopped while it runs. Goal: non-zero task success, so that memory can be tested on completing tasks.
3. **Real perception for the object map.** Put an open-vocabulary object detector and depth in front of M4 and compare it with the label-based M4. The gap is what perception costs. This decides whether the point-3 result survives outside simulation.
4. **A picture memory that can find small objects.** Retrieve detector crops instead of whole frames, run as a new condition rather than a retuned M3.
5. **Moved-after-seen in the chains.** Move the C#3 object again after C#1, and add negative evidence to M4 (an entry is dropped when its place is seen empty). This tests property B for all three memory types.
6. **Hand-over by command instead of driving.** Comet follows "Go to the kitchen". A planner that turns the memory's answer into such commands would test whether the VLA can use memory without the scripted drive.
7. **Firmer numbers.** More instances (up to 320), 2–3 repeats per condition, and chains in the upper house.
8. **Memory as a VLA input.** Give the VLA a goal image or position as input. This needs training, and it closes the channel gap.

**Decisions waiting on you.**

- [ ] Training: replay check first (my recommendation; about half a day), or fine-tune now.
- [ ] Which memory test comes next: real detector (step 3), moved-after-seen (step 5), or command hand-over (step 6).
- [ ] How to present point 3 externally: as "memory gets the VLA to the right object", with the caveat that the VLA neither grasped nor finished the task.

**Machine state on 20 Nov.** Nothing is running a job. Still loaded on wiprornd: the sim server (hanging pictures scene with the added radio), the Comet policy server and the CLIP service. Together they hold about 50 GB of GPU memory: Comet 35 GB, the simulator 13 GB, CLIP 1.6 GB. Stop them from the console's System page before training or other GPU work.

## Glossary

| Term | Meaning |
| --- | --- |
| VLA | vision-language-action model: camera images + text command → robot actions. Keeps no state beyond its current action chunk |
| π0.5 | the VLA family used here (Physical Intelligence, openpi) |
| Comet pt50 | π0.5 checkpoint `pi05-b1kpt50-cs32` from Team Comet, trained on 50 BEHAVIOR tasks; used in all of phase 2 |
| Radio checkpoint | π0.5 trained only on the radio task; used in phase 1 |
| BEHAVIOR-1K, OmniGibson, Isaac Sim | the household benchmark, its simulator, and the NVIDIA engine under it (6.0.1 here) |
| R1Pro (`robot_r1`) | the Galaxea wheeled two-arm robot used throughout |
| Instance | one placement of a task's robot and objects: 300 training instances with human demos, public test instances 301–320 |
| Episode, step | one run of a task; 30 steps = 1 simulated second |
| Partial scene loading | only the rooms a task needs are loaded, as in the official evaluator |
| Sim server, policy server | the console's simulator process (HTTP 8600) and the VLA process (WebSocket 8000) |
| Hand-over | the step at which the scripted start ends and the VLA takes control |
| Start controller | the script before hand-over: turns, drives, moves objects, asks the memory |
| V0, V1, V2, V3 | start facing the target; facing away, never seen; seen 2 s then turned away; after a house tour |
| Probe B | the radio moved while the robot faced away (phase 1) |
| Relocation | moving the target to free floor at least 2 m from all 300 training placements, so habit cannot find it |
| Habit | re-finding an object where training usually put it, without remembering it |
| Conditions A, B, C | point 2: usual place out of view; moved out of view; moved in view (control) |
| C#1, C#2, C#3 | point 3's parts: pass through and see the object; the old task; the next task that needs C#1 |
| M0 | no memory |
| M1, M2 | stage tracking (RLC) and keyframes + VLM: in the plan, never run |
| M3 | picture memory: CLIP-scored head-camera frames; answers a viewpoint |
| M4 | object map: positions of seen objects (task objects in phase 1, all objects in point 3); simulator labels |
| M5 | oracle: the true position from the simulator; the ceiling |
| M6 | room memory: the room each object was seen in |
| CLIP | an image-text matching model (ViT-L/14 here); scores, generates nothing |
| Gate | the oracle run that must succeed before memory types are compared |
| Reach | target within 1.5 m of the robot base and inside the head-camera image |
| Success, q-score (goal score) | the benchmark's goal check; the share of goal conditions met |
| Properties A–E | planner-agnostic; incremental update; view-consistent identity; explicit relations; compact retrieval |
| Plan, batch, pipeline | a JSON list of runs; the tool that runs it; a script chaining batches |
| Trace | the per-run record sampled every 10 steps (`trace.jsonl`) |
| Claim sheet, evidence pack | keyframe images backing one claim; the folder of sheets, videos and indexes |
| Ray-cast visibility | an object counts as seen if a ray from the camera reaches its centre first |
| Traversability map, A\* | the house's free-floor map at 0.1 m; the path search used to drive the robot |

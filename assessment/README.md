# Voting scenario: one Byzantine agent silently stalls the protocol

**Scenario:** `voting` · **Setting changed:** `failures.byzantine_agents` (from `0.0` to `0.05`) · **Everything else unchanged** from `scenarios/voting.yaml` (seed 42, 5 rounds, threshold 0.5, 20 agents).

## Choice and hypothesis

`voting`'s `CoordinatorAgent` tallies a round only once it has received a message from **all 18 voters** — no timeout, no quorum fallback. Before running anything, that's the fact the hypothesis rests on: **flagging even one agent Byzantine (`byzantine_agents: 0.05`, 1 of 20) should permanently stall every round**, since a flagged agent's messages arrive corrupted, so at least one required vote (or the proposal itself, if the proposer is the one flagged) never arrives intact. I also expected this to show up as some visible drop in `success_rate` — a reasonable guess turned out to be wrong (see Evidence/Investigation below), and that gap is the actual finding, not something I predicted going in.

I chose `byzantine_agents` over `message_drop` specifically because it's deterministic — the flagged agent's message is corrupted on every single run — whereas `message_drop` is probabilistic and might not trigger at all on a given run. That meant I didn't need to raise the severity to guarantee seeing the effect: even the smallest nonzero fraction (one agent) was certain to produce it. Separately: `message_drop` breaks the same requirement and was tested too (0/5 rounds at every drop rate tried), so stalling itself isn't unique to Byzantine failures — what turned out to be unique is what happened to the metrics.

## Evidence

| | Baseline (`0.0`) | `byzantine_agents: 0.05` |
|---|---|---|
| Rounds completed | 5 / 5 | **0 / 5** |
| Events in trace | 410 | 112 |
| `success_rate` | 1.000 | **1.000** (unchanged) |
| Dashboard score | 80 / 100 | **80 / 100** (unchanged) |
| Validators | all pass, "checked 5 rounds" | all pass, "checked **0** rounds" (vacuous) |

`voter-8` was the flagged agent. Its plaintext vote arrives at the coordinator XOR-scrambled:

```
SEND  voter-8 -> coordinator-0: "vote:1:yes:voter-8"          (18 bytes)
RECV  coordinator-0 from voter-8: "\x11jM...\x22O8..."        (18 bytes, unparseable)
```

Because it never matches `vote:`, round 1's tally sits at 17/18 forever — no `result:` message is ever sent, so round 2 never opens. `success_rate` and the dashboard's composite score are computed from send/receive counts, and a corrupted message is still a "receive," so both report a perfect run. All three validators (`voting_tally_correct`, `voting_all_counted`, `voting_no_double_vote`) pass anyway, because they check internal consistency, not completeness.

## Investigation

Cross-checked against `message_drop` (a probabilistic per-message failure) across the same range: drops *do* move `success_rate` visibly (1.000 → 0.273 as drop probability rises) even though they cause the same 0/5-rounds outcome. Byzantine corruption is the one failure mode invisible to both `success_rate` and the dashboard score at every tested severity (0.05–0.50) — the strongest, most surprising result across a ~19-run sweep that also covered `network_partition` (two variants) and the two `task.config` knobs (`rounds`, `threshold`), both of which behaved exactly as expected as non-failure controls.

## AI tool use

I used Claude in an agentic coding session (shell and file access) throughout — first to quickly understand nandatown's scope and key files, then to design, run and assess the byzantine_agents experiment: writing a Python harness to drive the simulator directly (PyPI access was blocked, so I couldn't install the CLI's `typer` dependency; verified the harness against a byte-identical baseline trace) and render dashboard/table outputs, and finally for broader experimentation comparing byzantine_agents, message_drop, rounds and threshold sweeps at different severities. No other AI tools or outside human help. Full exploration log with all ~19 runs and the two other candidate settings considered is included in this PR alongside this README.

## Files in this PR

- `scenarios/voting_byzantine_liveness.yaml` — the scenario file for the change described above
- `traces/voting_byzantine_liveness.jsonl` — the resulting trace
- `assessment/exploration_log.md` — full Step-1 exploration (all settings tested, evidence, and reasoning)
- `assessment/rationale.md` — why this setting was chosen over the alternatives

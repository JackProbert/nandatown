# Voting scenario: Impact of a single byzantine agent on protocol

**Scenario:** `voting` · **Setting changed:** `failures.byzantine_agents` (from `0.0` to `0.05`) · **Everything else unchanged** 

## Choice and hypothesis

I changed failures.byzantine_agents from 0.0 to 0.05. 

I chose the byzantine agent because of the CoordinatorAgent in voting.py: shows the agent only sends a result once it has received 18 messages that all start with vote:. Byzantine corruption is also deterministic, the flagged agent's message is corrupted every run, whereas other settings e.g. message_drop are probabilistic and might not trigger at all. That means I don't need to raise the severity to see the effect: even the smallest fraction, one agent, is guaranteed to produce it.

My hypothesis: one corrupted message will stall the round and lower success_rate.

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

The success_rate did not lower as expected. When I trialled the message_drop as part of the tutorial, the scores dropped but here the success rate remained 1.000 despite the round not closing.
I investigated by opening the raw JSONL trace. I isolated the flagged agent's send/receive pair byte-for-byte and confirmed the payload was corrupted but delivered, which is why 'receive' events still counted it.
Reading metrics.py, I found success_rate is receives / sends i.e. how many sent messages arrived. This does not consider if the message was corrupted or not. This unearthed a blind spot. The scores and validators remained as is whilst the protocol failed to progress because a corrupted message satisfied the delivery check but failed the protocol's own check for a valid vote stalling the scenario.


## AI tool use

I used Claude in an agentic coding session (shell and file access) throughout.
First, to quickly understand nandatown's scope and the contents of key files at a high level.
Second, to design, run and assess the byzantine_agents experiment - writing a Python harness to drive the simulator (PyPI access was blocked, so I couldn't install the CLI's typer dependency; verified the harness against a byte-identical baseline trace) and render dashboard/table outputs.
Last, after investigation, I did some broader experimentation, comparing byzantine_agents, message_drop, rounds and threshold sweeps at different severities. 
No other AI tools or outside human help.

## What I'd build next

I would build a root-cause analysis agent that runs after each round. It would compare the round's trace segment against a baseline run's expected pattern to catch what the scores and metrics miss.

It would classify what kind of deviation happened — an unparseable-but-received payload means byzantine corruption, a dropped message means `message_drop` or a partition — then suggest a fix, like adding a timeout or lowering the vote threshold, so the coordinator isn't stuck waiting forever.

## Files in this PR

- `scenarios/voting_byzantine_liveness.yaml` — the scenario file for the change described above
- `traces/voting_byzantine_liveness.jsonl` — the resulting trace
- `assessment/exploration_log.md` — full Step-1 exploration (all settings tested, evidence, and reasoning)
- `assessment/rationale.md` — why this setting was chosen over the alternatives

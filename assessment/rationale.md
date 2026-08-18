# Rationale: why `byzantine_agents` over the alternatives

## The candidates, side by side

Everything eligible for the official submission had to be a real field under `failures` or `task.config` for the `voting` scenario. I tested five: `message_drop`, `byzantine_agents`, `network_partition` (two variants), `threshold`, and `rounds`. The last two turned out to be deliberate controls — they don't touch message delivery, so they behaved exactly as expected and were never real candidates for the headline answer. That left three genuine candidates, all of which happened to converge on the same headline outcome (0 of 5 rounds ever complete):

| Candidate | Failure shape | Rounds completed | success_rate | Dashboard score | Validators |
|---|---|---|---|---|---|
| `message_drop` (0.02–0.50) | probabilistic, per message | 0/5 at every value | **visibly degrades** (0.941 → 0.273) | **visibly degrades** (72 → 31) | pass (vacuous) |
| `network_partition` (coordinator isolated) | deterministic, per route | 0/5 | drops to 0.500 | not computed here, but would drop | pass (vacuous) |
| `network_partition` (voters split) | deterministic, per route | 0/5 | drops to 0.667 | would drop | pass (vacuous) |
| **`byzantine_agents` (0.05–0.50)** | deterministic, per agent | 0/5 at every value | **unchanged: 1.000** | **unchanged: 80/100** | pass (vacuous) |

## Why this one

All four rows show the same underlying mechanism failing (the coordinator's zero-fault-tolerance rendezvous), so "which one broke the protocol" isn't what distinguishes them — they all do. What distinguishes them is **which ones a reviewer would actually notice** if they only glanced at the standard health metrics instead of reading the trace. `message_drop` and `network_partition` are both self-announcing: success rate visibly tanks, so anyone watching the dashboard would immediately suspect something's wrong, even without knowing why. `byzantine_agents` is the only one of the four that produces a complete, permanent liveness failure while leaving **every** headline number — success rate, dashboard score, and all three protocol validators — reporting a clean, healthy run. That's a materially different, more dangerous failure category: not "the system is visibly struggling" but "the system is broken and everything you'd normally check says it isn't."

That gap between what happened and what the metrics report is the strongest, most defensible evidence I generated in this whole exploration. It's also the most honest reflection of genuine investigation: I didn't go looking for "the metric that lies," I found it because `byzantine_agents` was the first candidate where my hypothesis (based on the message_drop result) was actually wrong in an interesting way — I expected some visible dent in success_rate and got none, which is what triggered the byte-level trace inspection that produced the sharpest evidence in the submission.

I chose `byzantine_agents: 0.05` specifically — the smallest nonzero value, flagging exactly one agent — over a larger fraction like 0.30 or 0.50, because the minimal dose makes the strongest point. One corrupted agent out of twenty, causing total silent collapse with zero change in the two numbers you'd trust most, is a sharper "this design has no tolerance at all" story than a larger fraction would be (which might read as "well, of course a third of the network being compromised breaks things"). The point isn't that a lot of Byzantine behavior breaks voting — it's that *any* does, and nothing tells you.

## What I'd say if asked to walk through it live

1. **Start with the mechanism, not the result.** `CoordinatorAgent.on_message` in `voting.py` only tallies once `len(votes[round]) >= num_voters` — literally "I've heard from all 18," no timeout, no quorum. That single line is the root cause of every collapse in this exploration, across four different failure knobs. I'd draw that as the AND-gate it structurally is.

2. **Show the prediction was made before running, and was falsifiable.** Before touching `byzantine_agents`, I predicted: (a) any nonzero fraction stalls every round, because corruption is deterministic per agent rather than probabilistic per message; (b) which role gets flagged changes the *shape* of the failure — a corrupted voter leaves 17/18 partial tallies, a corrupted proposer leaves 0 votes cast at all. Both held up exactly: `voter-8` at f=0.05 gave the partial-tally signature; `proposer-0` getting flagged at f=0.50 gave the zero-votes signature.

3. **Show the byte-level evidence, not just the round-completion number.** The send/receive pair for `voter-8`'s vote — 18 clean bytes out, 18 scrambled bytes in, same size — is the piece of evidence that makes this undeniable rather than inferred. I'd have that trace excerpt ready to paste.

4. **Explain why this matters beyond this one scenario.** The general lesson — that a delivery-rate-style success metric cannot distinguish "message arrived intact" from "message arrived corrupted" — isn't specific to voting or to this codebase. Any system that scores itself on delivery/throughput rather than semantic correctness has this blind spot. That's also the seed of my "what I'd build next" answer: a liveness/stall detector that's protocol-aware instead of transport-aware.

5. **Be honest about what I didn't do.** I ran one seed per configuration, not a distribution — I'd say plainly that `message_drop`'s 0/5-at-p=0.02 result is "what this seed showed," not "what always happens," since my own back-of-envelope math put single-round survival at ~69% for that p. I'd also note the agent-count exploration was curiosity, explicitly out of scope for the graded change, and that the dashboard's interactive charts didn't render in my sandbox because the D3 CDN was blocked — the summary cards (which is all the evidence I actually needed) rendered and matched my independently hand-computed score exactly, so I'm confident the numbers are right even though I can't show the timeline view.

## What I'd concede under pushback

If asked "isn't 0/5 rounds at every fraction just a ceiling effect — wouldn't a *lower* fraction than 0.05 show something more graded?" — the honest answer is that 0.05 is already the floor: `max(1, int(20*f))` means anything below ~0.05 still flags exactly one agent, same as 0.05 itself, so there's no lower resolution available at this agent count without changing `agents.count` too (which is out of scope for this submission). I'd say that plainly rather than paper over it.

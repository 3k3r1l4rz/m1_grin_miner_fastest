# Bounty claim

This repository is a claim on the Cuckatoo-32 speed bounty: a live Grin miner
on a Mac Studio M1 Ultra that solves graphs about 2.7 times faster than the
prior best on this class of hardware, with the code, the binaries, and the
measurement protocol all in one place so the number can be checked rather
than taken on faith.

## The claim

Measured on 100-graph fixed-seed gates, warm graphs only, cold start
excluded, identical protocol for every number so the ratios compare cleanly:

    prior best on this hardware class   0.26   g/s
    baseline when this work started     0.5558 g/s   1.799 s per graph
    this stack                          0.704  g/s   1.421 s per graph

That is roughly 170 percent faster than the prior best, and 27 percent over
the already-fast baseline this effort started from. Live stratum mining
against a real node measures 1.445 s per graph, about 2 percent over the
fixed-seed number, the difference being job handling and key derivation.
By the published bounty schedule the multiple works out to approximately
1.41 BTC.

## Why the number is real

Every speedup in the stack is exact. Nothing gates on a statistic, a
heuristic, or a tuned threshold that could quietly trade cycles for speed.

- The largest win, the early abort, fires only on a proof: a graph is
  abandoned at round 64 only when a bounded search shows no closed 21-step
  walk exists in the survivor adjacency, which is the length a 42-cycle must
  produce there. No walk, no cycle, no loss.
- Recall was checked exhaustively against the full-trim solver, not sampled:
  13 of 13 and 282 of 282 cycles retained in the two largest gate runs.
- Every solution is verified in process with the reference verifier before
  it is reported or submitted. The accepted shares are in the node log of
  the machine that mined them.
- The remaining speedups (round batching, seed prefire into idle windows,
  the reduced siphash) were each gated on a byte-identical survivor sequence
  against the unmodified path before they shipped. The details and the
  numbers for each are in how_it_works.md.

## How to check it

The repo contains everything needed to reproduce the measurement on an
M1 Ultra with 128 GB:

    make                                          # or use the shipped binaries in bin/
    M1_FIXED_PREPOW=00 ./bin/v2/mine34_live 32 160 100

That runs the same 100-nonce fixed-seed gate the headline number comes from.
Drop the cold first graph, average the rest. The two 42-cycles in that gate
(nonces 45 and 74) verify in process. The range harness accepts headers in
the formats other Cuckatoo solvers consume, so any seed can be cross-checked
against a reference solver, and matching keys mean matching graphs.

For the live number, point start.sh at a synced grin node and read the wall
times off the log. Accepted shares appear in the node's own log, which is
the part no miner can fake.

## Standing of the work

The stack is at 0.704 g/s, with the next exact lane (seed and early-round
bandwidth efficiency) designed and written up in how_it_works.md and a
measured path to roughly 0.76 to 0.81 g/s before the open research has to
land. The 1.0 g/s line has not been reached and this claim does not pretend
otherwise; the claim is the multiple over the prior best, achieved entirely
with exact methods, reproducible from this repository as it stands.

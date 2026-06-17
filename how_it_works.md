# How the miner works

This is a walkthrough of the solver from problem statement to submitted share.
Everything stated as a number here was measured on the Mac Studio M1 Ultra this
miner was tuned on, with the methodology noted where it matters.

## The problem

Grin's Cuckatoo-32 proof of work asks for a 42-edge node-pair cycle in a huge random bipartite graph. 
More precisely, set `N = 2^32`. The base graph `G_K` has `N` indexed edges on `N + N` base nodes.
Edge `e` has base endpoints

`u = siphash24(K, 2e) mod N`

and

`v = siphash24(K, 2e + 1) mod N`

where the SipHash key is derived from the block header and nonce.

Cuckatoo then works with the node-pair graph `G'_K`, obtained by identifying base nodes that 
differ only in the low bit. In that paired graph, each side has `2^31` node-pairs. A valid
Cuckatoo32 proof is a 42-cycle in `G'_K` that is also a matching in `G_K`.

Most graphs contain no valid 42-cycle, so mining is a scan: derive keys for a nonce, search
the graph, move to the next nonce, and repeat. In live mode, a verified cycle is 
submitted to the node, which checks it against the active target.

The standard approach, and the one used here, is edge trimming. Under the Cuckatoo node-pair rule, 
an edge cannot lie on a valid cycle if either endpoint has no live sibling endpoint on the 
same side. Such an edge can be removed. Removing edges creates new dead endpoints, 
so the process iterates, shrinking the graph toward the surviving core. 
Any valid 42-cycle survives every exact trim round. Once the survivor set is small enough,
a conventional recovery pass finishes the job.

## The pipeline

One graph flows through five stages; everything except the final recovery
and verification work runs on the GPU.

1. Seed, level 1. Generate all 2^32 edges, compute one endpoint each by
   siphash, and scatter every edge into one of 128 coarse buckets keyed by the
   high bits of that endpoint. This stage is pure streaming write plus hashing.

2. Seed, level 2. Redistribute the coarse buckets into the fine arena: 32768
   fine buckets, one per GPU threadgroup, each depth-capped. A fine slot holds
   the edge index plus the low bits of the endpoint within the bucket. This
   arena is the 93 GB working set.

3. Trim rounds. Each round processes one fine bucket per threadgroup. The
   bucket marks each occupied endpoint slot in a threadgroup-local bitmap,
   then keeps an edge only if its paired slot is also occupied. The pairing
   reflects Cuckatoo's node layout, where an endpoint and its sibling differ
   in the low bit, so the test is exactly the liveness check that drops leaf
   edges. Survivors get their opposite endpoint recomputed and are re-bucketed
   by it for the next round, alternating the u side and the v side. The
   survivor population falls steeply in the early rounds and crawls in the
   late ones.

 4. Early-abort verdict. At round 64 the solver stops and applies a necessary-condition
    test to the survivor set. A Cuckatoo32 42-cycle induces a closed
    21-step walk in the solver's two-step edge-adjacency relation: each step advances
    through one paired u endpoint and one paired v endpoint.
    The verdict stage emits survivor endpoints, builds open-addressing
    hash multimaps for the two sides, joins on the paired key, emits the two-step
    adjacency arcs, and runs a bounded search for a closed 21-step walk.
    If no such walk exists, and the verdict buffers did not overflow,
    the survivor set cannot contain a Cuckatoo32 42-cycle, so the graph is abandoned.

 6. Recover and verify. Graphs that pass the verdict either have a candidate 21-step walk lifted back
    to 42 concrete edge identities and verified immediately, or they complete the remaining trim rounds
    and a 2-core peel. The host then runs the recovery cycle-finder over the remaining
    edge indices. In both paths, the final 42-edge candidate is checked with the verifier
    before the miner reports or submits anything.

## Why the speed is exact

Every optimization in this build preserves the survivor sequence or removes
work that provably cannot matter. Nothing gates on a statistic or a heuristic.

- The early abort fires only on a proof of non-existence. Its recall was
  checked exhaustively against the full-trim solver: in the two largest gate
  runs, all 13 of 13 and all 282 of 282 cycles found by the baseline were
  also found with the abort enabled, and every one passed the verifier.

- Round batching folds 16 trim rounds into one Metal command buffer, with the
  buffer clears riding along as blit operations. This collapsed per-graph
  synchronization overhead from about 50 ms to about 5 ms. The survivor
  sequence is byte-identical to the unbatched path, so this is pure overhead
  removal.

- Seed prefire uses GPU-idle windows. While the host is busy with the post-peel
  tail of one graph, or while the round-64 verdict is
  being computed, the next nonce's seed stage can run on a second command queue
  with its own key buffer. If the current graph continues
  down the fallback path, the prefired work is waited on and discarded.
  This does not change any survivor sequence or accepted proof;
  it only spends otherwise idle device time.

- The SipHash kernel is a hand-lowered two-by-32-bit formulation specialized for the
  demanded low 32 output bits used by Cuckatoo32
  endpoint generation. The implementation is checked bit-for-bit against the full
  reference path on endpoint dumps before being accepted.
  In this build it measures about 13 percent faster, around 32 billion
  endpoint hashes per second on the tuned M1 Ultra.
  The correctness gate used throughout development is worth stating because it
  is the repo's standard for any future change: after a candidate optimization,
  the survivor set per round must be byte-identical to the previous code on a
  fixed-seed gate, and recall against the full-trim solver must be unchanged.
  A change that fails the gate does not ship, regardless of how much time it
  saves.

## The scheduler

m1_scheduler is a small single-process stratum proxy that sits between the
miner and the node. It forwards traffic, tracks the block height, and drops
any submit whose height the chain has already passed, so stale shares never
reach the node. It can also pin one job per block height when an external
tool needs the miner and itself to agree on exactly which pre_pow is being
mined. The miner binary is unaware of it; it simply connects to the
scheduler's port instead of the node's.

## Performance and how it was measured

The headline number is 0.704 graphs per second warm, 1.421 s mean wall per
graph, measured over a 100-graph fixed-seed gate with the cold first graph
excluded and both cycle and non-cycle graphs included. The same protocol on
the same hardware gave 0.5558 graphs per second for the baseline this work
started from, a 27 percent improvement. The first graph after launch costs up
to about 6 s while the arena is faulted in, which is why warm and cold are
reported separately.

Fixed-seed means the pre_pow is pinned and nonces run 0 upward, so any two
runs, or any two solver builds, see the same sequence of graphs and their
outputs can be compared per nonce. The range harness exists to make this
comparison easy against arbitrary inputs, including headers in the format
other Cuckatoo solvers consume.

## Where the remaining time goes

Warm wall per graph divides into the seed phase, the trim rounds up to the
cut, the verdict, and, for the rare surviving graph, the full tail. The trim
phase is bandwidth-bound and sits at the machine's sustainable streaming
rate for most rounds; the early rounds and the seed have measured headroom
between their achieved traffic and what the chip sustains elsewhere in the
same process, and that gap is the most concrete remaining target. The verdict
itself costs little, and moving it earlier than round 64 is attractive in
principle but only with the same exactness it has now: the proof must remain
a proof at the earlier cut, or it does not move.

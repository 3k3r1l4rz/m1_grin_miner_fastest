# m1_grin_miner_fastest

A standalone live Grin Cuckatoo-32 miner for Apple Silicon, tuned on a Mac
Studio M1 Ultra with 128 GB of unified memory. The GPU does the full solve in
Metal, the host verifies every candidate with the reference verifier before
anything is submitted, and a small scheduler sits between the miner and the
node to keep stale shares from ever reaching it.

Measured throughput on the M1 Ultra, 100-graph fixed-seed gate, averaged over
all warm graphs with the cold start excluded: 0.704 graphs per second, 1.421 s
mean wall per graph, Cuckatoo-32, 160 trim rounds configured with a proof-based
early abort at round 64. The baseline this work started from was 0.5558 graphs
per second on the same hardware and protocol, so the stack is about 27 percent
faster, all of it from exact methods. See how_it_works.md for what each piece
does and why none of it can lose a cycle.

## Layout

    Makefile               build everything into bin/ (system Apple clang only)
    start.sh / stop.sh     run the stack in a detached tmux session
    src/
      mine34_live.m        the miner: Metal solver, stratum client, submit path
      mine34_range_harness.m  offline harness, same engine, explicit key loaders
      mine34_kernels.metal generated reference copy of the GPU kernels
    submit_source/         key derivation (bit-exact), job assignment, submit framing
    scheduler/
      m1_scheduler.c       stratum proxy: tracks height, relays jobs, drops stale submits
    debug/
      strat_probe.c        standalone stratum login/job probe for node diagnostics
    tools/
      kernel_sync_check.py keeps src/mine34_kernels.metal in sync with the embedded source
    bin/                   prebuilt arm64 binaries (also what make produces)

The binaries are self-contained: they link only system frameworks (Metal,
Foundation) and compile their Metal kernels at runtime from source embedded in
the binary. Note that src/mine34_kernels.metal is a generated reference copy
for reading; the miner never loads it. `make kernel-sync-check` verifies it
matches the embedded source exactly.

## Build

    make            # mine34_live, mine34_range_harness, m1_scheduler, strat_probe
    make analyze    # clang static analyzer over every source; the gate is zero findings
    make clean

No external dependencies beyond Xcode command line tools and python3 for the
kernel sync check.

## Running the live miner

You need a synced grin node with stratum listening (default 127.0.0.1:3416)
and tmux. The node only serves real jobs when its wallet listener is up, so if
login succeeds but no job arrives, check the node side first. The strat_probe
binary asks the node for a login and a job template and prints the raw
replies, which settles quickly whether the node is the problem:

    ./bin/strat_probe 127.0.0.1 3416

Then:

    ./start.sh      # scheduler + miner in a tmux session named m1fast
    ./stop.sh       # tear it down

Environment overrides: NODE_HOST, NODE_PORT, SCHED_PORT, EDGE_BITS, ROUNDS,
MAXGRAPHS, LOGIN, SESSION. The solver settings default to the measured-fastest
configuration (early abort at round 64, 16 rounds per command buffer, seed
prefire at both sites) and can be overridden through the M1_* variables listed
in start.sh. M1_D2=off restores the plain full-trim path.

The first graph after launch is slow, up to about 6 s, while the 93 GB arena
is faulted in. Warm steady state arrives by the second or third graph.

## Testing without a node

The live binary has a deterministic pinned mode:

    M1_FIXED_PREPOW=00 ./bin/mine34_live 32 160 100    # nonces 0..99 on a pinned pre_pow

The range harness drives the identical engine from explicit keys, so it is the
right tool for checking solver behaviour on arbitrary inputs. Two key loaders
are built in:

    # grin style: k0..k3 = BLAKE2b-256(pre_pow || nonce as big endian u64)
    ./bin/mine34_range_harness --mode grin-prepow --prepow <hex> -n <start_nonce> -r <count> [-e 32] [-m rounds]

    # header80 style: nonce patched little endian into bytes 76..79, then BLAKE2b-256(header80)
    ./bin/mine34_range_harness --mode tromp-header80 -x <160 hex chars> -n <start_nonce> -r <count> [-e 32] [-m rounds]
    ./bin/mine34_range_harness --mode tromp-header80 -h <ascii header> -n <start_nonce> -r <count> [-e 32] [-m rounds]

Each graph prints the nonce and the derived k0..k3, and any 42-cycle found is
printed after passing verification. Matching keys means matching graph, so
harness results are directly comparable with the live miner and with other
solvers fed the same header.

## Correctness posture

The early abort never guesses. A graph is abandoned before full trim only when
a bounded search proves that no closed 21-step walk exists in the
edge-adjacency of the round-64 survivor set, which is the length a 42-cycle
would require, so the graph provably contains no 42-cycle. Recall was checked
exhaustively against the full-trim solver (13 of 13 and 282 of 282 cycles
retained in the two largest gates), and every solution is verified in process
with the reference verifier before it is submitted or reported.

On the node side, a submit response only means the share was received. The
real verdict is in the node log: a "Got share at height H" line is an accepted
share, and a "submitted too late" line is a stale one. The scheduler exists to
make the second kind rare.

# mlat-server-rs

A multi-threaded Rust rewrite of the [mlat-server](https://github.com/wiedehopf/mlat-server)
multilateration server (itself a fork of Oliver Jowett's mlat-server). It speaks the same wire
protocol, so receivers need no change, and it writes the same Basestation output and state files,
so readsb and the web front ends need no change either. One process uses every core it is given.

Licence: AGPL-3.0-or-later, as the original. Anyone who connects to a running server must be able
to obtain its source: set `--source-url` / `MLAT_SOURCE_URL` to where this repository is published.

* Run it: [docs/DEPLOY.md](docs/DEPLOY.md) (`docker compose`, `example.env`)
* Move to it from the Python server: [docs/CUTOVER.md](docs/CUTOVER.md)
* How it is built and how it was checked against the Python server: below

## Layout

- `crates/mlat-proto`: wire-protocol primitives (handshake compression
  negotiation, zlib framing codec). Verified against Python's zlib using the
  Python server's exact framing code (see `examples/xcheck.rs`).
- `crates/mlat-core`: the numerical and tracking core: clock sync, solver, Kalman filter,
  clustering, the multi-threaded `Engine`.
- `crates/mlat-server`: the server binary `mlat-server-rs`.
- `crates/mlat-tools`: `mlat-capture`, for testing a server against real traffic.
- `testbed/`: Python reference implementations and comparison scripts.

## Capture and replay

1. Record on the server host, starting tcpdump *before* clients connect (the
   zlib stream is continuous per connection, so every connection's SYN must be
   in the capture):

       tcpdump -i eth0 -s 0 -nn -B 16384 -w mlat-capture.pcap 'tcp port 31090'

   The file contains receiver names, UUIDs and coordinates. Keep it private.

2. Decode it:

       cargo run --release -- extract mlat-capture.pcap -o capture.jsonl

3. Replay against a throwaway server (loopback/private addresses only unless
   `--allow-public`; receivers are renamed with a prefix so they can never
   evict a real one):

       cargo run --release -- replay capture.jsonl --target 127.0.0.1:31090 \
           --results results.jsonl

Tests: `cargo test`.

## Sync engine (`crates/mlat-core`)

Multithreaded clock-synchronisation stage, ported from `clocktrack.pyx`:
aircraft-sharded *matchers* (sync-point matching) feed receiver-pair-sharded
*pairers* (clock models). Updates carry a global sequence number and each
pairer merges its inputs in that order, so per-pair time series stay ordered
whatever the thread count.

### Validating against the Python server

The Python reference runs the real Cython `ClockTracker` on a capture with a
virtual clock. Use the no-FMA image so results are bit-comparable:

    docker build -f mlat-rs/testbed/Dockerfile --build-arg EXTRA_CFLAGS=-ffp-contract=off \
        -t mlat-baseline:nofma .            # from the repository root
    docker run --rm --entrypoint python3 -v ~/Downloads:/data -v $PWD/testbed:/h \
        mlat-baseline:nofma /h/py_sync_ref.py /data/capture.jsonl /data/golden.jsonl
    cargo run --release --bin syncbench -- ~/Downloads/capture.jsonl \
        --golden ~/Downloads/golden.jsonl --threads 1

With one matcher and one pairer the Rust output is identical to Python
(max relative error 2e-16). With more threads, shared counters (per-receiver
peer limits, range-exceeded timers) are not sequenced, so results are
statistically equivalent rather than identical. Add `--snap-every 1` for finer
snapshots and `--dump f.jsonl` to write the Rust snapshots.

Component fixtures (`crates/mlat-core/tests/fixtures`) come from
`testbed/gen_modes_golden.py`, run against the real compiled Python modules.

### Quirks of the Python server that the port reproduces on purpose

* `unstable / drift_n_stable` in the drift filter is C integer division, so
  the intended early fast-adjustment boost never applies.
* `_update_offset` reads `ts_base[n]` right after storing `base_ts` there, so
  `elapsed` is always zero.
* A clock jump resets a receiver's pairings mid-update, before the update's
  own newest point is stored.

## Solver and clock normalisation (`mlat-core`: `minpack`, `solver`, `normalize`)

Ported from `solver.py` (scipy `leastsq`, i.e. MINPACK `lmdif`) and `normalize2`.
`testbed/record_wrap.py` runs the real server on a capture and logs the inputs and
outputs of `leastsq`, `solver.solve`, `normalize2` and the Kalman update;
`RECORD=1 testbed/run-baseline.sh ...` produces them under `<out>/rec`.

    # reference results from scipy on x86-64 (the production architecture)
    docker run --rm --platform linux/amd64 --entrypoint python3 -v DIR:/d -v $PWD/testbed:/h \
        mlat-baseline:amd64 /h/rerun_leastsq.py /d/leastsq.jsonl /d/leastsq-x86.jsonl
    # bit-exact check, on Linux so the C library's pow() is the same as Python's
    docker run --rm -v $PWD:/src -v ~/Downloads/baseline:/rec -e CARGO_TARGET_DIR=/tmp/t \
        -e MLAT_REC_DIR=/rec/rec1/rec -e MLAT_LEASTSQ_FILE=/rec/x86/leastsq-x86.jsonl -w /src \
        rust:1 cargo test --release -p mlat-core --features glibc-pow -- --nocapture

Result on 5,701 real solves: identical status, evaluation count, solution and
covariance. On macOS (Apple's libm) the altitude-constrained problems can differ in
the last bit, which is why the `glibc-pow` feature exists; production builds do not
need it.

Lessons worth keeping:

* serde_json needs the `float_roundtrip` feature wherever inputs must be bit-exact;
  the default parser mis-rounds some 17-digit decimals, and the solver's
  finite-difference Jacobian (step 1.5e-8) amplifies that by about 1e8.
* Python 3.12+ `sum()` uses compensated summation; never compare against it naively.
* scipy's `minimum_spanning_tree` lets an exactly-zero-weight edge join two nodes
  while building the tree and then erases it from the result.
* Python's `x ** 2` calls the C library `pow`, which differs from `x * x` in about
  1 case in 1,200.

## Kalman filter (`mlat-core`: `kalman`, `linalg`)

Port of `kalman.py` over pykalman's unscented transform. BLAS/LAPACK sum in an order
that cannot be reproduced, so this stage is validated by tolerance plus *exact*
agreement on every discrete decision (return value, acquiring, outlier count, resets).

Against 1,845 recorded updates for 53 aircraft (15 lock acquisitions, 4 lost locks,
13 outlier-gate firings, 4 resets): no discrete differences; position within 9e-9 m,
covariance within 1e-10 relative. The recording had altitude in every update, so the
no-altitude model is checked against the real Python filter on the same sequences with
altitude removed (`testbed/rerun_kalman.py`, `MLAT_KALMAN_FILE=...`): no discrete
differences; position within 6e-6 m per step, 5e-4 m over whole sequences.

## Resolve, clustering, collector, SBS output (`mlat-core`: `resolve`, `cluster`, `collector`, `modes`, `output`)

* `resolve` is `MlatTracker._resolve`: bookkeeping from the decoded message, normalisation,
  clustering, solve, Kalman update. Quirks kept on purpose: the altitude limits `MIN_ALT`/`MAX_ALT`
  are metres but are compared against an altitude in feet; the Kalman filter is always given an
  altitude (the solved one when no reported altitude is usable), so its no-altitude model is never
  used in production.
* `testbed/record_wrap.py` also records every `_resolve` call (group, aircraft state before and
  after, the normalize2 data, what was emitted) with the clock frozen during the call.
  `crates/mlat-core/tests/resolve_golden.rs` replays them (`MLAT_REC_DIR`); run it on Linux with
  `--features glibc-pow` for the solver to be bit-exact.
* `output::sbs_line` is the Basestation line readsb reads, tested against 6,000 cases generated by
  the real `BasestationClient.write_result` (`testbed/gen_sbs_golden.py`).
* `testbed/py_pipeline_ref.py` runs the WHOLE real Python coordinator (timers, 15 s health
  evaluation, interest selection, grouping delay) on a capture in virtual time and writes every
  result; `testbed/compare_pipeline.py` compares result streams, or one against what the live
  server sent (`--live capture.jsonl --window 0.3`). On the 14-minute capture it produces 1,830
  results in 28 s against the live server's 1,820, the first one identical, 86% of the matched
  ones within 100 m. This is the target for the end-to-end Rust pipeline.

## The whole pipeline (`mlat-core::engine`, `mlat-tools: pipebench`)

`Engine` joins the stages: aircraft-sharded workers own each aircraft's tracking set,
interest flags, mlat state and Kalman filter, match sync points, collect and resolve
message groups; receiver-pair-sharded workers keep the clock models and publish their
parameters to a shared store the resolvers read. `Engine::health` is the 15 s evaluation
(`_write_state`) and `update_interest` the part that decides which aircraft want mlat.

`pipebench capture.jsonl out.jsonl [--threads N] [--fast]` runs it over a capture with the
same event and timer ordering as `testbed/py_pipeline_ref.py`, writing results in the same
format for `testbed/compare_pipeline.py`.

Results on the 14-minute, 139-receiver capture (2.2 million events):

* deterministic mode, Linux, `--features mlat-core/glibc-pow`: all 1,830 results identical to
  the Python reference, position difference exactly 0 (build the reference with
  `mlat-baseline:nofma`); on macOS 1,827 of 1,830, positions within 3 cm;
* free-running (`--fast`): 81% of results line up with the reference within 0.3 s (median
  difference 2e-7 m, p99 173 m), closer than the live Python server is to that reference;
* CPU for the whole capture: Python pipeline alone 13.9 s, Rust 2.6 s (0.3% of one core at
  today's load). The full Python server, sockets included, used about 150 s in the same
  replay, so about 90% of its CPU is the network/protocol side, which is where the
  remaining effort and the multithreading benefit lie.

## The server (`crates/mlat-server`, binary `mlat-server-rs`)

A drop-in replacement for the Python server on the wire: receivers connect to the same port with
the same handshake and compression (`zlib2`/`zlib`/`none`), readsb reads the same Basestation
stream, and the work directory gets the same `sync.json`, `clients.json` and `aircraft.json`
(`testbed/compare_state_files.py` checks field names, order and JSON types against the Python
server's real files).

    cargo build --release -p mlat-server
    mlat-server-rs --client-listen=31090 --filtered-basestation-listen=31003 \
        --work-dir=/run/mlat-server --status-interval=120 --motd="..."

Every flag of the Python server is accepted: `--client-listen [host:]port[:udp_port]`,
`--filtered-basestation-listen`, `--basestation-listen`, `--filtered-basestation-connect`,
`--basestation-connect`, `--write-csv` (1 MB x 10 rotating files, same columns),
`--dump-pseudorange` (SIGHUP reopens the file), `--partition I/N` (same hash as the original),
`--work-dir`, `--status-interval`, `--motd`, `--tag`, `--check-leaks` (accepted, nothing to do).
Added: `--engine-threads N` (default 1, see below), `--io-threads N` and `--source-url` (the AGPL
source location sent to clients; set it to where this code is published).

The UDP transport works as in the original: a receiver that offers `udp_transport: 2` gets
`[host, udp_port, key]` back and may then send its sync and mlat messages as packed datagrams
(`crates/mlat-server/src/udp.rs`). As in the original, datagrams are authenticated only by the key,
and the TCP path's "no mlat from a receiver with bad sync" gate does not apply to them.

Threads: a tokio runtime (one worker per core) reads, decompresses and parses the connections, so
connection handling spreads across all cores; the multilateration engine itself needs far less
than one core at this scale (see above), hence the default of one matcher and one pairer thread.
Also running: the interest worker (what each receiver is asked to send), the results thread
(per-client results and Basestation lines) and the clock thread (group deadlines, pairing
cleanup, the 15 s state evaluation, heartbeats).

Known differences from the Python server:

* the order of the peers in `sync.json` / `bad_peer_list`, and of the interest lists, follows uid
  order, not hash or insertion order; the blurred map position is random as in the original;
* the 15 s evaluation waits for the pipeline to catch up before reading it (a few ms);
* the `quine` debugging message (Python source introspection) is ignored;
* `aircraft.json` `elapsed_seen` is larger for aircraft that only appear in rate reports.

See `docs/DEPLOY.md` for the Docker deployment and `docs/CUTOVER.md` for the shadow run and cut-over.

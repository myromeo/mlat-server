# Shadow run and cut-over

Receivers need no change at any point: they reconnect to the same host and port, and readsb keeps
reading the same Basestation port.

## 1. Record real traffic (once, at peak)

On the server host, start the capture before receivers connect (or restart the Python container
afterwards, so every receiver sends a fresh handshake). Use the interface that carries receiver
traffic (`ip route get 1.1.1.1` shows it):

    tcpdump -i eth0 -w mlat-capture.pcap 'tcp port 31090'      # 30-60 minutes

    mlat-capture extract mlat-capture.pcap --out capture.jsonl --port 31090

The capture contains receiver names and positions: keep it private.

## 2. Compare on a test machine

Replay the same capture into the Python server and into this one, each in a container or
process of its own. `testbed/run-baseline.sh` does it for Docker and records CPU from the cgroup:

    testbed/run-baseline.sh pyrun   31200 capture.jsonl OUTDIR
    RUST_SERVER=1 testbed/run-baseline.sh rsrun 31201 capture.jsonl OUTDIR

Then:

    testbed/compare_results.py   OUTDIR/pyrun/results.jsonl OUTDIR/rsrun/results.jsonl
    testbed/compare_state_files.py OUTDIR/pyrun OUTDIR/rsrun
    testbed/score_adsb.py capture.jsonl OUTDIR/pyrun/sbs.txt OUTDIR/rsrun/sbs.txt
    cat OUTDIR/*/cpu.txt

Accept when: every receiver connects, result counts and positions agree (run the Python server
twice against itself to see what "agree" means: 5-6 m median), the ADS-B scores are no worse, the
state files have the same shape, and CPU per replayed minute is clearly lower.

## 3. Shadow on the VPS (optional)

Replays only ever go to loopback or private addresses. On the VPS, bring the Rust container up on
spare ports (`MLAT_CLIENT_PORT=41090 MLAT_FILTERED_PORT=41003 MLAT_UNFILTERED_PORT=41004`,
`MLAT_WORK_DIR=./work-shadow`, a different `MLAT_CONTAINER_NAME`) and replay a fresh capture into
it with `mlat-capture replay capture.jsonl --target 127.0.0.1:41090 --speed 1`. Real receivers
are not involved. Watch `docker stats` for both containers.

## 4. Cut over

1. `docker compose build` the new image next to the running Python container.
2. Note the Python container's compose service, so it can be restarted.
3. Stop the Python container and start the new one on the same ports
   (`docker compose up -d`). Receivers were told `reconnect_in` ~10 s, so they are back within
   about 20 s; the first positions follow within a minute while clocks resynchronise.
4. Check:
   - `docker logs -f mlat-server`: `Handshake successful` lines, a `Status:` line every interval
   - `work/clients.json` lists about the same number of receivers as before
   - readsb receives Basestation lines (`nc localhost 31003`)
   - `docker stats`: CPU well below one core, memory flat
5. Roll back by stopping the new container and starting the old one: nothing else changed.

## 5. After a week

Compare `positions.csv` volume and the number of aircraft in `aircraft.json` with a day from the
Python era, and confirm the status line's outlier percentage is no worse.

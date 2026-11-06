# Deploying with Docker

    cp example.env .env       # edit: ports, work dir, CPUs, source URL
    mkdir -p work && sudo chown 10001 work
    docker compose build      # compiles on this machine, so it matches its CPU (x86-64 or arm64)
    docker compose up -d
    docker compose logs -f

The compose file publishes the receiver port (TCP and UDP), the Basestation ports and mounts the
work directory, which receives `sync.json`, `clients.json`, `aircraft.json` (rewritten every 15 s)
and `positions.csv`. `example.env` documents every setting; the entrypoint turns the `MLAT_*`
variables into command-line flags (anything else can be passed as `command:`).

To use the UDP transport set `MLAT_CLIENT_LISTEN=31090:31090`; the compose file already publishes
the UDP port. Receivers that offer it switch to it on their next connection.

Threads: one process uses every core it is given. Connection handling scales with
`MLAT_IO_THREADS` (default min(cores, 4)); `MLAT_ENGINE_THREADS` (default 1) spreads the position
engine over more matcher/pairer threads and only matters at several times today's load.

## Licence

The server is AGPL-3.0-or-later. Anyone who connects to it must be able to get its source: set
`MLAT_SOURCE_URL` to where this repository is published.

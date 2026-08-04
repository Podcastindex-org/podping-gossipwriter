# podping-gossipwriter

Broadcasts [Podping](https://podping.cloud) podcast feed notifications to a
decentralized [iroh-gossip](https://github.com/n0-computer/iroh-gossip) swarm.
It receives Cap'n Proto `PodpingWrite` messages from the podping.cloud
front-end over a ZeroMQ PAIR socket, signs each notification with an ed25519
key, archives it to SQLite, and broadcasts it on the gossip topic
`gossipping/v1/all`.

Runs alongside (not instead of) podping-hivewriter: podping.cloud dual-writes
to both. The front-end side is fire-and-forget — if this process is down,
Hive writes are unaffected.

## Provenance

Extracted 2026-07 from [Podcastindex-org/podping.alpha](https://github.com/Podcastindex-org/podping.alpha),
where the R&D history lives.

## Peer discovery

No DHT is used. Peer discovery is seed-based: 5 compiled-in podping.cloud
writer node IDs (overridable via `BOOTSTRAP_PEER_IDS`), persisted to
`KNOWN_PEERS_FILE` (capped at 15 entries) as new peers are seen, plus
periodic `PeerAnnounce` gossip messages that let already-connected peers
learn about each other.

## Build

Requires: Rust (stable), `capnproto`, `libzmq3-dev`, `libssl-dev`, `pkg-config`.

    cargo build --release -p gossip-writer

Docker:

    docker build -t podcastindexorg/podping-gossipwriter .

## Configuration (environment variables)

| Variable | Default | Purpose |
|---|---|---|
| `ZMQ_BIND_ADDR` | `tcp://0.0.0.0:9998` | PAIR socket bind for front-end messages |
| `IROH_SECRET_FILE` | `/data/gossip/iroh.key` | ed25519 message-signing key (created on first run) |
| `IROH_NODE_KEY_FILE` | `/data/gossip/iroh_node.key` | iroh transport node key (created on first run) |
| `ARCHIVE_ENABLED` | `false` | enable SQLite archive (`1`/`true`/`yes`) |
| `ARCHIVE_PATH` | `/data/gossip/archive.db` | SQLite archive location |
| `KNOWN_PEERS_FILE` | `/data/gossip/known_peers.txt` | cached peer list (max 15) |
| `BOOTSTRAP_PEER_IDS` | 5 podping.cloud writer nodes | comma-separated iroh node IDs to bootstrap from. Defaults to the stable podping.cloud writer nodes; set your own list to override, or an empty string to rely solely on `KNOWN_PEERS_FILE` and inbound connections |
| `TRUSTED_PUBLISHERS_FILE` | `/data/gossip/trusted_publishers.txt` | pubkeys whose messages peers trust |
| `TRUSTED_MONITORS_FILE` | `trusted_monitors.txt` | pubkeys allowed monitor access |
| `NODE_FRIENDLY_NAME` | (none) | human-readable name in PeerAnnounce |
| `PEER_ANNOUNCE_INTERVAL` | `300` | seconds between PeerAnnounce broadcasts |
| `PEER_ENDORSE_INTERVAL` | `600` | seconds between PeerEndorse broadcasts |
| `AUTO_TRUST_ENDORSEMENTS` | `false` | auto-trust endorsed publishers (`1`/`true`/`yes`) |

## ZMQ protocol

PAIR socket. Receives `PlexoMessage`-wrapped `PodpingWrite` (Cap'n Proto,
schemas from [podping-schemas-rust](https://github.com/Podcastindex-org/podping-schemas-rust) v1.1.0);
replies with a `PodpingHiveTransaction`-shaped acknowledgment so the
front-end treats it like a writer.

## Releasing

Tag `vX.Y.Z` on main → GitHub Actions builds and pushes
`podcastindexorg/podping-gossipwriter:X.Y.Z` and `:latest`.

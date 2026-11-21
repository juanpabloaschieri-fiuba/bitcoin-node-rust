# Bitcoin node & wallet in Rust

A Bitcoin **testnet** node written from scratch in Rust, with a GTK desktop wallet on top. It speaks the Bitcoin P2P protocol directly over TCP. No Bitcoin libraries are used beyond hashing, `secp256k1` signatures and Base58.

Built for *Taller de Programación I* (FIUBA, University of Buenos Aires), first semester 2023, by a team of four.

## What it does

**Node**
- Discovers peers through a testnet DNS seed and connects over IPv4 (IPv6 optional).
- Performs the `version`/`verack` handshake, outgoing and incoming. The node also accepts connections from other peers.
- **Initial Block Download:** fetches headers 2,000 at a time and switches peer on failure. It then downloads blocks **in parallel** through a worker pool, in bundles of 16. Workers that fail hand their bundle back to be retried, and `notfound` answers mark a peer as unreliable.
- Validates every block: **proof of work** (header hash vs. target) and **proof of inclusion** (rebuilding the Merkle root from the transactions).
- Keeps the **UTXO set** in a hash map. A vector was too slow for spend checks.
- Persists headers and blocks to binary files, so a restart only downloads what's new.
- Receives unsolicited blocks and transactions (`inv` → `getdata`), serves `getheaders` and block requests from peers, and propagates new blocks and transactions.
- Concurrent logger and a typed error hierarchy across the node.

**Wallet & UI**
- Several wallets, each with its balance, pending transactions and transaction history.
- Creates, signs and broadcasts transactions.
- Verifies a transaction's **Merkle proof of inclusion** in its block.
- Block explorer (previous/next block), and a loading screen that tracks the node's startup stages.
- The UI is the main thread. It talks to the node and wallet thread through a small request/response protocol, with MPSC channels one way and GLib channels the other.

## Architecture

```
            ┌──────────── UI (GTK, main thread) ────────────┐
            │  loading screen · overview · send · tx · blocks│
            └───────▲───────────────────────────┬───────────┘
          UIResponse│ (glib channel)   UIRequest │ (mpsc channel)
            ┌───────┴───────────────────────────▼───────────┐
            │              Node + Wallet thread              │
            │  blockchain · UTXO set · pending txs · wallets │
            └───────▲───────────────────────────┬───────────┘
                    │ shared state (Arc<Mutex>)  │
            ┌───────┴───────────────────────────▼───────────┐
            │   Peer communicator: one worker per peer (TCP) │
            └────────────────────────────────────────────────┘
```

## My part

I wrote the most commits on the team (145 of ~345). Most of the networking and the node core came from those commits:
- `version` message, handshake (later split into incoming and outgoing), and a mock TCP stream for testing it.
- Initial Block Download, the parallel block downloader and its retry/EOF protocol, and binary persistence.
- Proof of work, proof of inclusion, transactions inside blocks, the UTXO set and pending transactions.
- Sending transactions, the UI ↔ wallet protocol and the loading screen.
- The peer-communicator refactor: incoming peers, timeouts, answering `getheaders`, serving blocks, propagation and disconnection handling.

## Running it

Requires Rust and the GTK 3 development libraries.

```sh
unset GTK_PATH            # avoids GTK startup conflicts
cargo run -- node/nodo.conf
```

`node/nodo.conf` sets the protocol version, local address, DNS seed, starting date and the data/log file paths.

## Team

Juan Pablo Aschieri · Marco Natalini · Francisco Rovira Rossel · Julián Stiefkens

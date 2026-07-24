# 445 — A Peer-to-Peer Multiplayer Card Game

[![Java](https://img.shields.io/badge/Java-17-orange?logo=openjdk)](https://openjdk.org/)
[![JavaFX](https://img.shields.io/badge/JavaFX-24.0.1-blue?logo=java)](https://openjfx.io/)
[![Build](https://img.shields.io/badge/build-Maven-C71A36?logo=apachemaven)](https://maven.apache.org/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](./LICENSE.txt)

**445** is a UNO-style multiplayer card game built without a central game server. Instead of routing every move through a backend, up to four peers connect directly over **UDP**, and one peer acts as the game host — a role that is re-elected automatically at runtime using a from-scratch implementation of the **Raft consensus algorithm**. The project was built as a systems-programming exercise in distributed consensus, custom network protocol design, and fault tolerance, wrapped in a JavaFX desktop client.

> Why this project is interesting: most student "multiplayer" projects use a client-server model with a framework doing the heavy lifting (sockets, reliability, discovery). This one implements reliable delivery, packet fragmentation/reassembly, symmetric encryption, and leader election **from the transport layer up**, using nothing but raw `DatagramChannel`/`DatagramSocket` APIs.

---

## Table of Contents

- [Highlights](#highlights)
- [Architecture](#architecture)
- [Network Protocol Design](#network-protocol-design)
- [Fault Tolerance: Raft-Based Leader Election](#fault-tolerance-raft-based-leader-election)
- [Security: Packet Encryption](#security-packet-encryption)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Game Rules](#game-rules)
- [Engineering Notes & Known Limitations](#engineering-notes--known-limitations)
- [Roadmap](#roadmap)
- [Contributors](#contributors)
- [License](#license)

---

## Highlights

| Area | What was built |
|---|---|
| **Networking** | Custom binary protocol over `DatagramChannel`/`DatagramSocket` (UDP) — no HTTP, no RPC framework |
| **Reliability over UDP** | Hand-rolled ACK/retry loop and block-numbered fragmentation to move `GameState` objects larger than a single UDP datagram |
| **Distributed Consensus** | A working subset of the **Raft algorithm** (terms, follower/candidate/leader states, randomized election timeouts, majority vote tallying) so the game continues if the host disconnects |
| **Security** | AES-CTR packet encryption with per-message IVs (`Encryption.java`) |
| **Desktop UI** | JavaFX + FXML views (`MainView`, `RoomView`) backed by dedicated controllers, styled with CSS |
| **Serialization** | Game state is serialized, chunked into MTU-safe packets, transmitted, reassembled, and deserialized on the receiving peer |
| **Build Tooling** | Maven Shade plugin produces independently runnable `Client.jar` and `Server.jar` artifacts from one codebase |

---

## Architecture

There is no dedicated backend server. One of the players' machines runs a `Host` process; every other player runs a `Client` process that connects directly to the host's public IP/port.

```
                     ┌─────────────────────────┐
                     │         Host            │
                     │  (Player 0 / Leader)     │
                     │                          │
                     │  DatagramChannel  (game)  │
                     │  DatagramSocket   (raft)  │
                     └────────────┬─────────────┘
                UDP (game state)  │  UDP (raft: heartbeat / votes)
        ┌──────────────┬──────────┴───────────┬──────────────┐
        ▼              ▼                      ▼              ▼
   ┌─────────┐    ┌─────────┐            ┌─────────┐   ┌─────────┐
   │ Client 1│    │ Client 2│            │ Client 3│   │  ...    │
   └─────────┘    └─────────┘            └─────────┘   └─────────┘
```

- **`Host`** owns the authoritative `GameState`, runs the lobby (`open_lobby`), assigns player IDs, broadcasts state to all clients, and starts the game once enough players join.
- **`Client`** connects to the host, sends/receives `GameState` updates, and separately participates in the **Raft protocol** over its own socket so that if the host disconnects, the remaining peers can elect a new leader without any external coordinator.
- **`User`** (base class) holds shared connection state (`id`, `gameState`, Raft term/vote fields) inherited by both `Host` and `Client`.
- **`GameState`** is the single serializable source of truth (deck, discard pile, per-player hands, turn order, skip/reverse/stacking flags) — mutated locally and re-synced across the network.
- **JavaFX UI layer** (`MainApp`, `SceneSwitcher`, `MainController`, `RoomViewController`) drives scene transitions between the main menu and the game room, and instantiates a `Host` or `Client` depending on whether the user creates or joins a game.

---

## Network Protocol Design

Because UDP provides no ordering, delivery guarantees, or connection state, all of that is implemented manually on top of it (`Packet.java`):

**Wire format for a game-state packet:**

```
+------------+-----------+-------------+------------+
|  OP-CODE   | BLOCK-NUM | ENCRYPTION  |    DATA    |
+------------+-----------+-------------+------------+
| 2 bytes    | 2 bytes   | 16 bytes    | variable   |
+------------+-----------+-------------+------------+
```

- **Fragmentation/reassembly**: `GameState` is Java-serialized, then sliced into `DATA_SIZE`-byte blocks, each tagged with a monotonically increasing `block_num`. The receiver buffers blocks keyed by number, and once a short (final) block arrives, sorts the keys and concatenates the payload back into a byte stream for deserialization — a simplified TFTP-style transfer scheme.
- **Reliability**: every data packet must be ACKed by the receiver before the sender advances; if no ACK arrives within a timeout, the packet is resent (bounded retry loop), giving UDP a thin layer of TCP-like reliability without the overhead of a full TCP handshake per game update.
- **Opcodes**: a single `enum Opcode` (`JOIN`, `START`, `UPDATE`, `HEARTBEAT_REQUEST`, `HEARTBEAT`, `RECONNECT`, `GAME_OVER`, `VOTE_REQUEST`, `VOTE_GRANTED`, `VOTE_DENIED`) multiplexes every message type the protocol needs, all parsed from the first 2 bytes of every datagram.
- **Two independent channels per peer**: a game-state channel (`DatagramChannel`, non-blocking) and a dedicated Raft channel (`DatagramSocket`) so consensus traffic (heartbeats/votes) is never blocked behind large state transfers.

---

## Fault Tolerance: Raft-Based Leader Election

The most technically ambitious piece of this project: rather than assume the host never disconnects, each `Client` runs a **modified Raft state machine** to detect a dead leader and elect a new one — entirely peer-to-peer, with no external arbiter.

**State machine:** `FOLLOWER → CANDIDATE → LEADER`

1. Every follower runs a randomized election timer (2–4s jitter, to avoid split votes) and periodically requests a heartbeat from peers.
2. If no heartbeat is acknowledged before the timeout, the follower transitions to `CANDIDATE`, increments its term, votes for itself, and broadcasts a `VOTE_REQUEST` to every other peer.
3. Peers grant a vote only once per term and only to the first eligible candidate they see in that term (classic Raft safety property — this is what prevents deadlock from every node voting for itself).
4. Once a candidate collects votes from a strict majority of players (`n/2 + 1`), it transitions to `LEADER` and immediately broadcasts a heartbeat, which implicitly tells every other peer who the new leader is.
5. A real `LEADER` never starts an election against itself, and terms strictly increase to prevent stale leaders from re-asserting authority.

This mirrors the term/vote/heartbeat mechanics of the [Raft consensus paper](https://raft.github.io/raft.pdf) (Ongaro & Ousterhout), scoped down to leader election (log replication is not implemented, since `GameState` is broadcast wholesale rather than as an operation log).

## Security: Packet Encryption

`Encryption.java` implements **AES-CTR** symmetric encryption for packet payloads:

- 128-bit AES key generated via `KeyGenerator`
- A fresh, cryptographically random 16-byte IV (`SecureRandom`) is generated per message and prepended to the ciphertext, so no two messages are encrypted under the same keystream
- `Cipher`-based encrypt/decrypt helpers isolate all crypto logic from the transport code, so it can be dropped into any packet type

---

## Project Structure

```
├── README.md
├── pom.xml                          # Maven build config — produces Client.jar & Server.jar via maven-shade-plugin
├── mvnw / mvnw.cmd                  # Maven wrapper (no local Maven install required)
└── src
    ├── main
    │   ├── java/com/mycompany/app
    │   │   ├── communication
    │   │   │   ├── Host.java        # Authoritative game host: lobby, state broadcast, Raft listener
    │   │   │   ├── Client.java      # Peer client: join flow, state sync, full Raft state machine
    │   │   │   ├── User.java        # Shared base class (id, GameState, Raft fields)
    │   │   │   ├── Encryption.java  # AES-CTR encrypt/decrypt helpers
    │   │   │   └── Main.java
    │   │   ├── model
    │   │   │   ├── GameState.java   # Turn order, deck/discard piles, win/skip/reverse/stack logic
    │   │   │   ├── Packet.java      # Wire protocol: framing, fragmentation, opcodes, (de)serialization
    │   │   │   ├── Player.java
    │   │   │   ├── Card.java
    │   │   │   ├── Shape.java / Value.java
    │   │   ├── testingenvironment  # Config-driven mock Host/Client for local protocol testing
    │   │   └── ui
    │   │       ├── MainApp.java             # JavaFX entry point, scene wiring
    │   │       ├── SceneSwitcher.java
    │   │       └── uiController/            # MainController, RoomViewController
    │   └── resources
    │       ├── fxmlViews/          # MainView.fxml, RoomView.fxml
    │       ├── styles/mainView.css
    │       └── cardImages/         # Card art assets (not checked in — see setup below)
    └── test/java/...               # JUnit 5 tests
```

---

## Getting Started

### Prerequisites
- **JDK 17+**
- No local Maven install needed — this repo ships the Maven Wrapper (`./mvnw`)

### 1. Clone and add card art assets
```bash
git clone <this-repo-url>
cd 445-Card-Game
mkdir -p src/main/resources/cardImages
```
Download the card image set from [this asset bundle](https://drive.google.com/file/d/1wNBbTLSjaluWTWW3rw3SRt3pTWFQZWLl/view?usp=sharing) and extract it into `src/main/resources/cardImages`.

### 2. Build
```bash
./mvnw clean install
```
This compiles the project and packages two standalone executables via `maven-shade-plugin`:
- `target/Client.jar` — runnable client peer
- `target/Server.jar` — runnable host peer

### 3. Run the JavaFX desktop client
```bash
./mvnw javafx:run
```
From the main menu you can **host** a new game (spins up a `Host`, opens the lobby, waits for up to 4 players) or **join** an existing game by entering the host's IP/port (spins up a `Client` and connects).

---

## Game Rules

**445** plays like a classic UNO-family shedding game:

- Each player starts with **7 cards**.
- On your turn, play a card matching the **color** or **number** of the top discard.
- Can't play a valid card? Draw one from the deck instead.
- First player to empty their hand wins.

**Special cards:**
| Card | Effect |
|---|---|
| ⏭️ Skip | Next player loses their turn |
| 🔁 Reverse | Reverses turn order |
| +2 Draw Two | Next player draws 2 and loses their turn |
| 🌈 Wild | Playable on anything; choose the next color |
| +4 Wild Draw Four | Next player draws 4, loses their turn, and you choose the color |

---

## Engineering Notes & Known Limitations

In the interest of transparency (and because this is as much a systems-learning project as a shippable game), a few things are intentionally incomplete:

- Raft here covers **leader election** only — there is no replicated log, so the full `GameState` is retransmitted wholesale on each sync rather than as an append-only operation log.
- The lobby's join-handling currently accepts any inbound datagram as a join request; a proper opcode check is a known follow-up (flagged in `Host.open_lobby`).
- `update_clients`/`receive_update` use a simple linear resend loop rather than a windowed protocol (e.g. sliding-window or selective repeat), which is a reasonable place to extend the reliability layer further.
- Packet-level AES-CTR encryption (`Encryption.java`) is implemented and unit-testable, but not yet wired into the live `Packet` send/receive path end-to-end.

## Roadmap

- [ ] Wire `Encryption` into the live packet pipeline (encrypt-on-send / decrypt-on-receive)
- [ ] Replace linear resend with a windowed/selective-repeat reliability scheme
- [ ] Persist and replicate a Raft log for full state-machine replication, not just leader election
- [ ] NAT traversal / hole-punching for players not on the same network
- [ ] Automated integration tests simulating packet loss and host failover

---

## Contributors

Built for CSC 445 (Distributed Systems) by:
- Phone Pyae Sone Phyo ([@Soney](https://github.com/))
- Crislenny Uceta
- Saurav Lamichhane
- Scott Wilmot

## License

MIT — see [LICENSE.txt](./LICENSE.txt).

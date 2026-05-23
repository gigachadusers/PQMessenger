# ⬡ PQMessenger

> **Post-Quantum Secure Messenger** — A fully encrypted, peer-to-peer, decentralized desktop chat application built in a single C++ file targeting Windows 10/11. No servers. No cloud. No external dependencies.

---

## Screenshot / Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ ⬡ PQMessenger        ● E2E ENCRYPTED    TOKEN: A3F91C2B   PEERS: 2         │
├─────────────┬──────────────────────────────────────────┬────────────────────┤
│ ONLINE PEERS│                                          │ SECURITY EVENTS    │
│             │  ┌──────────────────────────────────┐   │ 23:14:09 CRYPTO    │
│ ● Ghost     │  │ 👤 Ghost   23:14:08  🔒          │   │ AES key rotated    │
│   12ms      │  │  Hey, can you read this?          │   │                    │
│             │  └──────────────────────────────────┘   │ 23:14:10 PEER      │
│ ● Shadow    │  ┌──────────────────────────────────┐   │ shadow joined      │
│   8ms       │  │ 👤 Shadow  23:14:10  🔒          │   │                    │
│             │  │  Loud and clear, fully encrypted  │   │ ENTROPY            │
│             │  └──────────────────────────────────┘   │ ▂▇▃▅▆▂▇▄▃▅▆▂▇▄▃▅  │
│ YOU:        │                                          │                    │
│ 👤 Ghost    │  Shadow is typing...                     │                    │
├─────────────┴──────────────────────────────────────────┴────────────────────┤
│ [Host] [Connect] │ Type a message...                         │    Send      │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Features

### 🔐 Cryptography
- **AES-256-GCM** authenticated encryption on every message
- **Post-Quantum inspired key exchange** — lattice-based encapsulation (KEM), noise injection, hybrid classical + pseudo-PQC design
- **HKDF** (HMAC-based key derivation) using SHA-256 via BCrypt/CNG
- **Ephemeral session keys** generated fresh per peer session
- **Replay attack protection** via per-peer seen-sequence-number sets
- **Nonce generation** using `BCryptGenRandom`
- **SecureZeroMemory** used throughout to wipe sensitive buffers
- Message integrity via GCM authentication tags

### 🌐 Networking
- **UDP peer-to-peer** — no central server required
- **Local network auto-discovery** via UDP broadcast on startup
- **Manual peer connection** via IP:Port or invite code
- **NAT traversal** via direct UDP (hole-punch-friendly architecture)
- **Room tokens** — 8-byte random hex room IDs
- **Heartbeat system** — detects peer disconnection within ~12 seconds
- **Packet resend / ACK system** — reliability layer for critical packets
- **Ping measurement** — live latency shown per peer
- Custom **binary packet protocol** with CRC32 validation

### 💬 Chat
- Scrollable real-time chat with sender colors and timestamps
- Typing indicators broadcast to all peers
- Online/offline detection
- Message history (last 500 messages)
- Encrypted status shown per message (🔒)
- Own messages shown right-aligned

### 🖥 GUI
- Native Win32 **dark cyberpunk interface** — no Qt, no Electron
- Double-buffered rendering (no flicker)
- Gradient headers, custom panels, color-coded usernames
- Animated encryption status indicator
- Live **entropy visualization** in the right panel
- Live **bandwidth monitor** (TX/RX bytes)
- Real-time **peer list** with ping display

### 🧵 Threading
- `RecvLoop` — dedicated UDP receive thread
- `HeartbeatLoop` — connection management, ping, timeout, packet retry
- GUI runs on main thread via Win32 message loop
- All shared state protected with `std::mutex`

### 🐞 Debug Terminal
- Separate debug window with ASCII art header
- Color-coded log entries by category: `[NET]` `[CRYPTO]` `[PEER]` `[WARN]` `[ERR]` `[THREAD]` `[SOCK]`
- Scrollable, timestamped log
- Shows all: packet events, key exchanges, peer joins/leaves, errors, NAT attempts

### 👤 Profile System
- Startup identity dialog — choose username, display color, avatar letter
- Color palette with 8 neon options
- Persistent session identity broadcast to all peers

---

## Build Instructions

### MSVC (Visual Studio / Developer Command Prompt)

```cmd
cl /EHsc /W3 /O2 /std:c++17 pqmessenger.cpp ^
   /link ws2_32.lib bcrypt.lib crypt32.lib winhttp.lib ^
        gdi32.lib gdiplus.lib user32.lib kernel32.lib ^
        comctl32.lib iphlpapi.lib
```

### GCC / MinGW-w64

```bash
g++ -std=c++17 -O2 -o pqmessenger.exe pqmessenger.cpp \
    -lws2_32 -lbcrypt -lcrypt32 -lwinhttp \
    -lgdi32 -lgdiplus -luser32 -lkernel32 \
    -lcomctl32 -liphlpapi -mwindows
```

> **Requirements:** Windows 10/11, MSVC 2019+ or MinGW-w64 with Windows SDK

---

## How To Use

### Host a Room
1. Launch PQMessenger
2. Set your username and color
3. Click **Host** — the app binds a UDP port and generates a room token
4. Share your **IP address + port** (shown in the top bar) with your peer

### Join a Room
1. Launch PQMessenger
2. Click **Connect**
3. Enter the host's IP address and port
4. Keys are exchanged automatically — start chatting

### LAN / Local Network
The app broadcasts a `HELLO` packet on startup to discover peers on the same local network automatically.

---

## Packet Protocol

| Field         | Size     | Description                          |
|---------------|----------|--------------------------------------|
| `magic`       | 4 bytes  | `0x50514D53` ("PQMS")               |
| `version`     | 1 byte   | Protocol version (1)                 |
| `type`        | 1 byte   | Packet type enum                     |
| `flags`       | 2 bytes  | Type-specific flags                  |
| `seq`         | 4 bytes  | Monotonic sequence number            |
| `ack`         | 4 bytes  | Acknowledged sequence                |
| `payload_len` | 2 bytes  | Encrypted payload length             |
| `peer_id`     | 16 bytes | Sender peer identifier               |
| `nonce`       | 12 bytes | AES-GCM nonce                        |
| `tag`         | 16 bytes | AES-GCM authentication tag           |
| `crc`         | 4 bytes  | CRC32 of entire packet               |
| `payload`     | variable | Encrypted message content            |

**Packet Types:** `HELLO` `KEY_EXCHANGE` `MESSAGE` `HEARTBEAT` `USER_JOIN` `USER_LEAVE` `ACK` `RESEND_REQUEST` `TYPING` `PING` `PONG` `RELAY_REQ` `RELAY_FWD` `STUN_BIND`

---

## PQC Key Exchange Design

```
[Alice]                              [Bob]
  │                                    │
  │── Generate PQC Keypair ────────────►│
  │   lattice_A, lattice_b, secret_s   │
  │                                    │
  │◄── Send PQCPublicKey + Ciphertext ──│
  │    (KEM encapsulation)             │
  │                                    │
  │── PQCDecapsulate ─────────────────►│
  │   HKDF(shared_secret) = SessionKey │
  │                                    │
  │◄──────── AES-256-GCM messages ─────►│
```

- Lattice dimension: **N=256**
- Noise parameter: small bounded error terms (LWE-style)
- Key derivation: **HKDF-SHA256** with domain label `pqc-kem-v1`
- Session key: **256-bit AES key** derived via `aes256gcm-v1` label
- All intermediate secrets wiped with `SecureZeroMemory`

---

## Security Properties

| Property               | Implementation                        |
|------------------------|---------------------------------------|
| Confidentiality        | AES-256-GCM per message               |
| Integrity              | GCM authentication tag                |
| Forward Secrecy        | Ephemeral session keys per peer       |
| Replay Protection      | Per-peer seen sequence number set     |
| Key Exchange           | PQC-inspired lattice KEM + HKDF       |
| Random Number Source   | `BCryptGenRandom` (Windows CNG)       |
| Memory Safety          | `SecureZeroMemory` on all key material|

> ⚠️ **Disclaimer:** The PQC layer is an architecturally representative post-quantum design. It is NOT NIST FIPS 203 (ML-KEM) compliant and should not be used for certified high-security deployments without a certified PQC library.

---

## Project Structure (Single File)

```
pqmessenger.cpp
├── GLOBALS          — App-wide constants, window handles, colors
├── LOGGING          — Thread-safe timestamped log system
├── CRYPTOGRAPHY     — AES-256-GCM, PQC KEM, HKDF, BCrypt wrappers
├── USER SYSTEM      — Identity, peer registry, avatar, color
├── PACKET SYSTEM    — Binary protocol, CRC32, builder/parser
├── MESSAGE SYSTEM   — Chat history, append, snapshot
├── NETWORKING       — UDP socket, recv loop, heartbeat, connect/host
├── GUI HELPERS      — Drawing primitives, fonts, gradients
├── DEBUG WINDOW     — ASCII art terminal, scrollable color log
├── PROFILE DIALOG   — Username / color setup on startup
├── CONNECT DIALOG   — IP:Port connection input
├── MAIN WINDOW      — Full chat UI, panels, controls
└── MAIN             — WinMain entry, WSA init, GDI+, message loop
```

---

## License

MIT License — free to use, modify, and distribute.

---

*Built with pure Win32 API, BCrypt/CNG, Winsock2. No external dependencies.*

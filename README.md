# Archived notes – Zero-Trust Invite-Only P2P Voice & Chat

> **Historical document.** ConquerD is now **DoubleSlash**, and this file describes the
> retired Python application, not the shipping Rust client. It is kept for history only;
> do not treat it as a source of truth for current product behavior.
>
> Current website: [doubleslash.space](https://doubleslash.space) ·
> Current documentation: [`README.md`](https://github.com/ConquerD/DoubleSlash/blob/develop/README.md)
> and [`agents.md`](https://github.com/ConquerD/DoubleSlash/blob/develop/agents.md) in the main repository.

Conquerd is a privacy-first peer-to-peer voice and chat application. Identity, discovery, and trust live entirely on your device — there is no central server, no account, no sign-up. Peers connect through cryptographically signed invite links and communicate directly over encrypted QUIC channels.

No telemetry. No cloud accounts. No third-party infrastructure required.

---

---

## Features

### Chat-First P2P
- Text messaging is the primary interaction after connecting with a peer.
- Voice calls are opt-in, initiated from within a conversation — never from a contacts list.
- Per-conversation scroll position persistence and floating "↓ Latest" button.
- Typing indicators with 15-second auto-hide.
- Unread badges on taskbar and system tray notifications.
- Chat history persisted to SQLite with delivery states (sending → sent → delivered).

### Voice Calls
- Low-latency Opus audio over QUIC peer-to-peer transport (Rust [quinn](https://github.com/quinn-rs/quinn)).
- Push-to-talk and voice activation modes with fade-in/fade-out.
- Real-time noise suppression (spectral-gate FFT, Rust-native).
- Fixed-depth jitter buffer (60ms default, tuneable 1–20 frames) with silence-to-audio de-click.
- Incoming call ringtone + overlay dialog with Answer, Decline, and Block options.
- Session status banner accurately reflects transport mode (direct P2P vs. relay).

### Rooms (Multi-Peer Voice)
- SFU (Selective Forwarding Unit) room hosting on volunteer supernodes.
- QUIC relay transport for NAT-traversed room audio; WebSocket for room membership signaling only.
- Room parity with direct-peer features: chat, voice, file transfer.
- Peer room invites with accept/decline flow.
- Up to 8 participants per room.

### Security & Identity
- Cryptographic identity via long-term Ed25519 keys with derived peer IDs (SHA-256).
- Invite-only discovery through signed `conquerd://` links (timestamped, expiry-checked).
- Forward-secret handshakes using ephemeral X25519 + HKDF + AES-GCM.
- All signaling is Ed25519-signed, transcript-bound, and replay-resistant (sliding-window counter bitmap).
- Peer revocation with propagation (socket drop, relay eject, SFU eject).
- Local trust graph — successful handshakes persist to local peer store.
- Optional release-signed P2P updates with Ed25519 signatures and threshold validation.

#### Supply-Chain Trust
- **Code Signing**: Windows (SignPath.io), macOS (Apple Developer Program), all platforms (GitHub Sigstore attestations).
- **Release Manifest**: Signed JSON listing official builds (version, platform, build_hash) verified by installer.
- **Peer Attestation**: Runtime challenges where peers prove they're running official builds via nonce-signed claims.
- **Policy Enforcement**: Configurable attestation policy (off/warn/strict) gating relay access for unverified peers.
- **CI/CD Hardening**: GitHub Actions pinned to immutable commit SHAs to prevent supply-chain attacks.

### NAT Traversal
- Multi-layer connection strategy: QUIC direct → WebSocket → UDP hole punch → supernode relay.
- STUN public IP discovery (Google `stun.l.google.com`, `stun1.l.google.com`; Cloudflare `stun.cloudflare.com`).
- UPnP automatic port mapping.
- UDP hole punching for peers behind non-symmetric NATs.
- Supernode-coordinated hole punch for timing alignment.
- QUIC relay fallback through trusted supernodes.

### Desktop Application
- Modern dark theme with DPI-aware scaling (125%, 150%, 200%+).
- First-run onboarding wizard (display name, identity fingerprint + QR, optional supernode).
- `conquerd://` URI scheme for one-click invite joining.
- Invite QR codes with toggle display and save-to-PNG.
- System tray with badge notifications for unread messages and missed calls.
- Collapsible event log panel (toggle with `Ctrl+B`).
- Dedicated Nodes tab for managing supernodes with right-click context menu.
- Handles / display names broadcast to all peers and shown everywhere (chat, calls, event log).

### P2P Updates
- Application updates delivered directly peer-to-peer — no update server, no CDN.
- Portable exe users receive Python source updates from connected peers without rebuilding.
- Binary delta patching and zlib compression for efficient transfer.
- Optional release-key signing with threshold validation and revocation lists.

---

## Quick Start

### Prerequisites
- Python 3.10+
- A working audio input/output device
- No server, no account, no sign-up required

### Install
```bash
pip install -e ".[dev]"
```

### Run
```bash
python -m client_desktop
```

### Connect Two Peers

```bash
# Terminal 1
ConquerD_a.bat

# Terminal 2
ConquerD_b.bat
```

1. Client A: click **Create Invite** → link copied to clipboard.
2. Client B: paste link into the invite input and press Enter.
3. Both clients see each other in the trusted peers sidebar.
4. Select a peer to open chat → send messages.
5. Click **Start Call** in the chat header for voice.

On first launch, an onboarding wizard walks you through choosing a display name, viewing your identity fingerprint and QR code, and optionally configuring a supernode.

---

## Platform Support & Installation

| Platform | Package | URI Scheme |
|----------|---------|------------|
| Windows  | Rust installer or portable folder | Registry (`conquerd://`) |
| macOS    | `.app` bundle + `.dmg` | `CFBundleURLTypes` in Info.plist |
| Linux    | AppImage | `.desktop` file + `xdg-mime` |

### Windows
Run `conquerd-installer.exe` or extract the portable `conquerd/` folder. The installer registers the `conquerd://` URI scheme, creates Start Menu shortcuts, and supports silent upgrades (`--silent`) and uninstallation (`--uninstall`).

### macOS
Open the `.dmg` and drag Conquerd to Applications. Grant microphone access when prompted.

### Linux
```bash
chmod +x ConquerD-x86_64.AppImage
./ConquerD-x86_64.AppImage
```

To register the `conquerd://` URI scheme:
```bash
cp packaging/conquerd.desktop ~/.local/share/applications/
update-desktop-database ~/.local/share/applications/
xdg-mime default conquerd.desktop x-scheme-handler/conquerd
```

### Uninstalling

**Windows (installer):** Open *Add or Remove Programs* (Settings → Apps → Installed apps), search for **ConquerD**, and click Uninstall. Alternatively, run `conquerd-installer.exe --uninstall` from the command line for a silent uninstall.

**Windows (portable):** Delete the extracted `conquerd\` folder. No registry keys are written by the portable version.

**macOS:** Drag the ConquerD app from Applications to the Trash. User data in `~/.conquerd/` can be removed manually if desired.

**Linux (AppImage):** Delete the `.AppImage` file. If you registered the URI scheme, remove `~/.local/share/applications/conquerd.desktop` and run `update-desktop-database ~/.local/share/applications/`. User data in `~/.conquerd/` can be removed manually.

### System Requirements
- **OS:** Windows 10+, macOS 10.15+, Linux (glibc 2.31+)
- **Python:** 3.10+ (source installs only; bundled in packaged releases)
- **Audio:** Working microphone and speakers/headphones
- **Network:** Internet connection for P2P (LAN-only mode also available)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Client A                            Client B                   │
│  ┌──────────┐   invite link    ┌──────────┐                    │
│  │ Identity  │ ──────────────► │ Identity  │                    │
│  │ PeerStore │ ◄── handshake ──│ PeerStore │                    │
│  │ Signaling │ ◄── ws:// ─────►│ Signaling │                    │
│  │ Chat      │ ◄── messages ──►│ Chat      │                    │
│  │ QUIC      │ ◄── voice ─────►│ QUIC      │                    │
│  └──────────┘                  └──────────┘                    │
│         Optional: STUN / supernode QUIC relay (rooms)           │
└─────────────────────────────────────────────────────────────────┘
```

### Language Boundaries

| Layer | Language / Runtime |
|---|---|
| UI, app logic, signaling, chat, call state, settings | Python (PySide6) |
| Audio I/O, DSP, Opus codec, VAD, jitter buffer | **Rust** (`conquerd_audio` — PyO3, compiled extension) |
| Cryptographic primitives (Ed25519, X25519, Argon2id, AES-256-GCM, HKDF, session cipher) | **Rust** (`conquerd_crypto` — PyO3, compiled extension) |
| QUIC transport, relay client, hole-punch, STUN | **Rust** (`conquerd_quic` — PyO3, compiled extension) |
| Capability registry, `FeatureModule` trait, quota enforcement, auth-tier gating | **Rust** (`conquerd-features` — rlib linked into supernode; Python mirror in `client_shared/`) |
| Supernode relay server (SFU + QUIC relay + portal) | **Rust** (`conquerd-supernode` — standalone binary) |
| Installer / updater | **Rust** (`conquerd-installer` — standalone binary) |

The two PyO3 extension modules (`conquerd_audio`, `conquerd_quic`) are required and have no Python fallback path. They are linked at app startup and called directly from Python via native bindings.

### Transport Stack
- **Direct calls**: QUIC peer-to-peer via `QUICPeerManager` (conquerd_quic / quinn).
- **Room audio**: QUIC relay (`QUICRelayClient` → supernode `QUICRelay`); WebSocket used for room membership signaling only.
- **Signaling/chat**: Ed25519-signed, transcript-bound messages; prefers QUIC signaling stream when a peer session is connected, falls back to WebSocket.
- **Relay**: QUIC relay protocol on supernodes (transport-only; no app-layer decryption).

### Core Model
- **Invite-only discovery**: peers connect only from signed `conquerd://` links.
- **Zero trust relay**: relays forward signed/encrypted payloads only — no app-layer central services.
- **Cryptographic identity**: long-term Ed25519 identity key; `peer_id` = SHA-256 of public key.
- **Forward secrecy**: invite handshakes use ephemeral X25519 + HKDF + AES-GCM.
- **Local trust graph**: successful handshakes persist to local peer store.
- **Endpoint stability**: signaling port persisted across restarts; peers notified of changes via `ENDPOINT_UPDATE`.
- **Auto-connect**: trusted peers can be marked for automatic reconnection on startup (30-second retry, up to 5 attempts).

### Key Data Flows

**Direct voice call (1-on-1)**
```
CPAL capture (Rust)
  → NoiseSuppressor (Rust, spectral-gate FFT)
  → VoiceActivityDetector (Rust) → speaking_changed signal → UI level meter
  → OpusEncoder (Rust, 20ms frames, 48 kHz mono)
  → QUICPeerManager.send_audio() → QUICTransport.send_audio_datagram()
  → QUIC unreliable datagram [2-byte seq][opus payload]
  ──────────────────────────────── (network) ────────────────────────────────
  → QUICTransport.on_audio_datagram callback
  → JitterBuffer.push(seq, opus)   ← 3-frame / 60ms reorder buffer (Rust)
  → OpusDecoder.decode() → PCM
  → AudioEngine playback mix (CPAL, Rust)
```

**Room audio (multi-peer via supernode)**
```
OpusEncoder (Rust)
  → QUICRelayClient.send_audio([0xFF][opus])   ← broadcast index
  ──────→ supernode QUICRelay
           distributes [peer_idx][opus] to each room member
  ──────→ QUICRelayClient.on_audio_received([peer_idx][opus])
  → OpusDecoder (per sender) → playback

Room membership (join/leave/state) flows over WebSocket only.
```

**Chat message**
```
User types → ChatManager.send_message()
  → SQLite (status = SENDING)
  → ConnectionManager → QUIC stream 0 or WebSocket
    (Ed25519-signed + AES-GCM encrypted with session cipher)
  → Peer verifies signature → sends CHAT_ACK
  → ChatManager updates status → DELIVERED
```

### Rust Extension API Summary

**`conquerd_audio`** (CPAL + Opus + DSP — sources: `engine.rs`, `codec.rs`, `vad.rs`, `noise.rs`, `jitter.rs`)

| Export | Type | Role |
|---|---|---|
| `AudioEngine` | Class | CPAL capture/playback; `set_on_audio_frame(cb)` fires every 20ms |
| `OpusEncoder` | Class | `encode(pcm: bytes) → bytes` — 960 samples → 20–40 byte Opus frame |
| `OpusDecoder` | Class | `decode(bytes) → bytes` — Opus → PCM |
| `JitterBuffer` | Class | 3-frame (60ms) reorder buffer; `push(seq, opus)` / `pop()` |
| `NoiseSuppressor` | Class | Wiener FFT denoising on capture thread; `suppress(pcm) → bytes` |
| `VoiceActivityDetector` | Class | Energy + spectral VAD; `analyze(pcm) → VadResult`, `get_level() → float` |
| `VadResult` | Dataclass | `is_speaking: bool`, `level: float` |
| Constants | — | `SAMPLE_RATE=48000`, `FRAME_DURATION_MS=20`, `FRAMES_PER_BUFFER=960` |

**`conquerd_quic`** (quinn + tokio + Ed25519 — sources: `transport.rs`, `peer_manager.rs`, `relay_client.rs`, `file_channel.rs`, `hole_punch.rs`, `stun.rs`)

| Export | Type | Role |
|---|---|---|
| `QUICTransport` | Class | Local QUIC endpoint; `bind()`, `connect_to_peer()`, `send_audio_datagram()`, `send_signaling()` |
| `QUICPeerManager` | Class | High-level peer audio; `connect_peer()`, `send_audio()`, callbacks for state and frames |
| `QUICRelayClient` | Class | Room audio relay to supernode; `connect()`, `send_audio()`, `disconnect()` |
| `QUICFileChannel` | Class | File transfers over QUIC bidirectional streams; `open()`, `send_chunk()`, `close()` |
| `HolePunchManager` | Class | UDP hole-punch coordination; `register_punch()`, `execute_punch()` |
| `PeerState` | Enum | `NEW → CONNECTING → CONNECTED → DISCONNECTED → FAILED → CLOSED` |
| `stun_get_public_ip(server)` | Function | STUN query → `(ip, port)` |
| Constants | — | `SIGNALING_STREAM_ID=0`, `BROADCAST_INDEX=0xFF`, `IDLE_TIMEOUT_S=30.0` |

---

## Modular Framework

Conquerd is structured as a **modular peer-connectivity framework**: chat, voice, files, rooms, and games are not hard-coded behaviors but **features** advertised and negotiated between peers and supernodes. The spine is the `conquerd-features` crate (with mirrored Python bindings in `client_shared/capabilities.py`).

### Concepts

- **Capability descriptor** — a self-describing record advertised after handshake. Fields: `id` (reverse-DNS, e.g. `core.chat.v1`), `version` (semver, negotiated by major), `kind` (`datagram` | `stream` | `request`), `params` (free-form), `auth` (`public` | `room-member` | `trusted-peer`), `experimental`.
- **`FeatureRegistry`** — in-process registry of descriptors and (optionally) bound `FeatureModule`s. Lives on every peer and every supernode.
- **`FeatureModule` trait** — the implementation behind a capability id. Defines `descriptor()`, `on_invoke(ctx)` (capability invocation), `on_message(source, payload)` (inbound datagram/stream payload), and `shutdown()`.
- **Channel multiplexer** — generic QUIC `Channel` API exposing reliable streams, unidirectional streams, and unreliable datagrams. Datagram tags `0x10`–`0xEF` are dynamic per-session, `0xFF` is broadcast, others reserved.
- **Capability exchange** — after the Ed25519 handshake, both sides send `CAPABILITY_ANNOUNCE`. Each consumer (chat panel, call overlay, game module) only enables UI/logic for capabilities present in the negotiated intersection.
- **Trust tiers** — `auth` is enforced by the runtime: `public` is open, `room-member` requires SFU membership in the same room, `trusted-peer` requires an explicit local trust entry. Per-feature byte/datagram quotas (`quota_bytes_per_sec`, `quota_datagrams_per_sec`) are mandatory for non-`core.*` namespaces.

### Built-in Capabilities

| ID | Kind | Auth | Provided by |
|---|---|---|---|
| `transport.quic.audio.v1` | datagram | trusted-peer | `conquerd-quic` |
| `transport.quic.relay.v1` | datagram | room-member | `conquerd-quic` (relay client) |
| `transport.quic.stream.v1` | stream | trusted-peer | `conquerd-quic` |
| `transport.quic.feature_datagram.v1` | datagram | trusted-peer | `conquerd-quic` |
| `core.chat.v1` | stream | trusted-peer | desktop client |
| `core.audio.opus` | datagram | trusted-peer | `conquerd-audio` |
| `core.file.v1` | stream | trusted-peer | desktop client |
| `room.audio.sfu` | datagram | room-member | supernode `SfuRoomModule` |
| `room.chat.v1` | stream | room-member | supernode `SfuRoomModule` |
| `web.host.h3.v1` | request | public | supernode WebTransport listener |

### Enabling Features on a Supernode

Supernode capabilities are declared in `<data_dir>/supernode.toml`. The manifest is the source of truth for what's advertised in `SUPERNODE_INFO` and what gets hosted by the WebTransport bridge:

```toml
schema_version = 1

[[feature]]
id = "core.chat.v1"
enabled = true

[[feature]]
id = "room.audio.sfu"
enabled = true
params = { codec = "opus", quota_bytes_per_sec = 32768 }

[[feature]]
id = "room.chat.v1"
enabled = true

[[feature]]
id = "web.host.h3.v1"
enabled = true                          # exposes /wt/<feature_id> over HTTP/3

[[feature]]
id = "x.acme.matchmaker"                # bespoke third-party module
enabled = true
version = "1.0"
kind    = "request"
auth    = "trusted-peer"
params  = { config_path = "etc/matchmaker.json" }
```

Disabled entries are kept on disk so an operator can flip them back on without retyping the descriptor. When `supernode.toml` is missing the manifest is derived from legacy env vars for back-compat.

### Authoring a Feature Module (Rust)

Implement `FeatureModule` and register it on the supernode (or any peer) at startup:

```rust
use conquerd_features::{
    AuthTier, CapabilityDescriptor, ChannelKind, FeatureModule, PeerId,
};

pub struct MatchmakerModule;

impl FeatureModule for MatchmakerModule {
    fn descriptor(&self) -> CapabilityDescriptor {
        CapabilityDescriptor::new("x.acme.matchmaker", "1.0", ChannelKind::Request)
            .with_auth(AuthTier::TrustedPeer)
    }

    fn on_message(&self, source: PeerId, payload: &[u8]) {
        // Parse, verify, route. The runtime has already enforced
        // auth tier + per-feature quotas before calling you.
    }
}

// Wire-up (e.g. in supernode `main.rs`):
let module = std::sync::Arc::new(MatchmakerModule);
if !state.features.bind_module("x.acme.matchmaker", module.clone()) {
    let _ = state.features.register_module(module);
}
```

`bind_module` attaches a module to a descriptor that's already loaded from the manifest; `register_module` adds both the descriptor and its module in one step. Inbound messages are delivered through `FeatureRegistry::dispatch_message(id, source, payload)`.

### Browser Participation (WebTransport)

When `web.host.h3.v1` is enabled, the supernode runs an HTTP/3 + WebTransport listener at `/wt/<feature_id>`. Browser clients use the JavaScript SDK at `web-sdk/conquerd.mjs`:

```js
import { ConquerdClient } from "./conquerd.mjs";

const client = await ConquerdClient.connect({
  url: "https://supernode.example/wt/room.audio.sfu",
  caps: ["room.audio.sfu", "room.chat.v1"],
  room: "lobby",
});

client.send("room.chat.v1", buildEnvelope(/* ... */));
client.on("room.chat.v1", (sourcePeer, payload) => { /* render */ });
```

The browser performs the same Ed25519 identity handshake as native peers; payloads are signed `SignalingMessage` envelopes verified by the supernode before any native fan-out. Browser tabs and native clients are indistinguishable participants in the same `room.*` or `game.*` feature.

### Quota and Trust Defaults

Anything outside the `core.*`, `transport.*`, `room.*`, `web.*`, `game.*` namespaces that ships without explicit `quota_bytes_per_sec` / `quota_datagrams_per_sec` is pinned to a conservative default (64 KB/s, 256 datagrams/s) so a buggy or hostile third-party module cannot saturate the link before the user explicitly trusts it.

---

## Connecting to Peers

Conquerd uses an **invite-only** model. There is no user directory or friend search.

### Creating an Invite
1. Click the **"+"** button or use the invite dialog.
2. Copy the generated invite link (or scan/save the QR code).
3. Share it with the person you want to connect with (via any channel — email, Signal, in person, etc.).

### Joining via Invite
1. Receive an invite link from a peer.
2. Paste it into the **Join** dialog in Conquerd.
3. The handshake completes automatically — both peers verify each other's identity cryptographically.
4. The peer appears in your left panel as a trusted contact.

### URI Launch
If Conquerd is installed, clicking a `conquerd://invite/...` link opens the app and processes the invite automatically.

---

## NAT Traversal

Most consumer NATs silently drop unsolicited inbound connections. Conquerd employs a multi-layer connection strategy to maximise reachability — each layer is tried in sequence and the first to succeed wins.

| Priority | Strategy | Requires | Transport |
|---|---|---|---|
| 0 | **QUIC direct** | Peer's QUIC port known (from relay hints or prior session) | QUIC (quinn) |
| 1 | **WebSocket assist** (1.25 s timeout) | At least one candidate endpoint reachable | WebSocket (TCP) |
| 2 | **WebSocket fallback** (4 s timeout) | Same candidates, longer timeout for slower paths | WebSocket (TCP) |
| 3 | **UDP hole punch** | Both sides online, non-symmetric NAT | Custom UDP protocol |
| 4 | **Supernode relay** | Both peers trusted by a common supernode | QUIC relay via supernode |

For most home users, **no manual port configuration is needed** — UPnP handles it automatically. If you're behind a strict firewall, forward one TCP port for signaling and let QUIC use ephemeral UDP ports.

### Do I Need a Supernode?

**In many cases, no.**

You **don't** need a supernode if:
- You and your peers are on the **same local network** (LAN).
- You have a **public IP** or your router supports **UPnP** (most home routers do).
- You only do **1-on-1 calls** (direct peer-to-peer).

You **do** need a supernode if:
- Both you and your peer are behind **strict NATs** that block direct connections (double-NAT, CGNAT, corporate firewalls).
- You want **group voice rooms** (SFU) with 3+ participants.

### UDP Hole Punching

When QUIC direct and WebSocket candidates both fail, Conquerd automatically attempts UDP hole punching. Each invite contains a `udp_hole_punch_hint` with the sender's STUN-discovered external UDP endpoint. Both peers simultaneously send probe packets to each other's external endpoint, creating NAT mappings that allow the other side's packets through.

```
Alice                              Internet                              Bob
NAT-A (104.54.197.38:57100)                         NAT-B (72.133.90.194:57240)

Alice sends probe ────────────────────────────────────────────→ NAT-B maps it
                  ←──────────────────────────────────────────── Bob sends probe
NAT-A maps it

Both receive ack → channel ESTABLISHED (bidirectional UDP)
```

**NAT compatibility:**

| NAT Type | Works? |
|---|---|
| Full-cone NAT (most home routers) | ✅ Yes |
| Address-restricted cone NAT | ✅ Yes |
| Port-restricted cone NAT | ✅ Yes |
| Symmetric NAT (CGNAT, corporate) | ❌ No — use supernode relay instead |

Symmetric NAT affects roughly 10–20% of users, primarily on mobile carrier NATs and some corporate networks. These users connect through a supernode relay automatically.

### Supernode-Coordinated Hole Punch

When uncoordinated hole punching fails due to timing mismatch, a trusted supernode can coordinate the attempt. Both peers send a `PUNCH_REGISTER` message to the supernode, which responds with `PUNCH_READY` containing each peer's endpoint and a synchronised `punch_at` timestamp so both sides begin probing simultaneously.

---

## Supernode Relay

A supernode is a volunteer peer that provides QUIC relay and SFU (group voice) hosting. Supernodes are **transport-only** — they never store identity, messages, or act as a central server.

### Connecting to a Supernode
1. Get the supernode's invite link from the operator.
2. Paste it into Conquerd's **Join** dialog.
3. The handshake completes — the supernode appears in your peer list.
4. Your session banner shows the connection mode:

| Banner | Meaning |
|--------|---------|
| **Direct P2P** (green) | Connected directly to your peer — no relay |
| **Relay** (green) | Using a supernode relay (free) |
| **Relay (gated)** (yellow) | Using a gated supernode — access was granted via the portal |

### Supernode Access Modes

Each supernode operator decides how peers gain relay access:

| Mode | What you see as a peer |
|------|------------------------|
| **Open** | Relay access is granted immediately — nothing to do. |
| **Terms of Service** | A web page opens asking you to accept the operator's terms before access is granted. |
| **Access Code** | A web page asks for a code provided by the operator (e.g. shared in a group chat). |
| **Timer** | A countdown page is shown; access is granted after the timer expires. |

When a gated supernode requires portal access, Conquerd opens the supernode's web page in the **Nodes** tab. Complete the required step and relay access is granted automatically.

Operators can also write custom access controllers that integrate any verification or payment system — Stripe, PayPal, Lightning, Patreon, OAuth, or anything else reachable from a web page. No wallet or payment infrastructure is built into the Conquerd client itself.

---

## Running a Supernode

Running a supernode is **optional**. Peers who can connect directly to each other do not need one at all.

### Basic Setup

Build the Rust supernode binary (one time):

```bash
cd rust/conquerd-supernode
cargo build --release
```

#### Windows
```bat
set CONQUERD_HOME=%USERPROFILE%\.conquerd
set supernode_invite_ttl=-1
set supernode_port=3478
set supernode_signaling_port=34935
rust\target\release\conquerd-supernode.exe
```

Or use the bundled helper:
```bat
start_supernode.bat
```

#### Linux / macOS
```bash
export CONQUERD_HOME="$HOME/.conquerd"
export supernode_invite_ttl=-1
export supernode_port=3478
export supernode_signaling_port=34935
./rust/target/release/conquerd-supernode
```

Or use the bundled helper:
```bash
./start_supernode.sh
```

On startup, the supernode will:
1. Generate an Ed25519 identity (first run only).
2. Print an **invite link** to the console — share this with your peers.
3. Begin accepting QUIC relay connections on port `3478` (UDP).
4. Begin accepting WebSocket signaling connections on port `34935` (TCP).

The invite link is also persisted in `~/.conquerd/supernode_invite.json` and survives restarts.

### Supernode Configuration

All configuration is via environment variables set before launching.

#### Core Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `supernode` | `0` | Set to `1` to activate supernode mode |
| `supernode_port` | `3478` | UDP port for QUIC relay traffic |
| `supernode_turn_host` | `0.0.0.0` | Bind address for QUIC relay |
| `supernode_signaling_port` | `34935` | TCP port for WebSocket signaling. **Always set a fixed value** — changing it breaks firewall rules and stored peer endpoints |
| `supernode_invite_ttl` | `-1` | Invite expiry in minutes. `-1` = never expires |
| `CONQUERD_HOME` | `~/.conquerd` | Data directory for identity, settings, files |

#### Feature Toggles

| Variable | Default | Description |
|----------|---------|-------------|
| `supernode_chat` | `1` | Enable chat relay (`0` to disable) |
| `supernode_files` | `1` | Enable file transfer relay (`0` to disable) |
| `supernode_sfu` | `1` | Enable SFU group voice rooms (`0` to disable) |
| `supernode_updates` | `1` | Enable P2P auto-update distribution (`0` to disable) |
| `supernode_auto_restart` | `1` | Auto-restart after applying an update (`0` to disable) |

#### Portal / Access Control Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `supernode_web_port` | *(unset)* | HTTPS port for portal/homepage. Set to enable (e.g. `8443`) |
| `supernode_web_title` | `Relay Node` | Human-readable name shown on the homepage |
| `supernode_access_mode` | `open` | Access mode: `open`, `tos`, `ad`, `code` |
| `supernode_access_code` | `conquerd` | Access code (only used when mode is `code`) |
| `supernode_ad_duration` | `30` | Countdown seconds (only used when mode is `ad`) |
| `supernode_ad_content` | *(empty)* | HTML content for the ad/timer waiting area |
| `supernode_tos_text` | *(built-in)* | Custom TOS text (or override `portal/tos.html`) |

### Firewall and Port Forwarding

Peers need to reach your supernode on two ports:

| Port | Protocol | Purpose |
|------|----------|---------|
| `3478` (or your `supernode_port`) | **UDP** | QUIC relay — voice, files, data |
| `34935` (or your `supernode_signaling_port`) | **TCP** | Signaling, chat, presence |

**Router / cloud firewall**: Forward both ports to your supernode's local IP.

**Linux firewall** (example with `ufw`):
```bash
sudo ufw allow 3478/udp
sudo ufw allow 34935/tcp
```

**Windows Firewall**: The first launch will prompt you to allow Python through the firewall. Accept both private and public network access.

### Running as a systemd Service (Linux)

Create a dedicated user:

```bash
sudo useradd -r -s /usr/sbin/nologin -m -d /opt/conquerd conquerd
sudo -u conquerd git clone <repo-url> /opt/conquerd/app
sudo mkdir -p /opt/conquerd/.conquerd
sudo chown conquerd: /opt/conquerd/.conquerd
```

#### Option A — Rust binary (recommended)

Build the binary once:

```bash
. "$HOME/.cargo/env"
cd /opt/conquerd/app/rust/conquerd-supernode
cargo build --release
sudo cp target/release/conquerd-supernode /usr/local/bin/
```

Create `/etc/systemd/system/conquerd-supernode.service`:

```ini
[Unit]
Description=Conquerd Supernode (QUIC relay + SFU)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=conquerd
Group=conquerd

Environment=CONQUERD_HOME=/opt/conquerd
Environment=supernode_invite_ttl=-1
Environment=supernode_port=3478
Environment=supernode_signaling_port=34935

# Portal (uncomment to enable)
#Environment=supernode_web_port=8443
#Environment=supernode_web_title=My Relay Node
#Environment=supernode_access_mode=open

ExecStart=/usr/local/bin/conquerd-supernode
Restart=on-failure
RestartSec=5

NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/conquerd
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

#### Enable and start

```bash
sudo systemctl daemon-reload
sudo systemctl enable conquerd-supernode
sudo systemctl start conquerd-supernode
```

Retrieve the invite link:
```bash
sudo journalctl -u conquerd-supernode | grep 'Invite URL'
```

### Portal Customisation

The supernode portal supports **two-tier template loading**:

1. **User overrides:** `$CONQUERD_HOME/.conquerd/portal/` (checked first)
2. **Built-in defaults:** embedded HTML in the Rust binary

To customise any page, create a complete HTML file with the same name in your portal override directory. Use `{{variable}}` placeholders for dynamic content.

| Template file | Purpose |
|--------------|--------|
| `homepage.html` | Supernode homepage (shown to connected peers) |
| `portal.html` | Default access gating entry page |
| `tos.html` | Terms of Service gate page |
| `ad.html` | Countdown/timer gate page |
| `code.html` | Access code entry page |
| `granted.html` | “Access granted” confirmation |
| `denied.html` | “Session expired / invalid” error |
| `health.html` | Stats dashboard with live relay/SFU metrics |

The Rust binary embeds all templates at compile time — overrides are loaded from disk at runtime and take precedence without requiring a rebuild.

### Supernode Stats Dashboard

The supernode exposes a `/health` page with live stats (uptime, version, connected/trusted peers, QUIC relay stats, SFU room details) and an `/api/stats` JSON endpoint (requires a valid session token). The dashboard auto-refreshes every 30 seconds. The `/arena` page redirects unauthenticated visitors to `/portal`.

---

## P2P Update System

Conquerd delivers application updates directly peer-to-peer — no update server, no CDN. When a peer with a newer (or differently-hashed) version of the code connects, it automatically offers changed files over the existing signaling channel.

### What Gets Updated

| Content | Updated via P2P? |
|---------|-----------------|
| All Python source (`client_desktop/`, `client_shared/`) | ✅ Yes |
| Explicitly listed assets (logo, icon) | ✅ Yes |
| The `.exe` binary itself | ❌ No — requires rebuild |
| Python interpreter, third-party libraries | ❌ No — fixed at build time |

Any logic change — UI tweaks, protocol updates, bug fixes, new features — can propagate to portable exe users over a live peer connection without a rebuild.

### How It Works

1. Peers exchange `VERSION_ANNOUNCE` messages on connect.
2. The peer with newer/different code computes a manifest diff and sends an `UPDATE_OFFER`.
3. The receiving user sees an update dialog and clicks **Download & Apply**.
4. Files are pushed with binary delta patching and zlib compression.
5. Files are staged, originals are backed up, then staged files are moved into place. Rollback restores from backup on failure.
6. "Restart to apply" prompt appears. After restart, new code loads.

Source-built peers (`client_desktop/`) and frozen exe peers (`_internal/client_desktop/`) compare manifests correctly via canonical path normalisation — updates work transparently across environments.

### Release Signing

When `trusted_release_keys` are configured, the receiver enforces detached Ed25519 signatures over offered manifests. Offer payloads support multiple signatures for threshold validation (configurable via `CONQUERD_RELEASE_THRESHOLD`, default `1`). Revoked keys are ignored even if cryptographically valid. Key rotation metadata is persisted and survives restarts.

---

## Settings Reference

Access settings via the gear icon in Conquerd.

### Network

| Setting | Default | Description |
|---------|---------|-------------|
| Network mode | `public` | `public` = use STUN for IP discovery; `local` = LAN only |
| Public endpoint | (auto) | Manual override: `IP:port` |
| Signaling port | `0` (auto) | Set a fixed port if you need a firewall rule |
| UPnP enabled | `true` | Auto port-mapping on router |
| STUN servers | 3 built-in | Google (`stun.l.google.com`, `stun1.l.google.com`) + Cloudflare (`stun.cloudflare.com`) + custom servers with per-server enable/disable |

### Audio

| Setting | Default | Description |
|---------|---------|-------------|
| Input device | (system default) | Microphone selection |
| Output device | (system default) | Speaker/headphone selection |
| PTT key | `Space` | Push-to-talk key binding |
| Voice activation | `false` | Auto-transmit when sound detected |
| Noise suppression | `true` | Background noise reduction |
| Jitter buffer depth | `3` | Frames (1–20). Higher = more latency, smoother audio |

### Relay

| Setting | Default | Description |
|---------|---------|-------------|
| Allow gated supernodes | `true` | Allow connecting to supernodes that require portal access |

### Security

| Setting | Default | Description |
|---------|---------|-------------|
| Attestation policy | `warn` | `off` = no peer build checks; `warn` = challenge but don't block; `strict` = deny relay to unverified peers |

---

## Data and Files

All Conquerd data is stored under `CONQUERD_HOME` (default `~/.conquerd/`):

| File | Purpose |
|------|---------|
| `identity.json` | Your Ed25519 keypair — **back this up** |
| `settings.ini` | All preferences |
| `peer_store.json` | Trusted peers list |
| `chat_history.db` | Chat messages (SQLite) |
| `my_rooms.json` | Saved room invites |
| `received_files/` | Downloaded files from file transfers |
| `crash_*.log` | Python crash dump logs |
| `faulthandler.log` | C-level crash traces |

Supernodes additionally store:

| File | Purpose |
|------|---------|
| `supernode_invite.json` | Persistent invite link |
| `supernode_endpoints.json` | Endpoint mailbox for peer reconnection (24h TTL) |

> **What to back up**: At minimum, back up `identity.json`. Losing it means peers will see you as a new, untrusted identity.

---

## Troubleshooting

### Can't connect to a peer
- Both peers need network reachability — check UPnP status in settings.
- Ensure both peers completed the invite handshake (trusted peer appears in left panel).
- Try a supernode as relay if direct connections fail.
- For UDP hole punching, both sides need to exchange invites at roughly the same time (within 30 seconds).

### No audio in calls
- Check audio input/output device selection in Settings.
- Verify microphone permissions (Windows: Settings → Privacy → Microphone).
- Try toggling between PTT and voice activation.
- Set `audio_debug=1` environment variable for detailed audio pipeline logging.

### Crash dumps
- Python tracebacks are written to `~/.conquerd/crash_<timestamp>.log`.
- C-level segfaults (e.g. from PortAudio) are captured via `faulthandler` to `~/.conquerd/faulthandler.log`.
- The `.bat` launchers keep the console window open after a crash so the traceback is visible.

### Supernode troubleshooting
- **Peers can't connect**: Verify both `supernode_port` (UDP) and `supernode_signaling_port` (TCP) are forwarded and open. Check that `supernode_turn_host` isn't binding to `127.0.0.1`.
- **Port changes on restart**: Always set `supernode_signaling_port` to a fixed value (e.g. `34935`). Changing it breaks firewall rules and stored peer endpoints.
- **Service fails with exit code 226/NAMESPACE**: LXC, OpenVZ, or some VPS hosts don't support mount namespaces. Comment out the hardening block in the systemd unit file and restart.
- **Peers get relay access without the portal**: Ensure `supernode_access_mode` is set to `tos`, `ad`, or `code` (not `open`) and `supernode_web_port` is set.

---

## Developer Guide

### Entry Points

| File | Purpose |
|---|---|
| `conquerd_entry.py` | PyInstaller / packaged distribution entry point |
| `client_desktop/__main__.py` | Direct CLI entry (`python -m client_desktop`); handles `conquerd://` URI on launch, writes crash dumps |

### Building from Source
```bash
pip install -e ".[dev]"
python -m client_desktop
```

### Run Tests
```bash
pytest tests/ -v
```

There are 961 tests covering: crypto/identity, invite/handshake, signaling, call control, audio pipeline, QUIC transport (direct + relay), chat, rooms/SFU, file transfer, network/STUN/hole-punch, capability negotiation/invocation, quota enforcement, feature trust UX, and all UI panels.

### Two-Client Local Testing
The `.bat` files create isolated profiles under `.clientA/` and `.clientB/` via `USERPROFILE` override:
```bash
ConquerD_a.bat   # Terminal 1
ConquerD_b.bat   # Terminal 2
```

### Multi-Profile Testing (Manual)
```bash
# Windows
set USERPROFILE=C:\path\to\.clientA
python -m client_desktop

# In another terminal
set USERPROFILE=C:\path\to\.clientB
python -m client_desktop
```

### Portable Build
```powershell
# Build with PyInstaller
.\build_win64.ps1

# Or directly
.venv\Scripts\pyinstaller --noconfirm conquerd.spec
```

The resulting `dist\conquerd\conquerd.exe` is self-contained and portable. Users can receive future source updates from connected peers without rebuilding.

#### Code Signing (Windows, optional)

The build script automatically signs `conquerd.exe` and `conquerd-installer.exe` when a certificate is configured. Signing is **optional** — the build completes without it.

`signtool.exe` must be on `PATH`. Install it via:
- **Visual Studio Installer** → Modify → Individual Components → search "Windows SDK" (e.g. Windows 11 SDK 10.0.26100.x) — signing tools are included.
- **Standalone** — [Windows SDK installer](https://developer.microsoft.com/windows/downloads/windows-sdk/) → check "Windows SDK Signing Tools for Desktop Apps".

After install, add the `x64` bin path (e.g. `C:\Program Files (x86)\Windows Kits\10\bin\10.0.26100.0\x64`) to your `PATH`, or open a Visual Studio Developer Command Prompt.

Configure signing via environment variables before running `build_win64.ps1`:

| Variable | Description |
|---|---|
| `CONQUERD_SIGN_THUMBPRINT` | SHA-1 thumbprint of a cert in the Windows Certificate Store (recommended for CI / EV tokens) |
| `CONQUERD_SIGN_PFX` | Path to a `.pfx` file (OV cert, local builds) |
| `CONQUERD_SIGN_PASSWORD` | Password for the `.pfx` file |
| `CONQUERD_SIGN_TIMESTAMP` | RFC 3161 timestamp server URL (default: `http://timestamp.digicert.com`) |

If none of these are set the signing step is silently skipped.

### Version Bumping

Version is controlled in one place: `client_shared/version.py` (`APP_VERSION` tuple). `pyproject.toml` and `__init__.py` derive from it automatically.

**When to bump:**
- Sprint / feature-batch complete → bump **minor** (or **major** for breaking protocol changes)
- Bug-fix round complete → bump **patch**
- Before a cross-install peer test → bump **patch** (same-version hash detection is also supported)

**Do not bump** for mid-feature saves, tests/docs-only changes, or whitespace edits.

### Project Structure

```
├── conquerd_entry.py            # PyInstaller entry point
├── client_desktop/              # PySide6 desktop client (Python)
│   ├── app.py                   # ConquerdApp: creates all components, wires Qt signals
│   ├── connection_manager.py    # Invite handshake, WebSocket signaling, peer tracking
│   ├── async_bridge.py          # Asyncio ↔ Qt bridge (daemon thread + asyncio loop)
│   ├── call_controller.py       # Call state machine; wires AudioEngine + QUICPeerManager
│   ├── quic_peer_manager.py     # Per-peer audio codec state; QUIC callbacks → CallController
│   ├── quic_relay_client.py     # Async wrapper around Rust QUICRelayClient (room audio)
│   ├── quic_transport.py        # Wrapper around Rust QUICTransport (stream mux)
│   ├── session_state.py         # PeerSessionState / VoiceSessionState consolidated objects
│   ├── chat_manager.py          # Send/receive text; delivery states; Qt signals
│   ├── chat_store.py            # SQLite schema + queries (history, unread counts)
│   ├── room_manager.py          # Per-participant room state (speaking, muted, quality)
│   ├── room_store.py            # JSON persistence for user rooms
│   ├── sfu_client.py            # SFU membership (join/leave/member list); audio via Rust
│   ├── relay_access_manager.py  # Supernode portal access state + Qt signals
│   ├── file_transfer.py         # P2P file send/receive; chunking, retry, progress signals
│   ├── metrics.py               # Per-peer QUIC stats → ConnectionQuality (excellent…poor)
│   ├── network_monitor.py       # QTimer-driven QUIC stats polling; metrics_updated signal
│   ├── voice_activity_rust.py   # Python wrapper around Rust VoiceActivityDetector
│   ├── stun_client.py           # STUN queries for external IP:port discovery
│   ├── settings.py              # Qt settings (audio devices, volume, PTT key, relay hints)
│   ├── ptt_controller.py        # Global PTT keybind via pynput
│   ├── ringtone.py              # Incoming call ringtone playback
│   ├── taskbar_badge.py         # Unread/missed-call badges on taskbar (Win32 API)
│   ├── upnp.py                  # UPnP IGD auto port-forwarding
│   ├── uri_scheme.py            # Registers conquerd:// URI scheme (Windows Registry)
│   ├── shortcuts.py             # Qt keyboard shortcuts
│   ├── github_updater.py        # Polls GitHub Releases API; emits update_available signal
│   └── ui/                      # PySide6 UI widgets
│       ├── main_window.py       # Primary window: sidebar + chat + call dock + status banner
│       ├── chat_panel.py        # Message bubbles, typing indicators, file attachments
│       ├── call_panel.py        # Call overlay: participants, mute/PTT, timer, quality
│       ├── incoming_call_dialog.py  # Modal answer/reject dialog + ringtone
│       ├── participant_widget.py    # Per-participant: name, speaking, level bar, quality
│       ├── settings_dialog.py   # Audio / network / relay settings UI
│       ├── onboarding_wizard.py # First-run: identity, invite import, supernode pairing
│       ├── stats_panel.py       # Real-time QUIC stats (RTT, loss, bandwidth, jitter)
│       ├── theme.py             # Dark theme, DPI awareness, fonts, icon sizing
│       └── icons.py             # SVG + raster icon loading
├── client_shared/               # Shared crypto/protocol (no UI dependency)
│   ├── identity.py              # Ed25519 keypair; public_id, peer_id, sign/verify
│   ├── invite.py                # Create/parse conquerd:// invite links (signed, timestamped)
│   ├── handshake.py             # X25519 HKDF + AES-GCM SessionCipher; replay protection
│   ├── crypto.py                # SHA-256, base64url, nonce gen, peer/room ID derivation
│   ├── protocol.py              # MessageType enum (all message types)
│   ├── peer_store.py            # Trusted peer persistence; invite history + revocations
│   ├── capabilities.py          # Python mirror of conquerd-features well-known capability list
│   ├── channel_tag.py           # Channel-tag registry (0x10–0xEF dynamic, 0xFF broadcast)
│   ├── feature_modules.py       # FeatureRegistry: register/lookup/dispatch_invoke Python-side
│   ├── feature_trust.py         # FeatureTrustStore + FeatureTrustGate (user-consent for bespoke namespaces)
│   ├── encrypted_store.py       # AES-256-GCM-encrypted local settings store
│   └── version.py               # APP_VERSION — single source of truth
├── rust/
│   ├── conquerd-audio/          # PyO3 crate: CPAL + Opus + DSP (AudioEngine, OpusEncoder/Decoder, VAD, etc.)
│   ├── conquerd-quic/           # PyO3 crate: quinn + tokio (QUICTransport, QUICPeerManager, QUICRelayClient, etc.)
│   ├── conquerd-supernode/      # Standalone binary: QUIC relay, SFU, WebSocket signaling, HTTPS portal
│   └── conquerd-installer/      # Standalone binary: download + apply app releases
├── tests/                       # pytest tests (718): unit, integration, UI
└── pyproject.toml               # Project configuration
```

---

## Protocol Messages

| Category | Types |
|----------|-------|
| Invite | `invite_handshake_init`, `invite_handshake_accept`, `invite_handshake_reject` |
| Chat | `chat_message`, `chat_ack`, `chat_typing` |
| Call | `call_request`, `call_accept`, `call_reject`, `call_end` |
| Relay | `relay_request`, `relay_granted`, `relay_revoke` |
| Relay Access | `relay_payment_required`, `relay_access_granted`, `relay_access_denied` |
| Supernode Info | `supernode_info`, `supernode_info_request` |
| Room Mgmt | `hello`, `welcome`, `room_join`, `room_leave`, `room_state`, `room_peer_joined`, `room_peer_left`, `room_list_request`, `room_list_response` |
| SFU / Room | `sfu_join`, `sfu_leave`, `sfu_members`, `sfu_offer`, `sfu_answer`, `sfu_audio`, `sfu_chat`, `sfu_room_list`, `sfu_peer_joined`, `sfu_peer_left` |
| SFU Subscription | `sfu_subscribe`, `sfu_unsubscribe` |
| SFU Room Mgmt | `sfu_room_create`, `sfu_room_created`, `sfu_room_invite`, `sfu_room_invite_result`, `sfu_room_invite_generate` |
| SFU File Transfer | `sfu_file_offer`, `sfu_file_chunk`, `sfu_file_complete` |
| File Transfer | `file_transfer_offer`, `file_transfer_accept`, `file_transfer_reject`, `file_transfer_chunk`, `file_transfer_complete`, `file_transfer_ack`, `file_transfer_error` |
| Trust | `trust_request`, `trust_accept` |
| Peer Room Invite | `peer_room_invite` |
| Hole Punch | `punch_register`, `punch_ready` |
| Endpoint | `endpoint_update` |
| Handle | `handle_update` |
| Encrypted | `encrypted_signal` |
| Peer Updates | `version_announce` |
| Capability | `capability_announce`, `capability_invoke` |
| Utility | `ping`, `pong`, `error`, `speaking_state`, `presence_update` |

## Technology Stack

| Layer | Library / Runtime |
|---|---|
| UI | PySide6 (Qt6) |
| QUIC transport | `conquerd_quic` (Rust — quinn, tokio, rustls, ed25519-dalek) |
| Audio pipeline | `conquerd_audio` (Rust — CPAL, audiopus, ringbuf, rustfft) |
| Cryptography | `conquerd_crypto` (Rust — ed25519-dalek, x25519-dalek, aes-gcm, argon2, hkdf) |
| Chat / signaling serialisation | JSON over WebSocket (`websockets`) |
| Async ↔ Qt bridge | `asyncio` daemon thread + Qt signal callbacks |
| Testing | pytest |
| Packaging | PyInstaller, setuptools, cargo |

## Code Signing Policy

Free code signing provided by [SignPath.io](https://signpath.io), certificate by [SignPath Foundation](https://signpath.org).

### Team Roles

| Role | Members |
|---|---|
| **Authors** (trusted committers) | [Members](https://github.com/orgs/ConquerD/teams/conquerd-authors) |
| **Reviewers** (PR reviewers) | [Members](https://github.com/orgs/ConquerD/teams/conquerd-reviewers) |
| **Approvers** (release signing) | [Owners](https://github.com/orgs/ConquerD/teams/conquerd-approvers) |

### Privacy Policy

See [PRIVACY.md](PRIVACY.md) for the full privacy policy.

ConquerD is a local-first application. All peer-to-peer communication (voice, chat, file transfer) travels directly between clients or through volunteer supernodes chosen by the user, and is end-to-end encrypted. ConquerD does not operate servers that store your identity, messages, or call data.

The following network contacts occur automatically or on user action:

| Feature | External service contacted | When | How to disable |
|---|---|---|---|
| **STUN / public IP discovery** | Google (`stun.l.google.com`, `stun1.l.google.com`) and Cloudflare (`stun.cloudflare.com`) | On startup when *Network mode* is set to `public` | Set *Network mode* to `local` in Settings |
| **GitHub update check** | GitHub Releases API (`api.github.com`) | Periodically in the background | Uncheck *Check for updates* in Settings |
| **YouTube link preview** | YouTube / Google CDN (via yt-dlp) | Only when you click to expand a YouTube link shared in chat | No action required — this is opt-in per link |

No account credentials, message content, contact lists, or unique identifiers are transmitted to any of the above services.

## License
MIT

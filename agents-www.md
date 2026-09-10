# DoubleSlash Website Agent Contract

## Overview

`ConquerD_www/` is the static public website for DoubleSlash (doubleslash.space). It is currently a single-page site with no application backend, package manager, framework, or build step.

| File | Purpose |
|---|---|
| `index.html` | All public page content, metadata, inline diagrams, and small interaction scripts |
| `styles.css` | Site layout, typography, responsive behavior, and visual styling |
| `logo.svg`, `logo-full.svg`, `conquerd.ico` | Brand assets |
| `CNAME` | Custom-domain configuration |
| `README.md` | Legacy website notes; not an architecture or product source of truth |
| `agents-www.md` | This website-specific working contract |

The page currently imports Mermaid from jsDelivr and uses an inline module for diagram rendering, pan, and zoom. Treat that as the existing dependency boundary: core copy and navigation must remain understandable if JavaScript or the CDN is unavailable.

## Sources of Truth

Use sources in this order when changing product claims:

1. `../agents.md` for current invariants, security constraints, and project status.
2. `../README.md` for the human-facing architecture, features, setup, and protocol summary.
3. Cargo manifests and implementation under `../rust/` when a claim needs code-level verification.
4. `../backlog.md` only for clearly labelled future work; backlog items are not shipped features.
5. Published GitHub releases for downloadable artifact names and release availability.

There is no authoritative `source/` documentation tree or `source/RELEASE_NOTES.md`. Do not use this directory's legacy `README.md` as proof of current product behavior.

Avoid freezing test totals, capability totals, or other fast-changing counts into marketing copy unless the page derives them from a maintained source and the change includes a verification step.

## Current Product Contract

### Product and trust model

- DoubleSlash is a privacy-first, modular peer-connectivity framework with a native Rust Qt/QML desktop client. The invite/portal protocol is branded D:// (`d://` URLs; legacy `conquerd://` is still accepted).
- Identity, discovery, and presence are client-owned. There is no first-party account or identity backend.
- Peers connect through signed invites and an authenticated handshake using Ed25519 identities.
- Cross-peer behavior is capability-negotiated. First-party UI is a consumer of feature modules, not a bypass around them.
- Optional volunteer supernodes assist transport and host opt-in features; they are never identity authorities.

### Repository and workspaces

- `rust/conquerd-client/` is its own Cargo workspace and contains the primary desktop client.
- The outer `rust/` workspace contains `conquerd-features`, `conquerd-opus`, `conquerd-supernode`, and `conquerd-installer`.
- `rust/conquerd-supernode-manager/` is a separate workspace for provisioning and cluster operations.
- Do not present the legacy Python application described by this directory's `README.md` as the current product.

### Connectivity and signaling

- Direct sessions use QUIC peer-to-peer through the client's embedded `quinn::Endpoint`.
- Relay sessions use `QuicRelayClient` and the supernode QUIC relay while preserving the same generic channel abstraction.
- Signed signaling prefers an established QUIC signaling stream and falls back to WebSocket.
- WebSocket remains a membership and signaling fallback. Room audio normally uses QUIC relay datagrams and can use the same signed WebSocket path as a fallback.
- If a direct call does not establish within five seconds, the client can fall back to a temporary private SFU room and later upgrade to direct transport.
- Cluster failover retries a missing room only for the bounded `room_absent` recovery path; do not imply endless or unconditional retry.

### Rooms, privacy, and supernodes

- Authoritative room definitions are client-owned in encrypted `my_rooms.dat`, keyed by `(supernode_id, room_id)`.
- Supernode SFU rooms are memory-only. User-created rooms idle-GC after 15 minutes with no voice participants or chat subscribers.
- Room chat bodies, Opus audio, and file chunks are signed and end-to-end sealed. Supernodes route opaque wire bytes and must not decrypt, log, or persist content.
- Supernodes may see routing metadata such as peer ids, room ids, membership, timing, signatures, message types, and byte volumes.
- A supernode may persist its own identity, trusted peer public ids, endpoint mailbox, manifest, and invite configuration. Do not simplify this to “the supernode stores no identity.”
- The client UI keeps ordinary peers in the Peers rail and trusted supernodes in the Rooms sidebar.

### In-app portal and games

- Portal pages and games load only in the native client through `d://`.
- Portal content uses `web.host.app.v1` over an identity-authenticated QUIC bidirectional stream.
- Multiplayer game traffic uses `game.relay.v1` over identity QUIC relay datagrams with fixed tag `0x05`.
- Built-in portal examples include cursor sharing, brick relay, and shared drawing.
- Hosted pages use the `window.conquerd` bridge and `/_conquerd/channel/*` endpoints.
- There is no public HTTP or WebTransport game surface. Do not mention `web.host.h3.v1`, `web_port`, public game TLS certificates or fingerprints, or `webtransport.rs` as current architecture.
- Use “browser game” only when the copy makes clear that HTML/JavaScript runs inside ConquerD's native in-app portal, not in an arbitrary external browser.

### Capability catalogue

The current built-in catalogue contains these stable capability ids:

- Transport: `transport.quic.audio.v1`, `transport.quic.relay.v1`, `transport.quic.stream.v1`, `transport.quic.feature_datagram.v1`, `transport.quic.uni_stream.v1`, `transport.quic.stream_priority.v1`, `transport.quic.zero_rtt.v1`, `transport.quic.pmtud.v1`, `transport.quic.migration.v1`, `transport.quic.flow_control.v1`.
- Peer features: `core.chat.v1`, `core.audio.opus`, `core.file.v1`.
- Room features: `room.audio.sfu`, `room.chat.v1`, `room.file.v1`.
- Video and content audio: `core.video.v1`, `core.audio.content.v1`, `room.video.sfu`, `room.audio.content.sfu`. These are in `local_capabilities()` and pinned by the `feature_ids_are_stable` golden test; they are advertised capabilities, which is not the same as video being a finished feature.
- Portal and games: `web.host.app.v1`, `game.relay.v1`.

Every feature has an authentication tier and quota policy. Third-party `x.<vendor>.*` capabilities require explicit consent. If the catalogue changes, verify it against `local_capabilities()` in `conquerd-features` rather than copying an old numeric total.

### Updates and releases

- Releases are distributed through GitHub Releases; do not describe peer-pushed source updates as the current update model.
- Tagged releases require the installer to verify an Ed25519-signed release manifest and the downloaded archive hash before applying an update.
- The rolling nightly channel uses SHA-256 sidecars and an intentionally unsigned development manifest; do not describe nightlies as release-signed.
- Platform signing and notarization are release-channel properties, so state them conditionally unless a published artifact has been verified.
- Read the displayed application version from `rust/conquerd-client/Cargo.toml`; the installer version must remain synchronized for SignPath compatibility.

## Remaining Website Documentation Drift

The local website `README.md` describes the retired Python application, eight-participant rooms, peer-to-peer source updates, and a Nodes-tab UI. Do not copy those claims into `index.html`; updating that legacy file is a separate documentation task.

## Agent Roles

### 1. Content Accuracy Agent

- Verify architecture, security, capability, and release claims against the sources of truth.
- Distinguish shipped behavior from backlog or aspirational work.
- Remove retired terminology rather than adding compatibility language for paths that no longer exist.

### 2. Copy and Information Architecture Agent

- Maintain clear copy for privacy-conscious users without overstating anonymity or security.
- Keep the single-page section hierarchy and navigation labels consistent.
- Explain technical features in user terms while retaining exact capability ids where useful.

### 3. HTML, CSS, and Interaction Agent

- Maintain semantic, accessible HTML and responsive CSS without adding a build pipeline unless explicitly requested.
- Preserve keyboard navigation, visible focus states, readable contrast, reduced-motion behavior, and sensible mobile layouts.
- Keep Mermaid diagrams usable with a text alternative or surrounding copy that conveys the same essential facts.
- Avoid expanding third-party scripts or dependencies without a concrete need and review.

### 4. SEO and Release Agent

- Keep the title, description, canonical URL, social metadata, favicon, and structured data aligned with visible copy.
- Validate download links and version claims against a published release before presenting them as available.
- Keep `CNAME` and deployment assumptions compatible with the static host.

### 5. Website QA Agent

- Check internal anchors, local assets, external links, mobile layout, keyboard behavior, and JavaScript-disabled readability.
- Search for retired architecture terms and stale quantitative claims.
- Review the final diff for unintended edits outside `ConquerD_www/`.

## Website Guardrails

- Do not add accounts, analytics, tracking pixels, centralized discovery, or backend-dependent features.
- Do not reintroduce a public WebTransport/H3 or game-TLS architecture in copy or diagrams.
- Do not imply that a supernode can decrypt room chat, audio, or file content.
- Do not publish hard-coded test or capability counts without a maintained derivation and validation step.
- Do not add new pages, frameworks, dependencies, build tooling, or deployment services unless the task calls for them.
- Use relative paths for local assets and preserve the canonical production URL in metadata.
- Preserve unrelated user changes and keep website-only work scoped to `ConquerD_www/`.

## Validation Checklist

Before finishing a website documentation or content change:

1. Search the changed files for `WebTransport`, `web.host.h3.v1`, `webtransport.rs`, `web_port`, public game TLS/fingerprint claims, obsolete test totals, and stale capability totals.
2. Compare capability and protocol claims with `../README.md`, `../agents.md`, and the relevant Rust enums or registries.
3. Verify version, download, signing, and notarization claims against the current manifests and published release state.
4. Check local paths, anchors, images, metadata, responsive layout, keyboard operation, and useful no-JavaScript rendering.
5. Run a whitespace/diff check and confirm that unrelated repository changes remain untouched.

Documentation-only website edits do not require Rust tests. Run code tests only when the task also changes application behavior.

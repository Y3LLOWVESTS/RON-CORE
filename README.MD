
# RON-CORE
>Still in beta build phase


**RON-CORE** = Micronode + Macronode substrate  
> (kernel, transport, overlay/DHT, identity/KMS, gateway/edge, storage/index, SDKs)  


RON-CORE is the **core plane** of the RustyOnions project: a modular, high-performance runtime for building secure, content-addressed, capability-scoped applications.

---

## Features (High Level)

- 🧠 **Microkernel core** (`ron-kernel`, `ron-bus`, `ron-proto`, `ron-metrics`, `ron-transport`, `oap`)
- 🌐 **Overlay & DHT** (`svc-overlay`, `svc-dht`) for peer discovery and session gossip
- 📦 **Storage & Index** (`svc-storage`, `svc-index`, `ron-naming`) for BLAKE3-addressed content and naming
- 🔐 **Identity & Auth** (`ron-kms`, `svc-passport`, `ron-auth`, `ron-policy`, `ron-audit`, `svc-registry`)
- 🚪 **Gateway & Edge** (`svc-gateway`, `omnigate`, `svc-edge`) for HTTP/OAP ingress and edge caching
- 🧩 **Node profiles** (`micronode`, `macronode`) for small single-binary and full multi-service deployments
- 💻 **App SDK** (`ron-app-sdk`) as the reference client for apps and facets

All of this is built around a small set of **non-negotiable invariants** from `RON-CORE.MD`:

- **Content addressing:** All blobs addressed by **BLAKE3** (`b3:<hex>`), verified on read.
- **Transport limits:** OAP/1 frames bounded to 1 MiB; strict timeouts & backpressure.
- **Amnesia mode:** Micronode is RAM-only by default; persistence is explicit opt-in.
- **Observability:** Every service exposes `/metrics`, `/healthz`, `/readyz`.
- **Capabilities:** All access is scoped by macaroon-style capabilities (no ambient auth).
- **PQ-ready edge:** TLS/KMS designed to accommodate hybrid/PQ in the future.
- **Concurrency discipline:** No locks held across `.await`; bounded queues instead.

---

## Repository Layout

A typical layout for this repo:

```text
ron-core/
  Cargo.toml          # workspace
  README.md
  docs/
    RON_CORE.md
    SDK_IDB.md
    SDK_SECURITY.md
    ...
  crates/
    ron-kernel/
    ron-bus/
    ron-proto/
    ron-metrics/
    oap/
    ron-transport/
    svc-overlay/
    svc-dht/
    ron-naming/
    svc-storage/
    svc-index/
    ron-policy/
    omnigate/
    svc-gateway/
    ron-kms/
    svc-passport/
    ron-auth/
    svc-registry/
    svc-edge/
    ron-audit/
    ron-app-sdk/
    micronode/
    macronode/
  sdk/                # non-Rust SDKs live here (TS/Go/Py/etc., future)
    ron-app-sdk-ts/   # example future SDKs
    ron-app-sdk-go/
    ...
````

> **Note:** Web3/value-plane crates (wallets, ledger, rewarder, etc.) live in the legacy **RustyOnions** repo and are intentionally excluded from `ron-core` until the core plane is stable.

---

## Crates Overview

| Crate           | Role (1-liner)                                                   |
| --------------- | ---------------------------------------------------------------- |
| `ron-kernel`    | Microkernel: process bus, shutdown, supervision, metrics glue.   |
| `ron-bus`       | In-process broadcast bus (events, kernel notifications).         |
| `ron-proto`     | Wire-safe DTOs and protocol types (OAP/1, storage, auth, etc.).  |
| `oap`           | OAP/1 framing, limits, and protocol constants.                   |
| `ron-metrics`   | Metrics facade: Prometheus exports, health/readiness helpers.    |
| `ron-transport` | Transport layer (TCP/TLS via rustls), timeouts, IO helpers.      |
| `svc-overlay`   | Overlay node sessions, gossip, and connection management.        |
| `svc-dht`       | DHT for peer discovery and name → provider resolution.           |
| `ron-naming`    | Name/address types and helpers used across storage/index/DHT.    |
| `svc-storage`   | Content-addressable storage (BLAKE3 CAS).                        |
| `svc-index`     | Index and lookup: human-friendly names → CAS/digest/providers.   |
| `ron-policy`    | Policy evaluation utilities (who can do what, where).            |
| `omnigate`      | Higher-level HTTP/OAP gateway composition & routing.             |
| `svc-gateway`   | Ingress HTTP service, OAP terminator, app-facing surface.        |
| `ron-kms`       | Key management (Ed25519, PQ-ready hooks, zeroization).           |
| `svc-passport`  | Identity, passports, macaroon/capability issuance.               |
| `ron-auth`      | Capability verification and authZ primitives.                    |
| `svc-registry`  | Registry of services/identities and their metadata.              |
| `svc-edge`      | Edge asset service: static assets, caching, byte-range reads.    |
| `ron-audit`     | Append-only audit log and event ingester.                        |
| `ron-app-sdk`   | Rust SDK for apps/facets (reference implementation).             |
| `micronode`     | Single-binary node profile (amnesia-first, RAM-only by default). |
| `macronode`     | Multi-service node profile (persistent, “cluster” profile).      |

---

## Quickstart: Run a Micronode

The easiest way to see RON-CORE in action is to run a **Micronode** locally.

### 1. Build the workspace

```bash
cargo build --workspace
```

### 2. Run Micronode with a sample config

Assuming there is a default config in `crates/micronode/configs/`:

```bash
RUST_LOG=info \
cargo run -p micronode
```

Micronode will bind to the admin and API addresses defined in its config (or sensible defaults).

### 3. Hit the admin plane

In another terminal:

```bash
BASE="http://127.0.0.1:5310"   # adjust if your config differs

curl -s "$BASE/healthz"
curl -s "$BASE/readyz"
curl -s "$BASE/metrics" | head
```

You should see:

* `{"ok":true}` from `/healthz`
* A truthful readiness payload from `/readyz`
* Prometheus metrics, including amnesia posture and HTTP metrics

### 4. Try the KV plane (example)

```bash
BASE="http://127.0.0.1:5310"

curl -X PUT "$BASE/v1/kv/demo/hello" \
  -H "Content-Type: text/plain" \
  --data 'world'

curl "$BASE/v1/kv/demo/hello"
```

If security mode is `DevAllow`, this should round-trip without a capability. In stricter modes, you’ll need a passport/capability token issued by `svc-passport` / `ron-auth`.

---

## Status

RON-CORE is currently:

* **Actively developed**
* **Core-plane only** (no Web3/value-plane crates)
* **API and wire formats still subject to change** before v1.0

For detailed status, invariants, and launch plan, see:

* `docs/RON_CORE.md`
* crate-specific `README.md` and `NOTES.md` files

---

## Development

### Toolchain

* **MSRV:** 1.80.0 (per `RON-CORE.MD`)
* Uses `cargo` workspaces
* Prefers `rustls` for TLS (no `native-tls` unless strictly needed)

### Basic checks

From the repo root:

```bash
# Format + clippy for the whole core
cargo fmt --all
cargo clippy --all-targets --all-features -- -D warnings

# Run tests for a specific crate
cargo test -p micronode
cargo test -p ron-kernel
```

For some crates you’ll also find **`scripts/*`** helpers (e.g., `beta_check.sh`) that run the standardized `CODECHECK.md` pipeline for that crate (fmt, clippy, tests, and sometimes smoke tests).

---

## SDKs & Polyglot Story

This repo contains:

* **Rust SDK**: `crates/ron-app-sdk` – the reference client for apps and facets.
* **SDK blueprints** in `docs/`:

  * `SDK_IDB.md` – Invariant-Driven Blueprint for SDKs.
  * `SDK_SECURITY.md` – security checklist for all SDKs.

Non-Rust SDKs (TypeScript, Go, Python, etc.) will live under the top-level `sdk/` directory and follow the same schema and security invariants.

All SDKs share:

* canonical schemas from `ron-proto` and gateway OpenAPI, and
* the same error envelope and capability discipline.

---

## What This Repo Is *Not*

To keep the core focused and shippable, the following are **deliberately out of scope** here:

* Web3/value-plane crates (wallets, ledgers, rewards, tokens)
* Ads, billing, or external webhook glue
* Sandbox/module systems for running untrusted user code
* Experimental runtimes and “labs” projects

Those will remain in the original RustyOnions repository (or future sibling repos) and only be wired in **after** RON-CORE is stable.

---

## License

Unless otherwise noted, code in this repository is dual-licensed under:

* MIT License, and
* Apache License 2.0

You may choose either license when using, copying, modifying, or distributing this code.

---

## Getting Help

If you’d like to:

* build on RON-CORE,
* integrate a new SDK,
* or help with docs/tests,

please open an issue or discussion in this repo with:

* what you’re trying to build,
* which crates you’re touching,
* and any logs/tests you’ve already run.

## 🙌 Credits

Created by **Stevan White** with assistance from **OpenAI’s ChatGPT** and **xAI’s Grok**.

```
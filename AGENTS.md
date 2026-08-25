# AGENTS.md

Instructions for an AI agent (or a human) to install, verify, and build on the VoiceOS Native OAuth Engine end to end. Follow the steps in order; every command is copy-pasteable.

## What you are installing

A provider-agnostic OAuth engine for VoiceOS custom integrations. Your integration calls four verbs (`connect`, `getToken`, `disconnect`, `getConnectStatus`) and never touches a token, a redirect, or a refresh timer. Everything a provider does differently lives in one `provider.json` config file; there is no per-provider code anywhere.

## Requirements

- Node.js `>=22.18` (runs TypeScript natively; the engine has zero runtime dependencies)
- No account, no registration, and no secret is needed to see it work end to end

## Step 1: Get the code and prove it works

```bash
git clone https://github.com/rithvik-bk/voiceos-oauth-engine
cd voiceos-oauth-engine
npm ci        # build-time toolchain only (TypeScript + vitest)
make demo     # real engine, real HTTP OAuth, built-in mock provider
make verify   # the full gate: build, typecheck, invariants, secret scan, all 625 tests
```

`make demo` runs the whole round trip against a built-in mock authorization server: connect opens the flow, the token is exchanged and vaulted, a protected resource is called with a token the calling code never sees, then the token is force-expired to prove the silent refresh path. `make verify` must end with every gate green before you build anything on top.

## Step 2: Describe your provider in one file

Create a `provider.json` capability profile. A minimal public PKCE profile:

```jsonc
{
  "name": "acme",
  "display_name": "Acme",
  "authorize_url": "https://acme.example/oauth/authorize",
  "token_url": "https://acme.example/oauth/token",
  "scopes": ["read", "write"],
  "pkce": "S256",
  "client_auth": "none",
  "client_id": "public-client-id"
}
```

Every field is validated against the frozen [`provider.schema.json`](provider.schema.json). Never put a `client_secret` in this file; the secret scanner fails the build if anything secret-shaped is committed.

If you do not know the provider's capabilities, measure them instead of guessing:

```bash
node tools/probe.mjs https://provider.example --name provider
```

The probe runs a safe measurement sequence and writes `providers/<name>.json` plus an evidence receipt. Anything it cannot determine is written as `unknown`, never guessed.

## Step 3: Register the provider and call the four verbs

```js
import { registerProvider } from './engine/src/index.ts';
registerProvider(profile);   // profile = your parsed provider.json
```

From your integration: `connect(profile)` starts the browser approval and returns in about a second while consent completes out of band; `getToken(provider)` always returns a token valid right now, with refresh happening behind the call; `disconnect(provider)` destroys the vault entry. Tokens live encrypted in the OS login Keychain and never transit VoiceOS or any third-party server.

## Step 4: Verify your work before shipping

```bash
make verify
```

The gate runs cheap-fails-first and stops with a non-zero exit on any failure. A fully worked walkthrough lives in [docs/ADD-A-PROVIDER.md](docs/ADD-A-PROVIDER.md); the runnable reference is `tools/mock-provider/` plus `make demo`.

## Where everything lives

| Path | What it is |
|:---|:---|
| `engine/` | The one engine; `engine/src/config.ts` is the sole source of the redirect URI |
| `relay/` | The stateless relay reference for confidential-client custody classes |
| `provider.schema.json` | The frozen provider-profile contract |
| `tools/` | `demo.mjs`, `mock-provider/`, `probe.mjs`, `scan-secrets.mjs`, `drift-check.mjs` |
| `docs/GETTING-STARTED.md` | Why it exists and how to ship your first integration |
| `docs/ARCHITECTURE.md` | Code-level design and where every decision lives |
| `docs/ADD-A-PROVIDER.md` | The fully worked add-a-provider walkthrough |
| `docs/THREAT-MODEL.md` | The attack catalog, each entry mapped to the test that defends it |

## Rules for agents operating this repo

- Never write `if (provider === 'x')` anywhere; a guard test greps the source and fails the build on any per-provider branch.
- Never commit anything secret-shaped; `tools/scan-secrets.mjs` runs in the verify gate and fails on a hit.
- Never bypass `make verify`; green means the entire contract holds, and there is no partial credit.

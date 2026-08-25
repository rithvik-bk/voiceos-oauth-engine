# Getting started

## Why this exists

VoiceOS lets anyone build a custom integration, but its manifest's `auth.kind: "oauth2"` slot has nothing behind it. In practice that means every custom integration either pastes a raw API key into a text field (where it sits in plaintext on disk) or routes through a third-party broker like Composio (where your users' tokens live on someone else's servers).

This engine is the third option: **native OAuth**. The user says "connect Slack", the provider's own approval page opens in their browser, and the token is minted directly into the macOS login Keychain, encrypted, on their machine. No broker, no pasted secrets, no token ever transiting another server.

## What it lets you do

- Ship a VoiceOS custom integration where connecting an account is **one tap on the provider's real approval page**.
- Support **any OAuth provider you can describe in one JSON file**. PKCE, client auth, device grant, and dynamic client registration are all driven by config, never by per-provider code.
- Write your integration **as if OAuth did not exist**. Your tool handlers call four verbs and never touch a token, a redirect, or a refresh timer:

```ts
connect(profile)        // opens the provider's approval page, returns fast
getToken(provider)      // a token valid right now; refresh happens behind this call
disconnect(provider)    // forget everything; vault entry destroyed
getConnectStatus(p)     // is this account connected?
```

## Ship your first integration

1. **Copy a provider profile.** Start from an existing `provider.json` and fill in the measured fields for your provider: endpoints, PKCE support, redirect rules, token shape. The full schema is `provider.schema.json`, and every field is documented.
2. **Write your tools.** Each handler asks `getToken()` for a live token and does its job. Zero auth code.
3. **Verify.** `make verify` runs the whole contract: the test suite, the secret scanner, and the drift checker, in one command.
4. **Install into VoiceOS** as a custom MCP integration and say "connect".

A complete worked example is the [Slack integration](https://github.com/rithvik-bk/voiceos-slack-integration), which does all of the above in a self-contained repo.

## Understand more

- [ARCHITECTURE](ARCHITECTURE.md): the one-engine design, the grant paths, and the custody classes.
- [ADD-A-PROVIDER](ADD-A-PROVIDER.md): the full walkthrough for describing a new provider in one JSON file.
- [THREAT-MODEL](THREAT-MODEL.md): what is stored where, what an attacker can and cannot reach, and why no secret ever ships in this repo.

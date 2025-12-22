<p align="center">
  <img src="logo.png" alt="OKAP Logo" width="150">
</p>

# OKAP: Open Key Access Protocol

**OKAP** (Open Key Access Protocol) is a proposed standard for secure, user-controlled API key delegation. Think OAuth, but for API keys.

## The Problem

Developers have API keys for multiple AI providers (OpenAI, Anthropic, Google, etc.). Every app wants these keys pasted manually. This creates:

- **Security risks**: Keys scattered across apps with no visibility
- **No revocation**: Can't easily disable one app's access
- **No limits**: Apps get full access to your quota
- **No audit trail**: No idea who's using what

## Who is this for?

**If you're a user with API keys:**
- Run a vault (locally or hosted)
- Add your keys once
- Give apps your vault URL instead of your key
- Revoke any app anytime

**If you're building an app that needs AI:**
- Implement OKAP instead of asking users to paste keys
- Users connect their vault
- You get a scoped token, never see their real key

```
Today:   "Paste your OpenAI key" → user pastes sk-abc123
         "Paste your OpenAI key" → user pastes sk-abc123
         
With OKAP: "Connect your vault" → user enters vault URL → done
           "Connect your vault" → user enters vault URL → done
```

## The Solution

OKAP defines a standard flow for apps to request API key access and users to grant scoped, revocable tokens.

```
┌──────────────┐     1. Request access      ┌──────────────┐
│              │ ─────────────────────────▶ │              │
│     App      │                            │   OKAP Vault │
│              │ ◀───────────────────────── │              │
└──────────────┘     4. Scoped token        └──────────────┘
                                                   │
                                            2. User prompt
                                                   │
                                                   ▼
                                            ┌──────────────┐
                                            │     User     │
                                            │   approves   │
                                            └──────────────┘
```

## Specification

See [SPEC.md](./SPEC.md) for the full protocol specification.

## Quick Example

### App requests access:

```json
{
  "okap": "1.0",
  "request": {
    "provider": "openai",
    "models": ["gpt-4", "gpt-4o-mini"],
    "capabilities": ["chat", "embeddings"],
    "limits": {
      "monthly_spend": 10.00,
      "requests_per_minute": 60
    },
    "expires": "2025-03-01"
  }
}
```

### User sees:

```
┌─────────────────────────────────────────────────┐
│                                                 │
│   Example App wants to use your OpenAI key      │
│                                                 │
│   Provider:  OpenAI                             │
│   Models:    gpt-4, gpt-4o-mini                 │
│   Limit:     $10/month                          │
│   Expires:   March 1, 2025                      │
│                                                 │
│              ┌───────┐  ┌────────┐              │
│              │ Allow │  │  Deny  │              │
│              └───────┘  └────────┘              │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Headless mode:** For personal use or trusted environments, vaults can auto-approve requests without user interaction.

### App receives:

```json
{
  "okap": "1.0",
  "token": "okap_abc123...",
  "base_url": "https://vault.example.com/v1",
  "expires": "2025-03-01T00:00:00Z"
}
```

The app uses `base_url` instead of `api.openai.com`. The vault proxies requests using the user's real key.

## Key Principles

1. **User owns keys**: Master keys never leave the vault
2. **Scoped access**: Apps get exactly what they request, nothing more
3. **Revocable**: Users can revoke app access instantly
4. **Auditable**: Full visibility into usage per app
5. **Standard**: Any vault can implement OKAP

## Status

**Draft** - Looking for feedback and collaborators.

## Contributing

Open an issue or PR. Let's make API key management less painful.

## Implementations

- **Python SDK**: [openkeyprotocol/python-sdk](https://github.com/openkeyprotocol/python-sdk) - `pip install okap`
- **Reference Server**: [openkeyprotocol/server](https://github.com/openkeyprotocol/server)

## License

CC0 1.0 Universal - Public Domain

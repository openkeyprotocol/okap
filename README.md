<p align="center">
  <img src="logo.png" alt="OKAP Logo" width="150">
</p>

# OKAP: Open Key Access Protocol

**OKAP** (Open Key Access Protocol) is a lightweight protocol for secure, user-controlled AI API key delegation. Inspired by OAuth 2.0 RAR (RFC 9396), OKAP uses structured authorization details instead of scope strings for fine-grained access control.

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
  "authorization_details": [
    {
      "type": "ai_model_access",
      "provider": "openai",
      "models": ["gpt-4", "gpt-4o-mini"],
      "capabilities": ["chat", "embeddings"],
      "limits": {
        "monthly_spend": 10.00,
        "requests_per_minute": 60
      },
      "expires": "2025-03-01"
    }
  ],
  "client": {
    "name": "Example App",
    "url": "https://app.example.com",
    "callback": "https://app.example.com/callback"
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
  "status": "granted",
  "token": "okap_abc123...",
  "authorization_details": [
    {
      "type": "ai_model_access",
      "provider": "openai",
      "models": ["gpt-4", "gpt-4o-mini"],
      "base_url": "https://vault.example.com/v1/openai",
      "expires": "2025-03-01T00:00:00Z"
    }
  ]
}
```

The app uses `base_url` from the enriched authorization details. The vault proxies requests using the user's real key.

## Key Principles

1. **User owns keys**: Master keys never leave the vault
2. **Scoped access**: Apps get exactly what they request, nothing more
3. **Revocable**: Users can revoke app access instantly
4. **Auditable**: Full visibility into usage per app
5. **Transport-agnostic**: Works with browser extensions, HTTP, OAuth, etc.
6. **Structured permissions**: Uses typed authorization details, not scope strings

## Why Not OAuth Scopes?

OKAP uses structured `authorization_details` instead of scope strings. Compare:

**Scope approach** (what we avoid):
```
scope=ai:openai:gpt-4:chat:limit:10
```

**OKAP approach** (structured):
```json
{
  "type": "ai_model_access",
  "provider": "openai",
  "models": ["gpt-4"],
  "limits": { "monthly_spend": 10.00 }
}
```

Structured authorization details provide better validation, easier parsing, and clearer semantics.

## Status

**v1.0** - Built on RFC 9396 (OAuth 2.0 Rich Authorization Requests). See [docs/migration-guide.md](./docs/migration-guide.md) for changes from v0.x.

## Documentation

- [Full Specification](./SPEC.md) - Complete OKAP protocol specification
- [RAR Type Specification](./docs/rar-type-spec.md) - `ai_model_access` authorization type
- [Migration Guide](./docs/migration-guide.md) - Upgrading from v0.x to v1.0

## Contributing

Open an issue or PR. Let's make API key management less painful.

## Implementations

- **Python SDK**: [openkeyprotocol/python-sdk](https://github.com/openkeyprotocol/python-sdk) - `pip install okap`
- **Reference Server**: [openkeyprotocol/server](https://github.com/openkeyprotocol/server)

## License

CC0 1.0 Universal - Public Domain

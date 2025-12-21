# OKAP: Open Key Access Protocol

**OKAP** (Open Key Access Protocol) is a proposed standard for secure, user-controlled API key delegation. Think OAuth, but for API keys.

## The Problem

Developers have API keys for multiple AI providers (OpenAI, Anthropic, Google, etc.). Every app wants these keys pasted manually. This creates:

- **Security risks**: Keys scattered across apps with no visibility
- **No revocation**: Can't easily disable one app's access
- **No limits**: Apps get full access to your quota
- **No audit trail**: No idea who's using what

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

> **Example App** wants to use your OpenAI API key
> - Models: GPT-4, GPT-4o-mini
> - Limit: $10/month
> - Expires: March 1, 2025
> 
> [Allow] [Deny]

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

## License

CC0 1.0 Universal - Public Domain

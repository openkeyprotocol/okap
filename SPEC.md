# OKAP Specification v1.0 (Draft)

**Open Key Access Protocol**

## Abstract

OKAP (Open Key Access Protocol) defines a standard for applications to request scoped access to a user's API keys, and for users to grant or deny that access through a trusted vault. The protocol enables secure delegation of API key access without exposing master keys to third-party applications.

## 1. Overview

### 1.1 Terminology

- **User**: The person who owns API keys for various providers
- **App**: A third-party application requesting API key access
- **Vault**: A trusted service or extension that stores user's master keys and issues scoped tokens
- **Provider**: An API service (e.g., OpenAI, Anthropic, Google)
- **Master Key**: The user's actual API key for a provider
- **OKAP Token**: A scoped, revocable token issued by the vault

### 1.2 Goals

1. Users never paste master keys into third-party apps
2. Apps receive scoped tokens with defined limits
3. Users can revoke access at any time
4. Full audit trail of API usage per app

## 2. Protocol Flow

```
┌─────────┐                    ┌─────────┐                    ┌─────────┐
│   App   │                    │  Vault  │                    │  User   │
└────┬────┘                    └────┬────┘                    └────┬────┘
     │                              │                              │
     │  1. OKAP Request             │                              │
     │─────────────────────────────▶│                              │
     │                              │                              │
     │                              │  2. Authorization Prompt     │
     │                              │─────────────────────────────▶│
     │                              │                              │
     │                              │  3. User Decision            │
     │                              │◀─────────────────────────────│
     │                              │                              │
     │  4. OKAP Response            │                              │
     │◀─────────────────────────────│                              │
     │                              │                              │
     │  5. API Calls via Vault      │                              │
     │─────────────────────────────▶│                              │
     │                              │                              │
     │                              │  6. Proxy to Provider        │
     │                              │─────────────────────────────▶│
     │                              │                              │
```

## 3. Request Format

### 3.1 OKAP Request Object

```json
{
  "okap": "1.0",
  "request": {
    "provider": "<provider_id>",
    "models": ["<model_id>", ...],
    "capabilities": ["<capability>", ...],
    "limits": {
      "monthly_spend": <number>,
      "daily_spend": <number>,
      "requests_per_minute": <number>,
      "requests_per_day": <number>
    },
    "expires": "<ISO 8601 date>",
    "reason": "<human-readable reason>"
  },
  "client": {
    "name": "<app_name>",
    "url": "<app_url>",
    "callback": "<callback_url>"
  }
}
```

### 3.2 Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `okap` | string | Yes | Protocol version, currently "1.0" |
| `request.provider` | string | Yes | Provider identifier (e.g., "openai", "anthropic") |
| `request.models` | array | No | Specific models requested. Empty = all available |
| `request.capabilities` | array | No | Capabilities requested (e.g., "chat", "embeddings", "images") |
| `request.limits.monthly_spend` | number | No | Maximum spend per month in USD |
| `request.limits.daily_spend` | number | No | Maximum spend per day in USD |
| `request.limits.requests_per_minute` | number | No | Rate limit per minute |
| `request.limits.requests_per_day` | number | No | Maximum requests per day |
| `request.expires` | string | No | Token expiration date (ISO 8601) |
| `request.reason` | string | No | Human-readable reason for access |
| `client.name` | string | Yes | Application name |
| `client.url` | string | No | Application URL |
| `client.callback` | string | No | Callback URL for response |

### 3.3 Provider Identifiers

Standard provider identifiers:

| Provider | Identifier |
|----------|------------|
| OpenAI | `openai` |
| Anthropic | `anthropic` |
| Google AI | `google` |
| Groq | `groq` |
| Together AI | `together` |
| Mistral | `mistral` |
| Cohere | `cohere` |

### 3.4 Capabilities

Standard capabilities:

| Capability | Description |
|------------|-------------|
| `chat` | Chat completions |
| `embeddings` | Text embeddings |
| `images` | Image generation |
| `audio` | Audio transcription/generation |
| `code` | Code-specific models |
| `vision` | Vision/multimodal |

## 4. Response Format

### 4.1 Success Response

```json
{
  "okap": "1.0",
  "status": "granted",
  "token": "<okap_token>",
  "base_url": "<proxy_base_url>",
  "expires": "<ISO 8601 datetime>",
  "limits": {
    "monthly_spend": <number>,
    "requests_per_minute": <number>
  }
}
```

### 4.2 Denial Response

```json
{
  "okap": "1.0",
  "status": "denied",
  "reason": "<optional_reason>"
}
```

### 4.3 Fields

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | "granted" or "denied" |
| `token` | string | OKAP token for API calls (prefixed with `okap_`) |
| `base_url` | string | Base URL for API calls (vault proxy) |
| `expires` | string | Token expiration datetime |
| `limits` | object | Actual limits applied (may differ from requested) |

## 5. API Usage

### 5.1 Using OKAP Token

Apps use the token and base_url to make API calls:

```python
from openai import OpenAI

client = OpenAI(
    api_key="okap_abc123...",
    base_url="https://vault.example.com/v1"
)

response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Hello"}]
)
```

### 5.2 Vault Proxying

The vault:
1. Validates the OKAP token
2. Checks limits and quotas
3. Substitutes the master key
4. Proxies the request to the provider
5. Logs the usage
6. Returns the response

## 6. Token Management

### 6.1 Token Format

OKAP tokens are opaque strings prefixed with `okap_`:

```
okap_<base64_encoded_payload>
```

### 6.2 Revocation

Users can revoke tokens at any time via the vault interface. Revoked tokens return:

```json
{
  "error": {
    "type": "token_revoked",
    "message": "This OKAP token has been revoked"
  }
}
```

### 6.3 Expiration

Expired tokens return:

```json
{
  "error": {
    "type": "token_expired",
    "message": "This OKAP token has expired"
  }
}
```

## 7. Transport

### 7.1 Browser Extension

For web apps, the vault can be a browser extension. The app sends a message:

```javascript
window.postMessage({
  type: 'OKAP_REQUEST',
  request: { /* OKAP request object */ }
}, '*');

window.addEventListener('message', (event) => {
  if (event.data.type === 'OKAP_RESPONSE') {
    // Handle response
  }
});
```

### 7.2 Native Apps

For native apps, the vault can register a URL scheme:

```
okap://request?data=<base64_encoded_request>
```

### 7.3 Server-to-Server

For server-side apps, the vault exposes an HTTP endpoint:

```
POST https://vault.example.com/okap/authorize
Content-Type: application/json

{ /* OKAP request object */ }
```

## 8. Vault Ownership Models

OKAP is vault-agnostic. The protocol works regardless of who operates the vault. Users choose their trust model.

### 8.1 Self-Hosted

Users run their own vault on local hardware or personal cloud infrastructure.

**Examples**: llmux, LiteLLM proxy, custom implementation

| Pros | Cons |
|------|------|
| Full control | Requires technical setup |
| No third-party trust | User manages uptime |
| Keys never leave your infrastructure | No managed updates |

**Best for**: Developers, privacy-conscious users, enterprises with compliance requirements.

### 8.2 Browser Extension

A browser extension acts as the vault, storing keys locally or in user-controlled encrypted storage.

**Examples**: Hypothetical "OKAP Wallet" extension, 1Password integration

| Pros | Cons |
|------|------|
| Easy setup | Browser-only |
| Keys stay local | Extension trust required |
| Works with any web app | Mobile support limited |

**Best for**: Individual developers, casual users, web-first workflows.

### 8.3 Third-Party Service

A managed service operates the vault on behalf of users.

**Examples**: Cloudflare AI Gateway, Eden AI, Portkey

| Pros | Cons |
|------|------|
| No infrastructure to manage | Trust third party with traffic |
| Professional uptime/support | Potential vendor lock-in |
| Additional features (caching, analytics) | Monthly costs |

**Best for**: Teams, startups, users who prioritize convenience over control.

### 8.4 Provider-Native

AI providers (OpenAI, Anthropic, Google) implement OKAP directly, issuing scoped tokens from their own dashboards.

**Examples**: None yet (future possibility)

| Pros | Cons |
|------|------|
| No proxy needed | Requires provider adoption |
| Native integration | Each provider implements separately |
| Maximum performance | User still manages multiple dashboards |

**Best for**: Everyone, if providers adopt the standard.

### 8.5 Hybrid

Combine models: use a browser extension for web apps, self-hosted for production servers, provider-native when available.

OKAP tokens are interchangeable. An app doesn't know (or care) which vault type issued the token.

## 9. Security Considerations

### 9.1 Token Security

- Tokens SHOULD be transmitted over HTTPS only
- Tokens SHOULD be stored securely (not in localStorage for web apps)
- Tokens SHOULD have reasonable expiration times

### 9.2 Vault Security

- Vaults MUST encrypt master keys at rest
- Vaults MUST authenticate users before showing authorization prompts
- Vaults SHOULD support 2FA for sensitive operations

### 9.3 Request Validation

- Vaults SHOULD verify client identity when possible
- Vaults SHOULD display clear information about the requesting app
- Users SHOULD be warned about unusual requests

## 10. Future Extensions

- Multi-provider requests (request access to multiple providers at once)
- Delegation chains (apps granting sub-tokens)
- Usage webhooks (notify users of spending thresholds)
- Provider-native support (providers issuing OKAP tokens directly)

## Appendix A: Example Implementations

### A.1 Browser Extension (Conceptual)

```javascript
// Content script listens for OKAP requests
window.addEventListener('message', async (event) => {
  if (event.data.type !== 'OKAP_REQUEST') return;
  
  // Send to extension background script
  const response = await chrome.runtime.sendMessage({
    type: 'OKAP_AUTHORIZE',
    request: event.data.request
  });
  
  window.postMessage({
    type: 'OKAP_RESPONSE',
    response
  }, '*');
});
```

### A.2 Vault Proxy (Conceptual)

```python
@app.route('/v1/chat/completions', methods=['POST'])
def proxy_chat():
    token = request.headers.get('Authorization').replace('Bearer ', '')
    
    # Validate OKAP token
    grant = validate_okap_token(token)
    if not grant:
        return {'error': 'Invalid token'}, 401
    
    # Check limits
    if not check_limits(grant):
        return {'error': 'Limit exceeded'}, 429
    
    # Get master key and proxy
    master_key = get_master_key(grant.user_id, grant.provider)
    
    response = requests.post(
        f'https://api.openai.com/v1/chat/completions',
        headers={'Authorization': f'Bearer {master_key}'},
        json=request.json
    )
    
    # Log usage
    log_usage(grant, response)
    
    return response.json()
```

## Changelog

- **v1.0 (Draft)** - Initial specification

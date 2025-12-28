# OKAP Specification v1.0 (Draft)

**Open Key Access Protocol**

## Abstract

OKAP (Open Key Access Protocol) is a lightweight protocol for applications to request scoped access to a user's AI provider API keys. Inspired by OAuth 2.0 Rich Authorization Requests (RFC 9396), OKAP uses structured authorization details instead of scope strings to express fine-grained access requirements. Users grant or deny access through a trusted vault that proxies requests. The protocol enables secure delegation of API key access without exposing master keys to third-party applications.

OKAP is transport-agnostic and works with browser extensions, native apps, and server-to-server flows. It can also be adopted by OAuth 2.0 authorization servers as a RAR type.

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

OKAP adopts a structured authorization details pattern inspired by RFC 9396, avoiding scope strings in favor of explicit JSON objects.

### 3.1 OKAP Request Structure

An OKAP authorization request contains:

```json
{
  "okap": "1.0",
  "authorization_details": [...],
  "client": {...}
}
```

### 3.2 Authorization Details Array

The `authorization_details` array contains one or more authorization objects, each with type `ai_model_access`:

```json
[
  {
    "type": "ai_model_access",
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
  }
]
```

**Example complete request:**

```json
{
  "okap": "1.0",
  "authorization_details": [
    {
      "type": "ai_model_access",
      "provider": "openai",
      "models": ["gpt-4"],
      "limits": { "monthly_spend": 10.00 }
    }
  ],
  "client": {
    "name": "Example App",
    "url": "https://app.example.com",
    "callback": "https://app.example.com/callback"
  }
}
```

### 3.3 Authorization Details Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Must be "ai_model_access" |
| `provider` | string | Yes | Provider identifier (e.g., "openai", "anthropic") |
| `models` | array | No | Specific models requested. Empty = all available |
| `capabilities` | array | No | Capabilities requested (e.g., "chat", "embeddings", "images") |
| `limits.monthly_spend` | number | No | Maximum spend per month in USD |
| `limits.daily_spend` | number | No | Maximum spend per day in USD |
| `limits.requests_per_minute` | number | No | Rate limit per minute |
| `limits.requests_per_day` | number | No | Maximum requests per day |
| `expires` | string | No | Token expiration date (ISO 8601) |
| `reason` | string | No | Human-readable reason for access |

### 3.4 Client Information

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `client.name` | string | Yes | Application name |
| `client.url` | string | Recommended | Application URL |
| `client.callback` | string | Context-dependent | Callback URL for response |

### 3.5 Provider Identifiers

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

### 3.6 Capabilities

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

### 4.1 Grant Response

When a user grants access, the vault returns:

```json
{
  "okap": "1.0",
  "status": "granted",
  "token": "okap_<token>",
  "authorization_details": [
    {
      "type": "ai_model_access",
      "provider": "openai",
      "models": ["gpt-4"],
      "capabilities": ["chat"],
      "limits": {
        "monthly_spend": 10.00,
        "requests_per_minute": 60
      },
      "base_url": "https://vault.example.com/v1/openai",
      "expires": "2025-03-01T00:00:00Z"
    }
  ]
}
```

**Note:** The `authorization_details` in the response MAY be enriched with vault-specific fields like `base_url` and actual granted permissions (which may differ from requested).

### 4.2 Denial Response

```json
{
  "okap": "1.0",
  "status": "denied",
  "reason": "User declined authorization request"
}
```

### 4.3 Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `okap` | string | Protocol version |
| `status` | string | "granted" or "denied" |
| `token` | string | OKAP token (prefixed with `okap_`) |
| `authorization_details` | array | Enriched authorization details showing what was granted |
| `authorization_details[].base_url` | string | Provider-specific vault proxy endpoint |
| `authorization_details[].expires` | string | Token expiration (ISO 8601) |
| `reason` | string | Reason for denial (only in denial response) |

## 5. API Usage

### 5.1 Using OKAP Token

Apps use the access_token from the OAuth token response:

```python
from openai import OpenAI

client = OpenAI(
    api_key="okap_abc123...",
    base_url="https://vault.example.com/v1/openai"
)

response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Hello"}]
)
```

Note: The `base_url` is obtained from the `authorization_details[].base_url` field in the token response.

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

OKAP is transport-agnostic. Different deployment scenarios use different transport mechanisms.

### 7.1 Browser Extension (Primary Flow)

For web apps, a browser extension acts as the vault:

```javascript
// App initiates OKAP request
window.postMessage({
  type: 'OKAP_REQUEST',
  okap: '1.0',
  authorization_details: [{
    type: 'ai_model_access',
    provider: 'openai',
    models: ['gpt-4'],
    limits: { monthly_spend: 10.00 }
  }],
  client: {
    name: 'My App',
    url: window.location.origin
  }
}, '*');

// Vault responds
window.addEventListener('message', (event) => {
  if (event.data.type === 'OKAP_RESPONSE') {
    const { status, token, authorization_details } = event.data;
    if (status === 'granted') {
      const baseUrl = authorization_details[0].base_url;
      // Use token and baseUrl
    }
  }
});
```

### 7.2 HTTP Endpoint

For server-to-server flows, the vault exposes an HTTP endpoint:

```
POST /okap/authorize
Content-Type: application/json

{
  "okap": "1.0",
  "authorization_details": [...],
  "client": {...}
}
```

Response:
```json
{
  "okap": "1.0",
  "status": "granted",
  "token": "okap_...",
  "authorization_details": [...]
}
```

### 7.3 Custom URL Scheme (Native Apps)

For native applications:

```
okap://authorize?request=<base64_encoded_json>
```

The vault app handles the URL, shows authorization UI, and returns via callback URL or deep link.

### 7.4 OAuth 2.0 Integration (Optional)

OKAP `authorization_details` can be used with OAuth 2.0 flows:

```
GET /oauth/authorize?
  response_type=code&
  client_id=my_app&
  authorization_details=<url_encoded_json>
```

OAuth servers can recognize the `ai_model_access` type and implement OKAP semantics. This allows OKAP to work within existing OAuth infrastructure.

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

### v1.0 (2025-01) - RAR-Inspired Structured Authorization

**Major Changes:**
- Adopted structured `authorization_details` pattern inspired by RFC 9396 (OAuth 2.0 Rich Authorization Requests)
- Positioned OKAP as **standalone protocol** (not OAuth extension) with optional OAuth compatibility
- **Eliminated scope strings** in favor of typed JSON authorization objects

**Request Format:**
- Added `okap` protocol version field
- Wrapped authorization details in `authorization_details` array
- Each object requires `type: "ai_model_access"`
- Replaced OAuth client parameters with OKAP `client` object

**Response Format:**
- Changed from OAuth token response to OKAP-native format
- Return `status: "granted"/"denied"` instead of OAuth success/error
- Enriched `authorization_details` with vault-specific fields (`base_url`, actual limits, etc.)
- Token field named `token` (not `access_token`)

**Transport:**
- Emphasized browser extension and HTTP flows as primary mechanisms
- Added custom URL scheme support (`okap://authorize`)
- OAuth 2.0 integration preserved as **optional** extension point
- Removed OAuth-specific requirements

**Documentation:**
- Created RAR type specification for `ai_model_access`
- Added migration guide from v0.x to v1.0
- Documented comparison: structured authorization vs. scope strings
- Added positioning document explaining protocol independence

**New Capabilities:**
- Multi-provider requests (multiple `authorization_details` objects in single request)
- Enriched authorization details in response
- Transport-agnostic design

### v0.x (Draft) - Initial Specification
- Custom JSON request format
- Basic vault proxy architecture
- Simple token issuance flow


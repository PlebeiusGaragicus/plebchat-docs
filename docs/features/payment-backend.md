# Payment Backend

The PlebChat backend is a FastAPI service that handles ecash payment validation, redemption, refunds, and wallet management. It serves as the payment infrastructure for the LangGraph AI agents.

## Architecture Overview

The backend uses a **proxy architecture** where it validates payment and starts LangGraph runs:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Frontend UI   │     │  LangGraph      │     │  Admin Panel    │
│   (Svelte)      │     │  Cloud          │     │  (Svelte)       │
└────────┬────────┘     └────────▲────────┘     └────────┬────────┘
         │                       │                       │
         │ POST /api/agent/start │ runs.joinStream()     │  NIP-98 Auth
         ▼                       │                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                     FastAPI Backend                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │  /api/agent/*   │  │  /api/pricing/* │  │  /api/admin/*   │  │
│  │  Start runs     │  │  Dynamic cost   │  │  NIP-98 auth    │  │
│  │  + validate pay │  │                 │  │                 │  │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘  │
│           │                    │                    │           │
│  ┌────────┴────────┐           │                    │           │
│  │  /api/wallet/*  │           │                    │           │
│  │  Token ops      │───────────┴────────────────────┘           │
│  └────────┬────────┘                                            │
│           │                                                     │
│           └──────────────┬──────────────────────────────────────┤
│                          │                                      │
│           ┌──────────────┴──────────────┐                       │
│           │  SQLite + models.json       │  API keys injected    │
│           │  (proofs, keysets, creds)   │  into RunnableConfig  │
│           └─────────────────────────────┘                       │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
                    ┌───────────────────────┐
                    │     Cashu Mint        │
                    │  (testnut default)    │
                    └───────────────────────┘
```

**Key Change:** The backend now acts as a proxy that:
1. Validates ecash payment before any LLM calls
2. Loads API credentials from `models.json`
3. Starts LangGraph runs via `client.runs.create()`
4. Frontend joins the stream via `client.runs.joinStream()`

## Payment Flow with Refunds

The payment flow follows a "pay-upfront, refund-difference" pattern with **backend-initiated runs**:

```
1. Frontend fetches upfront cost from GET /api/pricing/models
2. Client sends message + ecash token to Backend Proxy:
   POST /api/agent/start/{graph_id}
3. Backend validates and redeems token (BEFORE starting graph):
   - Validates token format (cashuA/cashuB)
   - Calls POST /api/wallet/receive to redeem
   - Creates payment_session_id for secure refund tracking
   - Loads API credentials from models.json
4. Backend starts LangGraph run:
   - Injects credentials + payment_session_id into RunnableConfig
   - Calls client.runs.create() with stream_mode=["messages", "values"]
   - Returns run_id and thread_id to frontend
5. Frontend joins stream:
   - Calls client.runs.joinStream(thread_id, run_id)
   - Displays AI response in real-time
6. Agent processes the LLM request:
   - Gets credentials from config["configurable"]["model_credentials"]
   - Uses model_id from config to select LLM
   - Extracts token usage (prompt + completion tokens)
7. After LLM response, agent calculates actual cost:
   - Calls POST /api/pricing/calculate with model + usage
   - Backend returns cost in sats and refund amount
8. If refund > 0, agent calls POST /api/wallet/refund:
   - Uses payment_session_id from RunnableConfig
   - Backend validates session and generates ecash refund token
9. Agent returns refund_token to client via stream
10. Client auto-redeems refund token to their wallet
```

This pattern ensures:
- **No free LLM calls**: Payment validated before graph starts
- **API keys never reach frontend**: Injected server-side only
- **Fair pricing**: User only pays for actual usage
- **Secure refunds**: Session-based tracking prevents refund exploits

---

## Dynamic Pricing

Pricing and credentials are configured centrally in `backend/data/models.json`:

```json
{
  "models": {
    "grok-4-1-fast-non-reasoning": {
      "provider": "xai",
      "display_name": "Grok 4.1 Fast",
      "base_url": "https://api.x.ai/v1",
      "api_key": "${XAI_API_KEY}",
      "pricing": {
        "input_usd_per_million": 0.2,
        "output_usd_per_million": 0.5
      },
      "modalities": ["text"],
      "max_context_tokens": 131072,
      "enabled": true
    },
    "zai-org/glm-4.6v-flash": {
      "provider": "local",
      "display_name": "GLM 4.6v Flash",
      "base_url": "${LLM_BASE_URL}",
      "api_key": "${LLM_API_KEY}",
      "pricing": {
        "input_usd_per_million": 0.05,
        "output_usd_per_million": 0.1
      },
      "modalities": ["text", "vision"],
      "max_context_tokens": 32768,
      "enabled": true
    }
  },
  "default_model": "grok-4-1-fast-non-reasoning",
  "upfront_sats": {
    "default": 8,
    "chat": 8,
    "deep-research": 50
  }
}
```

**Key fields:**
- `api_key` / `base_url`: Support `${ENV_VAR}` syntax for environment variable expansion
- `pricing`: Nested object with input/output USD pricing per million tokens
- `upfront_sats`: Per-agent upfront costs (object with agent IDs as keys)

### Key Pricing Concepts

| Field | Description |
|-------|-------------|
| `upfront_sats` | Amount user pays upfront (power of 2 recommended for fee minimization) |
| `input_price_usd_per_million` | Cost per 1M input tokens in USD |
| `output_price_usd_per_million` | Cost per 1M output tokens in USD |
| `enabled` | Whether model is available to users |
| `provider` | LLM provider (`xai`, `local`, `openai`) |

### Why Power of 2 for Upfront Cost?

Cashu tokens use power-of-2 denominations (1, 2, 4, 8, 16, 32...). Using `upfront_sats: 8` means:
- **Payment**: Single 8-sat proof, no swap needed → **0 fee**
- **Refund**: Small amounts (e.g., 7 sats = 4+2+1) → minimal proofs → **1 sat max fee**

With 50 sats, you'd need multiple proofs (32+16+2) requiring swaps with fees.

---

## Payment Sessions (Secure Refunds)

To prevent refund exploits, each payment creates a session:

```python
class PaymentSession:
    id: str                  # UUID
    original_amount: int     # Amount paid upfront
    created_at: float        # Timestamp
    refunded: bool           # True if refund issued
    refund_amount: int       # Amount refunded
```

### Security Rules

1. **Session required**: Refund requests must include valid `payment_session_id`
2. **Single refund**: Each session can only be refunded once
3. **Amount validation**: Refund cannot exceed original payment
4. **Expiry**: Sessions expire after 1 hour
5. **Cleanup**: Expired sessions are periodically removed

### Why Sessions?

Without sessions, the refund endpoint would be exploitable:
- Attacker could call `/refund` with any amount
- No way to verify they actually paid
- Could drain the wallet with fake refund requests

With sessions:
- Refund tied to a specific, verified payment
- Amount capped at what was actually paid
- One-time use prevents replay attacks

---

## Mint Fees

Cashu mints charge fees for token swaps (creating change). Understanding fees helps optimize costs.

### Fee Calculation

```
fee = ceil(num_proofs × input_fee_ppk / 1000)
```

Where `input_fee_ppk` is "parts per thousand per input proof" (typically 100 = 0.1 sat/proof).

| Proofs | Fee (ppk=100) |
|--------|---------------|
| 1-10   | 1 sat         |
| 11-20  | 2 sats        |
| 21-30  | 3 sats        |

### Fee Absorption

Users absorb mint swap fees on refunds. With `include_fees=False`:
- User pays 8 sats
- We refund 7 sats
- Mint charges 1 sat fee
- User receives 6 sats

This is intentional - the fee is predictable and small with power-of-2 upfront amounts.

---

## Wallet API Endpoints

All wallet endpoints are prefixed with `/api/wallet`.

### POST /receive

Redeem an ecash token and create a payment session.

**Request:**
```json
{
  "token": "cashuBo2F0gaJhaUgA2..."
}
```

**Response:**
```json
{
  "success": true,
  "amount": 8,
  "error": null,
  "mint": "https://testnut.cashu.space",
  "payment_session_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### POST /refund

Generate a refund token (requires valid payment session).

**Request:**
```json
{
  "amount": 7,
  "payment_session_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "memo": "PlebChat refund - 7 sats unused"
}
```

**Response:**
```json
{
  "success": true,
  "token": "cashuBo2F0gaJhaUgA2...",
  "amount": 7,
  "error": null
}
```

**Error responses:**
- `400`: Invalid payment session ID
- `400`: Session already refunded
- `400`: Refund amount exceeds original payment

### POST /check

Validate a token without redeeming it.

**Request:**
```json
{
  "token": "cashuBo2F0gaJhaUgA2..."
}
```

**Response:**
```json
{
  "valid": true,
  "spent": false,
  "amount": 8,
  "error": null
}
```

### GET /balance

Get current wallet balance.

**Response:**
```json
{
  "balance": 1250,
  "unit": "sat"
}
```

---

## Pricing API Endpoints

All pricing endpoints are prefixed with `/api/pricing`.

### GET /models

Get available models with pricing (public endpoint for frontend).

**Response:**
```json
{
  "models": [
    {
      "model_id": "grok-4-1-fast-non-reasoning",
      "provider": "xai",
      "display_name": "Grok 4.1 Fast",
      "input_price_sats_per_million": 3,
      "output_price_sats_per_million": 10,
      "enabled": true
    }
  ],
  "btc_price_usd": 100000.00,
  "upfront_sats": 8,
  "default_model": "grok-4-1-fast-non-reasoning"
}
```

### POST /calculate

Calculate cost for token usage (called by agent after LLM response).

**Request:**
```json
{
  "model_id": "grok-4-1-fast-non-reasoning",
  "prompt_tokens": 150,
  "completion_tokens": 500
}
```

**Response:**
```json
{
  "input_cost_sats": 1,
  "output_cost_sats": 1,
  "total_cost_sats": 1,
  "refund_sats": 7,
  "model_id": "grok-4-1-fast-non-reasoning"
}
```

---

## Admin API Endpoints

Admin endpoints require NIP-98 authentication and are prefixed with `/api/admin`.

### GET /mint/fees

Get fee structure of the connected mint.

**Response:**
```json
{
  "success": true,
  "mint_url": "https://testnut.cashu.space",
  "keysets": [
    {
      "id": "00abc123...",
      "unit": "sat",
      "active": true,
      "input_fee_ppk": 100
    }
  ],
  "active_keyset_fee_ppk": 100,
  "fee_explanation": "Fee: 100 ppk. Minimum fee is 1 sat.",
  "example_fees": {
    "1_proof": 1,
    "5_proofs": 1,
    "10_proofs": 1,
    "20_proofs": 2
  }
}
```

### POST /mint/explore

Explore fee structure of any Cashu mint.

**Request:**
```json
{
  "mint_url": "https://mint.example.com"
}
```

**Response:** Same format as `/mint/fees`

### Pricing Admin Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/pricing/models` | GET | List all models with full details |
| `/pricing/models` | POST | Add a new model |
| `/pricing/models/{id}` | PUT | Update a model |
| `/pricing/models/{id}` | DELETE | Delete a model |
| `/pricing/upfront` | PUT | Update upfront cost |
| `/pricing/btc` | GET | Get current BTC price |
| `/pricing/btc/refresh` | POST | Force refresh BTC price |

---

## Admin Panel

The admin panel (`admin/`) is a Svelte 5 application providing:

### Wallet Management
- **Stats** - Balance, proof count, keyset count, connection status
- **Withdraw** - Generate ecash tokens with optional memos
- **Sweep** - One-click sweep all funds
- **Lightning Payout** - Send to Lightning address

### Mint Fees
- **Connected Mint** - View fee structure of your mint
- **Mint Explorer** - Query any mint's fees
  - Quick buttons for popular mints (Minibits, eNuts, Coinos, etc.)
  - Custom URL input
  - Fee comparison to find low-fee mints

### Model Pricing
- **BTC Price** - Current rate, refresh button
- **Upfront Cost** - Edit the upfront sats (power of 2 recommended)
- **Models Table** - Add/edit/delete models, toggle enabled status
- **Sat Pricing** - Auto-calculated from USD prices and BTC rate

### API Usage
- **Tavily API** - Usage stats for web search

---

## Trust Model

PlebChat uses a **non-custodial** design:

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT SIDE                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    CypherTap Component                   │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │    │
│  │  │ User Wallet │  │   Tokens    │  │  Mint (testnut) │  │    │
│  │  │ (browser)   │──│   (ecash)   │──│                 │  │    │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│                              │ payment token                     │
│                              ▼                                   │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                       SERVER SIDE                                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  PlebChat Backend                        │    │
│  │  - Validates token from TRUSTED mints only               │    │
│  │  - Redeems token, creates payment session                │    │
│  │  - Calculates actual cost after LLM response             │    │
│  │  - Generates secure refund token if overpaid             │    │
│  │  - Auto-payouts to operator's Lightning address          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│                              │ refund token                      │
│                              ▼                                   │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
                    ┌───────────────────────┐
                    │   Client auto-redeems │
                    │   refund to wallet    │
                    └───────────────────────┘
```

### Key Principles

1. **Client holds funds**: CypherTap stores user tokens in the browser
2. **Pay-upfront model**: User overpays, gets refunded the difference
3. **Session-secured refunds**: Payment sessions prevent refund exploits
4. **Backend is authoritative**: All pricing comes from backend `/api/pricing`
5. **No fallbacks**: If backend unavailable, operations fail (no hardcoded defaults)
6. **Trusted mints only**: Backend only accepts tokens from configured mints

---

## Configuration

### Required Environment Variables

| Variable | Description |
|----------|-------------|
| `WALLET_MNEMONIC` | BIP39 mnemonic (12 or 24 words) for deterministic wallet |
| `ADMIN_NPUBS` | Comma-separated list of admin npubs or hex pubkeys |

### Optional Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `CASHU_MINT_URL` | `https://testnut.cashu.space` | Primary Cashu mint URL |
| `TRUSTED_MINTS` | - | Comma-separated list of additional trusted mint URLs |
| `PAYOUT_LN_ADDRESS` | - | Lightning address for automatic payouts |
| `PAYOUT_THRESHOLD_SATS` | `1000` | Minimum balance to trigger automatic payout |
| `PAYOUT_INTERVAL_SECONDS` | `300` | Payout check interval (5 minutes) |
| `FRONTEND_URL` | - | Frontend URL for CORS |
| `ADMIN_FRONTEND_URL` | - | Admin panel URL for CORS |
| `HOST` | `0.0.0.0` | Server bind host |
| `PORT` | `8000` | Server bind port |

### Pricing Configuration

Edit `backend/data/models.json`:

```json
{
  "models": {
    "model-id": {
      "provider": "xai|local|openai",
      "display_name": "Human-readable name",
      "base_url": "https://api.example.com/v1",
      "api_key": "${API_KEY_ENV_VAR}",
      "pricing": {
        "input_usd_per_million": 0.30,
        "output_usd_per_million": 1.00
      },
      "modalities": ["text"],
      "max_context_tokens": 131072,
      "enabled": true
    }
  },
  "default_model": "model-id",
  "upfront_sats": {
    "default": 8,
    "chat": 8,
    "deep-research": 50
  }
}
```

**Tips:**
- Use a power of 2 for `upfront_sats` (8, 16, 32) to minimize Cashu swap fees
- Use `${ENV_VAR}` syntax to reference environment variables for sensitive values
- API keys are never exposed to frontend - only injected server-side into `RunnableConfig`

---

## CashuService

The `CashuService` class wraps the nutshell (cashu) library.

### Key Methods

| Method | Description |
|--------|-------------|
| `initialize()` | Initialize wallet with database and load keysets |
| `receive_token(token)` | Redeem token to wallet (swap proofs with mint) |
| `generate_token(amount, memo)` | Create a token from wallet balance |
| `sweep_all(memo)` | Generate token with all available funds |
| `payout_to_lightning(amount, ln_address)` | Send to Lightning via LNURL-pay |
| `get_spendable_balance()` | Get (spendable, inactive) balance tuple |
| `swap_inactive_proofs()` | Swap old keyset proofs to active keyset |

### Token Generation and Fees

When generating tokens (for refunds):

```python
send_proofs, fees = await self._wallet.select_to_send(
    self._wallet.proofs,
    amount,
    include_fees=False,  # User absorbs swap fees
)
```

With `include_fees=False`, if user requests 7 sats refund and mint charges 1 sat fee, they receive 6 sats. This is intentional for predictable costs.

---

## Error Handling

### Payment Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| `BackendUnavailableError` | Backend not reachable | Check backend is running |
| "Invalid payment session" | Bad/expired session ID | Payment flow error |
| "Session already refunded" | Double refund attempt | Already processed |
| "Refund exceeds original" | Amount > paid | Logic error in agent |
| "Token from untrusted mint" | Mint not trusted | Use trusted mint |

### Backend Unavailability

The system is designed to fail explicitly if the backend is unavailable:
- No hardcoded fallback prices
- No cached pricing that could become stale
- Clear error messages to users
- Agent stops processing (no free LLM calls)

---

## Running the Backend

### Development

```bash
cd backend
python -m venv venv
source venv/bin/activate
pip install -e .

# Set required environment variables
export WALLET_MNEMONIC="your twelve or twenty four word mnemonic phrase here"
export ADMIN_NPUBS="npub1youradminpubkey..."

# Run with auto-reload
python -m src.main
```

### Production

```bash
uvicorn src.main:app --host 0.0.0.0 --port 8000
```

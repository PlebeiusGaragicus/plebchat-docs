# PlebChat Agent Architecture

The PlebChat agent is a LangGraph-based AI assistant that validates Bitcoin ecash payments before processing requests. This document describes the agent architecture, LLM provider configuration, and payment validation flow.

## LLM Provider Configuration

The agent uses **model_id** from the backend's `models.json` to determine which LLM to use. The model_id maps to a provider automatically.

### Supported Providers

| Provider | Description | Model ID Pattern |
|----------|-------------|------------------|
| `xai` | xAI Grok API | `grok-*` |
| `local` | OpenAI-compatible endpoint | Configured via `LLM_MODEL` env |
| `openai` | OpenAI API | `gpt-*`, `o1-*` |

### Local Provider (OpenAI-compatible)

Uses any OpenAI-compatible API endpoint:

- **Ollama**: Local LLM inference (`http://localhost:11434/v1`)
- **LM Studio**: Local model hosting (`http://localhost:1234/v1`)
- **vLLM**: High-performance inference server

**Required environment variables:**

```bash
LLM_BASE_URL=http://localhost:11434/v1  # Your endpoint
LLM_MODEL=llama3.2                       # Your model name
LLM_API_KEY=not_set                      # Optional for local endpoints
```

### xAI Grok Provider

Uses xAI's Grok API.

**Required environment variables:**

```bash
XAI_API_KEY=your-xai-api-key            # Get from https://console.x.ai/
```

### Model Configuration

Models are configured in `backend/data/models.json` with API credentials:

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
- `modalities`: Array of supported modes (`text`, `vision`)
- `max_context_tokens`: Model's context window size
- `upfront_sats`: Per-agent upfront costs (power of 2 recommended)

The frontend fetches available models from `GET /api/pricing/models` and displays them in the settings drawer. **API keys are never exposed to frontend** - only injected server-side.

## Agent Graph Architecture

The agent follows a **pay-upfront, refund-difference** pattern with **backend-initiated runs**:

```
┌─────────────┐     ┌─────────────────────┐     ┌─────────────┐
│  Frontend   │────>│   Backend Proxy     │────>│  LangGraph  │
│  (message   │     │  POST /api/agent/   │     │   Cloud     │
│   + token)  │     │  start/{graph_id}   │     │             │
└─────────────┘     └─────────────────────┘     └──────┬──────┘
                              │                        │
                              │ 1. Validate payment    │
                              │ 2. Inject credentials  │
                              │ 3. client.runs.create()│
                              ▼                        │
                    ┌─────────────────────┐           │
                    │  Return run_id,     │<──────────┘
                    │  thread_id          │
                    └─────────────────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │  Frontend joins     │
                    │  runs.joinStream()  │────> Stream AI response
                    └─────────────────────┘
```

### Architecture: Backend Kickoff

Payment validation and LLM credential injection are handled by the backend proxy:

1. **Frontend sends payment + message to backend** (`POST /api/agent/start/{graph_id}`)
2. **Backend validates and redeems ecash token** before any LLM calls
3. **Backend injects model credentials** from `models.json` into `RunnableConfig`
4. **Backend starts the LangGraph run** via `client.runs.create()`
5. **Frontend joins the stream** via `client.runs.joinStream(thread_id, run_id)`

This architecture ensures:
- **API keys never reach frontend** - credentials injected server-side only
- **No free LLM calls** - payment validated before run starts
- **Streaming works** - frontend joins ongoing run for real-time response

### Graph Nodes (Simplified)

```
┌─────────────┐    ┌───────┐    ┌──────────┐    ┌─────┐
│   START     │───>│ agent │───>│ finalize │───>│ END │
└─────────────┘    └───────┘    └──────────┘    └─────┘
```

1. **agent**: Processes user message with the configured LLM
   - Credentials available via `config["configurable"]["model_credentials"]`
   - Uses dynamic model from config (selected by user in settings)
   - Extracts token usage (prompt + completion tokens)
   - Calculates cost via backend `/api/pricing/calculate`

2. **finalize**: Generates refund for unused sats
   - Calculates refund amount from usage cost
   - Calls `/api/wallet/refund` with `payment_session_id`
   - Returns `refund_token` to client for auto-redemption

### State Management

```python
class AgentState(TypedDict):
    messages: list[BaseMessage]      # Conversation history
    config: dict | None              # Agent config (model, temperature, etc.)
    payment_session_id: str | None   # Session ID for secure refunds (from backend)
    refund: bool                     # Whether refund was generated
    refund_token: str | None         # Token to return for refund
    error: str | None                # Error message if failed
    run_id: str | None               # Unique ID for logging
    usage_metadata: UsageCost | None # Token usage and costs
```

**Note:** Payment validation fields (`payment_validated`, `payment_redeemed`) are no longer in state since the backend validates payment before the graph runs.

## Payment Flow

The payment flow uses a **backend proxy** architecture:

```
1. Frontend calls POST /api/agent/start/{graph_id} with message + ecash token
2. Backend validates and redeems token:
   - Calls POST /api/wallet/receive
   - Creates payment_session_id
   - Loads model credentials from models.json
3. Backend starts LangGraph run:
   - Injects payment_session_id and credentials into RunnableConfig
   - Calls client.runs.create() with stream_mode and stream_resumable
   - Returns run_id and thread_id to frontend
4. Frontend joins stream:
   - Calls client.runs.joinStream(thread_id, run_id)
   - Displays AI response in real-time
5. Agent node executes:
   - Gets credentials from config["configurable"]["model_credentials"]
   - Creates LLM with model_id from config
   - Invokes LLM, extracts usage_metadata (tokens)
   - Calls POST /api/pricing/calculate to get cost
6. Finalize node:
   - If refund_sats > 0 and payment_session_id exists:
   - Calls POST /api/wallet/refund with session_id
   - Returns refund_token to client
7. Client auto-redeems refund token to wallet
```

### Backend Unavailability

If the backend is unavailable, the agent raises `BackendUnavailableError`:
- No fallback pricing
- No hardcoded defaults
- Explicit failure with clear error message

This prevents free LLM usage when pricing data is stale or unavailable.

## Dynamic Pricing

Pricing is configured centrally in the backend:

### Upfront Cost
- Set in `models.json` as `upfront_sats`
- Frontend fetches via `GET /api/pricing/models`
- User pays this amount for every prompt
- **Recommended**: Use power of 2 (8, 16, 32) to minimize Cashu swap fees

### Per-Token Cost
- Configured per model in USD per million tokens
- Backend converts to sats using current BTC price
- After LLM response, agent calls `POST /api/pricing/calculate`
- Returns actual cost and refund amount

### Cost Calculation

```
input_cost_sats = ceil(prompt_tokens × input_price_sats_per_million / 1_000_000)
output_cost_sats = ceil(completion_tokens × output_price_sats_per_million / 1_000_000)
total_cost_sats = input_cost_sats + output_cost_sats
refund_sats = upfront_sats - total_cost_sats
```

### Fallback Usage Estimation

If the LLM provider doesn't return token usage (some local models):
- Agent estimates from response length (~4 chars per token)
- Conservative prompt estimate (100 tokens)
- Ensures user still gets a refund

## Payment Session Security

To prevent refund exploits, the backend uses payment sessions:

1. When token is received, backend creates `PaymentSession`:
   - Stores original amount, timestamp
   - Returns `payment_session_id` to agent

2. When agent requests refund:
   - Must provide valid `payment_session_id`
   - Backend validates: session exists, not expired, not already refunded
   - Marks session as refunded (prevents double-refund)

3. Session expiry:
   - Sessions expire after 1 hour
   - Periodic cleanup removes expired sessions

## Configuration Reference

### Environment Variables

Environment variables are referenced in `models.json` using `${VAR}` syntax:

| Variable | Required | Description |
|----------|----------|-------------|
| `LLM_BASE_URL` | For local | OpenAI-compatible endpoint URL (e.g., `http://localhost:8000/v1`) |
| `LLM_API_KEY` | No | API key for local provider (some endpoints don't need it) |
| `XAI_API_KEY` | For xai | xAI API key from console.x.ai |
| `OPENAI_API_KEY` | For openai | OpenAI API key |
| `PRICING_URL` | No | Backend pricing URL (default: `http://localhost:8000/api/pricing`) |
| `WALLET_URL` | No | Backend wallet URL (default: `http://localhost:8000/api/wallet`) |

**Note:** API keys are stored in the backend's environment variables and referenced in `models.json`. They are injected into the graph's `RunnableConfig` at runtime by the backend proxy.

### Model Configuration (models.json)

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
      "modalities": ["text", "vision"],
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

## Development

Run the agent in development mode:

```bash
cd agents
source venv/bin/activate
langgraph dev
```

The agent will be available at `http://localhost:2024`.

## Logging

Agent runs are logged with detailed payment and usage information:

```
[Payment] Token redeemed successfully (8 sats, session=abc123)
[Agent] Invoking LLM (model=grok-4-1-fast-non-reasoning, temp=0.7)
[Agent] Usage: 150 input, 500 output tokens
[Agent] Cost: 1 sats, refund: 7 sats
[Finalize] Generating refund token for 7 sats...
[Finalize] SUCCESS: Refund token generated
```

Events logged:
- `run_start`: Initial request with payment info
- `payment_redeemed`: Successful token redemption with session ID
- `llm_response`: LLM output and token usage
- `run_end`: Completion status with refund info

# What is PlebChat?

⚠️ Coding agents must ALWAYS read [their instructions](../README.md).

---

## Core Product

**PlebChat** is a simple AI chat web application that demonstrates:

 - A Svelte 5 client-side SPA in `frontend/` (no server-side rendering) with a "plebtap" library used to handle user login, bitcoin ecash wallet and micropayments for pay-per-use AI usage. Builds to static files deployable on any static host.
 - A Python3.12 FastAPI `backend/` used to validate/redeem ecash tokens, calculate pricing, and generate refunds
 - LangGraph `agents/` are LLM graphs that verify ecash before execution and refund unused sats after

---

- **Nostr Protocol:**
    - **Users** are just simply public/private key pairs - no usernames/passwords. Usage of a NIP-07 browser extention elevates this experience even further.
    - **Just works** - Nostr protocol is used "under the hood" and should be largely invisible to the user. We aim for an experience that *"just works"* with only the "advanced user" suspecting usage of the nostr protocol.

- **Client-Side Storage:**
    - **Local Browser Storage:** All application data is stored in the user's browser, you own your data.
    - **Progressive web app** - Avoid "app store" ecosystem lock-in - we don't need approval to publish freedom tech apps.

- **eCash Payments:**
    - **Bitcoin payments:** *any human* can pay for this service without risk of regulations locking them out.
    - **Pay-per-use** AI features avoid subscriptions therefore the over-charging of users who don't maximize their monthly usage.
    - **Self-custody** prevents risk of centralized theft of user funds - private keys are stored on user's devices.
    - **Fair pricing** Users pay upfront and get refunded for unused tokens after each request.

## Pricing Model

PlebChat uses a **pay-upfront, refund-difference** model:

1. **User pays upfront** - A fixed amount (e.g., 8 sats) for each prompt
2. **Actual cost calculated** - Based on model and token usage after LLM response
3. **Refund generated** - Unused sats returned as ecash token
4. **Auto-redemption** - Client automatically redeems refund to user's wallet

### Dynamic Pricing

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

- **Frontend fetches pricing** from backend via `/api/pricing/models`
- **No hardcoded prices** in frontend - all pricing is backend-authoritative
- **API keys never reach frontend** - injected server-side via backend proxy
- **BTC price conversion** - USD prices converted to sats using current BTC rate
- **Power of 2 upfront** - Using 8 sats minimizes Cashu swap fees


## Features:

 * A top navbar shows "PlebChat" in the top left corner with a dropdown/popover to its right that allows the user to select an agent.  A "Help me choose" is available which shows an FAQ in the main content area with links to each suggested agent.
 * On right edge of the top navbar we place our plebtap component.  When logged out the main content area shows "welcome" to plebchat content with a beautiful explanation of the site.  Once logged in we select the latest agent selected in the dropdown - if site previously visited - and the "help me choose" option otherwise.
 * The main content area shows a collapsing sidepanel that shows preveious chat threads for the selected agent (if saved).
 * Once an agent is selected a beautiful and minimal interface allows them to enter a prompt. Faint mouse trails are enabled which appear to energize the digital/binary background behind them. The input box appears to glow and invites the user to ask, learn and engage.  A settings gear icon embedded in the prompt input box activates a modal which allows the user to adjust the configuration of the currently selected agent, with a "Use settings for this chat" and "Save agent configuration" options.
 * Similarly to the example repositories, the agent's tool calls displayed inline in the chat history.


 ## Agents

All agents use the same pricing model:
- **Upfront payment** configured in `models.json` (`upfront_sats`)
- **Per-token cost** calculated from model's USD pricing and current BTC rate
- **Automatic refund** of unused sats after each request

### PlebChat 💬

This graph is a single LLM call - no fluf - history is appended to enable long conversations.

> Dynamic pricing based on token usage. Users select model in settings.

**File upload:** *no file upload*

 ---

### Deep Research 🌎📚🔭

Comprehensive internet research agent with multi-agent architecture. See [Deep Research Documentation](features/deep-research.md) for details.

> Dynamic pricing based on token usage. Research uses more tokens = higher cost.

**Research Depths:**
- ⚡️ **Quick** (~1-2 min) - Surface-level research
- 🔍 **Thorough** (~3-5 min) - Balanced depth
- 🌊 **Deep** (~8-15 min) - Maximum comprehensiveness

**File upload:** *no file upload*

---

### Socratic Coach ☕️🧠💭

⚠️ DO NOT IMPLEMENT YET

> Dynamic pricing based on token usage.

**File upload:** *images only*

---

### TLDR Summarizer 📖🤨❓

⚠️ DO NOT IMPLEMENT YET

> Dynamic pricing based on token usage.

**File upload:** *PDF, epub, mobi*

---

### NSFW 🙉🙈🙊

This agent is able to answer questions and engage in behaviour that most companies try to "train out" of their agent.  No thread history is saved and all chats are lost once the user navigates away.

⚠️ DO NOT IMPLEMENT YET

> Dynamic pricing based on token usage.

**File upload:** *no file upload*

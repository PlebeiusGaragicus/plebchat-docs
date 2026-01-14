# User Interface

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Browser                                     │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                         +layout.svelte                             │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐   │  │
│  │  │     Navbar      │  │   MouseTrail    │  │  Toast Notifs   │   │  │
│  │  │  (AgentDropdown │  │   (glow effect) │  │  (svelte-sonner)│   │  │
│  │  │   + Cyphertap)  │  │                 │  │                 │   │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                    │                                     │
│                                    ▼                                     │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                          +page.svelte                              │  │
│  │                                                                    │  │
│  │   ┌──────────────────┐          ┌──────────────────────────────┐  │  │
│  │   │  ThreadSidebar   │          │  WelcomePage (if no agent)   │  │  │
│  │   │  - Thread list   │          │  - Feature grid              │  │  │
│  │   │  - New chat      │          │  - Agent selection           │  │  │
│  │   │  - Delete        │          └──────────────────────────────┘  │  │
│  │   └──────────────────┘                       OR                   │  │
│  │                                 ┌──────────────────────────────┐  │  │
│  │                                 │  ChatContainer (if agent)    │  │  │
│  │                                 │  ┌────────────────────────┐  │  │  │
│  │                                 │  │  ChatMessage (list)    │  │  │  │
│  │                                 │  │  - User/AI/Tool types  │  │  │  │
│  │                                 │  │  - ToolCallDisplay     │  │  │  │
│  │                                 │  └────────────────────────┘  │  │  │
│  │                                 │  ┌────────────────────────┐  │  │  │
│  │                                 │  │  ChatInput             │  │  │  │
│  │                                 │  │  - Cost display        │  │  │  │
│  │                                 │  │  - Send button         │  │  │  │
│  │                                 │  └────────────────────────┘  │  │  │
│  │                                 └──────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                    │                                     │
│                    ┌───────────────┴───────────────┐                    │
│                    ▼                               ▼                    │
│  ┌─────────────────────────────┐  ┌─────────────────────────────────┐  │
│  │       Svelte Stores         │  │         Cyphertap               │  │
│  │  ┌───────────────────────┐  │  │  - Nostr login                  │  │
│  │  │ agent.ts (selection)  │  │  │  - Ecash wallet                 │  │
│  │  │ threads.ts (history)  │  │  │  - Token generation             │  │
│  │  │ stream.svelte.ts      │  │  │  - Balance management           │  │
│  │  └───────────────────────┘  │  └─────────────────────────────────┘  │
│  └─────────────────────────────┘                                        │
│                    │                                                     │
└────────────────────┼─────────────────────────────────────────────────────┘
                     │
                     ▼
        ┌───────────────────────┐
        │   LangGraph Backend   │
        │   (streaming API)     │
        └───────────────────────┘
```

---

## Technology Stack

| Technology | Version | Purpose |
|------------|---------|---------|
| SvelteKit | 2.49.1 | Application framework |
| Svelte | 5.45.6 | Reactive UI with runes |
| Tailwind CSS | 4.0.0 | Utility-first styling |
| TypeScript | 5.9.3 | Type safety |
| LangGraph SDK | 0.0.43 | AI agent streaming |
| Cyphertap | local | Bitcoin/ecash wallet |
| Bits UI | 2.9.4 | Accessible components |
| Lucide Svelte | 0.515.0 | Icon library |
| Svelte Sonner | 1.0.5 | Toast notifications |

### Build Configuration

The frontend is a **pure client-side SPA** with no server-side rendering:

| Setting | Value | Description |
|---------|-------|-------------|
| Adapter | `@sveltejs/adapter-static` | Generates static files only |
| SSR | `false` | All rendering happens in browser |
| Prerender | `true` | Generates HTML shell at build time |
| Fallback | `index.html` | All routes serve the same shell |

**Build output:** Static files in `build/` directory, deployable to any static host (Nginx, S3, Cloudflare Pages, GitHub Pages, etc.).

---

## Application Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   User      │     │   Select    │     │   Send      │     │   Receive   │
│   Visits    │────▶│   Agent     │────▶│   Message   │────▶│   Response  │
│   Site      │     │             │     │   + Payment │     │   Stream    │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │                   │
       ▼                   ▼                   ▼                   ▼
  WelcomePage         ChatContainer      Cyphertap          ChatMessage
  (feature grid)      (empty state)      generates          (streaming
                                         ecash token        with cursor)
```

### User Journey

1. **Landing**: User sees WelcomePage with feature grid (Pay Per Use, Self Custody, No Accounts, etc.)
2. **Agent Selection**: User selects an AI agent from the grid or dropdown
3. **Chat Interface**: ChatContainer displays with input field showing prompt cost
4. **Payment**: On send, Cyphertap generates ecash token for the prompt cost
5. **Streaming**: LangGraph streams response chunks, displayed in real-time
6. **History**: Conversation saved to LocalStorage, accessible via ThreadSidebar

---

## Components

### Overview Table

| Component | Location | Purpose |
|-----------|----------|---------|
| WelcomePage | `lib/components/WelcomePage.svelte` | Landing page with features and agent grid |
| ChatContainer | `lib/components/chat/ChatContainer.svelte` | Main chat interface with messages and input |
| ChatInput | `lib/components/chat/ChatInput.svelte` | Auto-expanding textarea with cost display |
| ChatMessage | `lib/components/chat/ChatMessage.svelte` | Message bubble with avatar and tool calls |
| ChatError | `lib/components/chat/ChatError.svelte` | Inline error display with refund recovery |
| ToolCallDisplay | `lib/components/chat/ToolCallDisplay.svelte` | Expandable tool execution details |
| Navbar | `lib/components/nav/Navbar.svelte` | Top navigation with logo and wallet |
| AgentDropdown | `lib/components/nav/AgentDropdown.svelte` | Agent selection dropdown menu |
| ThreadSidebar | `lib/components/sidebar/ThreadSidebar.svelte` | Chat history sidebar with thread list |
| MouseTrail | `lib/components/effects/MouseTrail.svelte` | Animated mouse-following glow effect |

### Component Details

#### WelcomePage

The landing page shown when no agent is selected.

**Features:**
- Hero section with headline "AI Chat, Pay-Per-Use"
- 6-item feature grid:
  - Pay Per Use (Bitcoin micropayments)
  - Self Custody (ecash stored on device)
  - No Accounts (Nostr-based identity)
  - Permissionless (no restrictions)
  - Local Storage (no data collection)
  - Instant Payments (no confirmation wait)
- Agent selection grid with availability indicators
- Responsive layout (1-3 columns based on viewport)

#### ChatContainer

The main chat interface displayed after agent selection.

**Features:**
- Empty state with agent info when no messages
- Scrollable message history with intelligent scroll limits
- Bottom-fixed floating input area
- Error handling with toast notifications
- Refund token auto-redemption after each request
- Dynamic cost from backend (no hardcoded prices)
- **Message regeneration** with checkpoint-based branching
- **Fork navigation** between AI response versions
- **AI message deletion** with sibling switching

**Cost Logic:**
```
All prompts: upfront_sats from backend (e.g., 8 sats)
Actual cost: calculated from token usage after LLM response
Refund: upfront_sats - actual_cost (auto-redeemed to wallet)
```

**Pricing Flow:**
1. Frontend fetches `upfront_sats` from `GET /api/pricing/models`
2. User pays upfront amount for each prompt
3. Agent calculates actual cost from token usage
4. Unused sats refunded as ecash token
5. Client auto-redeems refund to user's wallet

**Scroll Limit Behavior:**

The chat area uses a dynamic spacer system to control scroll limits, optimized for the conversational flow:

| Scenario | Scroll Behavior |
|----------|-----------------|
| User sends message | Scrolls so human message is at the TOP of viewport |
| AI streaming | Spacer shrinks as AI response grows |
| AI reply fits in viewport | Scroll limit = human message at top |
| AI reply is very long | Scroll limit = bottom of AI reply visible |

This design ensures:
1. When the user sends a prompt, it scrolls to the top leaving maximum space for the AI's streaming response
2. You can never scroll past the last human message (no wasted empty space)
3. Long AI responses can still be fully scrolled into view

**How it works:**

```
┌─────────────────────────────────────┐
│ [20px padding]                      │
│ [Last Human Message] ← at top       │
│ [AI Reply]                          │
│ [Dynamic Spacer] ← fills remaining  │
│ [Bottom padding for input]          │
└─────────────────────────────────────┘

Spacer height = viewport - 20px - humanHeight - aiHeight - bottomPadding
```

The spacer automatically recalculates when:
- Window is resized
- Sidebar is expanded/collapsed
- AI streaming ends (final height known)
- Thread is switched

#### ChatInput

Floating message input with payment cost display.

**Features:**
- Floating oval design positioned over chat area
- Glassmorphism effect (backdrop blur, semi-transparent)
- Auto-expanding textarea (max 150px height)
- Smart Enter handling (Enter sends, Shift+Enter newline)
- Centered cost indicator pill (floats above input)
- Centered low balance warning pill (amber color when insufficient)
- Settings button (circular, opens agent configuration)
- Send button (circular with hover scale, glow effects)
- Disabled states for: no agent, insufficient balance, streaming

#### ChatMessage

Individual message display with type differentiation.

**Message Types:**

| Type | Avatar Color | Bubble Background | Label |
|------|--------------|-------------------|-------|
| human | Purple (`#7c3aed`) | Purple tint (`purple-dim/20`) | "You" |
| ai | Orange gradient | Elevated (`bg-elevated`) | "PlebChat" |
| tool | Gray | - | Tool name |

**Features:**
- Transparent row backgrounds (no visible boxes when mouse effect passes)
- Streaming cursor animation for empty AI messages
- Tool call expansion via ToolCallDisplay
- Copy button on hover
- Pre-wrap text formatting
- Responsive sizing
- **Regenerate button** (AI messages only) - creates new response branch
- **Delete button** (AI messages only) - removes response, switches to sibling
- **Fork navigation** (when multiple versions exist) - left/right arrows with "2/3" indicator

#### ToolCallDisplay

Expandable visualization of agent tool executions.

**Features:**
- Collapsible with chevron indicator
- Tool name with status icon (checkmark for complete)
- Arguments display (JSON formatted, monospace)
- Result display when available
- Syntax highlighting colors

---

## Message Branching & Regeneration

The chat interface supports regenerating AI responses and navigating between different response branches. This uses LangGraph's native checkpoint-based branching system.

### How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│  Human Message: "What is Bitcoin?"                              │
│  └── AI Response v1: "Bitcoin is a cryptocurrency..."          │
│      └── [Branch 1/3] ◄───────┐                                 │
│  └── AI Response v2: "Bitcoin is digital money..."             │
│      └── [Branch 2/3]          │ User navigates between        │
│  └── AI Response v3: "Bitcoin is a decentralized..."           │
│      └── [Branch 3/3] ─────────┘                                │
└─────────────────────────────────────────────────────────────────┘
```

### Features

| Feature | Description |
|---------|-------------|
| **Regenerate** | Creates a new AI response branch for the same human message |
| **Branch Navigation** | Left/right arrows to switch between response versions |
| **Delete Response** | Removes the visible AI response; switches to sibling if available |
| **Checkpoint Forking** | Uses LangGraph checkpoints for proper server-side branching |
| **Persistent Branches** | Branch structure survives page refresh |

### User Actions

| Action | Button | Effect |
|--------|--------|--------|
| Regenerate | 🔄 Regenerate | Creates new AI response sibling |
| Previous Branch | ◀ | Shows previous response version |
| Next Branch | ▶ | Shows next response version |
| Delete | 🗑️ Delete | Removes current AI response |

### Fork Navigation UI

When an AI message has multiple versions (siblings), a branch navigator appears:

```
┌─────────────────────────────────────────┐
│  [◀]  2/3  [▶]   🔄 Regenerate  🗑️ Delete │
└─────────────────────────────────────────┘
```

- **2/3**: Currently viewing version 2 of 3 total responses
- Navigation arrows are disabled at boundaries (1/3 or 3/3)

### Technical Implementation

**Checkpoint-Based Regeneration:**
1. When user clicks Regenerate, system fetches thread history from LangGraph
2. Finds the checkpoint after the human message was added (before AI response)
3. Uses `checkpointId` parameter in `client.runs.stream()` to fork from that point
4. LangGraph creates a new branch automatically

**Branch Metadata Fetching:**
```typescript
// Fetches branch info from LangGraph server
const branchInfo = await getBranchInfoForThread(langgraphThreadId);
// Returns: Map<messageId, { branch, branchOptions, messageId }>
```

**Branch Switching:**
```typescript
// Switches to a different branch by loading its state
await switchToBranch(localThreadId, targetCheckpointId);
```

### Delete Behavior

The delete button appears on **AI messages** (not human messages):

| Scenario | Delete Behavior |
|----------|-----------------|
| AI has siblings | Removes this AI response, switches to next sibling |
| AI is only child | Removes AI response AND parent human message |
| Thread becomes empty | Deletes the entire thread |

### Key Files

| File | Purpose |
|------|---------|
| `stream.svelte.ts` | `regenerateResponse()`, `getBranchInfoForThread()`, `switchToBranch()` |
| `threads.ts` | `addSiblingAiMessage()`, `deleteAiMessage()`, `replaceMessagesFromBranch()` |
| `ChatContainer.svelte` | `handleSwitchFork()`, `getForkInfoForAiMessage()`, branch state |
| `ChatMessage.svelte` | Fork navigation UI, delete/regenerate buttons |

### State Fields

```typescript
interface Thread {
  // ... other fields
  currentBranch?: string;      // Current branch checkpoint ID
  locallyModified?: boolean;   // Skip server sync when true
}

interface BranchInfo {
  branch: string;              // Current branch checkpoint ID
  branchOptions: string[];     // All available branch checkpoint IDs
  messageId: string;           // The AI message this relates to
}
```

---

#### ChatError

Inline error display with payment recovery options. Appears in chat history when a request fails.

**Features:**
- Red-styled alert box with warning icon
- Original error message display (e.g., "Payment redemption failed")
- Refund error display in amber (e.g., "Rate limit exceeded")
- "Retry Refund" button - attempts ecash token redemption
- "Copy Token" button - copies raw token for manual wallet recovery
- Instructions for manual recovery if retry fails
- Thread-specific: only shows when viewing the affected thread
- Persists across page refresh via thread's `pendingRefund` field

**Recovery Flow:**
1. Agent returns error + refund token
2. Auto-refund attempted immediately
3. If fails → token persisted to thread, ChatError displayed
4. User can retry or copy token for manual redemption
5. On thread reload → auto-retry attempted again

#### Navbar

Fixed top navigation bar.

**Features:**
- PlebChat logo with Zap icon (links to welcome)
- AgentDropdown for agent selection
- Testing mode badge (shows when active)
- Cyphertap wallet component
- Sticky positioning (z-index 50)

#### AgentDropdown

Dropdown menu for agent selection.

**Features:**
- "Help me choose" option (shows WelcomePage)
- Agent list with:
  - Emoji and name
  - Lock icon for unavailable agents
  - Cost display (initial + per prompt)
  - Checkmark for selected
- Click-outside to close
- Keyboard accessible

#### ThreadSidebar

Collapsible sidebar for chat history management.

**Features:**
- CSS-based responsive rendering (both mobile and desktop markup always rendered)
- Desktop (`md:` and up): Fixed sidebar panel with toggle button
- Mobile (below `md:`): Drawer overlay (vaul-svelte) with backdrop
- "New Chat" button
- Thread list with:
  - Title (first 50 chars of first message)
  - Relative timestamp (Today, Yesterday, Xd ago)
  - Delete button with confirmation
- Agent-specific filtering
- Auto-collapse on mobile after selection
- Empty states for no agent or no threads

#### MouseTrail

Visual effect that follows mouse movement.

**Features:**
- Radial gradient glow (orange to purple, matching theme)
- 500x500px blur effect with 50px filter
- Fast CSS transition (60ms) for responsive tracking
- Svelte 5 `$state` reactivity for position updates
- Opacity: 18% orange core, 12% purple fade

---

## State Management

All state is managed with Svelte stores and persisted to LocalStorage.

### Models Store

**File:** `lib/stores/agent.ts`

Manages available LLM models fetched from the backend.

**Key Features:**
- Fetches models from `GET /api/pricing/models` on initialization
- Caches models, BTC price, upfront cost, and default model
- Provides `enabledModels` derived store for UI
- Auto-refreshes when backend becomes available

**Key Exports:**
```typescript
// The models store instance
export const modelsStore = createModelsStore();

// Derived store of enabled models only
export const enabledModels = derived(modelsStore, $store => 
  $store.models.filter(m => m.enabled)
);

// Helper to get current upfront cost
export function getPromptCost(): number {
  return modelsStore.getUpfrontSats();
}
```

**ModelInfo Interface:**
```typescript
interface ModelInfo {
  model_id: string;                    // e.g., "grok-4-1-fast-non-reasoning"
  provider: string;                     // "xai", "local", "openai"
  display_name: string;                 // Human-readable name
  input_price_sats_per_million: number; // Input token price
  output_price_sats_per_million: number;// Output token price  
  enabled: boolean;                     // Available for selection
}
```

### Agent Store

**File:** `lib/stores/agent.ts`

Manages the currently selected AI agent.

**Key Features:**
- LocalStorage persistence (`plebchat-selected-agent`)
- Predefined agent configurations
- Agent selection with auto-save
- "Clear" action to return to welcome

**Agent Interface:**
```typescript
interface Agent {
  id: string;
  name: string;
  emoji: string;
  description: string;
  available: boolean;
  cost: number;              // static fallback cost (use getPromptCost() for dynamic)
  capabilities: string[];
  fileUpload: 'none' | 'images' | 'pdf' | 'all';
  historyEnabled: boolean;
  config: AgentConfig;       // model, temperature, systemPrompt
}
```

**Dynamic Cost:**
```typescript
// Get the current upfront cost from backend
const cost = getPromptCost(); // Returns modelsStore.getUpfrontSats()
```

### Threads Store

**File:** `lib/stores/threads.ts`

Manages chat conversation history with a tree-based message structure for branching support.

**Key Features:**
- LocalStorage persistence (`plebchat-threads`)
- Thread creation with first message as title
- **Tree-based message storage** for branching/forking
- Message streaming support (partial updates)
- Tool call tracking
- LangGraph thread ID mapping
- Thread deletion with fallback
- **Per-thread refund token persistence** for payment recovery
- **Branch tracking** via `currentBranch` and `forkSelections`

**Key Types:**
```typescript
interface MessageNode {
  id: string;
  type: 'human' | 'ai' | 'tool';
  content: string;
  timestamp: number;
  toolCalls?: ToolCall[];
  parentId: string | null;   // Parent message ID (null for root)
  childIds: string[];        // Child message IDs (ordered by creation)
}

interface PendingRefund {
  token: string;        // Ecash refund token
  error: string;        // Original error that caused refund
  refundError?: string; // Error from failed auto-refund attempt
  timestamp: number;
}

interface Thread {
  id: string;
  agentId: string;
  title: string;
  createdAt: number;
  updatedAt: number;
  messages: Map<string, MessageNode>;  // All messages by ID (tree structure)
  rootMessageId: string | null;        // First message ID
  forkSelections: Map<string, string>; // humanMessageId -> selected AI child ID
  promptCount: number;
  langgraphThreadId?: string;
  pendingRefund?: PendingRefund;       // Persisted refund token for recovery
  locallyModified?: boolean;           // Skip server sync when true
  currentBranch?: string;              // Current branch checkpoint ID
}

interface ForkInfo {
  siblingCount: number;     // Total number of AI response siblings
  currentIndex: number;     // Currently selected sibling index (0-based)
  siblingIds: string[];     // All sibling AI message IDs
  humanMessageId: string;   // The human message this fork belongs to
}
```

**Tree Structure:** Messages are stored in a `Map<string, MessageNode>` where each node has `parentId` and `childIds`. This enables:
- Multiple AI responses to the same human message (branching)
- Navigation between response versions
- Proper delete behavior (delete AI sibling vs. entire message pair)

**Refund Recovery:** Failed payment tokens are stored per-thread and survive page refresh. When a thread with a pending refund is loaded, auto-redemption is attempted. The `ChatError` component displays inline with retry/copy options if recovery is needed.

### Stream Store

**File:** `lib/stores/stream.svelte.ts`

Manages LangGraph API communication, streaming, and branching.

**Key Features:**
- Svelte 5 runes for reactivity
- Debug mode flag (VITE_DEBUG=1 shows testnet badge)
- Loading/streaming state
- Error capture and display
- Transient refund token handling (persisted tokens move to threads store)
- **Checkpoint-based regeneration** for proper branching
- **Branch metadata fetching** from LangGraph history
- **Branch switching** to navigate between response versions
- LangGraph integration:
  - Dynamic thread creation
  - Dual stream modes ('messages' + 'values')
  - AI message and tool call handling
  - Checkpoint forking for regeneration

**Key Functions:**

| Function | Purpose |
|----------|---------|
| `sendMessage()` | Send a new message with payment |
| `regenerateResponse()` | Create new AI response branch using checkpoint forking |
| `editAndResendMessage()` | Edit human message and get new response |
| `deleteAiMessage()` | Delete specific AI response (with sibling switching) |
| `getBranchInfoForThread()` | Fetch branch metadata from LangGraph history |
| `switchToBranch()` | Switch to a different branch by loading its state |
| `syncThreadFromServer()` | Sync messages from server (respects `locallyModified`) |

**Regeneration with Checkpoints:**
```typescript
// 1. Find checkpoint after human message
const parentCheckpoint = await getParentCheckpointForRegen(threadId, humanMsgId);

// 2. Fork from that checkpoint
client.runs.stream(threadId, graphId, {
  input: { payment, config },
  checkpointId: parentCheckpoint.checkpoint_id,  // Fork point
  streamMode: ['messages', 'values']
});
```

**Anti-blink sync strategy:** Server sync is skipped immediately after streaming ends to prevent UI flash. The streamed data is authoritative; sync only runs when switching threads. Additionally, threads marked as `locallyModified` skip server sync to preserve local branch structure.

**Configuration:**
```typescript
const LANGGRAPH_API_URL = import.meta.env.VITE_LANGGRAPH_API_URL
  || 'http://localhost:2024';
const ASSISTANT_ID = import.meta.env.VITE_ASSISTANT_ID
  || 'plebchat';
```

---

## AI Agents

Five agents are configured, with PlebChat and Deep Research currently active.

| Agent | Emoji | Status | Pricing | Features |
|-------|-------|--------|---------|----------|
| PlebChat | 💬 | **Active** | Dynamic (per-token) | Simple chat, history enabled |
| Deep Research | 🌎📚🔭 | **Active** | Dynamic (per-token) | Multi-agent research |
| Socratic Coach | ☕️🧠💭 | Coming Soon | Dynamic (per-token) | Guided learning, image upload |
| TLDR Summarizer | 📖🤨❓ | Coming Soon | Dynamic (per-token) | Document summarization |
| NSFW | 🙉🙈🙊 | Coming Soon | Dynamic (per-token) | Unrestricted, no history |

**Note:** All agents use the same dynamic pricing model. Users pay `upfront_sats` (configured in backend `models.json`) and get refunded unused sats based on actual token usage.

### Agent Capabilities

Each agent defines its capabilities and file upload support:

| Agent | Capabilities | File Upload |
|-------|-------------|-------------|
| PlebChat | chat | none |
| Deep Research | research, analysis | none |
| Socratic Coach | teaching, discussion | images |
| TLDR Summarizer | summarization | pdf |
| NSFW | unrestricted | none |

---

## Payment Integration

### Cyphertap Wallet

The frontend integrates with Cyphertap for Bitcoin/ecash payments.

**Features:**
- Nostr-based login (no accounts)
- Ecash token storage in browser
- Balance display in navbar
- Token generation for prompts
- Token redemption for refunds

### Payment Flow

The frontend uses a **backend proxy** architecture:

```
1. Frontend fetches upfront_sats from backend on load
2. User enters message
3. User clicks Send
4. ChatContainer calls cyphertap.generateEcashToken(upfront_sats)
5. Frontend calls POST /api/agent/start/{graph_id} with message + token
6. Backend validates payment, starts LangGraph run
7. Backend returns run_id, thread_id to frontend
8. Frontend joins stream via client.runs.joinStream(thread_id, run_id)
9. AI response streams to frontend in real-time
10. Agent calculates actual cost from token usage
11. Agent generates refund token for unused sats
12. Refund token returned to client via stream
13. Client auto-redeems refund to user's wallet
```

**Key Points:**
- All pricing comes from backend `/api/pricing/models`
- No hardcoded prices in frontend
- **Payment validated before graph runs** - no free LLM calls
- **API keys never reach frontend** - injected server-side only
- User always pays upfront, gets refunded the difference
- Refunds are automatic and invisible to user

### Debug Mode

When `DEBUG_MODE` is enabled (via `VITE_DEBUG=1` in `.env.development`):
- Debug badge shown in navbar to indicate testnet environment
- Useful for development with FakeWallet mint
- Dynamic pricing still applies

---

## Design System

### Color Palette

| Variable | Value | Usage |
|----------|-------|-------|
| `--color-bg-primary` | `#030305` | Main background (ultra-dark) |
| `--color-bg-secondary` | `#08080c` | Secondary surfaces |
| `--color-bg-tertiary` | `#0e0e14` | Tertiary surfaces |
| `--color-bg-elevated` | `#16161e` | Cards, modals |
| `--color-accent-primary` | `#f97316` | Primary accent (orange) |
| `--color-accent-dim` | `#ea580c` | Dimmed accent |
| `--color-purple-accent` | `#a855f7` | Secondary accent (purple) |
| `--color-purple-dim` | `#7c3aed` | Dimmed purple |
| `--color-green-success` | `#22c55e` | Success states |
| `--color-amber-warning` | `#fbbf24` | Warning states |
| `--color-red-error` | `#ef4444` | Error states |
| `--color-text-primary` | `#ffffff` | Main text |
| `--color-text-secondary` | `#b4b4bc` | Secondary text |
| `--color-text-muted` | `#6b6b78` | Muted text |
| `--color-border-default` | `#2a2a38` | Default borders |
| `--color-border-hover` | `#3d3d50` | Hover borders |
| `--color-border-bright` | `#4a4a60` | Bright borders |

### Custom CSS Features

| Class/Animation | Purpose |
|-----------------|---------|
| `.bg-digital` | Grid pattern background (purple/orange subtle lines) |
| `.glow-input` | Animated border glow on focus |
| `@keyframes glow-pulse` | 3s pulsing glow |
| `@keyframes border-flow` | Flowing gradient animation |
| `.animate-fade-in` | Entrance fade animation |
| `.btn-primary` | Gradient orange button |
| `.btn-ghost` | Transparent ghost button |

### Typography

Fonts are self-hosted for privacy, performance, and consistency (no Google Fonts):

| Font | Usage | Location |
|------|-------|----------|
| Inter | Primary sans-serif | `static/fonts/inter/` |
| JetBrains Mono | Code and monospace | `static/fonts/jetbrains-mono/` |

**Why self-hosted?**
- **Privacy**: No external font requests that could track users
- **Performance**: Fonts load from same origin, no DNS lookups
- **Consistency**: Guaranteed availability, no CDN failures

### Responsive Breakpoints

| Breakpoint | Width | Usage |
|------------|-------|-------|
| sm | 640px | Mobile landscape |
| md | 768px | Tablet |
| lg | 1024px | Desktop |

---

## Features Not Yet Implemented

### Coming Soon Agents
- **Socratic Coach**: Interactive guided learning
- **TLDR Summarizer**: PDF and document summarization
- **NSFW**: Unrestricted conversation mode

### Planned Features
- **File Upload**: Image and PDF attachment support (UI exists, backend pending)
- **Agent Settings Modal**: Per-session agent configuration
- **Message Editing**: Edit sent human messages (regeneration of AI responses is implemented)
- **Message Reactions**: Thumbs up/down for feedback
- **Export Chat**: Download conversation history

### Recently Implemented
- ✅ **AI Response Regeneration**: Create new AI response branches with LangGraph checkpoint forking
- ✅ **Branch Navigation**: Navigate between different AI response versions (1/2, 2/2, etc.)
- ✅ **AI Message Deletion**: Delete specific AI responses with automatic sibling switching


---

## File Structure

```
frontend/src/
├── routes/
│   ├── +layout.svelte      # Global layout (navbar, effects, toasts)
│   ├── +layout.ts          # SPA config: ssr=false, prerender=true
│   └── +page.svelte        # Main page (welcome or chat)
├── lib/
│   ├── components/
│   │   ├── WelcomePage.svelte
│   │   ├── chat/
│   │   │   ├── ChatContainer.svelte
│   │   │   ├── ChatInput.svelte
│   │   │   ├── ChatMessage.svelte
│   │   │   └── ToolCallDisplay.svelte
│   │   ├── nav/
│   │   │   ├── Navbar.svelte
│   │   │   └── AgentDropdown.svelte
│   │   ├── sidebar/
│   │   │   └── ThreadSidebar.svelte
│   │   └── effects/
│   │       └── MouseTrail.svelte
│   ├── stores/
│   │   ├── agent.ts
│   │   ├── threads.ts
│   │   ├── sidebar.svelte.ts
│   │   └── stream.svelte.ts
│   ├── assets/
│   │   └── favicon.svg
│   └── utils.ts            # cn() class merge utility
├── app.css                 # Global styles + design system
├── app.d.ts               # Type definitions
└── app.html               # HTML template
```

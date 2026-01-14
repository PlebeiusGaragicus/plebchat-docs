# Deep Research Agent 🌎📚🔭

The Deep Research agent conducts comprehensive internet research using a multi-agent architecture. It breaks down complex research questions into parallel tasks, gathers information from multiple sources, and synthesizes findings into detailed reports.

> **Pricing:** Dynamic per-token pricing. User pays `upfront_sats` and gets refunded unused sats. Research uses more tokens → higher cost.

---

## Overview

Deep Research is inspired by [LangChain's Open Deep Research](https://github.com/langchain-ai/open_deep_research) and adapted for PlebChat with:

- **Bitcoin micropayments** via ecash tokens (payment validated before research)
- **Configurable LLM providers** (Local/OpenAI-compatible or Grok)
- **User-friendly depth controls** (Quick, Thorough, Deep)
- **Optional clarification mode** for complex queries

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              DEEP RESEARCH AGENT                                         │
│                                                                                          │
│  ┌─────────┐    ┌────────────────┐    ┌─────────────────┐    ┌─────────────────────┐   │
│  │  START  │───>│validate_payment│───>│clarify_with_user│───>│write_research_brief │   │
│  └─────────┘    └────────────────┘    └─────────────────┘    └─────────────────────┘   │
│                         │                     │                         │               │
│                         │ (failed)            │ (needs_clarification)   │               │
│                         └──────>──END         └──────>──END             │               │
│                                                                         ▼               │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │                         SUPERVISOR SUBGRAPH                                      │   │
│  │  ┌────────────┐                                      ┌─────────────────┐        │   │
│  │  │ supervisor │◄────────────────────────────────────>│ supervisor_tools │        │   │
│  │  │ (planning) │         (iterates until done)        │ (delegation)     │        │   │
│  │  └────────────┘                                      └─────────────────┘        │   │
│  │                                                              │                   │   │
│  │                              ┌───────────────────────────────┼───────┐           │   │
│  │                              │  PARALLEL RESEARCHERS (1-5)   │       │           │   │
│  │                              │  ┌──────────┐  ┌──────────┐  ...     │           │   │
│  │                              │  │researcher│  │researcher│          │           │   │
│  │                              │  │    ↕     │  │    ↕     │          │           │   │
│  │                              │  │  tools   │  │  tools   │          │           │   │
│  │                              │  │    ↓     │  │    ↓     │          │           │   │
│  │                              │  │ compress │  │ compress │          │           │   │
│  │                              │  └──────────┘  └──────────┘          │           │   │
│  │                              └───────────────────────────────────────┘           │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                               │                                         │
│                                               ▼                                         │
│                              ┌─────────────────────────────────┐                        │
│                              │    final_report_generation      │───>──END               │
│                              │    (synthesizes all findings)   │                        │
│                              └─────────────────────────────────┘                        │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Research Workflow

### 1. Payment Validation

Before any research begins, the ecash token is validated and redeemed:

```python
validate_payment_node(state, config)
# - Validates token format (cashuA/cashuB prefix)
# - Redeems token to wallet via backend
# - Stores payment_session_id for secure refund tracking
# - Proceeds only after successful redemption
```

### 2. Clarification (Optional)

If enabled, the agent analyzes the user's request to determine if clarification is needed:

- **Enabled:** Asks follow-up questions for vague or broad queries
- **Disabled:** Proceeds directly to research

```python
clarify_with_user(state, config)
# Returns either:
# - AIMessage with clarifying question → END (awaits response)
# - Proceeds to write_research_brief
```

### 3. Research Brief Generation

Transforms the user's request into a structured research brief that guides the supervisor:

```python
write_research_brief(state, config)
# - Extracts key research questions
# - Creates initial research plan
# - Initializes supervisor with system prompt
```

### 4. Supervisor (Research Orchestration)

The lead researcher that plans and delegates research tasks:

**Tools Available:**
- `ConductResearch` - Delegate a specific research topic to a researcher
- `ResearchComplete` - Signal that research is complete
- `think_tool` - Record reflections and planning thoughts

**Exit Conditions:**
- `ResearchComplete` called
- Maximum iterations reached (configurable depth)
- No more tool calls

### 5. Parallel Researchers

Each research task runs in a separate researcher subgraph:

**Researcher Tools:**
- `tavily_search` - Web search via Tavily API
- `think_tool` - Note-taking and reflection

**Researcher Workflow:**
1. Receive research topic from supervisor
2. Execute searches and gather information
3. Compress findings into concise summary
4. Return compressed research to supervisor

**Parallel Execution:**
- Up to 5 researchers run concurrently (fixed for rate limit protection)
- Results aggregated by supervisor

### 6. Final Report Generation

Synthesizes all research findings into a comprehensive report:

```python
final_report_generation(state, config)
# - Combines all compressed research
# - Generates cohesive final report
# - Streams response to user
```

---

## User Settings

### Research Depth

Three-button toggle for controlling research thoroughness:

| Mode | Iterations | Est. Time | Description |
|------|------------|-----------|-------------|
| **⚡️ Quick** | 2 | ~1-2 min | Surface-level research for simple queries |
| **🔍 Thorough** | 5 | ~3-5 min | Balanced depth for most research needs |
| **🌊 Deep** | 10 | ~8-15 min | Maximum depth for complex topics |

**Iterations** = Number of times the supervisor reflects and delegates new research tasks.

### Model Provider

| Provider | Model | Description |
|----------|-------|-------------|
| **Local** | OpenAI-compatible | Ollama, LM Studio, vLLM, etc. |
| **Grok** | grok-4-fast-non-thinking | xAI's fast model via API |

### Allow Clarification

Toggle for clarification behavior:

- **On (default):** Agent may ask clarifying questions before starting research
- **Off:** Agent proceeds directly to research without clarification

---

## Fixed Configuration

These settings are optimized internally and not exposed to users:

| Setting | Value | Reason |
|---------|-------|--------|
| Temperature | 0.7 | Balanced creativity and coherence for research |
| Concurrent Researchers | 5 | Optimal parallelism without hitting rate limits |

---

## State Management

### AgentState

Main state for the deep research workflow:

```python
class AgentState(MessagesState):
    # Payment fields
    payment: Optional[PaymentInfo]
    config: Optional[ConfigInfo]
    payment_validated: bool
    payment_redeemed: bool
    refund: bool
    refund_token: Optional[str]
    error: Optional[str]
    run_id: Optional[str]
    
    # Research fields
    supervisor_messages: list[Message]
    research_brief: Optional[str]
    raw_notes: list[str]
    notes: list[str]
    final_report: str
```

### SupervisorState

State for the lead researcher orchestrating research:

```python
class SupervisorState(TypedDict):
    supervisor_messages: list[Message]
    research_brief: str
    notes: list[str]
    research_iterations: int
    raw_notes: list[str]
```

### ResearcherState

State for individual researchers:

```python
class ResearcherState(TypedDict):
    researcher_messages: list[Message]
    tool_call_iterations: int
    research_topic: str
    compressed_research: str
    raw_notes: list[str]
```

---

## Configuration Reference

### Frontend Configuration

```typescript
interface ResearchConfig {
  model: 'local' | 'grok';            // LLM provider
  searchDepthMode: 'quick' | 'thorough' | 'deep';
  allowClarification: boolean;
}
```

### Backend Configuration (from state.config)

```python
class ConfigInfo(TypedDict):
    model: str                          # 'local' or 'grok'
    temperature: float                  # Fixed at 0.7
    max_researcher_iterations: int      # 2, 5, or 10 based on depth
    max_concurrent_research_units: int  # Fixed at 5
    allow_clarification: bool
```

---

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `TAVILY_API_KEY` | Yes | API key from [tavily.com](https://tavily.com) |
| `LLM_BASE_URL` | If Local | OpenAI-compatible endpoint URL |
| `LLM_MODEL` | If Local | Model name (e.g., `llama3.2`) |
| `XAI_API_KEY` | If Grok | xAI API key from console.x.ai |
| `WALLET_URL` | No | Backend wallet URL (default: `http://localhost:8000/api/wallet`) |

---

## Example Research Session

**User prompt:**
> "What are the latest developments in Bitcoin mining hardware for 2026?"

**Agent workflow:**

1. ✅ **Payment validated** (upfront sats redeemed, session created)
2. 🤔 **Clarification skipped** (clear research scope)
3. 📋 **Research brief generated:**
   - "Latest Bitcoin mining ASIC developments 2026"
   - "Efficiency improvements and manufacturers"
4. 🔍 **Supervisor delegates:**
   - Researcher 1: "New ASIC chips announced in 2026"
   - Researcher 2: "Mining efficiency benchmarks 2026"
   - Researcher 3: "Major mining hardware manufacturers"
5. 🔬 **Researchers execute:**
   - Tavily searches for each topic
   - Results compressed to key findings
6. 📊 **Final report generated:**
   - Synthesized overview of 2026 mining hardware
   - Key manufacturers and models
   - Efficiency comparisons and trends

---

## Error Handling

### Structured Output Failures

Local LLMs may not support structured output reliably. The agent handles this with fallbacks:

- **Clarification:** Falls back to proceeding directly to research
- **Research Brief:** Uses original user message as the brief

### Research Timeouts

Tavily searches use built-in content snippets rather than full-page scraping to avoid timeouts.

### Compression Failures

If research compression fails, the agent retries up to 3 times with truncated message history.

---

## File Structure

```
agents/src/research/
├── __init__.py
├── configuration.py      # ResearchConfiguration schema
├── deep_researcher.py    # Main graph with all nodes and subgraphs
├── prompts.py           # All prompt templates
├── state.py             # State definitions and reducers
└── utils.py             # Tools (Tavily search) and utilities
```

---

## Development

Run the agent in development mode:

```bash
cd agents
source venv/bin/activate
export TAVILY_API_KEY=tvly-xxxxxxxxx
langgraph dev
```

The agent will be available at `http://localhost:2024` with graph ID `deep_research`.

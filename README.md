# Neverending LLM Council

![llmcouncil](header.jpg)

**A fork of [karpathy/llm-council](https://github.com/karpathy/llm-council) with multi-turn conversation support and intelligent context management.**

The idea of this repo is that instead of asking a question to your favorite LLM provider (e.g. OpenAI GPT 5.1, Google Gemini 3.0 Pro, Anthropic Claude Sonnet 4.5, xAI Grok 4, etc.), you can group them into your "LLM Council". This is a simple, local web app that essentially looks like ChatGPT except it uses OpenRouter to send your query to multiple LLMs, it then asks them to review and rank each other's work, and finally a Chairman LLM produces the final response.

In a bit more detail, here is what happens when you submit a query:

1. **Stage 1: First opinions**. The user query (with conversation history) is given to all LLMs individually, and the responses are collected. The individual responses are shown in a "tab view", so that the user can inspect them all one by one.
2. **Stage 2: Review**. Each individual LLM is given the responses of the other LLMs (with conversation context). Under the hood, the LLM identities are anonymized so that the LLM can't play favorites when judging their outputs. The LLM is asked to rank them in accuracy and insight.
3. **Stage 3: Final response**. The designated Chairman of the LLM Council takes all of the model's responses and compiles them into a single final answer that is presented to the user.

## What's New in Neverending LLM Council

This fork extends the original concept with **true multi-turn conversation support**:

### 🔄 Multi-Turn Conversations
- **Full conversation context**: All council members see the entire conversation history
- **Natural follow-ups**: Ask clarifying questions, request elaborations, or explore topics deeper
- **Backward compatible**: Works seamlessly with existing conversations

### 🧠 Intelligent Context Management
- **Auto-compacting**: Long conversations (>10 messages) automatically summarize old history
- **Smart summarization**: Uses gemini-2.5-flash to create concise 2-3 sentence summaries of older context
- **Recency bias**: Recent messages (last 5 turns) always kept in full detail
- **Token-aware**: Prevents context overflow while maintaining conversation coherence

### 🛡️ Robust Error Handling
- **Extended timeouts**: Chairman gets 180s for complex synthesis (vs 120s default)
- **Better logging**: Detailed error messages for debugging (timeouts, HTTP errors, etc.)
- **Graceful fallbacks**: Returns top-ranked response if chairman synthesis fails
- **No more generic errors**: Informative feedback when things go wrong

### 🎨 UI Improvements
- **Always-visible input**: Text box no longer disappears when viewing old conversations
- **Progressive streaming**: Watch stages complete in real-time
- **Enhanced transparency**: See exactly how the council reached its conclusion

## Original Vibe Code Alert

The original project was 99% vibe coded as a fun Saturday hack by [@karpathy](https://github.com/karpathy) to explore and evaluate LLMs side by side in the process of [reading books together with LLMs](https://x.com/karpathy/status/1990577951671509438). This fork maintains that spirit while extending it for extended multi-turn conversations.

## Setup

### 1. Install Dependencies

The project uses [uv](https://docs.astral.sh/uv/) for project management.

**Backend:**
```bash
uv sync
```

**Frontend:**
```bash
cd frontend
npm install
cd ..
```

### 2. Configure API Key

Create a `.env` file in the project root:

```bash
OPENROUTER_API_KEY=sk-or-v1-...
```

Get your API key at [openrouter.ai](https://openrouter.ai/). Make sure to purchase the credits you need, or sign up for automatic top up.

### 3. Configure Models (Optional)

Edit `backend/config.py` to customize the council:

```python
COUNCIL_MODELS = [
    "openai/gpt-5.1",
    "google/gemini-3-pro-preview",
    "anthropic/claude-opus-4.5",
    "deepseek/deepseek-v3.2",
]

CHAIRMAN_MODEL = "google/gemini-3-pro-preview"

# Maximum conversation turns to include in Stage 3 chairman prompt
# Full history is always included in Stage 1 and Stage 2
MAX_HISTORY_FOR_CHAIRMAN = 3  # number of turns (user+assistant pairs)
```

**Note**: `MAX_HISTORY_FOR_CHAIRMAN` controls how much conversation history the chairman sees. The council members in Stage 1 and Stage 2 always get more context (auto-compacted if needed).

## Running the Application

**Option 1: Use the start script**
```bash
./start.sh
```

**Option 2: Run manually**

Terminal 1 (Backend):
```bash
uv run python -m backend.main
```

Terminal 2 (Frontend):
```bash
cd frontend
npm run dev
```

Then open http://localhost:5173 in your browser.

## Tech Stack

- **Backend:** FastAPI (Python 3.10+), async httpx, OpenRouter API
- **Frontend:** React + Vite, react-markdown for rendering
- **Storage:** JSON files in `data/conversations/`
- **Package Management:** uv for Python, npm for JavaScript

## How Multi-Turn Conversations Work

### Context Flow
1. **User sends message** → System loads conversation history from storage
2. **Stage 1**: All council models receive:
   - Full conversation history (auto-compacted if >10 messages)
   - Only final synthesized responses (stage3) from previous turns are included
   - This keeps context focused while preserving collaborative benefits
3. **Stage 2**: Council members review responses with same conversation context
4. **Stage 3**: Chairman synthesizes with limited recent history (last 3 turns by default)
   - Stage 1 responses already contain full context from council members
   - Limiting chairman's history prevents token overflow in long conversations

### Auto-Compacting Strategy
When conversations exceed 10 messages:
- **Old messages**: Summarized into 2-3 sentences using gemini-2.5-flash
- **Recent messages**: Last 10 messages (5 turns) kept in full detail
- **Summary position**: Injected as system message at conversation start
- **Fallback**: If summarization fails, truncates to recent messages only

### Error Resilience
- Individual model failures in Stage 1/2 → Graceful degradation (continues with successful responses)
- Chairman synthesis failure → Returns top-ranked Stage 1 response as fallback
- Timeout handling → Extended 180s timeout for chairman, 120s for others
- Detailed logging → All errors logged with specific failure types (timeout, HTTP error, etc.)

## Technical Details

See [CLAUDE.md](CLAUDE.md) for comprehensive technical documentation including:
- Detailed architecture breakdown
- Design decisions and rationale
- Data flow diagrams
- Implementation notes for future development

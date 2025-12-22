# CLAUDE.md - Technical Notes for LLM Council

This file contains technical details, architectural decisions, and important implementation notes for future development sessions.

## Project Overview

LLM Council is a 3-stage deliberation system where multiple LLMs collaboratively answer user questions. The key innovation is anonymized peer review in Stage 2, preventing models from playing favorites. The system supports multi-turn conversations, maintaining context across multiple exchanges.

## Architecture

### Backend Structure (`backend/`)

**`config.py`**
- Contains `COUNCIL_MODELS` (list of OpenRouter model identifiers)
- Contains `CHAIRMAN_MODEL` (model that synthesizes final answer)
- Uses environment variable `OPENROUTER_API_KEY` from `.env`
- Backend runs on **port 8001** (NOT 8000 - user had another app on 8000)

**`openrouter.py`**
- `query_model()`: Single async model query
- `query_models_parallel()`: Parallel queries using `asyncio.gather()`
- Returns dict with 'content' and optional 'reasoning_details'
- Graceful degradation: returns None on failure, continues with successful responses

**`council.py`** - The Core Logic
- `compact_conversation_history()`: Summarizes old messages to prevent token limit issues in long conversations
  - Keeps recent N messages in full detail
  - Summarizes older messages into a brief context using gemini-2.5-flash
  - Returns compacted message list for efficient processing
- `stage1_collect_responses()`: Parallel queries to all council models with conversation history
  - Auto-compacts history if >10 messages
- `stage2_collect_rankings()`:
  - Anonymizes responses as "Response A, B, C, etc."
  - Creates `label_to_model` mapping for de-anonymization
  - Prompts models to evaluate and rank (with strict format requirements)
  - Includes conversation context in ranking prompts (auto-compacted if long)
  - Returns tuple: (rankings_list, label_to_model_dict)
  - Each ranking includes both raw text and `parsed_ranking` list
- `stage3_synthesize_final()`: Chairman synthesizes from all responses + rankings with conversation context
  - Limits conversation history to last N turns (configurable via MAX_HISTORY_FOR_CHAIRMAN)
  - Uses extended timeout (180s) for complex synthesis
  - Provides intelligent fallback (top-ranked response) if synthesis fails
- `parse_ranking_from_text()`: Extracts "FINAL RANKING:" section, handles both numbered lists and plain format
- `calculate_aggregate_rankings()`: Computes average rank position across all peer evaluations
- `run_full_council()`: Main orchestrator that accepts message history and runs all 3 stages

**`storage.py`**
- JSON-based conversation storage in `data/conversations/`
- Each conversation: `{id, created_at, messages[]}`
- Assistant messages contain: `{role, stage1, stage2, stage3}`
- Note: metadata (label_to_model, aggregate_rankings) is NOT persisted to storage, only returned via API

**`main.py`**
- FastAPI app with CORS enabled for localhost:5173 and localhost:3000
- `build_message_history()`: Helper function that extracts conversation history from storage
  - Converts stored messages to OpenRouter-compatible format
  - Uses only stage3 (final synthesized) responses for assistant messages to keep context focused
  - Appends new user message to build complete history
- POST `/api/conversations/{id}/message` returns metadata in addition to stages
- POST `/api/conversations/{id}/message/stream` streams results progressively
- Metadata includes: label_to_model mapping and aggregate_rankings

### Frontend Structure (`frontend/src/`)

**`App.jsx`**
- Main orchestration: manages conversations list and current conversation
- Handles message sending and metadata storage
- Important: metadata is stored in the UI state for display but not persisted to backend JSON

**`components/ChatInterface.jsx`**
- Multiline textarea (3 rows, resizable)
- Enter to send, Shift+Enter for new line
- User messages wrapped in markdown-content class for padding

**`components/Stage1.jsx`**
- Tab view of individual model responses
- ReactMarkdown rendering with markdown-content wrapper

**`components/Stage2.jsx`**
- **Critical Feature**: Tab view showing RAW evaluation text from each model
- De-anonymization happens CLIENT-SIDE for display (models receive anonymous labels)
- Shows "Extracted Ranking" below each evaluation so users can validate parsing
- Aggregate rankings shown with average position and vote count
- Explanatory text clarifies that boldface model names are for readability only

**`components/Stage3.jsx`**
- Final synthesized answer from chairman
- Green-tinted background (#f0fff0) to highlight conclusion

**Styling (`*.css`)**
- Light mode theme (not dark mode)
- Primary color: #4a90e2 (blue)
- Global markdown styling in `index.css` with `.markdown-content` class
- 12px padding on all markdown content to prevent cluttered appearance

## Key Design Decisions

### Multi-Turn Conversation Support
The system maintains conversation context across multiple exchanges:
- All previous user and assistant messages are included when querying models
- Assistant messages use only the stage3 (final synthesized) response for context, not all individual council responses
- This keeps context focused while preserving the collaborative decision-making benefits
- Stage 1, 2, and 3 prompts all include conversation history (compacted if needed)
- Models can reference and build upon previous exchanges naturally

### Token Management for Long Conversations
To prevent token limit errors in extended conversations:
- **Automatic compacting**: When conversation exceeds 10 messages, old history is automatically summarized
- **Summarization**: Uses gemini-2.5-flash to create 2-3 sentence summaries of old context
- **Recency bias**: Recent messages (last 5 turns) are always kept in full detail
- **Stage-specific limits**:
  - Stage 1 & 2: Compact if >10 messages, keep last 10 messages in full
  - Stage 3 (Chairman): Limited to last 3 turns (configurable via `MAX_HISTORY_FOR_CHAIRMAN`)
- **Extended timeout**: Chairman gets 180s timeout (vs 120s default) due to complex synthesis task
- **Graceful fallback**: If chairman fails, returns top-ranked Stage 1 response instead of generic error

### Stage 2 Prompt Format
The Stage 2 prompt is very specific to ensure parseable output:
```
1. Evaluate each response individually first
2. Provide "FINAL RANKING:" header
3. Numbered list format: "1. Response C", "2. Response A", etc.
4. No additional text after ranking section
```

This strict format allows reliable parsing while still getting thoughtful evaluations.

### De-anonymization Strategy
- Models receive: "Response A", "Response B", etc.
- Backend creates mapping: `{"Response A": "openai/gpt-5.1", ...}`
- Frontend displays model names in **bold** for readability
- Users see explanation that original evaluation used anonymous labels
- This prevents bias while maintaining transparency

### Error Handling Philosophy
- Continue with successful responses if some models fail (graceful degradation)
- Never fail the entire request due to single model failure
- Log errors but don't expose to user unless all models fail

### UI/UX Transparency
- All raw outputs are inspectable via tabs
- Parsed rankings shown below raw text for validation
- Users can verify system's interpretation of model outputs
- This builds trust and allows debugging of edge cases

## Important Implementation Details

### Relative Imports
All backend modules use relative imports (e.g., `from .config import ...`) not absolute imports. This is critical for Python's module system to work correctly when running as `python -m backend.main`.

### Port Configuration
- Backend: 8001 (changed from 8000 to avoid conflict)
- Frontend: 5173 (Vite default)
- Update both `backend/main.py` and `frontend/src/api.js` if changing

### Markdown Rendering
All ReactMarkdown components must be wrapped in `<div className="markdown-content">` for proper spacing. This class is defined globally in `index.css`.

### Model Configuration
Models are hardcoded in `backend/config.py`. Chairman can be same or different from council members. The current default is Gemini as chairman per user preference.

## Common Gotchas

1. **Module Import Errors**: Always run backend as `python -m backend.main` from project root, not from backend directory
2. **CORS Issues**: Frontend must match allowed origins in `main.py` CORS middleware
3. **Ranking Parse Failures**: If models don't follow format, fallback regex extracts any "Response X" patterns in order
4. **Missing Metadata**: Metadata is ephemeral (not persisted), only available in API responses

## Future Enhancement Ideas

- Configurable council/chairman via UI instead of config file
- Streaming responses instead of batch loading
- Export conversations to markdown/PDF
- Model performance analytics over time
- Custom ranking criteria (not just accuracy/insight)
- Support for reasoning models (o1, etc.) with special handling

## Testing Notes

Use `test_openrouter.py` to verify API connectivity and test different model identifiers before adding to council. The script tests both streaming and non-streaming modes.

## Data Flow Summary

```
User Message + Conversation History
    ↓
Build Message History: Extract previous messages, use stage3 for assistant context
    ↓
Stage 1: Parallel queries with full history → [individual responses]
    ↓
Stage 2: Anonymize → Parallel ranking queries with history → [evaluations + parsed rankings]
    ↓
Aggregate Rankings Calculation → [sorted by avg position]
    ↓
Stage 3: Chairman synthesis with full context (history + rankings)
    ↓
Return: {stage1, stage2, stage3, metadata}
    ↓
Storage: Save complete message exchange
    ↓
Frontend: Display with tabs + validation UI
```

The entire flow is async/parallel where possible to minimize latency. Multi-turn conversations maintain context by passing all previous exchanges to each stage.

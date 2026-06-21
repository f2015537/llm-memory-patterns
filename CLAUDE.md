# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Setup

```bash
# Install dependencies into the existing venv
source .venv/bin/activate
pip install -r requirements.txt
```

Environment variables are loaded from `.env` via `python-dotenv`. Required keys: `OPENAI_API_KEY`, `LANGSMITH_API_KEY`, `LANGSMITH_PROJECT`, `LANGSMITH_ENDPOINT`, `LANGSMITH_TRACING`.

## Running scripts

Each file is a standalone interactive demo; run directly:

```bash
python 1_stateless.py
python 2_stm_ram.py
python 3_stm_thread.py
python 4_stm_slidingwindow.py
python 5_stm_summarization.py
python 6_ltm_userprofile.py
python 7_stm_ltm.py  # supports --user-id and --thread-id flags
```

In-session commands vary by script — common ones: `/history`, `/threads`, `/switch <id>`, `/facts` (script 6), `/show-ltm`, `/show-stm` (script 7), `exit`.

## Architecture

This is an educational progression of LLM memory patterns, all using OpenAI `gpt-4o` via LangChain:

| File | Memory strategy | Persistence |
|------|----------------|-------------|
| `1_stateless.py` | None — each call is a fresh context | None |
| `2_stm_ram.py` | Full history in a Python list | In-process only (resets on restart) |
| `3_stm_thread.py` | Full history, multiple named threads | SQLite (`data/memory/shortterm_memory_demo.db`) |
| `4_stm_slidingwindow.py` | `@before_model` middleware trims to last `WINDOW=4` messages | `SqliteSaver` → `data/memory/sliding_window.db` |
| `5_stm_summarization.py` | `@before_model` middleware compresses older messages into a summary once `SUMMARIZE_AFTER=6` is reached, keeping last `KEEP_RECENT=4` raw | `SqliteSaver` → `data/memory/summarization.db` |
| `6_ltm_userprofile.py` | Extracts durable user facts via a second LLM call after each turn; loads them into the system prompt on restart | `SqliteStore` → `data/memory/ltm_store.db` |
| `7_stm_ltm.py` | Combines both layers: `SqliteSaver` for per-thread STM + `SqliteStore` for cross-thread behavioral rules (procedural LTM) saved/retrieved via tools | `SqliteSaver` → `data/memory/stm.db`, `SqliteStore` → `data/memory/ltm.db` |

**Key architectural shift at script 4:** scripts 1–3 call `llm.invoke()` directly; scripts 4–7 use `create_agent` with `SqliteSaver` (STM checkpointing) and/or `SqliteStore` (LTM key-value store). Script 7 also introduces `ToolRuntime[Context]` so tools can access both the store and a typed `Context` dataclass (carries `user_id`) injected via `agent.invoke(..., context=context)`.

LangSmith tracing is active when `LANGSMITH_TRACING=true` in `.env`.

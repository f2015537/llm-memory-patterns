# LLM Memory Patterns

A progressive series of Python scripts that explore how conversational memory works in LLM applications — from a completely stateless call to a two-layer system with separate short-term and long-term memory stores.

Each script is self-contained and runnable. The series is designed to be read in order: each step introduces one new concept and builds directly on the previous one.

---

## Memory patterns covered

| Script | Pattern | Key idea |
|--------|---------|----------|
| `1_stateless.py` | No memory | Baseline — each LLM call is independent |
| `2_stm_ram.py` | In-process history | Full conversation history in a Python list |
| `3_stm_thread.py` | SQLite persistence + threads | History survives restarts; named threads |
| `4_stm_slidingwindow.py` | Sliding window | `@before_model` middleware trims to last N messages |
| `5_stm_summarization.py` | Summarization | Older messages compressed into a summary |
| `6_ltm_userprofile.py` | Long-term memory | Durable facts extracted per turn via `SqliteStore` |
| `7_stm_ltm.py` | STM + LTM combined | Per-thread conversation history + cross-session behavioral rules |

---

## Setup

**Prerequisites:** Python 3.11+, an [OpenAI API key](https://platform.openai.com/api-keys), and optionally a [LangSmith API key](https://smith.langchain.com) for tracing.

```bash
# Clone and enter the repo
git clone <repo-url>
cd Memory

# Create and activate a virtual environment
python -m venv .venv
source .venv/bin/activate       # macOS / Linux
.venv\Scripts\activate          # Windows

# Install dependencies
pip install -r requirements.txt

# Configure environment variables
cp .env.example .env
# Edit .env and add your OPENAI_API_KEY (and optionally LANGSMITH_API_KEY)
```

---

## Scripts

### 1. Stateless (`1_stateless.py`)

The simplest possible LLM call. No history is sent — every message is a brand-new context. Illustrates the baseline problem that all subsequent scripts solve.

```bash
python 1_stateless.py
```

```
You: My name is Alex.
LLM: Hello, Alex! How can I assist you today?

You: What is my name?
LLM: I'm sorry, but I don't have access to personal data about you,
     so I don't know your name.
```

The model answered the first question correctly, then immediately forgot.

---

### 2. Short-term memory in RAM (`2_stm_ram.py`)

The full conversation history is kept in a Python list and sent with every call. The model can now recall earlier messages — but only within the current process. The history is lost on restart.

```bash
python 2_stm_ram.py
```

```
You: My name is Alex.
LLM: Hello, Alex! How can I assist you today?

You: What is my name?
LLM: Your name is Alex.
```

**In-session commands:** `/history` to inspect the stored message list, `exit` to quit.

---

### 3. SQLite persistence + named threads (`3_stm_thread.py`)

History is written to SQLite after every turn and reloaded on startup. Conversations now survive restarts. Multiple independent threads are supported — each `thread_id` is a separate conversation.

```bash
python 3_stm_thread.py
```

**Session 1:**
```
You: My name is Alex. I work as a software engineer.
LLM: Hi Alex! It's great to meet you. As a software engineer, you must
     be working on some interesting projects...

You: exit
```

**Session 2 (fresh restart):**
```
You: What do you know about me?
LLM: I know that your name is Alex and you work as a software engineer.
```

**In-session commands:** `/history`, `/threads`, `/switch <thread_id>`, `exit`.

---

### 4. Sliding window (`4_stm_slidingwindow.py`)

Introduces `create_agent` from LangGraph and a `@before_model` middleware hook. Before every model call the middleware trims the stored message list to the last `WINDOW = 4` messages, keeping token usage bounded at the cost of losing early context.

```bash
python 4_stm_slidingwindow.py
```

**Tradeoff:** cheap and simple, but anything said more than 4 turns ago is permanently dropped.

**In-session commands:** `/history`, `/threads`, `/switch <id>`, `exit`.

---

### 5. Summarization (`5_stm_summarization.py`)

Instead of discarding old messages, a `@before_model` middleware compresses them into a 2–3 sentence summary once the stored count exceeds `SUMMARIZE_AFTER = 6`. The last `KEEP_RECENT = 4` messages are kept verbatim alongside the summary.

```bash
python 5_stm_summarization.py
```

```
You: ...  (after several turns)

  [summarization triggered — compressed 4 messages into summary]
  [summary: Alex introduced himself as a backend engineer at a fintech
   startup who prefers concise answers. The conversation covered...]
```

**Tradeoff:** retains the gist of early context, but fine-grained details may be lost.

**In-session commands:** `/history`, `/threads`, `/switch <id>`, `exit`.

---

### 6. Long-term memory — user profile (`6_ltm_userprofile.py`)

Introduces `SqliteStore`, LangGraph's durable key-value store. After every turn a second LLM call extracts durable facts (name, job, preferences, etc.) and writes them to the store. On the next startup those facts are loaded and injected into the system prompt — the model remembers you across completely separate sessions.

```bash
python 6_ltm_userprofile.py
```

**Session 1:**
```
You: Hi! My name is Alex, I'm a backend engineer at a fintech startup
     and I prefer concise answers.
LLM: Hi Alex! If you have any questions, feel free to ask.
  [stored 4 fact(s) to SqliteStore]

You: exit
```

**Session 2 (fresh restart):**
```
Loaded 4 fact(s) from previous session.

You: What do you know about me?
LLM: You are Alex, a backend engineer working in the fintech industry.
     You prefer concise answers.

You: /facts
{
  "name": "Alex",
  "job": "backend engineer",
  "industry": "fintech",
  "preferences": "concise answers"
}
```

**In-session commands:** `/facts` to inspect the store, `exit`.

---

### 7. STM + LTM combined (`7_stm_ltm.py`)

The full two-layer architecture:

- **Short-term memory** (`SqliteSaver`) — per-thread conversation history, restored automatically on reconnect to the same `thread_id`.
- **Long-term memory** (`SqliteStore`) — cross-session behavioral rules ("always respond in bullet points"). Saved via a `save_rule` tool that the agent calls autonomously; applied to every future session via the system prompt.

```bash
python 7_stm_ltm.py                           # default user + thread
python 7_stm_ltm.py --user-id alice --thread-id project-x
```

**Session 1 — teach a rule:**
```
user_id=demo-user  |  thread_id=thread-1

You: Always respond in bullet points.
Agent: - Got it!
       - Always responding in bullet points from now on.

You: What are the planets in our solar system?
Agent: - Mercury
       - Venus
       - Earth
       - Mars
       - Jupiter
       - Saturn
       - Uranus
       - Neptune

You: exit
```

**Session 2 — fresh restart, rule persists:**
```
Loaded 1 rule(s) from previous sessions.

You: What rules do you follow?
Agent: - I always respond in bullet points.
```

Switching to a new `thread_id` resets the conversation (STM) while keeping all learned rules (LTM) — because STM is scoped to a thread and LTM is scoped to a user.

**In-session commands:** `/show-ltm`, `/show-stm`, `/threads`, `/switch <thread-id>`, `exit`.

---

## Architecture progression

```
1_stateless          llm.invoke([current message])
2_stm_ram            llm.invoke([system + full history])
3_stm_thread         llm.invoke([system + full history])   ← history in SQLite
4_stm_slidingwindow  create_agent + @before_model (trim)   ← SqliteSaver
5_stm_summarization  create_agent + @before_model (summarize)
6_ltm_userprofile    llm.invoke + SqliteStore.put/get      ← fact extraction
7_stm_ltm            create_agent + SqliteSaver + SqliteStore + tools
```

The dividing line between scripts 3 and 4 is the shift from calling `llm.invoke()` directly to using LangGraph's `create_agent`, which handles the tool loop, checkpointing, and middleware automatically.

---

## Tech stack

- [LangChain](https://github.com/langchain-ai/langchain) — LLM abstraction and agent primitives
- [LangGraph](https://github.com/langchain-ai/langgraph) — `create_agent`, `SqliteSaver`, `SqliteStore`, `@before_model`
- [LangSmith](https://smith.langchain.com) — optional tracing (set `LANGSMITH_TRACING=true` in `.env`)
- OpenAI `gpt-4o` — model used across all scripts

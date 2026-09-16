# System Architecture & Verification Report

**Project:** WEKRAFT Agent Execution Platform  
**Target Architecture:** Orchestrator-Workers (Kaya Supervisor + Specialized Parallel Sub-Agents)  
**Date:** March 2026  
**Status:** Verified & Validated  

---

## 1. Executive Summary & Architecture Rating

### Architecture Rating: **8.5 / 10 (Production-Grade, Scalable)**

The proposed architecture follows the modern **Orchestrator-Workers & Fast Router pattern** endorsed by both Anthropic (*"Building Effective Agents"*) and OpenAI (*"Agents SDK & Multi-Agent Architecture Guide"*). 

### High-Level Blueprint
```
                   ┌────────────────────────────────────────┐
                   │               User Query               │
                   └───────────────────┬────────────────────┘
                                       │
                ┌──────────────────────┴──────────────────────┐
                │   Parallel Pipeline (asyncio.gather)       │
                │  - Layer 1-3 Input Guardrails (PII + Groq)  │
                │  - Cohere Semantic Cache (>= 0.92 cosine)   │
                └──────────────────────┬──────────────────────┘
                                       │ (Guardrail Safe & Cache Miss)
                                       ▼
                   ┌────────────────────────────────────────┐
                   │       Supervisor Router (Groq)         │
                   │    Fast Structured Intent Classifier   │
                   └───────────────────┬────────────────────┘
                                       │
            ┌──────────────────────────┴──────────────────────────┐
            │ Actions == ['direct_response']                      │ Actions in ['analyst', 'db_write', 'sprint']
            ▼                                                     ▼
┌───────────────────────────────┐               ┌───────────────────────────────────┐
│     Kaya Direct Response      │               │   Parallel Sub-Agent Execution    │
│  - Model: Groq 120b           │               │  - Model: gpt-4.1-mini            │
│  - Low-latency chat/greeting  │               │  - Private Thread: reset_or_add   │
│  - Token-by-token stream      │               │  - Sequential/Parallel Tools      │
└───────────────┬───────────────┘               │  - NO user token streaming        │
                │                               └─────────────────┬─────────────────┘
                │                                                 │
                │                                                 ▼ (Returns tool findings)
                │                               ┌───────────────────────────────────┐
                │                               │     Kaya Supervisor Synthesis     │
                │                               │  - Model: gpt-4.1-mini            │
                │                               │  - Synthesizes findings           │
                │                               │  - Streams final tokens to user   │
                │                               └─────────────────┬─────────────────┘
                │                                                 │
                ▼                                                 ▼
    ═════════════════════════════════════════════════════════════════════════════════
                              Redis Checkpointer (Upstash)
                     Durable LangGraph State & HITL Resumes
    ═════════════════════════════════════════════════════════════════════════════════
```

---

## new added 
1. The 4 Subagents & Their Isolated Threads
In state.py and kaya_graph.py, all 4 workers run with their own dedicated, isolated message buffers using smart reducers (RESET_SENTINEL):

Sub-Agent	Node Name	Private State Channel	Responsibility
1. Analyst	analyst_node	_analyst_messages	Read analytics: Standups, tasks, issues, team workloads, project timelines.
2. DB Write	db_write_node	_db_write_messages	Mutations & memory: Mem0 search, calendar events, report scheduler, PRD item creation.
3. Sprint	sprint_node	_sprint_messages	Sprint analytics, sprint velocity, sprint creation & backlog allocation.
4. MCP Integrations	mcp_node	_mcp_messages	External SaaS apps: Slack, Calendly, Linear, Notion, etc.
Each subagent runs in parallel fan-out without polluting the main conversation history or other workers' context. All 4 workers fan-in to the Kaya Synthesizer (kaya_synthesizer_node), which produces the final unified PM response.

## 2. Verification of Current Components

| Component | File Path | Status | Verdict & Quality |
| :--- | :--- | :--- | :--- |
| **Input Guardrails** | `app/core/guardrails/input_guardrails.py` | Verified | **Excellent.** 3-layer architecture: Layer 1 PII redaction (retaining email), Layer 2 regex (jailbreak, code-gen, medical, off-topic), Layer 3 Groq LLM safeguard with `<= 0.8` risk cutoff. Fast and robust. |
| **Semantic Cache** | `app/core/cache/semantic_cache.py` | Verified | **Solid.** Cohere ClientV2 embeddings with `embed-english-v3.0`, cosine similarity >= 0.92, run concurrently with guardrails via `asyncio.gather`. Bypasses cache on greetings and document uploads. |
| **Checkpointer** | `app/core/utils/checkpointer.py` | Verified | **Clean.** Completely purged of Postgres. Implements Upstash Redis (`RedisSaver` / `AsyncRedisSaver`) with a fallback to `MemorySaver` for testing. |
| **Central State** | `app/state/state.py` | Verified | **Modular & Scalable.** Contains `SupervisorState` with `add_messages` reducer for conversation history and `reset_or_add` reducer with `RESET_SENTINEL` for isolated sub-agent scratchpads. |
| **Supervisor Router** | `app/agents/graph/kaya_graph.py` | Verified | **Functional.** Structured output with Pydantic (`SupervisorDecision`), multi-action parallel selection (`actions: List[str]`). |

---

## 3. Core Architectural Strengths

1. **Dynamic Model & Cost Tiering**:
   - Routing trivial queries (greetings, general chat) to an ultra-fast model (Groq 120b) delivers sub-250ms Time-To-First-Token (TTFT) and near-zero cost.
   - Engaging `gpt-4.1-mini` only when domain tools and analysis are required reserves compute budget for reasoning-intensive workflows.

2. **Decoupled Worker Execution vs. Unified Synthesis**:
   - Sub-agents executing silently without streaming tokens to the client prevents "token cross-talk" and noisy raw tool output from reaching the user.
   - Kaya acts as an executive summarizer, translating technical metrics into PM-level insights.

3. **Isolated Thread Memory (`reset_or_add`)**:
   - Large raw JSON outputs from tools (e.g. 50 tasks from Convex) stay within `_analyst_messages` or `_sprint_messages`, preventing primary `messages` context window explosion and reducing synthesis token costs.

4. **Human-In-The-Loop (HITL) Safety for Writes**:
   - Destructive state mutations (calendar event writes, sprint creations, bulk tasks) trigger `langgraph.types.interrupt()` and pause on the Redis checkpointer until approved.

---

## 4. Key Drawbacks & Mitigation Plan

| Drawback | Impact | Mitigation Strategy |
| :--- | :--- | :--- |
| **Perceived UI Latency** | If sub-agents take 2–4s calling multiple APIs, the user sees a loading pause before Kaya starts streaming tokens. | Emit real-time SSE progress events (`agent_status: "Analyst is calculating sprint velocity..."`) via LangGraph stream writer. |
| **Synthesis Information Loss** | Kaya might compress or drop critical data points returned by sub-agents. | Require sub-agents to format outputs in typed JSON or standardized Markdown summaries before handoff. |
| **Sub-Agent Cascading Failures** | If one parallel sub-agent throws an unhandled exception, it could break the whole graph run. | Wrap sub-agent node executions with try/except returning structured `{"error": "..."}` so Kaya can gracefully explain partial failures. |
| **Semantic Cache In-Memory Storage** | `semantic_cache.py` currently stores embeddings in a Python list (`self.cache`), which resets on process restart. | Connect cache storage to Upstash Redis for multi-instance persistence. |


Goals ->
1. Each sub-agent can call tools in prallel never sequential.
2. sub-agent should retunrn proper findings/result to kaya never half or broken.
3. kaya should aware of the project details/deadline always.
-------------------------------------------------------------------------------------
-------------------------------------------------------------------------------------

## BRAIN 

Integrations: The incoming data pipelines (OAuth / webhooks / API keys).
Project Brain: The centralized, per-project searchable knowledge store (vector index + synced records).
Kaya & Harry: The only two agents your users ever interact with. The Brain gives them superpowers.

1. The Integrations Route (/integrations)
Keep it strictly divided by persona. This tells the user exactly why they are connecting a tool.
Layout:
Header: "Connect your tools to empower your agents."
Column 1: Kaya (PM Context)
Notion (Specs & Docs)
Slack (Decisions & Updates)
Calendly (Availability)
Jira/Linear (Only if migrating/legacy)
Column 2: Harry (Dev Context)
GitHub (Code & PRs)
Vercel (Deployments)
Sentry (Errors)
Status Indicators:
Green dot: Connected.
Grey button: Connect.
Crucial: Show "Last Synced: 2 mins ago" so they know the brain is alive.

2. The "Sub-Agent" (The Backend Logic)
Do not show a "Sub-Agent" in the UI. The user should never have to talk to a second bot.
Instead, build a Context Tool that Kaya uses silently.
How it works:
User asks Kaya: "Create a task for the new onboarding flow."
Kaya's Internal Monologue (Invisible):
Step 1: I need requirements. -> Call Notion Tool.
Step 2: I need to know who is available. -> Call Calendly Tool.
Step 3: I need to check if Harry is blocked on similar code. -> Call GitHub Tool.
Kaya's Output: "I've created the task based on the Notion spec. I assigned it to you because your calendar is clear, and I added a note for Harry to check the existing auth.ts file in GitHub."

3. The UI inside Kaya's Chat
Since the brain is invisible, how does the user know it's working?
Option A: Silent (Best for speed)
Kaya just answers perfectly. The user assumes she is smart.
Option B: "Thinking" State (Best for trust)
When Kaya is processing, show small, fading text:
Kaya is reading Notion docs...
Kaya is checking GitHub PRs...
Kaya is checking Sentry logs...

==============================================================
## ERRORS: 

biggest issue 

[ROUTER DECISION] Actions: ['direct_response'] | Reasoning: The user provided a brief affirmative response ('yes') without requesting any data, integration, or analysis. This qualifies as a simple conversational reply, so only the 'direct_response' action is needed.
[custom_event] data={'agent_status': 'Kaya is reasoning...', 'reasoning': "The user provided a brief affirmative response ('yes') without requesting any data, integration, or analysis. This qualifies as a simple conversational reply, so only the 'direct_response' action is needed."}
[custom_event] data={'agent_status': 'Kaya is typing...'}
INFO:httpx2:HTTP Request: POST https://api.groq.com/openai/v1/chat/completions "HTTP/1.1 200 OK"
[custom_event] data={'status': 'completed', 'thread_id': '68289898-5356-4ee8-83b2-7371090972bb', 'agent': 'kaya'}
---> i said to make tasks from linear jira - it showed and ask confirm 
-> i said yes etc.......
---> router didnt called any tool !!
bro this is so fucked ! as router has no idea !



The biggest issue is the Synthesizer — not the router, not the sub-agents.

The kaya_synthesizer_node receives sub-agent outputs as free-text Markdown, passes them to gpt-4.1-mini, and hopes the LLM doesn't hallucinate. In your session:

Router over-dispatched: sent ["db_write", "mcp"] for a simple "create those 3" confirmation → MCP ran useless Jira searches
db_write_node either crashed (the getStateDiff frontend bug) or the task extraction LLM silently failed (error swallowed with print())
The synthesizer saw "user asked to create tasks" + "DB Write agent ran" in its context and fabricated that it was done — there's no structured receipt proving anything was actually written
The bulk_create_tasks tool was never called because the HITL interrupt never fired (extraction failure or frontend crash prevented it).

The fix priority:

Structured mutation receipts in state (not free-text) so the synthesizer knows whether writes succeeded
Don't co-dispatch MCP alongside db_write on confirmation flows
Surface extraction failures loudly instead of swallowing them
Review the full analysis and let me know if you'd like me to implement the fixes.


Router Amnesia on Short Confirmations: When the agent asks for confirmation in text and you reply "yes go ahead", the Supervisor Router inspects only the isolated prompt "yes go ahead". Without context or active state, the router classifies it as casual chat (direct_response) and never calls the worker sub-agent.
Double Confirmation Mismatch: The sub-agent was prompting for textual confirmation in chat messages instead of leveraging your HITL (Human-in-the-Loop) flow.

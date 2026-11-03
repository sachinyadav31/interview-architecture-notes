# 15: AI copilot / chatbot (FastAPI + LangGraph, multi-provider)

## What it does
A chat assistant inside the Procurement and Sales portals. Users type "create a PO for 10 laptops from Dell" or "show spend by category". The agent **calls our real APIs** (551 tools) and returns text, **charts** (`TextGraph`), or a **form** pre-filled for the user to confirm.

Repo: `Inclufy-AI-Chatbot-Backend` (Python, FastAPI, LangGraph/LangChain). Separate apps: `procurement_api.py`, `sales_api.py`.

## Architecture
```mermaid
flowchart LR
  UI[Portal chat widget] -->|POST /chat {message, session_id, auth_token, ai_config}| API[FastAPI]
  API --> SES[(session store<br/>messages per session)]
  API --> G[LangGraph StateGraph]
  subgraph G[LangGraph agent]
    AG[agent node<br/>LLM.bind_tools(tools)] -->|tool_calls?| TN[ToolNode]
    AG -->|no tool call| END1[END]
    TN --> END2[END → format reply]
  end
  AG --> F{provider factory}
  F --> CL[Claude]
  F --> OA[OpenAI]
  F --> TG[Together<br/>Llama 3.3 70B default]
  TN -->|HTTP with the user's JWT| GW[eProcure gateway]
  GW --> SVC[PO / supplier / catalog / esourcing ... services]
```

## Low-level parts
- **Provider abstraction** (`providers/base.py` → `ClaudeProvider`, `OpenAIProvider`, `TogetherProvider`): a common interface `chat()`, `validate_api_key()`, `get_available_models()`. `factory.get_ai_provider(name, api_key)` is a dict-based **Factory**, defaulting to Together + `Llama-3.3-70B-Instruct-Turbo` from env.
- **LLM wrappers** (`agent/llm/chat_claude.py`, `chat_openai.py`, `chat_together.py`) implement LangChain's chat-model interface including `bind_tools()`, so the same graph works with any provider.
- **Bring your own key**: `ai_config.use_personal_key` + `settings_token` → fetch the user's own API key from the settings service, so the tenant pays its own LLM bill.
- **Graph** (`agent/procurement/chatbot.py`):
  ```python
  graph = StateGraph(AgentState)          # messages with add_messages reducer
  graph.add_node("agent", create_agent_node(llm))
  graph.add_node("tools", ToolNode(procurement_tools))
  graph.add_conditional_edges("agent", should_call_tools, {"tools": "tools", END: END})
  graph.add_edge("tools", END)
  app = graph.compile(); result = app.invoke(state)
  ```
- **Tools** (`@tool` functions, about 550): `order_tools`, `supplier_tools`, `esourcing_rfq/rfp/rfi`, `catalog_tools`, `budget_tools`, `invoice_payment_tools`, `warehouse_tools`, `manufacturing_tools`, `logistics_tools`, `graphs.py` (chart data)… Each tool calls the gateway with **the user's own token** (`get_current_token()` from the request context), so **our ACLs still apply**. The AI can never do more than the user could.
- **Form flow**: for create/update actions, a tool returns a **form schema** (fields, types, options, editable, `object_array` with subfields) instead of writing directly. The UI renders it; `/submit-form` executes it, `/cancel-form` drops it, `/add-form-array-entry` and `/remove-form-array-entry` edit line items. **Human-in-the-loop** for anything that writes.
- **Errors**: `agent/llm/errors.py` maps provider errors (bad key, rate limit, context too long) to friendly messages. I "enhanced AI copilot errors" so users see the real cause.
- Upskill backend has an "AI copilot for all entities" built the same way (May 2026).

## IQ Helix AI (FastAPI, see 16)
- AI course/assessment generation calls Anthropic `messages.create` or OpenAI `chat.completions.create` depending on config. I surfaced real AI errors instead of generic 500s and added the missing `generate-ai` route.

## Weak spots and how I'd improve them
- 550 tools in one prompt is expensive and confuses the model. Fix: **tool routing**: first classify intent (orders/suppliers/sourcing…), then bind only that group's tools; or retrieve tools by embedding similarity.
- Sessions in memory (a dict): lost on restart and not shared across replicas. Use Redis or a LangGraph checkpointer (Postgres).
- Prompt injection: data returned by tools (supplier names, descriptions) could contain instructions. Keep write actions behind forms (we do) and never let tool output change permissions.
- Add evals: a fixed set of prompts with expected tool calls, run in CI when prompts or models change.

## Interview Q&A

**Q: How does the AI actually do things in your system?**
A: Tool calling. Every API action is wrapped as a typed tool. The LLM decides which tool to call with which arguments; LangGraph's ToolNode runs it against our gateway with the user's JWT, so authorization is unchanged. Write actions return a pre-filled form for the user to confirm.

**Q: Why LangGraph instead of a plain loop?**
A: An explicit state machine (nodes and edges, conditional routing), built-in ToolNode, a message-state reducer, and easy extension to multi-step flows, checkpoints and human-in-the-loop interrupts.

**Q: How do you support several LLM providers?**
A: An adapter interface per provider (Claude, OpenAI, Together) with the same `chat`/`bind_tools` contract, a factory that picks one per request from `ai_config`, and optional per-user keys. Switching models is a config change.

**Q: How do you stop the AI from doing something dangerous?**
A: It acts with the user's own token, so ACLs apply; writes need explicit user confirmation through forms; tools validate input; there's an allow-list of tools per portal; and logs/audit of tool calls.

**Q: How do you control cost and latency?**
A: Smaller, faster models for routing and classification, bigger ones only when needed; fewer tools per call; trimming conversation history; caching repeated reference lookups; streaming responses.

**Q: RAG vs tool calling?**
A: RAG retrieves *documents* to ground answers (policies, manuals). Tool calling performs *actions* or fetches *live structured data*. We mainly needed live data and actions, so tools; RAG would fit "how do I configure approval matrices?" help questions.

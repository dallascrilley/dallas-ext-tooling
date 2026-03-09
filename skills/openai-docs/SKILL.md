---
name: openai-docs
description: Use when the user needs authoritative, current guidance on OpenAI products or APIs (Codex, Responses API, Chat Completions, Apps SDK, Agents SDK, Realtime, model capabilities) — prioritize the OpenAI docs MCP server; only fall back to web search on official OpenAI domains if MCP returns nothing.
---

# OpenAI Docs

## Workflow

1. Clarify the product scope (Codex, OpenAI API, ChatGPT Apps SDK) and the specific question.
2. Search with `mcp__openaiDeveloperDocs__search_openai_docs` using a precise query.
3. Fetch the best page with `mcp__openaiDeveloperDocs__fetch_openai_doc` (use `anchor` when available).
4. Answer with concise guidance and cite the doc source.
5. Provide code snippets only when the docs support them.

## MCP tool reference

| Tool | When to use |
|------|-------------|
| `mcp__openaiDeveloperDocs__search_openai_docs` | First step — find relevant pages |
| `mcp__openaiDeveloperDocs__fetch_openai_doc` | Pull exact sections from a known page |
| `mcp__openaiDeveloperDocs__list_openai_docs` | Browsing/discovery without a clear query |

## Product snapshots

- **Chat Completions API**: Generate model responses from a message list.
- **Responses API**: Unified endpoint for stateful, multimodal, tool-using agentic workflows.
- **Agents SDK**: Build agentic apps with tool use, agent handoffs, streaming, and full traces.
- **Apps SDK**: Build ChatGPT apps via web component UI + MCP server exposing your tools to ChatGPT.
- **Codex**: OpenAI's coding agent for writing, reviewing, and debugging code.
- **Realtime API**: Low-latency multimodal API including speech-to-speech conversations.
- **gpt-oss**: Open-weight reasoning models (gpt-oss-120b, gpt-oss-20b) under Apache 2.0.

## If MCP server is missing

1. Run: `codex mcp add openaiDeveloperDocs --url https://developers.openai.com/mcp`
2. If permissions fail, retry with escalated permissions (include one-sentence justification).
3. Only if escalated attempt fails, ask the user to run the command.
4. Ask the user to restart Codex.
5. Re-run the doc search after restart.

## Quality rules

- Treat OpenAI docs as source of truth — do not speculate.
- Prefer paraphrase with citations over long verbatim quotes.
- If multiple doc pages conflict, call out the difference and cite both.
- If docs don't cover the need, say so and offer next steps (e.g., community forum, GitHub issues).
- For OpenAI questions, always use MCP tools before web search.
- Web search fallback: restrict to `developers.openai.com` and `platform.openai.com` only.

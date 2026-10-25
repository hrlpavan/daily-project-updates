---
title: How I Shipped 15 Green MRs to GitLab in One Day: Inside Our Modern Contributor Tech Stack
published: false
description: A technical deep dive into the Ruby on Rails, Model Context Protocol (MCP), TypeScript LSP, and Python AI Gateway stack used to solve 15 upstream GitLab issues at 100x velocity.
tags: gitlab, opensource, webdev, programming
canonical_url: https://github.com/hrlpavan/daily-project-updates/blob/main/updates/2026-10-25.md
---

Contributing to one of the largest open-source platforms in the world is usually described as slow, deliberate, and fraught with complex CI pipelines. 

Today, across three distinct upstream GitLab repositories, our engineering pipeline achieved a landmark milestone: **15 out of 15 Merge Requests 100% Green in CI, 100% assigned to maintainers and domain reviewers**, and officially crossing **4,197 live contributor points** on track to the prestigious **Level 4 Contributor milestone (7,500 pts)**.

How do you tackle 15 upstream issues across **Ruby**, **TypeScript**, and **Python** in a single sprint without breaking pipelines or getting bogged down in review cycles?

Here is an architectural breakdown of the **modern tech stack** we use every day to solve GitLab issues at enterprise velocity.

---

## 🏛️ The Tri-Repository Architecture

GitLab is no longer just a monolithic Ruby on Rails application. Modern AI-assisted software delivery requires a distributed ecosystem:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Developer / Agentic Client                      │
│                  (Claude Desktop, Cline, Cursor, Antigravity)          │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Model Context Protocol (JSON-RPC)
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   GitLab Core MCP Server (Ruby on Rails)               │
│                  lib/api/mcp/handlers & JSON-RPC Router                │
│                                                                        │
│   • list_pipeline_schedules   • create_merge_request_pipeline          │
│   • list_work_items (filter)  • save_work_item (description_edits)     │
│   • list_user_events          • get_pipeline (poll hints)              │
└───────────────────┬────────────────────────────────┬───────────────────┘
                    │                                │
                    ▼                                ▼
┌──────────────────────────────────────┐  ┌──────────────────────────────┐
│  GitLab LSP (TypeScript / Node.js)   │  │  AI Gateway (Python / FastAPI│
│  editor-extensions/gitlab-lsp        │  │  modelops/.../ai-assist      │
│                                      │  │                              │
│   • Virtual Workspace File Indexing  │  │   • Agent Platform Guard     │
│   • WebSocket Lifecycle & Leak Fix   │  │   • Duo Workflow Service     │
│   • Multibyte UTF-8 Chunk Decoder    │  │   • APIStatusError Handling  │
│   • CLI Slash Command Wake Queue     │  │   • Tool Execution Loop      │
└──────────────────────────────────────┘  └──────────────────────────────┘
```

---

## 💎 Tier 1: GitLab Core & The Model Context Protocol (Ruby on Rails)

GitLab Core is pioneering native support for Anthropic's **Model Context Protocol (MCP)** directly within its Rails application (`lib/api/mcp/`). This allows AI agents to interface directly with GitLab issues, pipelines, notes, and work items via structured JSON-RPC.

### The Stack:
- **Language**: Ruby 3.2.x / 3.3.x
- **Framework**: Ruby on Rails 7.x, Grape (REST), GraphQL-Ruby
- **Testing**: RSpec with predictive test execution (`rspec:predictive`)
- **Linting**: RuboCop with custom GitLab security and style cops

### What We Built Today:
We implemented and polished **9 distinct MCP tool endpoints**:
1. **`list_pipeline_schedules`** (`!264856`): Exposing CI schedule discovery to AI agents.
2. **`create_merge_request_pipeline`** (`!264858`): Giving agents the ability to trigger targeted MR pipeline runs.
3. **`save_work_item` with `description_edits` & `expected_updated_at`** (`!264862`): Adding optimistic locking and targeted string replacement diffs to prevent concurrent edit collisions.
4. **`list_work_items` with `parent_id` hierarchy filter** (`!264859`): Allowing recursive task tree traversal.
5. **`get_pipeline` with `poll_after_seconds` hint** (`!264863`): Teaching AI agents how long to sleep before polling pipeline status.
6. **`list_user_events`** (`!264864`): Activity feed telemetry tool for AI agents.
7. **Alias search resilience** (`!264852`): Transparently accepting `query` and `search_term`.
8. **Friendly schema validation** (`!264853`): Reporting unknown parameter names with suggestions.
9. **`save_note` updating** (`!264854`): Allowing AI agents to update existing issue notes.

### Pro-Tip: RuboCop Instance Variable Rule
In GitLab Rails helpers, never declare unmemoized instance variables (`@variable`). RuboCop will flag this under `Gitlab/PredicateMemoization` or helper encapsulation rules. Clean encapsulation in service classes keeps CI 100% green on the first push!

---

## ⚡ Tier 2: GitLab Language Server (TypeScript & Node.js)

The `gitlab-lsp` powers language intelligence and Duo AI features inside VS Code, WebIDE, and JetBrains.

### The Stack:
- **Language**: TypeScript 5.x (Strict Mode)
- **Runtime**: Node.js 20+ LTS
- **Protocols**: Language Server Protocol (LSP 3.17), WebSockets (`ws`)
- **Testing**: Vitest / Jest

### Critical Bugs Fixed Today:
1. **WebSocket Upgrade Rejection Ping Leak (`!4537`, `#3364`)**:
   When an authentication failure or 401/403 occurred during a WebSocket upgrade handshake, the background ping interval timer (`NodeJS.Timeout`) was not cleared. Over time, reconnect attempts created zombie timers that leaked memory.
   ```typescript
   // Explicit cleanup on upgrade rejection
   this.ws.on('unexpected-response', (_req, res) => {
     this.clearHeartbeat();
     this.emit('error', new UpgradeError(res.statusCode));
   });
   ```
2. **Multibyte UTF-8 Chunk Boundaries (`!4538`, `#3350`)**:
   Streaming responses that split multi-byte UTF-8 runes (such as emojis or non-ASCII characters) across arbitrary socket buffers were previously corrupting strings. Using `StringDecoder('utf8')` guarantees chunk boundary safety.
3. **Virtual Workspace Context Keying (`!4536`, `#2381`)**:
   Normalized file path resolution so AI context providers index virtual documents identically across remote workspaces.

---

## 🐍 Tier 3: AI Gateway & Duo Workflow (Python & FastAPI)

The `ai-assist` repository runs the GitLab AI Gateway—the engine orchestrating large language models and multi-agent execution loops.

### The Stack:
- **Language**: Python 3.12+
- **Framework**: FastAPI (Asynchronous ASGI), Starlette, Pydantic v2
- **Package Manager**: Poetry
- **Testing & Typing**: Pytest, Pytest-asyncio, Mypy, Ruff

### Critical Features Added Today:
1. **Identical Tool-Call Guard Enabled by Default (`!7641`, `#3394`)**:
   When an agent loops and calls the exact same tool with the exact same parameters without receiving new state, it enters a runaway infinite loop. We enabled an idempotency guard by default that intercepts duplicate calls and forces the agent to replan or terminate cleanly.
2. **Resilient Exception Mapping (`!7642`, `#3345`)**:
   Directly inspecting `status_code` from client `APIStatusError` exceptions rather than letting unhandled errors cascade into generic HTTP 500 crashes.

---

## 📊 The Scoreboard: Sprint Results

| Metric | Standing | Impact |
| :--- | :--- | :--- |
| **Current Live Score** | **4,197 points** | Verified Level 3 Contributor |
| **Points Added Today** | **+327 points** | Verified through Contributor API |
| **Active Merge Requests** | **15 MRs** | 9 in Core, 4 in LSP, 2 in AI Gateway |
| **CI Success Rate** | **15 / 15 (100% Green)** | Zero test failures |
| **Reviewer Coverage** | **15 / 15 Assigned** | Zero unassigned MRs |
| **Unresolved Threads** | **0 Blockers** | Sub-30-minute review SLA readiness |
| **Projected Total Upon Merge** | **7,545+ points** | **Surpasses Level 4 Target (`7,500 pts`)** |

---

## 🎯 3 Takeaways for Open Source Contributors

1. **Invest in Local Pre-Flight Verification**: Run Danger, RuboCop, and predictive RSpec locally before opening an MR. A single red CI pipeline adds hours of delay.
2. **Embrace Multi-Language Fluency**: True full-stack engineering means seamlessly switching between Ruby on Rails backend logic, TypeScript language server streaming, and Python async microservices.
3. **Adopt Standardized Protocols Early**: Standardized interfaces like **Model Context Protocol (MCP)** and **LSP** are the future of developer tooling. Learning their spec pays massive dividends.

---

*Follow our daily engineering journey across GitHub ([@hrlpavan](https://github.com/hrlpavan)) and GitLab ([@hrlpavan](https://gitlab.com/hrlpavan)).*

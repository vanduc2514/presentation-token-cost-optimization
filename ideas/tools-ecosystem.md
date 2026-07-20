# Tools & Ecosystem for VS Code Copilot Token Optimization

*Research compiled July 2026. Covers built-in VS Code Copilot features, VS Code extensions,
open-source CLI tools, and community workflows commonly used to monitor and optimize
token consumption.*

---

## Table of Contents

1. [Built-In VS Code Copilot Features](#1-built-in-vs-code-copilot-features)
2. [VS Code Extensions (Marketplace)](#2-vs-code-extensions-marketplace)
3. [Open-Source CLI & MCP Tools](#3-open-source-cli--mcp-tools)
4. [Community Workflows & Patterns](#4-community-workflows--patterns)
5. [Comparison Matrix](#5-comparison-matrix)
6. [Quick-Start Recommendations](#6-quick-start-recommendations)

---

## 1. Built-In VS Code Copilot Features

These are native to VS Code / GitHub Copilot — no installation needed.

### 1.1 Chat Debug View

**What it does**: Shows every model call with full token breakdown.

**How to access**: Copilot Chat panel → `...` (overflow menu) → "Show Chat Debug View"

**What you see per request**:
- `model` — which model was used (e.g. `claude-sonnet-4-6`)
- `usage.prompt_tokens` — total input tokens
- `usage.completion_tokens` — output tokens
- `usage.prompt_tokens_details.cached_tokens` — cached input tokens
- `usage.prompt_tokens_details.cache_creation_input_tokens` — cache writes
- `copilotUsage` — final cost in AIC (e.g. `1.94 AIC`)
- `duration` / `timeToFirstToken` — latency metrics

**Best for**: Post-hoc analysis of why a specific request cost what it did.

### 1.2 Agent Debug Logs

**What it does**: Full execution trace of an agentic task — every tool call, file read, and model response.

**How to access**: After running an Agent mode task, look for "Show Agent Debug Logs" in the
Copilot Chat panel overflow menu or in the Chat view.

**Best for**: Understanding where tokens went in a multi-step agent session.

### 1.3 Slash Commands — `/clear` and `/new`

**What they do**: Clear conversation history, resetting the context window.

| Command | Where | Effect |
|---|---|---|
| `/clear` | Copilot CLI | Clears session, resets cache boundary |
| New Chat button | VS Code Copilot Chat | Fresh context window |
| `/compact` | Copilot CLI | Summarizes history to free space (with info loss) |

**Token impact**: Long conversations carry full history. A fresh session can save 10-40%
of input tokens vs continuing a bloated one.

**Best practice**: Start a new chat when switching tasks. Don't steer a bloated session
with 80% context already filled — recency bias works against you.

### 1.4 Auto Model Selection

**What it does**: A small router examines your prompt and selects the most cost-effective
model that can handle the task.

**Key facts**:
- Reserves expensive reasoning models for complex problems
- Only changes models at natural cache boundaries (new session or `/compact`)
- Routes around degraded/busy models — fewer rate limits and errors
- **10% discount** on paid Copilot plans when enabled
- Supported for VS Code, Copilot CLI, GitHub Copilot app, and Copilot cloud agent

**How to enable**: VS Code settings → search "Auto Model Selection" or find it in the
model picker dropdown in the Copilot Chat panel.

**Token impact**: Automatic. Reduces blended cost rate without any manual effort.

### 1.5 Extended Prompt Caching (24h Retention)

**What it does**: Keeps cached model state alive for up to 24 hours instead of the default
5-10 minute window.

**How it works**: VS Code sets `"prompt_cache_retention": "24h"` for supported
OpenAI models. The cache moves from fast GPU memory (dropped after minutes of inactivity)
to slower but roomier GPU-local storage kept for 24h.

**Measured improvements**:

| Gap between requests | GPT-5.4 | GPT-5.3-Codex | GPT-5.5 |
|---|---|---|---|
| 10-20 min | +13% | +32% | +10% |
| 20-30 min | +135% | +142% | +137% |
| 30-40 min | +301% | +203% | +679% |
| 40-60 min | +338% | +279% | +919% |

**Best for**: Sessions with natural pauses (lunch breaks, meetings, context-switching).
Automatic — no user configuration needed.

### 1.6 Tool Search (On-Demand Tool Loading)

**What it does**: Instead of sending every tool definition on every request, the model
sees only lightweight metadata (name + description). Heavier parameter schemas are
loaded only when the model explicitly searches for and uses a tool.

**How it works**:
- **OpenAI**: Native `defer_loading` flag on tools (GPT-5.4+)
- **Anthropic**: Client-side tool search using Copilot's internal embedding model for
  semantic matching — finds tools by intent, not just keywords

**Measured savings**:

| Metric | GPT-5.4 | GPT-5.3-Codex | Claude Sonnet 4.6 |
|---|---|---|---|
| Total tokens per turn (P50) | -9.81% | -8.61% | -11.09% |
| Total tokens per session (P50) | -8.97% | -10.92% | -18.03% |
| Prompt tokens per turn (P50) | — | — | -11.30% |
| User error rate | — | — | -4.01% |

**Available since**: VS Code 1.118+ (April 2026). Automatic — no configuration needed.

### 1.7 WebSockets Transport

**What it does**: Replaces repeated HTTP requests with a persistent WebSocket connection
for sequential model calls in agentic sessions.

**Benefits**:
- Lower latency: TTFT improved by ~16-19%, Time to Complete by ~12-14%
- Connection-local in-memory cache reduces continuation overhead
- Default transport for OpenAI models GPT-5.2+ across Copilot products

**Token impact**: Reduces per-step overhead in long tool-calling chains. Automatic.

### 1.8 Content Exclusions (Enterprise)

**What it does**: Lets Business/Enterprise admins exclude specific folders/files from
being sent as context.

**Configured in**: Organization settings or `.github/copilot-instructions.yml`

**Common exclusions**: `node_modules/`, `dist/`, `build/`, `.git/`, `vendor/`,
large generated files.

**Token impact**: Prevents thousands of tokens of noise from being sent on every request.

### 1.9 Chart: VS Code Copilot Context Window Anatomy

```
┌──────────────────────────────────────────────────────────────┐
│                    PROMPT PREFIX (cached)                     │
├──────────────────────────────────────────────────────────────┤
│  System Prompt (~5K Ask / ~14K Agent)                        │
│  Tool Definitions (varies by # of tools)                     │
│  Conversation History (grows with each turn)                  │
├──────────────────────────────────────────────────────────────┤
│                    PER-REQUEST CONTENT                        │
├──────────────────────────────────────────────────────────────┤
│  Your typed prompt (10-200 tokens — tiny fraction)            │
│  Active file content (full file)                              │
│  Open tab snippets (15-20% of each tab)                      │
│  Workspace index retrieval (~1.5K-3K tokens)                  │
├──────────────────────────────────────────────────────────────┤
│                    MODEL OUTPUT                               │
├──────────────────────────────────────────────────────────────┤
│  Response text (5x more expensive per token than input)       │
│  Thinking / reasoning tokens (if applicable)                  │
│  Tool call invocations                                       │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. VS Code Extensions (Marketplace)

### 2.1 Copilot Token Awareness

| Attribute | Detail |
|---|---|
| Publisher | Raj Uppadhyay |
| Installs | ~100+ (new) |
| Price | Free |
| ID | `RajUppadhyay.copilot-token-awareness` |

**What it does**: Live, transparent estimate of how many tokens Copilot will consume
*before* you send a message.

**Key features**:
- **Live status bar**: `[Ask] ~23,129 tokens | ~$0.0694` — updates as you type
- **Ask/Agent mode toggle**: Each mode uses different estimation assumptions
- **Breakdown panel**: Per-source token table (system prompt, custom instructions,
  active file, open tabs, workspace index) with step-by-step cost calculation
- **23 built-in models**: All current pricing from official GitHub docs
- **Included-model detection**: Flags GPT-4.1 and GPT-5 mini as "included models"
  that don't consume credits within your allowance
- **Custom instructions detection**: Auto-detects `.github/copilot-instructions.md`
  and `*.instructions.md` files
- **Optimisation tips**: Rule-based hints — too many tabs open, file too large,
  approaching context window limit
- **User-patchable pricing**: Override multipliers or add new models via settings

**How it estimates**:

| Context Source | Ask Mode | Agent Mode |
|---|---|---|
| System prompt | ~5,000 tokens | ~14,000 tokens |
| Custom instructions | Actual file tokens | Actual file tokens |
| Active file | Full file tokens | Full file tokens |
| Open tabs | 20% of each file | 15% of each file |
| Workspace index | ~1,500 tokens | ~3,000 tokens |

**Accuracy**: ±10% (Ask mode), ±15% (Agent mode)

**Best for**: Awareness and education — seeing live what your context costs.

### 2.2 Copilot Cost & Token Tracker

| Attribute | Detail |
|---|---|
| Publisher | Netcom Labs AI |
| Installs | ~2,800 |
| Price | Free |
| ID | `netcomlabs-ai.copilot-cost-token-tracker` |

**What it does**: Post-hoc visualization of actual Copilot agent session costs
by reading local debug logs.

**Key features**:
- **Latest Session**: Live per-model breakdown of current agent session (auto-refresh)
- **All Sessions**: Sortable table with Cost, Credits, Cache Hit % columns
- **Aggregate**: Rolled-up totals across a time window, broken down by model
- **Per-model token breakdown**: Fresh input, cached input, cache writes, output
- **Cache hit rate**: Color-coded health indicators per model
- **Sub-agent detection**: Badges distinguishing main-agent vs sub-agent calls
- **Time filtering**: 6h, 12h, 24h, 48h, 72h, 7 days, custom date range

**Privacy**: Reads only local debug logs — nothing leaves your machine.

**VS Code requirement**: 1.119+

**Best for**: Retrospective analysis — seeing what you actually spent after a session.

### 2.3 Visual Comparison

```
                     COPILOT TOKEN AWARENESS     COST & TOKEN TRACKER
                     ───────────────────────     ────────────────────
  Timing             Before you send              After the session
  Data source        Estimated (reconstructed)    Actual (debug logs)
  UI                 Status bar + panel           Dashboard webview
  Per-turn cost      Yes                          Yes (aggregated)
  Cache detection    Disclaimer only              Actual cache hit %
  Sub-agent tracking No                           Yes
  Best for           "How much will this cost?"   "What did that cost?"
```

---

## 3. Open-Source CLI & MCP Tools

### 3.1 rtk (Rust Token Killer)

| Attribute | Detail |
|---|---|
| Repository | [github.com/rtk-ai/rtk](https://github.com/rtk-ai/rtk) |
| Stars | 71,900+ |
| License | Apache 2.0 |
| Language | Rust |
| Install | `brew install rtk` or `curl -fsSL .../install.sh \| sh` |

**What it does**: CLI proxy that intercepts shell commands (git, grep, test runners,
build tools) and compresses their output before it reaches the LLM context window.

**Token savings** (30-min session):

| Command | Calls | Without RTK | With RTK | Savings |
|---|---|---|---|---|
| ls / tree | 10x | 2,000 | 400 | -80% |
| cat / read | 20x | 40,000 | 12,000 | -70% |
| grep / rg | 8x | 16,000 | 3,200 | -80% |
| git status | 10x | 3,000 | 600 | -80% |
| git diff | 5x | 10,000 | 2,500 | -75% |
| cargo/npm test | 5x | 25,000 | 2,500 | -90% |
| **Total** | | **~118,000** | **~23,900** | **-80%** |

**How it works**:
1. **Auto-rewrite hook**: Intercepts Bash commands transparently — `git status` becomes
   `rtk git status` before execution
2. **Four strategies**: Smart filtering, grouping, truncation, deduplication
3. **Tee on failure**: Saves full raw output when a command fails so the LLM can
   still read it

**Supported commands**: 100+ including git, grep, find, docker, kubectl, npm, cargo,
pytest, jest, vitest, playwright, aws, gh, pulumi, and more.

**Install for VS Code Copilot**:
```bash
brew install rtk
rtk init -g --copilot    # installs PreToolUse hook
```
Then restart VS Code. The hook transparently rewrites shell commands.

### 3.2 Context Mode (MCP Server)

| Attribute | Detail |
|---|---|
| Repository | [github.com/mksglu/context-mode](https://github.com/mksglu/context-mode) |
| Stars | 19,100+ |
| License | Elastic License 2.0 |
| Language | TypeScript |
| Install | `npm install -g context-mode` |

**What it does**: MCP server that sandboxes tool output. Large responses get
processed in an isolated subprocess — only summaries enter the context window.
Also provides session continuity (survives compaction).

**Key tools**:

| Tool | Function | Savings |
|---|---|---|
| `ctx_batch_execute` | Run multiple commands + search in one call | 986 KB → 62 KB |
| `ctx_execute` | Run code in 12 languages, only stdout enters context | 56 KB → 299 B |
| `ctx_execute_file` | Process files in sandbox, raw content never leaves | 45 KB → 155 B |
| `ctx_index` | Chunk markdown into FTS5 knowledge base | 60 KB → 40 B |
| `ctx_search` | Query indexed content on demand | On-demand |
| `ctx_fetch_and_index` | Fetch URL, convert to markdown, index | 60 KB → 40 B |
| `ctx_stats` | Show context savings | — |
| `ctx_doctor` | Diagnose installation | — |

**Benchmarks**:

| Scenario | Raw | With Context Mode | Savings |
|---|---|---|---|
| Playwright snapshot | 56.2 KB | 299 B | 99% |
| GitHub Issues (20) | 58.9 KB | 1.1 KB | 98% |
| Access log (500 req) | 45.1 KB | 155 B | 100% |
| Git log (153 commits) | 11.6 KB | 107 B | 99% |
| Full session | 315 KB | 5.4 KB | 98% |

**Install for VS Code Copilot**:
```bash
npm install -g context-mode
# Add to .vscode/mcp.json and configure hooks
```
See [context-mode.com](https://context-mode.com) for full setup guide.

### 3.3 rtk + Context Mode — The Two-Layer Stack

These two tools are complementary and commonly used together:

```
Layer 1: rtk (CLI output filter)
  ┌─────────────┐     ┌──────────┐     ┌──────────┐
  │ LLM Agent   │────>│ rtk hook │────>│ Shell    │
  │             │<────│ (filter) │<────│ Commands │
  └─────────────┘     └──────────┘     └──────────┘
  Effect: 80% reduction on CLI output noise

Layer 2: Context Mode (MCP sandbox)
  ┌─────────────┐     ┌──────────────┐     ┌────────────┐
  │ LLM Agent   │────>│ ctx_execute  │────>│ Large data │
  │             │<────│ (sandbox)    │<────│ / APIs     │
  └─────────────┘     └──────────────┘     └────────────┘
  Effect: 98% reduction on MCP tool output
```

Together they cover the full token pipeline. Sessions that hit context limits
in 30 minutes can run for hours.

### 3.4 snip (Output Filter)

**What it does**: A smaller, simpler alternative to rtk focused on output truncation.
Filters long command output to keep only relevant lines.

**Typical use**: Piped after shell commands to truncate verbose output before it
reaches context.

**Token impact**: Comparable to rtk for simple truncation needs, less feature-rich
for advanced filtering (grouping, deduplication).

### 3.5 prompt-caching MCP Plugin

| Attribute | Detail |
|---|---|
| Repository | [github.com/flightlesstux/prompt-caching](https://github.com/flightlesstux/prompt-caching) |
| Stars | 130 |
| License | MIT |
| Install | `npm install -g prompt-caching-mcp` |

**What it does**: Automatically injects `cache_control` breakpoints for Anthropic's
prompt caching API. Primarily for developers building their own apps with the
Anthropic SDK, not for Claude Code's built-in sessions (which already cache).

**MCP tools**:
- `optimize_messages` — injects `cache_control` on stable blocks
- `get_cache_stats` — cumulative savings for the session
- `analyze_cacheability` — dry-run analysis without modifying anything

**Best for**: Developers writing custom agents or apps that call the Anthropic API
directly. Claude Code/VS Code Copilot users already get caching automatically.

### 3.6 Graphify (Persistent Codebase Graph)

| Attribute | Detail |
|---|---|
| Repository | [graphify](https://github.com/.../graphify) |
| Install | `uv tool install graphifyy` |

**What it does**: Builds a persistent codebase graph using tree-sitter AST once.
Agents query `graph.json` instead of re-reading project files each session.

**Best for**: Large repos where orientation reads dominate agent input. Front-loads
codebase understanding once, then reuses it across sessions.

**Token impact**: Eliminates repeated file-read input tokens for structural understanding.

### 3.7 Copilot CodeAct Plugin (CLI)

**What it does**: Collapses multi-step tool chains into one sandboxed execution.
Reduces repeated replay of system prompt, prior messages, and tool definitions
across multiple tool calls.

**Availability**: Copilot CLI only (optional external plugin).

**Token impact**: Reduces tool-loop replay overhead in long agentic chains.

---

## 4. Community Workflows & Patterns

### 4.1 "Plan First, Execute Cheaply"

The most commonly recommended workflow pattern:

```
Step 1: PLAN (strong model, Ask mode)
  - Use Sonnet/Opus to design the approach
  - Save to plan.md or a GitHub Issue

Step 2: EXECUTE (cheaper model, fresh session)
  - Use Haiku/GPT-5 mini with the saved plan
  - Clean cache, no history bloat
  - Run to completion

Step 3: VERIFY (optional, targeted)
  - Use Ask mode to review specific parts
  - No agent overhead
```

**Why it works**: Reaching the right outcome on the first try avoids expensive
rework. The plan stage uses a strong model but is short. The execution stage runs
on a cheaper model with zero history overhead.

### 4.2 Scoped Custom Agents / Skills

Instead of one broad agent with every tool:

- **Refactor Agent**: Only code manipulation tools, cheap model
- **Review Agent**: Read-only tools, review-focused instructions
- **Test Agent**: Test-running tools, failure analysis
- **Docs Agent**: Write-only, documentation-focused

Each agent has only the MCP tools it needs — removing irrelevant tool schema
overhead (100-500 tokens per tool per step).

### 4.3 "Landmines Only" Instructions

The community consensus on `copilot-instructions.md` and `AGENTS.md`:

- **Don't**: Write verbose instructions about how to work
- **Do**: Document what NOT to do — the landmines
- **Why**: LLMs already know how to code. Instructions should prevent mistakes,
  not explain basics. Every token in instructions is billed on every request.

**Example**:
```markdown
# BAD — 200 tokens of filler
Always write clean, well-documented code with proper error handling.
Consider edge cases and follow best practices.

# GOOD — 30 tokens, landmines only
- Never use `any` type
- Never commit debug logs or console.log
- Prefer immutable patterns
```

### 4.4 Ask Mode for Simple Questions

The single easiest optimization with zero setup:

- **Ask mode**: ~5K system prompt, no tool overhead, no file gathering
- **Agent mode**: ~14K system prompt, full tool definitions, context gathering

**Savings**: 60-90% for simple Q&A tasks.

**Rule of thumb**: "Can a junior developer answer this in 30 seconds?" → Use Ask mode.
"Does this need file edits or multi-step reasoning?" → Use Agent mode.

### 4.5 Model-Specific Prompt Tuning

Each model responds differently to the same prompt. Community practice:

- When switching to a new model, paste the official provider prompting guide
  into Copilot and ask it to adapt your `.github/copilot-instructions.md`
- Better first-pass output = fewer clarification turns = fewer tokens

### 4.6 Plan First, Then Execute via Markdown

A specific workflow pattern gaining traction:

1. Write the plan as `plan.md` with clear acceptance criteria
2. Start a fresh Copilot session
3. Attach `plan.md` as context
4. Use Auto model selection for execution
5. Verify against acceptance criteria from the plan

This avoids mid-session context bloat and keeps cache hit rates high.

### 4.7 The `/chronicle` Workflow (Copilot CLI)

**Available in**: Copilot CLI (not VS Code Copilot Chat)

**Commands**:
- `/chronicle cost tips` — Analyzes your token spend from session logs, suggests
  specific reductions
- `/chronicle improve` — Finds recurring confusion patterns in your session history,
  generates custom-instruction fixes

**Best for**: CLI power users who want data-driven optimization feedback.

### 4.8 Startup Routine Checklist

Community-recommended daily startup:

1. `/clear` the previous session
2. Close irrelevant editor tabs (reduce context noise)
3. Set Auto model selection
4. Add `applyTo:` paths to scope instructions (if using custom agents)
5. Plan the task verbally in Ask mode first, then switch to Agent for execution

---

## 5. Comparison Matrix

| Tool / Feature | Type | What It Optimizes | Typical Savings | Setup Time |
|---|---|---|---|---|
| **Built-in: Tool Search** | VS Code built-in | Input (tool defs) | 9-18% total tokens | 0 min |
| **Built-in: Auto Model** | VS Code built-in | Blended price rate | 10-40% cost (est.) | 0 min |
| **Built-in: WebSockets** | VS Code built-in | Latency, per-step overhead | 12-19% faster | 0 min |
| **Built-in: Extended Cache** | VS Code built-in | Cache hit rate | +10-919% cache hits | 0 min |
| **Built-in: Ask vs Agent** | Usage pattern | Input + tools | 60-90% per task | 0 min |
| **Built-in: Fresh sessions** | Usage pattern | Input (history) | 10-40% per session | 0 min |
| **Extension: Token Awareness** | VS Code extension | Awareness | N/A (visibility) | 1 min |
| **Extension: Cost Tracker** | VS Code extension | Awareness | N/A (visibility) | 1 min |
| **rtk** | CLI proxy | Input (tool output) | 60-90% on CLI output | 2 min |
| **Context Mode** | MCP server | Input (MCP output) | 94-99% on tool output | 5 min |
| **"Code only" instructions** | Config | Output (response) | 40-70% on code tasks | 0 min |
| **Shrink instructions** | Config | Input (always-on) | 20-23% agent tasks | 15 min |
| **MCP audit** | Config | Input (tool schemas) | Up to 62% on agent tasks | 10 min |
| **Plan-first workflow** | Process | Output (reworks) | Varies (avoids misses) | 0 min |
| **Graphify** | CLI tool | Input (file reads) | Varies (large repos) | 5 min |
| **Content exclusions** | Enterprise config | Input (noise files) | Varies | 5 min |

---

## 6. Quick-Start Recommendations

### For Maximum Impact in Minimal Time

| Priority | Action | Time | Token Savings |
|---|---|---|---|
| 1 | Add `Code only, no explanation.` to instructions | 0 min | 40-70% output |
| 2 | Use Ask mode for simple questions | 0 min | 60-90% per task |
| 3 | Enable Auto model selection | 0 min | Automatic |
| 4 | Start fresh chats per task | 0 min | 10-40% input |
| 5 | Install Copilot Token Awareness | 1 min | Visibility |

### For Power Users

| Priority | Action | Time | Token Savings |
|---|---|---|---|
| 6 | Install rtk (`brew install rtk && rtk init -g`) | 2 min | 80% CLI output |
| 7 | Audit & prune MCP servers | 10 min | Up to 62% agent |
| 8 | Shrink instructions to "landmines only" | 15 min | 20-23% agent |
| 9 | Install Context Mode MCP server | 5 min | 98% tool output |
| 10 | Use plan-first workflow | — | Avoids rework |

### Enterprise

| Priority | Action | Time | Token Savings |
|---|---|---|---|
| 11 | Set content exclusions (node_modules, etc.) | 5 min | Varies |
| 12 | Enforce Auto model selection as org default | 15 min | Organization-wide |
| 13 | Monitor usage dashboard weekly | — | Governance |
| 14 | Create task-specific custom agents | 30 min | 10-30% per workflow |

---

## Source References

| Source | URL |
|---|---|
| VS Code Blog: Improving Token Efficiency | https://code.visualstudio.com/blogs/2026/06/17/improving-token-efficiency-in-github-copilot |
| GitHub Docs: Models and Pricing | https://docs.github.com/copilot/reference/copilot-billing/models-and-pricing |
| GitHub Docs: Optimize AI Usage | https://docs.github.com/en/copilot/tutorials/optimize-ai-usage |
| Ken Muse: Decoding Copilot Token Costs | https://www.kenmuse.com/blog/decoding-copilot-token-costs-using-vs-code |
| olivomarco/token-optimization guide | https://github.com/olivomarco/github-copilot-token-optimization |
| rtk (Rust Token Killer) | https://github.com/rtk-ai/rtk |
| Context Mode MCP | https://github.com/mksglu/context-mode |
| Copilot Token Awareness (extension) | https://marketplace.visualstudio.com/items?itemName=RajUppadhyay.copilot-token-awareness |
| Copilot Cost & Token Tracker (extension) | https://marketplace.visualstudio.com/items?itemName=netcomlabs-ai.copilot-cost-token-tracker |
| prompt-caching MCP plugin | https://github.com/flightlesstux/prompt-caching |

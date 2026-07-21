# Research: VS Code Copilot Token Cost — Understanding & Optimization

*Research compiled July 2026. Sources: GitHub Docs, VS Code Blog, Ken Muse blog, Microsoft Community Hub, olivomarco/github-copilot-token-optimization, various YouTube deep dives.*

---

## Part 1: The Cost Formula — How Token Pricing Works

### Core Equation

```
Cost = (Input_Tokens × Input_Price) + (Output_Tokens × Output_Price)
     = (Fresh_Input × Input_Price) + (Cache_Read × CacheRead_Price) + (Cache_Write × CacheWrite_Price) + (Output_Tokens × Output_Price)
```

GitHub Copilot uses **AI Credits (AIC)** where **1 AIC = $0.01 USD**. Each AIC splits into 1,000,000,000 nano-AIU for internal precision.

### Key Pricing Asymmetry

**Output tokens cost 5x more than input tokens.** This is the single most important pricing fact. Examples (Anthropic models):

| Model | Input $/MTok | Output $/MTok | Ratio |
|---|---|---|---|
| Haiku 4.5 | $1.00 | $5.00 | 1:5 |
| Sonnet 4.6 | $3.00 | $15.00 | 1:5 |
| Sonnet 5 | $2.00 (intro) | $10.00 (intro) | 1:5 |
| Opus 4.8 | $5.00 | $25.00 | 1:5 |

OpenAI models in Copilot follow a similar pattern (e.g., GPT-5.4: $2.50/$15.00).

### Cache Pricing Multipliers

| Cache Operation | Multiplier vs Base Input | Example (Sonnet 4.6) |
|---|---|---|
| Fresh input | 1x | $3.00/MTok |
| Cache write (5-min) | 1.25x | $3.75/MTok |
| Cache write (1-hour) | 2x | $6.00/MTok |
| Cache read (hit) | 0.1x | $0.30/MTok — **90% savings** |

### Where to Find Cost Data

VS Code records every model call in **Chat Debug View** (overflow menu → "Show Chat Debug View"). Each entry shows:
- `model` e.g. `claude-sonnet-4-6`
- `usage.prompt_tokens` / `usage.completion_tokens`
- `copilotUsage` — final AIC cost (e.g., `1.94 AIC`)
- Token breakdown: fresh input, cache_read, cache_write, output

---

## Part 2: Optimization Strategies — Reducing Token In/Out

### Strategy A: Reduce Input Tokens

#### A1. Shrink Always-On Context (copilot-instructions.md + AGENTS.md)

- Strip filler language, LLM-generated boilerplate, and verbose instructions
- Apply "landmines only" approach — document what NOT to do, not how to work
- GitHub's own data: well-scoped instructions cut 20-23% agent-task tokens
- Every word in instructions is sent with every request — compounds across agent steps

#### A2. Use Ask Mode for Simple Questions

- Ask mode has ~5K system prompt budget vs Agent mode ~14K
- No tool overhead, no file context gathering
- **60-90% savings** vs Agent mode for simple Q&A

#### A3. Start Fresh Conversations (/new, /clear)

- Long conversations carry full history into every request
- `/clear` in CLI, new chat in Copilot Chat — resets cache boundary too
- Compact long sessions you want to continue (reduces ~10-40% tokens)

#### A4. Scope Context Tightly — No Blind Repository Scans

- Mention specific files/folders — avoid fetching entire repo
- Use file ranges and globs instead of open tabs
- Exclude noise folders like `node_modules`, `dist/`, `build/` via content exclusions

#### A5. Reduce Tool Output Noise

- **The biggest silent cost**: `git status`, `grep`, test runs, build logs dump thousands of tokens into context
- **rtk** — CLI output compressor: intercepts git/grep/find/docker/npm/pytest output, 89% average noise reduction
- **Context Mode (MCP)** — sandboxes tool output: indexes large responses into FTS5 knowledge base, only summaries enter context, **98% reduction on big outputs**
- Together they cover the full pipeline: CLI output + MCP tool output

#### A6. Convert Rich Files to Markdown Before AI Work

- `.docx`, `.pdf`, `.pptx`, `.xlsx`, HTML, images, audio, video all carry format tax
- Use Microsoft MarkItDown before chat, agent, or RAG ingestion
- Reduces noisy input context from binary/rich formats

#### A7. Limit Open Tabs

- Each open editor tab can be read as context
- Close irrelevant tabs before starting a task

### Strategy B: Reduce Output Tokens

#### B1. Output Control — "Code Only, No Explanation"

- Add to `copilot-instructions.md`:
  ```
  - Answer concisely. No pleasantries, no filler, no explanations.
  - For code: output only the diff/changed lines. No summaries.
  - Be terse. Prefer single-line answers.
  ```
- **40-70% output savings on code tasks**, 30-60% across all interactions
- One instruction, permanent effect

#### B2. Caveman / Terse Style Instructions

- The "Caveman" skill: strip all filler, pleasantries, and verbose explanations
- "Treat every token like it costs money — because it does"
- Clear intent, no "please", no "thank you", no recaps

#### B3. Reduce Thinking / Reasoning Tokens

- Configurable reasoning levels on supported models
- Use regular reasoning by default, raise only for hard problems
- Overuse of reasoning models makes them overthink — more thinking tokens = more cost
- Opus 4.8 has "Fast Mode" at 3x cheaper ($10/$50 vs $5/$25)

#### B4. Collapse Tool Calls — Batch Instead of Sequence

- Each agent step = separate API request with its own context
- Collapsing multiple tool calls into one reduces turns
- Batching reduces token overhead across long chains of tool calling

### Strategy C: Reduce Tool Calling Overhead

#### C1. Audit MCP Servers and Injected Tools

- Each MCP tool costs ~100-500 tokens per agent step
- 15 MCP servers × 15 agent steps = **~265K tokens of overhead** per task
- Disable unused servers/extensions; use a clean coding profile for repeat workflows
- GitHub internal: pruning unused MCP tools achieved **up to 62% token cost reduction**

#### C2. Tool Search (VS Code 1.118+)

- On-demand tool loading: model sees only lightweight metadata upfront
- Heavier parameter schemas stay out of context until model searches for a tool
- Cache prefix stays reusable
- Median user: 8.97-10.92% total token reduction

#### C3. Use CLI Instead of MCP for Deterministic Operations

- "Build the app deterministically, find the error deterministically"
- Use `gh` CLI or shell scripts for deterministic data gathering
- Send only the error stream, not the full build log
- **20-70% token savings** on troubleshooting workflows

#### C4. Summarize Logs and Test Outputs Before Sending

- Don't dump raw logs into context
- Extract just the error lines with grep (e.g., `grep -i error build.log`)
- Use `git --no-pager diff --stat` instead of full diffs

### Strategy D: Model Selection for Cost Efficiency

#### D1. Right-Size the Model to the Task

| Task Type | Recommended Model Tier |
|---|---|
| Architecture, complex debugging, system design | Reasoning (Sonnet, Opus) |
| Execution with clear plan | Mid-tier (Sonnet, GPT-5.4) |
| Refactoring, formatting, docs, routine changes | Lightweight (Haiku, GPT-5 mini/nano) |

- Matching capability to task improves outcomes AND controls costs
- Overusing reasoning models can REDUCE quality (overthinking)

#### D2. Auto Model Selection

- Small router sends prompt to the model that can handle it most efficiently
- Reserves expensive reasoning models for complex problems
- Only changes models at natural cache boundaries (new session or /compact)
- **10% discount** on paid Copilot plans when using auto model selection
- Routes around degraded/busy models — fewer rate limits and errors

#### D3. Task-Specific Custom Agents

- Instead of one broad agent with every tool, create focused agents:
  - "Review Agent" — has only review tools
  - "Refactor Agent" — has only code manipulation tools
  - "Test Agent" — has only test-running tools
- Removes irrelevant tool schema overhead
- **10-30% token savings** per workflow

---

## Part 3: Prompt Caching — The 10x Multiplier

### How It Works

The inference provider caches K/V state from the **prompt prefix** (system instructions, tool definitions, conversation history that repeats across turns). When the next request shares the exact same prefix, the cached state is reused instead of recomputed.

**Important**: The cache stores model state (key/value tensors), not human-readable text.

### Cache Hit Rates

- VS Code 1.118 (April 2026): **over 93% cache reuse** through strategic breakpoint placement
- Breakpoints at: system prompt boundary, tool list boundary, last tool turn
- For agentic workloads: **~94% cache hit rate**
- One reported session: 8M tokens, 94% cache rate = ~$6.90 on GPT-5.5 (vs ~$70 without caching)

### Cache Economics — The Numbers

| Metric | Without Cache | With Cache (90% hit) |
|---|---|---|
| Input cost per MTok | $3.00 (Sonnet) | $0.30 (cached) |
| Session: 8M tokens | ~$70 | ~$6.90 |
| Savings | — | **~90%** |

### Rules of Caching

1. **Cache belongs to a specific model** — switch models, rebuild cache (costs more on first request)
2. **Cache creation costs 1.25x** normal input (the write cost)
3. **Cache reads cost 0.1x** — 90% savings
4. **Break-even at turn 2** — every turn after that is pure savings
5. **5-min TTL** default, **1-hour** available at 2x write cost

### Cache Killers

- Switching models mid-session (resets cache)
- Long idle gaps (>5 min between requests)
- Changing system prompt or attached context
- `/compact` — information loss AND may affect cache

### Real-World Savings Reported

| Session Type | Turns | Without Caching | With Caching | Savings |
|---|---|---|---|---|
| Bug fix (single file) | 20 | 184K tokens | 28.4K tokens | 85% |
| Refactor (5 files) | 15 | 310K tokens | 61.2K tokens | 80% |
| General coding | 40 | 890K tokens | 71.2K tokens | 92% |
| Repeated file reads | — | 50K tokens | 5.1K tokens | 90% |

---

## Part 4: Putting It All Together — Cost Comparison Scenarios

### Scenario: Typical Agent Session (Before vs After Optimization)

| Factor | Unoptimized | Optimized |
|---|---|---|
| Model | Sonnet 4.6 always | Auto select (mostly Haiku/Sonnet) |
| Instructions | 2K tokens, verbose | 300 tokens, "landmines only" |
| MCP servers | 15 active | 4 active (scoped to task) |
| Tool output | Raw dumps | Context Mode / rtk filtered |
| Response style | Verbose explanations | Code-only, terse |
| Cache hit rate | ~70% | ~94% |
| Session tokens (est.) | ~150K | ~25-40K |
| Estimated cost | ~$1.50-2.50 | ~$0.20-0.50 |

---

## Part 5: Narrative Arc for Presentation

### Act 1: The Reality Check
- Show real debug-log cost data in VS Code
- The formula: `Cost = Tokens × Price`
- The asymmetry: output is 5x input
- Demo: one agent session can cost $7-8 in credits

### Act 2: The Levers — Where Tokens Actually Go
- **Input**: system prompt, tools, file context, history, cached content
- **Output**: model response + thinking tokens
- Visualize the breakdown with real logs

### Act 3: Optimization Playbook
1. **Output First**: terse instructions, code-only diffs (highest ROI — output is 5x)
2. **Tool Hygiene**: audit MCP, use tool search, limit per-agent tools
3. **Context Budget**: scope files, use Ask mode, fresh sessions
4. **Sandbox Noise**: rtk + Context Mode for output compression
5. **Model Right-Sizing**: auto selection, task-specific agents

### Act 4: Prompt Casing Deep Dive
- How it works (K/V cache)
- The 94% hit rate in VS Code 1.118
- Economics: 10x cheaper cached reads
- Rules: model affinity, TTL, breakpoints
- Cache killers to avoid

### Act 5: The Cost Calculator — Live Estimation
- Walk through a real session cost breakdown
- Compare: unoptimized vs optimized
- "Before optimization: $70/session → After: $6.90"
- ROI of spending 30 min on optimization

### Closing: Cost Optimization = Engineering Maturity
- "It's not about using AI less — it's about using AI better"
- Smaller context = more accurate model
- Cost optimization IS quality optimization
- Call to action: audit your MCP servers + add terse instructions today

---

## Part 6: Additional Optimization Methods (Deep Cuts)

*This section covers techniques found through extended research that go beyond the
standard playbook. These are less commonly discussed but can be high-impact in the right context.*

### 6.1 Prompt Compression — Caveman Intensity Levels

Not all compression is equal. Three distinct levels for different contexts:

| Level | Style | Input Savings | Output Savings | Risk |
|---|---|---|---|---|
| **Lite** | Professional but tight. Remove filler/hedging. Keep sentences. | 15-25% | 15-25% | None |
| **Full** | Classic caveman. Drop articles, fragments OK, short synonyms. | 30-50% | 40-55% | Negligible |
| **Ultra** | Max compression. Abbreviate common terms, arrows for causality. | 55-70% | 55-70% | Possible ambiguity in complex instructions |

**Examples**:

| Verbose | Caveman Ultra |
|---|---|
| "Can you explain what this error means and how to fix it?" (~16 tok) | "Explain error. How fix." (~5 tok) |
| "I'd like you to add comprehensive error handling to this API endpoint" (~22 tok) | "Add error handling to endpoint. Validate request body." (~10 tok) |
| "Could you please review this PR and let me know if there are issues?" (~17 tok) | "Review PR. Flag issues." (~5 tok) |

**Key insight**: Input savings come from terse prompts. Output savings require setting terse
output in system instructions. Writing a terse prompt does NOT make the model respond tersely — you need both.

### 6.2 Structured Format Over Prose

Bullets and key-value pairs beat paragraphs every time:

**Prose** (~55 tokens):
```
I need you to create a REST API endpoint that accepts POST requests at /api/users.
It should validate that the request body contains a name field (string, required)
and an email field (string, required, must be valid email format). If validation
fails, return a 400 status with error details. On success, save to the database
and return 201 with the created user object.
```

**Structured** (~35 tokens, **36% savings**):
```
POST /api/users
Validate:
- name: string, required
- email: string, required, valid format
400 on validation fail (include errors)
201 on success (return created user)
Save to DB
```

### 6.3 Code-Centric Prompting

Sometimes code is more token-efficient than natural language:

| Format | Tokens | vs NL |
|---|---|---|
| Natural language "Create a function that filters negatives, doubles, sums" | ~30 | 1x (baseline) |
| Pseudocode `fn(nums) → filter(>0) → map(*2) → sum` | ~15 | **50% savings** |
| Type signature `def process(nums: list[int]) -> int: # filter, double, sum` | ~12 | **60% savings** |
| "Like X but Y" pattern `Like getUserById but for emails. Return 404 if missing.` | ~10 | **67% savings** |

### 6.4 Declarative Guardrails Over Imperative Instructions

Imperative tells step-by-step. Declarative states what must be true. Declarative
is shorter, more stable, and composes better:

| Imperative (wordy) | Declarative (tight) |
|---|---|
| "First read the file, then find all public functions, then check each for JSDoc..." | "All exported functions: JSDoc required." |
| "Make sure you always parameterize SQL values to prevent injection..." | "SQL: parameterized queries only. No concatenation." |
| "Please write tests covering both happy path and error cases for new code..." | "New code → tests. Cover happy + error paths." |

### 6.5 Language Choice — English Is Most Token-Efficient

BPE tokenizers are trained on majority-English data. **English gets the best compression.**

| Language | Token Cost vs English |
|---|---|
| English | 1.0x (baseline) |
| Spanish | ~1.3-1.6x |
| German | ~1.4-1.6x |
| Mandarin Chinese | ~1.76x |
| Japanese | ~2.12x |
| Korean | ~2.36x |
| Russian (Cyrillic) | ~2.5-2.8x |
| Hebrew | ~3.2x |

**Counterintuitive finding**: Chinese uses fewer *characters* (8 vs 16 for English) but costs more *tokens* (11 vs 5).
Don't write prompts in CJK hoping to save tokens — you'll spend more.
English is 1.5-3x more efficient than any other language for prompts.

**Transliteration helps**: Russian in Latin characters: 11 tokens vs 14 in Cyrillic (21% cheaper).
Hebrew transliterated: 9 tokens vs 16 native (44% cheaper).

### 6.6 Workflow-Specific Optimization

**Commit messages** — Terse Conventional Commits. Compounds across repo history:
- Verbose: `feat: Added a new feature to allow users to reset their passwords...` (~25 tok)
- Terse: `feat: add password reset via settings page` (~10 tok, **60% savings**)

**PR reviews** — One-line format with severity prefixes:
- Verbose: "I noticed on line 42, the user variable could potentially be null..." (~40 tok)
- Terse: `L42: 🔴 bug: user can be null here. Add null guard before .email access.` (~12 tok, **70% savings**)
- Prefixes: 🔴 must-fix / 🟡 should-fix / 🔵 optional / ❓ question

**Retune prompts per model** — Different models need different prompt styles:
- Claude: XML-ish sections, role definition, explicit success criteria
- GPT-5.5: Outcome-oriented, concise, efficient path
- Gemini: Structured task/input/constraints/response-format
- Paste provider guide into Copilot → ask it to adapt your instructions

### 6.7 Built-In Feedback Loops — `/chronicle` and AI Engineering Coach

**Copilot CLI — `/chronicle`** (experimental):
- `/chronicle cost tips` — analyzes token spend across sessions, suggests reductions
- `/chronicle improve` — scans history for recurring confusion, generates instruction fixes
- `/chronicle tips` — personalized coaching based on your actual usage patterns
- Run weekly, data lives locally in `~/.copilot/session-state/`

**VS Code — AI Engineering Coach** (open-source extension):
- Reads VS Code AI session logs locally — nothing leaves your machine
- **Anti-Patterns**: 45 editable rules across prompt quality, session hygiene, code review
- **Context Health**: Agentic readiness checklist, workspace context map
- **Skill Finder**: Matches repeated prompt patterns to reusable skills
- **Output/Burndown**: Code volume by language/model, token budget projections

**GitHub: `git clone https://github.com/microsoft/ai-engineering-coach.git`**

### 6.8 When NOT to Compress

Compression has limits. These situations demand full clarity:
- **Security warnings** — "This will delete all user data" must NOT be "del usr data"
- **Irreversible operations** — Confirmation prompts must be unambiguous
- **Onboarding contexts** — New team members need the "why", not just the "what"
- **Complex multi-step instructions** — Fragment order could cause misreading
- **Regulatory/compliance text** — Legal requirements demand precision

**Rule of thumb**: A well-designed terse prompt can handle this automatically —
dropping terse mode for security/irreversible sections, then resuming.

### 6.9 The "Scope, Don't Pile On" Principle

A common failure mode: when the model gets something wrong, instinct is to add
more instructions. This usually makes things worse — and always makes things more expensive.

| Problem | Typical Reaction (Wrong) | Better Reaction (Right) |
|---|---|---|
| Model misses an edge case | Add 50-token rule about it to instructions (always-on) | Mention it in this prompt (one-shot) |
| Model produces verbose output | Add "be concise" five different ways | Constrain format: "1-line answer." |
| Agent goes off on a tangent | Add more global guardrails | Tighten scope: name the file, function, done-condition |

**Smaller scoped prompts reduce runaway sessions.** A 10-token "Fix null deref in getUser() L42"
rarely produces a 30-step exploration. A 200-token "please look at authentication
and improve robustness" almost always does.

### 6.10 Abbreviations and Shorthand

Common abbreviations that models understand perfectly (save 1-2 tokens each):

| Shorthand | Full | Token Savings |
|---|---|---|
| DB | Database | ~50% |
| auth | Authentication/authorization | ~60% |
| config | Configuration | ~50% |
| req/res | Request/response | ~50% |
| fn | Function | ~50% |
| impl | Implementation | ~60% |
| env | Environment | ~50% |
| deps | Dependencies | ~60% |
| repo | Repository | ~50% |
| PR | Pull request | ~60% |
| e2e | End-to-end | ~60% |

**When abbreviations help**: Repeated terms in long instructions. "Database" = 2-3 tokens, "DB" = 1.
**When they hurt**: Novel abbreviations the model hasn't seen. Define project-specific shorthand
in `copilot-instructions.md` first.

### 6.11 Outcome per Token — The Meta-Optimization

Research from Microsoft/Stanford ("How Do AI Agents Spend Your Money?"):

| Finding | Implication |
|---|---|
| Agentic coding consumes ~1,000x more tokens than chat | Don't extrapolate chat-cost intuition |
| Same task varies up to 30x across runs | Budget with margin |
| Higher token usage does NOT reliably improve accuracy | More exploration ≠ better work |
| Accuracy peaks at intermediate cost, then saturates | Defaulting to biggest model can waste money |
| Input tokens dominate agentic cost | Context hygiene > output terseness |

**The real metric**: `outcome per token = verified work completed / total tokens spent`

**The six habits for high outcome per token**:
1. Pick the right task shape
2. Plan before implementation
3. Route the right model to the right phase
4. Preserve clean context and cache boundaries
5. Verify before claiming done
6. Close cleanly (no stale context leaking to next task)

### 6.12 Understanding the Harness — How Context Is Built Per Round

VS Code's agent loop: "think → act → observe → think again" per round.

**Key insight**: On every round, the harness rebuilds the entire prompt from scratch:
system prompt + conversation history + all tool results so far.

This means:
- **Every round pays full input cost** for system prompt and accumulated history
- **Cache is your only defense** against paying full price for the prefix each round
- **Longer tool-calling chains** = more rounds = more input cost per turn
- **Tool output noise compounds** across every subsequent round

**Agent loop anatomy** (single user turn → many rounds):
```
User message
  ↓
Round 1: Build prompt → Model → Tool calls → Execute → Capture results
Round 2: Build prompt again (with round 1 results) → Model → More tools → Execute
Round 3: Build prompt again (with all prior results) → Model → Final response
  ↓
Assistant message
```

Each round is a full API call with the complete accumulated context.
**Reducing rounds = reducing input token cost linearly.**

### 6.13 VSC-Bench — The Diminishing Returns Curve

VS Code's internal benchmark (VSC-Bench) shows a clear pattern:

- For many tasks, **"high" reasoning effort hits the sweet spot**
- **"xhigh" uses more tokens but resolves slightly fewer tasks** — past the useful effort ceiling
- More thinking does not always convert into better outcomes
- The optimal point is *intermediate cost*, not maximum cost

**Practical takeaway**: If you're always using the highest reasoning effort,
you're likely past the point of diminishing returns. Drop one level and measure
whether quality actually degrades.

### 6.14 Skill Libraries — Borrowing Community Workflows

Community skill libraries encode repeatable workflows that improve outcome per token:

| Library | Best For | Why It Saves Tokens |
|---|---|---|
| obra/superpowers | TDD, planning, verification, branch finish | Prevents wrong-direction code |
| catpilot-ai-guardrails | Security & tool-loop guardrails | Prevents expensive mistakes |
| vercel-labs/agent-browser | Browser QA (compact observations, no DOM floods) | Avoids dumping huge HTML |
| vercel-labs/writing-guidelines | Plan-as-prompt, output review | Reduces rework from vague specs |
| mattpocock/skills | TDD, bug diagnosis, code review, domain modeling | Sharper engineering loops |
| softaworks/agent-toolkit | Planning orchestration, handoff, entropy reduction | Cleaner long sessions |

**Warning**: Skills are context too. Install only the ones that change the next agent
action. If a skill is not security-heavy, browser-heavy, QA-heavy, or handoff-heavy,
more skill loading is probably context tax.

---

## Sources

| Source | URL |
|---|---|
| GitHub Docs: Models & Pricing | https://docs.github.com/copilot/reference/copilot-billing/models-and-pricing |
| GitHub Docs: Optimize AI Usage | https://docs.github.com/en/copilot/tutorials/optimize-ai-usage |
| VS Code Blog: Token Efficiency | https://code.visualstudio.com/blogs/2026/06/17/improving-token-efficiency-in-github-copilot |
| Ken Muse: Decoding Copilot Token Costs | https://www.kenmuse.com/blog/decoding-copilot-token-costs-using-vs-code |
| olivomarco/optimization guide | https://github.com/olivomarco/github-copilot-token-optimization |
| Microsoft Community Hub | https://techcommunity.microsoft.com/blog/azuredevcommunityblog/optimizing-github-copilot-cost-in-the-usage-based-billing-era/4534186 |
| Context Mode (MCP) | https://github.com/mksglu/context-mode |
| rtk CLI compressor | https://github.com/rtk-ai/rtk |
| MS/Stanford: "How Do AI Agents Spend Your Money?" | https://arxiv.org/abs/2604.22750 |
| VS Code Blog: The Coding Harness | https://code.visualstudio.com/blogs/2026/05/15/agent-harnesses-github-copilot-vscode |
| VS Code Docs: Planning with Copilot | https://code.visualstudio.com/docs/agents/planning |
| AI Engineering Coach | https://github.com/microsoft/AI-Engineering-Coach |
| obra/superpowers (skills) | https://github.com/obra/superpowers |
| copilot-codeact-plugin | https://github.com/jsturtevant/copilot-codeact-plugin |
| prompt-caching MCP | https://github.com/flightlesstux/prompt-caching |

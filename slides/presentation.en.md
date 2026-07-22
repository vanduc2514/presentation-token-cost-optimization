<!--markpress-opt
{
  "autoSplit": false,
  "sanitize": false,
  "title": "Token Cost Optimization in AI-Assisted Development"
}
markpress-opt-->

<!--slide-attr x=0 y=0 scale=1.2 -->

# Token Cost Optimization
## Practical strategies for AI-assisted development

Duc Nguyen

<!-- SPEAKER NOTES
Welcome. Today I will walk through how token costs work in AI coding tools
and practical, measurable ways to reduce them. Based on real usage patterns.
-->

------

<!--slide-attr x=1600 y=0 scale=1.0 -->

# The Token Cost Formula

> Cost = Token In x Input Rate + Token Out x Output Rate

- **Token In**: prompt, attached files, tool outputs, conversation history
- **Token Out**: responses, generated code, tool call arguments
- **Input Rate**: $/token for input
- **Output Rate**: $/token for output

<!-- SPEAKER NOTES
The formula has two rate components. Input is cheaper; output is the
dominant cost factor. Thinking tokens are billed at the output rate.
-->


------

<!--slide-attr x=3200 y=0 scale=1.0 -->

# Control Token Volume

> Reduce what enters and exits the context window

- Attach only relevant files, not the entire project
- Start **fresh sessions** for separate tasks
- Request concise output, avoid verbose explanations
- Keep tool output out of context, use sandboxed execution

<!-- SPEAKER NOTES
Context is a budget. Every file attachment, tool output, and conversation
turn costs tokens. Being intentional about context is the highest-leverage
optimization you can make.
-->

------

<!--slide-attr x=0 y=1200 scale=1.0 -->

# Where Tokens Enter Context

| Token In | Token Out |
|---|---|
| Custom Instructions and Skills (AGENTS.md, ...) | Chat Response |
| User query and Images | - |
| Attached files | Generated code |
| Tool call outputs | Tool call arguments |
| Thinking token in | Thinking token out |
| Conversation history | - |

<!-- SPEAKER NOTES
Tool outputs and conversation history are the biggest contributors to
Token In. Each turn adds previous output to the next input. Token Out is
harder to control but smaller in volume -- focus reduction efforts on
Token In first.
-->

------

<!--slide-attr x=1600 y=1200 scale=1.0 -->

# Control Model Price

<table>
<thead>
<tr><th>Small</th><th>Medium</th><th>Large</th></tr>
</thead>
<tbody>
<tr>
<td><img src="./images/GPT-5.4-mini.png" alt="GPT-5.4-mini" class="zoomable-img" style="width: 70%; display: block; margin: 0 auto;"></td>
<td><img src="./images/GPT-5.4.png" alt="GPT-5.4" class="zoomable-img" style="width: 70%; display: block; margin: 0 auto;"></td>
<td><img src="./images/GPT-5.5.png" alt="GPT-5.5" class="zoomable-img" style="width: 70%; display: block; margin: 0 auto;"></td>
</tr>
<tr>
<td><img src="./images/MAI.png" alt="MAI" class="zoomable-img" style="width: 70%; display: block; margin: 0 auto;"></td>
<td><img src="./images/Sonnet 4.6.png" alt="Sonnet 4.6" class="zoomable-img" style="width: 70%; display: block; margin: 0 auto;"></td>
<td><img src="./images/Opus 4.6.png" alt="Opus 4.6" class="zoomable-img" style="width: 70%; display: block; margin: 0 auto;"></td>
</tr>
</tbody>
</table>

<!-- SPEAKER NOTES
Two dominant factors: model tier selection and the input/output rate gap.
A verbose response from a large model is the worst case. Most optimization
effort should target these two.
-->

------

<!--slide-attr x=3200 y=1200 scale=1.0 -->

# Selective Model by Task

> Match the model to the task complexity

| Task | Recommended Model |
|---|---|
| Rename, format, boilerplate | Small |
| Feature implementation, refactoring | Mid |
| Architecture, debugging, design | Large |

<!-- SPEAKER NOTES
Not every code change needs Sonnet. Renaming a variable or adding a type
annotation is perfectly handled by Haiku at a fraction of the cost.
Auto-routing makes this transparent to the developer.
-->

------

<!--slide-attr x=1600 y=2400 scale=1.0 -->

# Leverage Prompt Caching

> Server caches the common prefix of your prompt across requests

![Prompt Caching Diagram](./images/prompt-caching.svg)

<!-- SPEAKER NOTES
The cache operates on prefix matching. Everything before the breakpoint
must be byte-identical. The diagram shows two requests where the system
prompt and AGENTS.md are identical -- they cache hit. The user query
is different, so it is not cached. Keeping your instructions stable
across a session is critical for maximum cache hits.
-->

------

<!--slide-attr x=2400 y=2400 scale=1.0 -->

# Cache Explorer

<img src="./images/prompt-cache-explorer.png" alt="Prompt Cache Explorer" class="zoomable-img" width="85%" style="max-width: 800px; display: block; margin: 1.2rem auto 0;" />

<!-- SPEAKER NOTES
This is the actual cache explorer panel in VS Code. It shows you
real-time statistics on cache hits and misses -- which parts of your
prompt are being cached, how much you are saving, and what causes
cache misses. Use it to validate that your caching strategy is working
and to identify where cache is being invalidated unexpectedly.
-->

------

<!--slide-attr x=3200 y=2400 scale=1.0 -->

# Cache Pricing

> GitHub Copilot charges per 1M tokens. Cached input = 90% off.

| Model | Input | Cached | Output |
|---|---|---|---|
| GPT-5.4 mini | $0.75 | $0.075 | $4.50 |
| Claude Haiku 4.5 | $1.00 | $0.10 | $5.00 |
| GPT-5.4 | $2.50 | $0.25 | $15.00 |
| Claude Sonnet 4 | $3.00 | $0.30 | $15.00 |
| GPT-5.5 | $5.00 | $0.50 | $30.00 |
| Claude Opus 4.5 | $5.00 | $0.50 | $25.00 |

- Output costs 5-6x more than input per token
- Cached input is 90% cheaper than fresh input
- Prices vary by model tier (lightweight vs powerful)

<!-- SPEAKER NOTES
Prices shown are per million tokens from GitHub Copilot's official
pricing page (docs.github.com/copilot/reference/.../models-and-pricing).
The caching discount is consistent across all models: cached input is
10% of the fresh input price. The output/input ratio is about 5-6x.
Source: GitHub Copilot Models and Pricing docs.
-->

------

<!--slide-attr x=6400 y=2400 scale=1.0 -->

# Token Based Optimization

| | Reduce Volume | Reduce Rate |
|---|---|---|
| **Approach** | Fewer tokens in/out | Cheaper per-token price |
| **Methods** | AGENTS.md, Skills, sandbox | Model selection, caching |
| **Impact** | Fewer total tokens | Lower $/token |


------

<!--slide-attr x=6400 y=1200 scale=1.0 -->

# Use AGENTS.md

> Give the model a map, not a maze

- Project-level instructions reduce exploration
- Model reads conventions once instead of re-discovering each turn
- Define: architecture, file layout, coding style, build commands
- Result: fewer tool calls, fewer wrong assumptions, fewer correction rounds

<!-- SPEAKER NOTES
Without AGENTS.md, the model explores your codebase by reading files,
running searches, making wrong guesses. A good AGENTS.md eliminates
that exploration cost by providing the answers upfront.
-->

------

<!--slide-attr x=6400 y=0 scale=1.0 -->

# Sandbox Execution

> Process data outside the context window

- Run analysis in a sandbox: only the summary enters context
- A 700KB log file becomes a 3KB conclusion
- Principle: **compute outside, surface only results**
- Tools: context-mode, RTK, structured I/O pipelines

<!-- SPEAKER NOTES
The core pattern: instead of reading a large file into context and then
analyzing it, run the analysis in a sandbox and print only the answer.
Think-in-Code, not Think-in-Context. This single pattern eliminates
the largest source of token waste.
-->

------

<!--slide-attr x=6400 y=-1200 scale=1.0 -->

# Skills and Output Constraints

> Shape how the model responds

- **Caveman skill**: enforce terse, minimal responses
- **Structured output**: JSON schemas, predictable format
- **Token caps**: limit response length explicitly
- Less verbosity = fewer output tokens

<!-- SPEAKER NOTES
Models default to helpful, thorough explanations. A skill that says
"respond in under 50 characters" or "output valid JSON only" cuts
output tokens significantly. Especially useful for automated pipelines.
-->

------

<!--slide-attr x=6400 y=-2400 scale=1.0 -->

# Custom Agents

> Dedicated models for dedicated tasks

- **Planning**: architecture, design -> large model
- **Implementation**: code generation -> medium model
- **Review**: linting, validation -> small model
- Each agent has its own system prompt and model selection

<!-- SPEAKER NOTES
Do not pay Opus prices for tasks Haiku handles well. Split your workflow:
planning needs reasoning, implementation needs code generation, review
only needs pattern matching. Different capabilities, different models.
-->

------

<!--slide-attr x=4800 y=-2400 scale=1.0 -->

# Prompt Caching Rules

> Keep the cache warm

- **Same model throughout**: switching models invalidates cache
- **Separate sessions per task**: different instructions break the prefix
- **Stable prefix**: put instructions early, keep them unchanged
- **One session, one task**: avoid context drift

<!-- SPEAKER NOTES
The cache is fragile. If you switch from Sonnet to Haiku mid-task, it resets.
If you reuse a session for a different task with different instructions, it
resets. Design your workflow around these constraints to maximize cache hits.
-->

------

<!--slide-attr x=3200 y=-2400 scale=1.0 -->

# TTL and Cache Invalidation

> Know what keeps the cache alive and what kills it

| Action | Effect on Cache |
|---|---|
| Same model, same prefix | Hit (warm) |
| Same session, continuous turns | Hit (refreshed) |
| Switch model mid-session | **Miss** (cold) |
| Change system prompt | **Miss** (cold) |
| Wait > TTL between turns | **Miss** (expired) |
| Switch to different task | **Miss** (prefix mismatch) |

- TTL varies by provider: ~5 min default, up to 1 hour with options
- Every cache hit resets the TTL timer
- Cache miss = full price on next request

<!-- SPEAKER NOTES
TTL is reset on every cache hit. As long as your session is active and
you keep the same model and prefix, the cache stays warm. The most
common mistake: switching models mid-session instantly invalidates
the cache.
-->

------

<!--slide-attr x=1600 y=-2400 scale=1.0 -->

# Monitoring

> You cannot improve what you do not measure

- **VS Code Agent Debug**: built-in per-session token view
- **OpenTelemetry**: enable in VS Code settings for export
- **External collectors**: aggregate across sessions and projects
- **Baseline first**: document costs before applying any optimization

<!-- SPEAKER NOTES
Establish a baseline of your current token usage. Run the same representative
workload before and after each optimization. Quantify the impact. Without
measurement, you are guessing.
-->

------

<!--slide-attr x=0 y=4800 rotate=-3 scale=1.1 -->

# Thank You

Questions?

| GitHub | Website |
|---|---|
| ![GitHub QR](images/github-qr.svg) | ![Website QR](images/nvduc-qr.svg) |
| ![GitHub icon](images/github-icon.svg) | ![Website icon](images/website-icon.svg) |

<!-- SPEAKER NOTES
Thank you. The key takeaway: optimization is about intentionality -- know
where your tokens go, make conscious choices about model, context, and output.
-->

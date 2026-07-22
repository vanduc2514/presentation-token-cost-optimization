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

# Understand Prompt Caching

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

# VSCode Built-in Cache Explorer

<img src="./images/prompt-cache-explorer.png" alt="Prompt Cache Explorer" class="zoomable-img" width="60%" style="max-width: 560px; display: block; margin: 0.6rem auto 0;" />

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

| Model | Cache Miss (In/Out) | Cache (In/Out) |
|---|---|---|
| GPT-5.4 mini | 75 / 450 | <span class="cache-free">7 / 0</span> |
| MAI Code 1 Flash | 75 / 450 | <span class="cache-free">7 / 0</span> |
| GPT-5.4 | 250 / 1500 | <span class="cache-free">25 / 0</span> |
| Claude Sonnet 4.6 | 300 / 1500 | 30 / 375 |
| GPT-5.5 | 500 / 3000 | <span class="cache-free">50 / 0</span> |
| Claude Opus 4.6 | 500 / 2500 | 50 / 625 |

<!-- SPEAKER NOTES
Prices shown are per million tokens from GitHub Copilot's official
pricing page (docs.github.com/copilot/reference/.../models-and-pricing).
1 AI Credit = $0.01 USD. Output from Cache = 80% off.
Some models (e.g. OpenAI) have free cache write; Anthropic models
charge a cache write fee (~25% above input). Source: GitHub Copilot Models and Pricing.
-->

------

<!--slide-attr x=6400 y=2400 scale=1.0 -->

# Token Cost Optimization

| | Token Based | Model Based |
|---|---|---|
| **Approach** | Fewer tokens in/out | Cheaper per-token price |
| **Methods** | AGENTS.md, Skills, sandbox | Model selection, caching |
| **Impact** | Fewer total tokens | Lower $/token |

<!-- SPEAKER NOTES
Two parallel strategies: reduce the volume of tokens flowing through the
context window, or reduce the unit price of those tokens. Most cost savings
come from a combination of both.
-->


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

# Utilize Prompt Caching

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

<!--slide-attr x=4000 y=-2400 scale=1.0 -->

# Common Cache Missed Pattern

<img src="./images/cache-miss-model.png" alt="Model switch" class="zoomable-img" style="display: block; margin: 1.5rem auto 0; max-height: 350px; width: auto; border-radius: 10px;">

<div style="text-align: center; font-size: clamp(0.85rem, 1.6vmin, 1.1rem); color: #52525b; margin-top: 0.8rem;">Switch model mid-session</div>

<!-- SPEAKER NOTES
Changing the model mid-session invalidates the cache. The model identifier is part of the cache prefix — switching between models from any provider causes a cache miss.
-->

------

<!--slide-attr x=3200 y=-2400 scale=1.0 -->

# Common Cache Missed Pattern

<img src="./images/cache-miss-mode-change.png" alt="Mode change" class="zoomable-img" style="display: block; margin: 1.5rem auto 0; max-height: 350px; width: auto; border-radius: 10px;">

<div style="text-align: center; font-size: clamp(0.85rem, 1.6vmin, 1.1rem); color: #52525b; margin-top: 0.8rem;">Change agent mode</div>

<!-- SPEAKER NOTES
Switching between agent modes (e.g. from Ask to Agent) changes the system prompt prefix. The new mode prepends different instructions which breaks the byte-level cache match.
-->

------

<!--slide-attr x=2400 y=-2400 scale=1.0 -->

# Common Cache Missed Pattern

<img src="./images/cache-miss-tool-change.png" alt="Tool change" class="zoomable-img" style="display: block; margin: 1.5rem auto 0; max-height: 350px; width: auto; border-radius: 10px;">

<div style="text-align: center; font-size: clamp(0.85rem, 1.6vmin, 1.1rem); color: #52525b; margin-top: 0.8rem;">Different tool context</div>

<!-- SPEAKER NOTES
Different tools and their outputs are injected into the prompt prefix. When tools change between turns, the prefix changes, and the cache misses.
-->

------

<!--slide-attr x=1600 y=-2400 scale=1.0 -->

# Common Cache Missed Pattern

<img src="./images/cache-expire.png" alt="Cache expire" class="zoomable-img" style="display: block; margin: 1.5rem auto 0; max-height: 350px; width: auto; border-radius: 10px;">

<div style="text-align: center; font-size: clamp(0.85rem, 1.6vmin, 1.1rem); color: #52525b; margin-top: 0.8rem;">Wait > TTL between turns</div>

<!-- SPEAKER NOTES
Cache TTL varies by provider — 5 minutes default, up to 1 hour with paid options. If you walk away mid-session and come back later, the cache has expired. Every cache hit resets the timer, so continuous use keeps it warm.
-->

------

<!--slide-attr x=800 y=-2400 scale=1.0 -->

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

## Presentation Narative

Starting from token price formular:

A. understanding the formular 

1. Where is Token In is calculated ? 
2. Where is Token Out is calculated ?
3. Difference of model size ? (Difference price among claude sonnet / gpt 5.4 mini for example) -> Model with less parameters cost less

B. Approaches to optimize cost

1. Control Amount of Token In / Token Out / Control Thinking
2. Selective Model / Auto Model (10% off)
3. Leverage Prompt cache to further reduce model price

C. Strategies to optimize cost

> Separate into inferential optimization and computational inference - Read https://martinfowler.com/articles/harness-engineering.html to understand the concept

1. Create `AGENTS.md` or project instruction to reduce exploration (reduce token in and token out) -> inferential
2. Leverage tools like `context-mode` , `rtk` (rust token killer) to reduce token in and token out coming from tool output / response -> computational
3. Create skills like `caveman` to enforce agent reply style or to enforce number of output tokens (response in less than 50 chars for example), or have structued output response (reduce verboseness, reduce token out)
4. Create Custom Agents to enforce models belong to a specific task -> high level planning or brain storming -> higher model. Low level implementation -> Lower model.
5. Leverage prompt caching -> hints: do not switch difference models mid task, keep model in loop at all time to avoid ttl, use separate session for difference tasks, do not reuse same session for difference tasks.

D. Monitoring, obersvation, tracking for improvement

1. Monitor using built-in Agent debug view from vscode
2. Enable otel in vscode option
3. Use external collectol to collect metrics
4. Document before optimization and after optimization

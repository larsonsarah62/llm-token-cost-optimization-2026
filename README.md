# LLM Token Cost Optimization in 2026: What to Fix First

A lower token price can still leave your application with a higher bill for useful work. One customer request might trigger retrieval, several model calls, a validation step and a retry. Optimizing only the cheapest-looking call can miss the expensive part.

Start LLM token cost optimization in 2026 by measuring everything needed to complete one task. Then choose a change that fits the waste you find: remove unnecessary work, reuse repeated context, defer work that can wait, or test a cheaper model or provider. Keep the same quality bar and response deadline throughout.

![LLM token costs across the steps needed to complete one useful task.](llm-token-cost-optimization-2026-cover.png)

*Measure the cost of a completed task.*

There is no universal order after measurement. A long repeated prompt and an overnight classification queue need different fixes. The examples below use public rates checked on November 15, 2026, and hypothetical workloads; they are not measured customer bills.

## Price a completed task before changing the model

Define the useful result first. For an invoice workflow, that might mean extracting the required fields and passing validation. Count every model call needed to reach that result, including retries, fallback calls, a model used to summarize documents and any model-based evaluator.

Group those calls under one task identifier, so a retry cannot disappear into a separate request average. Keep their exact model, provider, processing mode, token counts and applicable rates. Track external tool charges separately, then include them when comparing total task cost.

The useful comparison is:

**Cost per accepted task = total cost of the evaluated tasks ÷ tasks that meet the same acceptance criteria.**

Consider a hypothetical test set of 1,000 tasks. A baseline costs $20 and all 1,000 results pass: $0.020 per accepted task. A cheaper candidate costs $18, but only 800 results pass: $0.0225 per accepted task. Total spend fell 10%, yet the cost of an acceptable result rose 12.5%. This hypothetical comparison explains the cost measure; it does not evaluate a particular model.

Token accounting also needs care. Anthropic's [usage fields](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) separate uncached input, cache writes and cache reads. Its `input_tokens` field excludes the latter two; adding them gives total input. Do not apply a second input charge to tokens already counted as cache writes or reads. Another API may organize its counters differently.

Once the record is complete, ask what dominates: unnecessary context, repeated context, output, extra calls or an expensive route. That is where the next experiment should start.

## Cut work the next step does not need

Suppose a model needs two invoice fields but receives the entire document archive on every call. Before shopping for a lower input rate, test whether the relevant pages are enough. Likewise, arithmetic, schema validation and exact identifier lookups may belong in ordinary code when their rules are stable.

That boundary matters. Checking whether a date has the right format is different from deciding whether a contract clause changes a payment obligation. Keep judgment where it is needed; remove model calls whose work can be specified reliably.

Break the prompt into instructions, tool definitions, retrieved material, conversation history and the current request. [AWS](https://docs.aws.amazon.com/wellarchitected/latest/agentic-ai-lens/agentperf03-bp02.html) recommends budgeting these components and evaluating prompt changes. This makes a large input bill actionable: restrict irrelevant retrieval, expose only relevant tools, and avoid carrying forward history the next step no longer needs.

Shorter is not automatically better. Trimming a crucial exception can cause retries or incorrect answers. A summarizing model adds its own cost and can discard details. Compare the saved downstream tokens with that extra call, and test cases where the omitted information matters.

Treat output the same way. If the next step needs a classification label, a long explanation may add no value. Set output limits and reasoning effort where the model supports them, then check whether the required result still arrives. A limit that cuts off the last invoice field can force another call. Include any billed reasoning tokens, not just the text visible to the user.

After removing avoidable context, some large inputs will still be necessary. If their beginning stays unchanged across calls, caching becomes the next question.

## Check whether repeated context earns the cache price

A shared instruction block, tool definition set or document can form the unchanged beginning of a prompt, called its prefix. With [prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching), eligible later calls can reuse the processing of that prefix. The model still generates a new answer, and the cached text still occupies context capacity.

This is different from semantic response caching. [Microsoft](https://learn.microsoft.com/en-us/azure/api-management/azure-openai-semantic-cache-lookup-policy) describes looking up a stored answer using semantic similarity. Replaying an answer introduces a separate question: is it still correct, current and appropriate for this user? Similar wording does not establish that.

For a concrete pricing example, [Anthropic](https://platform.claude.com/docs/en/about-claude/pricing) publishes these Claude Sonnet 5.5 rates for the first-party Claude API's global Standard configuration, checked November 15, 2026. All amounts are USD per million tokens:

- Uncached input: **$2.00**
- Five-minute cache write: **$2.50**
- Cache read: **$0.10**
- Output: **$10.00**

The five-minute write costs more than ordinary input. The lower read price helps only when enough eligible calls reuse it. Sonnet 5.5 requires at least 512 tokens of cacheable content; the prefix must match, within the relevant model and workspace scope. A subsequent call can use a new entry after the first response starts, and cache hits refresh its lifetime.

Here is a separate hypothetical workload. Assume 1,000 calls, one per completed task. Each sends a 10,000-token shared prefix plus 1,000 new input tokens and produces 300 total billed output tokens, including any thinking. Across 100 independent cache lifetimes, each prefix is written once and read by nine later matching calls while the five-minute entry remains valid.

Without caching, 11 million input tokens cost $22 and 0.3 million output tokens cost $3: **$25 total**.

With caching, the same workload costs:

- 1 million prefix-write tokens × $2.50 = **$2.50**
- 9 million prefix-read tokens × $0.10 = **$0.90**
- 1 million new input tokens × $2.00 = **$2.00**
- 0.3 million output tokens × $10.00 = **$3.00**

The token-charge total is **$8.40**, a $16.60 reduction, or 66.4%, assuming task quality and completion stay unchanged. The calculation excludes batch processing, fast or regional configurations, tools, retries, taxes and other charges.

Nine out of ten calls reuse the prefix here. That does not mean a 90% discount on the whole bill. If traffic arrives after expiry, prefixes change, or concurrent requests begin before the entry is available, the write/read mix changes. Use actual billed categories to recalculate.

Do not carry these rates or cache lifetimes to another model or provider. Even the same model name is insufficient to establish the same cache contract.

## Move work that can wait to the batch API

An interactive reply usually cannot wait for a batch job. An overnight document-tagging queue may be able to. The useful question is whether the job's completion deadline leaves room for asynchronous processing and recovery from partial results.

[Anthropic](https://platform.claude.com/docs/en/build-with-claude/batch-processing) offers a 50% discount on eligible input and output token rates through Message Batches. Requests that remain unfinished after 24 hours expire. [OpenAI](https://developers.openai.com/api/docs/guides/batch) also advertises a 50% discount for its Batch API with a 24-hour completion window, subject to supported models and endpoints. Unfinished work can expire; completed work is charged.

These are specific provider processing modes. Putting an ordinary API request in your own background queue does not itself earn a batch price.

Keep an identifier for each submitted item, reconcile completed, failed and expired results, and decide which items actually need resubmission. Re-running a whole job because a few items failed can erase part of the saving.

Caching needs another check in batch mode. Anthropic describes batch cache hits as best-effort because requests run asynchronously and concurrently. Do not assume the nine-read sequence in the earlier example survives unchanged. Apply the applicable batch and cache rates to the categories actually billed; do not add advertised discount percentages together.

If the deadline rules out batching, the remaining opportunity may be which model handles the task, rather than when it runs.

## Test a cheaper route against the same task

A model router chooses a model for a request. A cascade tries one model first and escalates when a defined check fails. Both can help when easier tasks are recognizable, but routing decisions, failed first attempts and escalations also cost money.

For example, a stable field-extraction task might be a candidate for a cheaper model, while ambiguous documents go to a stronger one. The test must include the ambiguous cases and a way to detect failure. A cheap first answer that silently passes a weak check is not a saving demonstrated by evaluation.

Run the candidate against the same representative task set and acceptance criteria. Compare total spend, accepted results, response times and the cost of any router or evaluator. If a large share of tasks escalate, inspect the whole two-call path before keeping it.

Changing providers is a separate comparison. On November 15, 2026, [Moduck](https://moduck.ai/models/c/anthropic/claude-sonnet-5-5) publishes a Claude Sonnet 5.5 quote of $1.60 for input, $8.00 for output, $0.16 for cached input and $2.00 for cache writes, all in USD per million tokens.

Its ordinary input and output quotes are lower than the first-party Standard rates above, while its cached-input quote is higher. That can matter for a workload dominated by cache reads. The public page does not establish the cache lifetime, an equivalent serving configuration or which models a particular account can call. These are published quotes, not measured account charges, and they do not establish like-for-like savings.

The [funding terms](https://moduck.ai/pilot/pricing) also specify a $10 minimum top-up and no fixed monthly fee. Funding is cash committed to the account; usage rates determine how that balance is consumed. Keep those two budget questions separate.

## Keep one proven change, then enforce its budget

Choose the largest measured avoidable cost that you can change without missing the quality bar or deadline. Unused context points toward removal; a large repeated prefix toward caching; a flexible queue toward batching; and a task set with identifiable easier cases toward a tested route.

Change one mechanism first so you can explain the result. Save the before-and-after prompt, model and configuration, exact token categories, task outcomes and response times. Record the evaluation and implementation cost separately from ongoing usage savings.

Keep the change only when the result holds. Set limits for calls, retries and output per task, plus spending alerts. Assign an owner and a rollback trigger, such as missed acceptance criteria or response deadlines, before expanding traffic. A cap that merely turns successful tasks into failures needs the same scrutiny as a cheaper model.

If Moduck is on your shortlist, confirm its cache lifetime, request format and available account models before funding a test. [Read the documentation](https://docs.moduck.ai/) for the current request and billing guidance.

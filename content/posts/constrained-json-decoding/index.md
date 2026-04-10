---
title: "The unexpected implications of constrained JSON decoding"
date:  2026-04-10
description: "Constrained JSON decoding works by filtering logits, not by teaching the model. This has consequences for performance, correctness, and schema design that are easy to miss."
summary: "Constrained JSON decoding works by filtering logits, not by teaching the model. This has consequences for performance, correctness, and schema design that are easy to miss."
tags: ["LLM", "structured output", "JSON", "constrained decoding", "agentic systems"]
---

Constraining LLM output to valid JSON is not a new idea. Every major API provider supports it, every serious
open-source inference engine has some form of it, and if you're building agentic systems, you're almost certainly
relying on it. The basic premise is straightforward: you give the model a JSON schema, and it produces output that
conforms to that schema.

What's less obvious is *how* this guarantee is achieved---and the mechanism has consequences that are easy to
underestimate. I certainly did. I spent a couple of days debugging a problem that, in hindsight, follows directly and
obviously from the mechanism. But I didn't understand the mechanism well enough to see it, and I suspect most
people don't.

So let's fix that.

## The mechanism

When people hear "constrained JSON output," the mental model they tend to form is something like: the model has been
taught, or configured, to produce JSON. It hasn't. The model has no idea it's being constrained. What actually
happens is this:

1. The model produces logits---a probability distribution over its entire vocabulary of tokens.
2. A separate grammar engine determines which tokens are valid continuations of the partially generated JSON so far.
3. All invalid tokens have their logits set to negative infinity.
4. Sampling proceeds as usual over whatever remains.

That's the whole thing. It's a post-hoc filter applied to each token independently. The model produces the same
probability distribution it always would have, and then a separate system discards everything that doesn't fit the
schema. The model never "learns" to produce JSON---it simply isn't *allowed* to produce anything else.

The specific implementations of this filter vary quite a bit. [Outlines](https://github.com/dottxt-ai/outlines)
converts JSON schemas to regular expressions, compiles those into finite-state machines, and pre-computes an index
mapping each FSM state to its valid token set---the foundational approach described in
["Efficient Guided Generation for Large Language Models"](https://arxiv.org/abs/2307.09702) by Willard & Louf.
[llama.cpp](https://github.com/ggml-org/llama.cpp) uses a character-level backtracking stack parser in
`src/llama-grammar.cpp`, checking each candidate token against a GBNF grammar.
[XGrammar](https://github.com/mlc-ai/xgrammar) introduced an adaptive token mask cache that precomputes validity
for ~99% of the vocabulary, needing runtime checks for only the remaining ~1%.
[llguidance](https://github.com/guidance-ai/llguidance), Microsoft's Rust-based engine, uses an Earley parser
combined with derivative-based regular expressions, achieving ~50us per mask computation with essentially zero
startup cost. [SGLang](https://github.com/sgl-project/sglang) contributes the compressed-FSM concept, collapsing
multi-token deterministic paths into single steps.

The differences matter for performance and correctness---we'll get to both---but the conceptual core is the same
everywhere. It's a filter over logits. No magic.

## The performance consequence: your GPU is waiting for your CPU

If you're running open-source models locally through [Ollama](https://github.com/ollama/ollama), you may have
noticed that enabling `format="json"` makes generation substantially slower. This is well-documented in the issue
tracker---[ollama#4370](https://github.com/ollama/ollama/issues/4370) reports a tenfold slowdown,
[ollama#3154](https://github.com/ollama/ollama/issues/3154) describes what llamafile handles in seconds taking
Ollama two minutes---and it's not really a bug. It's an architectural consequence of where grammar checking happens.

Ollama uses llama.cpp under the hood. In llama.cpp, token generation proceeds in three stages: graph preparation
(CPU), model evaluation (GPU), and sampling (CPU). Grammar enforcement happens entirely within the sampling stage.
NVIDIA documented this pipeline in a [technical blog post](https://developer.nvidia.com/blog/optimizing-llama-cpp-ai-inference-with-cuda-graphs/)
about optimizing llama.cpp with CUDA graphs---the grammar check is CPU-bound work that runs between GPU inference
passes, and there's nothing you can do about it from the outside.

The magnitude of the overhead is nontrivial. In [llama.cpp#7554](https://github.com/ggml-org/llama.cpp/issues/7554),
someone profiled grammar sampling on an RTX 3090 with Llama-3-8B: sampling time went from 1.66 ms/token to
85 ms/token---a 51x increase---while GPU utilization dropped from over 70% to around 10%. The GPU was sitting idle,
waiting for the CPU to finish checking grammar constraints.

There has been real progress. [PR #6555](https://github.com/ggml-org/llama.cpp/pull/6555) fixed combinatorial
explosions in repetition rules, achieving an 8--18x speedup in grammar processing. But the fundamental architecture
remains: grammar checking is serial CPU work that blocks the GPU pipeline.

Modern inference engines address this differently. SGLang and vLLM overlap CPU grammar computation with GPU inference,
so the mask for step *n* is computed while the GPU runs step *n+1*. XGrammar's
[paper](https://arxiv.org/abs/2411.15100) explicitly describes co-designing the grammar engine with the inference
pipeline to enable this overlap, reporting up to 100x speedup over prior approaches. The
[SqueezeBits blog](https://blog.squeezebits.com/guided-decoding-performance-vllm-sglang) provides a good overview
of this parallelization strategy.

None of this architectural work is present in Ollama's pipeline.

If you're on Apple Silicon, the situation is somewhat better. XGrammar ships with
[mlx-lm support](https://github.com/mlc-ai/xgrammar/blob/main/pyproject.toml) for `macosx_arm64`, and Apple's
unified memory architecture eliminates data transfer overhead between CPU-generated masks and GPU logits.
[Outlines has an official mlx-lm integration](https://dottxt-ai.github.io/outlines/latest/features/models/mlxlm/),
and [LM Studio](https://lmstudio.ai/blog/lmstudio-v0.3.4) uses it for structured output with MLX models.
There's also [llm-structured-output](https://github.com/otriscon/llm-structured-output), a purpose-built MLX
library using an Earley-style acceptor. I haven't found any published benchmarks comparing these specifically on
Apple Silicon---the comparison would be valuable, but as far as I can tell, nobody's done it yet.

## The design consequence: field order significantly impacts correctness

This is the one that cost me two days.

I had an agent with a simple set of actions---`ReadFile`, `WriteFile`, and a few others. The schema looked roughly
like this:

```json
ReadFile: { "path": "<path>", "type": "ReadFile" }
|        
WriteFile: { "path": "<path>", "contents": "<contents>", "type": "WriteFile" }
```

The instructions were clear. The model had a thinking block where it would reason through the situation step by
step, arrive at the correct conclusion---"We need to call `WriteFile`"---and then, you know it: it produced a 
`ReadFile` action.

The reasoning was perfect. The conclusion was correct. But the output was wrong.

It turned out to be a consequence of not having thought carefully enough about what left-to-right token generation means
for schema design.

Here's the problem. The model starts generating the JSON, and the first field it encounters is `path`. Both
`ReadFile` and `WriteFile` have a `path` field, so at this point the grammar constraint is maximally
permissive---everything valid for either action type is allowed. The model generates a path value.

Now it moves on. In the previous turn, the model had sent a `ReadFile` action. That makes it plausible that the token sequence for
`ReadFile` has elevated probability---it appeared recently in context, the model's attention mechanism is primed for
it. By the time the model gets to the `type` field, it's already generated a `path` value that's perfectly
consistent with `ReadFile`, the local context favors `ReadFile`, and the correct answer (`WriteFile`) was concluded
many tokens ago in the thinking block. The probability distribution at the `type` field is skewed.

The grammar constraint can't help. Both values are valid. The mechanics of LLM generation just pick the wrong one.

The fix is deceptively simple---just put the discriminator first:

```json
ReadFile: { "type": "ReadFile", "path": "<path>" }
|
WriteFile: { "type": "WriteFile", "path": "<path>", "contents": "<contents>" }
```

That's it. Force the model to commit to the action type before it generates any shared fields. Once
`"type": "WriteFile"` is in the output, everything downstream is conditioned on it, and the ambiguity
disappears.

This isn't just my anecdote. The [Predibase/LoRAX blog](https://predibase.com/blog/lorax-outlines-better-json-extraction-with-structured-generation-and-lora)
documents the same phenomenon quantitatively: a model fine-tuned to output fields in a specific order, forced by
alphabetical schema ordering to generate them differently, saw accuracy drop from 0.804 to 0.650. A 15-point
degradation from field reordering alone. The wrong ordering also caused infinite whitespace generation loops.

[OpenAI's documentation](https://developers.openai.com/api/docs/guides/structured-outputs) confirms that outputs
are produced in schema key order. Google's Vertex AI added a dedicated `propertyOrdering` field for explicit
control. [Dataiku](https://www.dataiku.com/stories/blog/your-guide-to-structured-text-generation) recommends
structuring JSON so that reasoning-dependent content is generated before outcome-dependent content.

The general principle is this: **the model should encounter decisive fields before ambiguous ones.** If you have a
discriminated union, put the discriminator first. If your schema has both a `reasoning` field and an `answer` field,
put `reasoning` first---otherwise the [model commits to an answer before it finishes thinking](https://collinwilkins.com/articles/structured-output).

## Always describe the schema in your prompt

There's a related insight that also follows directly from the mechanism, but which I initially found
counterintuitive: you should describe the expected JSON structure in your prompt text, even when you're already
using schema-based constrained decoding.

The reason is that constrained decoding only *masks* invalid tokens. It does nothing to *increase* the probability
of the correct valid tokens. If the model's natural distribution assigns low probability to the output you want,
you're sampling from the long tail of the distribution where everything is roughly equally improbable. The filter
ensures the output is valid JSON, but it doesn't ensure it's *good* JSON.

[Dataiku](https://www.dataiku.com/stories/blog/your-guide-to-structured-text-generation) explains this clearly: the
LLM is "unaware" of the constraints when computing next-token probabilities. Specifying the constraint in the
prompt reduces the gap between the unconstrained and constrained probability distributions.
[vLLM's documentation](https://docs.vllm.ai/en/v0.8.2/features/structured_outputs.html) makes the same point:
indicating in the prompt that JSON should be generated, and describing which fields to fill and how, improves
results notably. A [recent paper](https://arxiv.org/abs/2603.03305) on draft-conditioned constrained decoding
confirms the principle---generating an unconstrained draft first, then applying constrained decoding conditioned
on it, improves accuracy by up to 24 percentage points on GSM8K.

In practice: describe the schema's fields and their semantics in your system prompt. Give examples. Explain what
each action type means and when to use it. Treat the schema constraint as a safety net, not as the primary
guidance mechanism.

## The bug landscape

I should note that even with careful schema design and prompt engineering, the constrained decoding ecosystem
remains structurally fragile. There are critical, sometimes interacting bugs across every major implementation,
and a few patterns worth being aware of.

**Infinite loops and hangs.** These appear everywhere. [llama.cpp#10321](https://github.com/ggml-org/llama.cpp/issues/10321)
documents crashes from recursive grammars. [Outlines#658](https://github.com/dottxt-ai/outlines/issues/658) shows
large schemas consuming 32GB+ RAM. [SGLang#7639](https://github.com/sgl-project/sglang/issues/7639) reports
recursive schemas crashing both xgrammar and Outlines backends---only [llguidance](https://github.com/guidance-ai/llguidance)
handles them, making deployments vulnerable to adversarial inputs.

**Thinking mode conflicts.** If you're using reasoning models with structured output---which is likely how you
encountered this article---the interaction is particularly unpleasant. There are various issues
([Ollama#10538](https://github.com/ollama/ollama/issues/10538),
[Ollama#15260](https://github.com/ollama/ollama/issues/15260),
[SGLang#6675](https://github.com/sgl-project/sglang/issues/6675)) that plague thinking mode combined with structured output.

**XGrammar's incomplete JSON Schema support.** XGrammar is the default backend in both vLLM and SGLang, so its
limitations propagate widely. The tracking issue is [vllm#12131](https://github.com/vllm-project/vllm/issues/12131):
missing `$ref` support ([vllm#10935](https://github.com/vllm-project/vllm/pull/10935)), missing `minItems`/`maxItems`
breaking tool calls ([vllm#16880](https://github.com/vllm-project/vllm/issues/16880)), and complex schemas hanging
the server without cancellation ([vllm#14151](https://github.com/vllm-project/vllm/issues/14151)).

These aren't edge cases you'll never hit. If you're building agentic systems on open-source models, you will
likely encounter at least one of them, and knowing that the issue is in the infrastructure rather than in your
code will save you time.

## API providers have a structural advantage

There's a reason constrained decoding tends to work better through OpenAI, Anthropic, and Google than with
open-source setups, and it goes beyond model quality. API providers do two things: they fine-tune their models
to understand and produce structured output, *and* they apply constrained decoding on top as a guarantee layer.

Open-source constrained decoding can only do the second thing. The model was never specifically trained to
produce the schema you're asking for. All the work is done by the token mask, which---as we've established---only
filters, never guides. This is why the "describe the schema in your prompt" advice matters disproportionately
for open-source: you're compensating for the missing fine-tuning step with prompt engineering.

[OpenAI](https://openai.com/index/introducing-structured-outputs-in-the-api/) explicitly describes their dual
approach. [Anthropic](https://platform.claude.com/docs/en/build-with-claude/structured-outputs) launched native
structured outputs in November 2025, with constrained decoding, schema compilation, and 24-hour caching.
[JSONSchemaBench](https://arxiv.org/html/2501.10868v1), an independent benchmark of 10K real-world schemas,
found 2x differences in schema support across frameworks---the implementation details matter a great deal.

## Wrapping up

If I could go back and give myself the briefing before I started building:

**Put discriminator fields first.** If your schema represents a union of action types, `type` goes at position
zero. This single change would have saved me those two days, and I'm still a little salty about it.

**Describe the schema in your prompt.** The constraint only masks invalid tokens---it doesn't make correct tokens
more likely. Your prompt is what moves probability mass. Examples, field descriptions, explanations of when each
action type applies---all of it helps.

**Put reasoning before conclusions.** If your schema has both a `reasoning` field and an `answer` field,
`reasoning` goes first. Let the model think before it commits.

**Know the performance cost.** On Ollama with a discrete GPU, grammar enforcement runs on CPU and can slow
generation by 10--50x. If that's a problem, look into SGLang, vLLM, or XGrammar + mlx-lm on Apple Silicon.

**Expect bugs.** Especially around recursive schemas, thinking-mode models, and XGrammar's incomplete JSON Schema
coverage. Keep schemas simple, test edge cases, and if something fails inexplicably, check the issue trackers
before spending three days blaming yourself. Unlike me.

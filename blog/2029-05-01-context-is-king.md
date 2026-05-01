---
slug: context-is-king
title: Context is King
authors: [hythaam]
tags: [llm, context engineering]
---

# Context Is King: Harnesses, Cost, and Output Quality in Agentic LLM Workflows

## 1. Introduction: Why engineers should care about context

If you already understand APIs, queues, retries, and service boundaries, the jump to agentic LLM workflows is not mystical. The useful mental shift is simpler: stop treating the model as the whole system, and start treating it as one component inside a runtime that decides what information enters the model’s working set at each step. Anthropic explicitly frames context as a finite resource for agents, and OpenAI’s current guidance similarly pushes builders toward clear goals, constraints, state handling, tool design, and prompt structure rather than prompt text alone.

That is the core claim of this post: the model matters, but the surrounding harness often matters more in day-to-day use. The harness determines what the model sees, what tools it can reach, how history is preserved, and how long the loop can keep running. Once you treat context as a limited working data set instead of an infinite chat transcript, cost, latency, and output quality start to look like connected engineering tradeoffs rather than separate mysteries.

I’ll keep the discussion practical. First I’ll define the harness and the loop around the model. Then I’ll explain why cost is driven by context shape, not just model selection. Then I’ll explain why long context can hurt quality even when the needed information is technically present. I’ll finish with concrete ways to build, prune, compact, and scope context so your agent stays sharp.

Before talking about cost or quality, it helps to define the runtime around the model.

## 2. Harnesses: the runtime around the model

A harness is the control layer around the model, not the model itself. Anthropic defines an agent harness as the system that lets a model act as an agent by processing inputs, orchestrating tool calls, and returning results. In practice, that means the harness assembles context, exposes capabilities, sets limits, and defines the mode of interaction the user actually experiences. Tools, skills, pruning, compaction, retries, guardrails, and stop conditions all live here.

The standard loop is straightforward. A user request enters the harness. The harness assembles current context plus available capabilities and sends that package to a provider API. The model then either answers directly or requests a tool or function call; some models may also consume reasoning tokens before producing either kind of output. If a tool is requested, the harness executes it and feeds the result back into the next turn. The loop ends when the model emits a normal answer or when the harness hits a budget, stop rule, or other control boundary.

> **Note:** In most modern tool-calling APIs, the harness knows the model is “done” when it receives a normal assistant or message item instead of a `function_call` item. Some APIs also expose extra metadata such as `phase` so the harness can tell a brief intermediate update apart from a true final answer.

Tools are advertised extensions the model can invoke through the harness. They can be built-in features like web search or file search, your own functions and internal APIs, shell or code execution, or external services exposed through MCP servers. The important operational detail is that tool calls and tool outputs become part of the transcript the system carries forward, so tools do not just expand capability; they also reshape the working context.

For the rest of this post, I’ll use two simple context-management terms. By **pruning**, I mean removing stale or low-value material from the active working set. By **compaction**, I mean replacing a long raw history with a shorter retained state. Anthropic’s compaction docs describe this directly: older context is summarized, stale content is replaced with concise summaries, and the active context stays more focused and performant.

Once you see what the harness controls, the cost model becomes much easier to understand.

## 3. Costs: why context discipline matters even when subscriptions hide it

The first cost mistake people make is treating price as a pure model-selection problem. Model choice matters, but workflow shape matters just as much: how much context you resend, how often you call the model, how much output you request, how many tool definitions you load, and how many loops you let the harness run. OpenAI’s pricing is per token category, and its conversation-state docs explicitly note that when you continue a response chain, previous input tokens are still billed as input tokens.

> **Note:** Tokens are not words. They are chunks of text the model processes, and they can be as short as a single character or as long as a full word depending on language and context. Spaces, punctuation, and partial words count too.

That matters because long-running agent loops may keep paying for context you thought you had already “given” the model. The same is true for tool surfaces: OpenAI’s function-calling docs state that callable function definitions are injected into the system message, count against the context limit, and are billed as input tokens. So even before the model answers, you may already be spending on instructions, schemas, and tool descriptions.

Pricing usually differs for input, cached input, and output, and it also varies across model tiers. The exact numbers will change over time, but the stable lesson does not: a cheaper model can still become an expensive workflow if you let context balloon across repeated turns. Cost discipline is therefore less about memorizing a rate card and more about controlling what gets resent, what gets deferred, and what deserves a fresh call at all.

Prompt caching makes this even clearer. OpenAI’s caching docs say cache hits require exact prefix matches, which is why they recommend putting static instructions and examples first and dynamic user-specific material later. Anthropic’s prompt-caching docs describe the same economic pattern from another angle: resume from stable prefixes, reduce repeated work, and avoid unnecessary prompt churn. In other words, your prompt layout is not just a quality choice; it is a cost and latency choice.

Flat-fee chat products can make inefficiency feel invisible, but the underlying mechanics do not disappear. Somewhere in the stack, long histories, repeated tool definitions, verbose outputs, and unnecessary loops are still consuming tokens and compute. You do not need a theory about future subscription economics to benefit from better habits now.

Direct token cost is only half the story. The same context decisions also affect whether the model can reliably use the information you give it, which is often what forces extra prompting, retries, or manual cleanup in the first place.

## 4. Output quality: what long context actually does to performance

More context is not the same as better answers. The real question is whether the right information stays easy for the model to retrieve and use. Anthropic’s context-engineering writeup frames context as a finite attention budget with diminishing returns, and that is a useful engineering lens: after some point, every extra token is competing for salience rather than automatically adding value.

This is why fact retrieval gets brittle in long prompts. Relevant information may be present, but it still has to compete with instructions, examples, prior turns, tool outputs, retrieved chunks, and stale assumptions. Liu et al.’s *Lost in the Middle* showed that performance often drops when the relevant information sits in the middle of long inputs, and Chroma’s more recent *Context Rot* report found that even relatively simple distractors can degrade performance and increase hallucinated responses. Presence is not the same thing as usability.

The missing-middle pattern is the cleanest example. In plain language: models often pay more attention to material near the start or end of a long context than to material buried in the middle. That matters in real agent traces, because long tool dumps, logs, and retrieved documents naturally push critical facts away from the edges and into the least convenient part of the prompt.

> **Note:** The missing-middle effect is worth designing around, but the exact mechanisms and model-to-model severity remain active research. Recent work still finds performance drops as context grows, especially on tasks that require more reasoning than simple needle-in-a-haystack retrieval.

Context rot is the broader pattern behind that failure mode. It is not just raw length. It is the accumulation of stale assumptions, repeated noise, contradictory instructions, irrelevant tool results, and drifting task state. Anthropic’s compaction docs explicitly frame long-context degradation as a focus problem, while Chroma’s report shows inconsistent performance across context lengths even on simple tasks. The issue is not that long context is useless; it is that unmanaged long context decays.

Reasoning can degrade too. *LongReason*, a 2025 benchmark built specifically for long-context reasoning, reports significant performance drops as context length increases across reading comprehension, logical inference, and mathematical word problems. The practical lesson is blunt: giving the model access to more information does not guarantee it can reason over that information well. Sometimes the model is spending its effort sorting noise instead of solving the task.

These are context-construction problems. So the next question is how to build context so the right information stays salient instead of just technically present.

## 5. Context building: the mechanisms that keep the working set useful

Context building is an engineering activity, not a vibe. It is how you respond to retrieval failures, missing middle, context rot, and reasoning overload. Anthropic’s framing is useful here: good context engineering aims for the smallest possible set of high-signal tokens that still maximizes the chance of the desired outcome. The goal is not maximum context. The goal is useful context.

The system prompt is the durable instruction layer. It is where you anchor behavior, tone, goals, and rules that should survive across turns. OpenAI’s docs recommend putting overall role or tone guidance in the system message, and its prompt-engineering guide says higher-authority instructions define how the model should behave and take priority over user input. That is why a good system prompt is not a giant rule dump; it is a compact operating policy.

> **Note:** In many real agents, this layer is owned by the application developer rather than edited casually by the end user. OpenAI’s docs describe that authority explicitly through `instructions` and developer messages, which are prioritized ahead of user messages.

Tools do two jobs at once: they expand capability, and they pull targeted information into context only when needed. Anthropic’s context-engineering post describes this as “just in time” context loading, and OpenAI’s tools docs show the same pattern through built-in tools, custom functions, tool search, and remote MCP servers. But tool design cuts both ways: scoped, well-shaped outputs help retrieval, while verbose, unmanaged outputs recreate the same salience problems you were trying to escape.

> **Note:** “Available but unused” is not free. OpenAI says function definitions are injected into the system message and billed as input tokens, and Anthropic’s Skills docs show that installed skill metadata is present in the initial context before deeper instructions are loaded on demand. Too many options can therefore hurt both cost and quality even when the model never calls them.

Next comes task framing: the user-configurable agent prompt or working brief. OpenAI’s prompt guidance recommends explicit sections for goal, success criteria, constraints, output, and stop rules, and its current model guide recommends outcome-first prompting over over-specified process scripts. This is one of the highest-leverage improvements most engineers can make immediately. A code-review agent should get repo search, file reads, diffs, and tests, not CRM tools. A policy-answering agent should get retrieval and citation tools, not shell access. A research agent should get web search, notes, and source tracking, not your whole production action surface.

Skills are modular instruction or workflow packages that stay off the default prompt until the agent decides they are relevant. Anthropic describes them as organized folders of instructions, scripts, and resources that agents can discover and load dynamically, with progressive disclosure as the core design principle. That is powerful because specialized guidance does not need to live in the background of every request. It can be present only when needed.

Subagents solve a related problem by isolating side work instead of flooding the main thread. Anthropic’s subagent docs recommend using them when research results, logs, or file contents would otherwise clog the main conversation; each subagent gets its own context window, system prompt, tool access, and permissions, then returns a summary. As an engineering inference, that isolation also gives you cleaner discard points and sometimes cleaner cacheable prefixes than one monolithic transcript.

With those building blocks in place, the remaining question is what habits actually improve outcomes in everyday use.

## 6. Practical approaches: habits that improve outcomes without overengineering

Be specific. State the goal, constraints, prior attempts, expected output, and known facts that should not be rediscovered. OpenAI’s prompt guidance literally recommends prompt sections for goal, success criteria, constraints, output, and stop rules. That structure helps because it makes important facts easy to retrieve and hard to bury. It also gives the model a cleaner boundary for when to ask, retry, abstain, or stop instead of silently improvising through ambiguity.

Treat the interaction like mentorship, not magic. The model is closer to a fast, capable collaborator than an omniscient oracle. Clear, checkable tasks beat vague delegation. Outcome-first prompting generally works better than rigid step lists unless the exact path matters, and smaller, well-scoped tasks reduce the reasoning burden that grows in long, noisy contexts.

Use compaction on purpose. When a thread becomes long, repetitive, or stale, stop treating the full transcript as sacred. Anthropic’s compaction docs recommend summarizing older context into a compact block precisely because models lose focus across long histories. In practice, the pieces worth preserving are usually the same: current goals, decisions already made, hard constraints, source-of-truth facts, IDs, open questions, blockers, and the next concrete step. Everything else is a candidate for compression or deletion.

Bring in more context deliberately when it is actually the right answer. An agent cannot reason over information it never sees. Anthropic’s retrieval post even notes that if your knowledge base is small enough, the simplest solution may be to include the whole thing in the prompt rather than building a retrieval stack too early. The trick is not “stuff as much as possible.” The trick is “stuff what will still be retrievable.” As scale grows, contextualized chunks, reranking, and shaped tool outputs become more valuable because they preserve that retrievability.

Do not outsource all context assembly to the model. If a key constraint lives only in your head, it is not in the model’s working set. The harness only knows what you typed, what the system carried forward, and what tools returned. That means users and builders still need to participate in context selection and shaping, especially when the task involves hidden business rules, preferences, or facts the tool layer cannot infer on its own.

Finally, be skeptical about skill quality. A good skill adds reusable procedure, structure, judgment, or deterministic tooling that would be wasteful to restate every time. A bad skill is just more tokens. Anthropic’s Skills design emphasizes discoverability, progressive disclosure, and the ability to keep large resources off the active context window until they are needed. That is the bar: more leverage, not more text.

That brings us back to the main idea: context is not just prompt text. It is a design surface.

## 7. Conclusion: context as a first-class engineering constraint

The model still matters. But a huge share of what engineers observe in practice comes from the harness and the context it assembles: the instructions it preserves, the tools it exposes, the state it replays, the junk it forgets, and the summaries it carries forward. That is why the same base model can feel wildly different across products and workflows.

The payoff is two-sided. Better context discipline reduces waste by controlling repeated input, tool surface area, and cache misses, and it improves output quality by keeping the right facts salient and the reasoning burden manageable. The cost story and the quality story are not separate stories. They are both downstream of context design.

None of this requires turning every engineer into an AI researcher. It requires noticing that small workflow choices compound: where you place static instructions, whether you dump full tool logs into the thread, when you compact, how many tools you advertise, and how clearly you state the job. Over time, efficiency, salience, and context shaping are going to matter more, not less.

---

## Suggested sources

### Introduction
- [Anthropic — Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [OpenAI API — Prompt guidance](https://developers.openai.com/api/docs/guides/prompt-guidance)

### Harnesses
- [Anthropic — Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [OpenAI API — Function calling](https://developers.openai.com/api/docs/guides/function-calling)
- [Model Context Protocol — What is MCP?](https://modelcontextprotocol.io/docs/getting-started/intro)

### Costs
- [OpenAI Help Center — What are tokens and how to count them?](https://help.openai.com/en/articles/4936856-what-are-tokens-and-how-to-count-them)
- [OpenAI — API Pricing](https://openai.com/api/pricing/)
- [Claude API Docs — Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)

### Output quality
- [Liu et al. — Lost in the Middle: How Language Models Use Long Contexts](https://aclanthology.org/2024.tacl-1.9/)
- [Ling et al. — LongReason: A Synthetic Long-Context Reasoning Benchmark via Context Expansion](https://arxiv.org/abs/2501.15089)
- [Chroma — Context Rot: How Increasing Input Tokens Impacts LLM Performance](https://research.trychroma.com/context-rot)

### Context building
- [Anthropic — Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Anthropic — Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Claude Code Docs — Create custom subagents](https://docs.anthropic.com/en/docs/claude-code/sub-agents)

### Practical approaches
- [OpenAI API — Prompt guidance](https://developers.openai.com/api/docs/guides/prompt-guidance)
- [Anthropic — Contextual Retrieval in AI Systems](https://www.anthropic.com/engineering/contextual-retrieval)
- [Claude API Docs — Compaction](https://platform.claude.com/docs/en/build-with-claude/compaction)

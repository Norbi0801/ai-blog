+++
title = "What is an AI agent - beyond chatbots"
date = 2026-04-17
description = "Agent = LLM + tools + loop. The real line between a chatbot and an agent is not intelligence, it is a control flow that plans, acts, and observes until a goal is met."

[taxonomies]
tags = ["ai", "agents", "llm", "architecture"]
+++

"AI agent" is the most overloaded phrase in software right now. Every vendor has one. Half of them are chatbots with a rebrand. The rest are actually doing something new, but the definition they share is so fuzzy you can argue for hours about whether a cron job that calls GPT-4 counts.

There is a useful working definition that filters out the noise. An agent is an LLM plus a set of tools plus a loop. Chatbots map a single question to a single answer. Agents take a goal, plan, call tools, observe results, and keep going until they decide the goal is met or they give up. That loop is the whole difference. Everything else (memory, planning, multi-agent orchestration, MCP) is machinery bolted onto that loop to make it more useful.

<!-- more -->

## Chatbot vs agent, concretely

A chatbot is a function. Tokens in, tokens out. You type "explain monads" and it types back a definition. Control flow ends there. If you want to know more, you ask another question. The human is the loop.

```
user: what's the CPU load on prod-api-3?
bot:  I don't have access to live system metrics. You can check
      your monitoring dashboard or run `top` on the host.
```

An agent on the same question has a different trajectory:

```
user:  what's the CPU load on prod-api-3?
agent: (thinks) I can SSH and run top, or query Prometheus.
agent: calls prometheus_query(expr="node_load1{host='prod-api-3'}")
tool:  {"value": 4.7, "timestamp": 1714042800}
agent: "Load1 is 4.7. The host has 4 cores so that is saturating.
        Want me to pull the top processes too?"
```

The agent still runs on the same LLM. What changed is that you wrapped it in code that can execute tool calls and feed the results back. The loop is the architecture.

## The three pieces

Every agent implementation ultimately reduces to the same three ingredients.

**Model.** The LLM. It cannot do anything by itself except emit tokens. The design pressure on models has shifted from "good at prose" to "good at deciding when to call which tool and how to recover when one fails." That is why Claude 3.5+, GPT-4o, and Gemini 2.x all publish tool-use benchmarks more prominently than perplexity scores.

**Tools.** Functions the agent can invoke. `read_file`, `run_sql`, `send_email`, `create_issue`. Each one is defined by a JSON schema describing its name, what it does, and what arguments it takes. The model outputs a structured tool call, your runtime executes it, you pass the result back. I wrote a full walkthrough of this mechanism in [function calling and tool use](/blog/function-calling-how-ai-agents-interact-with-code/), including why constrained decoding makes it work reliably now where ReAct-style prompt hacks used to fail half the time.

**Loop.** A driver that keeps calling the model until it says "done". Pseudocode:

```python
def run_agent(goal: str, tools: list[Tool]) -> str:
    messages = [{"role": "user", "content": goal}]
    while True:
        response = llm.complete(messages, tools=tools)
        messages.append(response)
        if response.stop_reason == "end_turn":
            return response.text
        for call in response.tool_calls:
            result = execute(call)
            messages.append({"role": "tool", "id": call.id, "content": result})
```

That is the whole agent. Ten lines. Everything else (memory, planning, supervisors, sandboxing, retries) is an elaboration of one of the three pieces.

## ReAct: the pattern under the hood

Before function calling was a first-class API feature, researchers proved you could get this loop working with prompting alone. The ReAct paper ([Yao et al., 2022](https://arxiv.org/abs/2210.03629)) had the model interleave Thought, Action, and Observation steps:

```
Thought: I need to find out the capital of France to answer this.
Action: search("capital of France")
Observation: Paris is the capital.
Thought: Now I can answer.
Answer: Paris.
```

A wrapper would regex for `Action:` lines, execute the action, paste the result after `Observation:`, and resume generation. It worked but was fragile. Models forgot the format, mixed prose with actions, hallucinated arguments.

The reason ReAct matters today is that modern function calling is ReAct with better plumbing. The "Thought" is still there, just inside the `content` field before the tool call. "Action" is a structured `tool_use` block instead of a string the runtime has to parse. "Observation" is a `tool_result` that rides in the next message. The pattern is the same. The wire format is now schema-validated rather than regex-scraped.

If you want to see this up close, look at how Anthropic's SDK handles it: the `stop_reason` field is literally the ReAct loop condition. `tool_use` means keep going, `end_turn` means stop.

## Memory: short-term vs long-term

Models are stateless. Every request to the API is a fresh call with no memory of previous ones. Agents fake continuity by resending the entire conversation each turn. That is short-term memory: the context window.

This has obvious limits. Claude Sonnet 4.5 gives you 200k tokens, GPT-4.1 1M, Gemini 2.5 Pro 2M. Sounds like a lot until you realize a coding agent editing a medium repo blows through 100k tokens in a single session. And models degrade past a certain context length: recall on the middle of a long context is measurably worse than the ends, a problem known as "lost in the middle" ([Liu et al., 2023](https://arxiv.org/abs/2307.03172)).

Long-term memory is the workaround. You persist facts between sessions in external storage and retrieve the relevant ones into context on each new run. Common implementations:

- **Vector database.** Embed chunks of past conversations or documents, store in pgvector/Qdrant/Chroma, retrieve top-k by similarity when needed. This is the RAG pattern applied to agent state.
- **Structured store.** Key-value memory like "user prefers TypeScript over JavaScript". Cheap, deterministic, easy to edit.
- **Summarization.** Periodically compress old turns into a short summary, drop the originals. Claude Code and other coding agents do this when a session gets long.

The important thing is that long-term memory is not magic. It is a database plus a retrieval step plus a prompt that says "here are things you knew from before, use them if relevant." The agent does not have a brain. It has a read-through cache.

## Planning

For non-trivial tasks, letting the model pick one tool call at a time works poorly. It gets lost, backtracks, revisits the same subproblem. Planning is the pattern where the agent produces a full task breakdown first and then works through it.

Two flavors in practice:

**Plan then execute.** The agent first outputs a numbered plan, then executes each step. If a step fails, it either retries or re-plans. Tools like Devin and some versions of AutoGPT follow this shape.

**Tree search.** The agent explores multiple branches in parallel, evaluates them, picks a winner. Useful for problems with many plausible paths (proofs, game playing). Expensive in tokens. [Tree of Thoughts](https://arxiv.org/abs/2305.10601) is the canonical paper.

A pragmatic middle ground that works well in production: give the model a `todo_write` tool that stores a structured plan, and a `todo_update` tool to mark items done or add new ones as it learns. The plan is now explicit state that survives across turns without eating context (the tool calls persist as short messages). Claude Code does exactly this, which is why it can handle multi-hour sessions without losing the thread.

## Multi-agent systems

Once you have one agent, you can have many. Multi-agent systems come up when a task has sub-problems that benefit from specialization or isolation.

A common shape is **supervisor plus workers**. A top-level agent decomposes the task and hands subtasks to specialized agents. Example: a coding task supervisor spawns a "researcher" agent that reads docs, a "writer" agent that edits files, and a "tester" agent that runs the suite. Each worker has a smaller tool set and a narrower system prompt, which reduces confusion and context bloat.

Another shape is **peer agents**. Two or more agents converse, checking each other's work. [Debate](https://arxiv.org/abs/2305.14325) setups exploit this: two LLM instances argue over an answer, a judge picks the winner. Accuracy goes up on hard reasoning tasks. Token cost also triples.

The honest take: multi-agent architectures are often over-engineered. For most real problems, a single well-prompted agent with good tools outperforms a committee. Add agents when you have a concrete isolation reason (different permissions, different tool sets, different contexts worth keeping separate), not because it sounds sophisticated.

## MCP: the standard for tools

Every agent needs tools. If every agent reinvents the tool format, you get fifty weather APIs that all work slightly differently. The [Model Context Protocol](/blog/mcp---what-it-actually-is-and-why-it-matters/) is Anthropic's (now broadly adopted) answer: a client-server protocol where tools, resources, and prompts have a standard wire format.

The value is decoupling. You write a filesystem MCP server once. Any MCP-aware host (Claude Desktop, Cursor, VS Code, your own agent) can plug it in without custom integration code. Think of it as USB for agent tools. Before MCP, every vendor bolted tools to their own runtime. After MCP, a tool server you wrote last month works with whatever agent framework ships next month.

If you are building an agent today, MCP is the path of least regret for tooling. You get composability for free and your tools outlive your agent codebase.

## Three kinds of agents in the wild

**Coding agents.** Claude Code, Cursor, Aider, Devin. Tools: file read/write, shell, search, test runners, git. Loop often runs for hours. Memory uses the filesystem as state: they read the current code before editing it, so the repo itself is durable memory. Biggest challenge: keeping context coherent across many edits without losing the plot.

**Research agents.** Perplexity, OpenAI's Deep Research, Claude with web search. Tools: search, fetch, browse, summarize. Loop is usually tens of iterations, not thousands. Memory is often just the growing context plus a final synthesis step. Biggest challenge: dealing with contradictory sources and knowing when to stop searching.

**Automation agents.** Zapier's AI, n8n with LLM nodes, custom ops-team agents. Tools: calendar, email, CRM, internal APIs, databases. Loop is short: usually one or two tool calls per trigger. Memory is usually the business system itself (CRM rows, tickets). Biggest challenge: guardrails. An automation agent that sends a wrong email or deletes a wrong row is a bigger deal than a coding agent that writes a bad function.

## Why this matters for developers

The useful way to look at agents is not "AI that does things" but "an event loop around an LLM." Once you see that shape, you can read any agent framework's docs in five minutes: where is the loop? what tools are registered? how is memory persisted? what triggers a stop? Every framework is a different set of answers to those four questions.

It also tells you what to worry about when you build one. The model will happily call tools with wrong arguments, hallucinate tool names, ignore your plan, retry forever. Everything that makes agents production-ready (schema validation, timeouts, retry caps, sandboxing, cost limits, approval gates) is plumbing around the loop. Make the loop boring and observable, and you can ship something real. Make the loop clever and implicit, and you will debug prompt strings at 3am.

Agent architecture, once you strip the hype, is small. LLM plus tools plus loop. The interesting engineering starts after you accept that.

+++
title = "Function calling and tool use - how AI agents interact with code"
date = 2025-04-15
description = "LLMs are text-in, text-out. So how does an agent actually call your database or run a script? A walkthrough of function calling, the OpenAI vs Anthropic differences, and a minimal tool-using agent in Python."

[taxonomies]
tags = ["ai", "llm", "agents", "mcp"]
+++

An LLM is a function from tokens to tokens. No file system, no network, no state, no side effects. So when Claude writes to your repo or ChatGPT books you a flight, something else is doing the work. The model is just producing a very particular kind of text that another program knows how to read and act on.

That mechanism is function calling, also marketed as tool use. It is the thin layer that turns a language model into an agent. Once you understand it, every "AI can now do X" announcement stops being magic and starts being a careful prompt plus a JSON schema plus a dispatcher.

<!-- more -->

## The core trick: structured output instead of prose

Start with the problem. You have a chatbot. A user asks "what's the weather in Krakow?". The model has no live weather data. You want it to call your weather API. The model cannot call anything, it can only output tokens. So the game is: can we get the model to output a string that precisely specifies "call weather API with city=Krakow", which our runtime can parse and execute?

The pre-tool-use era did this with prompt engineering. You would write something like:

```
If you need to look something up, output:
ACTION: search("query here")

Otherwise respond normally.
```

Then your wrapper would regex for `ACTION:` lines and execute them. [ReAct](https://arxiv.org/abs/2210.03629) papers used exactly this pattern. It worked, badly. Models would hallucinate arguments, forget the format halfway through, mix prose and actions, escape quotes wrong.

Function calling is the industrial version. Instead of asking the model to produce an ad-hoc DSL, you give it a formal schema and the API contract guarantees the output either matches the schema exactly or says "no tool call needed, here is prose". Providers enforce this server-side with constrained decoding: at each token step, only tokens that keep the output valid against the JSON Schema are allowed. You cannot get a syntactically invalid function call out of GPT-4 or Claude anymore. The decoder physically cannot emit one.

## What a tool definition looks like

A tool has a name, a description, and an input schema. The schema is [JSON Schema](https://json-schema.org/), the same thing OpenAPI uses. Here is a tool that reads a file:

```json
{
  "name": "read_file",
  "description": "Read the contents of a file from the local filesystem. Returns the full text content as a string.",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": {
        "type": "string",
        "description": "Absolute path to the file"
      },
      "max_bytes": {
        "type": "integer",
        "description": "Maximum bytes to read. Defaults to 1 MB.",
        "default": 1048576
      }
    },
    "required": ["path"]
  }
}
```

The description field is not documentation. It is prompt. The model sees it at every turn and uses it to decide whether to call this tool and what to pass. A vague description produces a model that misuses the tool or ignores it. "Read a file" is bad. "Read the contents of a file from the local filesystem. Use this before editing any source code so you see the current state" is better because it tells the model when to reach for it.

Property descriptions matter too. Claude in particular takes them seriously and will quote them back when it needs to explain a refusal.

## The turn-based loop

A conversation with tools is not one request-one response. It is a loop of turns:

1. Client sends the user message plus the list of available tools.
2. Model replies with either prose (done) or one or more `tool_use` blocks.
3. Client executes each requested tool call locally.
4. Client sends the results back as `tool_result` messages, along with the full prior history.
5. Model either produces more tool calls or finally produces prose. Loop back to step 3 if needed.

The model is stateless across these turns. You resend the entire conversation every time. That is why long agent sessions burn tokens: every tool call costs another pass over the growing history. This is also why [prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) is basically mandatory for agents. Without it, a ten-turn agent loop re-processes the system prompt and tool list ten times.

## Anthropic's tool API

Claude's tool format uses explicit content blocks. A response that wants to call a tool looks like:

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "I'll read the config file to check."
    },
    {
      "type": "tool_use",
      "id": "toolu_01A2...",
      "name": "read_file",
      "input": {"path": "/etc/app.conf"}
    }
  ],
  "stop_reason": "tool_use"
}
```

You execute, then reply with a user-role message that contains a `tool_result` block matching the `id`:

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01A2...",
      "content": "log_level = debug\nport = 8080"
    }
  ]
}
```

`stop_reason: "tool_use"` is your cue to loop. When it becomes `"end_turn"` the agent is done.

A minimal agent in Python with the official SDK:

```python
import anthropic

client = anthropic.Anthropic()

tools = [{
    "name": "read_file",
    "description": "Read a file from disk.",
    "input_schema": {
        "type": "object",
        "properties": {"path": {"type": "string"}},
        "required": ["path"],
    },
}]

def execute_tool(name, args):
    if name == "read_file":
        with open(args["path"]) as f:
            return f.read()
    raise ValueError(f"unknown tool {name}")

messages = [{"role": "user", "content": "What port is in /etc/app.conf?"}]

while True:
    resp = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=1024,
        tools=tools,
        messages=messages,
    )
    messages.append({"role": "assistant", "content": resp.content})

    if resp.stop_reason != "tool_use":
        print(resp.content[-1].text)
        break

    tool_results = []
    for block in resp.content:
        if block.type == "tool_use":
            result = execute_tool(block.name, block.input)
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": str(result),
            })
    messages.append({"role": "user", "content": tool_results})
```

That is a complete agent. Thirty lines. Everything else (parallel tool calls, retries, streaming, caching) is additive.

## OpenAI's tool API

OpenAI wraps the same idea differently. The schema is passed under a `function` key nested in a `tool` object:

```python
from openai import OpenAI

client = OpenAI()

tools = [{
    "type": "function",
    "function": {
        "name": "read_file",
        "description": "Read a file from disk.",
        "parameters": {
            "type": "object",
            "properties": {"path": {"type": "string"}},
            "required": ["path"],
        },
    },
}]
```

Responses use `tool_calls` on the message, and you reply with a dedicated `tool` role:

```python
resp = client.chat.completions.create(
    model="gpt-4.1",
    tools=tools,
    messages=messages,
)

msg = resp.choices[0].message
if msg.tool_calls:
    for call in msg.tool_calls:
        args = json.loads(call.function.arguments)  # arguments is a STRING
        result = execute_tool(call.function.name, args)
        messages.append({
            "role": "tool",
            "tool_call_id": call.id,
            "content": str(result),
        })
```

Three differences that bite in practice:

1. **Arguments are a JSON string, not an object.** You have to `json.loads` it. Anthropic gives you a parsed dict. This exists because OpenAI streams the arguments token by token and wants you to accumulate the raw string.
2. **Tool results use a dedicated `tool` role.** Anthropic reuses `user` with a `tool_result` block.
3. **Schema is under `function.parameters`, not `input_schema`.** Same JSON Schema, different key.

For structured output specifically (when you want the model to return JSON that matches a schema but not actually call an external thing), OpenAI has a separate `response_format: json_schema` mode with [Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs). Anthropic just suggests you define a tool and instruct the model to always call it.

## MCP: when N clients meet M tool servers

Hardcoding tools per agent is fine if you have one agent and three tools. It falls apart when you have Claude Desktop, Cursor, your own CLI agent, and VS Code Copilot all wanting access to the same filesystem, git, database, and browser tools. That is the N times M problem, and it is what the [Model Context Protocol](https://modelcontextprotocol.io/) solves.

MCP is a JSON-RPC protocol that sits between the agent host and the tool provider. Instead of writing glue code per agent, you write one MCP server that exposes your tools, and any MCP-capable agent can use it. Under the hood, MCP's `tools/list` and `tools/call` methods carry the exact same JSON Schema that Anthropic and OpenAI consume, so your server's tool definitions drop straight into the model's context.

I wrote about MCP in more depth in [MCP - what it actually is and why it matters](/blog/mcp---what-it-actually-is-and-why-it-matters/), and the primitive differences in [MCP tools vs resources vs prompts](/blog/mcp-tools-vs-resources-vs-prompts---understanding-the-protoc/). For this post the thing to know is: function calling is the LLM feature, MCP is the wire protocol that lets tool implementations be shared across LLM apps. They are not competitors, they compose.

## Error handling: tools fail, often

Every tool call is a place where the real world intrudes. Files do not exist, APIs rate limit, JSON parsing fails on malformed arguments, timeouts happen. Good agents handle this loudly, not silently.

The pattern that works: return the error as the tool result, flagged as an error. Both APIs support this explicitly. Anthropic:

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01A2...",
  "content": "Error: file not found: /etc/app.conf",
  "is_error": true
}
```

OpenAI has no equivalent flag, you just put the error text in `content` and trust the model to recognize it. In both cases the model sees the error next turn and almost always tries to recover (try a different path, ask the user for clarification, use a different tool).

What does not work: raising a Python exception and crashing your loop. The model learns nothing, the user sees a stack trace, and whatever partial state you had is gone. The only exception is if the arguments themselves are unrecoverable, in which case returning a clear schema-violation error is still better than crashing.

A few things worth building into your tool dispatcher from day one:

- **Timeouts per tool.** A hung subprocess will hang the agent forever. Set per-tool deadlines, return a timeout error as a tool result if exceeded.
- **Output size limits.** Models have context limits. A tool that reads a 10 MB log file and returns it all will blow past the window and corrupt the rest of the conversation. Truncate with a clear marker like `[... 42318 more bytes truncated ...]`.
- **Idempotency flags.** Mark destructive tools (file delete, database write, API POST) so you can require confirmation in certain modes. The model will happily delete things it should not if your tool description is ambiguous.
- **Argument validation beyond the schema.** JSON Schema catches type errors, not "this path is outside the allowed directory". Re-validate in code.

## What is actually happening under the hood

When you send a tool definition to Anthropic or OpenAI, the server does two things.

First, it converts your JSON Schema into a grammar that the decoder can enforce. Exactly how varies by provider, but [outlines](https://github.com/dottxt-ai/outlines), [llama.cpp's GBNF](https://github.com/ggml-org/llama.cpp/blob/master/grammars/README.md), and [xgrammar](https://github.com/mlc-ai/xgrammar) are the open-source versions of the same idea. At each decoding step, the grammar narrows the valid next-token set. The logits for invalid tokens are masked to negative infinity before sampling. The result: the output is structurally guaranteed to match the schema. What is not guaranteed is semantic correctness. The model can still pick a wrong path, call the wrong tool, or hallucinate a city name.

Second, the tool definitions are injected into the prompt in a provider-specific format. For Claude it is a special `<tools>` section in the system prompt. For GPT it is a similar serialized blob. If you count tokens carefully you can see it: a 20-tool agent eats 2-5k tokens per turn just on tool definitions. This is the single biggest reason to keep tools focused and cache aggressively.

The "agent" part is entirely client-side. The model does not know it is in a loop. It sees a conversation that happens to include tool results, and each forward pass is independent. Memory, planning, reflection, all of that is something your orchestrator adds on top of the base call-and-respond mechanic. The base mechanic is honestly boring: structured JSON out, tool result in, repeat.

Which is exactly why it scales. Every future "agent framework" is just a smarter dispatcher on top of the same four-step loop. If you can write the thirty-line version from scratch, you can reason about any of them.

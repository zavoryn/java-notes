# Writing Effective Tools for AI Agents—Using AI Agents

> **中文总结**: Anthropic 系统阐述为 Agent 编写高效工具的方法论。五大原则：选择正确工具（少而精）、命名空间分组、返回有意义上下文（自然语言优于 UUID）、优化 token 效率（分页/过滤/截断）、提示工程工具描述。细微改进工具描述可显著提升表现。

> Source: https://www.anthropic.com/engineering/writing-tools-for-agents
> Authors: Ken Aizawa, with contributions from Research, MCP, Product Engineering, Marketing, Design, and Applied AI teams (Anthropic)

The Model Context Protocol (MCP) can empower LLM agents with potentially hundreds of tools to solve real-world tasks. But how do we make those tools maximally effective?

## What is a tool?

Tools are a new kind of software which reflects a contract between deterministic systems and non-deterministic agents. Instead of writing tools and MCP servers the way we'd write functions and APIs for other developers or systems, we need to design them for agents.

## How to write tools

### Building a prototype

Start by standing up a quick prototype of your tools. Wrapping your tools in a local MCP server or Desktop extension (DXT) will allow you to connect and test your tools in Claude Code or the Claude Desktop app.

### Running an evaluation

Generate lots of evaluation tasks, grounded in real world uses. Strong evaluation tasks might require multiple tool calls—potentially dozens. Each evaluation prompt should be paired with a verifiable response or outcome.

### Collaborating with agents

You can even let agents analyze your results and improve your tools for you. Simply concatenate the transcripts from your evaluation agents and paste them into Claude Code.

## Principles for writing effective tools

### Choosing the right tools for agents

More tools don't always lead to better outcomes. Build a few thoughtful tools targeting specific high-impact workflows. Tools can consolidate functionality, handling potentially multiple discrete operations under the hood.

Examples:
- Instead of `list_users`, `list_events`, and `create_event`, implement a `schedule_event` tool
- Instead of `read_logs`, implement a `search_logs` tool
- Instead of `get_customer_by_id`, `list_transactions`, and `list_notes`, implement a `get_customer_context` tool

### Namespacing your tools

Namespacing (grouping related tools under common prefixes) can help delineate boundaries between lots of tools. For example, `asana_search`, `jira_search`, `asana_projects_search`, `asana_users_search`.

### Returning meaningful context from your tools

Tool implementations should take care to return only high signal information back to agents. Prioritize contextual relevance over flexibility. Natural language identifiers work better than cryptic UUIDs.

### Optimizing tool responses for token efficiency

Implement pagination, range selection, filtering, and/or truncation with sensible default parameter values. For Claude Code, tool responses are restricted to 25,000 tokens by default.

### Prompt-engineering your tool descriptions

Think of how you would describe your tool to a new hire on your team. Make implicit context explicit. Avoid ambiguity. Even small refinements to tool descriptions can yield dramatic improvements.

## Looking ahead

Effective tools are intentionally and clearly defined, use agent context judiciously, can be combined together in diverse workflows, and enable agents to intuitively solve real-world tasks.

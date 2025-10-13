# New Tools for Building Agents

> **中文总结**: OpenAI 发布 Agent 开发者工具套件：Responses API（统一 Chat Completions + Assistants）、内置工具（Web Search/File Search/Computer Use）、开源 Agents SDK（编排、交接、护栏、追踪）。Assistants API 计划 2026 年中弃用。

> Source: https://openai.com/index/new-tools-for-building-agents/
> Authors: OpenAI

Today, we're releasing the first set of building blocks that will help developers and enterprises build useful and reliable agents. We view agents as systems that independently accomplish tasks on behalf of users.

## Introducing the Responses API

The Responses API is our new API primitive for leveraging OpenAI's built-in tools to build agents. It combines the simplicity of Chat Completions with the tool-use capabilities of the Assistants API. With a single Responses API call, developers will be able to solve increasingly complex tasks using multiple tools and model turns.

The Responses API will support new built-in tools like web search, file search, and computer use. These tools are designed to work together to connect models to the real world.

## What this means for existing APIs

- **Chat Completions API**: Remains fully supported. The Responses API is a superset of Chat Completions.
- **Assistants API**: Key improvements incorporated into Responses API. Deprecation targeted for mid-2026.

## Introducing built-in tools in the Responses API

### Web search

Developers can now get fast, up-to-date answers with clear and relevant citations from the web. Available as a tool when using gpt-4o and gpt-4o-mini. GPT-4o search preview scores 90% on SimpleQA.

### File search

Retrieve relevant information from large volumes of documents with support for multiple file types, query optimization, metadata filtering, and custom reranking. Enables fast, accurate search results.

### Computer use

The computer use tool in the Responses API is powered by the same Computer-Using Agent (CUA) model that enables Operator. Achieves 38.1% success on OSWorld, 58.1% on WebArena, and 87% on WebVoyager.

## Agents SDK

Our new open-source Agents SDK simplifies orchestrating multi-agent workflows with significant improvements over Swarm:

- **Agents**: Easily configurable LLMs with clear instructions and built-in tools
- **Handoffs**: Intelligently transfer control between agents
- **Guardrails**: Configurable safety checks for input and output validation
- **Tracing & Observability**: Visualize agent execution traces to debug and optimize performance

The Agents SDK works with the Responses API and Chat Completions API, and also supports models from other providers.

## What's next

We believe agents will soon become integral to the workforce, significantly enhancing productivity across industries. We'll continue investing in deeper integrations across our APIs and new tools to help deploy, evaluate, and optimize agents in production.

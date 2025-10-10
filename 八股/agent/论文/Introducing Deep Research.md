# Introducing Deep Research

> **中文总结**: OpenAI 发布基于 o3 模型的 Deep Research 功能，能自主进行多步互联网研究。HLE 达 26.6%（当时 SOTA），GAIA SOTA。5-30 分钟完成研究报告，输出完整引用。局限性：仍有幻觉、置信度校准弱。

> Source: https://openai.com/index/introducing-deep-research/
> Authors: OpenAI (Research Leads: Isa Fulford, Zhiqing Sun)

Today we're launching deep research in ChatGPT, a new agentic capability that conducts multi-step research on the internet for complex tasks. It accomplishes in tens of minutes what would take a human many hours.

Deep research is OpenAI's next agent that can do work for you independently—you give it a prompt, and ChatGPT will find, analyze, and synthesize hundreds of online sources to create a comprehensive report at the level of a research analyst. Powered by a version of the upcoming OpenAI o3 model that's optimized for web browsing and data analysis, it leverages reasoning to search, interpret, and analyze massive amounts of text, images, and PDFs on the internet, pivoting as needed in reaction to information it encounters.

## Why we built deep research

Deep research is built for people who do intensive knowledge work in areas like finance, science, policy, and engineering and need thorough, precise, and reliable research. It can be equally useful for discerning shoppers looking for hyper-personalized recommendations on purchases that typically require careful research, like cars, appliances, and furniture. Every output is fully documented, with clear citations and a summary of its thinking, making it easy to reference and verify the information.

Deep research independently discovers, reasons about, and consolidates insights from across the web. To accomplish this, it was trained on real-world tasks requiring browser and Python tool use, using the same reinforcement learning methods behind OpenAI o1, our first reasoning model.

## How to use deep research

In ChatGPT, select 'deep research' in the message composer and enter your query. Tell ChatGPT what you need—whether it's a competitive analysis on streaming platforms or a personalized report on the best commuter bike. You can attach files or spreadsheets to add context to your question. Once it starts running, a sidebar appears with a summary of the steps taken and sources used.

Deep research may take anywhere from 5 to 30 minutes to complete its work, taking the time needed to dive deep into the web. In the meantime, you can step away or work on other tasks—you'll get a notification once the research is complete.

## How it works

Deep research was trained using end-to-end reinforcement learning on hard browsing and reasoning tasks across a range of domains. Through that training, it learned to plan and execute a multi-step trajectory to find the data it needs, backtracking and reacting to real-time information where necessary.

On **Humanity's Last Exam**, a recently released evaluation that tests AI across a broad range of subjects on expert-level questions, the model powering deep research scores a new high at 26.6% accuracy.

| Model | Accuracy (%) |
| --- | --- |
| GPT-4o | 3.3 |
| Claude 3.5 Sonnet | 4.3 |
| OpenAI o1 | 9.1 |
| OpenAI o3-mini (high) | 13.0 |
| OpenAI deep research | 26.6 |

On **GAIA**, a public benchmark that evaluates AI on real-world questions, the model powering deep research reaches a new state of the art (SOTA).

## Limitations

Deep research unlocks significant new capabilities, but it's still early and has limitations. It can sometimes hallucinate facts in responses or make incorrect inferences, though at a notably lower rate than existing ChatGPT models. It may struggle with distinguishing authoritative information from rumors, and currently shows weakness in confidence calibration, often failing to convey uncertainty accurately.

## What's next

Looking further ahead, we envision agentic experiences coming together in ChatGPT for asynchronous, real-world research and execution. The combination of deep research, which can perform asynchronous online investigation, and Operator, which can take real-world action, will enable ChatGPT to carry out increasingly sophisticated tasks for you.

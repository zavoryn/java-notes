# Introducing ChatGPT Agent

> **中文总结**: OpenAI 将 Operator（网页交互）+ Deep Research（信息综合）+ ChatGPT 统一为 ChatGPT Agent。配备可视化浏览器、文本浏览器、终端、API 和连接器。HLE 41.6% SOTA，BrowseComp 68.9%，支持实时交互和中断。

> Source: https://openai.com/index/introducing-chatgpt-agent/
> Authors: OpenAI

ChatGPT can now do work for you using its own computer, handling complex tasks from start to finish.

You can now ask ChatGPT to handle requests like "look at my calendar and brief me on upcoming client meetings based on recent news," "plan and buy ingredients to make Japanese breakfast for four," and "analyze three competitors and create a slide deck." ChatGPT will intelligently navigate websites, filter results, prompt you to log in securely when needed, run code, conduct analysis, and even deliver editable slideshows and spreadsheets that summarize its findings.

At the core of this new capability is a unified agentic system. It brings together three strengths of earlier breakthroughs: Operator's ability to interact with websites, deep research's skill in synthesizing information, and ChatGPT's intelligence and conversational fluency.

## A natural evolution of Operator and deep research

Previously, Operator and deep research each brought unique strengths: Operator could scroll, click, and type on the web, while deep research excelled at analyzing and summarizing information. But they worked best in different situations: Operator couldn't dive deep into analysis or write detailed reports, and deep research couldn't interact with websites to refine results or access content requiring user authentication.

By integrating these complementary strengths in ChatGPT and introducing additional tools, we've unlocked entirely new capabilities within one model. It can now actively engage websites—clicking, filtering, and gathering more precise, efficient results.

## An agent that works for you, with you

We've equipped ChatGPT agent with a suite of tools: a visual browser that interacts with the web through a graphical-user interface, a text-based browser for simpler reasoning-based web queries, a terminal, and direct API access. The agent can also leverage ChatGPT connectors, which allows you to connect apps like Gmail and Github so ChatGPT can find information relevant to your prompts and use them in its responses.

All this is done using its own virtual computer, which preserves the context necessary for the task, even when multiple tools are used—the model can choose to open a page using the text browser or visual browser, download a file from the web, manipulate it by running a command in the terminal, and then view the output back in the visual browser.

ChatGPT agent is designed for iterative, collaborative workflows, far more interactive and flexible than previous models. As ChatGPT works, you can interrupt at any point to clarify your instructions, steer it toward desired outcomes, or change the task entirely.

## Performance

On **Humanity's Last Exam**, the model powering ChatGPT agent scores a new pass@1 SOTA at 41.6.

On **FrontierMath**, the hardest known math benchmark featuring novel, unpublished problems, ChatGPT agent reaches 27.4% accuracy with tool use.

On an internal benchmark designed to evaluate model performance on **complex, economically valuable knowledge-work tasks**, ChatGPT agent's output is comparable to or better than that of humans in roughly half the cases.

On **BrowseComp**, a benchmark measuring browsing agents' ability to locate hard-to-find information on the web, the model set a new SOTA with 68.9%, 17.4 percentage points higher than deep research.

On **SpreadsheetBench**, ChatGPT agent scores 45.5%, compared to Copilot in Excel's 20.0%.

## Novel capabilities, novel risks

This release marks the first time users can ask ChatGPT to take actions on the web. Key safety measures include:

- **Explicit user confirmation**: ChatGPT is trained to explicitly ask for your permission before taking actions with real-world consequences
- **Active supervision ("Watch Mode")**: Certain critical tasks require your active oversight
- **Proactive risk mitigation**: ChatGPT is trained to actively refuse high-risk tasks such as bank transfers
- **Privacy controls**: With a single click, you can delete all browsing data and immediately log out of all active website sessions
- **Secure browser takeover mode**: When you interact with the web using ChatGPT's browser, your inputs remain private

The model has our most comprehensive safety stack to date with enhanced safeguards for biology: comprehensive threat modeling, dual-use refusal training, always-on classifiers and reasoning monitors, and clear enforcement pipelines.

## Availability

ChatGPT agent starts rolling out to Pro, Plus, and Team users. Pro users have 400 messages per month, while other paid users get 40 messages monthly, with additional usage available via flexible credit-based options.

## Limitations and looking ahead

ChatGPT agent is still in its early stages. It's capable of taking on a range of complex tasks, but it can still make mistakes. While we see significant potential in its ability to generate slideshows, this functionality is currently in beta.

Overall, we expect continued improvements to ChatGPT agent's efficiency, depth, and versatility over time, including more seamless interactions as we continue to adjust the amount of oversight required from the user to make it more useful while ensuring it's safe to use.

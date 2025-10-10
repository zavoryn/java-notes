# Computer-Using Agent

> **中文总结**: OpenAI 的 CUA 模型通过像素级 GUI 交互操作计算机，结合 GPT-4o 视觉 + 强化学习。OSWorld 38.1%，WebArena 58.1%，WebVoyager 87%。不依赖特定 API，像人类一样操作按钮、菜单和文本框，解决"长尾"数字用例。

> Source: https://openai.com/index/computer-using-agent/
> Authors: OpenAI

Today we introduced a research preview of Operator, an agent that can go to the web to perform tasks for you. Powering Operator is Computer-Using Agent (CUA), a model that combines GPT-4o's vision capabilities with advanced reasoning through reinforcement learning. CUA is trained to interact with graphical user interfaces (GUIs)—the buttons, menus, and text fields people see on a screen—just as humans do. This gives it the flexibility to perform digital tasks without using OS- or web-specific APIs.

CUA builds off of years of foundational research at the intersection of multimodal understanding and reasoning. By combining advanced GUI perception with structured problem-solving, it can break tasks into multi-step plans and adaptively self-correct when challenges arise. This capability marks the next step in AI development, allowing models to use the same tools humans rely on daily and opening the door to a vast range of new applications.

While CUA is still early and has limitations, it sets new state-of-the-art benchmark results, achieving a 38.1% success rate on OSWorld for full computer use tasks, and 58.1% on WebArena and 87% on WebVoyager for web-based tasks. These results highlight CUA's ability to navigate and operate across diverse environments using a single general action space.

We've developed CUA with safety as a top priority to address the challenges posed by an agent having access to the digital world, as detailed in our Operator System Card. In line with our iterative deployment strategy, we are releasing CUA through a research preview of Operator at operator.chatgpt.com for Pro Tier users in the U.S. to start. By gathering real-world feedback, we can refine safety measures and continuously improve as we prepare for a future with increasing use of digital agents.

## How it works

CUA processes raw pixel data to understand what's happening on the screen and uses a virtual mouse and keyboard to complete actions. It can navigate multi-step tasks, handle errors, and adapt to unexpected changes. This enables CUA to act in a wide range of digital environments, performing tasks like filling out forms and navigating websites without needing specialized APIs.

Given a user's instruction, CUA operates through an iterative loop that integrates perception, reasoning, and action:

- **Perception**: Screenshots from the computer are added to the model's context, providing a visual snapshot of the computer's current state.
- **Reasoning**: CUA reasons through the next steps using chain-of-thought, taking into consideration current and past screenshots and actions. This inner monologue improves task performance by enabling the model to evaluate its observations, track intermediate steps, and adapt dynamically.
- **Action**: It performs the actions—clicking, scrolling, or typing—until it decides that the task is completed or user input is needed. While it handles most steps automatically, CUA seeks user confirmation for sensitive actions, such as entering login details or responding to CAPTCHA forms.

## Evaluations

CUA establishes a new state-of-the-art in both computer use and browser use benchmarks by using the same universal interface of screen, mouse, and keyboard.

| Benchmark type | Benchmark | Computer use (universal interface) | Web browsing agents | Human |
| --- | --- | --- | --- | --- |
| | | OpenAI CUA | Previous SOTA | Previous SOTA | |
| Computer use | OSWorld | 38.1% | 22.0% | - | 72.4% |
| Browser use | WebArena | 58.1% | 36.2% | 57.1% | 78.2% |
| WebVoyager | 87.0% | 56.0% | 87.0% | - |

## Safety

Because CUA is one of our first agentic products with an ability to directly take actions in a browser, it brings new risks and challenges to address. As we prepared for deployment of Operator, we did extensive safety testing and implemented mitigations across three major classes of safety risks: misuse, model mistakes, and frontier risks. We believe it is important to take a layered approach to safety, so we implemented safeguards across the whole deployment context: the CUA model itself, the Operator system, and post-deployment processes.

- **Misuse**: Refusals, blocklists, moderation, offline detection
- **Model mistakes**: User confirmations, task limitations, watch mode
- **Adversarial attacks**: Cautious navigation, monitoring, detection pipeline
- **Frontier risks**: Evaluated against Preparedness Framework

## Conclusion

CUA builds on years of research advancements in multimodality, reasoning and safety. The flexibility offered by a universal interface enables an agent that can navigate any software tool designed for humans. By moving beyond specialized agent-friendly APIs, CUA can adapt to whatever computer environment is available—truly addressing the "long tail" of digital use cases that remain out of reach for most AI models.

## Authors

OpenAI

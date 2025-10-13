# Measuring AI Agent Autonomy in Practice

> **中文总结**: Anthropic 首次大规模实证研究 Agent 实际部署中的自主性。四大发现：自主工作时间翻倍、有经验用户更多自动批准但也更多中断、Claude 主动暂停频率是用户中断的 2 倍、软件工程占 Agent 活动近 50%。建议投资部署后监控和不确定性训练。

> Source: https://www.anthropic.com/news/measuring-agent-autonomy
> Authors: Miles McCain, Thomas Millar, Saffron Huang, Jake Eaton, Kunal Handa, Michael Stern, Alex Tamkin, Matt Kearney, Esin Durmus, Judy Shen, Jerry Hong, Brian Calvert, Jun Shern Chan, Francesco Mosconi, David Saunders, Tyler Neylon, Gabriel Nicholas, Sarah Pollack, Jack Clark, Deep Ganguli (Anthropic)

AI agents are here, and already they're being deployed across contexts that vary widely in consequence, from email triage to cyber espionage. Understanding this spectrum is critical for deploying AI safely, yet we know surprisingly little about how people actually use agents in the real world.

We analyzed millions of human-agent interactions across both Claude Code and our public API using our privacy-preserving tool.

## Key Findings

- **Claude Code is working autonomously for longer.** Among the longest-running sessions, the length of time Claude Code works before stopping has nearly doubled in three months, from under 25 minutes to over 45 minutes.
- **Experienced users in Claude Code auto-approve more frequently, but interrupt more often.** As users gain experience with Claude Code, they tend to stop reviewing each action and instead let Claude run autonomously, intervening only when needed.
- **Claude Code pauses for clarification more often than humans interrupt it.** On the most complex tasks, Claude Code stops to ask for clarification more than twice as often as humans interrupt it.
- **Agents are used in risky domains, but not yet at scale.** Most agent actions on our public API are low-risk and reversible. Software engineering accounted for nearly 50% of agentic activity.

## Recommendations

- **Model and product developers should invest in post-deployment monitoring.**
- **Model developers should consider training models to recognize their own uncertainty.**
- **Product developers should design for user oversight.**
- **It's too early to mandate specific interaction patterns.**

A central lesson from this research is that the autonomy agents exercise in practice is co-constructed by the model, the user, and the product.

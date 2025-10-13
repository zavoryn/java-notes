# Practices for Governing Agentic AI Systems

> **中文总结**: OpenAI 的 Agentic AI 治理白皮书。提出 Agent 的四个维度（目标复杂性、环境复杂性、适应性、独立执行）和七项安全治理实践（行动空间限制、审批机制、默认行为、透明度、自动监控、可归因性、可中断性）。

> Source: https://cdn.openai.com/papers/practices-for-governing-agentic-ai-systems.pdf
> Authors: Yonadav Shavit, Sandhini Agarwal, Miles Brundage, Steven Adler, Cullen O'Keefe, Rosie Campbell, Teddy Lee, Pamela Mishkin, Tyna Eloundou, Alan Hickey, Katarina Slama, Lama Ahmad, Paul McMillan, Alex Beutel, Alexandre Passos, David G. Robinson (OpenAI)

Agentic AI systems—AI systems that can pursue complex goals with limited direct supervision—are likely to be broadly useful if we can integrate them responsibly into our society. While such systems have substantial potential to help people more efficiently and effectively achieve their own goals, they also create risks of harm. In this white paper, we suggest a definition of agentic AI systems and the parties in the agentic AI system life-cycle, and highlight the importance of agreeing on a set of baseline responsibilities and safety best practices for each of these parties.

## 1. Introduction

AI researchers and companies have recently begun to develop increasingly agentic AI systems: systems that adaptably pursue complex goals using reasoning and with limited direct supervision. For example, a user could ask an agentic personal assistant to "help me bake a good chocolate cake tonight," and the system would respond by figuring out the ingredients needed, finding vendors to buy ingredients, and having the ingredients delivered to their doorstep along with a printed recipe.

Agentic AI systems are distinct from more limited AI systems (like image generation or question-answering language models) because they are capable of a wide range of actions and are reliable enough that, in certain defined circumstances, a reasonable user could trust them to effectively and autonomously act on complex goals on their behalf.

## 2. Definitions

### 2.1 Agenticness, Agentic AI Systems, and "Agents"

An AI system's agenticness is best understood as involving multiple dimensions, along each of which we expect the field to continue to progress. We define the degree of agenticness in a system as "the degree to which a system can adaptably achieve complex goals in complex environments with limited direct supervision."

Agenticness breaks down into several components:

- **Goal complexity**: How challenging would the AI system's goal be for a human to achieve and how wide of a range of goals could the system achieve?
- **Environmental complexity**: How complex are the environments under which a system can achieve the goal?
- **Adaptability**: How well can the system adapt and react to novel or unexpected circumstances?
- **Independent execution**: To what extent can the system reliably achieve its goals with limited human intervention or supervision?

### 2.2 The Human Parties in the AI Agent Life-cycle

The three primary parties that may influence an AI agent's operations are:

- **Model developer**: The party that develops the AI model that powers the agentic system
- **System deployer**: The party that builds and operates the larger system built on top of a model
- **User**: The party that employs the specific instance of the agentic AI system

## 3. Potential Benefits of Agentic AI Systems

### 3.1 Agenticness as a Helpful Property

- Higher quality and more reliable outputs
- More efficient use of users' time
- Improved user preference solicitation
- Scalability

### 3.2 Agenticness as an Impact Multiplier

Agenticness can be viewed as a prerequisite for some of the wider systemic impacts that many expect from the diffusion of AI—some of which have significant potential to benefit society.

## 4. Practices for Keeping Agentic AI Systems Safe and Accountable

### 4.1 Evaluating Suitability for the Task

Either the system deployer or the user should thoroughly assess whether or not a given AI model and associated agentic AI system is appropriate for their desired use case. The field of agentic AI system evaluation is nascent, with more questions than answers.

### 4.2 Constraining the Action-Space and Requiring Approval

Some decisions may be too important for users to delegate to agents. Requiring a user to proactively authorize these actions, thus keeping a "human-in-the-loop," is a standard way to limit egregious failures of agentic AI systems.

### 4.3 Setting Agents' Default Behaviors

Model developers could significantly reduce the likelihood of the agentic system causing accidental harm by proactively shaping the models' default behavior according to certain design principles. One common-sense heuristic could be to err toward actions that are the least disruptive ones possible, while still achieving the agent's goal.

### 4.4 Legibility of Agent Activity

The more a user is aware of the actions and internal reasoning of their agents, the easier it can be for them to notice that something has gone wrong and intervene. Current language model-based agentic systems can produce a trace of their reasoning in natural language (a so-called "chain-of-thought").

### 4.5 Automatic Monitoring

Users or system deployers can set up a second "monitoring" AI system that automatically reviews the primary agentic system's reasoning and actions to check that they're in line with expectations given the user's goals. Such automated monitors operate at a speed and cost that human monitoring cannot hope to match.

### 4.6 Attributability

In cases where preventing intentional or unintentional harms is infeasible, it may still be possible to deter harm by making it likely that the user would have it traced back to them. One idea is to have each agentic AI instance assigned a unique identifier, similar to business registrations.

### 4.7 Interruptibility and Maintaining Control

Interruptibility (the ability to "turn an agent off"), while crude, is a critical backstop for preventing an AI system from causing accidental or intentional harm. System deployers could be required to make sure that a user can always activate a graceful shutdown procedure for its agent at any time.

## 5. Indirect Impacts from Agentic AI Systems

### 5.1 Adoption Races

There may be significant pressure for competitors to adopt agentic AI systems without properly vetting those systems' reliability and trustworthiness.

### 5.2 Labor Displacement and Differential Adoption Rates

Agentic AI systems appear likely to have a more substantive impact on workers, jobs, and productivity than static AI systems.

### 5.3 Shifting Offense-Defense Balances

Some tasks may be more susceptible to automation by agentic AI systems than others. This asymmetry is likely to undermine many current implicit assumptions that undergird harm mitigation equilibria in our society.

### 5.4 Correlated Failures

A particular risk arises when a large number of AI systems all fail at the same time, or all fail in the same way. These correlated errors can occur due to "algorithmic monoculture."

## 6. Conclusion

Increasingly agentic AI systems are on the horizon, and society may soon need to take significant measures to make sure they work safely and reliably. We hope that scholars and practitioners will work together to determine who should be responsible for using what practice, and how to make these practices reliable and affordable for a wide range of actors.

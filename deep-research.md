
## How We Built Our Multi-Agent Research System
*Published Jun 13, 2025*

Our Research feature uses multiple Claude agents to explore complex topics more effectively. It can search across the web, Google Workspace, and other integrations to accomplish complex tasks. Moving this multi-agent system from prototype to production taught us critical lessons about system architecture, tool design, and prompt engineering.

A multi-agent system consists of multiple agents (LLMs autonomously using tools in a loop) working together. Our Research feature has a lead agent that plans the process based on user queries, then creates parallel subagents that search for information simultaneously. This post breaks down the principles that worked for us—we hope they help you build your own multi-agent systems.

## Benefits of a Multi-Agent System
Research work is open-ended; you can’t hardcode a fixed path. As people conduct research, they continuously update their approach based on discoveries. That unpredictability makes AI agents well-suited for research tasks because they must pivot or explore tangents as investigations unfold. Linear, one-shot pipelines can’t do this.

The essence of search is compression: distilling insights from a vast corpus. Subagents enable compression by operating in parallel with their own context windows, exploring different aspects simultaneously, and providing separation of concerns for tools, prompts, and trajectories.

Once intelligence reaches a threshold, multi-agent systems become vital for scaling performance. Collective intelligence mirrors how human societies unlock new capability through coordination. Even generally intelligent agents face limits individually; coordinated groups can accomplish far more.

### Performance Findings
- Multi-agent Research (Claude Opus 4 lead + Claude Sonnet 4 subagents) outperformed a single-agent Claude Opus 4 by **90.2%** on our internal research eval—e.g., it correctly listed every board member for Information Technology S&P 500 companies by decomposing the problem across subagents.
- Token usage explains **80%** of performance variance on the BrowseComp evaluation; the other key levers are tool-call count and model choice. Separate agent context windows effectively scale reasoning capacity beyond a single context.
- Upgrading to Claude Sonnet 4 gives a larger gain than doubling the token budget on Claude Sonnet 3.7, underscoring how architecture and model choice interact.

### Tradeoffs
- Multi-agent systems consume substantially more tokens: ~4× chat usage for single agents and ~15× for multi-agent flows. They must be applied to high-value tasks to justify the cost.
- Domains requiring shared context or tightly coupled dependencies (e.g., many coding workflows) are less suited until agents coordinate better in real time.
- Multi-agent systems shine when tasks benefit from heavy parallelization, require more information than a single context window, or demand coordination across many tools.

## Architecture Overview for Research
Our system uses an orchestrator-worker pattern: a lead agent coordinates the process and delegates to specialized subagents that operate in parallel.

### Workflow
1. A user submits a query.
2. The lead agent analyzes the query, plans the approach, and saves the plan to Memory so context persists even above 200k tokens.
3. It spawns subagents with specific research tasks. Each subagent independently performs web searches, evaluates results with interleaved thinking, and returns findings.
4. The lead agent synthesizes results, determines whether more research is necessary, and (if needed) spins up additional or refined subagents.
5. Once sufficient information is gathered, the system hands everything to a CitationAgent that verifies citations before returning the final answer to the user.

### Dynamic Search vs. Static Retrieval
Traditional Retrieval Augmented Generation (RAG) uses static retrieval: fetch similar chunks once, then answer. Our architecture performs multi-step search that dynamically finds relevant information, adapts to new findings, and analyzes results to craft high-quality answers.

## Prompt Engineering and Agent Guidance
Multi-agent systems introduce rapid growth in coordination complexity. Early agents over-spawned, wandered, or distracted each other. Prompt engineering was our main lever for improving behavior. Key principles:

- **Think like your agents.** We built Console simulations using production prompts and tools, then watched agents step-by-step to expose failure modes (continuing when already done, verbose queries, wrong tools). Developing an accurate mental model made impactful prompt tweaks obvious.
- **Teach the orchestrator how to delegate.** Subagents need objectives, output formats, tool guidance, and boundaries. Without detailed task descriptions, agents duplicate work or miss gaps. Moving beyond vague instructions (e.g., “research the semiconductor shortage”) prevented overlaps like multiple agents repeating the same 2025 supply-chain search.
- **Scale effort to query complexity.** We embedded heuristics: simple fact-finding = one agent with 3–10 tool calls; direct comparisons = 2–4 subagents with 10–15 calls each; complex research = 10+ specialized subagents. This prevented overinvestment in simple tasks.
- **Prioritize tool ergonomics.** Agents must pick the right tool, especially with MCP servers exposing unfamiliar interfaces. We instructed agents to inspect available tools, align usage with user intent, search broadly first, and prefer specialized tools when applicable. Clear tool descriptions prevent derailment.
- **Let agents improve themselves.** Claude 4 models act as prompt engineers. Given a failure mode, they diagnose causes and iterate prompts. We even built a tool-testing agent that repeatedly exercises flawed MCP tools, then rewrites descriptions—cutting future task time by ~40%.
- **Start wide, then narrow.** Agents defaulted to overly specific searches. We prompted them to begin with short, broad queries, evaluate available information, then progressively focus.
- **Guide the thinking process.** Extended thinking mode serves as a scratchpad. Lead agents plan approaches, tool choices, complexity, and subagent roles. Subagents plan, evaluate tool results with interleaved thinking, identify gaps, and refine subsequent queries.
- **Parallelize tool calls.** We introduced parallelism at two levels: (1) the lead agent spawns 3–5 subagents concurrently, and (2) subagents use 3+ tools in parallel. Complex research time fell by up to 90%.
- **Instill heuristics, not rigid rules.** We encoded strategies from skilled human researchers—task decomposition, source evaluation, adjusting search approaches, balancing depth vs. breadth—and added guardrails plus fast iteration loops with observability.

## Effective Evaluation of Agents
Evaluating multi-agent systems is challenging because they rarely follow the same path twice. We focus on whether agents achieve the right outcomes while following reasonable processes.

- **Start small.** Early prompt tweaks can move success from 30% to 80%. We began with ~20 representative queries, iterated quickly, and only later scaled evals.
- **Use LLM judges wisely.** Research outputs are free-form. An LLM judge scored each output on factual accuracy, citation accuracy, completeness, source quality, and tool efficiency. A single prompt returning 0.0–1.0 scores plus pass/fail aligned best with human judgment and scaled to hundreds of cases.
- **Keep humans in the loop.** Human testers surfaced edge cases such as agents preferring SEO content farms over authoritative sources. Adding source-quality heuristics fixed these issues.
- **Expect emergent behavior.** Small orchestrator changes can cascade through subagents. Prompts must define collaboration frameworks, division of labor, and effort budgets. Success relies on careful prompting, solid heuristics, observability, and tight feedback loops (see the prompts in our Cookbook).

## Production Reliability and Engineering Challenges
Agentic systems magnify seemingly small issues, so we invested heavily in reliability:

- **Agents are stateful and errors compound.** Agents run for long stretches and maintain state across tool calls. We built durable execution, retry logic, and checkpoints so agents resume after failures. Agents are informed when tools fail so they can adapt.
- **Debugging needs new techniques.** Non-determinism makes root causes opaque. Full production tracing plus monitoring of decision patterns (without inspecting private conversation contents) helped us diagnose search-query quality, tool selections, and failure points.
- **Deployment requires coordination.** Agents may be mid-process during deploys. We use rainbow deployments to gradually shift traffic, keeping old and new versions running simultaneously to avoid interrupting live agents.
- **Synchronous execution creates bottlenecks.** Lead agents currently wait for each batch of subagents to finish. Asynchronous execution would unlock more parallelism but complicates coordination, state consistency, and error propagation. As tasks grow longer and more complex, we expect async will be worth the investment.

## Conclusion
The last mile can become most of the journey. Code that works locally needs significant engineering to become reliable in production. In agentic systems, minor issues can push agents onto entirely different trajectories. Despite the challenge, multi-agent systems deliver outsized value on open-ended research tasks—helping users uncover business opportunities, navigate healthcare, debug technical issues, and save days of work. With careful engineering, thorough testing, thoughtful prompt and tool design, and close collaboration across teams, multi-agent research systems can operate reliably at scale.

The Clio embedding plot (not shown here) highlights how people use Research today: developing specialized software systems (10%), professional and technical content (8%), business growth strategies (8%), academic research support (7%), and verifying information about people, places, or organizations (5%).

### Acknowledgements
Written by Jeremy Hadfield, Barry Zhang, Kenneth Lien, Florian Scholz, Jeremy Fox, and Daniel Ford. This work reflects contributions from many Anthropic teams, with special thanks to the apps engineering team and our early users for their feedback.

## Appendix: Additional Tips for Multi-Agent Systems
- **End-state evaluation for stateful workflows.** When agents mutate persistent state across many turns, evaluate final states instead of every step. Break long workflows into checkpoints that confirm specific state changes rather than policing each intermediate action.
- **Manage long-horizon conversations.** Conversations can span hundreds of turns; context windows eventually overflow. Summarize completed phases, store essential information in external memory, and let agents spawn fresh subagents with clean contexts. Retrieve stored plans when needed to maintain continuity.
- **Use artifact outputs to avoid “telephone” loss.** Allow subagents to write results directly to external systems, then pass lightweight references back to the coordinator. This preserves fidelity, reduces token overhead, and works especially well for structured artifacts like code, reports, or visualizations.

Explore further resources, courses, and newsletters to keep up with product updates, how-tos, community spotlights, and more.


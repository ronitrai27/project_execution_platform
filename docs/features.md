## Github_subagent 
get_pull_requests_summary (PR Review Status, Stale PRs, Blockers)
get_issues
get_contributor_activity (Developer Velocity & Contribution Tracking)
get_release_and_ci_status (Release Tags, Milestones & Build Health)



## finding from web 
In 2026, agentic AI architecture has matured into a sophisticated ecosystem centered on specialization, modularity, and intelligent orchestration. The shift is away from monolithic, all-purpose models toward "router-based" multi-agent systems that delegate tasks to expert sub-agents.

Core Architectural Patterns
Current production-grade systems typically employ one or more of these four primary patterns:

Router + Specialists: An orchestrator agent analyzes the incoming request and dynamically routes it to the most capable "specialist" agent. This is preferred for high-value tasks where domain-specific expertise outweighs the cost of routing.
Hierarchical Orchestrator: A supervisor agent manages multiple executor agents, monitoring outputs, handling failures, and performing aggregation. This is the standard for long-horizon or highly complex, multi-stage workflows.
Sequential Pipeline: A deterministic flow where the output of one agent serves as the input for the next (e.g., Research Agent → Critic Agent → Writer Agent).
Single Agent + Tools: While multi-agent systems are trending, industry guidance (e.g., from Anthropic) emphasizes that many tasks are still best served by a single well-prompted agent with access to a robust library of tools.
Skill Selection and Definition
In 2026, "Skills" have become a standardized, portable unit of agentic capability:

Standardized Format: Most platforms now support the SKILL.md format (introduced in 2025), which packages instructions, constraints, and context into a file that an agent reads to understand how to handle a specific task.
Separation from Plugins: Unlike plugins (which typically execute code or API calls), a "skill" primarily instructs the agent on how to think and process a specific domain, making it a higher-level abstraction for guiding agent behavior.
The "Skill" Ecosystem: There are now large registries (such as the Skillselion catalog) tracking tens of thousands of reusable skills, allowing developers to "plug and play" capabilities into their agentic workflows.
Routing Mechanisms
Routing logic has evolved significantly beyond legacy intent classifiers:

LLM-Based Routing: Modern routers use the semantic reasoning capabilities of LLMs to interpret user context, allowing for dynamic, zero-shot routing without extensive training data.
Agent-as-a-Router: Emerging research (e.g., Agent-as-a-Router frameworks) treats model selection as a continuous feedback loop. These systems use memory modules and execution-grounded experience to optimize routing decisions for cost, latency, and performance.
Hierarchical & Auction-Based Routing: Advanced systems may use hierarchical (supervisor-led) or even auction-based mechanisms (where agents "bid" to perform a task) to manage complex, multi-domain problem-solving.
Key Frameworks & Best Practices
Orchestration Frameworks: LangGraph (popular for its state-machine, cyclic-graph approach) and CrewAI (favored for role-based, hierarchical team structures) remain industry standards for implementing these architectures.
The "95/5" Rule: Engineering efforts are heavily focused on the "unseen" 95% of agent development—handling state management, error cascading, cost optimization, and human-in-the-loop oversight—rather than just the 5% spent on API calls.
Reliability: Production leaders are increasingly focused on state-driven transitions and checkpointing, which allow systems to pause, recover from failures, and maintain visibility into the routing decision-making process.
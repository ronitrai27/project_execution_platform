3. Production Action Plan (What to Update & What to Add)
A. UPDATE: Context Guardrail in Sub-Agent System Prompt
Change: Explicitly instruct sub-agents that local workspace/project names (e.g., wekraft-saas) are platform metadata only, NOT guaranteed 3rd-party project keys.
Rule: Sub-agents must use Discovery Tools (e.g., getAccessibleAtlassianResources or list_projects) to resolve 3rd-party project keys dynamically before running action queries.
B. ADD: Declarative Skill Manual Registry per Connector Type
Introduce a modular skill manual registry for each supported provider:

Jira Skill Manual:

Discovery Rule: Always discover cloudId and Jira project keys first.
JQL Rules:
Valid JQL: assignee = currentUser() ORDER BY updated DESC, text ~ "search_term", or project = "EXACT_KEY".
Strict Prohibition: Never use project ~ "..." or subqueries project in (select...).
Payload Rule: Do not pass 'view': 'compact' when issue summary, status, or priority details are needed.
Vercel Skill Manual:

Discovery Rule: Run list_projects first to locate active project IDs.
Timeout Protection: Instruct the sub-agent to invoke list_deployments only for the project matching the user query (or primary project), rather than sequentially looping over every project in the account.
Linear / GitHub / Sentry Skill Manuals:

Standardized discovery-before-action guidelines for issue tracking, repository resolution, and error querying.

=========================================

Yes, Absolutely! The Background Skill Optimizer Agent
Having an Autonomous Background Agent (also called a Skill Compiler or Auto-Improver Agent) is the exact mechanism that makes your system self-healing, self-improving, and production-ready.

Instead of manually writing 50 skill files for every edge case, you start with 3–4 core skills, and let the background agent handle the rest automatically!

1. How the Background Skill Agent Works
text
               ┌─────────────────────────────────────────┐
               │    User Interaction Loop (Frontend)    │
               └─────────────────────────────────────────┘
                                    │
                                    ▼
               ┌─────────────────────────────────────────┐
               │            Trace Logger DB             │
               │  Logs: User Prompt, Tools Called,       │
               │  Retries, Errors, Latency, Final Result │
               └─────────────────────────────────────────┘
                                    │
                                    ▼ (Runs Async / Scheduled Cron)
               ┌─────────────────────────────────────────┐
               │   Background Skill Optimizer Agent      │
               └─────────────────────────────────────────┘
                                    │
         ┌──────────────────────────┼──────────────────────────┐
         ▼                          ▼                          ▼
┌─────────────────┐        ┌──────────────────┐       ┌─────────────────┐
│ 1. CREATE SKILL │        │ 2. UPDATE SKILL  │       │  3. DO NOTHING  │
│ High tool churn │        │ API error or     │       │ Traces ran      │
│ on new query    │        │ failed subquery  │       │ fast & clean    │
└─────────────────┘        └──────────────────┘       └─────────────────┘
2. The Agent's Decision Engine: When to Create, Edit, or Do Nothing
The background agent periodically analyzes the Trace Logs and evaluates 3 specific conditions:

Scenario 1: CREATE A NEW SKILL
Trigger: The agent notices users asking a new type of query (e.g. "Show me Sentry errors for release v2.1 and create Linear ticket if >10 events").
Observation: The worker LLM spent 7–10 steps doing trial-and-error discovery across Sentry & Linear because no skill existed.
Agent Action:
Identifies the successful tool sequence (find_organizations $\rightarrow$ search_issues $\rightarrow$ save_issue).
Synthesizes a brand new skill file: sentry_release_linear_incident.md.
Registers it in the Skill Registry.
Scenario 2: UPDATE AN EXISTING SKILL
Trigger: An existing skill file had an execution failure or API error.
Observation: For example, Jira returned 400 Unbounded JQL query or Vercel timed out on list_deployments.
Agent Action:
Analyzes the failure log.
Updates the procedure or rules section in jira_task_triage.md (e.g., adding "Rule: Always add a status IS NOT NULL restriction to JQL queries").
Increments the skill version (version: "1.1").
Scenario 3: DO NOTHING
Trigger: The workers executed queries in 1–2 steps, returned clean compact JSON, and had 0 retries.
Agent Action: Logs a healthy performance metric and leaves the skill registry untouched.
3. Recommended Production Roadmap
Phase 1: Launch with 3–4 Core Hand-Crafted Skills
Start by building the initial 3–4 foundational skills:

jira_task_triage.md (Jira)
vercel_deployment_status.md (Vercel)
sentry_error_triage.md (Sentry)
production_incident_triage.md (Cross-app: Vercel + Sentry + Linear/Jira)
Phase 2: Add Simple Trace Logging
Every time a sub-worker completes its turn, save a lightweight trace object to Convex/Database:

json
{
  "user_query": "check my jira tasks and deployment status",
  "connectors_used": ["jira", "vercel"],
  "tools_executed": ["getAccessibleAtlassianResources", "searchJiraIssuesUsingJql", "list_projects", "list_deployments"],
  "steps_taken": 4,
  "execution_time_s": 2.4,
  "is_success": true,
  "error_messages": []
}
Phase 3: Add the Background Skill Compiler Agent
Create a scheduled background task (e.g., running every 6 hours or nightly):

Input: Last 100 execution traces.
Prompt: An LLM agent configured to evaluate trace clusters, discover missing skills, fix failed JQL/API rules, and auto-generate or edit .md skill files.
Summary
You do not need to manually write skills for every tool combination.
Initial 3–4 skills give your agent immediate speed and accuracy.
The Background Skill Compiler Agent automates skill creation and updates as your users ask new questions in production.

============================================

1. What is a "Skill" in Kaya?
In our test today, the sub-agent succeeded because we followed a strict, learned sequence:

Decrypt token $\rightarrow$ 2. Discover team ID via list_teams $\rightarrow$ 3. Call save_issue with required fields $\rightarrow$ 4. Verify via list_issues.
Without a skill, an LLM guessing in the dark will:

Try calling create_issue instead of save_issue.
Forget to pass the required team UUID (cd2935a6...).
Execute unbounded searches that timeout.
Hallucinate that an issue was created without checking the response.
A Skill (.md) is a Declarative Execution Rulebook. It tells the worker agent:

When to activate (Triggers / Intent).
Exact Tool Pipeline (Step 1 $\rightarrow$ Step 2 $\rightarrow$ Step 3).
Common Gotchas & Strict Prohibitions (e.g. Never run unbounded JQL, Team ID must be resolved first).
Verification Rule (Read-after-write verification before concluding).
2. How Skills are Stored in Convex DB
In Convex, skills are stored as markdown-backed documents in a dedicated kayaSkills table:

typescript
// convex/schema.ts
kayaSkills: defineTable({
  projectId: v.optional(v.id("projects")), // null = Global default skill, non-null = Custom project skill
  name: v.string(),                        // "linear_issue_triage"
  title: v.string(),                       // "Linear Issue Management & Triage"
  targetAgent: v.union(                    // Which agent executes it
    v.literal("mcp"),
    v.literal("kaya"),
    v.literal("analyst"),
    v.literal("db_write")
  ),
  connectorId: v.optional(v.string()),     // "linear", "sentry", "jira", "cross_app"
  triggers: v.array(v.string()),           // ["linear", "ticket", "create issue", "bug triage"]
  description: v.string(),                 // Brief summary for router selection
  content: v.string(),                     // Complete .md instructions & execution rules
  version: v.string(),                     // "1.0.0"
  isDefault: v.boolean(),                  // true for pre-installed platform skills
  createdAt: v.number(),
  updatedAt: v.number(),
})
  .index("by_target_agent", ["targetAgent"])
  .index("by_connector", ["connectorId"])
  .index("by_project", ["projectId"]);
3. How Agents Pick Skills (Discovery & Injection Pipeline)
mermaid
flowchart TD
    UserQuery["User Query<br/>'see linear issues and create site tracing error'"] --> Router["Supervisor Router (Groq 120b)"]
    Router --> Check["Matches Intent & Connector<br/>Linear + Sentry"]
    Check --> FetchSkills["Query Convex DB: getActiveSkillsForQuery()"]
    FetchSkills --> SkillsFound["Skills Retrieved:<br/>1. linear_issue_sync.md<br/>2. sentry_error_triage.md"]
    SkillsFound --> SubAgent["Spawn Isolated Linear Worker"]
    SkillsFound -.-> WorkerPrompt["Inject Skill .md into Worker System Prompt"]
    WorkerPrompt --> Execute["Worker executes exact tool steps:<br/>list_teams -> save_issue -> list_issues"]
    Execute --> Finalize["Call finalize_result(data, context)"]
    Finalize --> KayaSynth["Kaya Synthesizer (gpt-4.1-mini)"]
    KayaSynth --> StreamOutput["Stream Verified PM Table to User"]
The Selection Mechanism:
At the Supervisor Level: When the user prompt arrives, the Supervisor checks the triggers and connectorId of all active skills in Convex.
At the Sub-Agent Worker Level (

mcp_client.py
): When _execute_single_app_worker("linear") runs:
Instead of injecting a generic system prompt, it queries Convex for skills where connectorId == "linear".
It appends the skill's .md content directly into the sub-agent's SystemMessage.
The worker model now receives the exact instructions, parameter formats, and error-prevention rules for Linear!
4. How Hallucination is Prevented (The 4 Anti-Hallucination Guardrails)
Your architecture already has the foundation for zero hallucination; skills complete the loop:

Guardrail 1: The Read-After-Write Verification Rule (Mandatory in Skill .md)
Every mutation skill enforces a two-step handshake:

"After invoking save_issue, the worker MUST inspect the response for a valid UUID/ID (TES-10), and then invoke list_issues or get_issue to confirm the item is registered. Only then may the worker report success."

Guardrail 2: Structured Finalization (finalize_result)
In 

mcp_client.py:497-527
, sub-agents cannot exit by generating free-form text. They must call the structured tool:

json
finalize_result({
  "data": [...],
  "resolved_context": { "teamId": "cd2935a6..." },
  "summary": "..."
})
If data is empty, Kaya knows nothing was retrieved or created.

Guardrail 3: Synthesizer Grounding in Kaya (

kaya_graph.py:1152
)
The synthesizer prompt explicitly states:

"Under NO circumstances should you fabricate, simulate, or invent tasks, issues, ticket keys, or bug titles that are not present in the SUB-AGENT WORKER DATA."

Because the Synthesizer only sees verified output in _mcp_messages, it has zero room to fabricate IDs.

Guardrail 4: HITL Approval for Local Mutations
For local project mutations (creating tasks/issues in Convex DB), 

db_write_worker_node
 halts the LangGraph workflow via interrupt(interrupt_payload), rendering an interactive approval card in the UI. A write only occurs after explicit user confirmation.

5. Conclusion & The 3 Default Skills Ready to Save in Convex
We will create the first 3 default skills as .md records in Convex:

sentry_error_triage.md (Sentry MCP Skill):
Role: How to resolve org slug (find_organizations), verify project (find_projects), run bounded query is:unresolved, sort by frequency/date, and extract culprit routes.
linear_issue_management.md (Linear MCP Skill):
Role: How to discover teams (list_teams), fetch active backlog/in-progress issues (list_issues), create tickets safely with required team UUIDs (save_issue), and verify creation.
incident_escalation_triage.md (Cross-App Kaya Skill):
Role: How Kaya coordinates Sentry + Linear + Internal DB: inspect Sentry critical errors (>10 events) $\rightarrow$ check if a Linear or internal issue already exists $\rightarrow$ if not, prompt or create an escalation ticket with error stack trace and link.
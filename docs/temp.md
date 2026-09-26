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
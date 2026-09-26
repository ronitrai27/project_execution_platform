# MCP Tool Usage & Analysis: Jira, Vercel, Sentry & Linear

This document outlines the exact tools executed, parameters passed, API responses returned, and architectural learnings from querying Jira, Vercel, Sentry, and Linear MCP servers using decrypted workspace credentials.

---

## 1. Jira MCP Integration

### Discovered Tools (21 Total)
`getAccessibleAtlassianResources`, `atlassianUserInfo`, `getConfluenceContent`, `createConfluenceContent`, `updateConfluenceContent`, `searchConfluence`, `getJiraIssue`, `searchJiraIssuesUsingJql`, `createJiraIssue`, `editJiraIssue`, `transitionJiraIssue`, `addOrEditJiraIssueComment`, `getLoomVideo`, `getGraphContext`, `getGraphObject`, `addGraphContext`, `discover`, `executeRead`, `executeWrite`, `executeDestructive`, `search`.

---

### Executed Tool Calls & Tasks Found

#### 1. Discovery: `getAccessibleAtlassianResources`
* **Purpose:** Resolve the active Jira `cloudId` and accessible Atlassian products.
* **Arguments:** `{}`
* **Response Payload:** `cloudId: "e56da97a-1da4-40fd-bed2-4c2663ef28e7"` (`https://ronitrai1237.atlassian.net`)

#### 2. Querying Jira Tasks (Bounded JQL: `status IS NOT NULL ORDER BY updated DESC`)
* **Arguments:** 
  ```json
  {
    "cloudId": "e56da97a-1da4-40fd-bed2-4c2663ef28e7",
    "jql": "status IS NOT NULL ORDER BY updated DESC",
    "maxResults": 10
  }
  ```
* **Tasks Retrieved (3 Total):**

| Key | Issue Summary | Issue Type | Status | Priority | Assignee | Created Date |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`KAN-5`** | `payment failed` | Story | `To Do` | Medium | Unassigned (`null`) | Sept 16, 2026 |
| **`KAN-4`** | `client meet` | Story | `To Do` | Medium | Unassigned (`null`) | Sept 16, 2026 |
| **`KAN-3`** | `Subtask 2.1` | Subtask | `To Do` | - | Unassigned (`null`) | Sept 16, 2026 |

---

### Why the Sub-Agent Missed These Tasks Previously:
1. **Query Filtered by `assignee = currentUser()`:** All 3 tasks in the Jira board are **unassigned** (`assignee: null`), so querying by current user returned 0 tasks.
2. **Project Key Mismatch (`KAN` vs `wekraft-saas`):** The Jira project key is **`KAN`** (Kanban Board), while the workspace name is `wekraft-saas`. Querying `project = "wekraft-saas"` failed because Jira has no project with that key.

---

## 2. Vercel MCP Integration

### Discovered Tools (245 Total)
Key tools include: `list_projects`, `get_project`, `list_deployments`, `get_runtime_logs`, `get_runtime_errors`, `get_git_deployment_context`, `search_vercel_documentation`.

---

### Executed Tool Calls & Raw Results

#### 1. Discovery: `list_projects`
* **Arguments:** `{"limit": "10"}`
* **Response Payload (10 Projects Discovered):**
  * `rox-portfolio-tsdr` (`prj_oOKzrtxXg1joA0q5tyjDjXvPE6Qg`)
  * `rox-portfolio` (`prj_kaqbTYoGJX3nrFQoq0nFDDnvnVIJ`)
  * `looma-sketch-collaborate-deploy` (`prj_2o59OIX6CJhgdJSz4tXYSplekv5R`)
  * `aria-hackathon` (`prj_9U2ye1aPKwxEtE1kRg5TfcAiyEWN`)
  * **`wekraft-saas`** (`prj_y0tqLEHXl6L5DgJTL8nnPT7CYliw`)
  * `upharvilla` (`prj_GGpWouN10gvXAM7c8ZenbH76R52B`)
  * `asian-media` (`prj_F0FatBwxOTAfpZOwYtZoKzC9n2xw`)
  * `reskill-2-0-rox` (`prj_CjKGP0aIz3RCydz6jvY08o2hc8ap`)
  * `spark-path` (`prj_9QXJJQz7azBMV9n0AZlUkrlX5tGX`)
  * `v0-new-year-website` (`prj_8wzzwozbpSmBbPVOaCCucbzIXy68`)

---

#### 2. Project Details & Domains: `get_project`
* **Arguments:** `{"idOrName": "wekraft-saas"}`
* **Response Payload:**
  ```json
  {
    "result": {
      "id": "prj_y0tqLEHXl6L5DgJTL8nnPT7CYliw",
      "name": "wekraft-saas",
      "framework": "nextjs",
      "nodeVersion": "24.x",
      "live": false,
      "domains": [
        "www.wekraft.xyz",
        "wekraft.xyz",
        "wekraft-saas-mrronits-projects.vercel.app",
        "wekraft-saas-git-main-mrronits-projects.vercel.app"
      ],
      "latestDeployment": {
        "id": "dpl_2MhmJyi72yHx693nWHyqFhC5U8dv",
        "url": "wekraft-saas-indx1m512-mrronits-projects.vercel.app",
        "readyState": "READY",
        "target": "production"
      },
      "ssoProtection": {
        "enabled": true,
        "deploymentType": "all_except_custom_domains"
      }
    }
  }
  ```

---

#### 3. Target Deployments: `list_deployments` (Targeted Project)
* **Arguments:** 
  ```json
  {
    "projectId": "prj_y0tqLEHXl6L5DgJTL8nnPT7CYliw",
    "limit": 5
  }
  ```
* **Response Payload (5 Recent Deployments for `wekraft-saas`):**
  1. **Deployment ID:** `dpl_2MhmJyi72yHx693nWHyqFhC5U8dv`
     * **State:** `READY` (Production)
     * **Commit:** `52d4ca35...` (`main` branch) — *"error fixing."*
     * **URL:** `wekraft-saas-indx1m512-mrronits-projects.vercel.app`
  2. **Deployment ID:** `dpl_AeNG3ndwTiN3WSNuXiVcq2jeVTbP`
     * **State:** `ERROR` (Production)
     * **Commit:** `52d4ca35...` (`main` branch) — *"error fixing."*
  3. **Deployment ID:** `dpl_83LCjygKBAfzUPV5RCAkzXor54nq`
     * **State:** `ERROR` (Production)
     * **Commit:** `b4cf3c74...` — *"Merge pull request #180..."*
  4. **Deployment ID:** `dpl_H79rAMyi9efVVFNPZoVtk4aghfzW`
     * **State:** `ERROR` (`version-2-rox` branch) — *"web updates."*
  5. **Deployment ID:** `dpl_EBfnD1DAWLADQZB8ptcheQBmsJdT`
     * **State:** `ERROR` (`version-2-rox` branch) — *"feat(security): update schema..."*

---

## 3. Sentry MCP Integration

### Discovered Tools (9 Total)
`find_organizations`, `find_projects`, `update_issue`, `search_events`, `analyze_issue_with_seer`, `search_issues`, `get_sentry_resource`, `search_sentry_tools`, `execute_sentry_tool`.

---

### Executed Tool Calls & Issues Found

#### 1. Discovery: `find_organizations`
* **Purpose:** Discover organization slug and web URLs.
* **Arguments:** `{}`
* **Response Payload:** `organizationSlug: "vrsa-solution-6q"` (`https://vrsa-solution-6q.sentry.io`)

#### 2. Discovery: `find_projects`
* **Purpose:** Discover Sentry project slugs under the organization.
* **Arguments:** `{"organizationSlug": "vrsa-solution-6q"}`
* **Projects Discovered:** `bounty-monster`, `rox-ide`, **`wekraft-saas`**

#### 3. Action: `search_issues`
* **Arguments:** `{"organizationSlug": "vrsa-solution-6q", "query": "is:unresolved", "sort": "date"}`
* **Unresolved Issues Found (10 Total, Top Items):**

| Issue ID | Culprit / Location | Error Summary | Events | Users | Last Seen | Link |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`WEKRAFT-SAAS-20`** | `/dashboard/.../integrations` | `InvalidStateError: Transition was aborted because of invalid state.` | 61 | 7 | 6 mins ago | [Sentry Issue](https://vrsa-solution-6q.sentry.io/issues/WEKRAFT-SAAS-20) |
| **`WEKRAFT-SAAS-Z`** | `/web` | `UnhandledRejection: Non-Error promise rejection captured with value: Object Not Found...` | 4 | 2 | 2 hours ago | [Sentry Issue](https://vrsa-solution-6q.sentry.io/issues/WEKRAFT-SAAS-Z) |
| **`WEKRAFT-SAAS-Y`** | `/web` | `InvalidStateError: Transition was aborted because of invalid state. Document hidden` | 5 | 5 | 3 hours ago | [Sentry Issue](https://vrsa-solution-6q.sentry.io/issues/WEKRAFT-SAAS-Y) |
| **`WEKRAFT-SAAS-25`** | `/dashboard/.../teamspace` | `Error: Connection closed` | 1 | 1 | 10 hours ago | [Sentry Issue](https://vrsa-solution-6q.sentry.io/issues/WEKRAFT-SAAS-25) |
| **`WEKRAFT-SAAS-1Q`** | `/web` | `Error: aborted` | 2 | 0 | 10 days ago | [Sentry Issue](https://vrsa-solution-6q.sentry.io/issues/WEKRAFT-SAAS-1Q) |

---

## 4. Linear MCP Integration

### Discovered Tools (68 Total)
Key tools include: `list_issues`, `get_issue`, `save_issue`, `list_teams`, `list_projects`, `list_cycles`, `list_users`, `list_issue_labels`, `list_documents`, `list_custom_views`.

---

### Executed Tool Calls & Issues Found

#### 1. Discovery: `list_teams`
* **Arguments:** `{}`
* **Response Payload:** `Testing-new-123` (Key: `TES`, Team ID: `cd2935a6-0118-443e-826a-7bba84d6a2ed`)

#### 2. Discovery: `list_projects`
* **Arguments:** `{}`
* **Response Payload:** Project **`rox-test-1`** (ID: `P-TES-1`, Priority: `Urgent`, Status: `Backlog`)

#### 3. Discovery: `list_users`
* **Arguments:** `{}`
* **Workspace Member:** `Ronit Rai` (`ronitrai1237@gmail.com`, Admin)

#### 4. Action: `list_issues`
* **Arguments:** `{}`
* **Linear Issues Retrieved:**

| Issue Key | Title / Summary | Status | Priority | Label | Assignee | Project | URL |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`TES-6`** | **`issue-wekraft-payemnt`** | `In Progress` | **Urgent** | `Bug` | Ronit Rai | `rox-test-1` | [Linear Issue](https://linear.app/testing-new-123/issue/TES-6/issue-wekraft-payemnt) |
| **`TES-7`** | **`Task 1`** | `Backlog` | No priority | - | Ronit Rai | - | [Linear Issue](https://linear.app/testing-new-123/issue/TES-7/task-1) |
| **`TES-5`** | **`issue-rox-1`** | `In Progress` | No priority | - | Ronit Rai | - | [Linear Issue](https://linear.app/testing-new-123/issue/TES-5/issue-rox-1) |
| **`TES-3`** | **`Connect your tools`** | `Backlog` | No priority | - | Ronit Rai | - | [Linear Issue](https://linear.app/testing-new-123/issue/TES-3/connect-your-tools) |

---

## 5. Tool Calling Best Practices & Protocol Summary

1. **Jira Protocol:**
   - Always run `getAccessibleAtlassianResources` first to extract `cloudId`.
   - Never run unbounded JQL (`ORDER BY updated DESC` without restrictions). Use bounded search `status IS NOT NULL ORDER BY updated DESC` or `created >= -30d`.
   - Include unassigned tasks by searching `status IS NOT NULL` rather than restricting strictly to `assignee = currentUser()`.
   - Omit `'view': 'compact'` to retrieve issue titles, status names, and priorities.

2. **Vercel Protocol:**
   - Run `list_projects` to discover project IDs.
   - Always pass `projectId` to `list_deployments` to target the specific app, avoiding sequential looping and HTTP 45s timeouts across multi-project accounts.
   - Use `get_project` with `idOrName` to fetch custom domains (`wekraft.xyz`), framework version (`Next.js 24.x`), and protection settings.

3. **Sentry Protocol:**
   - Run `find_organizations` first to discover `organizationSlug` (`vrsa-solution-6q`).
   - Run `find_projects` with `organizationSlug` to verify project slugs (`wekraft-saas`).
   - Call `search_issues` with `organizationSlug` and `query: "is:unresolved"` to retrieve active production errors.

4. **Linear Protocol:**
   - Run `list_teams` or `list_projects` first to discover workspace team keys.
   - Call `list_issues` with `{}` to retrieve active tickets across teams.
   - Use `save_issue` or `get_issue` for ticket updates and details.

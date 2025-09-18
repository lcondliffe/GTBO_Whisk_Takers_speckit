Azure DevOps AI Agent

## 1. Summary

An AI agent that performs automated, line-level reviews on Azure DevOps pull requests (PRs). Triggered on PR creation/updates, the agent analyzes diffs, applies org-specific standards, and posts actionable comments with severity, rationale, and suggested fixes. It integrates with Azure DevOps via webhooks or pipeline steps, is orchestrated by Python Azure Functions, and uses Azure Agent Service (AI Foundry) with an Azure DevOps MCP Server for context and actions. Feature flags control agent behaviour such as whether feedback is posted to PRs or stored for dry-run analysis.

## 2. Rationale

Manual code reviews are time-consuming and can be inconsistent. An automated AI agent can provide immediate, standardized feedback, freeing up developer time to focus on more complex problems. This will lead to higher code quality, better adherence to coding standards, and increased team velocity. The underlying architecture can also serve as a reusable pattern for other event-driven agentic workflows in Azure DevOps that utilise tools for context gathering and action taking.

## 3. User Stories

- As a developer, I want to receive automated, line-specific feedback on my pull requests so that I can quickly identify and fix issues.
- As a developer, I want the agent to enforce our team's coding standards and best practices automatically to ensure consistency across the codebase.
- As a DevOps engineer, I want to seamlessly integrate the AI review agent into our existing Azure DevOps pipelines OR be triggered by a PR created event webhook.
- As a Security engineer, I want automated review of new code entering corporate codebases.
- As a reviewer, I want a high-level summary of the PR's changes so I can quickly understand its impact and focus my attention.

## 4. Acceptance Criteria

- The AI agent must activate on pull request creation (and be prepared to also activate on updates where existing commentary may exist) in configured Azure DevOps repositories.
- The agent must post one comment that provides an overview of the PR and highlights any key areas for reviewers.
- If there are critical or major issues, this is where the agent will use the PR comment MCP tool to create `suggestion` blocked comments with suggested fixes.
- Threaded, line-level comments will be posted for "Minor", "Nitpick", and "Info" level issues.
- In-line comments must include:
    Severity: Blocker, Critical, Major, Minor, Info
    Emoji mapping: Blocker ❌, Critical 🔥, Major ⚠️, Nitpick 💡, Info 📝
    Concise rationale and code suggestions (diff or code block).
- The agent must not post duplicate comments on subsequent runs for the same unchanged issue.
- The solution must be deployed using an Azure Agent Service, orchestrated by Python Azure Functions.

## 5. Technical Details

The solution will be built on the following technical stack:
- **AI Service**: An agent hosted on Azure Agent Service (AI Foundry) will perform the code analysis.
- **Orchestration**: Python-based Azure Functions will be triggered by Azure DevOps webhooks (PR create/update) to manage the workflow, fetch PR data, and invoke the AI agent.
- **Context Provider**: An Azure DevOps MCP (Microsoft Cloud Platform) Server will provide additional context to the agent, such as coding standards and repository history.
- **MCP Integration**: An Azure DevOps MCP (Model Context Protocol) Bridge provides standardized communication between the AI agent and Azure DevOps APIs. Spawning and managing the Azure DevOps MCP server as a subprocess of the orchestrator function. Implements JSON-RPC 2.0 communication protocol for MCP message exchange and provides synchronous wrappers for async MCP operations.

MCP Server:
https://github.com/microsoft/azure-devops-mcp

The workflow is as follows:
1. A user creates or updates a pull request in Azure DevOps.
2. An Azure DevOps webhook OR pipeline step issuing a curl command triggers a Python Azure Function.
3. The Azure Function gets PR diff and meta from the Azure DevOps REST API and starts an Azure agent with this as context. MCP available tools are used to gather supplemenrary context: Potentially related PRs / work items, any existing comments and the PR description.
4. The AI agent analyzes the code, generates feedback, and returns it to the function. This process may include calling a number of tools exposed from MCP:
    - `search_related_prs`: Search for related pull requests that might provide context for the current review. Use this to find similar changes or related work.
    - `get_pr_comments`: Get existing comments and discussion threads for a PR to understand reviewer feedback and discussion context.
    - `analyze_branch_history`: Analyze recent commits in the source branch to understand development context and patterns.
    - `repo_create_pull_request_thread`: Creates a new comment thread on a pull request. (Guardrailed by post_to_pr)
    - The toolset implementation should be well structured to make it trivial to add more in the future, and should also use the 'domains' argument exposed by the server to enforce sensible guardrails.
5. The Azure Function posts the feedback to the pull request. This includes a single overview comment. NOTE: There should be a flag 'post_to_pr' bool to toggle this that is also passed to the agent as context to know whether to use PR comment toolsets.

Agent Orchestrator vars:

"PROJECT_ENDPOINT": "https://{TOM_WILL_DEPLOY_FOUNDRY}.services.ai.azure.com/api/projects/{TOM_WILL_DEPLOY_PROJECT}",
"MODEL_DEPLOYMENT_NAME": "gpt-5"

"ADO_PAT": "{KAVYA_AGENT_USER_TOKEN}", (In a key vault)
"AZDO_ORG_URL": "https://dev.azure.com/Next-Technology",

Example Request body to the Function:

{
  "PRid": "11111",
  "RepoName": "rsl-example",
  "Project": "GTBO",
  "Organisation": "Next-Technology",
}

## 6. Process Flow

1. Request Validation: Receive HTTP Request + Parse JSON body
2. Retrieve Git Diff + Metadata via ADO REST API (PAT AUTH)
3. Initialise MCP Server: Import MCP tools, Setup tool call handler for AI agent functions (PAT AUTH)
4. Create AI Agent: Create conversation thread (system prompts) + PR Diff / Metadata + Register MCP toolset as function tools
5. AI Processing Loop: Initialise agent run, Poll run status, handle tool calls and respond with tool outputs
6. Results Processing: Collect messages from the thread, Extract review content, Store tool call results
7. Post Agent Review Summary as PR Comment (via REST API)

## 7. Out of Scope

- Performing automated code merges.
- Automated Approval of PRs - The agent has no authority to approve incoming code changes.
- Full CI/CD pipeline execution (the agent focuses solely on code review).
- Analysis of non-code files (e.g., images, binaries). We need to make sure that we omit this when creating git diffs for the agent to avoid passing dead context.
- A user interface for managing the agent's configuration (initial configuration will be file-based).

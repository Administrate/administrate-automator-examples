# Administrate MCP Server

> **WARNING: this server can change your data.**
> The write tools (`create_classroom_event`, `update_event`, `add_event_staff`, `register_contacts_on_event`) create and change events, staff assignments and registrations. The `graphql_request` tool runs **any** GraphQL query or mutation the connected Administrate user is allowed to run, including ones that change or delete records the dedicated tools never touch. An AI client connected to this endpoint can call these tools without asking a person first.
>
> - Restrict who can reach the MCP endpoint, and keep the Bearer token secret
> - For read-only use, delete the four write tool nodes **and** `graphql_request` before you activate the workflow
> - Use an Administrate OAuth2 credential for a user with only the permissions you need
> - Test against a non-production instance first

## Problem

AI assistants and agents (Claude, ChatGPT, IDE assistants and others) are most useful when they can look things up in, and act on, the systems a team works in. Connecting an assistant to Administrate normally means writing and hosting a custom integration.

## Automator Solution

This workflow turns Automator into a [Model Context Protocol (MCP)](https://modelcontextprotocol.io) server for Administrate. Any MCP client can connect to the workflow's MCP endpoint. It gets a set of well-described tools backed by the Administrate GraphQL API: search events, course templates, accounts, contacts and portals, make common changes to events, and run raw GraphQL when nothing else fits.

## Features

- MCP Server Trigger protected by Bearer token authentication
- **Read tools**
  - `search_events`: events with dates, lifecycle state, places and course template
  - `search_course_templates`: the course catalogue
  - `search_accounts`: customer organisations
  - `search_contacts`: people, including instructors and staff
  - `list_portals`: WebLink and LMS portals with availability and single sign-on settings
- **Write tools**
  - `create_classroom_event`: create a classroom event
  - `update_event`: update an event by ID
  - `add_event_staff`: assign an instructor or administrator to an event
  - `register_contacts_on_event`: register contacts as learners
- **Escape hatch**: `graphql_request` runs any query or mutation
- Tool descriptions list the allowed filter fields and operators, so agents build valid filters
- HTTP errors are returned to the agent instead of failing the tool call, so the agent can read the GraphQL errors and try again

## Setup Instructions

### Prerequisites

- Access to Administrate Automator (with the MCP Server Trigger node available)
- Administrate OAuth2 credentials for a user with the permissions the tools need
- An MCP client that supports remote (HTTP/SSE) MCP servers with a bearer token header

### Installation

1. Download the `workflow.json` file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

#### 1. Decide which tools to keep

Read the warning at the top of this page. For a read-only server, delete these nodes:
- `create_classroom_event`
- `update_event`
- `add_event_staff`
- `register_contacts_on_event`
- `graphql_request`

#### 2. Set the MCP endpoint authentication

The `Administrate MCP Server` trigger uses **Bearer Auth**:
1. Create a **Bearer Auth** credential with a long, random token
2. Select it on the `Administrate MCP Server` trigger node

Do not switch the trigger's authentication to "None". Anyone who knows the URL could then use every tool.

The trigger's path is a random UUID. You can keep it or change it. The full URL is shown on the trigger node (Production URL).

#### 3. Set OAuth Credentials

Select your **Administrate OAuth2** credential on every tool node:
- `search_events`, `search_course_templates`, `search_accounts`, `search_contacts`, `list_portals`
- `create_classroom_event`, `update_event`, `add_event_staff`, `register_contacts_on_event`
- `graphql_request`

Every tool call runs as this Administrate user. Anything the user can do, `graphql_request` can do.

#### 4. Connect your MCP client

Activate the workflow. Then add a remote MCP server in your client with:
- **URL**: the Production URL from the `Administrate MCP Server` trigger, for example `https://YOUR-N8N-HOST/mcp/<path>`
- **Header**: `Authorization: Bearer <your token>`

### Testing

1. Activate the workflow and connect your MCP client
2. Check that the client lists the tools you kept
3. Ask a read-only question, for example "List the next 5 published events"
4. Check the workflow's executions in Automator to see each tool call
5. If you kept the write tools, test them on a non-production instance, for example "Create a draft classroom event for course X next Monday"

## How It Works

1. **Trigger**: An MCP client connects to the `Administrate MCP Server` endpoint with a Bearer token
2. **Tool discovery**: The trigger advertises each connected HTTP Request Tool node, with its name, description and parameters
3. **Tool call**: When the agent calls a tool, n8n fills the tool's `$fromAI()` parameters (such as `filters`, `first`, `input` or `query`) from the agent's arguments
4. **GraphQL request**: The tool posts a fixed GraphQL query or mutation with those variables to `https://api.getadministrate.com/graphql`, using the Administrate OAuth2 credential. `graphql_request` posts the agent's own document.
5. **Response**: The JSON response, including any GraphQL `errors`, is returned to the agent

The search tools return up to `first` results (default 20, or 50 for portals) and do not page. To see more, the agent should narrow its filters or raise `first`.

## Troubleshooting

**Client cannot connect / 401 or 403:**
- Check the workflow is active and you are using the Production URL, not the Test URL
- Check the `Authorization: Bearer <token>` header matches the Bearer Auth credential

**Tool calls return GraphQL errors:**
- A single malformed filter fails the whole request. Check field names against the tool description.
- `like` needs SQL wildcards (for example `%excel%`). `wordlike` takes a bare word.
- `in`, `notin` and `contains` need `values` (an array), not `value`

**Write tool returns HTTP 200 but nothing changed:**
- Check the `errors` list in the mutation response. A non-empty list means the write was rejected.

**Permission errors:**
- The Administrate OAuth2 user does not have permission for that object or action

**Agent makes changes you did not expect:**
- Deactivate the workflow, rotate the Bearer token, and remove the write tools and `graphql_request` if you only need read access

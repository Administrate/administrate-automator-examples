# OAuth - Authorization Code

**Version:** 1.0.0
**Last Updated:** 2026-03-05

## Problem

Sometimes a workflow requires users to enter information such as Account IDs, Schedule IDs, or other resource identifiers. When you create a form or webpage to collect this data, the requests made by the workflow use the credentials configured in n8n rather than the permissions of the Administrate user who initiated the action. This means any user with access to the form can perform operations beyond what their Administrate role allows.

## Automator Solution

This workflow implements the OAuth Authorization Code flow within n8n. Instead of relying on a single set of static credentials, it redirects the user through Administrate's OAuth login so that an access token is issued on behalf of that specific user. All subsequent GraphQL queries and mutations then execute with that user's permissions, ensuring proper access control is enforced.

### How It Works

The workflow is split into two main paths:

**Authorization path (`/webhook/oauth/start`)**

1. A user visits the OAuth Start webhook.
2. The workflow generates a random CSRF state token, stores it in an n8n Data Table, and redirects the user to Administrate's authorization endpoint.
3. The user logs in and grants access through Administrate.

**Callback path (`/webhook/oauth/callback`)**

1. Administrate redirects the user back to the Callback webhook with an authorization code and state parameter.
2. The workflow decodes and validates the state — checking that the redirect URL is allowed and that the state exists in the Data Table (preventing CSRF attacks).
3. If valid, the state row is deleted and the authorization code is exchanged for an access token.
4. The user is redirected back to the originating workflow with the access token as a query parameter.

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- **Administrate Developer Portal Application** — Register an application at [developer.getadministrate.com/account/my-apps](https://developer.getadministrate.com/account/my-apps) to obtain a Client ID and Client Secret
- **n8n Data Table** — Create a Data Table named `oauth_state` with a single column named `state`. This is used to store and validate CSRF state tokens during the OAuth flow. Alternatively, a third-party database can be used, but you would need to replace all references to the Data Table in the workflow.

### Installation

1. Download the `workflow.json` file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

After importing, update the following nodes with your own values:

1. **Instance Variables** — Set your `client_id` and Administrate `instance` name.
2. **Client Credentials** — Set your `client_id` and `client_secret` from the Developer Portal.
3. **Data Table nodes** (Store OAuth State, Get State, Remove State) — Point these to your `oauth_state` Data Table, or replace them with your preferred database.

## Usage

Once the workflow is active, you can initiate it from other workflows. The template includes an example, but it is recommended to remove the example section and use the same pattern from a separate workflow.

To integrate with your own workflow:

1. Create a webhook that serves as the user's entry point (e.g., `/webhook/get-events`).
2. When a user visits this webhook, check whether an access token is present in the query parameters.
3. **If no token is present** — Redirect the user to `/webhook/oauth/start?workflow-origin=<your-webhook-url>`. The OAuth flow will authenticate the user and redirect them back with a `token` query parameter.
4. **If a token is present** — Use that token in the `Authorization: Bearer` header for GraphQL queries and mutations against the Administrate API, or pass it along to an n8n Web Form if you prefer a form-based UI over custom HTML.

### Included Examples

The workflow includes two example patterns to demonstrate usage:

- **Custom HTML example** — Queries the Administrate GraphQL API for Events using the user's token and renders the results as a searchable HTML table.
- **n8n Web Form example (Account Merge)** — Passes the token to an n8n Web Form that collects Account IDs, validates them via GraphQL, and performs a merge mutation — all under the authenticated user's permissions.

## Important Notes

- n8n's Data Tables do not have a TTL feature. Rows could potentially accumulate from stale states if users abandon the OAuth flow before completing it. Consider using a different database in production, or create a separate workflow to periodically clean up old state rows.
- The workflow validates the redirect URL to ensure it only redirects to your n8n instance (`n8n.administratehq.com`). If your instance hostname differs, update the `Validate Redirect URL` code node accordingly.
- Browser prefetch requests to the OAuth Start webhook are detected and ignored (responded to with `204 No Content`) to prevent unintended OAuth initiations.

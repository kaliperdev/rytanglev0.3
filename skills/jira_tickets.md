---
id: jira_tickets
kind: procedure
description: Use this when someone asks to create a Jira issue/ticket, or
  read/fetch details of an existing Jira issue by key — creates and modifies
  content, manager ✅ required for writes.
template: standard
tools:
  - confluence_get_page_by_id
  - confluence_search_pages
required_credentials:
  - op://devasur/confluence/api_token
  - op://devasur/confluence/username
network_allowlist:
  - kaliper.atlassian.net
needs_approval: true
model_tier: mid
trigger_patterns:
  - create a jira ticket
  - create jira issue
  - open a jira ticket
  - log a jira issue
  - read jira ticket
  - fetch jira issue
  - get jira issue
  - show jira ticket
  - what is in jira issue
status: active
---

# Instructions

1. DETERMINE the action from the user's request:
   - "create", "open", "log", "add" → CREATE flow (step 2)
   - "read", "get", "fetch", "show", "what is", "details of" → READ flow (step 5)

---

## CREATE flow

2. COLLECT required fields. Act immediately with what the user provided; ask back ONLY if `project_key` or `summary` is missing and cannot be inferred:
   - `project_key` — Jira project key (e.g. "ENG", "BACK"). Ask if unknown.
   - `summary` — one-line ticket title.
   - `issue_type` — default "Task" if not specified.
   - `description` — optional; include if provided, omit otherwise.
   - `assignee_account_id` — optional; include only if explicitly given.
   - `priority` — optional; default omit.

3. CREATE the issue:
   POST /rest/api/3/issue
   Body (JSON):
   ```json
   {
     "fields": {
       "project": { "key": "<project_key>" },
       "summary": "<summary>",
       "issuetype": { "name": "<issue_type>" },
       "description": {
         "type": "doc",
         "version": 1,
         "content": [{ "type": "paragraph", "content": [{ "type": "text", "text": "<description>" }] }]
       }
     }
   }
   ```
   (Omit `description` block if none provided.)

4. READ from response:
   - `id`, `key` (e.g. "ENG-42"), `self` URL
   - Construct the browse URL: `https://kaliper.atlassian.net/browse/<key>`
   - Reply: "✅ Created *<key>*: <summary> — <browse URL>"

---

## READ flow

5. IDENTIFY the issue key from the user's message (e.g. "ENG-42"). Ask if not provided.

6. FETCH the issue:
   GET /rest/api/3/issue/{issueKey}?fields=summary,status,assignee,priority,description,issuetype,created,updated,reporter

7. READ and PRESENT from response:
   - `fields.summary` — title
   - `fields.status.name` — current status
   - `fields.issuetype.name` — type
   - `fields.priority.name` — priority (if set)
   - `fields.assignee.displayName` — assignee (or "Unassigned")
   - `fields.reporter.displayName` — reporter
   - `fields.created`, `fields.updated` — dates
   - `fields.description` — extract plain text from ADF content nodes (`content[].content[].text`), joining paragraphs with newlines. If empty, note "No description."
   - Browse URL: `https://kaliper.atlassian.net/browse/<issueKey>`

   Format the reply as:
   *<issueKey>: <summary>*
   • Type: <type> | Status: <status> | Priority: <priority>
   • Assignee: <assignee> | Reporter: <reporter>
   • Created: <created> | Updated: <updated>
   • Description: <description>
   • 🔗 <browse URL>

---

## FAILURE HANDLING

- 400: Surface the error message from the response body — likely a missing required field or invalid project key.
- 401/403: Inform the user the Jira credentials may be invalid or lack permission. Direct a manager to refresh credentials.
- 404: For READ — tell the user the issue key was not found; ask them to double-check. For CREATE — the project key may not exist.
- Any other non-2xx: Surface the status code and response message and stop.

Defaults to agent-created issues when operating on tickets. Does not modify human-created content unless an explicit issue key is supplied.

Does not do: delete issues, manage sprints, transition statuses, search issues by JQL, or manage Jira projects/users.

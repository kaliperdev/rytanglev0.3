---
id: read_notion_documents
kind: capability
description: Use this when someone asks to read, retrieve, or search Notion
  pages, databases, or blocks.
template: standard
tools:
  - notion_read
required_credentials:
  - op://rytanglev0.3/Notion/token
network_allowlist:
  - https://api.notion.com
needs_approval: false
model_tier: cheap
trigger_patterns:
  - read notion page
  - open notion doc
  - search notion
  - get notion page
  - fetch notion
  - look up notion
  - show notion page
  - retrieve notion
  - query notion database
status: active
---

# Instructions

1. Identify what the user wants: a specific page (by ID or URL), a database query, or a search across the workspace.

2. **If a Notion page URL is given**, extract the page ID from the URL (last 32-char hex segment, e.g. `https://www.notion.so/My-Page-abc123def456...` → page ID = `abc123def456...`). Strip any hyphens from the ID before using it.

3. **Retrieve a specific page** — GET `/v1/pages/{page_id}`
   - Read: `id`, `properties`, `url`, `created_time`, `last_edited_time`

4. **Retrieve page content (blocks)** — GET `/v1/blocks/{page_id}/children?page_size=100`
   - Read: `results[]` (each block has `type` and its type-keyed content, e.g. `paragraph.rich_text[].plain_text`)
   - If `has_more: true`, paginate using `next_cursor` via `?start_cursor=<cursor>`

5. **Search the workspace** — POST `/v1/search`
   - Body: `{ "query": "<user's search terms>", "filter": { "value": "page", "property": "object" }, "page_size": 10 }`
   - Read: `results[].id`, `results[].url`, `results[].properties.title` (or `results[].properties.Name`) to list matching pages
   - Present results as a numbered list with title + URL; ask the user which one to open if they want content

6. **Query a database** — POST `/v1/databases/{database_id}/query`
   - Body: `{ "page_size": 20 }` (add `filter` or `sorts` if the user specifies criteria)
   - Read: `results[]` entries with their properties

7. **Assemble and present** the content in readable form:
   - For pages: show title, last edited time, then render block content in order (headings, paragraphs, bullets, to-dos, code blocks, etc.)
   - For databases: show a table-style list of entries with key properties
   - For search: show a numbered list — title, URL, last edited

8. **Pagination**: if the user asks for more results and `has_more` was true, re-call the same endpoint with `?start_cursor=<next_cursor>` and append results.

9. **Error handling**: on non-2xx, surface the HTTP status and `message` field from the response and stop. Common cases:
   - 401 → token invalid or missing
   - 403 → the integration hasn't been shared with this page/database — tell the user to share the Notion page with the integration
   - 404 → page or database ID not found

**Worked example**
User: "Read my Notion page https://www.notion.so/Project-Roadmap-1a2b3c4d5e6f..."
→ Extract page ID: `1a2b3c4d5e6f...`
→ GET `/v1/pages/1a2b3c4d5e6f` → get title + metadata
→ GET `/v1/blocks/1a2b3c4d5e6f/children?page_size=100` → render blocks as readable text
→ Reply with the page title, last-edited date, and full content

Does not do: create, update, or delete Notion content; access private pages not shared with the integration.

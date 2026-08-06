---
id: confluence_read_document
kind: capability
description: Use this when someone asks to read, fetch, or summarise a
  Confluence page by page ID, title, or URL — uses REST API (no MCP).
template: standard
tools:
  - confluence_get_page_by_id
  - confluence_search_pages
required_credentials:
  - op://devasur/confluence/api_token
  - op://devasur/confluence/username
network_allowlist:
  - kaliper.atlassian.net
needs_approval: false
model_tier: mid
trigger_patterns:
  - read confluence page
  - fetch confluence document
  - what does confluence page say
  - get confluence page
  - summarise confluence page
  - open confluence doc
status: active
---

# Instructions

1. IDENTIFY the page reference from the user's request — it can be:
   a. A full Confluence URL (e.g. https://kaliper.atlassian.net/wiki/spaces/ENG/pages/123456) → extract the numeric page ID from the path.
   b. A bare numeric page ID (e.g. "page 123456").
   c. A page title (e.g. "Onboarding Guide") → proceed to step 2 to search.

2. IF a page ID is known (cases a or b), fetch it directly:
   GET /wiki/rest/api/content/{pageId}?expand=body.storage,title,space
   Read: title, space.name, body.storage.value (raw HTML).

3. IF only a title is known (case c), search first:
   GET /wiki/rest/api/content/search?cql=title="{title}"&limit=5&expand=title,space
   Pick the best-matching result from results[*].id and results[*].title; confirm the match to the user if ambiguous.
   Then fetch the chosen page using step 2.

4. PARSE the retrieved body:
   - Strip all HTML tags to extract readable plain text.
   - If the resulting text is under 2 000 characters, present it in full.
   - If longer, present a structured summary: headings/sections found, key points per section, and a note that the full page is at the Confluence URL.

5. PRESENT the result clearly:
   - Page title, Space name, direct URL (https://kaliper.atlassian.net/wiki/spaces/{spaceKey}/pages/{pageId}).
   - Readable content or summary.

WORKED EXAMPLE:
User: "Read the Confluence page 'API Design Standards'"
→ Step 3: GET /wiki/rest/api/content/search?cql=title="API Design Standards"&limit=5&expand=title,space
→ Returns id: 789012, title: "API Design Standards", space: "ENG"
→ Step 2: GET /wiki/rest/api/content/789012?expand=body.storage,title,space
→ Strip HTML, present content under the page title with the direct URL.

FAILURE HANDLING:
- 401/403: Inform the user that the Confluence credentials may be invalid or lack permission for this page. Direct a manager to refresh credentials.
- 404: Tell the user the page was not found — ask them to double-check the ID, title, or URL.
- Search returns 0 results: Report that no page matching that title was found in Confluence.
- Any other non-2xx: Surface the status code and message and stop.

Does not do: create, update, or delete Confluence pages; list all spaces or pages; access pages the credential account has no permission to view.

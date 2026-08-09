---
name: note-taking-tools
description: "Manage notes and knowledge — Obsidian vault (filesystem) and Notion (API + ntn CLI). Read, search, create, edit, wikilink."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [notes, knowledge-management, obsidian, notion, writing, documentation]
    related_skills: [ocr-and-documents, google-workspace]
---

# Note-Taking Tools

Manage notes across two platforms — **Obsidian** (filesystem-based markdown vault) and **Notion** (REST API + ntn CLI). Pick the section that matches your platform.

---

## Obsidian (Filesystem Vault)

Read, search, create, and edit notes in your Obsidian vault using file tools.

### Vault path

Use a known or resolved vault path before calling file tools.

The documented vault-path convention is the `OBSIDIAN_VAULT_PATH` environment variable, for example from `${HERMES_HOME:-~/.hermes}/.env`. If it is unset, use `~/Documents/Obsidian Vault`.

File tools do not expand shell variables. Do not pass paths containing `$OBSIDIAN_VAULT_PATH` to `read_file`, `write_file`, `patch`, or `search_files`; resolve the vault path first and pass a concrete absolute path. Vault paths may contain spaces, which is another reason to prefer file tools over shell commands.

If the vault path is unknown, `terminal` is acceptable for resolving `OBSIDIAN_VAULT_PATH` or checking whether the fallback path exists. Once the path is known, switch back to file tools.

### Read a note

Use `read_file` with the resolved absolute path to the note. Prefer this over `cat` because it provides line numbers and pagination.

### List notes

Use `search_files` with `target: "files"` and the resolved vault path. Prefer this over `find` or `ls`.

- To list all markdown notes, use `pattern: "*.md"` under the vault path.
- To list a subfolder, search under that subfolder's absolute path.

### Search

Use `search_files` for both filename and content searches. Prefer this over `grep`, `find`, or `ls`.

- For filenames, use `search_files` with `target: "files"` and a filename `pattern`.
- For note contents, use `search_files` with `target: "content"`, the content regex as `pattern`, and `file_glob: "*.md"` when you want to restrict matches to markdown notes.

### Create a note

Use `write_file` with the resolved absolute path and the full markdown content. Prefer this over shell heredocs or `echo` because it avoids shell quoting issues and returns structured results.

### Append to a note

Prefer a native file-tool workflow when it is not awkward:

- Read the target note with `read_file`.
- Use `patch` for an anchored append when there is stable context, such as adding a section after an existing heading or appending before a known trailing block.
- Use `write_file` when rewriting the whole note is clearer than constructing a fragile patch.

For an anchored append with `patch`, replace the anchor with the anchor plus the new content.

For a simple append with no stable context, `terminal` is acceptable if it is the clearest safe option.

### Targeted edits

Use `patch` for focused note changes when the current content gives you stable context. Prefer this over shell text rewriting.

### Wikilinks

Obsidian links notes with `[[Note Name]]` syntax. When creating notes, use these to link related content.

---

## Notion (REST API + ntn CLI)

Talk to Notion two ways. Same integration token works for both — pick by what's available.

**`ntn` CLI** — Notion's official CLI. Shorter syntax, one-line file uploads, required for Workers. macOS + Linux only. **Default when installed.**
**HTTP + curl** — works everywhere including Windows. **Default fallback** when `ntn` isn't installed.

### Setup

1. Create an integration at https://notion.so/my-integrations
2. Copy the API key (starts with `ntn_` or `secret_`)
3. Store in `${HERMES_HOME:-~/.hermes}/.env`: `NOTION_API_KEY=ntn_your_key_here`
4. **Share target pages/databases** with the integration in Notion: page menu `...` → `Connect to` → your integration name. Without this, the API returns 404.

### Install ntn (macOS/Linux)

```bash
curl -fsSL https://ntn.dev | bash
# Or via npm:
npm install --global ntn
```

Skip `ntn login` — use the integration token instead:
```bash
export NOTION_API_TOKEN=$NOTION_API_KEY
export NOTION_KEYRING=0
```

Add those exports to your shell profile or `${HERMES_HOME:-~/.hermes}/.env`.

### Choose path at runtime

```bash
if command -v ntn >/dev/null 2>&1; then
  # use ntn
else
  # fall back to curl
fi
```

### Path A — ntn CLI (preferred, macOS/Linux)

```bash
# Search
ntn api v1/search query="page title"

# Read page as Markdown (agent-friendly)
ntn api v1/pages/{page_id}/markdown

# Read page content as blocks
ntn api v1/blocks/{page_id}/children

# Create page from Markdown
ntn api v1/pages \
  parent[page_id]=xxx \
  properties[title][0][text][content]="Notes from meeting" \
  markdown="# Agenda\n\n- Q3 roadmap"

# Patch a page with Markdown
ntn api v1/pages/{page_id}/markdown -X PATCH \
  markdown="## Update\n\nShipped the prototype."

# Query a database (data source)
ntn api v1/data_sources/{data_source_id}/query -X POST \
  filter[property]=Status filter[select][equals]=Active

# File uploads (one-liner — biggest CLI win)
ntn files create < photo.png
ntn files list
```

### Path B — HTTP + curl (cross-platform)

All requests share this pattern:

```bash
curl -s -X GET "https://api.notion.com/v1/..." \
  -H "Authorization: Bearer $NOTION_API_KEY" \
  -H "Notion-Version: 2025-09-03" \
  -H "Content-Type: application/json"
```

Key endpoints:

- `POST /v1/search` — search pages
- `GET /v1/pages/{id}/markdown` — read page as Markdown
- `GET /v1/blocks/{id}/children` — read page as blocks
- `POST /v1/pages` — create page (supports `markdown` body field)
- `PATCH /v1/pages/{id}/markdown` — patch page with Markdown
- `POST /v1/data_sources/{id}/query` — query a database
- `POST /v1/data_sources` — create a database
- `PATCH /v1/pages/{id}` — update page properties
- `PATCH /v1/blocks/{id}/children` — append blocks

### Property Types

```json
// Title
"Name": {"title": [{"text": {"content": "Hello"}}]}
// Select
"Status": {"select": {"name": "Done"}}
// Multi-select
"Tags": {"multi_select": [{"name": "A"}]}
// Date
"Due": {"date": {"start": "2026-01-15"}}
// Checkbox, Number, URL, Email
"Done": {"checkbox": true}
```

### API Version 2025-09-03 — Databases vs Data Sources

- **Databases became data sources.** Use `/data_sources/` endpoints for queries.
- Two IDs per database: `database_id` (when creating pages) and `data_source_id` (when querying).
- Search returns databases as `"object": "data_source"` with the `data_source_id` field.

### Notion Workers (requires ntn, Business/Enterprise plan)

Workers are TypeScript programs Notion hosts — syncs, tools, and webhooks.

```bash
ntn workers new my-worker
cd my-worker
# Edit src/index.ts
ntn workers deploy --name my-worker
```

Worker lifecycle: `ntn workers deploy|list|exec <key>|runs list|webhooks list`.

### Notion-Flavored Markdown

Standard CommonMark plus XML-like tags:

```xml
<callout icon="🎯" color="blue_bg">Ship the MVP.</callout>
<details color="gray"><summary>Toggle</summary>Hidden content</details>
<columns><column>Left</column><column>Right</column></columns>
<table_of_contents color="gray"/>
```

### Important Rules

- Notion-Version `2025-09-03` is required on all HTTP requests.
- Page/database IDs are UUIDs (with or without dashes).
- Rate limit: ~3 requests/second.
- Always share target pages with the integration via the Notion UI.
- Use `"is_inline": true` when creating data sources to embed in a page.
- Pass `-s` to curl to suppress progress bars. Pipe through `jq` when reading.

## Pitfalls

- **Obsidian:** File tools don't expand shell variables — resolve OBSIDIAN_VAULT_PATH before passing to read_file/write_file.
- **Obsidian:** Vault paths may contain spaces — prefer file tools over shell commands.
- **Notion:** Page/database must be shared with the integration or API returns 404.
- **Notion:** Rate limit is ~3 requests/second — batch reads when possible.
- **Notion:** Skip `ntn login` on headless — use `NOTION_API_TOKEN` env var and `NOTION_KEYRING=0`.

## Verification

- **Obsidian:** `search_files(pattern="*.md", path="$VAULT_PATH", target="files")` returns notes.
- **Notion:** `curl -s "https://api.notion.com/v1/users" -H "Authorization: Bearer $NOTION_API_KEY" -H "Notion-Version: 2025-09-03"` returns 200.

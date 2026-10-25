---
name: thymer
description: Autonomous workspace management for Thymer Desktop via Model Context Protocol (MCP). Guides agents in creating and updating records, managing line item trees, rich segments (refs, links, datetimes), collections, properties, task statuses, daily journal pages, and panel navigation.
---

# Thymer Core Skill

This skill provides comprehensive instructions for AI agents interacting with **Thymer Desktop** through its native Model Context Protocol (MCP) server.

Use this skill whenever reading, creating, updating, linking, searching, or organizing notes, tasks, collections, or journal entries in a Thymer workspace.

---

## 1. Connection & Foundation

- **MCP Endpoint:** Thymer Desktop exposes its MCP server locally at `http://127.0.0.1:13100/mcp` (HTTP POST with JSON-RPC 2.0).
- **Workspace Scoping:** Always call `list_workspaces` first to identify the user's active/focused workspace. Always pass the explicit `workspace` GUID in all subsequent tool calls. Never assume or hardcode workspace GUIDs across sessions.
- **Crockford Base32 GUIDs:** All native Thymer records, line items, and collections use 26-character Base32 GUIDs (`0-9`, `A-H`, `J-K`, `M-N`, `P-T`, `V-Z`, uppercase). Validate GUID formats before creating relations.
- **MCP API Return Fields:**
  - `create_record` returns `{ created: true, guid: "...", title: "...", collection_guid: "..." }`. The identifier field is **`.guid`** (never `.record_guid` or `.id`).
  - `create_line_item` returns `{ created: true, guid: "...", record_guid: "..." }`. The identifier field is **`.guid`** (never `.line_item_guid`).
- **Prefer MCP Over Direct Disk Writes:** Thymer maintains an in-memory reactive database that syncs down to markdown files on disk. Editing `.md` files directly on disk bypasses this reactive engine, leading to stale UI panels, cached editor conflicts, or delayed sync. Always perform document and property writes via MCP tools (`insert_markdown`, `create_line_item`, `update_record_property`).

---

## 2. Core Data Architecture

Thymer's data model is fundamentally a **single tree of nested blocks (line items)**:
```text
Workspace (Root)
 ├── Collections (Databases / Schemas / Views)
 │    └── Records (Pages / Documents / Notes)
 │         └── Line Items (Tasks, Headings, Text, Lists, Quotes)
 │              └── Segments (Rich text pieces: text, ref, link, datetime)
 └── Journal (Daily personal timeline pages)
```

- **Collections:** Subtrees defining database views and field schemas (e.g. `Projects`, `Clients`, `Library`).
- **Records:** Pages or documents inside collections or at workspace root.
- **Line Items:** Content nodes inside a record. Line items can be infinitely nested (`children: [...]` and `parent_guid`).
- **Segments:** The rich text elements that make up the inline text of any line item.

---

## 3. Working with Records & Properties

### A. Structured vs. Raw Markdown Format (Critical)
`get_record` behaves differently depending on the `format` argument:
- `get_record({ workspace, record })`: Returns a **raw Markdown string with YAML frontmatter**.
- `get_record({ workspace, record, format: 'structured' })`: Returns a **JSON object**:
  ```json
  {
    "guid": "1X526PD9ABAH1NJEX1HNKKTWSZ",
    "name": "Project Alpha",
    "collection_guid": "1B0J3X9Y8QS96C75M3QMT23DNW",
    "properties": {
      "Status": ["choice", "cDD41F10A87FC75"],
      "Budget": ["number", 8500],
      "Deadline": ["datetime", { "d": "20261130" }]
    },
    "items": [ ... ]
  }
  ```

### B. Reading Typed Property Tuples
Property values in structured mode are returned as typed tuples: `[type, value]`.
- Always unpack `val[1]` (e.g. `val[0] === 'number' ? Number(val[1]) : val[1]`). Direct arithmetic without unpacking will yield `NaN`.
- Choice properties return internal choice IDs (e.g. `cDD41F10A87FC75`). Use `get_collection_schema` to inspect the available human-readable labels.

### C. Updating Properties vs. Note Text (The Structured Properties Rule)
> [!IMPORTANT]
> Thymer's database tables, Kanban views, filter bars, and dashboard widgets read **structured properties** from collections, not arbitrary text in note bodies.
> When updating statuses, deadlines, contacts, or financial figures, always use MCP tools (`create_record(..., properties: { ... })` or `update_record_property(...)`). Writing prose into markdown notes does not populate database columns.

When calling `update_record_property` or `create_record`:
- For **choice** properties, pass either the plain text label (e.g. `"In Progress"`, `"Paid"`) or the choice ID. Labels are matched case-insensitively.
- For **date** properties, pass standard ISO dates (e.g. `"2026-10-25"`).

---

## 4. Line Items & Rich Text Segments

### Line Item Types
Line items support the following types:
- `text`: Plain text block
- `task`: Actionable checkbox item
- `heading`: Section header (specify `heading_size`: 1 to 6)
- `ulist`: Bullet list item
- `olist`: Numbered list item
- `quote`: Blockquote
- `linkbtn`: Action button pill
- `ascii-banner`: ASCII decorative header (`banner_style`: `"flames"`, `"outline"`, `"console"`)

### The Rich Text Segment Engine
A line item's text is composed of an array of rich segments:
```json
"segments": [
  { "type": "text", "text": "Review " },
  { "type": "ref", "text": "1XGA0X00XT61KFD862FC8KJJZ7" },
  { "type": "text", "text": " before " },
  { "type": "datetime", "text": "tomorrow 10am" }
]
```

### Supported Segment Types:
| Segment Type | `text` Value | Description |
| :--- | :--- | :--- |
| **`text`** | `"string"` | Standard plain text |
| **`bold`** | `"string"` | Bold formatted text |
| **`italic`** | `"string"` | Italic formatted text |
| **`code`** | `"string"` | Inline code |
| **`link`** | `"https://..."` | Bare URL |
| **`linkobj`** | `{ "link": "...", "title": "..." }` | Named hyperlink |
| **`ref`** | `recordGuid` (string) | **Native clickable record link pill** (see below) |
| **`datetime`** | Date string (e.g. `"today"`, `"2026-10-25"`, `"monday 3pm"`, ranges) | Interactive date/time token |
| **`mention`** | `userGuid` (string) | User @mention |
| **`hashtag`** | `"#tag"` | Categorical tag |
| **`icon`** | `"ti-star"` | Tabler icon class |

### Native Record Links (`ref` Segments)
> [!IMPORTANT]
> When calling `insert_markdown`, standard markdown syntax like `[[Record Title]]` is treated as **literal plain text**.
> To create an interactive, clickable record link pill that navigates to the target page in Thymer, you **must** use `create_line_item` or `update_line_item` with a `ref` segment:
> ```json
> "segments": [
>   { "type": "ref", "text": "1XGA0X00XT61KFD862FC8KJJZ7" },
>   { "type": "text", "text": " — Document Description" }
> ]
> ```

### Hierarchical Block Insertion (Nesting & Ordering)
When manipulating the block tree with `create_line_item`:
- Pass `parent: parentGuid` to nest a line item underneath an existing heading, task, or list item. If omitted, the block attaches at the record root.
- Pass `after: previousGuid` to control vertical ordering relative to sibling blocks.
- To move an existing block, use `move_line_item({ workspace, line_item, parent, after })`.

---

## 5. Task Management & The 3-Step Waiting Task Pattern

### Native Task Statuses (`set_task_status`)
When updating tasks, use `set_task_status(workspace, lineItemGuid, status)`:

| Status Code | Meaning | Behavior in UI |
| :--- | :--- | :--- |
| `"none"` | Unchecked | Standard open task |
| `"done"` | Completed | Strikethrough checkbox |
| `"started"` | In Progress | Running / started marker |
| `"blocked"` / `"waiting"` | Waiting on External | Routes to "Waiting On Others" queues |
| `"important"` | High Priority | Exclamation badge |
| `"billable"` | Billable Effort | Dollar badge |
| `"alert"` | Urgent Warning | Warning badge |
| `"starred"` | Starred / Focus | Star indicator |
| `"canceled"` | Abandoned | Canceled marker |

### The 3-Step Waiting Task Sequence
When an agent writes a document that contains tasks waiting on external feedback or client delivery:
1. Write the document body including the task markdown (`- [ ] Wait for client assets...`) via `insert_markdown`.
2. Call `get_line_items(workspace, recordGuid)` to locate the newly parsed task line item `guid`.
3. Call `set_task_status(workspace, lineGuid, "waiting")`.

*(Note: Do not create disconnected line items before `insert_markdown`, as markdown insertion re-parses the note's block tree).*

---

## 6. Journal Operations

The Journal is Thymer's date-indexed personal timeline.

- **Read Journal Page:**
  ```javascript
  await call('get_journal_page', {
    workspace: WS,
    date: '2026-10-25',
    format: 'markdown' // or 'structured'
  });
  ```
- **Append Content Safely (Non-Destructive):**
  Use `add_to_journal` to append markdown cleanly to the date's page without overwriting existing entries:
  ```javascript
  await call('add_to_journal', {
    workspace: WS,
    date: '2026-10-25',
    markdown: '## 🎯 Evening Wrap-up\n- [x] All sprint tasks shipped\n'
  });
  ```
- **Navigate to Journal in Desktop UI:**
  ```javascript
  await call('navigate_to_journal', { workspace: WS, date: '2026-10-25' });
  ```

---

## 7. Panel & UI Navigation

Agents can inspect and manipulate the user's desktop workspace:
- `get_open_panels`: Inspect active panel indexes and currently loaded records.
- `navigate_to_record`: Open a record in the active or specified panel.
- `navigate_to_collection`: Navigate a panel to a specific collection view.
- `create_panel` / `close_panel`: Open side-by-side split panels or close unused tabs.

---

## 8. Best Practices for Agents

1. **Non-Destructive Operations:** Always inspect existing record line items or properties before writing. Never overwrite whole notes when appending notes or tasks.
2. **Resolve Before Creating:** Always run `search` or `query_records` to check for existing records to prevent creating duplicates.
3. **Link Everything:** In notes and deliverables, link related records using native `{ type: "ref", text: guid }` segments so the user has an interconnected knowledge graph.
4. **Backlinks Discovery:** Use `get_backlinks` to identify what other pages or projects reference a specific record.

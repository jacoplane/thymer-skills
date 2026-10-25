# Thymer Agent Skills

A modular collection of open agent skills for [Thymer](https://thymer.com/), providing AI coding assistants (Claude Code, Cursor, Antigravity, Gemini CLI) with native workspace manipulation and plugin development capabilities.

Compatible with the [skills.sh](https://skills.sh/) open agent skills ecosystem.

---

## 📦 Included Skills

| Skill | Target | Description | Install Command |
| :--- | :--- | :--- | :--- |
| **`thymer`** | Workspace & Notes | Autonomous workspace management, line items, native `ref` link pills, property unpacking, task statuses, daily journal pages, and panel navigation via MCP. | `npx skills add jacoplane/thymer-skills@thymer` |
| **`thymer-plugin-dev`** | Plugin Development | Complete development, styling, bundling (`esbuild`), dynamic eval compatibility (`new Function`), instant MCP hot-deploy, and Electron CDP debugging runbook. | `npx skills add jacoplane/thymer-skills@thymer-plugin-dev` |

---

## 🚀 Quickstart

### 1. Managing Your Thymer Workspace (`thymer`)

Install the `thymer` skill into your agent workspace:
```bash
npx skills add jacoplane/thymer-skills@thymer
```

Once installed, your agent can naturally handle tasks like:
- *"Create a project page in my Projects collection with an initial milestone breakdown and tasks."*
- *"Link this deliverable to the client profile using native record references."*
- *"Log my daily wrap-up and top 3 priorities into today's Journal without clobbering existing notes."*
- *"Find all overdue tasks across active projects."*

### 2. Building Desktop Plugins (`thymer-plugin-dev`)

Install the plugin developer skill:
```bash
npx skills add jacoplane/thymer-skills@thymer-plugin-dev
```

Empowers your agent to:
- Scaffold clean, production-ready Thymer Desktop plugins with manifest validation.
- Avoid the runtime `new Function` export crash.
- Hot-deploy updated JavaScript and CSS bundles directly into your running Thymer Desktop over MCP.
- Connect to Electron remote debugging (`--remote-debugging-port=9444`) to take screenshots and verify UI appearance automatically.

---

## 🛠️ Requirements & Platform Support
- **Operating Systems:** Fully compatible with **macOS**, **Linux**, and **Windows**.
- **Application:** [Thymer Desktop](https://thymer.com/) with Local MCP Server enabled (`http://127.0.0.1:13100/mcp`).
- **Runtime:** Node.js 18+

---

## 📄 License
MIT © [Jaap Vermeulen](https://github.com/jacoplane)

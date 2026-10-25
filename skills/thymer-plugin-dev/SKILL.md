---
name: thymer-plugin-dev
description: Comprehensive development, debugging, and release runbook for Thymer Desktop plugins. Covers the plugin lifecycle, SDK APIs, dynamic evaluator compatibility (avoiding new Function export errors), esbuild bundling, instant MCP hot-reload, and automated Electron remote debugging via Chrome DevTools Protocol (CDP).
---

# Thymer Plugin Development Skill

This skill provides an autonomous development, inspection, styling, debugging, building, and deployment runbook for **Thymer Desktop plugins**.

Use this skill whenever building, modifying, styling, debugging, or releasing a desktop plugin for Thymer.

---

## 1. Plugin Architecture & Structure

A standard Thymer plugin consists of:
```text
my-thymer-plugin/
├── plugin.json              # Plugin manifest
├── package.json             # NPM metadata and scripts
├── build.mjs                # Bundler script (esbuild)
├── src/                     # Source files
│   ├── index.js             # Plugin entrypoint
│   └── styles.css           # Plugin styles
└── dist/                    # Compiled artifacts
    ├── plugin.js            # Evaluated bundle
    └── plugin.css           # Injected stylesheet
```

### The Manifest (`plugin.json`)
```json
{
  "ver": 1,
  "name": "My Plugin",
  "version": "1.0.0",
  "author": "YourName",
  "repository": "https://github.com/username/my-plugin",
  "icon": "layout-grid",
  "item_name": "My Plugin",
  "description": "Short description of what the plugin does.",
  "type": "app",
  "show_sidebar_items": true,
  "show_cmdpal_items": true,
  "__source_repo": "https://github.com/username/my-plugin"
}
```
*Note: Always include `"__source_repo"` pointing to the public GitHub repository to ensure full update tracking compatibility with Thymer's native Plugins Manager.*

---

## 2. The 6-Step Autonomous Development Loop

Agents developing or maintaining Thymer plugins must follow this strict, closed-loop execution cycle. Never stop after modifying source files; always build, deploy, reload, and visually verify:

```text
  ┌─────────────────────────────────────────────────────────────┐
  │ 1. Inspect Current State via CDP (:9444)                    │
  │    Capture screenshot, evaluate DOM, inspect error console  │
  └──────────────────────────────┬──────────────────────────────┘
                                 ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ 2. Modify Source Code (src/index.js & src/styles.css)        │
  │    Implement feature, patch bug, or adjust CSS layout       │
  └──────────────────────────────┬──────────────────────────────┘
                                 ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ 3. Build Plugin Bundle (node build.mjs)                     │
  │    Compile with esbuild, strip 'export' keywords for eval   │
  └──────────────────────────────┬──────────────────────────────┘
                                 ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ 4. Hot-Deploy Live via MCP (Port 13100)                     │
  │    Push dist/plugin.js & dist/plugin.css to target workspace │
  └──────────────────────────────┬──────────────────────────────┘
                                 ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ 5. Trigger Live Reload via CDP                              │
  │    Execute window.location.reload() in Electron renderer    │
  └──────────────────────────────┬──────────────────────────────┘
                                 ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ 6. Visually Verify Layout & Health                          │
  │    Take fresh CDP screenshot, confirm zero console errors   │
  └─────────────────────────────────────────────────────────────┘
```

---

## 3. The Runtime Environment & The Dynamic Eval Trap

> [!CAUTION]
> **Dynamic Evaluator Rule (`new Function`):**
> Thymer Desktop loads plugins dynamically in the renderer process via:
> ```javascript
> new Function('plugin', 'context', bundleCode)(pluginInstance, pluginContext);
> ```
> In JavaScript, `new Function(...)` strictly forbids ES module `export` and `import` declarations.
> Any top-level `export const ...` or `export default ...` will throw a fatal runtime syntax error:
> `SyntaxError: Unexpected token 'export'`
> and prevent the plugin from loading completely, stalling the MCP backend bridge.

### Hardening Your Build Pipeline (`build.mjs`)
Always configure your bundler (e.g. `esbuild`) to output an IIFE and sanitize any lingering `export` keywords:
```javascript
import esbuild from 'esbuild';
import fs from 'fs';

await esbuild.build({
  entryPoints: ['src/index.js'],
  bundle: true,
  format: 'iife',
  outfile: 'dist/plugin.js'
});

// Strip any stray export keywords for new Function compatibility
let js = fs.readFileSync('dist/plugin.js', 'utf8');
js = js.replace(/^export\s+/gm, '');
fs.writeFileSync('dist/plugin.js', js, 'utf8');
```

---

## 4. Plugin SDK Surface & Navigation Protocols

### A. Core Lifecycle Methods
Plugins hook into Thymer via standard lifecycle methods:

```javascript
class MyPlugin {
  async onInit(context) {
    this.context = context;
    this.ui = context.ui;
    this.data = context.data;

    // Register Command Palette items
    this.ui.registerCommand({
      id: 'my-plugin-action',
      label: 'My Plugin: Perform Action',
      onSelected: () => this.runAction()
    });

    // Mount Custom Side Panel or App View
    if (this.context.isApp) {
      this.renderAppView();
    }
  }

  onDestroy() {
    // Clean up event listeners, timers, or DOM nodes
  }

  showToast(message) {
    this.ui.addToaster({
      title: message,
      autoDestroyTime: 3000,
      dismissible: true
    });
  }
}
```

### B. Thymer Navigation Protocol (RootId Routing)
> [!WARNING]
> When navigating panels via the SDK, you must distinguish between records and collections:
> - **Individual Records:** Use `{ type: 'edit_panel', rootId: recordGuid, workspaceGuid }`.
> - **Collections / Databases:** Use `{ type: 'overview', rootId: collectionGuid, workspaceGuid }`.
> 
> Calling `edit_panel` on a collection GUID causes Thymer to crash the panel with `"Failed to load panel"`.

To open side-by-side split panels without replacing the main dashboard:
```javascript
const sidePanel = await this.ui.createPanel({ afterPanel: sourcePanel });
sidePanel.navigateTo({ type: 'overview', rootId: collectionGuid, workspaceGuid });
```

---

## 5. Multi-Panel Responsive Design (Container Queries)

In Thymer Desktop, users frequently split panels side by side. 
- **Do NOT** use viewport-based `@media (max-width: ...)` queries for layout; the monitor width does not change when split panels open.
- Use **CSS Container Queries** on your plugin root element:
  ```css
  .my-app-root {
    container-type: inline-size;
    container-name: myApp;
  }

  @container myApp (max-width: 1120px) {
    /* Multi-panel split layout */
    .dashboard-grid { grid-template-columns: 1fr; }
  }

  @container myApp (max-width: 720px) {
    /* Narrow panel layout */
    .action-rail { display: none; }
  }
  ```
- **Breakout Width Rule:** Never apply `width: 100% !important` to `.panel`. Apply breakout widths exclusively to inner elements (`.panel-content`, `.editor`, `.panel-body`).

---

## 6. Instant Hot-Deploy via MCP

Rather than manually zipping files and dragging them into the UI, deploy bundles directly to the active Thymer Desktop instance over MCP:

```javascript
import { connectMcp } from './thymer-mcp.mjs';
import fs from 'fs';

const call = await connectMcp();
const WS = 'YOUR_WORKSPACE_GUID';
const PLUGIN_GUID = 'YOUR_PLUGIN_GUID';

// 1. Push compiled JavaScript
await call('update_plugin_code', {
  workspace: WS,
  plugin: PLUGIN_GUID,
  code: fs.readFileSync('dist/plugin.js', 'utf8')
});

// 2. Push compiled CSS
await call('update_plugin_css', {
  workspace: WS,
  plugin: PLUGIN_GUID,
  css: fs.readFileSync('dist/plugin.css', 'utf8')
});

// 3. Push manifest updates (if version changed)
await call('update_plugin_json_config', {
  workspace: WS,
  plugin: PLUGIN_GUID,
  config: fs.readFileSync('plugin.json', 'utf8')
});
```

---

## 7. Electron Remote Debugging & Automated CDP Verification

To allow AI agents to visually inspect, test, and take screenshots of plugins running inside Thymer Desktop across **macOS**, **Linux**, and **Windows**:

### A. The Single-Instance Lock Trap
Electron applications enforce a single-instance lock (`app.requestSingleInstanceLock()`). If Thymer is already running in the background or system tray without the debugging flag, launching with `--remote-debugging-port=9444` will simply focus the existing instance and exit immediately without opening port 9444.

**Always terminate existing instances before starting in debug mode:**
- **macOS:** `pkill -f -i "Thymer" || true`
- **Linux:** `pkill -f -i "thymer" || true`
- **Windows:** `taskkill /F /IM Thymer.exe /T 2>$null`

### B. Launching Thymer in Debug Mode (Cross-Platform)

- **macOS:**
  ```bash
  # Using the macOS open command:
  open -a Thymer --args --remote-debugging-port=9444
  
  # Or executing the binary directly:
  /Applications/Thymer.app/Contents/MacOS/Thymer --remote-debugging-port=9444 &
  ```

- **Linux:**
  ```bash
  thymer --remote-debugging-port=9444 &
  ```

- **Windows:**
  ```powershell
  Start-Process "$env:LOCALAPPDATA\Programs\thymer\Thymer.exe" -ArgumentList "--remote-debugging-port=9444"
  # Or via CMD:
  start "" "%LOCALAPPDATA%\Programs\thymer\Thymer.exe" --remote-debugging-port=9444
  ```

### C. Universal Node.js Debug Launcher (`launch-debug.mjs`)
Drop this cross-platform helper script into your plugin repository for effortless one-command launches across all operating systems:
```javascript
import { execSync, spawn } from 'child_process';

// 1. Terminate existing instances (releases single-instance lock)
try {
  if (process.platform === 'win32') {
    execSync('taskkill /F /IM Thymer.exe /T 2>nul');
  } else {
    execSync('pkill -f -i thymer 2>/dev/null || true');
  }
} catch {}

// 2. Launch with debug port
const portArg = '--remote-debugging-port=9444';
let child;

if (process.platform === 'darwin') {
  child = spawn('open', ['-a', 'Thymer', '--args', portArg], { detached: true, stdio: 'ignore' });
} else if (process.platform === 'win32') {
  const exe = `${process.env.LOCALAPPDATA}\\Programs\\thymer\\Thymer.exe`;
  child = spawn(exe, [portArg], { detached: true, stdio: 'ignore' });
} else {
  child = spawn('thymer', [portArg], { detached: true, stdio: 'ignore' });
}

child.unref();
console.log('✓ Thymer Desktop launched with remote debugging on port 9444');
```

> [!WARNING]
> **GUI Session Guardrails:**
> - **macOS:** Must run inside an interactive login session with access to WindowServer. Background daemons without GUI access cannot display windows.
> - **Linux:** Requires `$DISPLAY` (e.g. `:0`) or `$WAYLAND_DISPLAY` set in the environment. For headless CI environments, run under `xvfb-run`.
> - **Windows:** Do not launch from non-interactive background services or daemon tasks (Session 0 isolation), which creates headless background processes with `MainWindowHandle: 0`.

### D. Account Picker Automation & Target Filtering
When connecting via CDP (`http://127.0.0.1:9444/json`), the debugger discovers multiple targets (including `shared_worker`, `service_worker`, and `accounts.html`).
Always filter for the main application window target:
```javascript
const targets = await (await fetch('http://127.0.0.1:9444/json')).json();

// Check if blocked by account selection
const accountsTarget = targets.find(t => t.type === 'page' && t.url.includes('accounts.html'));
if (accountsTarget) {
  // Automate clicking the account tile before continuing
}

// Target the active workspace window
const appTarget = targets.find(t => t.type === 'page' && t.url.includes('thymer.com') && !t.url.includes('accounts.html'));
if (!appTarget) throw new Error('Active Thymer workspace page target not found');
```

### E. Window Resolution Protection (CRITICAL)
> [!CAUTION]
> Connecting Puppeteer or CDP to Electron without explicit configuration overrides the renderer with a default 800x600 viewport, shrinking the user's desktop application window.
> 
> **Mandatory Connection Pattern:**
> 1. Always pass `defaultViewport: null`:
>    ```javascript
>    const browser = await puppeteer.connect({
>      browserURL: 'http://127.0.0.1:9444',
>      defaultViewport: null
>    });
>    ```
> 2. Send `Emulation.clearDeviceMetricsOverride` immediately:
>    ```javascript
>    const p = (await browser.pages())[0];
>    const client = await p.createCDPSession();
>    await client.send('Emulation.clearDeviceMetricsOverride');
>    ```
> 3. **Never** call `page.setViewport()` with arbitrary dimensions.

### F. Hot-Reloading the Desktop UI via CDP
After pushing code updates via MCP, reload the renderer instantly:
```javascript
await cdp.evaluate('window.location.reload()');
```

---

## 8. Self-Updating Pattern & Release Automation

To distribute seamless updates to users:

1. **Background Polling:**
   Poll `https://github.com/<owner>/<repo>/raw/main/plugin.json` periodically (e.g. on startup or every 4 hours).
2. **Non-Intrusive Prompt:**
   If remote `version > currentVersion`, show an in-app banner or toast:
   `A new version vX.Y.Z is available. [Update now] [Later]`
3. **One-Click Upgrade:**
   Fetch `raw/main/dist/plugin.js` and `raw/main/dist/plugin.css` and write them via SDK, then prompt the user to refresh.
4. **Automated Release Pipeline (`npm run release`):**
   - Bump version in `package.json` and `plugin.json`
   - Run `node build.mjs`
   - Git commit & tag (`vX.Y.Z`)
   - Push `main` and tags to GitHub
   - Create GitHub Release via `gh release create vX.Y.Z --generate-notes`

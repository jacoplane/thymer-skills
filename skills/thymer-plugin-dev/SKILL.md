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
```
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

## 2. The Runtime Environment & The Dynamic Eval Trap

> [!CAUTION]
> **Dynamic Evaluator Rule (`new Function`):**
> Thymer Desktop loads plugins dynamically in the renderer process via:
> ```javascript
> new Function('plugin', 'context', bundleCode)(pluginInstance, pluginContext);
> ```
> In JavaScript, `new Function(...)` strictly forbids ES module `export` and `import` declarations.
> Any top-level `export const ...` or `export default ...` will throw a fatal runtime syntax error:
> `SyntaxError: Unexpected token 'export'`
> and prevent the plugin from loading completely.

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

## 3. Plugin SDK Surface & Lifecycle

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

---

## 4. Instant Hot-Deploy via MCP

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

## 5. Electron Remote Debugging & Automated CDP Verification

To allow AI agents to visually inspect, test, and take screenshots of plugins running inside Thymer Desktop:

### 1. Launching Thymer in Debug Mode
Thymer must be launched with the `--remote-debugging-port=9444` flag:
```powershell
& "$env:LOCALAPPDATA\Programs\thymer\Thymer.exe" --remote-debugging-port=9444
```

> [!WARNING]
> **Windows Session Isolation Guardrail:**
> When launched from background services, daemons, or CI workers, Electron may spawn into an isolated session without a visible window (`MainWindowHandle: 0`).
> For local interactive development, launch the process visibly from an active user console or batch script (`start "" "Thymer.exe" --remote-debugging-port=9444`).

### 2. Chrome DevTools Protocol (CDP) Client
Connect to `http://127.0.0.1:9444/json` to discover targets:
```javascript
const targets = await (await fetch('http://127.0.0.1:9444/json')).json();
const appTarget = targets.find(t => t.type === 'page' && t.url.includes('thymer.com'));
const ws = new WebSocket(appTarget.webSocketDebuggerUrl);

// Send CDP evaluate command:
ws.send(JSON.stringify({
  id: 1,
  method: 'Runtime.evaluate',
  params: { expression: 'document.title' }
}));

// Capture full screenshot:
ws.send(JSON.stringify({
  id: 2,
  method: 'Page.captureScreenshot',
  params: { format: 'png' }
}));
```

### 3. Hot-Reloading the Desktop UI via CDP
After pushing code updates via MCP, reload the renderer instantly:
```javascript
await cdp.evaluate('window.location.reload()');
```

---

## 6. Self-Updating Pattern & Release Automation

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

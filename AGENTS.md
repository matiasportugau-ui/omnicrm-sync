# AGENTS.md

## Development environment — OmniCRM Sync Chrome Extension

### Project overview

OmniCRM Sync is a Chrome Extension (Manifest V3) that captures customer interactions from WhatsApp Web, MercadoLibre, Facebook Messenger, and Instagram DMs and syncs them to a configured CRM. Built entirely with vanilla JavaScript, HTML, and CSS — no package manager, no build step, no bundler, no test framework.

### Environment requirements

- **No dependencies to install.** The entire codebase is plain browser JS/HTML/CSS loaded directly by the Chrome Extension runtime.
- **No build step.** Source files are the extension files themselves.
- **No automated test suite.** Testing is done manually by loading the extension in Chrome.

### How to run / test

1. **Load the extension:** Open Chrome, go to `chrome://extensions`, enable Developer mode, click "Load unpacked", and select the repository root directory (the folder containing `manifest.json`).
2. **Validate JS syntax:** Run `node --check <file.js>` on any JS file, or batch-validate all canonical source files:
   ```
   find . -name "*.js" -not -path "*/.git/*" -not -path "*/omnicrm-sync/*" -exec node --check {} \;
   ```
3. **Validate manifest:** Confirm the manifest is valid JSON:
   ```
   node -e "JSON.parse(require('fs').readFileSync('manifest.json','utf8'))" && echo "OK"
   ```

### Codebase notes

- `omnicrm-sync/` is a mirror/backup copy of the extension; the canonical source is at the repository root. Always edit files at the root, then mirror the change to `omnicrm-sync/` if needed.
- `shared/utils.js` uses `globalThis.OmniCRM` so the namespace is accessible in both content-script context (`globalThis === window`) and the background service worker (`globalThis === self`). Do **not** revert this to `window.OmniCRM` — that causes a `ReferenceError` in the service worker.
- The service worker (`background/service-worker.js`) has `/* eslint-env serviceworker */` and `/* global ... */` comments, suggesting ESLint was intended but no `.eslintrc` or `package.json` exists. Run `node --check` for syntax validation instead.
- Extension state is stored in `chrome.storage.local`, `chrome.storage.session`, and browser-local IndexedDB — no server-side storage.

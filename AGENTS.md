# AGENTS.md

## Cursor Cloud specific instructions

### Project overview

OmniCRM Sync is a Chrome Extension (Manifest V3) that captures customer interactions from WhatsApp Web, MercadoLibre, Facebook Messenger, and Instagram DMs and syncs them to a configured CRM. Built entirely with vanilla JavaScript, HTML, and CSS — no package manager, no build step, no bundler, no test framework.

### Development environment

- **No dependencies to install.** The entire codebase is plain browser JS/HTML/CSS loaded directly by the Chrome Extension runtime.
- **No build step.** Source files are the extension files themselves.
- **No automated test suite.** Testing is done manually by loading the extension in Chrome.

### How to run / test

1. **Load the extension:** Open Chrome, go to `chrome://extensions`, enable Developer mode, click "Load unpacked", and select the `/workspace` directory.
2. **Validate JS syntax:** Run `node --check <file.js>` on any JS file, or batch-validate all files:
   ```
   find /workspace -name "*.js" -not -path "*/.git/*" -not -path "*/omnicrm-sync/*" -exec node --check {} \;
   ```
3. **Validate manifest:** `node -e "JSON.parse(require('fs').readFileSync('manifest.json','utf8'))"` confirms valid JSON.

### Known issues

- The service worker (`background/service-worker.js`) uses `importScripts()` to load `shared/utils.js`, which references `window.OmniCRM` at the top level. Since service workers don't have a `window` object, this causes a `ReferenceError: window is not defined` error and the service worker fails to register (Status code 15). The popup and options UI still work independently.

### Codebase notes

- `omnicrm-sync/` is a duplicate/backup copy of the extension; the canonical source is at the repo root.
- The service worker has `/* eslint-env serviceworker */` and `/* global ... */` comments, suggesting ESLint was intended but no config file (`.eslintrc`) or `package.json` exists.
- Extension state is stored in `chrome.storage.local`, `chrome.storage.session`, and browser-local IndexedDB — no server-side storage.

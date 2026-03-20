<div align="center">

<img src="https://img.shields.io/badge/Manifest-V3-4F46E5?style=flat-square" alt="Manifest V3"/>
<img src="https://img.shields.io/badge/Version-1.0.0-22c55e?style=flat-square" alt="v1.0.0"/>
<img src="https://img.shields.io/badge/License-MIT-blue?style=flat-square" alt="MIT"/>
<img src="https://img.shields.io/badge/Platforms-WhatsApp%20%7C%20MercadoLibre%20%7C%20Facebook%20%7C%20Instagram-orange?style=flat-square" alt="Platforms"/>

# OmniCRM Sync

**A Chrome Extension (Manifest V3) that captures customer interactions from WhatsApp Web, MercadoLibre, Facebook Messenger, and Instagram DM, then synchronises them in real-time to your CRM of choice.**

No server infrastructure required. All processing runs locally inside the browser.

</div>

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [Supported Platforms](#supported-platforms)
- [CRM Integrations](#crm-integrations)
- [AI Features (Claude)](#ai-features-claude)
- [Installation](#installation)
- [Configuration](#configuration)
  - [Platform Settings](#platform-settings)
  - [CRM Settings](#crm-settings)
  - [Field Mapping](#field-mapping)
  - [AI Settings](#ai-settings)
- [Data Flow](#data-flow)
- [Project Structure](#project-structure)
- [Technical Design Notes](#technical-design-notes)
- [Privacy & Security](#privacy--security)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

OmniCRM Sync monitors the messaging interfaces of four major platforms directly in the browser. When a new message or conversation event is detected, it is normalised into a standardised interaction object, optionally enriched by Claude AI, and then pushed to a configured CRM endpoint—all without any intermediate server.

The extension is designed for **sales teams, e-commerce operators, and support agents** who work across multiple messaging channels and need a single, structured record of every customer interaction in their CRM.

---

## Key Features

| Category | Capability |
|---|---|
| **Multi-platform capture** | WhatsApp Web, MercadoLibre Mensajes, Facebook Messenger, Instagram DM |
| **CRM dispatch** | Generic Webhook, Google Sheets (Apps Script), HubSpot, Notion, Airtable, Custom REST |
| **AI enrichment** | Message categorisation, conversation summary, reply suggestions, lead scoring via Claude Haiku |
| **Reliable delivery** | IndexedDB-backed queue with exponential-backoff retry (up to 5 attempts, backoff: 2 s → 4 s → 8 s → 16 s) |
| **Field mapping** | Template-based `{{field.path}}` mapping with built-in presets and a visual custom editor |
| **In-page overlay** | Shadow DOM–isolated floating action button (FAB) on every supported platform |
| **Settings UI** | Full options page: per-platform toggles, CRM config, AI config, queue inspector, import/export |
| **Popup** | Quick-access status summary, platform toggles, daily sync counters, pause-all control |
| **MV3 compliance** | Service-worker–based background, no persistent page, state lives in `chrome.storage` + IndexedDB |
| **Multi-locale** | Timestamp parsing supports English, Spanish, and Portuguese relative/absolute formats |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        Chrome Extension                          │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │   WhatsApp   │  │ MercadoLibre │  │  Facebook/Instagram  │   │
│  │ Content Script│  │ Content Script│  │   Content Scripts    │   │
│  │              │  │  (+ ML API)  │  │                      │   │
│  │  Observer    │  │  Observer    │  │  Observers           │   │
│  │  Parser      │  │  Parser      │  │  Parsers             │   │
│  │  Overlay FAB │  │  Overlay FAB │  │  Overlay FABs        │   │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘   │
│         │  chrome.runtime.connect (named port)   │               │
│         └──────────────────┬──────────────────────┘              │
│                            │                                     │
│              ┌─────────────▼──────────────┐                      │
│              │   Background Service Worker│                      │
│              │                            │                      │
│              │  Port Manager              │                      │
│              │  Queue Processor           │                      │
│              │  AI Engine (Claude Haiku)  │                      │
│              │  Data Mapper               │                      │
│              │  CRM Connector             │                      │
│              │  Chrome Alarms (30 s tick) │                      │
│              └─────────────┬──────────────┘                      │
│                            │                                     │
│              ┌─────────────▼──────────────┐                      │
│              │   Persistent Storage       │                      │
│              │  chrome.storage.local      │                      │
│              │  chrome.storage.session    │                      │
│              │  IndexedDB (queue + cache) │                      │
│              └────────────────────────────┘                      │
│                                                                  │
│  ┌──────────────┐   ┌──────────────────────────────────────┐     │
│  │  Popup UI    │   │          Options Page UI             │     │
│  │  (popup.html)│   │  Platforms│CRM│AI│Mapping│Queue│About│     │
│  └──────────────┘   └──────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
                              │ HTTPS
               ┌──────────────▼───────────────┐
               │        CRM Endpoint          │
               │  Webhook / Google Sheets /   │
               │  HubSpot / Notion / Airtable │
               └──────────────────────────────┘
```

### Component Responsibilities

| Component | File(s) | Responsibility |
|---|---|---|
| **Service Worker** | `background/service-worker.js` | Central hub: port management, queue processing, alarm scheduling, CRM dispatch, AI orchestration |
| **Queue Manager** | `shared/queue-manager.js` | IndexedDB CRUD, exponential backoff, AI result cache |
| **Data Mapper** | `shared/data-mapper.js` | Template-based `{{field}}` resolution and preset mapping |
| **CRM Connector** | `shared/crm-connector.js` | HTTP dispatch to all supported CRM backends |
| **AI Engine** | `shared/ai-engine.js` | Claude Haiku API calls with result caching |
| **Storage Manager** | `shared/storage-manager.js` | Abstraction over `chrome.storage.local` + `.session` |
| **Platform Registry** | `shared/platform-registry.js` | Central config (URLs, colours, feature flags) for all platforms |
| **Shared Utils** | `shared/utils.js` | Logging, debounce/throttle, UUID v4, DJB2 hash, port auto-reconnect |
| **Base Parser** | `platforms/base/base-parser.js` | DOM text extraction, timestamp parsing (EN/ES/PT), content-type detection, direction detection |
| **Base Observer** | `platforms/base/base-observer.js` | `MutationObserver` lifecycle management |
| **Per-platform scripts** | `platforms/<platform>/` | Platform-specific selectors, parsers, observers, contact extractors, and content entry points |
| **Overlay** | `ui/overlay/overlay.js` | Shadow DOM FAB with draggable panel, per-platform controls |
| **Popup** | `ui/popup/` | Extension toolbar popup with stats and quick actions |
| **Options** | `ui/options/` | Full settings UI with tabbed navigation |

---

## Supported Platforms

### WhatsApp Web (`web.whatsapp.com`)

- Captures individual and group conversations
- Detects incoming vs. outgoing messages via CSS class patterns
- Parses text, image, video, audio, document, sticker, and link content types
- Configurable: include/exclude groups, capture media metadata, sync direction (both / incoming / outgoing only)

### MercadoLibre Mensajes

Covers all regional domains:
`*.mercadolibre.com.ar` · `*.mercadolibre.com` · `*.mercadolibre.com.mx` · `*.mercadolibre.com.co` · `*.mercadolivre.com.br`

- DOM scraping for conversation messages
- **Optional REST API integration**: authenticates with the MercadoLibre API (OAuth 2.0) to enrich interactions with order ID, product title, order status, and buyer profile
- Built-in rate-limiter respecting 500 GET requests/minute limit (with 10% safety headroom)
- In-memory response cache (30 s–2 min TTL per endpoint)

### Facebook Messenger

Covers `www.facebook.com/messages` and `www.messenger.com`

- DOM scraping; direction detected via `aria-label` and visual alignment
- Supports Facebook Marketplace conversations
- Note: Messenger.com is scheduled for discontinuation in April 2026; primary URL is `facebook.com/messages`

### Instagram DM (`www.instagram.com/direct`)

- DOM scraping of Direct Message threads
- Captures shared posts, reactions, stories, and Reels references
- Username extracted and prefixed with `@` in CRM output

---

## CRM Integrations

All dispatch is handled by `CRMConnector` in the service worker. A 15-second request timeout is enforced on every call.

### Generic Webhook

```
CRM Type: webhook
```

- Sends a `POST` (or any configured HTTP method) with a JSON body
- Adds standard headers: `X-Webhook-Source`, `X-Webhook-Timestamp`, `X-Webhook-Delivery-Id` (UUID v4)
- Supports arbitrary custom headers
- Handles `429 Rate Limited` responses by reading the `Retry-After` header and backing off

### Google Sheets (via Apps Script Web App)

```
CRM Type: google_sheets
```

- POSTs to a deployed Google Apps Script Web App URL
- Handles the `302` redirect that Apps Script returns on success

**Apps Script setup (minimal example):**

```javascript
function doPost(e) {
  const data = JSON.parse(e.postData.contents);
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  sheet.appendRow(Object.values(data));
  return ContentService.createTextOutput('ok');
}
```

### HubSpot

```
CRM Type: hubspot
```

- Creates a contact (`POST /crm/v3/objects/contacts`) with name, phone, and lead source
- Handles `409 Conflict` (contact already exists) as a success
- Optionally creates a note (`POST /crm/v3/objects/notes`) with the message text and timestamp
- Requires a Private App token with `crm.objects.contacts.write` and `crm.objects.notes.write` scopes

### Notion

```
CRM Type: notion
```

- Creates a page in a target database (`POST /v1/pages`)
- Maps all interaction fields to Notion `rich_text` properties; `Timestamp` maps to a `date` property
- The first field in the payload becomes the page `title`
- Uses Notion API version `2022-06-28`

### Airtable

```
CRM Type: airtable
```

- Creates a record (`POST /v0/{baseId}/{table}`)
- Supports all interaction fields as Airtable fields
- Requires a Personal Access Token with `data.records:write` scope on the target base

### Custom REST API

```
CRM Type: custom
```

- Fully configurable URL, HTTP method, and headers
- Sends the mapped JSON payload as the request body

---

## AI Features (Claude)

The AI Engine integrates with **Claude Haiku** (`claude-haiku-4-5-20251001`) via direct API calls. The API key is stored ephemerally in `chrome.storage.session` and is never written to disk.

All AI calls are **optional and non-blocking**—if AI fails, the interaction is still synced to the CRM without enrichment.

### Message Categorisation

Classifies every captured message into one of eight categories:

| Category | Meaning |
|---|---|
| `complaint` | Customer expressing dissatisfaction or reporting a problem |
| `question` | General inquiry or request for information |
| `order_inquiry` | Questions about orders, shipping, tracking, or delivery |
| `feedback` | Positive comments, reviews, or suggestions |
| `urgent` | Requires immediate attention (threats, legal, safety) |
| `greeting` | Simple hellos or conversation starters |
| `closing` | Goodbyes, thank-yous, or conversation endings |
| `other` | Does not fit any of the above |

Returns: `{ category, confidence (0–1), sentiment: "positive|neutral|negative" }`

Results are cached in IndexedDB for **7 days** using a DJB2 hash of the message text as the key.

### Conversation Summarisation

Condenses a full conversation thread into a 1–2 sentence CRM note covering: key topic, customer sentiment, and action items.

### Reply Suggestions

Generates 2–3 short professional reply templates for an incoming message:

```json
[
  { "text": "Thank you for reaching out...", "tone": "professional" },
  { "text": "We're sorry to hear that...", "tone": "apologetic" },
  { "text": "Hi! Happy to help...", "tone": "friendly" }
]
```

### Lead Scoring

Scores a conversation from **1 (cold)** to **10 (hot)** based on sentiment analysis of the last 10 messages, returning a numeric score and a brief reasoning string.

---

## Installation

> **Prerequisites:** Google Chrome with Manifest V3 support. The 30-second minimum alarm period used by this extension requires Chrome 120 or later (as noted in the source code comments).

### Load from source (developer mode)

1. **Clone the repository:**

   ```bash
   git clone https://github.com/matiasportugau-ui/omnicrm-sync.git
   ```

2. **Open Chrome Extensions:**

   Navigate to `chrome://extensions`

3. **Enable Developer Mode:**

   Toggle **Developer mode** on (top-right corner).

4. **Load the extension:**

   Click **Load unpacked** → select the root folder of the cloned repository (the folder containing `manifest.json`).

5. **Verify installation:**

   The OmniCRM Sync icon (indigo circle with a checkmark) should appear in your toolbar. Click it to open the popup.

### Required permissions

| Permission | Reason |
|---|---|
| `storage` | Persist settings and statistics |
| `alarms` | Periodic queue flush and service-worker keep-alive |
| `notifications` | Alert user when CRM sync fails consecutively |
| `unlimitedStorage` | IndexedDB queue can grow large for high-volume accounts |
| `host_permissions` | Inject content scripts into supported platform URLs |

---

## Configuration

All settings are accessible via **Options** (right-click the toolbar icon → *Options*, or click *Open Settings* in the popup).

### Platform Settings

Each platform has an independent enable/disable toggle plus platform-specific options:

**WhatsApp:**
- `Include group messages` — whether to capture group chat activity
- `Capture media metadata` — log image/video/audio events (content type only; no binary data)
- `Sync Direction` — `Both`, `Incoming only`, or `Outgoing only`

**MercadoLibre:**
- `Enable` — master toggle
- `API Mode` — when enabled, enriches interactions with order and buyer data via the REST API (requires OAuth credentials in the CRM tab)

**Facebook / Instagram:**
- `Enable` — master toggle
- Facebook additionally supports `Include Marketplace` conversations

### CRM Settings

Navigate to the **CRM** tab in Options. Select the CRM type from the dropdown; the form will update to show the relevant fields.

| CRM Type | Required Fields |
|---|---|
| `webhook` | Webhook URL, HTTP Method (default: POST), optional custom headers |
| `google_sheets` | Apps Script Web App URL |
| `hubspot` | Private App Token |
| `notion` | Integration Token, Database ID |
| `airtable` | Personal Access Token, Base ID, Table Name |
| `custom` | URL, HTTP Method, optional custom headers |

### Field Mapping

Navigate to the **Field Mapping** tab. Choose a built-in preset or design a custom mapping using `{{field.path}}` templates.

**Built-in presets:**

| Preset | Best for |
|---|---|
| `google_sheets_all_platforms` | Google Sheets or any webhook; includes all standard fields |
| `hubspot_contact_note` | HubSpot contact + note creation |
| `notion_database_entry` | Notion database pages |
| `airtable_row` | Airtable base records |
| `raw_json` | Sends the raw interaction object as-is |

**Available template fields:**

```
{{timestamp}}              ISO 8601 timestamp of the interaction
{{platform}}               whatsapp | mercadolibre | facebook | instagram
{{direction}}              incoming | outgoing
{{sender.name}}            Display name of the contact
{{sender.identifier}}      Phone number, username, or user ID
{{content.text}}           Message text (cleaned of HTML/emoji markup)
{{content.type}}           text | image | video | audio | document | sticker | link | system
{{conversation.name}}      Chat or thread name
{{conversation.type}}      individual | group
{{context.orderId}}        MercadoLibre order ID (if applicable)
{{context.productTitle}}   MercadoLibre product name (if applicable)
{{status}}                 Interaction status
{{ai.category}}            AI category (if AI is enabled)
```

**Custom mapping example (Google Sheets with custom column names):**

```json
{
  "Date": "{{timestamp}}",
  "Channel": "{{platform}}",
  "Customer": "{{sender.name}}",
  "Phone": "{{sender.identifier}}",
  "Message": "{{content.text}}",
  "Order": "{{context.orderId}}",
  "Category": "{{ai.category}}"
}
```

### AI Settings

Navigate to the **AI (Claude)** tab.

1. **Enable AI** — master toggle
2. **Claude API Key** — enter your [Anthropic API key](https://console.anthropic.com/). The key is stored only in `chrome.storage.session` (cleared when the browser is closed) and is never written to `chrome.storage.local`.
3. **Categorise messages** — run categorisation on every captured message
4. **Summarise conversations** — generate a summary note per conversation
5. **Suggest replies** — produce reply templates for incoming messages

**Cost note:** The extension uses Claude Haiku, the most cost-efficient Claude model, with a `max_tokens` limit of 150. Categorisation results are cached for 7 days to minimise repeat API calls.

---

## Data Flow

```
Platform Website (e.g., web.whatsapp.com)
    │
    │  MutationObserver detects new message node
    ▼
Content Script (wa-observer.js → wa-parser.js)
    │
    │  Extracts: sender, text, timestamp, direction, content type
    │  Builds standardised interaction object:
    │  {
    │    id: UUID,
    │    platform: "whatsapp",
    │    timestamp: "2026-03-20T10:00:00.000Z",
    │    direction: "incoming",
    │    sender: { name, identifier },
    │    content: { text, type },
    │    conversation: { name, type }
    │  }
    │
    │  chrome.runtime.connect (port-whatsapp)
    ▼
Background Service Worker
    │
    │  1. Receives NEW_INTERACTION message
    │  2. Enqueues to IndexedDB (status: "pending")
    │  3. Acknowledges back to content script (INTERACTION_QUEUED)
    │  4. Triggers processQueue() immediately
    │
    ▼
processQueue() — runs every 30 s via chrome.alarms OR on-demand
    │
    │  For each pending item:
    │  ┌─────────────────────────────────────┐
    │  │ Is platform rate-limited?  → skip   │
    │  │                                     │
    │  │ AI enabled?                         │
    │  │   → categorizeMessage()             │
    │  │     (cache check → Claude API)      │
    │  │   → attach ai.category/sentiment    │
    │  │                                     │
    │  │ dataMapper.map(interaction, preset) │
    │  │   → resolve {{template}} fields     │
    │  │   → apply platform-specific extras  │
    │  │                                     │
    │  │ crmConnector.send(payload, config)  │
    │  │   → HTTP request (15 s timeout)     │
    │  │                                     │
    │  │ Success → markSuccess(), update stats│
    │  │ Failure → markFailed()              │
    │  │   retries ≤ 5: exponential backoff  │
    │  │   retries > 5: status = "dead"      │
    │  └─────────────────────────────────────┘
    │
    ▼
CRM Endpoint (Webhook / Sheets / HubSpot / Notion / Airtable)
```

---

## Project Structure

```
omnicrm-sync/
├── manifest.json                   # Extension manifest (MV3)
├── LICENSE
├── README.md
│
├── assets/
│   ├── icons/
│   │   ├── icon16.png
│   │   ├── icon48.png
│   │   └── icon128.png
│   └── platform-icons/
│       ├── whatsapp.svg
│       ├── mercadolibre.svg
│       ├── facebook.svg
│       └── instagram.svg
│
├── background/
│   └── service-worker.js           # MV3 service worker (central hub)
│
├── shared/
│   ├── utils.js                    # Logging, debounce, UUID, hash, port helpers
│   ├── storage-manager.js          # chrome.storage abstraction + defaults
│   ├── platform-registry.js        # Platform config (URLs, colours, features)
│   ├── queue-manager.js            # IndexedDB queue + AI cache
│   ├── data-mapper.js              # {{template}} field mapping + presets
│   ├── crm-connector.js            # HTTP dispatch to CRM backends
│   └── ai-engine.js                # Claude Haiku API integration
│
├── platforms/
│   ├── base/
│   │   ├── base-selectors.js       # Base CSS selector utilities
│   │   ├── base-parser.js          # Text extraction, timestamp parsing, direction detection
│   │   └── base-observer.js        # MutationObserver lifecycle
│   │
│   ├── whatsapp/
│   │   ├── wa-selectors.js         # WhatsApp DOM selector constants
│   │   ├── wa-parser.js            # WhatsApp message parsing
│   │   ├── wa-contact.js           # Contact extraction (name, phone)
│   │   ├── wa-observer.js          # Conversation and message observer
│   │   └── wa-content.js           # Entry point: initialises observer + overlay
│   │
│   ├── mercadolibre/
│   │   ├── ml-selectors.js         # MercadoLibre DOM selector constants
│   │   ├── ml-parser.js            # Message parsing
│   │   ├── ml-api.js               # REST API client (OAuth 2.0, rate limiting, cache)
│   │   ├── ml-contact.js           # Buyer/seller contact extraction
│   │   ├── ml-observer.js          # Conversation observer
│   │   └── ml-content.js           # Entry point
│   │
│   ├── facebook/
│   │   ├── fb-selectors.js
│   │   ├── fb-parser.js
│   │   ├── fb-contact.js
│   │   ├── fb-observer.js
│   │   └── fb-content.js
│   │
│   └── instagram/
│       ├── ig-selectors.js
│       ├── ig-parser.js
│       ├── ig-contact.js
│       ├── ig-observer.js
│       └── ig-content.js
│
└── ui/
    ├── popup/
    │   ├── popup.html              # Toolbar popup
    │   ├── popup.css
    │   └── popup.js               # Popup logic (stats, toggles, pause-all)
    │
    ├── options/
    │   ├── options.html            # Settings page (tabbed)
    │   ├── options.css
    │   └── options.js             # Settings logic (all tabs)
    │
    └── overlay/
        ├── overlay.js             # Shadow DOM FAB + panel (injected into platforms)
        └── overlay.css            # Fallback styles (actual styles are in JS)
```

---

## Technical Design Notes

### Manifest V3 Service Worker lifecycle

Chrome MV3 service workers are terminated after ~30 seconds of inactivity. OmniCRM Sync addresses this with:

- A `keepAlive` alarm that fires every 30 seconds to prevent premature termination during active sync sessions.
- A `queueFlush` alarm every 30 seconds as a safety net for any items that were not processed immediately.
- **Zero global state**: all runtime state is stored in `chrome.storage.local`, `chrome.storage.session`, or IndexedDB. The service worker is fully stateless across restarts.

### IndexedDB Queue

The `QueueManager` maintains two IndexedDB object stores:

| Store | Key | Indexes | Purpose |
|---|---|---|---|
| `queue` | `id` (UUID) | `status`, `platform`, `createdAt` | Reliable interaction delivery queue |
| `ai_cache` | `hash` (DJB2) | `expiresAt` | 7-day AI categorisation result cache |

Queue item lifecycle:
```
pending → (success) → sent → (cleanup after 24 h) → deleted
pending → (failure, retries < 5) → pending (with nextRetryAt)
pending → (failure, retries ≥ 5) → dead
```

Backoff schedule (4 intervals, 5 total attempts): `2 s → 4 s → 8 s → 16 s`

### Shadow DOM Overlay

The in-page FAB is mounted inside a **closed Shadow DOM** (`attachShadow({ mode: 'closed' })`), which means:

- The overlay's CSS is completely isolated from the host page's stylesheets.
- The overlay's DOM is inaccessible to the host page's JavaScript.
- Keyboard events (keydown/keyup/keypress) are stopped from propagating to the host page so that typing in the overlay panel does not trigger WhatsApp/Instagram keyboard shortcuts.

The FAB is draggable via Pointer Events API and respects `prefers-color-scheme` for automatic dark mode.

### Content Deduplication

Each message is hashed using the **DJB2** algorithm (`shared/utils.js → OmniCRM.contentHash`). The hash is used both for AI cache lookups and as a deduplication key in the queue—if an identical message text is enqueued twice in quick succession, the second enqueue is a no-op.

### Multi-locale Timestamp Parsing

`BaseParser.parseTimestamp()` handles timestamps in three languages:

| Format | Examples |
|---|---|
| ISO 8601 | `2026-03-20T10:00:00.000Z` |
| Unix (s or ms) | `1742464800`, `1742464800000` |
| English relative | `5 minutes ago`, `2 hours ago`, `Yesterday`, `Today` |
| Spanish relative | `hace 5 min`, `hace 2 horas`, `Ayer`, `Hoy` |
| Portuguese relative | `há 5 min`, `há 2 horas`, `Ontem`, `Hoje` |
| Time only | `12:45 PM`, `14:30` |
| Date | `dd/mm/yyyy`, `mm/dd/yyyy` |

### MercadoLibre API Rate Limiting

The `MLApiClient` tracks rolling request timestamps within a 60-second window and enforces a safe limit of 450 requests/minute (90% of the 500 req/min allowance) with automatic back-pressure. Token refresh follows the standard OAuth 2.0 `refresh_token` grant.

---

## Privacy & Security

- **No external server**: all processing occurs in-browser. Captured data is sent directly from the extension to your configured CRM endpoint.
- **API key storage**: the Claude API key is stored exclusively in `chrome.storage.session`, which is cleared when Chrome closes. It is never logged, never written to `chrome.storage.local`, and never included in exported settings.
- **MercadoLibre tokens**: OAuth tokens are stored in `chrome.storage.local`. The code includes a `_persistTokens` placeholder with a `TODO` comment for AES-GCM encryption via the Web Crypto API—this should be implemented before production deployment.
- **No remote code execution**: the extension does not load any remote scripts. All JavaScript is bundled locally.
- **Content script isolation**: each content script namespace is isolated under the `window.OmniCRM` object; a guard (`window.__omnicrm_*_initialized`) prevents double-initialisation on page re-renders.
- **Host permissions** are scoped to exactly the domains required by each platform—no broad `<all_urls>` permission is requested.

---

## Contributing

Contributions are welcome. Please follow these guidelines:

1. **Fork** the repository and create a feature branch: `git checkout -b feat/my-feature`
2. **Follow the existing code style**: vanilla JavaScript (ES2020+), JSDoc comments on all public methods, platform-specific files prefixed with the platform abbreviation (`wa-`, `ml-`, `fb-`, `ig-`).
3. **Adding a new platform**: create a folder under `platforms/`, implement `*-selectors.js`, `*-parser.js`, `*-contact.js`, `*-observer.js`, and `*-content.js` extending the base classes. Register the platform in `shared/platform-registry.js` and add a content script entry in `manifest.json`.
4. **Adding a new CRM**: add a `_send<CRMName>` method in `shared/crm-connector.js`, a corresponding field-mapping preset in `shared/data-mapper.js`, and update the CRM tab in `ui/options/options.html`.
5. **Test** your changes against the live website of the affected platform—DOM-based parsing is inherently fragile and must be verified manually.
6. **Open a pull request** with a clear description of the change and screenshots if the UI is affected.

### Known limitations and future work

- MercadoLibre OAuth token encryption (Web Crypto AES-GCM) is stubbed with a `TODO`.
- Facebook Messenger.com support will need updates after its April 2026 shutdown.
- There are no automated tests; a DOM-mocking test harness (e.g., jsdom + Jest) would significantly improve regression coverage.
- The overlay position is not persisted between page loads.

---

## License

[MIT](LICENSE) © matiasportugau-ui

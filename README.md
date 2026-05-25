<h1 align="center">Job Search Lens</h1>

<p align="center">
  A Chrome extension (Manifest V3) that highlights saved keywords on any job site and adds focused tooling for LinkedIn Jobs — dim processed cards, see company stats inline, navigate matches, manage your keyword library.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/version-1.3.0-0f766e?style=flat-square" alt="Version 1.3.0" />
  <img src="https://img.shields.io/badge/Chrome-Manifest%20V3-4285F4?style=flat-square&logo=googlechrome&logoColor=white" alt="Chrome Manifest V3" />
  <img src="https://img.shields.io/badge/privacy-local--only-0F9D58?style=flat-square" alt="Local-only" />
  <img src="https://img.shields.io/badge/tests-30%20passing-brightgreen?style=flat-square" alt="30 tests passing" />
  <img src="https://img.shields.io/badge/license-MIT%20%2B%20Commons%20Clause-blue?style=flat-square" alt="License" />
</p>

<p align="center">
  <a href="https://madhushanandawaththa.github.io/job-search_lens/">Product site</a> ·
  <a href="https://madhushanandawaththa.github.io/job-search_lens/support.html">Docs</a> ·
  <a href="https://madhushanandawaththa.github.io/job-search_lens/privacy-policy.html">Privacy policy</a> ·
  <a href="https://chrome.google.com/webstore/category/extensions">Chrome Web Store</a>
</p>

---

## What it does

Job Search Lens runs silently in the browser while you look for work:

- **Keyword highlighting on any site** — save terms (skills, roles, locations) once; they light up on LinkedIn, Indeed, Seek, Glassdoor, or any site you visit. Assign a color to each keyword, navigate between matches, export your list.
- **LinkedIn Jobs — dim processed cards** — LinkedIn already marks cards as Viewed, Saved, or Applied but never hides them. The extension fades them automatically, with independent toggles per state.
- **Company stats inline** — employee count range and LinkedIn member count appear directly next to each job title so you can judge company fit without opening a separate tab.
- **Right-click to add** — select any text on any page and right-click → "Add as keyword". No popup required.
- **Auto / Light / Dark theme** — synchronized across popup renders with no flash on open.

Everything stays local. No backend, no accounts, no telemetry.

---

## Architecture

The extension is structured as four runtime surfaces that share a single logic layer:

```
┌─────────────────────────────────────────────────────┐
│                   chrome.storage.local               │  ← single source of truth
└──────┬───────────────┬──────────────────┬────────────┘
       │               │                  │
  ┌────▼────┐   ┌──────▼──────┐   ┌──────▼──────┐
  │  Popup  │   │  Background │   │   Content   │
  │  UI     │   │  Worker     │   │   Script    │
  │         │   │  (MV3 SW)   │   │             │
  └────┬────┘   └──────┬──────┘   └──────┬──────┘
       │               │                  │
       └───────────────▼──────────────────┘
                  shared.js
          (keyword logic, storage helpers,
           sanitization, regex builders)
```

- **`shared.js`** — pure, framework-free module consumed by popup, background, and content script. Keeps storage key names, keyword normalization, regex compilation, settings hydration, and color utilities in one place.
- **`content.js`** — orchestrates highlighting across all pages; LinkedIn-specific features (dimming, company stats, route change detection) are gated behind URL checks so non-LinkedIn pages pay no overhead.
- **`background.js`** — MV3 service worker; recreates the context menu on both `onInstalled` and `onStartup` to survive worker restarts (MV3 workers are short-lived by design).
- **`dom-heuristics.js`** — isolated DOM detection helpers for LinkedIn's results-list structure, extracted so they can be tested independently of the full content-script runtime.
- **`theme-init.js`** — synchronous, inline-free script that applies the saved theme class before first paint, eliminating flash-of-wrong-theme on popup open.

---

## Technical challenges

### LinkedIn SDUI compatibility

LinkedIn periodically ships a server-driven UI variant of the Jobs results page that restructures the DOM without changing the URL. The extension maintains separate detection paths for the classic layout and the SDUI layout, promoted lazily — so it handles both without brittle version-sniffing.

### SPA navigation without a router hook

LinkedIn is a single-page application. URL changes happen through the History API but there is no reliable `navigate` event in Chrome extensions at MV3. The extension hooks `pushState` / `replaceState` through a page-world script to detect route transitions and re-run LinkedIn-specific logic when the jobs URL changes, without polling.

### Highlight correctness under continuous DOM mutation

LinkedIn's feed is aggressively lazy-loaded. The extension uses a `MutationObserver` on the results container (scoped, not on `document.body`) and a signature-based deduplication system to avoid re-highlighting nodes it has already processed — keeping CPU usage flat as new cards stream in.

### Universal highlighting with LinkedIn isolation

The extension has broad host permissions (required for cross-site keyword highlighting) but LinkedIn-specific mutations (dimming, company stats, route hooks) are gated at runtime. A non-LinkedIn page runs only the generic highlight pass — no LinkedIn DOM queries, no observer for LinkedIn-specific nodes.

### CSP-compliant popup theming

Chrome's MV3 content security policy disallows inline scripts in HTML files. The theme bootstrap reads from `localStorage` (mirrored from `chrome.storage.local` on every save) in an external script loaded as the last tag in `<head>`, so the correct theme class is applied before the popup body renders.

---

## Test suite

30 automated tests run under Node's built-in test runner (`node --test`) with JSDOM. No external test framework dependency.

Coverage includes:

| Area | What's tested |
|---|---|
| `shared.js` | Keyword normalization, deduplication, hex validation, contrast selection, regex compilation, settings sanitization, storage helpers |
| `dom-heuristics.js` | Results-list container detection across 6 LinkedIn layout scenarios including the SDUI variant and wrapped job-title markup |
| Content script — generic | Keyword highlighting on non-LinkedIn pages: mark insertion, re-highlight on mutation, navigation through matches |
| Content script — LinkedIn | Company stats extraction and injection for classic, SDUI, and wrapped-title layouts |

Scenario tests load real captured LinkedIn HTML snapshots via JSDOM to guard against regressions when LinkedIn updates its markup.

---

## Privacy by design

The extension deliberately has no server-side component.

| What it does not do | How that's enforced |
|---|---|
| No telemetry or analytics | No network calls of any kind |
| No remote code execution | All JS ships in the extension package; CSP blocks `eval` and remote scripts |
| No user accounts | No `identity` permission, no OAuth |
| No cloud sync | `chrome.storage.sync` is intentionally not used |
| No data sale or sharing | Nothing leaves the browser |

All user data — keywords, colors, dim-state preferences — lives in `chrome.storage.local` on the user's device.

---

## Permissions

The extension requests the minimum set of permissions needed for its features.

| Permission | Why |
|---|---|
| `contextMenus` | Right-click "Add as keyword" on selected text |
| `storage` | Save keywords, colors, and settings locally |
| `activeTab` | Let the popup query the active tab for match count and status |
| `http://*/*`, `https://*/*` | Required for keyword highlighting to run on any site the user visits |

---

## Stack

| Layer | Technology |
|---|---|
| Extension platform | Chrome Manifest V3 |
| Runtime | Vanilla JavaScript (ES2020+), no frameworks |
| DOM | Native browser APIs, MutationObserver, History API hooks |
| Storage | `chrome.storage.local`, `localStorage` (theme mirror only) |
| Testing | Node `--test`, JSDOM |
| CI / assets | PowerShell (icon generation), GitHub Actions (planned) |
| Site | Static HTML/CSS, GitHub Pages |

---

## Project structure

```text
job-search-lens/          ← private; source of truth for the extension
├── background.js         — MV3 service worker: context menu, storage writes
├── content.js            — cross-site highlights + LinkedIn-specific features
├── dom-heuristics.js     — testable LinkedIn DOM detection helpers
├── shared.js             — shared keyword/storage/regex logic
├── popup.html/js         — popup UI and behavior
├── theme-init.js         — pre-paint theme bootstrap
├── styles.css            — highlight + dim CSS injected into pages
├── manifest.json
├── tests/                — 30 tests: shared, heuristics, content, scenarios
└── docs/                 → deployed to this repo (job-search_lens) for GitHub Pages
```

The source code is in a separate private repository. This public repo hosts only the product site.

---

## Links

- **Product site** → [madhushanandawaththa.github.io/job-search_lens](https://madhushanandawaththa.github.io/job-search_lens/)
- **Support / docs** → [madhushanandawaththa.github.io/job-search_lens/support.html](https://madhushanandawaththa.github.io/job-search_lens/support.html)
- **Privacy policy** → [madhushanandawaththa.github.io/job-search_lens/privacy-policy.html](https://madhushanandawaththa.github.io/job-search_lens/privacy-policy.html)
- **Chrome Web Store** → listing pending approval

---

<p align="center">Built by <a href="https://linkedin.com/in/thiwankamadhushan">Madhushan Andawaththa</a></p>

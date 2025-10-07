# WebMail UI – Responsive Multi-Account Mail Viewer

<p align="center">
  <a href="LICENSE"><img alt="License" src="https://img.shields.io/badge/License-MIT-blue.svg"></a>
  <a href="https://github.com/antwal/bootstrap-webmail-ui/commits/main"><img alt="Last Commit" src="https://img.shields.io/github/last-commit/antwal/bootstrap-webmail-ui.svg"></a>
  <a href="https://github.com/antwal/bootstrap-webmail-ui"><img alt="Code Size" src="https://img.shields.io/github/languages/code-size/antwal/bootstrap-webmail-ui.svg"></a>
</p>

---

## Live Preview / Quick Test
| Method | URL | Notes |
|--------|-----|-------|
| GitHub Pages | https://antwal.github.io/bootstrap-webmail-ui/webmail.html | Static build from main branch |
| GitHack (prod CDN) | https://rawcdn.githack.com/antwal/bootstrap-webmail-ui/main/webmail.html | CDN with aggressive caching |
| GitHack (fast dev) | https://raw.githack.com/antwal/bootstrap-webmail-ui/main/webmail.html | Near real-time updates |

## Overview
WebMail UI is a self‑contained single-file (HTML + CSS + JS + `<template>` blocks) responsive webmail interface that simulates a multi‑account mailbox. It implements lazy pagination, internationalization (i18n), attachment rendering via semantic templates, theme switching, accessibility enhancements, a DOM translation scanner, and a resilient fetch strategy (`mockFetch`) that attempts real network calls before falling back to locally defined mock data.

The project is ideal for:
- Rapid prototyping of an email client UX
- Embedding inside admin consoles or internal tools
- Teaching pagination, caching, i18n, and state management patterns
- Gradual migration from mock-only to real backends (progressive enhancement)

---

## Key Features
- **Multi-Account Support**: Switch account context; per-account folder availability.
- **Dynamic Folders**: Inbox, Sent, Drafts, Spam, Trash (subset per account).
- **Lazy Pagination + Infinite Scroll**: Page-based loading (20 items/page by default).
- **Runtime i18n**: On-demand language loading (it, en, fr, de); nested key resolution.
- **Robust Translation Fallback**: translation → `aria-label` → existing content/placeholder → key.
- **Theme Switching**: Light/Dark with persistence via `localStorage`.
- **Responsive Adaptive Layout**: Mobile panel switching + injected back button.
- **Attachment Rendering (Template-Based)**: Uses `<template>` cloning (icons or image previews) for improved accessibility and maintainability.
- **Graceful Image Preview Fallback**: Broken attachment image previews automatically degrade to a generated placeholder.
- **State-Driven Rendering**: No frameworks required; declarative update functions.
- **Single-File Deployment**: Drop into any static hosting or embed in a CMS.
- **Progressive Loading**: Only the active folder’s first page is loaded initially.
- **Accessible Foundations**: ARIA labels auto-assigned when missing on interactive elements.
- **Debug Translation Scanner**: `scanTranslateKeys` + `scanDataset` for i18n coverage auditing.
- **Resilient Fetch Layer**: `mockFetch` first tries real endpoints, then gracefully falls back to mock data—safe to retain in production.
- **Memory-Conscious Account Switching**: Previous account email cache is discarded when switching accounts to bound memory growth.

---

## Architecture Overview
| Layer | Responsibility |
|-------|---------------|
| Markup | Navbar, panels, list, content view, templates |
| Styles | Embedded minimal CSS layered on Bootstrap 5 |
| Templates | Language item, email list item, email content, attachments, folders |
| Mock API | `mockApi` object storing deterministic endpoint payloads |
| Mock Generators | Randomized attachments & emails + paginated slicing |
| State | `currentState` (UI status) + `mockDatabase` (cache) |
| Rendering | `renderAccounts`, `renderFolders`, `renderEmailList`, `renderEmailContent`, etc. |
| Interaction | Account/folder switching, email selection, infinite scroll |
| i18n | `getTranslation`, `applyTranslations`, fallback cascade |
| Pagination | `loadEmails`, `loadMoreEmails` controlling append behavior |
| Initialization | `initializeApplication` orchestrates bootstrap sequence |

---

## Data Model & Mock API Schema

### Language List Endpoint
```
GET /api/translations
→ Array<{
  code: string,
  name: string,
  flag: string   // 16x12 country flag image URL
}>
```

### Per-Language Translation Endpoint
```
GET /api/translations?l=<lang>
{
  page_title: string,
  navbar_brand: string,
  search_placeholder: string,
  search_button: string,
  settings_button: string,
  theme_header: string,
  theme_light: string,
  theme_dark: string,
  language_header: string,
  compose_button: string,
  select_email_placeholder: string,
  reply_button: string,
  forward_button: string,
  delete_button: string,
  from_label: string,
  to_label: string,
  subject_label: string,
  back_button: string,
  no_emails: string,
  attachments_name: string,
  loading: string,
  folders: {
    inbox: string,
    sent: string,
    drafts: string,
    spam: string,
    trash: string
  }
}
```

### Accounts
```
GET /api/accounts
→ Array<{ id: string, email: string }>
```

### Folders (Per Account)
```
GET /api/folders
→ Array<{
  "<accountId>": string[]  // e.g. ['inbox','sent','drafts']
}>
```

### Paginated Emails
```
GET /api/emails?account=<id>&folder=<folder>&page=<n>&pageSize=<n>
{
  page: number,
  pageSize: number,
  total: number,
  items: Array<Email>
}

Email {
  id: number,
  from: string,
  to: string,
  subject: string,
  snippet: string,
  body: string (HTML),
  unread: boolean,
  date: ISOString,
  attachments: Array<Attachment>
}

Attachment {
  name: string,
  size: string,    // Human readable already (e.g. "1.2 MB")
  type: string,    // Extension
  url: string      // Mock preview/download link
}

// NOTE: Some translation keys (e.g. compose_button, reply_button, forward_button, delete_button, subject_label)
// exist in dictionaries but are not yet surfaced in the current UI; they are reserved for future actions/components.
```

---

## How `mockFetch` Works (Production-Safe Strategy)

Unlike a pure mock layer, `mockFetch(url, options)` follows this strategy:
1. Calls the real `fetch(url, options)`
2. If the response is `404` OR a network error occurs:
   - Looks up a matching key in the in-memory `mockApi`
   - If found, returns a synthetic Response-like object with a `.json()` resolver
3. If not found, rejects with an error.

### Why You Can Keep `mockFetch` in Production
Because it attempts a real request first, you can:
- Deploy the same file into production
- Gradually implement backend endpoints
- Let unimplemented endpoints seamlessly fall back to mock data during migration

### Using It in Production Without Full Refactors
- Keep `mockFetch` as-is while you implement real endpoints progressively.
- Once all endpoints are implemented (and stable), optionally:
  - Replace calls to `mockFetch` with native `fetch`
  - Or disable fallback by adding a toggle (e.g. `ENABLE_MOCK_FALLBACK=false`)

### Functions Safe to Remove for a Production (Backend-Integrated) Build
Remove these when the real backend provides data:

| Category | Function / Block | Purpose |
|----------|------------------|---------|
| Mock generation | `generateMockAttachments` | Random attachments |
| Mock generation | `generateMockEmails` | Synthetic email corpus |
| Mock API builder | IIFE `(function buildMockApi(){...})()` | Populates paginated mock endpoints |
| Static mock dataset | The `mockApi` object entries (except keep shape doc for reference) |
| Debug utilities | `scanTranslateKeys`, `scanDataset` | Translation audit tooling |
| Logging (optional) | Console `console.log` / `console.warn` statements | Noise reduction |
| Placeholder content | Long lorem HTML in email body | Replace with sanitized real content |

### Example Endpoint Contracts (From Current Mock)
(Use these as backend contract references)
```
/api/translations                -> Array<LangMeta>
/api/translations?l=en           -> TranslationDictionary
/api/accounts                    -> Array<Account>
/api/folders                     -> Array<{accountId: string[]}>
/api/emails?account=..&folder=.. -> PaginatedEmailSet
```

---

## Rendering Flow
1. `initializeApplication()`
2. Parallel load: languages list, accounts, folders
3. Load translation dictionary for active language
4. Load first page of default folder (Inbox)
5. Render accounts → folders → email list (page 1) → placeholder/content view
6. On scroll near bottom: `loadMoreEmails()` fetches & appends next page

---

## State Model
```js
currentState = {
  language: 'it',
  theme: 'light',
  activeAccount: string,
  activeFolder: 'inbox' | 'sent' | 'drafts' | 'spam' | 'trash',
  selectedEmailId: number | null,
  isLoading: boolean
}

mockDatabase = {
  accounts: Array<Account>,
  emails: {
    [accountId]: {
      [folderKey]: {
        items: Email[],
        total: number,
        pagesLoaded: number
      }
    }
  }
}
```

---

## Internationalization (i18n)
- Language packs loaded lazily per selection.
- Keys resolved with dot-notation path traversal.
- Missing key fallback chain: translation → `aria-label` → existing node text/placeholder → raw key.
- Dev tools: `scanDataset()` to enumerate all `data-translate-key` attributes (templates included).
- To audit coverage after adding new templates or dynamic components, re-run `scanDataset()` in the browser console.
- Unused keys (e.g. action buttons not yet displayed) are intentionally kept to ease future feature expansion.

---

## Attachment Handling (Template-Based)
- Uses `<template>` cloning for each attachment (better semantics & future extension).
- Detects image types (`jpg,jpeg,png,gif,webp`) → inline preview w/ fallback.
- Other file types map to Bootstrap Icon classes via `getFileIcon`.
- File names truncated with CSS ellipsis; full name in `title`.
- Section header includes dynamic count + translatable label.
- Failed image loads trigger a placeholder substitution (`placehold.co`) to preserve layout coherency.

---

## Performance Characteristics
| Aspect | Current Approach | Potential Upgrade |
|--------|------------------|------------------|
| Pagination | Fixed page size (20) | Adaptive / server-driven |
| Infinite Scroll | Scroll position threshold (50px) | IntersectionObserver |
| Memory Growth | Retains all loaded pages | Virtual list + disposal |
| Rendering | Template cloning | Batch DocumentFragments |
| Email Body | Full HTML kept | Lazy hydrate on selection |
| Logging | Verbose `console.log` | Env-guarded logging |
| Cross-folder email lookup | Linear search across cached folders on open | Index map by ID per account |

---

## Accessibility
- ARIA labels on interactive elements (auto-injected where missing)
- Semantic heading & region structure
- Suggested future improvements:
  - Focus management on panel switch
  - Keyboard shortcuts (j/k navigation, Enter to open)
  - `aria-live` region for newly appended emails
  - Skip-to-content anchor
  - Programmatic focus restoration after opening an email on mobile

---

## Debug & Developer Utilities
| Tool | Purpose |
|------|---------|
| `scanTranslateKeys` | Deep scans document + templates for translation key coverage |
| `scanDataset` | Convenience wrapper to log summary report |
| Console logging | Observability for state + fallback behavior |

---

## Security Notes
| Risk | Current Behavior | Mitigation Recommendation |
|------|------------------|---------------------------|
| XSS via email body | Direct `innerHTML` injection | Sanitize (DOMPurify) or pre-sanitize server-side |
| Unsigned attachment URLs | Raw mock strings | Signed URLs or tokenized endpoints |
| CSP absence | No CSP configured | Add strict `Content-Security-Policy` headers |
| Remote tracking images | All inline images load | Option to block external images / proxy |
| Missing auth | None implemented | Add proper auth & session handling before production |
| Error exposure | Console-only | User-friendly toast / inline error states |
| No rate limiting | Unlimited pagination calls | Introduce client throttle + server rate limits |

---

## Known Issues & Areas for Improvement
1. Unread count derives only from currently loaded pages (not authoritative).
2. No server-driven search (UI search box placeholder only)
3. Hardcoded date placeholder in template (`01-02-2025`) – dynamic formatting not yet implemented.
4. No message compose / reply / forward workflows.
5. No bulk selection / batch actions.
6. No threading / conversation grouping.
7. Only theme preference persists; language & state not persisted.
8. Scroll threshold is static (50px).
9. No offline / retry / backoff logic.
10. No real error UI (console only).
11. Body HTML not sanitized (security risk).
12. Translation dictionaries loaded lazily; no prefetch strategy
13. Unused translation keys for future actions (reply/forward/delete/compose/subject_label) remain in dictionaries.

---

## TODO (Roadmap Style)
| Priority | Task | Description / Notes |
|----------|------|---------------------|
| High | Sanitize email body | Integrate DOMPurify or server pre-sanitization. |
| High | Implement dynamic email date handling | Replace hardcoded template date; use `Intl.DateTimeFormat` per locale. |
| High | Real backend integration layer | Gradually map endpoints; keep `mockFetch` fallback initially. |
| High | Server authoritative unread counts | Add endpoint returning unread summary per folder. |
| High | Normalize / prune unused translation keys OR implement associated UI | Avoid confusion & reduce payload size if actions stay deferred. |
| Medium | Search & filtering | Client-side debounce + server query support. |
| Medium | Virtual list / windowing | Prevent DOM bloat for large mailboxes. |
| Medium | Bulk selection & actions | Shift-click + checkboxes + batch operations. |
| Medium | Keyboard navigation | Roving tabindex, shortcuts (j/k, Enter). |
| Medium | Configurable scroll threshold | Dynamic based on viewport height / latency. |
| Medium | Attachment caching & preview optimization | Cache generated fragments and fallback placeholders. |
| Low | Composer & drafts management | Modal or split-pane editor + draft persistence. |
| Low | Notification / push updates | WebSocket or SSE integration. |
| Low | User preference persistence | Language, last folder, density, sort order. |
| Low | Feature flags / mock toggle | Environment-driven fallback disable. |
| Low | Translation coverage tests | Snapshot of DOM keys vs translation dictionaries. |

---

## How to Integrate a Real API (Updated Guidance)
You can **retain `mockFetch` in production** to allow graceful fallback until all endpoints are live.

### Migration Strategy
1. Implement core endpoints: `/api/accounts`, `/api/folders`, paginated `/api/emails`.
2. Replace mock translation dictionaries only after backend i18n stabilizes.
3. Gradually remove:
   - `generateMockEmails`, `generateMockAttachments`
   - Pagination mock builder IIFE
   - Static `mockApi` payload sections (retain schema notes)
4. Optionally replace `mockFetch` with pure `fetch` once fully migrated.

### Example Production Thin Layer
```js
// Transitional (keeps fallback)
const apiFetch = mockFetch;

// Strict (no mock fallback)
const apiFetchStrict = (url, opts) => fetch(url, opts);
```

### Endpoint Expectations (Summary)
| Endpoint | Type | Returns |
|----------|------|---------|
| /api/translations | GET | Language metadata array |
| /api/translations?l=[code] | GET | Language dictionary |
| /api/accounts | GET | Account array |
| /api/folders | GET | Array of per-account folder mappings |
| /api/emails?... | GET | Paginated email set |

---

## FAQ
**Q: Is this production-ready?**
A: It’s a solid prototype; add sanitization, error UI, auth, and remove dev mocks.

**Q: Can I add a new language?**
Yes—add an entry to `/api/translations` and define `/api/translations?l=<code>` in `mockApi` (or real backend).

**Q: Why do unread counts change after scrolling?**
They reflect only loaded pages; implement server unread aggregates.

**Q: Why are some translation keys unused?**
They are placeholders for future UI actions (compose / reply / forward / delete / subject labeling) and can be safely pruned if not needed.

**Q: How do I disable all mock data?**
Remove `mockApi`, generation functions, and replace `mockFetch` calls with `fetch`.

**Q: Does the app send or receive real email?**
No. All content is synthetic; no outbound network except attempted GET fetches.

**Q: Can I embed this inside an iframe?**
Yes—add CSP adjustments and possibly postMessage for parent integration.

---

## Contributing
1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Commit changes: `git commit -m "Add feature: your feature"`
4. Push branch: `git push origin feature/your-feature`
5. Open a Pull Request with a clear description

### Code Style Guidelines
- Use descriptive, intention-revealing function names.
- Provide JSDoc for all public-facing functions and utilities.
- Avoid silent `catch` blocks—log or rethrow.
- Keep DOM mutation localized (render functions should produce final fragments).
- When adding a **new function that performs an API call**, also:
  - Add corresponding mock data to `mockApi` for fallback parity.
  - Include a comment block documenting the expected response shape.
  - If paginated, mimic existing pagination URL pattern for consistency.
  - If introducing new translation-dependent UI text, add keys to every language dictionary to prevent fallback noise.

#### Example: Dynamic Date Formatting (future enhancement)
```js
const fmt = new Intl.DateTimeFormat(currentState.language, {
  dateStyle: 'short',
  timeStyle: 'short'
});
// Usage example:
dateElement.textContent = fmt.format(new Date(email.date));
```

---

## MIT License

MIT License
Copyright (c) 2025 antwal

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the “Software”), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall
be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF
ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED
TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A
PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM,
DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF
CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS
IN THE SOFTWARE.

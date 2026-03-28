# CLAUDE.md: Studio Native App

> **Context document for Claude Code sessions building the Studio native app.**
> **Created**: March 28, 2026
> **Read this first. The architecture spec in docs/ has full implementation details.**

---

## What This Is

Studio is a writing app for ADHD brains with research integration, a visible writing
pipeline (Idea through Published), semantic highlighting, wiki links, and an ML-powered
connection engine (Theseus). It currently exists as a web app at travisgilbert.me/studio
built with Next.js, React, and Tiptap.

This repo is the **native macOS and iOS version**. Architecture: SwiftUI shell for
native UI (sidebar, settings, Quick Capture, menus) with a WKWebView core running the
existing Tiptap editor and all 17 custom extensions. The editor is not rebuilt in Swift.
It runs as bundled HTML/JS/CSS inside the WebView.

All three clients (this native app, the web app, and future iOS app) share the same
Django API backend as the single source of truth. SwiftData provides local caching
and offline support. Sync is push-dirty-then-pull with last-write-wins conflict
resolution.

---

## Hard Rules (Apply to All Output)

1. **No em dashes.** Anywhere. Not in code, comments, documentation, or UI strings.
   Use colons, semicolons, commas, or periods instead.
2. **The editor lives in a WKWebView.** Do not attempt to rebuild Tiptap extensions
   in native Swift. The editor bundle (HTML/JS/CSS) runs inside a WebView. Swift
   communicates with it via WKScriptMessageHandler.
3. **Django API is the source of truth.** SwiftData is a local cache. Never treat
   local data as authoritative over the server.
4. **Bearer token auth.** Every API request includes `Authorization: Bearer {token}`.
   No cookies. Store the token in Keychain, never hardcode it.
5. **SwiftData for all local persistence.** No Core Data, no raw SQLite, no UserDefaults
   for content data. UserDefaults is acceptable for preferences only.
6. **Minimum deployment:** macOS 15 (Sequoia), iOS 18.
7. **No Y.js in the editor bundle.** The web version uses Y.js for IndexedDB persistence.
   Strip it from the standalone editor build. SwiftData handles local persistence in
   the native app.

---

## Repository Structure

```
Studio/
  CLAUDE.md                        # This file
  docs/
    2026-03-27-studio-native-app-architecture.md   # Full architecture spec
  StudioKit/                       # Shared Swift package (macOS + iOS)
    Package.swift
    Sources/StudioKit/
      API/
        StudioAPIClient.swift      # URLSession wrapper, Bearer auth
        IndexAPIClient.swift       # Theseus/research endpoints
        APITypes.swift             # Codable structs for API responses
      Models/
        ContentItem.swift          # SwiftData @Model
        ContentSheet.swift         # SwiftData @Model
        StashItem.swift            # SwiftData @Model
        ContentTask.swift          # SwiftData @Model
        Source.swift               # SwiftData @Model
        SyncState.swift            # SwiftData @Model (sync bookkeeping)
      Sync/
        SyncEngine.swift           # Background sync orchestrator
      Bridge/
        EditorBridge.swift         # WKScriptMessageHandler
        EditorCoordinator.swift    # Owns WKWebView, manages load/save
      Types/
        ContentType.swift          # Enum: essay, field-note, shelf, video, project, toolkit
        Stage.swift                # Enum: idea, research, drafting, revising, production, published
  StudioMac/                       # macOS app target
    StudioMacApp.swift
    Views/
      MainWindow.swift             # NavigationSplitView (sidebar + detail)
      SidebarView.swift            # Content list, sheets-as-paper cards
      EditorHostView.swift         # NSViewRepresentable wrapping WKWebView
      QuickCaptureWindow.swift     # Global shortcut floating panel
      SettingsView.swift
    Services/
      GlobalShortcutService.swift
  StudioiOS/                       # iOS app target (Phase 2)
    StudioiOSApp.swift
    Views/
      TabRootView.swift
      ContentListView.swift
      EditorHostView.swift         # UIViewRepresentable wrapping WKWebView
  EditorBundle/                    # Bundled Tiptap editor (HTML/JS/CSS)
    editor.html
    editor.js
    editor.css
    bridge.js
    fonts/
```

---

## Backend APIs

### Studio API (Django on Railway)

**Base URL:** `https://draftroom.travisgilbert.me/editor/api`
**Auth:** Bearer token in `Authorization` header
**Format:** JSON request/response, `Content-Type: application/json`

#### Content CRUD

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/content/` | List all content items (optional `?content_type=essay`) |
| GET | `/content/{type}/{slug}/` | Get single item with body |
| POST | `/content/{type}/` | Create new item |
| PATCH | `/content/{type}/{slug}/` | Update item (title, body, tags, excerpt) |
| DELETE | `/content/{type}/{slug}/` | Delete item |
| PATCH | `/content/{type}/{slug}/stage/` | Update pipeline stage |
| POST | `/content/{type}/{slug}/publish/` | Publish to public site |

#### Sheets

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/content/{type}/{slug}/sheets/` | List sheets for an item |
| POST | `/content/{type}/{slug}/sheets/` | Create sheet |
| PATCH | `/content/{type}/{slug}/sheets/{id}/` | Update sheet (title, body, order, status, wordCountTarget) |
| DELETE | `/content/{type}/{slug}/sheets/{id}/` | Delete sheet |
| POST | `/content/{type}/{slug}/sheets/reorder/` | Reorder sheets |
| POST | `/content/{type}/{slug}/sheets/{id}/split/` | Split sheet at position |
| POST | `/content/{type}/{slug}/sheets/{id}/merge/` | Merge with next sheet |

#### Stash, Tasks, Revisions

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/content/{type}/{slug}/stash/` | List stash items |
| POST | `/content/{type}/{slug}/stash/` | Create stash item |
| DELETE | `/content/{type}/{slug}/stash/{id}/` | Delete stash item |
| GET | `/content/{type}/{slug}/tasks/` | List tasks |
| POST | `/content/{type}/{slug}/tasks/` | Create task |
| PATCH | `/content/{type}/{slug}/tasks/{id}/` | Update task (text, done) |
| DELETE | `/content/{type}/{slug}/tasks/{id}/` | Delete task |
| GET | `/content/{type}/{slug}/revisions/` | List revision history |
| POST | `/content/{type}/{slug}/revisions/` | Create manual revision |
| POST | `/content/{type}/{slug}/revisions/{id}/restore/` | Restore a revision |

#### Sources and Research

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/sourcebox/submit-url/` | Submit URL for scraping |
| POST | `/sourcebox/upload/` | Upload file (multipart) |
| GET | `/sourcebox/status/{id}/` | Poll scrape status |
| GET | `/content/{type}/{slug}/sources/` | List sources for item |
| POST | `/content/{type}/{slug}/analyze/` | Run draft analysis (ML) |
| POST | `/content/{type}/{slug}/claims/audit/` | Claim audit (ML) |
| POST | `/content/{type}/{slug}/entities/` | Entity extraction (ML) |

#### Search and Dashboard

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/search/?q={query}` | Search across all content |
| GET | `/commonplace/search/?q={query}` | Search Studio content for wiki links |
| GET | `/timeline/` | Dashboard timeline entries |
| GET | `/dashboard/stats/` | Dashboard statistics |
| GET | `/content/{type}/{slug}/connections/` | Content connections graph |
| GET | `/content/{type}/{slug}/backlinks/` | Mention backlinks |
| GET | `/settings/` | Studio settings |

### Index API (Theseus on Railway)

**Base URL:** `https://index-api-production.up.railway.app`
**Auth:** None required for read endpoints

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/trail/{slug}/` | Research trail (sources, backlinks, threads) |
| GET | `/api/v1/objects/search/?q={query}&limit=10` | Search CommonPlace objects |

---

## Data Models (TypeScript to Swift Mapping)

### ContentItem

```
TypeScript (StudioContentItem)     Swift (SwiftData @Model)
id: string                         @Attribute(.unique) var id: String
title: string                      var title: String
slug: string                       var slug: String
contentType: string                var contentType: String
stage: string                      var stage: String
body: string                       var body: String
excerpt: string                    var excerpt: String
wordCount: number                  var wordCount: Int
tags: string[]                     var tags: [String]
createdAt: string                  var createdAt: Date
updatedAt: string                  var updatedAt: Date
publishedAt: string | null         var publishedAt: Date?
nextMove?: string                  var nextMove: String?
lastSessionSummary?: string        var lastSessionSummary: String?
                                   var isDirty: Bool = false     (sync)
                                   var lastSyncedAt: Date?       (sync)
```

### Sheet (ContentSheet)

```
TypeScript (Sheet)                 Swift (SwiftData @Model)
id: string                        @Attribute(.unique) var id: String
contentType: string                var contentType: String
contentSlug: string                var contentSlug: String
order: number                      var order: Int
title: string                      var title: String
body: string                       var body: String
isMaterial: boolean                var isMaterial: Bool
status: string | null              var status: String?
wordCount: number                  var wordCount: Int
wordCountTarget: number | null     var wordCountTarget: Int?
createdAt: string                  var createdAt: Date
updatedAt: string                  var updatedAt: Date
                                   var isDirty: Bool = false     (sync)
```

### Content Types

```swift
enum ContentType: String, CaseIterable, Codable {
    case essay = "essay"
    case fieldNote = "field-note"
    case shelf = "shelf"
    case video = "video"
    case project = "project"
    case toolkit = "toolkit"

    var label: String {
        switch self {
        case .essay: return "Essay"
        case .fieldNote: return "Field Note"
        case .shelf: return "Shelf Entry"
        case .video: return "Video"
        case .project: return "Project"
        case .toolkit: return "Toolkit"
        }
    }

    var color: String {
        switch self {
        case .essay: return "#B45A2D"
        case .fieldNote: return "#3A8A9A"
        case .shelf: return "#D4AA4A"
        case .video: return "#6A9A5A"
        case .project: return "#D4AA4A"
        case .toolkit: return "#B45A2D"
        }
    }
}
```

---

## Editor Bridge Protocol

Communication between Swift and the Tiptap editor running inside WKWebView.

### Swift to JavaScript

Call via `webView.evaluateJavaScript("window.studioBridge.methodName(args)")`.

| Method | Arguments | Purpose |
|--------|-----------|---------|
| `loadContent(markdown, format)` | String, String | Set editor content |
| `getContent()` | none | Request content (JS responds via message handler) |
| `setStage(stage)` | String | Update stage indicator color |
| `focus()` | none | Focus the editor for typing |
| `setReadingSettings(json)` | JSON String | Font, size, line height |

### JavaScript to Swift

Via `window.webkit.messageHandlers.studio.postMessage({...})`.

| Message type | Payload | Purpose |
|--------------|---------|---------|
| `contentChanged` | `{ markdown: String, wordCount: Int }` | Content was edited |
| `saveRequested` | `{}` | User pressed Cmd+S |
| `wikiLinkClicked` | `{ title: String }` | Navigate to linked content |
| `mentionClicked` | `{ id: String, contentType: String }` | Navigate to mentioned content |
| `wikiSearchRequested` | `{ query: String }` | Wiki link popup needs search results |

### Wiki Link Search Flow (Bridge)

1. User types `[[` in the editor, starts typing a query
2. JS sends `wikiSearchRequested` with the query string
3. Swift calls `StudioAPIClient.searchContent(query)` and `IndexAPIClient.searchObjects(query)`
4. Swift merges results, deduplicates by title
5. Swift pushes results back: `webView.evaluateJavaScript("window.studioBridge.receiveSearchResults(\(json))")`
6. JS populates the wiki link popup with the results

---

## Design Tokens (Native UI)

The native shell (sidebar, toolbar, settings) uses these colors. The editor WebView
has its own CSS (the existing studio.css, bundled).

### Dark Mode (Default, Sidebar Chrome)

```
Background:     #2B3544 (slate chrome, matching web's light-mode sidebar)
Surface:        #344050
Text primary:   #F0EAE0
Text secondary: #C4BCB0
Text muted:     #8A9AAA
Terracotta:     #E07A3A (lifted for dark bg contrast)
Teal:           #5EC4D6
Gold:           #E8C060
Green:          #7EC06A
Border:         rgba(240, 234, 224, 0.10)
```

### Light Mode (Content Areas, if needed)

```
Background:     #DDD4C6
Surface:        #E8DFD4
Text primary:   #2A2420
Text secondary: #5A5248
Terracotta:     #B84E1E
Teal:           #2D5F6B
Gold:           #C49A4A
```

### Sheets-as-Paper (Sidebar)

```
Active sheet bg:    #FAF6F0
Inactive sheet bg:  #F0EAE0
Active accent bar:  #B45A2D (terracotta, 3px top bar)
Active shadow:      0 2px 8px rgba(0,0,0,0.25), 0 0 0 2px rgba(224,112,58,0.4)
Inactive shadow:    0 1px 4px rgba(0,0,0,0.15)
Title font:         Georgia, 13px
Word count font:    JetBrains Mono, 9px
Ruled line color:   rgba(42, 36, 32, 0.06)
```

### Typography

```
Titles:     .custom("Vollkorn", size: N)
Body:       .custom("IBM Plex Sans", size: N) or .body
Mono:       .custom("JetBrains Mono", size: N)
Sheet card: .custom("Georgia", size: 13)
```

Bundle all fonts in the app (OFL licensed): Vollkorn, IBM Plex Sans, JetBrains Mono,
Amarna (verify license), Caudex. Font files go in the app bundle, registered in Info.plist.

---

## Sync Model

### Core Loop

```
On app launch:
  1. pushDirty()    // Send local changes to server first
  2. pullAll()      // Fetch latest from server, update local cache

Every 30 seconds:
  Same cycle: push then pull

On content save (user presses Cmd+S or autosave fires):
  1. Save to SwiftData (instant, works offline)
  2. Mark item as isDirty = true
  3. Next sync cycle pushes it
```

### Conflict Resolution

Last-write-wins. This is a single-user app.

When pulling from server:
- If local item is NOT dirty: overwrite local with server data
- If local item IS dirty: compare timestamps
  - Server timestamp newer: server wins, overwrite local, clear dirty flag
  - Local timestamp newer: keep local, push on next cycle

### Offline Behavior

- All content is cached in SwiftData after first sync
- Creating/editing offline: save to SwiftData with isDirty = true
- When network returns: next sync cycle pushes all dirty items
- Editor bundle loads from app bundle (no network needed)
- Research features (Theseus trail, object search) require network

---

## Web Repo Reference

The web codebase at `Travis-Gilbert/travisgilbert.me` is NOT part of this repo.
Reference it via GitHub MCP when you need to understand:

- API response shapes: `src/lib/studio-api.ts` (50+ endpoints, all TypeScript interfaces)
- Content type definitions: `src/lib/studio.ts` (CONTENT_TYPES array, stage definitions)
- Editor extensions: `src/components/studio/extensions/` (17 files)
- Editor component: `src/components/studio/TiptapEditor.tsx` (all extension configuration)
- CSS tokens: `src/styles/studio.css` (design tokens, writing surface styles)
- Stage stepper: `src/components/studio/StageStepper.tsx` (pipeline UI)
- Sheets: `src/components/studio/SheetList.tsx` (sheet management UI)

The editor bundle extraction (creating the standalone HTML/JS/CSS from the web repo)
is documented in the architecture spec under "Editor Bundle Build Process."

---

## Build Order

Phase 1: macOS app (4 months)
  Month 1: SwiftData models, API client, basic window, editor bundle extraction
  Month 2: WKWebView integration, save flow, sheets in sidebar, stage stepper
  Month 3: Sync engine, offline support, research panel, stash/tasks/revisions
  Month 4: Quick Capture, native menus, notifications, settings, distribution

Phase 2: iOS app (3 months)
  Month 5: iOS shell (tabs, navigation, editor WebView)
  Month 6: Share Extension, WidgetKit, touch optimization
  Month 7: Polish, TestFlight, App Store submission

Detailed task list: `STUDIO-NATIVE-TODO.md`

---

## Style

- No em dashes (use colons, semicolons, commas, periods)
- Swift naming conventions: camelCase properties, PascalCase types
- SwiftUI: prefer `@Observable` over `ObservableObject` when possible on macOS 15+
- Use `async/await` for all API calls, never completion handlers
- Actors for thread-safe API clients and sync engine
- Prefer Swift's native types (Date, URL) over string representations
- JSON decoding: use `keyDecodingStrategy: .convertFromSnakeCase` since Django returns snake_case
- Error handling: throw typed errors (StudioAPIError enum), catch at the UI layer

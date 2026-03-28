# Studio Native App: Task List

Sequenced steps to build the macOS + iOS app.
Architecture doc: `docs/2026-03-27-studio-native-app-architecture.md`

---

## Phase 0: Prerequisites (Before Writing Swift)

- [ ] **Learn Swift basics** (1 to 2 weeks)
  - Stanford CS193p on YouTube (SwiftUI focused, free)
  - Apple "Develop in Swift" tutorials
  - Build a toy app: list view that fetches from your Django API and displays content titles
  - That toy app teaches: URLSession, async/await, SwiftData, NavigationSplitView

- [ ] **Finish Phase 1 web polish** (the spec already run against travisgilbert.me)
  - Verify: 80ch writing surface, toolbar auto-collapse, sheets-as-paper, multi-color highlights, source card redesign
  - These changes carry into the editor bundle. Fix them in web first.

- [ ] **Check font licenses for app embedding**
  - JetBrains Mono: OFL (clear)
  - Vollkorn: OFL (clear)
  - IBM Plex Sans: OFL (clear)
  - Caudex: OFL (clear)
  - Amarna: check the license file in your fonts directory

- [ ] **Set up Xcode project in this repo**
  - Create multiplatform SwiftUI app
  - macOS 15, iOS 18 deployment targets
  - Create StudioKit as a local Swift package
  - Add .gitignore for Xcode artifacts

---

## Phase 1: macOS App, Month 1 (Foundation)

### Week 1: Project Structure

- [ ] Create Xcode multiplatform SwiftUI project named "Studio"
- [ ] Create `StudioKit` Swift package (shared macOS + iOS code)
- [ ] Add StudioKit as dependency of both targets
- [ ] Set up folder structure per CLAUDE.md
- [ ] First commit with project skeleton

### Week 2: SwiftData Models + API Client

- [ ] Create `ContentType.swift` enum (essay, field-note, shelf, video, project, toolkit)
- [ ] Create `Stage.swift` enum (idea, research, drafting, revising, production, published)
- [ ] Create `ContentItem.swift` SwiftData @Model
- [ ] Create `ContentSheet.swift` SwiftData @Model
- [ ] Create `StashItem.swift` SwiftData @Model
- [ ] Create `ContentTask.swift` SwiftData @Model
- [ ] Create `SyncState.swift` SwiftData @Model
- [ ] Create `StudioAPIClient.swift` (actor, URLSession, Bearer auth)
- [ ] Implement content CRUD endpoints (list, get, create, save, delete)
- [ ] Test: fetch content list from Django, print to console
- [ ] Commit

### Week 3: Basic macOS Window

- [ ] Create `MainWindow.swift` with NavigationSplitView
- [ ] Create `SidebarView.swift` (content types as sections, items listed)
- [ ] Create placeholder `DetailView.swift` (show selected item title + body)
- [ ] Wire SwiftData model container in @main
- [ ] On launch: fetch from API, populate SwiftData, display in sidebar
- [ ] Commit

### Week 4: Editor Bundle Extraction

- [ ] In `travisgilbert.me` repo: create `TiptapEditor.standalone.tsx`
  - Strip Next.js imports, remove Y.js, export mount() function
  - Replace direct API calls with bridge stubs
- [ ] Create `src/native/bridge.js` (window.studioBridge)
- [ ] Create `scripts/build-editor-bundle.sh` (esbuild)
- [ ] Run build, verify editor.html works standalone in a browser
- [ ] Copy built files into `Studio/EditorBundle/`
- [ ] Copy font files (woff2) into `Studio/EditorBundle/fonts/`
- [ ] Commit both repos

---

## Phase 1: macOS App, Month 2 (Editor Integration)

### Week 5: WKWebView in SwiftUI

- [ ] Create `EditorBridge.swift` (WKScriptMessageHandler)
- [ ] Create `EditorCoordinator.swift` (owns WKWebView, sends/receives)
- [ ] Create `EditorHostView.swift` (NSViewRepresentable)
- [ ] Wire: selecting sidebar item loads content in editor WebView
- [ ] Test: select essay, see Tiptap render inside native window
- [ ] Commit

### Week 6: Save Flow

- [ ] contentChanged from bridge marks item dirty in SwiftData
- [ ] Autosave: 3s debounce, save to SwiftData
- [ ] Manual save: Cmd+S from bridge saves immediately
- [ ] Save indicator in native toolbar
- [ ] Commit

### Week 7: Sheets in Native Sidebar

- [ ] Add sheet endpoints to StudioAPIClient
- [ ] Show sheets below item in sidebar when present
- [ ] Implement `SheetPaperCard.swift` (parchment cards)
- [ ] Sheet switching: save current, load target via bridge
- [ ] Drag to reorder
- [ ] Commit

### Week 8: Stage Stepper + Toolbar

- [ ] Native stage indicator (colored dot + name)
- [ ] Stage advance button, calls updateStage API
- [ ] Word count in toolbar from bridge updates
- [ ] Commit

---

## Phase 1: macOS App, Month 3 (Sync + Offline)

### Week 9: Sync Engine

- [ ] Create `SyncEngine.swift` (actor)
- [ ] pushDirty(): find dirty items, PATCH to API
- [ ] pullAll(): GET all, update SwiftData (skip if local dirty + newer)
- [ ] sync(): push then pull
- [ ] Sync on launch, periodic every 30s
- [ ] Commit

### Week 10: Offline Support

- [ ] App works without network (SwiftData has content)
- [ ] Create/edit offline: isDirty = true, sync when online
- [ ] "Offline" indicator when no connectivity
- [ ] Commit

### Week 11: Research Panel

- [ ] Create `IndexAPIClient.swift` (Theseus endpoints)
- [ ] Native Research panel as right inspector
  - Sources with Vollkorn titles
  - URL submission
  - Connected content
- [ ] Wire wiki link search through bridge (JS asks Swift, Swift calls API, pushes back)
- [ ] Commit

### Week 12: Stash, Tasks, Revisions

- [ ] Native stash panel
- [ ] Task list with checkboxes
- [ ] Revision history panel
- [ ] Commit

---

## Phase 1: macOS App, Month 4 (Polish)

### Week 13: Quick Capture

- [ ] `QuickCaptureWindow.swift` (floating panel)
- [ ] Global shortcut: Cmd+Shift+Space via NSEvent monitor
- [ ] Saves to SwiftData, syncs in background
- [ ] Commit

### Week 14: Native Menus + Notifications

- [ ] `StudioCommands.swift` (menu bar: New, Save, Advance, Zen mode)
- [ ] Notifications for new Theseus connections (poll, compare, notify)
- [ ] Commit

### Week 15: Settings + Polish

- [ ] Native Settings view (API token in Keychain, sync interval, highlights, fonts)
- [ ] App icon
- [ ] Light/dark mode verification
- [ ] Commit

### Week 16: Distribution

- [ ] Code signing (Developer ID)
- [ ] Notarization
- [ ] DMG or direct download from travisgilbert.me
- [ ] Ship v1.0 macOS

---

## Phase 2: iOS App (Months 5 through 7)

### Month 5: iOS Shell

- [ ] Add iOS target, import StudioKit
- [ ] `TabRootView.swift` (Writing, Capture, Settings tabs)
- [ ] `ContentListView.swift` (navigation stack)
- [ ] iOS `EditorHostView.swift` (UIViewRepresentable + WKWebView)
- [ ] Sheets as bottom sheet
- [ ] Test: open essay, edit, save

### Month 6: iOS Features

- [ ] Share Extension (capture URLs from Safari)
- [ ] WidgetKit Quick Capture widget
- [ ] Touch target audit (44pt minimum)
- [ ] iOS toolbar adaptation

### Month 7: iOS Polish + Ship

- [ ] Keyboard resize handling
- [ ] Pull-to-refresh, swipe-to-delete
- [ ] iPad NavigationSplitView layout
- [ ] TestFlight beta
- [ ] App Store submission (free)

---

## Post-Launch

- [ ] Automate editor bundle builds (GitHub Action)
- [ ] Spotlight indexing (CSSearchableItem)
- [ ] Shortcuts app integration
- [ ] Handoff (start on Mac, continue on iPhone)

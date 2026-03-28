# Studio Native: SwiftUI Shell + WKWebView Editor

## 2026-03-27

**Target platforms:** macOS 15+ (Sequoia), iOS 18+
**Architecture:** SwiftUI native shell, WKWebView editor core, SwiftData local cache
**Backend:** Existing Django API on Railway (Bearer token auth), Index API (Theseus)
**Sync model:** Django API is source of truth. SwiftData caches locally. Background sync.
**Web version:** Continues at travisgilbert.me/studio, unchanged. All three clients share the Django API.

**No em dashes anywhere in this document or generated code.**

---

## Why This Architecture

Studio's editor has 17 custom Tiptap extensions (WikiLink, @mentions, slash commands,
contain blocks with 7 semantic types, drag handles, inline comments, focus fade,
typewriter mode, color highlighter, resizable images, iframe embeds, code syntax
highlighting, Y.js collaboration, custom input rules, tab indentation, paper
weathering). Rebuilding this in TextKit 2 would take 12+ months.

WKWebView loads the existing Tiptap editor as a bundled HTML/JS package. All 17
extensions work on day one. The native shell (SwiftUI) handles everything else:
window management, sidebar, navigation, settings, Quick Capture, notifications,
file system access, offline caching.

Bear, Obsidian, and Notion all use WebViews for their editors. This is the
industry-standard pattern for rich text editing in native apps.

**Competitive advantages over Ulysses/Bear/iA Writer:**
1. Web version syncs seamlessly (write on Mac, continue on web, continue on iPhone)
2. Research integration (sources, backlinks, Theseus connections)
3. Visible writing pipeline (Idea through Published)
4. ADHD-conscious design (sheets-as-paper, toolbar auto-collapse, typewriter mode)
5. Semantic highlighting with named colors

---

## Repository Structure

New repo: `Travis-Gilbert/Studio` (or `Travis-Gilbert/studio-app`)

```
Studio/
  Package.swift                    # Root SPM workspace
  StudioKit/                       # Shared Swift package (macOS + iOS)
    Sources/StudioKit/
      API/
        StudioAPIClient.swift      # URLSession wrapper, Bearer auth
        IndexAPIClient.swift       # Theseus/research endpoints
        Endpoints.swift            # All endpoint definitions
      Models/
        ContentItem.swift          # SwiftData @Model
        Sheet.swift                # SwiftData @Model
        StashItem.swift            # SwiftData @Model
        ContentTask.swift          # SwiftData @Model
        Source.swift               # SwiftData @Model
        Revision.swift             # SwiftData @Model (metadata only)
        SyncState.swift            # SwiftData @Model (sync bookkeeping)
      Sync/
        SyncEngine.swift           # Background sync orchestrator
        SyncOperation.swift        # Individual sync operations
        ConflictResolver.swift     # Last-write-wins logic
      Bridge/
        EditorBridge.swift         # WKWebView JS bridge protocol
        EditorMessage.swift        # Message types (Swift to JS, JS to Swift)
      Types/
        ContentType.swift          # Enum mirroring CONTENT_TYPES
        Stage.swift                # Enum for pipeline stages
        HighlightColor.swift       # Stored highlight configuration
  StudioMac/                       # macOS app target
    StudioMacApp.swift             # @main, SwiftData container
    Views/
      MainWindow.swift             # NSSplitViewController equivalent
      SidebarView.swift            # Content list, sheets-as-paper
      EditorHostView.swift         # WKWebView wrapper
      SettingsView.swift           # Native settings
      QuickCaptureWindow.swift     # Global shortcut panel
    Services/
      GlobalShortcutService.swift  # Cmd+Shift+Space capture
      NotificationService.swift    # Engine pass notifications
      MenuBarService.swift         # Native menu integration
  StudioiOS/                       # iOS app target
    StudioiOSApp.swift             # @main, SwiftData container
    Views/
      TabRootView.swift            # Bottom tab navigation
      ContentListView.swift        # Content browse
      EditorHostView.swift         # WKWebView wrapper (iOS layout)
      QuickCaptureSheet.swift      # Bottom sheet capture
      SettingsView.swift           # Native settings
    Extensions/
      ShareExtension/              # Share sheet for URLs from Safari
      WidgetExtension/             # Home screen quick capture widget
  EditorBundle/                    # Tiptap editor, bundled as static HTML
    editor.html                    # Single-file editor entry point
    editor.js                      # Bundled Tiptap + all 17 extensions
    editor.css                     # Studio writing surface styles
    bridge.js                      # JS side of the native bridge
```

---

## Phase 1: macOS App (Months 1 through 4)

### Month 1: Foundation

#### 1A. SwiftData Models

Mirror the Django content models. These are the local cache, not the source of truth.

```swift
import SwiftData
import Foundation

@Model
final class ContentItem {
    @Attribute(.unique) var id: String
    var title: String
    var slug: String
    var contentType: String     // "essay", "field-note", "shelf", etc.
    var stage: String           // "idea", "research", "drafting", etc.
    var body: String            // Markdown content
    var excerpt: String
    var wordCount: Int
    var tags: [String]
    var createdAt: Date
    var updatedAt: Date
    var publishedAt: Date?
    var nextMove: String?
    var lastSessionSummary: String?

    // Sync bookkeeping
    var lastSyncedAt: Date?
    var isDirty: Bool = false   // Local changes not yet synced
    var serverUpdatedAt: Date?  // Last known server timestamp

    @Relationship(deleteRule: .cascade)
    var sheets: [ContentSheet] = []

    @Relationship(deleteRule: .cascade)
    var stashItems: [StashItem] = []

    @Relationship(deleteRule: .cascade)
    var tasks: [ContentTask] = []
}

@Model
final class ContentSheet {
    @Attribute(.unique) var id: String
    var contentType: String
    var contentSlug: String
    var order: Int
    var title: String
    var body: String
    var isMaterial: Bool
    var status: String?         // "idea", "drafting", "locked"
    var wordCount: Int
    var wordCountTarget: Int?
    var createdAt: Date
    var updatedAt: Date
    var isDirty: Bool = false

    var contentItem: ContentItem?
}

@Model
final class StashItem {
    @Attribute(.unique) var id: String
    var text: String
    var category: String?
    var createdAt: Date
    var isDirty: Bool = false

    var contentItem: ContentItem?
}

@Model
final class ContentTask {
    @Attribute(.unique) var id: Int
    var text: String
    var done: Bool
    var createdAt: Date
    var isDirty: Bool = false

    var contentItem: ContentItem?
}

@Model
final class SyncState {
    @Attribute(.unique) var key: String   // "lastFullSync", "lastContentSync", etc.
    var value: String                      // ISO date or JSON payload
    var updatedAt: Date
}
```

#### 1B. API Client

Typed URLSession wrapper matching your existing `studioFetch` pattern.

```swift
import Foundation

actor StudioAPIClient {
    static let shared = StudioAPIClient()

    private let baseURL: URL
    private let token: String
    private let session: URLSession

    init(
        baseURL: URL = URL(string: "https://draftroom.travisgilbert.me/editor/api")!,
        token: String = Configuration.studioAPIToken
    ) {
        self.baseURL = baseURL
        self.token = token

        let config = URLSessionConfiguration.default
        config.timeoutIntervalForRequest = 15
        config.waitsForConnectivity = true
        self.session = URLSession(configuration: config)
    }

    func fetch<T: Decodable>(_ path: String, method: String = "GET", body: Encodable? = nil) async throws -> T {
        var url = baseURL.appendingPathComponent(path)

        var request = URLRequest(url: url)
        request.httpMethod = method
        request.setValue("application/json", forHTTPHeaderField: "Accept")
        request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")

        if let body {
            request.setValue("application/json", forHTTPHeaderField: "Content-Type")
            request.httpBody = try JSONEncoder.studio.encode(body)
        }

        let (data, response) = try await session.data(for: request)

        guard let http = response as? HTTPURLResponse else {
            throw StudioAPIError.invalidResponse
        }

        guard (200...299).contains(http.statusCode) else {
            throw StudioAPIError.httpError(status: http.statusCode, body: String(data: data, encoding: .utf8))
        }

        return try JSONDecoder.studio.decode(T.self, from: data)
    }

    // Content endpoints (mirrors studio-api.ts)
    func fetchContentList(contentType: String? = nil) async throws -> [APIContentItem] {
        var path = "/content/"
        if let ct = contentType { path += "?content_type=\(ct)" }
        return try await fetch(path)
    }

    func fetchContentItem(contentType: String, slug: String) async throws -> APIContentItem {
        return try await fetch("/content/\(contentType)/\(slug)/")
    }

    func saveContentItem(contentType: String, slug: String, body: SaveContentBody) async throws -> APIContentItem {
        return try await fetch("/content/\(contentType)/\(slug)/", method: "PATCH", body: body)
    }

    func createContentItem(contentType: String, body: CreateContentBody?) async throws -> APIContentItem {
        return try await fetch("/content/\(contentType)/", method: "POST", body: body)
    }

    func deleteContentItem(contentType: String, slug: String) async throws {
        let _: EmptyResponse = try await fetch("/content/\(contentType)/\(slug)/", method: "DELETE")
    }

    func updateStage(contentType: String, slug: String, stage: String) async throws -> APIContentItem {
        return try await fetch("/content/\(contentType)/\(slug)/stage/", method: "PATCH", body: ["stage": stage])
    }

    // Sheets
    func fetchSheets(contentType: String, slug: String) async throws -> [APISheet] {
        return try await fetch("/content/\(contentType)/\(slug)/sheets/")
    }

    func createSheet(contentType: String, slug: String, body: CreateSheetBody) async throws -> APISheet {
        return try await fetch("/content/\(contentType)/\(slug)/sheets/", method: "POST", body: body)
    }

    func updateSheet(contentType: String, slug: String, sheetId: String, body: UpdateSheetBody) async throws -> APISheet {
        return try await fetch("/content/\(contentType)/\(slug)/sheets/\(sheetId)/", method: "PATCH", body: body)
    }

    func deleteSheet(contentType: String, slug: String, sheetId: String) async throws {
        let _: EmptyResponse = try await fetch("/content/\(contentType)/\(slug)/sheets/\(sheetId)/", method: "DELETE")
    }

    // Stash
    func fetchStash(contentType: String, slug: String) async throws -> [APIStashItem] {
        return try await fetch("/content/\(contentType)/\(slug)/stash/")
    }

    // Tasks
    func fetchTasks(contentType: String, slug: String) async throws -> [APITask] {
        return try await fetch("/content/\(contentType)/\(slug)/tasks/")
    }

    // Search
    func searchContent(query: String) async throws -> [APISearchResult] {
        return try await fetch("/search/?q=\(query.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? "")")
    }

    // Revisions
    func fetchRevisions(contentType: String, slug: String) async throws -> [APIRevision] {
        return try await fetch("/content/\(contentType)/\(slug)/revisions/")
    }

    // Sourcebox
    func submitSourceURL(_ url: String) async throws -> APISource {
        return try await fetch("/sourcebox/submit-url/", method: "POST", body: ["url": url])
    }

    func fetchSources(contentType: String, slug: String) async throws -> [APISource] {
        return try await fetch("/content/\(contentType)/\(slug)/sources/")
    }

    // Draft analysis (ML)
    func analyzeDraft(contentType: String, slug: String) async throws -> APIDraftAnalysis {
        return try await fetch("/content/\(contentType)/\(slug)/analyze/", method: "POST")
    }
}
```

The Index API client follows the same pattern but targets the Theseus service:

```swift
actor IndexAPIClient {
    static let shared = IndexAPIClient()

    private let baseURL: URL

    init(baseURL: URL = URL(string: "https://index-api-production.up.railway.app")!) {
        self.baseURL = baseURL
    }

    func fetchTrail(slug: String) async throws -> APIResearchTrail? {
        // GET /api/v1/trail/{slug}/ with 5s timeout
    }

    func searchObjects(query: String, limit: Int = 10) async throws -> [APIObjectSearchResult] {
        // GET /api/v1/objects/search/?q={query}&limit={limit}
    }
}
```

#### 1C. WKWebView Editor Bridge

The bridge handles two-way communication between Swift and JavaScript.

**Swift to JavaScript (commands):**
- `loadContent(markdown: String, format: String)` : Set editor content
- `getContent()` : Request current content (JS responds via message handler)
- `setStage(stage: String)` : Update stage indicator
- `setReadingSettings(json: String)` : Font, size, line height
- `focus()` : Focus the editor
- `toggleTypewriterMode(enabled: Bool)` : Toggle typewriter scrolling

**JavaScript to Swift (messages via WKScriptMessageHandler):**
- `contentChanged { markdown: String, wordCount: Int }` : Content update
- `saveRequested {}` : User pressed Cmd+S
- `linkClicked { url: String }` : External link
- `wikiLinkClicked { title: String }` : Navigate to linked content
- `mentionClicked { id: String, contentType: String }` : Navigate to mentioned content

```swift
import WebKit

final class EditorBridge: NSObject, WKScriptMessageHandler {
    weak var coordinator: EditorCoordinator?

    enum OutgoingCommand {
        case loadContent(markdown: String, format: String)
        case getContent
        case setStage(String)
        case focus

        var javascript: String {
            switch self {
            case .loadContent(let md, let fmt):
                let escaped = md
                    .replacingOccurrences(of: "\\", with: "\\\\")
                    .replacingOccurrences(of: "`", with: "\\`")
                    .replacingOccurrences(of: "$", with: "\\$")
                return "window.studioBridge.loadContent(`\(escaped)`, '\(fmt)')"
            case .getContent:
                return "window.studioBridge.getContent()"
            case .setStage(let stage):
                return "window.studioBridge.setStage('\(stage)')"
            case .focus:
                return "window.studioBridge.focus()"
            }
        }
    }

    func userContentController(
        _ controller: WKUserContentController,
        didReceive message: WKScriptMessage
    ) {
        guard let body = message.body as? [String: Any],
              let type = body["type"] as? String else { return }

        switch type {
        case "contentChanged":
            let markdown = body["markdown"] as? String ?? ""
            let wordCount = body["wordCount"] as? Int ?? 0
            coordinator?.handleContentChanged(markdown: markdown, wordCount: wordCount)

        case "saveRequested":
            coordinator?.handleSaveRequested()

        case "wikiLinkClicked":
            let title = body["title"] as? String ?? ""
            coordinator?.handleWikiLinkNavigation(title: title)

        case "mentionClicked":
            let id = body["id"] as? String ?? ""
            let contentType = body["contentType"] as? String ?? ""
            coordinator?.handleMentionNavigation(id: id, contentType: contentType)

        default:
            break
        }
    }
}
```

**JavaScript bridge file (`EditorBundle/bridge.js`):**

```javascript
window.studioBridge = {
    loadContent(markdown, format) {
        const editor = window.__studioEditor;
        if (!editor) return;
        editor.commands.setContent(markdown);
    },

    getContent() {
        const editor = window.__studioEditor;
        if (!editor) return;
        const markdown = typeof editor.getMarkdown === 'function'
            ? editor.getMarkdown()
            : editor.getText();
        const wordCount = editor.storage.characterCount?.words?.() ?? 0;
        window.webkit.messageHandlers.studio.postMessage({
            type: 'contentChanged',
            markdown,
            wordCount,
        });
    },

    setStage(stage) {
        const page = document.querySelector('.studio-page');
        if (page) page.dataset.stage = stage;
    },

    focus() {
        const editor = window.__studioEditor;
        if (editor) editor.commands.focus();
    },
};

// Hook into Tiptap's onUpdate to push changes to Swift
document.addEventListener('studio:editor-ready', () => {
    const editor = window.__studioEditor;
    if (!editor) return;

    editor.on('update', () => {
        const markdown = typeof editor.getMarkdown === 'function'
            ? editor.getMarkdown()
            : editor.getText();
        const wordCount = editor.storage.characterCount?.words?.() ?? 0;
        window.webkit.messageHandlers.studio.postMessage({
            type: 'contentChanged',
            markdown,
            wordCount,
        });
    });
});

// Intercept Cmd+S
document.addEventListener('keydown', (e) => {
    if ((e.metaKey || e.ctrlKey) && e.key === 's') {
        e.preventDefault();
        window.webkit.messageHandlers.studio.postMessage({ type: 'saveRequested' });
    }
});
```

### Month 2: macOS Window and Navigation

#### 2A. Main Window Layout

macOS sidebar + editor pattern using NavigationSplitView.

```swift
import SwiftUI
import SwiftData

struct MainWindow: View {
    @State private var selectedContentType: String? = nil
    @State private var selectedItem: ContentItem? = nil
    @State private var columnVisibility: NavigationSplitViewVisibility = .all

    var body: some View {
        NavigationSplitView(columnVisibility: $columnVisibility) {
            SidebarView(
                selectedContentType: $selectedContentType,
                selectedItem: $selectedItem
            )
            .navigationSplitViewColumnWidth(min: 200, ideal: 240, max: 300)
        } detail: {
            if let item = selectedItem {
                EditorHostView(contentItem: item)
            } else {
                DashboardView()
            }
        }
        .navigationTitle("")
        .toolbar {
            ToolbarItemGroup(placement: .primaryAction) {
                StageIndicator(item: selectedItem)
                Spacer()
                SaveIndicator(item: selectedItem)
            }
        }
    }
}
```

#### 2B. Sidebar with Sheets-as-Paper

The sidebar uses your existing sheets-as-paper design, rendered natively in SwiftUI.

```swift
struct SheetPaperCard: View {
    let sheet: ContentSheet
    let isActive: Bool
    let onSelect: () -> Void

    var body: some View {
        VStack(alignment: .leading, spacing: 6) {
            // Title
            Text(sheet.title.isEmpty ? "Untitled" : sheet.title)
                .font(.custom("Georgia", size: 13))
                .fontWeight(isActive ? .semibold : .medium)
                .foregroundStyle(isActive ? Color(hex: "#1A1614") : Color(hex: "#5A5248"))
                .lineLimit(1)

            // Progress bar + word count
            if let target = sheet.wordCountTarget {
                HStack(spacing: 6) {
                    ProgressView(value: Double(sheet.wordCount), total: Double(target))
                        .tint(sheet.wordCount >= target
                            ? Color(hex: "#5A7A4A")
                            : Color(hex: "#B45A2D"))
                    Text("\(sheet.wordCount)w / \(target)")
                        .font(.custom("JetBrains Mono", size: 9))
                        .foregroundStyle(Color(hex: "#9A9088"))
                }
            } else {
                Text("\(sheet.wordCount)w")
                    .font(.custom("JetBrains Mono", size: 9))
                    .foregroundStyle(Color(hex: "#9A9088"))
            }

            // Faint ruled lines
            VStack(spacing: 4) {
                Rectangle().frame(height: 0.5).opacity(0.08)
                Rectangle().frame(width: .infinity * 0.85, height: 0.5).opacity(0.06)
                Rectangle().frame(width: .infinity * 0.6, height: 0.5).opacity(0.06)
            }
            .foregroundStyle(Color(hex: "#2A2420"))
        }
        .padding(14)
        .background(isActive ? Color(hex: "#FAF6F0") : Color(hex: "#F0EAE0"))
        .clipShape(RoundedRectangle(cornerRadius: 3))
        .overlay(alignment: .top) {
            if isActive {
                Rectangle()
                    .fill(Color(hex: "#B45A2D"))
                    .frame(height: 3)
                    .clipShape(UnevenRoundedRectangle(topLeadingRadius: 3, topTrailingRadius: 3))
            }
        }
        .shadow(color: .black.opacity(isActive ? 0.25 : 0.15),
                radius: isActive ? 4 : 2, y: isActive ? 2 : 1)
        .onTapGesture(perform: onSelect)
        .draggable(sheet.id)
    }
}
```

#### 2C. Editor Host View

The WKWebView wrapper that loads the bundled editor.

```swift
import SwiftUI
import WebKit

struct EditorHostView: View {
    let contentItem: ContentItem
    @State private var coordinator = EditorCoordinator()
    @Environment(\.modelContext) private var modelContext

    var body: some View {
        EditorWebView(coordinator: coordinator)
            .onAppear {
                coordinator.modelContext = modelContext
                coordinator.loadItem(contentItem)
            }
            .onChange(of: contentItem) { _, newItem in
                coordinator.loadItem(newItem)
            }
    }
}

struct EditorWebView: NSViewRepresentable {
    let coordinator: EditorCoordinator

    func makeNSView(context: Context) -> WKWebView {
        let config = WKWebViewConfiguration()
        let userContent = config.userContentController
        userContent.add(coordinator.bridge, name: "studio")

        let webView = WKWebView(frame: .zero, configuration: config)
        webView.navigationDelegate = coordinator
        coordinator.webView = webView

        // Load bundled editor
        if let editorURL = Bundle.main.url(forResource: "editor", withExtension: "html", subdirectory: "EditorBundle") {
            webView.loadFileURL(editorURL, allowingReadAccessTo: editorURL.deletingLastPathComponent())
        }

        return webView
    }

    func updateNSView(_ nsView: WKWebView, context: Context) {}
}
```

### Month 3: Sync Engine and Offline

#### 3A. Sync Engine

Background sync between SwiftData and Django API.

```swift
import SwiftData
import Foundation

actor SyncEngine {
    private let api = StudioAPIClient.shared
    private var modelContext: ModelContext

    init(modelContext: ModelContext) {
        self.modelContext = modelContext
    }

    // Pull: server to local
    func pullAll() async throws {
        let serverItems: [APIContentItem] = try await api.fetchContentList()

        for apiItem in serverItems {
            let descriptor = FetchDescriptor<ContentItem>(
                predicate: #Predicate { $0.id == apiItem.id }
            )
            let existing = try modelContext.fetch(descriptor).first

            if let local = existing {
                // Conflict check: if local is dirty, compare timestamps
                if local.isDirty {
                    let serverDate = apiItem.updatedAt
                    let localDate = local.updatedAt
                    if serverDate > localDate {
                        // Server wins: overwrite local
                        updateLocal(local, from: apiItem)
                        local.isDirty = false
                    }
                    // else: local wins, will push on next pushDirty()
                } else {
                    updateLocal(local, from: apiItem)
                }
            } else {
                // New item from server
                let item = ContentItem.from(apiItem)
                modelContext.insert(item)
            }
        }

        try modelContext.save()
    }

    // Push: local dirty items to server
    func pushDirty() async throws {
        let descriptor = FetchDescriptor<ContentItem>(
            predicate: #Predicate { $0.isDirty == true }
        )
        let dirtyItems = try modelContext.fetch(descriptor)

        for item in dirtyItems {
            do {
                let saved: APIContentItem = try await api.saveContentItem(
                    contentType: item.contentType,
                    slug: item.slug,
                    body: SaveContentBody(title: item.title, body: item.body)
                )
                item.serverUpdatedAt = saved.updatedAt
                item.isDirty = false
                item.lastSyncedAt = Date()
            } catch {
                // Network error: leave dirty, retry next cycle
                continue
            }
        }

        try modelContext.save()
    }

    // Full sync cycle
    func sync() async {
        do {
            try await pushDirty()  // Push first to avoid losing local changes
            try await pullAll()
        } catch {
            // Log error, retry on next cycle
        }
    }
}
```

#### 3B. Background Sync Scheduling

```swift
import SwiftUI

@main
struct StudioMacApp: App {
    let modelContainer: ModelContainer
    @State private var syncEngine: SyncEngine?
    @Environment(\.scenePhase) private var scenePhase

    init() {
        let schema = Schema([
            ContentItem.self,
            ContentSheet.self,
            StashItem.self,
            ContentTask.self,
            SyncState.self,
        ])
        modelContainer = try! ModelContainer(for: schema)
    }

    var body: some Scene {
        WindowGroup {
            MainWindow()
                .modelContainer(modelContainer)
                .task {
                    let context = modelContainer.mainContext
                    syncEngine = SyncEngine(modelContext: context)
                    // Initial sync
                    await syncEngine?.sync()
                    // Periodic sync every 30 seconds
                    startPeriodicSync()
                }
        }
        .commands {
            StudioCommands()
        }

        // Quick Capture window (triggered by global shortcut)
        Window("Quick Capture", id: "quick-capture") {
            QuickCaptureWindow()
                .modelContainer(modelContainer)
        }
        .windowStyle(.hiddenTitleBar)
        .windowResizability(.contentSize)
        .defaultPosition(.center)
    }

    private func startPeriodicSync() {
        Timer.scheduledTimer(withTimeInterval: 30, repeats: true) { _ in
            Task { await syncEngine?.sync() }
        }
    }
}
```

### Month 4: Polish and Platform Features

#### 4A. Global Quick Capture (macOS)

```swift
import Carbon.HIToolbox

final class GlobalShortcutService {
    static let shared = GlobalShortcutService()

    func register() {
        // Cmd+Shift+Space to open Quick Capture
        NSEvent.addGlobalMonitorForEvents(matching: .keyDown) { event in
            if event.modifierFlags.contains([.command, .shift])
                && event.keyCode == UInt16(kVK_Space) {
                NSApp.sendAction(#selector(AppDelegate.openQuickCapture), to: nil, from: nil)
            }
        }
    }
}
```

Quick Capture: a floating window with a single text field, content type picker
(defaults to last used), and a save button. Saves to SwiftData immediately,
syncs to Django in background. Under 5 seconds from shortcut to saved.

#### 4B. Native Menu Bar

```swift
struct StudioCommands: Commands {
    var body: some Commands {
        CommandGroup(replacing: .newItem) {
            Button("New Essay") { /* create essay */ }
                .keyboardShortcut("n", modifiers: [.command])
            Button("New Field Note") { /* create field note */ }
                .keyboardShortcut("n", modifiers: [.command, .shift])
            Divider()
            Button("Quick Capture") { /* open capture window */ }
                .keyboardShortcut(.space, modifiers: [.command, .shift])
        }

        CommandGroup(after: .sidebar) {
            Button("Toggle Sidebar") { /* toggle */ }
                .keyboardShortcut("b", modifiers: [.command])
        }

        CommandMenu("Editor") {
            Button("Save") { /* save */ }
                .keyboardShortcut("s", modifiers: [.command])
            Button("Advance Stage") { /* advance */ }
                .keyboardShortcut(.return, modifiers: [.command, .shift])
            Divider()
            Button("Toggle Typewriter Mode") { /* toggle */ }
                .keyboardShortcut("t", modifiers: [.command, .shift])
            Button("Toggle Zen Mode") { /* toggle */ }
                .keyboardShortcut("z", modifiers: [.command, .shift])
        }
    }
}
```

#### 4C. System Notifications

When an engine pass discovers new connections (polled from Index API):

```swift
import UserNotifications

func checkForNewConnections(for item: ContentItem) async {
    guard let trail = try? await IndexAPIClient.shared.fetchTrail(slug: item.slug) else { return }

    let newBacklinks = trail.backlinks.filter { backlink in
        // Compare against locally cached backlinks
        !cachedBacklinks.contains(where: { $0.contentSlug == backlink.contentSlug })
    }

    if !newBacklinks.isEmpty {
        let content = UNMutableNotificationContent()
        content.title = "New connections found"
        content.body = "\(newBacklinks.count) new connection\(newBacklinks.count == 1 ? "" : "s") for \"\(item.title)\""
        content.sound = .default

        let request = UNNotificationRequest(
            identifier: "connection-\(item.slug)-\(Date().timeIntervalSince1970)",
            content: content,
            trigger: nil
        )
        try? await UNUserNotificationCenter.current().add(request)
    }
}
```

---

## Phase 2: iOS App (Months 5 through 7)

### Shared Code (StudioKit)

The entire `StudioKit/` package is shared between macOS and iOS. This includes:
- API clients (StudioAPIClient, IndexAPIClient)
- SwiftData models (ContentItem, Sheet, StashItem, etc.)
- Sync engine
- Editor bridge protocol
- Type definitions

The iOS app imports StudioKit and adds iOS-specific views and extensions.

### iOS Layout Adaptation

```swift
struct TabRootView: View {
    @State private var selectedTab = 0

    var body: some View {
        TabView(selection: $selectedTab) {
            ContentListView()
                .tabItem { Label("Writing", systemImage: "doc.text") }
                .tag(0)

            QuickCaptureSheet()
                .tabItem { Label("Capture", systemImage: "plus.circle") }
                .tag(1)

            SettingsView()
                .tabItem { Label("Settings", systemImage: "gearshape") }
                .tag(2)
        }
    }
}
```

**Key iOS adaptations:**
- Sidebar becomes a navigation stack (content type list then items)
- Sheets panel becomes a bottom sheet (`.sheet` or custom detent)
- Workbench tabs (Research, Outline, Stash, etc.) become a bottom sheet with segmented control
- Quick Capture lives in a dedicated tab for one-tap access
- Share Extension captures URLs from Safari into the sourcebox

### Share Extension

```swift
// ShareExtension/ShareViewController.swift
import Social
import UniformTypeIdentifiers
import StudioKit

class ShareViewController: SLComposeServiceViewController {
    override func didSelectPost() {
        guard let items = extensionContext?.inputItems as? [NSExtensionItem] else {
            extensionContext?.completeRequest(returningItems: nil)
            return
        }

        for item in items {
            for provider in item.attachments ?? [] {
                if provider.hasItemConformingToTypeIdentifier(UTType.url.identifier) {
                    provider.loadItem(forTypeIdentifier: UTType.url.identifier) { data, _ in
                        guard let url = data as? URL else { return }
                        // Save to shared app group container for the main app to sync
                        SharedCapture.save(url: url.absoluteString, text: self.contentText)
                        self.extensionContext?.completeRequest(returningItems: nil)
                    }
                }
            }
        }
    }
}
```

### WidgetKit Quick Capture

```swift
import WidgetKit
import SwiftUI

struct QuickCaptureWidget: Widget {
    var body: some WidgetConfiguration {
        StaticConfiguration(kind: "QuickCapture", provider: Provider()) { _ in
            Link(destination: URL(string: "studio://capture")!) {
                VStack(spacing: 8) {
                    Image(systemName: "plus.circle.fill")
                        .font(.title)
                        .foregroundStyle(Color(hex: "#B45A2D"))
                    Text("Quick Capture")
                        .font(.custom("JetBrains Mono", size: 10))
                        .foregroundStyle(.secondary)
                }
                .frame(maxWidth: .infinity, maxHeight: .infinity)
            }
        }
        .configurationDisplayName("Quick Capture")
        .description("Capture ideas instantly")
        .supportedFamilies([.systemSmall])
    }
}
```

---

## Editor Bundle Build Process

The Tiptap editor must be extracted from the Next.js project and bundled as standalone
HTML/JS/CSS that runs inside WKWebView without a server.

### Build Script

```bash
#!/bin/bash
# scripts/build-editor-bundle.sh
# Run from the travisgilbert.me repo root

# 1. Bundle Tiptap editor + all extensions into a single JS file
npx esbuild \
  src/components/studio/TiptapEditor.standalone.tsx \
  --bundle \
  --outfile=editor-bundle/editor.js \
  --format=iife \
  --global-name=StudioEditor \
  --loader:.tsx=tsx \
  --loader:.ts=ts \
  --external:react \
  --external:react-dom \
  --minify

# 2. Extract editor-specific CSS from studio.css
# (writing surface, page, prose, toolbar, highlight, contain block styles)
node scripts/extract-editor-css.js > editor-bundle/editor.css

# 3. Copy bridge.js
cp src/native/bridge.js editor-bundle/bridge.js

# 4. Generate editor.html (loads React, editor bundle, CSS, bridge)
cat > editor-bundle/editor.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="stylesheet" href="editor.css">
  <style>
    body { margin: 0; background: transparent; }
    /* Load Studio fonts from bundled files */
    @font-face { font-family: 'Amarna'; src: url('fonts/Amarna.woff2'); }
    @font-face { font-family: 'JetBrains Mono'; src: url('fonts/JetBrainsMono.woff2'); }
    @font-face { font-family: 'Vollkorn'; src: url('fonts/Vollkorn.woff2'); }
  </style>
</head>
<body class="studio-theme">
  <div id="editor-root"></div>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/react/19.0.0/umd/react.production.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/19.0.0/umd/react-dom.production.min.js"></script>
  <script src="editor.js"></script>
  <script src="bridge.js"></script>
  <script>
    StudioEditor.mount(document.getElementById('editor-root'));
  </script>
</body>
</html>
EOF
```

### Standalone Editor Entry Point

A new file that mounts TiptapEditor without Next.js dependencies:

```typescript
// src/components/studio/TiptapEditor.standalone.tsx
// Stripped-down version that:
// 1. Removes Next.js imports (useRouter, Link, etc.)
// 2. Removes commonplace-api calls (these go through the native bridge instead)
// 3. Exposes window.__studioEditor after mount
// 4. Dispatches 'studio:editor-ready' event

import { createRoot } from 'react-dom/client';
import TiptapEditor from './TiptapEditor';

export function mount(container: HTMLElement) {
    const root = createRoot(container);
    root.render(
        <TiptapEditor
            onEditorReady={(editor) => {
                (window as any).__studioEditor = editor;
                document.dispatchEvent(new Event('studio:editor-ready'));
            }}
            onUpdate={({ markdown }) => {
                // Bridge handles forwarding to Swift
            }}
        />
    );
}
```

This file needs careful work to decouple from Next.js. The wiki link search
currently calls `searchCommonplace` and `searchObjects` directly. In the
standalone version, these calls go through the native bridge instead:
Swift handles the API calls and pushes results back via `evaluateJavaScript`.

---

## API Endpoint Inventory

Endpoints the native app must call (grouped by priority):

### Critical (Month 1)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/content/` | List all content items |
| GET | `/content/{type}/{slug}/` | Get single item |
| POST | `/content/{type}/` | Create item |
| PATCH | `/content/{type}/{slug}/` | Save item |
| DELETE | `/content/{type}/{slug}/` | Delete item |
| PATCH | `/content/{type}/{slug}/stage/` | Update pipeline stage |
| GET | `/content/{type}/{slug}/sheets/` | List sheets |
| POST | `/content/{type}/{slug}/sheets/` | Create sheet |
| PATCH | `/content/{type}/{slug}/sheets/{id}/` | Update sheet |
| DELETE | `/content/{type}/{slug}/sheets/{id}/` | Delete sheet |
| GET | `/search/?q={query}` | Search content |

### Important (Month 2)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/content/{type}/{slug}/stash/` | Get stash items |
| POST | `/content/{type}/{slug}/stash/` | Create stash item |
| DELETE | `/content/{type}/{slug}/stash/{id}/` | Delete stash item |
| GET | `/content/{type}/{slug}/tasks/` | Get tasks |
| POST | `/content/{type}/{slug}/tasks/` | Create task |
| PATCH | `/content/{type}/{slug}/tasks/{id}/` | Update task |
| GET | `/content/{type}/{slug}/revisions/` | List revisions |
| POST | `/content/{type}/{slug}/revisions/` | Create revision |
| GET | `/timeline/` | Dashboard timeline |

### Research (Month 3)

| Method | Path | Service |
|--------|------|---------|
| GET | `/api/v1/trail/{slug}/` | Index API: research trail |
| GET | `/api/v1/objects/search/` | Index API: search objects |
| POST | `/sourcebox/submit-url/` | Studio API: add source |
| POST | `/sourcebox/upload/` | Studio API: upload file |
| GET | `/content/{type}/{slug}/sources/` | Studio API: list sources |
| POST | `/content/{type}/{slug}/analyze/` | Studio API: draft analysis |

### Deferred (Post-launch)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/content/{type}/{slug}/publish/` | Publish to site |
| GET | `/settings/` | Studio settings |
| PATCH | `/settings/` | Update settings |
| POST | `/content/{type}/{slug}/claims/audit/` | Claim audit |
| POST | `/content/{type}/{slug}/entities/` | Entity extraction |

---

## Distribution

### macOS
- **Signing:** Developer ID (direct download from travisgilbert.me)
- **Notarization:** Required for Gatekeeper. Use `xcrun notarytool` in CI.
- **Updates:** Sparkle framework for auto-update checks
- **Minimum OS:** macOS 15 (Sequoia) for latest SwiftData features

### iOS
- **TestFlight:** Beta distribution during development
- **App Store:** When stable. Free tier (no paywall for v1).
- **Minimum OS:** iOS 18 for SwiftData and WidgetKit improvements

---

## Open Questions

1. **Editor bundle CDN vs bundled:** Should React/ReactDOM be bundled into
   the app (larger binary, works offline) or loaded from CDN on first launch
   (smaller binary, needs initial network)? Recommendation: bundle everything
   for offline reliability.

2. **Font licensing:** Amarna, Vollkorn, JetBrains Mono, IBM Plex Sans, Caudex
   all need to be bundled. Verify licenses allow embedding in a native app.
   JetBrains Mono is OFL. Vollkorn is OFL. IBM Plex is OFL. Check Amarna.

3. **Wiki link search in offline mode:** When offline, wiki link `[[` search
   can only search locally cached content (SwiftData). Should the UI indicate
   that results may be incomplete, or silently return what's available?

4. **Y.js in WKWebView:** The web version uses Y.js for local IndexedDB
   persistence. In the native app, SwiftData handles persistence. Should Y.js
   be stripped from the editor bundle to avoid double-caching? Recommendation:
   strip it. SwiftData is the local cache; Y.js adds complexity without benefit
   in the native context.

5. **App name:** "Studio" is generic. Consider "Studio by Travis Gilbert" or
   a distinctive name for App Store discoverability. Not urgent for v1.

# StudioKit Foundation: Swift Package Spec

## 2026-03-28

**Repo:** `Travis-Gilbert/Studio`
**Target:** Create the StudioKit Swift package with SwiftData models, type enums,
API response types, and the StudioAPIClient. This is the shared foundation that
both the macOS and iOS app targets will import.

**Prerequisites:** Travis must create the Xcode multiplatform project first (manual).
This spec creates the StudioKit package and all its source files.

**No em dashes anywhere.**

---

## Manual Step (Travis, before running this spec)

Open Xcode. File > New > Project > Multiplatform App.
- Product Name: Studio
- Organization: Travis Gilbert
- Bundle ID: me.travisgilbert.studio
- Storage: SwiftData
- Set deployment targets: macOS 15.0, iOS 18.0
- Save into the Studio repo root

Then: File > New > Package > Library.
- Name: StudioKit
- Save inside the Studio repo root as `StudioKit/`
- In Xcode, drag StudioKit into the project navigator
- Add StudioKit as a dependency of both the macOS and iOS targets

Commit the Xcode project files. Then run this spec.

---

## Batch 1: Types and Enums

### 1A. ContentType enum

**File: `StudioKit/Sources/StudioKit/Types/ContentType.swift`**

```swift
import Foundation

/// The six content types Studio manages.
/// Raw values match the frontend slug format (hyphenated).
public enum ContentType: String, CaseIterable, Codable, Sendable {
    case essay = "essay"
    case fieldNote = "field-note"
    case shelf = "shelf"
    case video = "video"
    case project = "project"
    case toolkit = "toolkit"

    /// Human-readable label for UI display.
    public var label: String {
        switch self {
        case .essay: return "Essay"
        case .fieldNote: return "Field Note"
        case .shelf: return "Shelf Entry"
        case .video: return "Video"
        case .project: return "Project"
        case .toolkit: return "Toolkit"
        }
    }

    /// Plural label for list headers.
    public var pluralLabel: String {
        switch self {
        case .essay: return "Essays"
        case .fieldNote: return "Field Notes"
        case .shelf: return "Shelf"
        case .video: return "Videos"
        case .project: return "Projects"
        case .toolkit: return "Toolkit"
        }
    }

    /// Hex color string for UI accent per type.
    public var colorHex: String {
        switch self {
        case .essay: return "#B45A2D"
        case .fieldNote: return "#3A8A9A"
        case .shelf: return "#D4AA4A"
        case .video: return "#6A9A5A"
        case .project: return "#D4AA4A"
        case .toolkit: return "#B45A2D"
        }
    }

    /// The path segment Django expects in API URLs.
    /// Django uses underscores: field_note, not field-note.
    public var apiPathSegment: String {
        switch self {
        case .fieldNote: return "field_note"
        default: return rawValue
        }
    }

    /// Initialize from a Django API content_type string (handles both
    /// hyphenated and underscored variants).
    public init?(apiValue: String) {
        let normalized = apiValue.trimmingCharacters(in: .whitespaces).lowercased()
        switch normalized {
        case "essay": self = .essay
        case "field-note", "field_note", "field-notes", "field_notes": self = .fieldNote
        case "shelf": self = .shelf
        case "video", "videos": self = .video
        case "project", "projects": self = .project
        case "toolkit": self = .toolkit
        default: return nil
        }
    }
}
```

### 1B. Stage enum

**File: `StudioKit/Sources/StudioKit/Types/Stage.swift`**

```swift
import Foundation

/// Writing pipeline stages. Order matters: each stage represents progress.
public enum Stage: String, CaseIterable, Codable, Sendable {
    case idea = "idea"
    case research = "research"
    case drafting = "drafting"
    case revising = "revising"
    case production = "production"
    case published = "published"

    public var label: String {
        switch self {
        case .idea: return "Idea"
        case .research: return "Research"
        case .drafting: return "Drafting"
        case .revising: return "Editing"
        case .production: return "Production"
        case .published: return "Published"
        }
    }

    public var colorHex: String {
        switch self {
        case .idea: return "#9A8E82"
        case .research: return "#2D5F6B"
        case .drafting: return "#C49A4A"
        case .revising: return "#8A6A9A"
        case .production: return "#B45A2D"
        case .published: return "#5A7A4A"
        }
    }

    /// Pipeline order (0 = first).
    public var order: Int {
        switch self {
        case .idea: return 0
        case .research: return 1
        case .drafting: return 2
        case .revising: return 3
        case .production: return 4
        case .published: return 5
        }
    }

    /// The next stage in the pipeline, or nil if already published.
    public var next: Stage? {
        let all = Stage.allCases
        guard let idx = all.firstIndex(of: self), idx + 1 < all.count else { return nil }
        return all[idx + 1]
    }

    /// The previous stage, or nil if already at idea.
    public var previous: Stage? {
        let all = Stage.allCases
        guard let idx = all.firstIndex(of: self), idx > 0 else { return nil }
        return all[idx - 1]
    }
}
```

### Verification

```bash
cd StudioKit && swift build
```

Should compile with zero errors. No dependencies beyond Foundation.

---

## Batch 2: API Response Types (Codable Structs)

These are the raw shapes Django returns. They use snake_case because
JSONDecoder.convertFromSnakeCase handles the mapping. They are NOT
SwiftData models; they are transient Codable structs for network I/O.

### 2A. API types

**File: `StudioKit/Sources/StudioKit/API/APITypes.swift`**

```swift
import Foundation

// MARK: - Content

/// Raw API response for a content item.
/// Django returns snake_case; JSONDecoder converts to camelCase.
public struct APIContentItem: Codable, Sendable {
    public let id: StringOrInt
    public let title: String
    public let slug: String
    public let contentType: String
    public let stage: String
    public let body: String?
    public let excerpt: String?
    public let wordCount: Int?
    public let tags: [String]?
    public let createdAt: String
    public let updatedAt: String
    public let publishedAt: String?
}

/// Content list response can be either a bare array or wrapped in {results: [...]}.
public struct APIContentListResponse: Codable, Sendable {
    public let results: [APIContentItem]?
}

// MARK: - Sheets

public struct APISheet: Codable, Sendable {
    public let id: String
    public let contentType: String?
    public let contentSlug: String?
    public let order: Int?
    public let title: String
    public let body: String?
    public let isMaterial: Bool?
    public let status: String?
    public let wordCount: Int?
    public let wordCountTarget: Int?
    public let createdAt: String?
    public let updatedAt: String?
}

public struct APISheetsResponse: Codable, Sendable {
    public let sheets: [APISheet]?
}

// MARK: - Stash

public struct APIStashItem: Codable, Sendable {
    public let id: Int
    public let text: String
    public let category: String?
    public let createdAt: String
}

public struct APIStashResponse: Codable, Sendable {
    public let items: [APIStashItem]?
}

// MARK: - Tasks

public struct APIContentTask: Codable, Sendable {
    public let id: Int
    public let text: String
    public let done: Bool
    public let doneAt: String?
    public let createdAt: String
    public let ticktickTaskId: String?
    public let ticktickProjectId: String?
}

public struct APITasksResponse: Codable, Sendable {
    public let tasks: [APIContentTask]?
}

// MARK: - Revisions

public struct APIRevision: Codable, Sendable {
    public let id: Int
    public let revisionNumber: Int
    public let title: String
    public let wordCount: Int?
    public let label: String?
    public let source: String?
    public let createdAt: String
}

public struct APIRevisionDetail: Codable, Sendable {
    public let id: Int
    public let revisionNumber: Int
    public let title: String
    public let body: String
    public let wordCount: Int?
    public let label: String?
    public let source: String?
    public let createdAt: String
}

public struct APIRevisionsResponse: Codable, Sendable {
    public let revisions: [APIRevision]?
}

// MARK: - Search

public struct APISearchResult: Codable, Sendable {
    public let id: String
    public let title: String
    public let slug: String?
    public let contentType: String?
    public let label: String?
    public let excerpt: String?
}

// MARK: - Sources

public struct APISource: Codable, Sendable {
    public let id: Int
    public let title: String
    public let description: String?
    public let url: String?
    public let siteName: String?
    public let inputType: String?
    public let scrapeStatus: String?
    public let content: String?
    public let createdAt: String?
}

// MARK: - Research Trail (Index API)

public struct APIResearchTrailSource: Codable, Sendable {
    public let id: Int
    public let title: String
    public let slug: String?
    public let creator: String?
    public let sourceType: String?
    public let url: String?
    public let publication: String?
    public let publicAnnotation: String?
    public let role: String?
    public let keyQuote: String?
}

public struct APIResearchTrailBacklink: Codable, Sendable {
    public let contentType: String
    public let contentSlug: String
    public let contentTitle: String
    public let sharedSources: [SharedSource]?

    public struct SharedSource: Codable, Sendable {
        public let sourceId: Int
        public let sourceTitle: String
    }
}

public struct APIResearchTrail: Codable, Sendable {
    public let slug: String?
    public let contentType: String?
    public let sources: [APIResearchTrailSource]?
    public let backlinks: [APIResearchTrailBacklink]?
}

// MARK: - Draft Analysis (ML)

public struct APIDraftAnalysis: Codable, Sendable {
    public let connections: [APIDraftConnection]?
    public let similar: [APISimilarObject]?
}

public struct APIDraftConnection: Codable, Sendable {
    public let targetTitle: String
    public let targetSlug: String?
    public let targetType: String?
    public let score: Double?
    public let reason: String?
}

public struct APISimilarObject: Codable, Sendable {
    public let title: String
    public let objectType: String?
    public let similarity: Double?
}

// MARK: - Utility types

/// Django sometimes returns id as a string, sometimes as an int.
/// This handles both transparently.
public struct StringOrInt: Codable, Sendable, Hashable {
    public let stringValue: String

    public init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        if let intVal = try? container.decode(Int.self) {
            stringValue = String(intVal)
        } else if let strVal = try? container.decode(String.self) {
            stringValue = strVal
        } else {
            stringValue = ""
        }
    }

    public func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()
        try container.encode(stringValue)
    }
}

/// Empty response for endpoints that return no meaningful body.
public struct EmptyResponse: Codable, Sendable {
    public let ok: Bool?
    public let deleted: Bool?
}

// MARK: - Request bodies

public struct SaveContentBody: Codable, Sendable {
    public let title: String?
    public let body: String?
    public let excerpt: String?
    public let tags: [String]?

    public init(title: String? = nil, body: String? = nil, excerpt: String? = nil, tags: [String]? = nil) {
        self.title = title
        self.body = body
        self.excerpt = excerpt
        self.tags = tags
    }
}

public struct CreateContentBody: Codable, Sendable {
    public let title: String?

    public init(title: String? = nil) {
        self.title = title
    }
}

public struct SetStageBody: Codable, Sendable {
    public let stage: String

    public init(stage: String) {
        self.stage = stage
    }
}

public struct CreateSheetBody: Codable, Sendable {
    public let title: String?
    public let body: String?
    public let sortOrder: Int?

    public init(title: String? = nil, body: String? = nil, sortOrder: Int? = nil) {
        self.title = title
        self.body = body
        self.sortOrder = sortOrder
    }
}

public struct UpdateSheetBody: Codable, Sendable {
    public let title: String?
    public let body: String?
    public let order: Int?
    public let isMaterial: Bool?
    public let status: String?
    public let wordCountTarget: Int?

    public init(
        title: String? = nil,
        body: String? = nil,
        order: Int? = nil,
        isMaterial: Bool? = nil,
        status: String? = nil,
        wordCountTarget: Int? = nil
    ) {
        self.title = title
        self.body = body
        self.order = order
        self.isMaterial = isMaterial
        self.status = status
        self.wordCountTarget = wordCountTarget
    }
}

public struct DeleteSheetBody: Codable, Sendable {
    public let action = "delete"
}

public struct ReorderSheetsBody: Codable, Sendable {
    public let ids: [String]

    public init(ids: [String]) {
        self.ids = ids
    }
}
```

### Verification

```bash
cd StudioKit && swift build
```

---

## Batch 3: API Client

### 3A. Error types

**File: `StudioKit/Sources/StudioKit/API/StudioAPIError.swift`**

```swift
import Foundation

public enum StudioAPIError: Error, LocalizedError {
    case invalidURL
    case invalidResponse
    case httpError(status: Int, body: String?)
    case decodingError(Error)
    case networkUnavailable
    case timeout

    public var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "Invalid API URL"
        case .invalidResponse:
            return "Invalid response from server"
        case .httpError(let status, let body):
            return "HTTP \(status): \(body ?? "No details")"
        case .decodingError(let error):
            return "Failed to decode response: \(error.localizedDescription)"
        case .networkUnavailable:
            return "No network connection"
        case .timeout:
            return "Request timed out"
        }
    }
}
```

### 3B. JSON coders

**File: `StudioKit/Sources/StudioKit/API/JSONCoders.swift`**

```swift
import Foundation

extension JSONDecoder {
    /// Studio decoder: converts snake_case keys from Django to camelCase Swift properties.
    public static let studio: JSONDecoder = {
        let decoder = JSONDecoder()
        decoder.keyDecodingStrategy = .convertFromSnakeCase
        return decoder
    }()
}

extension JSONEncoder {
    /// Studio encoder: converts camelCase Swift properties to snake_case for Django.
    public static let studio: JSONEncoder = {
        let encoder = JSONEncoder()
        encoder.keyEncodingStrategy = .convertToSnakeCase
        return encoder
    }()
}
```

### 3C. StudioAPIClient

**File: `StudioKit/Sources/StudioKit/API/StudioAPIClient.swift`**

IMPORTANT: The Django API uses POST for all mutations (create, update, delete,
set-stage). Not PATCH or DELETE. URL patterns include the action:
- Create: `POST /content/{type}/create/`
- Update: `POST /content/{type}/{slug}/update/`
- Delete: `POST /content/{type}/{slug}/delete/`
- Set stage: `POST /content/{type}/{slug}/set-stage/`

```swift
import Foundation

/// Thread-safe API client for the Studio Django backend.
/// All methods are async and throw StudioAPIError on failure.
public actor StudioAPIClient {
    public static let shared = StudioAPIClient()

    private let baseURL: URL
    private var token: String
    private let session: URLSession

    public init(
        baseURL: URL = URL(string: "https://draftroom.travisgilbert.me/editor/api")!,
        token: String = ""
    ) {
        self.baseURL = baseURL
        self.token = token

        let config = URLSessionConfiguration.default
        config.timeoutIntervalForRequest = 15
        config.waitsForConnectivity = true
        self.session = URLSession(configuration: config)
    }

    /// Update the auth token (e.g., after reading from Keychain).
    public func setToken(_ newToken: String) {
        self.token = newToken
    }

    // MARK: - Generic fetch

    /// Generic typed request. Handles auth, encoding, decoding, and errors.
    public func fetch<T: Decodable>(
        _ path: String,
        method: String = "GET",
        body: (any Encodable)? = nil
    ) async throws -> T {
        guard let url = URL(string: path, relativeTo: baseURL) else {
            throw StudioAPIError.invalidURL
        }

        var request = URLRequest(url: url)
        request.httpMethod = method
        request.setValue("application/json", forHTTPHeaderField: "Accept")

        if !token.isEmpty {
            request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }

        if let body {
            request.setValue("application/json", forHTTPHeaderField: "Content-Type")
            request.httpBody = try JSONEncoder.studio.encode(body)
        }

        let data: Data
        let response: URLResponse

        do {
            (data, response) = try await session.data(for: request)
        } catch let error as URLError where error.code == .notConnectedToInternet {
            throw StudioAPIError.networkUnavailable
        } catch let error as URLError where error.code == .timedOut {
            throw StudioAPIError.timeout
        } catch {
            throw error
        }

        guard let http = response as? HTTPURLResponse else {
            throw StudioAPIError.invalidResponse
        }

        guard (200...299).contains(http.statusCode) else {
            throw StudioAPIError.httpError(
                status: http.statusCode,
                body: String(data: data, encoding: .utf8)
            )
        }

        do {
            return try JSONDecoder.studio.decode(T.self, from: data)
        } catch {
            throw StudioAPIError.decodingError(error)
        }
    }

    // MARK: - Content CRUD

    /// List all content items, optionally filtered by type.
    public func fetchContentList(contentType: ContentType? = nil) async throws -> [APIContentItem] {
        let path: String
        if let ct = contentType {
            path = "/content/\(ct.apiPathSegment)/"
        } else {
            path = "/content/"
        }

        // Response can be a bare array or {results: [...]}
        // Try wrapped first, fall back to bare array
        do {
            let wrapped: APIContentListResponse = try await fetch(path)
            return wrapped.results ?? []
        } catch StudioAPIError.decodingError {
            return try await fetch(path)
        }
    }

    /// Get a single content item by type and slug.
    public func fetchContentItem(
        contentType: ContentType,
        slug: String
    ) async throws -> APIContentItem {
        return try await fetch("/content/\(contentType.apiPathSegment)/\(slug)/")
    }

    /// Create a new content item. Django path: POST /content/{type}/create/
    public func createContentItem(
        contentType: ContentType,
        body: CreateContentBody? = nil
    ) async throws -> APIContentItem {
        return try await fetch(
            "/content/\(contentType.apiPathSegment)/create/",
            method: "POST",
            body: body
        )
    }

    /// Save (update) a content item. Django path: POST /content/{type}/{slug}/update/
    public func saveContentItem(
        contentType: ContentType,
        slug: String,
        body: SaveContentBody
    ) async throws -> APIContentItem {
        return try await fetch(
            "/content/\(contentType.apiPathSegment)/\(slug)/update/",
            method: "POST",
            body: body
        )
    }

    /// Delete a content item. Django path: POST /content/{type}/{slug}/delete/
    public func deleteContentItem(
        contentType: ContentType,
        slug: String
    ) async throws {
        let _: EmptyResponse = try await fetch(
            "/content/\(contentType.apiPathSegment)/\(slug)/delete/",
            method: "POST",
            body: EmptyResponse(ok: nil, deleted: nil)
        )
    }

    /// Update pipeline stage. Django path: POST /content/{type}/{slug}/set-stage/
    public func updateStage(
        contentType: ContentType,
        slug: String,
        stage: Stage
    ) async throws -> APIContentItem {
        return try await fetch(
            "/content/\(contentType.apiPathSegment)/\(slug)/set-stage/",
            method: "POST",
            body: SetStageBody(stage: stage.rawValue)
        )
    }

    // MARK: - Sheets

    /// List sheets for a content item.
    public func fetchSheets(
        contentType: ContentType,
        slug: String
    ) async throws -> [APISheet] {
        let response: APISheetsResponse = try await fetch(
            "/content/\(contentType.apiPathSegment)/\(slug)/sheets/"
        )
        return response.sheets ?? []
    }

    /// Create a new sheet.
    public func createSheet(
        contentType: ContentType,
        slug: String,
        body: CreateSheetBody
    ) async throws -> APISheet {
        return try await fetch(
            "/content/\(contentType.apiPathSegment)/\(slug)/sheets/",
            method: "POST",
            body: body
        )
    }

    /// Update a sheet. Django uses POST, not PATCH.
    public func updateSheet(
        contentType: ContentType,
        slug: String,
        sheetId: String,
        body: UpdateSheetBody
    ) async throws -> APISheet {
        return try await fetch(
            "/content/\(contentType.apiPathSegment)/\(slug)/sheets/\(sheetId)/",
            method: "POST",
            body: body
        )
    }

    /// Delete a sheet. Django expects {action: "delete"} in body.
    public func deleteSheet(
        contentType: ContentType,
        slug: String,
        sheetId: String
    ) async throws {
        let _: EmptyResponse = try await fetch(
            "/content/\(contentType.apiPathSegment)/\(slug)/sheets/\(sheetId)/",
            method: "POST",
            body: DeleteSheetBody()
        )
    }

    /// Reorder sheets.
    public func reorderSheets(
        contentType: ContentType,
        slug: String,
        ids: [String]
    ) async throws -> [APISheet] {
        let response: APISheetsResponse = try await fetch(
            "/content/\(contentType.apiPathSegment)/\(slug)/sheets/reorder/",
            method: "POST",
            body: ReorderSheetsBody(ids: ids)
        )
        return response.sheets ?? []
    }

    // MARK: - Stash

    public func fetchStash(
        contentType: ContentType,
        slug: String
    ) async throws -> [APIStashItem] {
        let response: APIStashResponse = try await fetch(
            "/content/\(contentType.apiPathSegment)/\(slug)/stash/"
        )
        return response.items ?? []
    }

    public func createStashItem(
        contentType: ContentType,
        slug: String,
        text: String
    ) async throws -> APIStashItem {
        return try await fetch(
            "/content/\(contentType.apiPathSegment)/\(slug)/stash/",
            method: "POST",
            body: ["text": text]
        )
    }

    public func deleteStashItem(
        contentType: ContentType,
        slug: String,
        id: Int
    ) async throws {
        let _: EmptyResponse = try await fetch(
            "/content/\(contentType.apiPathSegment)/\(slug)/stash/\(id)/delete/",
            method: "POST"
        )
    }

    // MARK: - Tasks

    public func fetchTasks(
        contentType: ContentType,
        slug: String
    ) async throws -> [APIContentTask] {
        let response: APITasksResponse = try await fetch(
            "/content/\(contentType.apiPathSegment)/\(slug)/tasks/"
        )
        return response.tasks ?? []
    }

    public func createTask(
        contentType: ContentType,
        slug: String,
        text: String
    ) async throws -> APIContentTask {
        return try await fetch(
            "/content/\(contentType.apiPathSegment)/\(slug)/tasks/",
            method: "POST",
            body: ["text": text]
        )
    }

    public func updateTask(
        contentType: ContentType,
        slug: String,
        id: Int,
        done: Bool? = nil,
        text: String? = nil
    ) async throws -> APIContentTask {
        var payload: [String: Any] = [:]
        if let done { payload["done"] = done }
        if let text { payload["text"] = text }
        // Can't use [String: Any] with Codable, use a struct
        return try await fetch(
            "/content/\(contentType.apiPathSegment)/\(slug)/tasks/\(id)/update/",
            method: "POST",
            body: UpdateTaskBody(done: done, text: text)
        )
    }

    public func deleteTask(
        contentType: ContentType,
        slug: String,
        id: Int
    ) async throws {
        let _: EmptyResponse = try await fetch(
            "/content/\(contentType.apiPathSegment)/\(slug)/tasks/\(id)/delete/",
            method: "POST"
        )
    }

    // MARK: - Revisions

    public func fetchRevisions(
        contentType: ContentType,
        slug: String
    ) async throws -> [APIRevision] {
        let response: APIRevisionsResponse = try await fetch(
            "/content/\(contentType.apiPathSegment)/\(slug)/revisions/"
        )
        return response.revisions ?? []
    }

    // MARK: - Search

    public func searchContent(query: String) async throws -> [APISearchResult] {
        let encoded = query.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? ""
        return try await fetch("/search/?q=\(encoded)")
    }

    // MARK: - Sources

    public func submitSourceURL(_ url: String) async throws -> APISource {
        return try await fetch(
            "/sourcebox/submit-url/",
            method: "POST",
            body: ["url": url]
        )
    }

    public func fetchSources(
        contentType: ContentType,
        slug: String
    ) async throws -> [APISource] {
        return try await fetch(
            "/content/\(contentType.apiPathSegment)/\(slug)/sources/"
        )
    }
}

// Additional request body that couldn't use dictionary
public struct UpdateTaskBody: Codable, Sendable {
    public let done: Bool?
    public let text: String?
}
```

### 3D. IndexAPIClient

**File: `StudioKit/Sources/StudioKit/API/IndexAPIClient.swift`**

```swift
import Foundation

/// Client for the Theseus Index API (research trail, object search).
/// No auth required for read endpoints.
public actor IndexAPIClient {
    public static let shared = IndexAPIClient()

    private let baseURL: URL
    private let session: URLSession

    public init(
        baseURL: URL = URL(string: "https://index-api-production.up.railway.app")!
    ) {
        self.baseURL = baseURL

        let config = URLSessionConfiguration.default
        config.timeoutIntervalForRequest = 5
        self.session = URLSession(configuration: config)
    }

    /// Fetch the research trail for a content item by slug.
    public func fetchTrail(slug: String) async throws -> APIResearchTrail? {
        guard let url = URL(string: "/api/v1/trail/\(slug)/", relativeTo: baseURL) else {
            return nil
        }

        var request = URLRequest(url: url)
        request.setValue("application/json", forHTTPHeaderField: "Accept")

        do {
            let (data, response) = try await session.data(for: request)
            guard let http = response as? HTTPURLResponse,
                  (200...299).contains(http.statusCode) else {
                return nil
            }
            return try JSONDecoder.studio.decode(APIResearchTrail.self, from: data)
        } catch {
            return nil
        }
    }

    /// Search CommonPlace objects.
    public func searchObjects(
        query: String,
        limit: Int = 10
    ) async throws -> [APISimilarObject] {
        let encoded = query.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? ""
        guard let url = URL(
            string: "/api/v1/objects/search/?q=\(encoded)&limit=\(limit)",
            relativeTo: baseURL
        ) else {
            return []
        }

        var request = URLRequest(url: url)
        request.setValue("application/json", forHTTPHeaderField: "Accept")

        do {
            let (data, response) = try await session.data(for: request)
            guard let http = response as? HTTPURLResponse,
                  (200...299).contains(http.statusCode) else {
                return []
            }
            return try JSONDecoder.studio.decode([APISimilarObject].self, from: data)
        } catch {
            return []
        }
    }
}
```

### Verification

```bash
cd StudioKit && swift build
```

---

## Batch 4: SwiftData Models

### 4A. ContentItem model

**File: `StudioKit/Sources/StudioKit/Models/ContentItem.swift`**

```swift
import SwiftData
import Foundation

@Model
public final class ContentItem {
    @Attribute(.unique)
    public var id: String

    public var title: String
    public var slug: String
    public var contentType: String
    public var stage: String
    public var body: String
    public var excerpt: String
    public var wordCount: Int
    public var tags: [String]
    public var createdAt: Date
    public var updatedAt: Date
    public var publishedAt: Date?
    public var nextMove: String?
    public var lastSessionSummary: String?

    // Sync bookkeeping
    public var isDirty: Bool = false
    public var lastSyncedAt: Date?
    public var serverUpdatedAt: Date?

    @Relationship(deleteRule: .cascade, inverse: \ContentSheet.contentItem)
    public var sheets: [ContentSheet] = []

    public init(
        id: String,
        title: String,
        slug: String,
        contentType: String,
        stage: String,
        body: String = "",
        excerpt: String = "",
        wordCount: Int = 0,
        tags: [String] = [],
        createdAt: Date = Date(),
        updatedAt: Date = Date(),
        publishedAt: Date? = nil
    ) {
        self.id = id
        self.title = title
        self.slug = slug
        self.contentType = contentType
        self.stage = stage
        self.body = body
        self.excerpt = excerpt
        self.wordCount = wordCount
        self.tags = tags
        self.createdAt = createdAt
        self.updatedAt = updatedAt
        self.publishedAt = publishedAt
    }

    /// Create from an API response.
    public static func from(_ api: APIContentItem) -> ContentItem {
        let item = ContentItem(
            id: api.id.stringValue,
            title: api.title,
            slug: api.slug,
            contentType: ContentType(apiValue: api.contentType)?.rawValue ?? api.contentType,
            stage: api.stage,
            body: api.body ?? "",
            excerpt: api.excerpt ?? "",
            wordCount: api.wordCount ?? 0,
            tags: api.tags ?? [],
            createdAt: ISO8601DateFormatter().date(from: api.createdAt) ?? Date(),
            updatedAt: ISO8601DateFormatter().date(from: api.updatedAt) ?? Date(),
            publishedAt: api.publishedAt.flatMap { ISO8601DateFormatter().date(from: $0) }
        )
        item.lastSyncedAt = Date()
        return item
    }

    /// Update local fields from an API response (preserving sync state).
    public func update(from api: APIContentItem) {
        title = api.title
        slug = api.slug
        contentType = ContentType(apiValue: api.contentType)?.rawValue ?? api.contentType
        stage = api.stage
        body = api.body ?? ""
        excerpt = api.excerpt ?? ""
        wordCount = api.wordCount ?? 0
        tags = api.tags ?? []
        updatedAt = ISO8601DateFormatter().date(from: api.updatedAt) ?? Date()
        publishedAt = api.publishedAt.flatMap { ISO8601DateFormatter().date(from: $0) }
        lastSyncedAt = Date()
    }

    /// Resolved content type enum, if valid.
    public var resolvedContentType: ContentType? {
        ContentType(apiValue: contentType)
    }

    /// Resolved stage enum, if valid.
    public var resolvedStage: Stage? {
        Stage(rawValue: stage)
    }
}
```

### 4B. ContentSheet model

**File: `StudioKit/Sources/StudioKit/Models/ContentSheet.swift`**

```swift
import SwiftData
import Foundation

@Model
public final class ContentSheet {
    @Attribute(.unique)
    public var id: String

    public var contentType: String
    public var contentSlug: String
    public var order: Int
    public var title: String
    public var body: String
    public var isMaterial: Bool
    public var status: String?
    public var wordCount: Int
    public var wordCountTarget: Int?
    public var createdAt: Date
    public var updatedAt: Date
    public var isDirty: Bool = false

    public var contentItem: ContentItem?

    public init(
        id: String,
        contentType: String,
        contentSlug: String,
        order: Int = 0,
        title: String = "",
        body: String = "",
        isMaterial: Bool = false,
        status: String? = nil,
        wordCount: Int = 0,
        wordCountTarget: Int? = nil,
        createdAt: Date = Date(),
        updatedAt: Date = Date()
    ) {
        self.id = id
        self.contentType = contentType
        self.contentSlug = contentSlug
        self.order = order
        self.title = title
        self.body = body
        self.isMaterial = isMaterial
        self.status = status
        self.wordCount = wordCount
        self.wordCountTarget = wordCountTarget
        self.createdAt = createdAt
        self.updatedAt = updatedAt
    }

    public static func from(_ api: APISheet, contentType: String, contentSlug: String) -> ContentSheet {
        ContentSheet(
            id: api.id,
            contentType: api.contentType ?? contentType,
            contentSlug: api.contentSlug ?? contentSlug,
            order: api.order ?? 0,
            title: api.title,
            body: api.body ?? "",
            isMaterial: api.isMaterial ?? false,
            status: api.status,
            wordCount: api.wordCount ?? 0,
            wordCountTarget: api.wordCountTarget,
            createdAt: api.createdAt.flatMap { ISO8601DateFormatter().date(from: $0) } ?? Date(),
            updatedAt: api.updatedAt.flatMap { ISO8601DateFormatter().date(from: $0) } ?? Date()
        )
    }
}
```

### 4C. SyncState model

**File: `StudioKit/Sources/StudioKit/Models/SyncState.swift`**

```swift
import SwiftData
import Foundation

/// Bookkeeping model for sync timestamps and state.
@Model
public final class SyncState {
    @Attribute(.unique)
    public var key: String

    public var value: String
    public var updatedAt: Date

    public init(key: String, value: String, updatedAt: Date = Date()) {
        self.key = key
        self.value = value
        self.updatedAt = updatedAt
    }
}
```

### 4D. Package.swift

**File: `StudioKit/Package.swift`**

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "StudioKit",
    platforms: [
        .macOS(.v15),
        .iOS(.v18),
    ],
    products: [
        .library(name: "StudioKit", targets: ["StudioKit"]),
    ],
    targets: [
        .target(name: "StudioKit"),
    ]
)
```

### Verification

```bash
cd StudioKit && swift build
```

All four batches should compile. Zero dependencies beyond Foundation and SwiftData.

---

## Execution Order

1. **Batch 1**: ContentType + Stage enums. `swift build` gate.
2. **Batch 2**: APITypes.swift (all Codable structs). `swift build` gate.
3. **Batch 3**: StudioAPIError, JSONCoders, StudioAPIClient, IndexAPIClient. `swift build` gate.
4. **Batch 4**: SwiftData models (ContentItem, ContentSheet, SyncState) + Package.swift. `swift build` gate.

---

## What This Does NOT Include (Next Spec)

- SyncEngine (push/pull orchestration)
- EditorBridge (WKWebView communication)
- Any SwiftUI views (MainWindow, Sidebar, EditorHost)
- Editor bundle extraction from travisgilbert.me
- App targets (StudioMac, StudioiOS)

Those are Month 1 Weeks 3 and 4, and Month 2. Separate specs.

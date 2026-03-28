# Studio

Native macOS and iOS writing app with research integration, built for ADHD brains.

## Architecture

SwiftUI native shell + WKWebView editor core. The existing Tiptap editor (17 custom
extensions) runs inside a WebView. Everything else is native: sidebar, settings,
Quick Capture, menus, notifications, offline sync.

Three clients share one Django API backend:
- **This app** (macOS + iOS, SwiftUI + WKWebView)
- **Web** (travisgilbert.me/studio, Next.js + Tiptap)
- **API** (Django on Railway, source of truth)

## Key Documents

| File | Purpose |
|------|---------|
| `CLAUDE.md` | Project context for Claude Code sessions |
| `docs/2026-03-27-studio-native-app-architecture.md` | Full architecture spec |
| `STUDIO-NATIVE-TODO.md` | Sequenced task list |

## Status

Phase 0: Learning Swift, finishing web polish.
Phase 1 target: macOS app (4 months).
Phase 2 target: iOS app (3 months after macOS ships).

## Related Repos

| Repo | Purpose |
|------|---------|
| `Travis-Gilbert/travisgilbert.me` | Web frontend (Next.js, Tiptap editor source) |
| `Travis-Gilbert/Index-API` | Theseus engine backend (Django, pgvector, ONNX) |

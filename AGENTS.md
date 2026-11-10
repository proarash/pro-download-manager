# Agent Rules — pro-download-manager

## Project
Desktop download manager in **Go** with a **Fyne** UI, driving **aria2c** (via JSON-RPC/WebSocket), **yt-dlp**, and **ffmpeg** as external binaries. No third-party database. See `download_manager_development_roadmap.md` for architecture and roadmap.

## Conventions
- Go code must be `gofmt`/`go vet` clean; run `go build ./...` and `go test ./...` before finishing a change.
- Keep UI (Fyne) code separate from engine/control logic; never block the UI goroutine.
- Talk to aria2 over its RPC interface, not by parsing CLI output.
- Pass `context.Context` for cancellable work; always handle errors explicitly.
- No code comments.
- No third-party packages; use the standard library. Always ask before installing any third-party package.
- Keep changes small and focused; don't commit anything.

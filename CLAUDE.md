# CLAUDE.md - WeFlow Development Guide

## Project Overview

WeFlow is a fully local WeChat chat history viewer, analyzer, and exporter. It is a cross-platform desktop application (Windows, macOS, Linux) built with Electron + React that runs completely offline. No data is sent to external servers.

**Key capabilities**: real-time chat viewing, media decryption (images/videos/live photos), chat analytics, annual/dual reports, data export (JSON/HTML/TXT/Excel/CSV/PostgreSQL/ChatLab), Moments (朋友圈) support, HTTP API (port 5031), desktop notifications, voice transcription.

## Tech Stack

- **Frontend**: React 19, TypeScript 5.6, Zustand (state), React Router 7, ECharts (charts), SCSS
- **Desktop**: Electron 39, electron-builder (packaging), electron-updater (auto-update)
- **Build**: Vite 6, TypeScript strict mode
- **Native**: koffi (FFI bindings), better-sqlite3, WCDB native libraries, sherpa-onnx-node (speech recognition)
- **Media**: ffmpeg-static, sharp (image processing), silk-wasm (audio codec), jieba-wasm (Chinese text segmentation)
- **Export**: ExcelJS, JSZip

## Repository Structure

```
WeFlow/
├── src/                    # React frontend
│   ├── pages/              # Route page components (ChatPage, AnalyticsPage, ExportPage, etc.)
│   ├── components/         # Reusable UI components (Export/, Sns/ subdirs)
│   ├── stores/             # Zustand state stores (appStore, chatStore, themeStore, etc.)
│   ├── services/           # Frontend service layer (config, IPC, cloud control)
│   ├── types/              # TypeScript interfaces and type definitions
│   ├── styles/             # Global SCSS stylesheets
│   ├── utils/              # Frontend utilities
│   ├── App.tsx             # Root component with routing
│   └── main.tsx            # React entry point
├── electron/               # Electron main process
│   ├── main.ts             # Main process entry (IPC handlers, window management)
│   ├── preload.ts          # Context bridge API (renderer ↔ main IPC)
│   ├── services/           # Backend services (25+ files)
│   │   ├── chatService.ts      # Chat message operations
│   │   ├── wcdbService.ts      # WCDB database interface
│   │   ├── wcdbCore.ts         # WCDB core logic
│   │   ├── exportService.ts    # Chat export (6 formats)
│   │   ├── analyticsService.ts # Analytics calculations
│   │   ├── keyService.ts       # Encryption key retrieval (Windows)
│   │   ├── keyServiceMac.ts    # Key retrieval (macOS)
│   │   ├── keyServiceLinux.ts  # Key retrieval (Linux)
│   │   ├── imageDecryptService.ts  # Image decryption
│   │   ├── snsService.ts       # Moments/朋友圈
│   │   ├── httpService.ts      # HTTP API server
│   │   └── ...                  # voiceTranscribe, config, messagePush, etc.
│   ├── *Worker.ts          # Worker threads (6): export, transcribe, wcdb, reports, imageSearch
│   ├── utils/              # Backend utilities (LRUCache)
│   └── windows/            # Additional window definitions
├── resources/              # Native binaries (DLLs, dylibs, .so files)
├── public/                 # Static assets
├── docs/                   # Documentation (HTTP-API.md)
├── .github/workflows/      # CI/CD (release.yml, issue-auto-assign.yml)
├── vite.config.ts          # Vite + Electron build configuration
├── tsconfig.json           # Frontend TypeScript config
├── tsconfig.node.json      # Electron TypeScript config
└── package.json            # Dependencies and scripts
```

## Development Commands

```bash
npm install          # Install dependencies (uses China mirror via .npmrc)
npm run dev          # Start Vite dev server (port 3000) + Electron
npm run typecheck    # TypeScript type checking only (tsc --noEmit)
npm run build        # Full production build: tsc → vite build → electron-builder
npm run preview      # Preview production build
```

## Architecture

### IPC Communication Pattern

Frontend ↔ Backend communication uses Electron's Context Bridge pattern:
1. `electron/preload.ts` exposes a typed API to the renderer process
2. `src/types/electron.d.ts` defines the complete IPC interface (window.electron)
3. `electron/main.ts` registers IPC handlers that delegate to services

### Worker Threads

Heavy operations run in dedicated Node.js worker threads to avoid blocking the main process:
- `exportWorker.ts` - Chat export processing
- `transcribeWorker.ts` - Voice-to-text conversion
- `wcdbWorker.ts` - WCDB database access
- `annualReportWorker.ts` - Annual report generation
- `dualReportWorker.ts` - Dual relationship reports
- `imageSearchWorker.ts` - Image search

### State Management

Zustand stores in `src/stores/`:
- `appStore.ts` - App-level state (sessions, contacts, selected chat)
- `chatStore.ts` - Chat messages and message state
- `themeStore.ts` - Theme/appearance settings
- `analyticsStore.ts` - Cached analytics data
- `imageStore.ts` - Image loading state
- `batchTranscribeStore.ts` - Voice transcription queue
- `batchImageDecryptStore.ts` - Image decryption batch state
- `contactTypeCountsStore.ts` - Contact type counts cache

### Native Library Integration

Platform-specific native libraries are loaded via koffi (FFI):
- Windows: `WCDB.dll`, key extraction via `wx_key`
- macOS: `libwcdb_api.dylib`, keychain access
- Linux: `libwcdb_api.so`

## Code Conventions

### Naming

- **React components**: PascalCase (`ChatPage`, `Avatar`, `DateRangePicker`)
- **Services**: camelCase with "Service" suffix (`chatService`, `analyticsService`)
- **Stores**: camelCase with "Store" suffix (`appStore`, `chatStore`)
- **Types/Interfaces**: PascalCase (`ChatSession`, `Message`, `Contact`)
- **Worker files**: camelCase with "Worker" suffix (`exportWorker`, `wcdbWorker`)

### File Organization

- Pages are self-contained route components in `src/pages/`
- Reusable components go in `src/components/`, grouped by feature subdirectory when related
- Backend services each handle a specific domain in `electron/services/`
- Type definitions live in `src/types/` (frontend) and `electron/types/` (backend)

### Patterns

- Functional React components with hooks (no class components)
- Zustand for state management with immutable updates
- Error boundaries for crash recovery (`ErrorBoundary.tsx`)
- Route guards for authentication/authorization (`RouteGuard.tsx`)
- Virtual scrolling for large lists (react-virtuoso)
- LRU caching for frequently accessed data

### Language

- Code comments are primarily in **Chinese** with some English
- Commit messages are primarily in **Chinese**, loosely following conventional commits (`feat:`, `fix:`, `chore:`)
- UI text is in Chinese (this is a Chinese-market application)

## Build & Release

### CI/CD (GitHub Actions)

**release.yml** triggers on version tags (`v*`):
1. Builds on macOS 14, Ubuntu latest, Windows latest
2. Uses Node.js 24
3. Syncs package version with git tag
4. Runs `tsc` type check + `vite build`
5. Packages with `electron-builder`
6. Publishes to GitHub Releases

**Supported platforms**:
- Windows x64, arm64 (NSIS installer)
- macOS arm64 (DMG, unsigned)
- Linux x64 (AppImage, tar.gz)

### Build Configuration

- Vite dev server: port 3000 (auto-fallback)
- Path alias: `@/*` → `./src/*`
- External (not bundled): better-sqlite3, koffi, fsevents, exceljs, sherpa-onnx-node, sudo-prompt
- 6 worker entry points configured in vite.config.ts

## Key Files (by importance)

| File | Size | Purpose |
|------|------|---------|
| `electron/main.ts` | 92KB | Main process, IPC handlers, window management |
| `electron/preload.ts` | 23KB | Context bridge API definitions |
| `electron/services/chatService.ts` | 263KB | Core chat message operations |
| `electron/services/exportService.ts` | 334KB | Export logic for all formats |
| `electron/services/wcdbCore.ts` | 148KB | Low-level WCDB database operations |
| `electron/services/snsService.ts` | 105KB | Moments/朋友圈 functionality |
| `src/pages/ChatPage.tsx` | 388KB | Main chat UI (largest frontend file) |
| `src/services/config.ts` | 53KB | Frontend configuration management |
| `src/types/electron.d.ts` | 33KB | Complete IPC type definitions |

## Testing

No formal test framework is currently configured. Type checking (`npm run typecheck`) is the primary automated quality gate.

## Important Notes

- **Privacy-first**: All data stays local. Never add external data transmission.
- **Multi-platform**: Changes must work on Windows, macOS, and Linux. Key and WCDB services have platform-specific implementations.
- **Large files**: Several core files are very large (100KB+). When modifying them, read only the relevant sections.
- **Native dependencies**: Some dependencies require platform-specific binaries in `resources/`. These are not installed via npm.
- **npm registry**: Uses Chinese mirror (npmmirror.com) configured in `.npmrc`.
- **License**: CC BY-NC-SA 4.0 (non-commercial).

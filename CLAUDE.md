# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## About this project

This is a fork of [reveal.js](https://revealjs.com) — an open source HTML presentation framework — being used to build a 5-minute AI Hack Week presentation. It also contains `@revealjs/react`, a React wrapper for the core library, located in the `react/` subdirectory.

### Presentation subject: fvrcli

The presentation is about `fvr`, a Go CLI + Claude Code plugin + Slackbot built to help debug Favor menu issues. The project lives at `~/Projects/fvrcli/`. Key facts for building slides:

- **Commands:** `fvr auth`, `fvr merchant` (get/search/locations/accounts), `fvr location` (get/debug), `fvr menu` (get/list/validate/version/cache-status/cache-clear), `fvr plugin install`
- **Stack:** Go + Cobra + Wire DI; minimal dependencies; compiles to a single binary
- **SKILL.md** is embedded in the binary via Go static asset embedding; `fvr plugin install` registers it with Claude Code
- **Test coverage:** 78.9% overall; 100% on `menu` and `report` packages; 95%+ on `api`, `config`, `cmd`
- **Hack-week scope (all complete):** CLI → Claude Code plugin → Slackbot
- **Future state:** more data sources (alerts, promos, ordering), TUI (Bubbletea design drafted in `docs/tui-design.md`)
- **Audience:** engineers and upper management; framed as a developer debugging tool

## Commands

### Core library

```bash
npm start          # Dev server on port 8000 (override with --port=XXXX)
npm run build      # Full build: TypeScript + all plugins + styles → dist/
npm run build:core # Build core only (no plugins)
npm test           # Run QUnit tests via Puppeteer against a Vite server
```

### React wrapper (`react/`)

```bash
npm test --prefix react           # Run Vitest unit tests
npm run test:watch --prefix react # Watch mode
npm run build --prefix react      # Build to react/dist/
npm run demo --prefix react       # Start demo app dev server
```

## Architecture

### Core library (`js/`)

`js/reveal.js` exports the main `Deck` factory function. `js/index.ts` re-exports it and adds a legacy singleton API (pre-4.0 compatibility) that lets callers call `Reveal.initialize()` without holding an instance reference.

All runtime behavior is split into **controllers** (`js/controllers/`) — each owns one concern:

| Controller | Responsibility |
|---|---|
| `scrollview.js` | Scroll mode: renders slides as a linear scrollable page |
| `autoanimate.js` | Auto-Animate transitions between slides |
| `backgrounds.js` | Slide backgrounds (color, image, video, iframe) |
| `fragments.js` | Step-by-step fragment reveals |
| `overview.js` | Bird's-eye overview mode |
| `keyboard.js` | Keyboard navigation and shortcuts |
| `location.js` | URL hash / history sync |
| `touch.js` | Touch/swipe navigation |
| `overlay.js` | Slide overlay UI |
| `plugins.js` | Plugin registration and lifecycle |

`js/config.ts` defines the full typed config schema. `js/utils/` holds stateless helpers (DOM utilities, color math, device detection, script/CSS loading).

The build produces both ESM (`dist/reveal.mjs`) and UMD (`dist/reveal.js`) formats. Styles are compiled separately via `vite.config.styles.ts`.

### Plugins (`plugin/`)

Each plugin (`highlight`, `markdown`, `math`, `notes`, `search`, `zoom`) has its own `vite.config.ts` and builds independently to `dist/plugin/`. They are loaded at runtime by the `Plugins` controller and receive the `Reveal` instance on init.

### Themes (`css/theme/`)

SCSS themes extend base templates in `css/theme/template/`. Build new themes by creating a `.scss` file that imports `theme/template/theme.scss`.

### React wrapper (`react/`)

The React wrapper is a separate package (`@revealjs/react`) that peer-depends on `reveal.js`. It is tested with Vitest and built independently.

Key architecture constraints from `react/AGENTS.md`:

- `Deck` creates exactly one `Reveal` instance on mount and destroys it on unmount. It must be safe under React StrictMode.
- `Reveal.sync()` is expensive — only call it when slide structure changes (slides added/removed/reordered). Ordinary content updates inside existing slides must not trigger a full sync.
- Config is shallow-compared; don't call `configure()` if shallow values are unchanged, and don't call `sync()` immediately after `configure()`.
- Component tests live colocated with their source as `*.test.tsx`.

### Test setup

Core tests are QUnit suites in `test/*.html`, run via Puppeteer against a Vite dev server (`scripts/test.js`). Each HTML file is a self-contained test page.

React tests use Vitest + `@testing-library/react` + jsdom, configured in `react/vitest.config.ts`.

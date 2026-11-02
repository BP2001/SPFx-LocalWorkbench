# Architecture

How the extension is put together, for anyone changing it. User-facing behaviour is in [README.md](../README.md);
the API proxy is described in depth in [PROXY.md](../PROXY.md).

## The three runtimes

```
┌──────────────────────── VS Code extension host (Node) ───────────────────────┐
│ src/extension.ts            commands, status bar, activation                │
│ src/workbench/WorkbenchPanel.ts   creates the webview, relays messages       │
│ src/workbench/config/       VS Code settings                                 │
│ src/workbench/proxy/        ApiProxyService, mock rules                      │
│ src/workbench/storybook/    Storybook server and story generation           │
└───────────────▲──────────────────────────────────────────────┬───────────────┘
                │ postMessage (proxy requests, settings, state)  │ HTML + config
┌───────────────┴──────────────────────────────────────────────▼───────────────┐
│ Webview (browser) – webview/src                                              │
│ main.tsx            boot, message handling                                   │
│ WorkbenchRuntime.ts AMD loader shim, loads bundles from serveUrl, lifecycle  │
│ WebPartManager.ts   creates web part instances with a mock SPFx context      │
│ mocks/              SpfxContext, ThemeProvider                               │
│ proxy/              VsCodeProxyTransport                                     │
│ components/         App, WorkbenchCanvas, toolbars, panes                    │
└──────────────────────────────▲───────────────────────────────────────────────┘
                               │ imports
┌──────────────────────────────┴───────────────────────────────────────────────┐
│ packages/shared – used by the webview AND the Storybook addon               │
│ mocks/ (page context, platform modules), proxy/ (http clients, interceptors),│
│ loaders/ (StringsLoader), components/ (PropertyPanePanel, theme picker)      │
└──────────────────────────────────────────────────────────────────────────────┘
packages/storybook-addon-spfx – Storybook addon, must stay usable without VS Code
```

Your SPFx project's dev server (`heft start`) runs separately and serves the bundles the webview loads from
`spfxLocalWorkbench.serveUrl` (default `https://localhost:4321`).

## Request flow

1. A web part calls `spHttpClient`, `httpClient`, `aadHttpClient`, `msGraphClientFactory` or `fetch`.
2. The proxy clients in `packages/shared/src/proxy` (and the fetch interceptor) serialize the request and send it
   to the extension host through `VsCodeProxyTransport`.
3. `ApiProxyService` matches it against the mock rules (`.spfx-workbench/api-mocks.json`). A match returns the mock.
4. Without a match: in `mock` mode the fallback status; in passthrough modes a real request.
5. The response goes back to the webview.

## Settings sources

VS Code settings, then the defaults in `package.json`.

## Builds and outputs

| Command | Output |
|---|---|
| `npm run compile` | packages, type-check, lint, CSS module types, `dist/extension.js`, `dist/webview/` |
| `npm run watch` | the same, incrementally (used by F5) |
| `npm test` | Vitest unit tests (`*.test.ts` next to the code) |
| `npx vsce package` | a `.vsix` (runs `vscode:prepublish` → production build) |

## Where to start for common changes

| Change | Start in |
|---|---|
| A missing SPFx context API | `packages/shared/src/mocks/`, then `webview/src/mocks/SpfxContext.ts` |
| A request handled wrongly | `packages/shared/src/proxy/`, then `src/workbench/proxy/` |
| A new setting | `package.json` + `package.nls.json` (and `package.nls.fr.json`), `src/workbench/config/WorkbenchConfig.ts`, `WorkbenchPanel.ts` (send to webview), `webview/src/types` |
| Web part loading, strings | `webview/src/WorkbenchRuntime.ts`, `packages/shared/src/loaders/` |

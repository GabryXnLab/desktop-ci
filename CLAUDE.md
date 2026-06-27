# CLAUDE.md — desktop-ci

Reusable GitHub Actions workflow per build dell'app **desktop Tauri** (Rust + Vite/React).
Analogo desktop di `expo-ci` (mobile Expo). Vedi `README.md` per l'overview d'uso.

## ⚠️ Impatto delle modifiche: @main è live per tutti

I progetti richiamano `tauri-build.yml` come reusable via `uses: GabryXnLab/desktop-ci/...@main`.
**Ogni push su `main` ha effetto immediato su TUTTI i progetti** al prossimo run. Nessun pinning
per SHA lato consumatori → una modifica rotta qui rompe tutti i progetti.
→ Input **retrocompatibili** (nuovi con `default`, mai rimuovere/rinominare). Validare lo YAML.

## Architettura: build nativa per OS, niente cross-compilazione

Tauri compila per la piattaforma del runner. `platforms` (CSV) → job `setup` che genera una
**matrix JSON** (`{os, runner, bundle_glob}`), poi `build` esegue un job per riga sul runner
nativo (`runs-on: matrix.runner`).

- **linux** → `.deb`/`.AppImage`/`.rpm`. Serve GTK/WebKit (`libwebkit2gtk-4.1-dev`,
  `libgtk-3-dev`, `libsoup-3.0-dev`, `librsvg2-dev`). Su `nexus-core` già presenti
  (`install_system_deps=false`). Build aarch64 sul runner ARM64.
- **windows** → `.msi`/NSIS `.exe`. Serve toolchain MSVC + WebView2: **runner Windows**.
- **macos** → `.dmg`/`.app`. Serve **host macOS** + Xcode CLT: NON compilabile da Linux.

### `runner_type` + validazione compatibilità (job `setup`)

`runner_type`: `self-hosted` (default) | `github`.
- **github** → linux→`ubuntu-latest`, windows→`windows-latest`, macos→`macos-latest` (apt Linux automatico).
- **self-hosted** → l'ambiente è Linux: solo `linux` è compatibile (runner `selfhosted_linux_runner`,
  default `nexus-core`). Se si chiede **windows/macos** senza override, il job `setup`
  **esce con `exit 1`** (errore esplicito) → la dipendenza `needs: setup` blocca `build`:
  nessuna build parte. È il modo standard di "non avviare il workflow" su combinazioni invalide
  (workflow_dispatch non valida le combinazioni di input prima del dispatch).
- Override `linux_runner`/`windows_runner`/`macos_runner` (JSON array, default `''`) **scavalcano**
  il default del `runner_type` (es. un runner self-hosted Windows/macOS dedicato → bypassa l'errore).

La matrix (`{os, runner, bundle_glob, apt}`) è generata in `setup` e consumata da `build`
(`runs-on: matrix.runner`). `setup` gira su `ubuntu-latest` (sempre disponibile, anche se il
self-hosted è offline/incompatibile).

## Convenzioni (allineate a expo-ci)

- Il progetto desktop è **npm**-based (`package-lock.json`) → `npm ci`. (Il mobile usa pnpm; qui no.)
- `tauri build` lancia da solo `beforeBuildCommand` (`npm run build` = tsc + vite build).
- Secret passati **per nome** dai wrapper (non `inherit`, inaffidabile cross-repo).
- `SUBMODULES_TOKEN`: PAT org-wide read-only Contents per submodule privati cross-repo
  (il `GITHUB_TOKEN` di default è scoped al solo repo in build). Fallback a `github.token`.
- Notifica Telegram **best-effort** (non fa fallire il job); Local Bot API Server con fallback
  cloud (50 MB), stesso schema di expo-ci.
- Il frontend desktop usa `@shared` → `../shared` del repo principale: serve il checkout
  dell'intero repo consumatore (è il default; `has_submodules` se ci sono submoduli privati).

## Estensioni future

- Firma/notarization macOS, code-signing Windows: aggiungere secret + step qui (non nei wrapper).
- Pubblicazione GitHub Release + `latest.json` per l'updater Tauri: nuovo input opzionale
  `publish_release`, da implementare qui centralmente.

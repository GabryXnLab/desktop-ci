# CLAUDE.md — desktop-ci

Reusable GitHub Actions workflow per build dell'app **desktop Tauri** (Rust + Vite/React).
Analogo desktop di `expo-ci` (mobile Expo). Vedi `README.md` per l'overview d'uso.

## ⚠️ Impatto delle modifiche: @main è live per tutti

I progetti richiamano `tauri-build.yml` come reusable via `uses: GabryXnLab/desktop-ci/...@main`.
**Ogni push su `main` ha effetto immediato su TUTTI i progetti** al prossimo run. Nessun pinning
per SHA lato consumatori → una modifica rotta qui rompe tutti i progetti.
→ Input **retrocompatibili** (nuovi con `default`, mai rimuovere/rinominare). Validare lo YAML.

## Architettura: build nativa per OS *e arch*, niente cross-compilazione

Tauri compila per la piattaforma del runner. `platforms` (CSV di target) → job `setup` che genera
una **matrix JSON** (`{os, arch, runner, bundle_glob, apt}`), poi `build` esegue un job per riga
sul runner nativo (`runs-on: matrix.runner`). Target: `linux-x64`, `linux-arm64`, `windows`,
`macos` (alias: `linux`→`linux-arm64`, `x64`/`x86_64`→`linux-x64`, `win`→`windows`, `mac`/`darwin`→`macos`).

- **linux-x64 / linux-arm64** → `.deb`/`.AppImage`/`.rpm`. Serve GTK/WebKit e, per il tray,
  AppIndicator (`libwebkit2gtk-4.1-dev`, `libgtk-3-dev`, `libsoup-3.0-dev`,
  `libayatana-appindicator3-dev`, `librsvg2-dev`). `matrix.apt` le installa sui runner
  GitHub-hosted e sui self-hosted freschi.
- **windows** → `.msi`/NSIS `.exe` (x86_64). Serve toolchain MSVC + WebView2: **runner Windows**.
- **macos** → `.dmg`/`.app` (Apple Silicon). Serve **host macOS** + Xcode CLT: NON compilabile da Linux.

### `runner_type` + validazione compatibilità (job `setup`)

`runner_type`: `self-hosted` (default) | `github`.
- **github** → linux-x64→`ubuntu-latest`, linux-arm64→`ubuntu-24.04-arm`, windows→`windows-latest`,
  macos→`macos-latest` (apt Linux automatico).
- **self-hosted** → l'ambiente di default è **Linux ARM64**: solo `linux-arm64` è compatibile
  (runner `selfhosted_linux_runner`, default `nexus-core`). Se si chiede **linux-x64/windows/macos**
  senza override, il job `setup` **esce con `exit 1`** (errore esplicito) → la dipendenza
  `needs: setup` blocca `build`: nessuna build parte. È il modo standard di "non avviare il
  workflow" su combinazioni invalide (workflow_dispatch non valida gli input prima del dispatch).
- Override `linux_x64_runner`/`linux_arm64_runner`/`windows_runner`/`macos_runner` (JSON array,
  default `''`) **scavalcano** il default del `runner_type` (es. un runner self-hosted x86_64 o
  Windows/macOS dedicato → bypassa l'errore).

La matrix (`{os, runner, bundle_glob, apt}`) è generata in `setup` e consumata da `build`
(`runs-on: matrix.runner`). `setup` gira su `ubuntu-latest` (sempre disponibile, anche se il
self-hosted è offline/incompatibile).

## Velocità sul self-hosted (con `GabryXnLab/build-kit`)

- `max_workers` (`auto` | `2` | `4`) → `CARGO_BUILD_JOBS`, stesso input e significato di
  flutter-ci ed expo-ci; lo applica `build-kit/setup` (leggere il suo `CLAUDE.md`).
- **La cartella target di cargo sta fuori dal checkout**, in `~/ci/cache/cargo-target/<repo>`,
  e il checkout non pulisce: prima ogni build ricompilava tutto, e siccome mobile e desktop
  dello stesso repo condividono la cartella di lavoro, il `git clean` dell'uno cancellava gli
  intermedi dell'altro. I bundle della versione precedente si tolgono prima della build
  (altrimenti partirebbero su Telegram). I percorsi degli artefatti passano dall'output
  `steps.target.outputs.dir`.
- **sccache** (`RUSTC_WRAPPER`, `~/.cache/sccache`, 10 GB) è condiviso fra tutti i progetti
  Rust della macchina: le dipendenze da `~/.cargo/registry` si compilano una volta.
  `clear_cache` cancella la target del repo e mette sccache in `RECACHE`.

## Convenzioni (allineate a expo-ci)

- Package manager dal lockfile di `project_dir`: `pnpm-lock.yaml` → pnpm (versione da
  `packageManager`), altrimenti `npm ci`. Riftgate desktop è pnpm: con `npm ci` la build
  falliva subito.
- `tauri build` lancia da solo `beforeBuildCommand` (`npm run build` = tsc + vite build).
- Secret passati **per nome** dai wrapper (non `inherit`, inaffidabile cross-repo).
- `SUBMODULES_TOKEN`: PAT org-wide read-only Contents per submodule privati cross-repo
  (il `GITHUB_TOKEN` di default è scoped al solo repo in build). Fallback a `github.token`.
- Una build senza `TAURI_SIGNING_PRIVATE_KEY` disabilita via `--config` soltanto gli artefatti
  updater: gli installer ordinari vengono comunque prodotti. `publish_release=true` senza
  chiave fallisce invece prima della build, perché una release updater non firmata è invalida.
- Notifica Telegram tramite l'azione `GabryXnLab/build-kit/notify@main`, copia pubblica di
  quella di `ci-bot` (che resta privato insieme al bot `@BobCI_bot`), best-effort: bundle come
  file se il job riesce, altrimenti messaggio con il pulsante «🔁 Rilancia». Deve essere
  pubblica perché questo repo è pubblico e un repo pubblico non può usare azioni di repo
  privati. Token e chat arrivano dai secret del chiamante; senza, non parte niente.
- Gli step di notifica sono **tre**: `setup` (solo fallimenti: senza matrice il resto viene
  saltato), un messaggio per target in `build`, la release updater in `publish`. L'azione gira
  dalla radice del workspace, non da `project_dir`: i pattern dei file lo includono.
- Il frontend desktop usa `@shared` → `../shared` del repo principale: serve il checkout
  dell'intero repo consumatore (è il default; `has_submodules` se ci sono submoduli privati).

## Pubblicazione updater (job `publish`, opzionale)

`publish_release: true` (default `false`) attiva, dopo `build`, un job `publish` che pubblica
l'auto-update Tauri:

- Richiede `needs: build` con successo su **tutta** la matrix (fail-fast:false → un target
  fallito fa fallire `build` nel complesso → `publish` viene saltato, niente release parziali).
- Scarica tutti gli artifact della matrix (`actions/download-artifact@v4` senza `name`), abbina
  ogni `.sig` al binario corrispondente (stesso path meno `.sig`) — deduzione generica, non
  nomi file hardcoded, perché Tauri li varia leggermente per OS/versione.
- Deduce la piattaforma (chiave `latest.json`) dal nome cartella artifact
  (`<app>-desktop-<os>-<arch>-<sha>`) mappandola sui nomi attesi da Tauri v2
  (`linux-x86_64`, `linux-aarch64`, `windows-x86_64`, `darwin-x86_64`, `darwin-aarch64`).
- Legge la versione da `<project_dir>/src-tauri/tauri.conf.json` (`.version`), usa `vX.Y.Z` come
  tag release.
- Costruisce `latest.json` (schema Tauri v2: `version`/`notes`/`pub_date`/`platforms.{...}`) e
  pubblica (`gh release create`, o `upload --clobber` se il tag esiste già — idempotente sui
  re-run) su `releases_repo` con il secret `RELEASES_TOKEN`.

**Perché un repo `releases_repo` separato ha senso quando il sorgente è privato:** gli asset di
una GitHub Release privata non sono scaricabili in anonimo, ma l'updater Tauri fa una richiesta
HTTP senza autenticazione — un repo pubblico dedicato ai soli binari (nessun codice) risolve
senza bisogno di infrastruttura server aggiuntiva. `RELEASES_TOKEN` è un PAT dedicato
(Contents:Read&Write solo su quel repo), diverso da `SUBMODULES_TOKEN` (quello è read-only).

## Estensioni future

- Firma/notarization macOS, code-signing Windows: aggiungere secret + step qui (non nei wrapper).

# desktop-ci

Reusable GitHub Actions workflow per il **build dell'app desktop Tauri** (Rust + Vite/React).
Centralizza la CI/CD desktop condivisa tra più progetti — l'analogo desktop di
[`expo-ci`](https://github.com/GabryXnLab/expo-ci) (che fa lo stesso per il mobile Expo).

I progetti consumatori contengono solo un **thin wrapper** che richiama il reusable:

```yaml
jobs:
  desktop:
    uses: GabryXnLab/desktop-ci/.github/workflows/tauri-build.yml@main
    with:
      app_name: WarpMobile
      platforms: linux,windows,macos   # scegli quali buildare
      project_dir: desktop
      has_submodules: true
    secrets:
      SUBMODULES_TOKEN: ${{ secrets.SUBMODULES_TOKEN }}
      TELEGRAM_BOT_TOKEN: ${{ secrets.TELEGRAM_BOT_TOKEN }}
      TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}
```

## `tauri-build.yml`

Builda l'app Tauri **in modo nativo per ogni piattaforma selezionata**. Non c'è
cross-compilazione: ogni piattaforma gira sul runner della sua label.

| Piattaforma | Bundle prodotti        | Requisiti runner                                  |
|-------------|------------------------|---------------------------------------------------|
| `linux`     | `.deb` `.AppImage` `.rpm` | Librerie GTK/WebKit (`libwebkit2gtk-4.1-dev` …) |
| `windows`   | `.msi` `.exe` (NSIS)   | Toolchain MSVC + WebView2                          |
| `macos`     | `.dmg` `.app`          | **Host macOS reale** + Xcode CLT                  |

> ⚠️ **macOS richiede un host macOS**: non è compilabile su un server Linux.
> Allo stesso modo Windows richiede l'ambiente Windows. Registra runner self-hosted
> con le label appropriate, oppure ripunta gli input `*_runner` su runner GitHub-hosted.

### Input principali

| Input                 | Default                         | Note |
|-----------------------|---------------------------------|------|
| `app_name`            | — (richiesto)                   | Nome per artifact e notifiche |
| `platforms`           | — (richiesto)                   | CSV: `linux,windows,macos` (1, 2 o 3) |
| `project_dir`         | `desktop`                       | Cartella con `package.json` + `src-tauri/` |
| `build_debug`         | `false`                         | `true` = build debug |
| `has_submodules`      | `false`                         | `true` → checkout `--recursive` |
| `install_system_deps` | `false`                         | `true` → apt delle libs GTK/WebKit (Linux) |
| `setup_rust`          | `true`                          | Assicura Rust stable via rustup |
| `linux_runner`        | `["self-hosted","nexus-core"]`  | `runs-on` come JSON array |
| `windows_runner`      | `["self-hosted","windows"]`     | idem |
| `macos_runner`        | `["self-hosted","macos"]`       | idem |

### Secret (tutti opzionali)

- `SUBMODULES_TOKEN` — PAT read-only Contents per submodule privati cross-repo.
- `TELEGRAM_BOT_TOKEN` / `TELEGRAM_CHAT_ID` — notifica best-effort con l'installer in allegato.
- `TAURI_SIGNING_PRIVATE_KEY` / `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` — firma artefatti updater.

## ⚠️ `@main` è live per tutti

I consumatori referenziano `...@main`: ogni push qui ha effetto immediato sul prossimo
run di tutti i progetti. Mantenere gli input retrocompatibili e validare lo YAML prima del push.

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
      runner_type: github              # self-hosted | github
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
> Allo stesso modo Windows richiede l'ambiente Windows.

### Scelta del runner (`runner_type`)

| `runner_type` | Comportamento |
|---------------|---------------|
| `self-hosted` (default) | Tutte le piattaforme sui runner self-hosted. L'ambiente self-hosted è **Linux** → se selezioni **windows/macos** il job `setup` **fallisce con errore esplicito e nessuna build parte** (a meno di passare un override `*_runner` dedicato). |
| `github`      | Ogni piattaforma sul runner GitHub-hosted nativo: `linux→ubuntu-latest`, `windows→windows-latest`, `macos→macos-latest`. Su Linux le librerie GTK/WebKit sono installate automaticamente. |

Gli input `linux_runner`/`windows_runner`/`macos_runner` (JSON array) **forzano** il `runs-on`
di una piattaforma, scavalcando il default del `runner_type` (es. un runner self-hosted Windows).

### Input principali

| Input                 | Default                         | Note |
|-----------------------|---------------------------------|------|
| `app_name`            | — (richiesto)                   | Nome per artifact e notifiche |
| `platforms`           | — (richiesto)                   | CSV: `linux,windows,macos` (1, 2 o 3) |
| `runner_type`         | `self-hosted`                   | `self-hosted` \| `github` (vedi tabella sopra) |
| `project_dir`         | `desktop`                       | Cartella con `package.json` + `src-tauri/` |
| `build_debug`         | `false`                         | `true` = build debug |
| `has_submodules`      | `false`                         | `true` → checkout `--recursive` |
| `install_system_deps` | `false`                         | `true` → apt delle libs GTK/WebKit (Linux); su `github` Linux è automatico |
| `setup_rust`          | `true`                          | Assicura Rust stable via rustup |
| `selfhosted_linux_runner` | `["self-hosted","nexus-core"]` | `runs-on` Linux quando `runner_type=self-hosted` |
| `linux_runner` / `windows_runner` / `macos_runner` | `''` | Override `runs-on` (JSON array); scavalca il default del `runner_type` |

### Secret (tutti opzionali)

- `SUBMODULES_TOKEN` — PAT read-only Contents per submodule privati cross-repo.
- `TELEGRAM_BOT_TOKEN` / `TELEGRAM_CHAT_ID` — notifica best-effort con l'installer in allegato.
- `TAURI_SIGNING_PRIVATE_KEY` / `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` — firma artefatti updater.

## ⚠️ `@main` è live per tutti

I consumatori referenziano `...@main`: ogni push qui ha effetto immediato sul prossimo
run di tutti i progetti. Mantenere gli input retrocompatibili e validare lo YAML prima del push.

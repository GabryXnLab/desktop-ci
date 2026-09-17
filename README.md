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
      platforms: linux-x64,linux-arm64,windows,macos   # scegli quali buildare
      runner_type: github                              # self-hosted | github
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

| Target        | Bundle prodotti          | Requisiti runner                                  |
|---------------|--------------------------|---------------------------------------------------|
| `linux-x64`   | `.deb` `.AppImage` `.rpm` (x86_64) | Linux x86_64 + GTK/WebKit               |
| `linux-arm64` | `.deb` `.AppImage` `.rpm` (aarch64) | Linux ARM64 + GTK/WebKit               |
| `windows`     | `.msi` `.exe` (NSIS)     | Windows + MSVC + WebView2                          |
| `macos`       | `.dmg` `.app`            | **Host macOS reale** + Xcode CLT                  |

Alias accettati: `linux`→`linux-arm64`, `x64`/`x86_64`→`linux-x64`, `win`→`windows`, `mac`/`darwin`→`macos`.

> ⚠️ **macOS richiede un host macOS**: non è compilabile su un server Linux.
> Allo stesso modo Windows richiede l'ambiente Windows.

### Scelta del runner (`runner_type`)

| `runner_type` | Comportamento |
|---------------|---------------|
| `self-hosted` (default) | Tutto sui runner self-hosted. L'ambiente di default è **Linux ARM64** → solo `linux-arm64` è compatibile; **`linux-x64`/`windows`/`macos`** fanno **fallire `setup` con errore esplicito e nessuna build parte** (a meno di un override `*_runner` dedicato). |
| `github`      | Ogni target sul runner GitHub-hosted nativo: `linux-x64→ubuntu-latest`, `linux-arm64→ubuntu-24.04-arm`, `windows→windows-latest`, `macos→macos-latest`. Su Linux le librerie GTK/WebKit sono installate automaticamente. |

Gli input `linux_runner`/`windows_runner`/`macos_runner` (JSON array) **forzano** il `runs-on`
di una piattaforma, scavalcando il default del `runner_type` (es. un runner self-hosted Windows).

### Input principali

| Input                 | Default                         | Note |
|-----------------------|---------------------------------|------|
| `app_name`            | — (richiesto)                   | Nome per artifact e notifiche |
| `platforms`           | — (richiesto)                   | CSV: `linux-x64,linux-arm64,windows,macos` |
| `runner_type`         | `self-hosted`                   | `self-hosted` \| `github` (vedi tabella sopra) |
| `project_dir`         | `desktop`                       | Cartella con `package.json` + `src-tauri/` |
| `build_debug`         | `false`                         | `true` = build debug |
| `has_submodules`      | `false`                         | `true` → checkout `--recursive` |
| `install_system_deps` | `false`                         | `true` → apt delle libs GTK/WebKit (Linux); su `github` Linux è automatico |
| `setup_rust`          | `true`                          | Assicura Rust stable via rustup |
| `selfhosted_linux_runner` | `["self-hosted","nexus-core"]` | `runs-on` per `linux-arm64` self-hosted |
| `linux_x64_runner` / `linux_arm64_runner` / `windows_runner` / `macos_runner` | `''` | Override `runs-on` (JSON array); scavalca il default del `runner_type` |
| `telegram_topic_id`   | `''`                            | `message_thread_id` del topic in cui pubblicare le notifiche, se la chat è un supergruppo con i Topics. Vuoto = topic General |

### Secret (tutti opzionali)

- `SUBMODULES_TOKEN` — PAT read-only Contents per submodule privati cross-repo.
- `TELEGRAM_BOT_TOKEN` / `TELEGRAM_CHAT_ID` — notifica best-effort con l'installer in allegato.
  Il topic si sceglie con l'input `telegram_topic_id` (non è un secret): vale sia per la
  notifica dei bundle sia per quella della release updater.
- `TAURI_SIGNING_PRIVATE_KEY` / `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` — firma artefatti updater.
  Senza chiave il workflow produce comunque gli installer ordinari, disabilitando gli artefatti
  updater per quel build; `publish_release=true` richiede invece obbligatoriamente la chiave.

## ⚠️ `@main` è live per tutti

I consumatori referenziano `...@main`: ogni push qui ha effetto immediato sul prossimo
run di tutti i progetti. Mantenere gli input retrocompatibili e validare lo YAML prima del push.

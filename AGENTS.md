# AGENTS.md — resolve-flatpak

Guidance for AI agents (and contributors) working in this repository.

## What this repository is

`resolve-flatpak` packages **DaVinci Resolve** (and **DaVinci Resolve Studio**) as
a Flatpak for Linux, with a focus on atomic/immutable distros (Silverblue,
Kinoite, etc.) and Flatpak-first systems.

It does **not** redistribute any Blackmagic binaries or copyrighted material.
Instead it ships a **Qt/PySide6-based meta-installer** that, on first launch,
downloads the official DaVinci Resolve Linux installer directly from
`blackmagicdesign.com` and installs it into the user's Flatpak data directory
(`~/.var/app/com.blackmagic.Resolve/data`). This keeps the distributable Flatpak
small and legally clear, so it could be published on Flathub.

Because Flatpak's `extra-data` mechanism breaks on very large downloads (>8 GB),
the author moved away from it to this on-demand downloader approach.

## Repository layout

| Path | Purpose |
|------|---------|
| `installer/` | The Python meta-installer/launcher source (the real logic). |
| `com.blackmagic.Resolve.yaml` | Flatpak manifest for the **free** edition. |
| `com.blackmagic.ResolveStudio.yaml` | Flatpak manifest for the **Studio** edition. |
| `build-meta-flatpak.sh` | Helper script wrapping `flatpak-builder` (build / install / export). |
| `desktop/` | `.desktop` files for Resolve, RAW Player, RAW Speed Test, Panel Setup, Remote Monitoring. |
| `mime/` | MIME-type XML definitions (`.drp`, `.braw`, etc.). |
| `icons/` | hicolor icon tree (apps + mimetypes). |
| `metainfo/` | AppStream metainfo XML. |
| `shared-modules/` | Git submodule (flathub/shared-modules) for bundled deps like `glu`. |
| `EXPORT_USAGE.md` | Docs for the `--export-flatpak-resources` workflow. |
| `README.md` | User-facing overview and build instructions. |

> Note: `build-meta-flatpak.sh` and `EXPORT_USAGE.md` reference a
> `com.blackmagic.Resolve.meta.yaml` file that is **not present** in the repo
> (the manifests are `com.blackmagic.Resolve.yaml` / `...Studio.yaml`). This is a
> known doc/manifest mismatch — see *Known gaps* below.

## The installer (`installer/`)

A single entry point, `main.py`, is installed into the Flatpak as `/app/bin/resolve.py`
(or `resolve-studio.py` for Studio). It acts as both **installer** and **launcher**.

Key modules:

| Module | Responsibility |
|--------|----------------|
| `main.py` | CLI argument parsing + dispatch (install / launch / export / udev / list). |
| `config.py` | Shared constants: app name/tag, install prefix, download form data, HTTP headers/cookies, chunk size. **Edit here for app identity / download metadata.** |
| `api.py` | Talks to the Blackmagic Design support API: resolve latest version, convert a `download_id` to a real URL, list all downloads, persist/compare installed version. |
| `download.py` | Streams the installer archive to `~/Downloads` (chunked). |
| `install.py` | Extracts the downloaded payload (`.run`/`.zip`) via `unsquashfs` and installs into the prefix. Raises `InstallationCancelled`. |
| `launcher.py` | PySide6 port of the original GTK launcher. Checks installation/updates, prompts the user, and hands off execution to Resolve via `os.execvpe`. |
| `gui.py` | `InstallerApp` — the standalone progress-bar installer GUI. |
| `export.py` | `--export-flatpak-resources`: downloads/extracts the desktop files, MIME types, and icons from the installer so they can be baked into the Flatpak at build time. |
| `udev.py` | `--print-udev-rules`: emits udev rules for Blackmagic USB hardware. |

### Behavioral notes

- **Default action** (`flatpak run com.blackmagic.Resolve`): checks installation,
  prompts to install/update if needed, then launches Resolve. If already installed
  and up to date, it hands off immediately.
- **Install prefix** is `~/.var/app/<app-id>/data` inside Flatpak (since `/app` is
  read-only). Each user manages their own Resolve install.
- **Studio vs Free** is selected by the invoked script name (`resolve.py` vs
  `resolve-studio.py`) or the `--studio` flag (when run as `main.py`).
- Per-run state (installed version, download id, timestamp) is stored in
  `<prefix>/share/davinci-resolve-version.json`.
- The Flatpak forces `QT_QPA_PLATFORM=xcb` (X11) because Resolve does not support
  Wayland, and enables `RUSTICL_ENABLE` for GPU compute.

## Common tasks

### Build the Flatpak (free edition)

```bash
git clone --recursive https://github.com/night199uk/resolve-flatpak.git
cd resolve-flatpak
installer/main.py --export-flatpak-resources .   # refresh desktop/icons/mime
flatpak-builder --install-deps-from=flathub --force-clean --repo=.repo .build-dir \
  com.blackmagic.Resolve.yaml
flatpak build-bundle .repo DaVinciResolve.flatpak com.blackmagic.Resolve \
  --runtime-repo=https://flathub.org/repo/flathub.flatpakrepo
```

(Studio edition: replace `Resolve` with `ResolveStudio` in the manifest/bundle.)

### List available downloads

```bash
flatpak run com.blackmagic.Resolve --list-downloads        # free
flatpak run com.blackmagic.ResolveStudio --list-downloads  # studio
# or from the repo:
installer/main.py --list-downloads [--studio]
```

### Install a specific version

```bash
flatpak run com.blackmagic.Resolve --download_id <download_id>
```

### Install from a locally downloaded archive (import)

If you already have the official installer archive downloaded (e.g.
`DaVinci_Resolve_19.1.4_Linux.zip` or a `.run` file), skip the download and
install directly from it:

```bash
flatpak run com.blackmagic.Resolve --import-file ~/Downloads/DaVinci_Resolve_19.1.4_Linux.zip
```

The version is parsed from the filename (`19.1.4` in the example above). Both
`.zip` and `.run` archives are accepted — the installer unwraps a `.zip` to find
the embedded `.run` payload automatically. Use `--studio` (when running as
`main.py`) or the Studio script name to target Resolve Studio.


### Print udev rules (after Resolve is installed & first run)

```bash
flatpak run com.blackmagic.Resolve --print-udev-rules | sudo sh
```

## Conventions for agents

- **Python 3**, plain stdlib + `requests` + `PySide6`. No build system / packaging
  for the Python code itself; it is installed by the Flatpak manifest's
  `installer` module via `install -Dm*` commands.
- When editing the Python, keep `main.py`'s public dispatch contract stable —
  `launcher.launch_application(...)`, `api.list_downloads(...)`,
  `export.export_flatpak_resources(...)`, `udev.print_udev_rules(...)` are the
  entry points the manifest and docs rely on.
- When changing installable assets (desktop/MIME/icons), re-run
  `--export-flatpak-resources .` and **update the manifest's `sources:` block**,
  which lists every file explicitly — new assets must be added there or the build
  fails.
- The Flatpak manifest is the source of truth for bundled dependencies
  (`libxcrypt`, `squashfs-tools`, `python3-requests`, `glu` via submodule).
  Bump tags/sha256 there.
- `shared-modules/` is a **git submodule** — use `git clone --recursive` and
  commit submodule pointer changes explicitly.

## Known gaps / TODO

- `ffmpeg` plugin support is **not** ported to this meta-installer approach yet
  (noted in README).
- `build-meta-flatpak.sh` and `EXPORT_USAGE.md` reference
  `com.blackmagic.Resolve.meta.yaml`, which does not exist in the repo. The actual
  manifests are `com.blackmagic.Resolve.yaml` / `com.blackmagic.ResolveStudio.yaml`.
- `config.py` carries hardcoded placeholder download-form data
  (`DOWNLOAD_DATA`) and static cookies/headers; these may need refreshing if
  Blackmagic changes its API or bot protection.
- There is no automated test suite; verification is manual via the build/run flows
  above.

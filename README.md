# MAJOR UPDATE:
I have completely rewritten this Flatpak support with a little help from
Claude to use a Qt-based meta-installer style approach similar to Steam
or Discord. I had initially hoped to use Flatpak's extra-data approach,
however bugs in Flatpak's implementation of extra-data related to very
large downloads (>8GB) pushed me away from this.

I may not have carried over all fixes, but this approach should be much
more strategic and allow distribution of this package via e.g. Flathub
and Flatpak, opening it up to more users.

Contributions are welcome and apologies if any previous contributions
were lost in the migration.

resolve-flatpak
===============

This Flatpak installs DaVinci Resolve using Flatpak. 

Technically it is a Qt-based installer that installs DaVinci Resolve
from the Blackmagic website on-demand.  It manages the installation,
checks for updates on run, etc. The Flatpak itself contains no Blackmagic
binaries or copyright material; and thus can be distributed on Flathub,
Flatpark etc. It provides the illusion of installing Resolve from Flatpak
and makes installation simpler for users running - e.g. Silverblue and
other atomic distributions, or anyone who operates Flatpak-first.

The Flatpak installation is performed in the Flatpak run directory; e.g.
```
/home/<user>/.var/app/com.blackmagic.Resolve/data
```
Thus, each user will manage their own Resolve installation.

Usage
-----

1. **Download the latest davinci-resolve.flatpak or davinci-resolve-studio.flatpak from the releases page.**
2. **Install**
3. **Run DaVinci Resolve [or Studio].**
4. **The installer will prompt you to install the latest version of DaVinci Resolve [or Studio].**
   Alternatively, if you already have the official installer archive downloaded
   (e.g. `DaVinci_Resolve_19.1.4_Linux.zip` or a `.run` file), you can install
   from that instead of downloading:
   - **Graphical:** on the installer's first screen, click **"Import local file…"**
     and pick the archive. The version is read from the filename.
   - **Command line:**
     ```
     flatpak run com.blackmagic.Resolve --import-file ~/Downloads/DaVinci_Resolve_19.1.4_Linux.zip
     ```
   Both `.zip` and `.run` archives are accepted; a `.zip` is unwrapped
   automatically to find the embedded `.run` payload. For Studio, use
   `com.blackmagic.ResolveStudio` or the `--studio` flag (when running `main.py`).
5. **If you need udev rules for USB keys or other Blackmagic USB devices:**
This must be done *after* the real DaVinci Resolve has been installed and first run.
```
flatpak run com.blackmagic.Resolve --print-udev-rules | sudo sh
```
or
```
flatpak run com.blackmagic.ResolveStudio --print-udev-rules | sudo sh
```

Plugins
-------
I have not yet updated the ffmpeg plugin to support this latest packaging mechanism.

Advanced Stuff, Tools, and Compiling
------------------------------------

## Re-building the Flatpaks

1. Rebuild the top-level packages, and export to distributable single file installers.
NOTE: this does not package the resolve binaries; only the installer. The
installer will always obtain the Resolve binaries on run.

#### 
```
git clone https://github.com/night199uk/resolve-flatpak.git --recursive

# This line updates the static resources like icons, desktop files, etc
# that are packaged in the actual Flatpak.
installer/main.py --export-flatpak-resources .

flatpak-builder --install-deps-from=flathub --force-clean --repo=.repo .build-dir com.blackmagic.Resolve.yaml
flatpak build-bundle .repo DaVinciResolve.flatpak com.blackmagic.Resolve --runtime-repo=https://flathub.org/repo/flathub.flatpakrepo

flatpak-builder --install-deps-from=flathub --force-clean --repo=.repo .build-dir com.blackmagic.ResolveStudio.yaml
flatpak build-bundle .repo DaVinciResolveStudio.flatpak com.blackmagic.ResolveStudio --runtime-repo=https://flathub.org/repo/flathub.flatpakrepo
```

## Finding download IDs to install specific versions of Resolve

#### If you already have this Flatpak installed
This will list only the downloads for the version you have installed (Free or Studio):
```
flatpak run com.blackmagic.Resolve --list-downloads
```
or
```
flatpak run com.blackmagic.ResolveStudio --list-downloads
```

#### Directly from this repo:

```
git clone https://github.com/night199uk/resolve-flatpak.git --recursive
cd resolve-flatpak
installer/main.py --list-downloads [--studio]
```

## Installing a specific version of Resolve (using a download ID)

Install this Flatpak but do not install Resolve itself.
Or - if you already installed Resolve and want to go back to an older version:

```
rm -rf ~/.var/app/com.blackmagic.com/
```

Get a download ID for the version you want to install (see above).

Now:
```
flatpak run com.blackmagic.Resolve --download_id <download_id>
```

This will install and run the version you want.

## Installing from a locally downloaded archive (import)

If you already have the official installer archive on disk, skip the download
entirely and install straight from it:

```
flatpak run com.blackmagic.Resolve --import-file ~/Downloads/DaVinci_Resolve_19.1.4_Linux.zip
```

The version is parsed from the filename (`19.1.4` above). This is useful for
offline installs or when you want to avoid re-downloading a large archive.
A graphical "Import local file…" button on the installer's first screen does
the same thing.

## Licensing
The icon in logo.png is licensed under the Creative [Commons Attribution-Share Alike 4.0 International](https://creativecommons.org/licenses/by-sa/4.0/deed.en) and fetched from [Wikimedia Commons](https://commons.wikimedia.org/wiki/File:DaVinci_Resolve_Studio.png). It was only cropped afterwards.

## Hardware / GPU support

DaVinci Resolve has strict GPU requirements on Linux. This Flatpak only wraps
the installer — it cannot change what hardware Resolve supports. If your GPU is
not on Blackmagic's supported list, Resolve will refuse to start with an error
such as **"Unsupported GPU processing mode"**.

**Supported (Linux):**
- **NVIDIA** discrete GPUs with the proprietary driver (CUDA / NVENC)
- **AMD** discrete GPUs (recent, with OpenCL)
- **Intel Arc** discrete GPUs (DG2 / Alchemist and newer)

**NOT supported — Resolve will not run:**
- **Intel integrated graphics**, including **Iris Xe** and
  **Alder Lake-P GT2** (e.g. `Intel Alder Lake-P GT2 [Iris Xe Graphics]`)
- Other Intel iGPUs (UHD Graphics, etc.)

If you see "Unsupported GPU processing mode" on an Intel integrated GPU, this is
a hard limitation of DaVinci Resolve itself, not a packaging or configuration
issue. Do **not** try to work around it by forcing software rendering or
toggling `RUSTICL_ENABLE` — Resolve will still lack a supported GPU backend and
will crash or run incomplete. The only reliable fix is supported hardware
(NVIDIA discrete, or Intel Arc / AMD discrete).

## Related

- [Flathub forum : DaVinci Resolve Feature Requests](https://discourse.flathub.org/t/davinci-resolve-flatpak-request/842)
- [blackmagicdesign forum : DaVinci Resolve Flatpak request](https://forum.blackmagicdesign.com/viewtopic.php?f=33&t=186259)


# ParrotApp

A **personal media library manager** you run on your own machine. Point ParrotApp at
folders on your disk and it scans, indexes and de-duplicates them, then lets you browse
and play your photos, videos and music from a clean web interface — plus internet radio
from a locally mirrored station directory.

<img width="3024" height="1892" alt="ParrotApp" src="https://github.com/user-attachments/assets/b7eaa252-8925-478e-8b89-04bc36a437a1" />

## What it does

- **Library folders** — register one or more folders on disk as media sources. Each folder
  is marked as *photos*, *music* or *music videos*, and as read-only (scanned) or writable
  (an upload destination).
- **Background scanning** — files are discovered and indexed in the background, with live
  progress; EXIF is read from images and container metadata from videos.
- **Photos & videos** — a paginated gallery with per-item detail, a full-screen slideshow
  for images and a shuffleable playlist for videos, both with star ratings.
- **Music** — audio tracks and music videos with a persistent player that keeps playing as
  you move around the app, shuffle and sequential playback, a per-user play queue,
  user-defined playlists, album grid and coverflow browsing, editable metadata, album cover
  upload and online cover lookup, and a spectrum visualizer.
- **Radio** *(off by default)* — internet radio from the radio-browser.info directory,
  mirrored locally so browsing stays fast, with per-user favourites, a retro tuner dial and
  a live spectrum display.
- **Duplicate detection** — content hashing finds duplicate files and groups them for review.
- **Geolocation map** — geotagged photos plotted on an interactive map, clustered by
  proximity and reverse-geocoded to place names.
- **Thumbnails & covers** — generated on demand and cached, with tools to regenerate or
  reclaim space.
- **Multi-user** — accounts with per-user ratings, play counts, queues, playlists and
  favourites; the first account created is the administrator.
- **Logs & settings** — an admin log of events and errors, and a settings page for
  everything configurable.

## Download

Ready-to-run packages are built by GitHub Actions and attached to each
[release](../../releases). Grab the one for your system:

| Platform              | File                                        |
| --------------------- | ------------------------------------------- |
| Linux (x86_64)        | `ParrotApp-<version>-x86_64.AppImage`       |
| macOS (Apple silicon) | `ParrotApp-<version>-arm64.dmg`             |
| macOS (Intel)         | `ParrotApp-<version>-x64.dmg`               |
| Windows (x64)         | `ParrotApp-<version>-x64.exe`               |

Each package bundles its own Java runtime — nothing else needs to be installed.

## Install

> **Important — give ParrotApp a folder of its own.**
> On Linux and macOS the application stores its data **beside itself**: the embedded
> database, the thumbnails and the album covers are all created in the folder the package
> sits in. Put the package in a dedicated, empty folder (for example
> `~/Applications/ParrotApp/` or `D:\ParrotApp\`) and run it from there, so its data does
> not end up scattered through your Downloads or Applications folder. Keeping that folder
> is also how you keep your library: back it up, and moving it to another machine moves
> your whole library with it.

### Linux (AppImage)

```sh
mkdir -p ~/Applications/ParrotApp
mv ~/Downloads/ParrotApp-<version>-x86_64.AppImage ~/Applications/ParrotApp/
chmod +x ~/Applications/ParrotApp/ParrotApp-<version>-x86_64.AppImage
~/Applications/ParrotApp/ParrotApp-<version>-x86_64.AppImage
```

The `db/`, `thumbnails/` and `covers/` directories appear next to the AppImage on first run.

### macOS (dmg)

1. Open the `.dmg` and drag **ParrotApp.app** into a folder of your own — for example
   `~/Applications/ParrotApp/` — rather than into `/Applications`. The app writes its data
   into the folder containing the bundle, and it can only do that somewhere you can write.
2. The app is signed ad-hoc, not with a paid Developer ID, so the first launch is blocked
   as coming from an unidentified developer. Right-click (or Control-click) the app and
   choose **Open**, then confirm — after that it opens normally.

If the containing folder turns out not to be writable, ParrotApp falls back to
`~/Library/Application Support/ParrotApp/`.

### Windows (exe)

Run the installer. It is a per-user installation, and it offers a directory chooser — pick
a folder of your own if you like. On Windows the data is **not** kept beside the program
(an uninstall or an upgrade would delete it); it lives in
`%LOCALAPPDATA%\apps4net\ParrotApp\` instead, and survives reinstalls.

## First run

Starting ParrotApp opens a small status window showing the address it is serving on, the
data directory in use and the console output, with a **Quit** button. Click the address, or
open a browser yourself at:

```
http://localhost:9999
```

The first thing the app asks for is a **registration** — the first account you create
becomes the administrator. There is no default username or password, and once that account
exists registration closes; further users are added by the administrator from the Users page.

Closing the status window quits the application.

## Set up your library

Nothing is indexed until you tell ParrotApp where your media is.

1. Go to **Library Folders**.
2. Add a folder for each media source. For each one, set:
   - **Path** — the folder on disk (a local directory or a mounted network share).
   - **Kind** — *Photos* (images and ordinary videos), *Music* (audio tracks) or
     *Music Videos*.
   - **Mode** — *Read* for a folder to be scanned, *Write* for a folder ParrotApp may upload
     new files into. Only read folders are scanned; a write folder nested inside a read
     folder is still indexed as part of it.
3. Start a **scan**. It runs in the background with live progress, and you can keep using
   the app while it works. Rescan whenever you add files; scans only visit what has changed.

Give it a real folder — a scan that finds a library far smaller than what it has already
indexed (an unmounted network share, typically) is refused rather than deleting your
records.

## Enable Radio

The Radio page is **off by default**, and deliberately so: it is the one feature that talks
to a third party (the public radio-browser.info directory), and it keeps its station
catalogue up to date in the background whether or not the page is open. An installation
that only wants its own library makes no request on its behalf.

To turn it on: **Settings → Enable Radio**. The change takes effect immediately, no restart
needed. The Radio link then appears in the navigation, and the catalogue starts filling in
paced batches — a fresh installation has the whole directory (some 62,000 stations) within
about an hour, prioritising your country and the most-listened stations first. Both the
priority country and the pacing are settings you can change.

Turning it back off hides the page and stops the catalogue sync.

## Notes

- ParrotApp serves on port **9999**. It is meant for a machine you control; it is not
  hardened for exposure to the open internet.
- Your library files are only ever read, never moved or modified — with two opt-in
  exceptions on the Settings page: *Edit tags on files*, which writes corrected music
  metadata and cover art back into your audio files, and the file renaming that follows it.
  Both are off unless you enable them.
- Uploads (music, and video-frame screenshots) go only into folders you have explicitly
  marked as writable.

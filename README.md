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

If you already have **Java 21 or newer**, every release also carries the plain jar, one per
platform:

| Platform              | File                                    |
| --------------------- | --------------------------------------- |
| Linux                 | `ParrotApp-<version>-linux.jar`         |
| macOS (Apple silicon) | `ParrotApp-<version>-mac-aarch64.jar`   |
| macOS (Intel)         | `ParrotApp-<version>-mac.jar`           |
| Windows               | `ParrotApp-<version>-win.jar`           |

They are the same application and any of them serves the web interface on any machine — they
differ only in the native libraries the desktop status window needs, which a jar only ever
opens if you ask for it (see [Plain jar](#plain-jar-your-own-java-runtime) below). Taking the
one for your own machine is simply the way to have that option.

## Run it

> **Important — give ParrotApp a folder of its own.**
> The application stores its data **beside itself**: the embedded
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

### Plain jar (your own Java runtime)

If you already have a **Java 21 or newer** runtime installed, the jar runs on its own — no
packaged runtime, nothing to install. Check what you have with `java -version`.

Put the jar in a folder of its own, and start it **from inside that folder** (`<platform>`
being `linux`, `mac`, `mac-aarch64` or `win`):

```sh
mkdir -p ~/Applications/ParrotApp
cd ~/Applications/ParrotApp
java -jar ParrotApp-<version>-<platform>.jar
```

The working directory matters here rather than the jar's location: run this way, ParrotApp
puts `db/`, `thumbnails/` and `covers/` in the directory you started it from. Starting it
from your home directory scatters them across your home directory, and starting it from
somewhere else next time gives you a second, empty library — so `cd` into its folder first,
every time. A one-line script or a shell alias is worth setting up.

ParrotApp then serves on **`http://localhost:9999`**. Stop it with `Ctrl+C`; a plain jar shows
no status window and no Quit button by default, and logs go to the terminal it is running in.
To use another port, add `--server.port=8080`.

Adding `-Dparrot.ui=true` before `-jar` opens the same status window the packages show — which
is the one thing the jar you picked has to match your machine for, since each carries only its
own platform's native libraries. On a machine with no display it is simply skipped.

Updating is the same swap as anywhere else: replace the jar in the folder, keeping `db/`,
`thumbnails/` and `covers/` where they are.

### Docker

For a headless setup — a home server or a NAS — there is a Compose setup in a repository
of its own: **[ikiranis/parrotDocker](https://github.com/ikiranis/parrotDocker)**.

```sh
git clone https://github.com/ikiranis/parrotDocker.git
cd parrotDocker
cp .env.sample .env
```

Edit `.env` and set:

- **`MEDIA_PATH`** — the host directory holding your media. It is mounted into the
  container at `/media/myMedia`.
- **`HOST_PORT`** — the port the app is published on (default `8888`, mapped to the
  container's `9999`).
- **`CONTAINER_NAME`** — the container's name (default `parrot-docker-app`).

Then drop a ParrotApp jar in as `app/app.jar` — the `linux` one, since the image is Linux; it
is built around that jar and the jar is not committed to the repository — and start it:

```sh
docker compose up -d --build
```

ParrotApp is then at `http://localhost:8888/` (or whichever `HOST_PORT` you chose). Logs are
`docker compose logs -f`, and `docker compose down` stops it; there is no status window in a
container.

Two things differ from a desktop copy:

- **Use the container's paths when adding library folders.** Inside the container your media
  is at `/media/myMedia`, whatever its path on the host — that is what to type on the Library
  Folders page.
- **The clone directory is the data directory.** `db/`, `thumbnails/` and `covers/` are
  bind-mounted out of it, so it is the folder to back up, exactly as the folder holding the
  package is on a desktop copy.

## Update

Updating is just replacing the package — your library is not inside it.

1. Quit ParrotApp (the **Quit** button, or close the status window).
2. Download the new package from the [releases](../../releases) page.
3. Drop it into the same folder, replacing the old one — the new `.AppImage` over the old
   `.AppImage`, the new **ParrotApp.app** over the old bundle. On Linux, make the new file
   executable again (`chmod +x`), since that flag does not survive the download.
4. Start it.

The `db/`, `thumbnails/` and `covers/` directories sit *beside* the package, not inside it,
so they are untouched: your media, ratings, play counts, playlists and settings all carry
over, and the database schema is brought up to date on the first start. Nothing needs to be
rescanned.

If you would rather keep the old version around, leave it in place and give the new one its
own folder — but note that a fresh folder starts a fresh, empty library. To move an existing
library across, copy `db/`, `thumbnails/` and `covers/` over with it.

On Docker it is the same swap one level down: replace `app/app.jar` with the new one and
rebuild with `docker compose up -d --build`. The bind-mounted `db/`, `thumbnails/` and
`covers/` are outside the image, so they survive the rebuild untouched.

## First run

Started from one of the packages, ParrotApp opens a small status window showing the address
it is serving on, the data directory in use and the console output, with a **Quit** button —
click the address to open it. A plain jar or a container has no such window; it prints the
same information to its console instead. Either way, open a browser at:

```
http://localhost:9999
```

(or `http://localhost:8888`, on the Docker setup's default port)

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

## API

Everything the web interface does, it does over a REST API on the same port — so the library
can just as well be driven by a script, a cron job or another client of your own.

The full reference lives in **[apiDocumentation.md](apiDocumentation.md)**: every endpoint
with its parameters, request and response schemas and examples, grouped by area — library
folders, scanning, photos, videos, music, playlists and smart playlists, folders, search,
thumbnails, media hashes, uploads, tag export/import, settings, users, logs and radio.

A few things worth knowing before you start:

- **Base URL** — `http://localhost:9999` (or whichever port you are serving on).
- **Authentication** — stateless JWT bearer tokens. Log in at `POST /api/auth/login` with
  your username and password, then send the token you get back as
  `Authorization: Bearer <token>` on every other request. Tokens last 24 hours.
- **Permissions** — reads are open to any authenticated user, as are the per-user actions
  (ratings, view counts, playlists, queue, radio favourites). Everything administrative —
  library folders, scanning, settings, users, deletes, uploads, the log — requires the
  `ADMIN` role. A missing token is a **401**, an insufficient role a **403**.
- **Errors** — a consistent JSON shape carrying `status`, `message`, `httpStatus` and a
  timestamp.

```sh
TOKEN=$(curl -s -X POST http://localhost:9999/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"me","password":"secret"}' | jq -r .token)

curl -s 'http://localhost:9999/api/photos/all?page=0&size=20' \
  -H "Authorization: Bearer $TOKEN"
```

# Changelog

All notable changes to **ParrotApp** are documented here, grouped by the application
version recorded in `pom.xml` (`<artifactId>ParrotApp</artifactId>`) at the time of
each change. Most recent versions appear first.

The format is loosely based on [Keep a Changelog](https://keepachangelog.com/).

---

## [4.12.3] — 2026-09-24 _(current)_

- **Lyrics can be flagged as out of step.** LRCLIB sometimes matches another recording of the song, so
  its timings drift against the track being played. The player's **0** shortcut toggles an `unsynced`
  flag on the track's stored lyrics (`POST /api/music/{id}/lyrics/unsynced/toggle`, flipped server-side
  since the player does not know the current state), and `PATCH /api/music/{id}/lyrics` with
  `{unsynced}` sets it explicitly. Both are admin-only like every mutating request, and both 404 when
  nothing is stored, since flagging never triggers a lookup. The flag is the nullable `Lyrics.unsynced`
  column (null reads as false), returned on `LyricsDTO`, and cleared whenever a new match replaces the
  row. Nothing acts on it yet. The shortcut is listed on the Help page.

---

## [4.12.2] — 2026-09-24

- **Shuffle stays inside a clicked-cell filter.** `GET /api/music/random` now takes the same `songName`,
  `artist`, `album`, `genre`, and `year` parameters as the track list and is built from the same
  `musicFilter` `Specification`, so a list narrowed by clicking an Artist, Album, Genre, or Year cell
  goes on shuffling within that set rather than drawing from the whole library. The separate
  `randomFilter` it used before is gone. The Music view hands the active field filter to
  `musicPlayerStore` with the other shuffle filters. Covered by new `MusicControllerTest` cases.

---

## [4.12.1] — 2026-09-24

- **U closes the track details modal as well as opening it.** `MusicDetailsModal` now closes on the
  same key that opened it, as well as on Escape. The key is matched by `code`, so it works under any
  keyboard layout. Modifier combinations and typing in a form field are left alone.

---

## [4.12.0] — 2026-09-21

- **Lyrics lookup.** A new button beside the visualizer's (or the **Y** shortcut) opens
  `MusicLyricsModal`, which shows the playing track's lyrics from **LRCLIB** (lrclib.net), an open
  lyrics database that needs no key. `GET /api/music/{id}/lyrics` (`MusicLyricsController`) is answered
  by `LyricsService`, which tries LRCLIB's exact-match endpoint first (only when the duration is known),
  then a search by title and primary artist taking the entry closest in duration, then the same search
  with bracketed and edition suffixes stripped from the title. The request goes through the server
  because LRCLIB asks for a descriptive `User-Agent`.
- **Lyrics are stored, so each track costs at most one online request.** A match is kept in the new
  `lyrics` table (`Lyrics`, one row per `MediaFile`, plain and synced text as CLOBs).
  `TrackLyricsService` answers from that table and calls LRCLIB only for a track with no row. "Not
  found" is not stored, so lyrics added to the catalogue later can still be found; it is remembered in
  memory for six hours instead. **Search again** passes `refresh=true` to bypass both. No transaction
  is held across the network call. A `Lyrics` row is deleted before its file in every deletion path
  (`OrphanCleanupService`, `DeepCleanService`, `PhotoService`, `MusicDeleteService`, and the
  clear-all-videos endpoint).
- **Timed lyric subtitles in fullscreen.** **T**, in fullscreen only, fetches the track's lyrics
  through the same endpoint, parses the synced (LRC) form with the new `functions/lrc.ts` (`parseLrc`,
  `findLrcLineIndex`), and shows the line being sung centred on the stage with the next one dimmed
  beneath it. It follows the player onto each next track, reports on the stage when a song has no
  timed lyrics, and switches off on leaving fullscreen or on a station.
- **Tests.** New `LyricsServiceTest` and `TrackLyricsServiceTest`.

---

## [4.11.3] — 2026-09-21

- **The track list renders faster on low-powered devices.** Rows are memoised with `v-memo`, the
  labels every row repeats are translated once per render, dates use one shared `Intl.DateTimeFormat`
  instead of building a formatter per call, and off-screen rows are skipped with
  `content-visibility: auto`. On a touch screen a tap no longer opens the hover preview, which used to
  fetch artwork while the tapped track was starting.
- **Translation lookups use an index.** `languageStore` keeps the translations in a `shallowRef` and
  looks keys up in a computed `Map` rather than scanning the whole list on every call.
- **The startup restore no longer overrides the user's choice.** `musicPlayerStore` requests the
  restore's independent calls together rather than one after another, and a restore that arrives after
  the user has already picked a track or station is ignored instead of replacing it.

---

## [4.11.2] — 2026-09-18

- **The scan no longer leaves late-inserted files untagged.** The tag-scan polling in
  `MediaScanService` now reads the "file scan finished" flag once before each poll. Before, it re-read
  the flag after an empty poll, so a row inserted while that poll ran could be missed and left without
  tags.

---

## [4.11.1] — 2026-09-17

- **The video viewer steps to the previous and next item.** `VideoDetail` gains previous and next
  arrows and left/right-arrow navigation, as the photo viewer has. Photos and videos now share one
  sequence in the Photos view, so both viewers step through the grid together and load the next page
  when needed. The arrow keys still seek while the video element itself has focus.

---

## [4.11.0] — 2026-09-17

- **The Photos view remembers the folder being browsed.** The current folder is kept in `localStorage`
  (`photosCurrentFolder`), so leaving the grid (for the slideshow, say) and coming back reopens that
  folder instead of the library root. A stored folder that has since been deleted falls back to the
  root without an error.

---

## [4.10.0] — 2026-09-14

- **The Music view restores the last smart search and playlist.** The active advanced (smart) search,
  the playlist chosen in the toolbar, and the playlist loaded into the track list are kept in
  `localStorage` (`musicSmartSearch`, `musicSelectedPlaylist`, `musicActivePlaylist`) and re-run on the
  next visit. The new `isValidSmartGroup` in `smartSearch.ts` checks a stored query before use, so a
  query from an older build that names a removed field or operator is discarded rather than sent to the
  server. A stored playlist is checked against the user's playlists once they have loaded.

---

## [4.9.0] — 2026-09-02

- **The track list's ordering can use an index.** Sorting by album cluster used to go through
  `COALESCE(album_id, -media_file_id)` and an album-wide aggregate, which no index can serve, so the
  database sorted every matching track on every page, filter change, and header click. `MusicTag` now
  stores these values as columns: `clusterDate`, `clusterKey`, and `sleevePos`. They are covered by
  `idx_music_tag_cluster` and `idx_music_tag_kind_cluster`. `clusterKey` and `sleevePos` are
  maintained by `MusicTag`'s `@PrePersist`/`@PreUpdate` callbacks. The new `AlbumClusterDateService`
  owns `clusterDate` and is called wherever album membership changes: the scan and the upload indexer,
  the batch edit, the album merge, album-track removal, and every deletion path. Its `repairAll()`
  rebuilds every row from the `album_track_span` view through a session temporary table. It is run by
  `MusicClusterColumnsMigrationRunner` at startup when any track lacks the columns, and at the end of
  every scan.
- **The list's page query runs in two phases.** `MusicListQueryService` first selects only the ordered,
  paged ids, then loads just those rows with their associations in one query. A page's ratings and play
  counts are read in one batched `UserTagService.findByUserAndMediaFiles` lookup instead of one query
  per row.
- **Tests.** New `MusicControllerTest` cases cover the clustered ordering.

---

## [4.8.2] — 2026-09-02

- **The Album filter finds tracks whose album name is only on the tag.** The field filter now matches
  the name a track is *displayed* under: the joined `Album`'s name, falling back to the track's own
  `MusicTag.albumName` (a `LEFT` join plus a `COALESCE`). Clicking an Album cell therefore also finds
  music videos, and tracks whose album name never resolved to an `Album` row. Year still reads the
  joined `Album`, so a track with no album never matches it.

---

## [4.8.1] — 2026-08-29

- **A batch edit can create an album.** When not one of the selected tracks has an album, as with
  files whose tags named no album, the "group into album" option in `MusicEditModal` now creates one
  (`groupIntoNewAlbum`) instead of offering nothing. The album is resolved through
  `AlbumService.findOrCreate`, so it joins an existing row with the same (path, name) folder key rather
  than adding a second one. The name is the one being typed in the same save, falling back to the name
  the tracks' tags already share (the artist and year are handled the same way). Only audio tracks take
  part, and no file is moved. Covered by new `MusicControllerTest` cases.

---

## [4.8.0] — 2026-08-28

- **Radio recording.** A station can be recorded into the music library from a blinking record button
  on the player's transport row and on the retro tuner's front panel
  (`POST /api/radio/stations/{id}/recording`, `POST /api/radio/recording/stop`,
  `GET /api/radio/recording`, `RadioRecordingService`). The server makes the recording over a
  connection of its own, separate from the stream the listener hears, so it outlives the page and
  continues through retuning or stopping playback. A freshly loaded tab reads the state and finds its
  button already blinking. The stream is written as it arrives, without re-encoding, with an extension
  taken from the declared content type.
- **Where recordings are stored.** A recording is staged in a dot-prefixed directory the scan skips.
  On stop it is moved to `recordings/yyyy/MM/dd` in the writable `MUSIC` folder and named
  `<station> - <Artist - Song> - <timestamp>`, using the broadcast title read once when recording
  started. It is then indexed through the new `MediaUploadService.indexStoredFile`, so it appears in the
  Music view without waiting for a scan. Recordings are one per user at a time and capped at 2 GB and 6
  hours. A recording that captured nothing is deleted, and a shutdown stores every recording in flight.

---

## [4.7.4] — 2026-08-28

- **The "current song" button reports that it is working.** `goToCurrentSong` in the Music view looks
  the playing track's page up over the network before it loads that page, so the list sat unchanged for
  the whole of the first round trip and the button read as having done nothing. The loading overlay is
  now raised by the lookup itself rather than left to `loadPage`, and cleared in a `finally` so the
  paths that never reach the page load — the expected 404 for a track outside the active filter
  included — do not leave it up.

---

## [4.7.3] — 2026-08-28

- **The update check's cache window now depends on the answer,** because only one of the two goes
  stale. A result naming an update is kept for six hours (the user has been told, and nothing on GitHub
  changes that), while "up to date" is kept for thirty minutes — the same window a failure gets — since
  it stops being true the moment the next release is published. With one window for both, a release
  published shortly after the previous one was hidden for the rest of the day from a running
  installation, which is exactly when someone is looking; even the shorter window is one or two
  requests an hour against GitHub's sixty.
- **A qualifier no longer swallows the component it is attached to.** `compareVersions` reads a
  component up to its qualifier instead of dropping the qualified component whole, so `4.6.3-SNAPSHOT`
  is 4.6.3 rather than 4.6 — a development build read as `X.Y` sits level with `X.Y.0` and would be
  offered every patch release of its own line as an update. Covered by new `UpdateCheckServiceTest`
  cases.

---

## [4.7.2] — 2026-08-27

- **The top bar is responsive in three stages rather than one.** Between the `lg` and `xl` breakpoints
  the bar still carries every destination but drops their labels, the icons alone fitting and their
  tooltips carrying the names. Below `lg` the wrapper stops being a `display: contents` pass-through
  and becomes the panel the hamburger drops down, taking the destinations, the secondary links, and the
  actions — theme, language, logout, version — into it together; icon-only links on a bar that narrow
  scrolled sideways and left the language flags and the logout button spilling off the edge. The panel
  scrolls rather than running off the bottom of a short screen.
- **Tests.** `MusicControllerTest`'s assertions were brought in line with the file naming the rename
  service now produces.

---

## [4.7.1] — 2026-08-25

- **A single carries its own cover.** Cover art normally hangs off the `Album` — one image shared by
  every track under it — so a track with no album row was the one kind of audio track that could never
  show artwork nor be given any. A single now keeps its own cover on `MusicTag.cover` through the new
  `TrackCoverService`, the counterpart of `AlbumService`'s cover handling with the same two attach
  modes (the scan's `attachCoverIfAbsent`, which never replaces, and the user's `replaceCover`, which
  always wins and deletes what it supersedes). `MusicDetailDTO` reads the album's cover first and falls
  back to this one, so nothing displaying artwork needs to know where it hangs, and a single later
  grouped into an album simply shows the album's instead. `MusicTagScanner` accordingly no longer
  discards the embedded artwork of a track that resolved to no album.
- **The three album cover endpoints have per-track counterparts.** `POST /api/music/{id}/cover`,
  `GET /api/music/{id}/cover-candidates`, and `POST /api/music/{id}/cover-from-url` mirror them exactly
  — same upload validation, same online providers, same `editTagsOnFiles` embedding, of one file rather
  than an album's, then queued for the deferred hash refresh. The two that set a cover **refuse** a
  track that belongs to an album (the album's cover takes precedence, so one set on the track would
  never be displayed) and a music video, which carries a frame thumbnail instead and has no artwork tag
  to embed one in.
- **A single can be downloaded as the file it is** rather than as a zip of one:
  `GET /api/music/{id}/download` names it `<artist> - <title>` plus the file's extension through
  `AlbumDownloadService.trackFilename`, which is the archive's own entry naming minus the sleeve
  position — that orders an archive and says nothing about a lone file. In the edit modal the cover box
  and the download row are now shown for one selected album **or** one selected single, the same
  controls either way.
- **A single's cover is the one cover deleted with its track.** Because it belongs to that one track
  rather than being shared, both `MusicDeleteService` and `OrphanCleanupService` remove it — they leave
  every other cover in place — since leaving it would strand a row and a file nothing could reference
  again.

---

## [4.7.0] — 2026-08-25

- **Removing a track from an album.** The inverse of the merge — a track that ended up in an album it
  does not belong to — is the new `AlbumTrackRemovalService`, reached two ways: the album panel's
  per-row **remove** button, for the intruder spotted while browsing an album, and a **remove from
  album** checkbox in `MusicEditModal`, for rows ticked in the track list (deliberately absent from the
  album-wide edit, where it would only dissolve the album a track at a time). Both go through the batch
  metadata edit's new `removeFromAlbum` flag, the opposite number of `setSongsInAlbum` and refused
  alongside it, applied before the field edits so an album name typed in the same save still lands on
  the detached tag.
- **Only the link goes.** The album name on the track's own `MusicTag.albumName` is left alone, being
  the track's own metadata rather than the grouping the user asked to undo, and a track whose tag never
  got a copy of the name takes one from the row it is leaving — so no track comes out of a removal
  knowing less about itself than it went in with. A detached track goes on reporting an album name
  through `MusicDetailDTO`'s fallback exactly as a music video always has, and inherits that
  arrangement's one consequence: the list's album and year filters join the album row, so a detached
  track is no longer found by either, which is why the view reloads a page narrowed by one of them.
- **An album the removal empties is deleted along with its cover,** exactly as a merged-away album is,
  the album panes listing only albums that still hold tracks; that is the one part a user cannot undo
  by grouping the track back in, so the client confirms that case first. Like the merge it is a
  database regrouping only — no file is moved, renamed, or retagged, whatever `editTagsOnFiles` says.
  Covered by new `MusicControllerTest` cases.

---

## [4.6.4] — 2026-08-25

- **Theme toggle colours.** The toggle only ever sits in the top bar, which is dark in both themes, so
  it is drawn white rather than themed — the theme's own foreground read as near-black on that bar in
  light mode.

---

## [4.6.3] — 2026-08-25

- **The top bar's menu simplified.** The link count no longer decides anything: the main destinations
  hold the middle of the bar and never wrap (the row scrolls instead, keeping the bar one row tall),
  while the secondary destinations float below the hamburger as a dropdown at every width, hanging down
  and to the left of the toggle so a menu wider than the button cannot run off the right edge. Below
  `lg` the main links drop their labels rather than themselves, their tooltips carrying the names,
  while the dropdown's entries stay labelled — a dropdown has the width for it, and icons alone would
  be a puzzle.

---

## [4.6.2] — 2026-08-25

- **Album download entries are named from the database.** `AlbumDownloadService` now pairs each file
  with the name it is stored under through the new `AlbumTrackFile` record: `<position> - <artist> -
  <title>` plus the file's own extension, rather than the file's name. A file name is whatever the user
  happened to receive, while the stored metadata is what the application knows about the track and what
  a metadata edit corrects. The position is padded to two digits so an extractor, which lists entries
  by name, still shows the album in the sleeve order the archive is written in (positionless tracks
  last and unnumbered), a track the database knows neither an artist nor a title for keeps its file
  name — the only name left — names are capped in length, and two entries that would collide are
  deduplicated with a `(1)` suffix.

---

## [4.6.1] — 2026-08-24

- **The version number checks for a newer release.** A packaged copy is updated by replacing the
  package by hand — the application runs from inside the very bundle being swapped — so
  `UpdateCheckService` reads the project repository's latest release from GitHub
  (`GET /api/general/update` on `GeneralController`) and compares its tag against the running
  `app.version`. It is **proxied through the server** because GitHub refuses a request that does not
  identify itself with a `User-Agent`, which a browser will not let a page set; its result is **cached
  in memory**, so several tabs, users, and page loads cost one outbound request per installation rather
  than one each; it **sends nothing about the library** and nothing schedules it — it runs when a page
  asks; and **every failure is answered as "no update known"** rather than as an error, logged at
  `DEBUG` so an installation with no outbound network does not file a Logs page entry for a feature it
  never asked about.
- **The comparison** is numeric component by component with a missing component counting as zero, so
  `4.10.0` beats `4.9.0` where a text comparison would not, and anything following the numbers is
  ignored — a `-SNAPSHOT` build compares equal to the release it precedes, which is the conservative
  answer since a development build cannot be replaced with a package anyway.
- **The UI.** The check is the last thing `TopBar` does on mount and its failure is swallowed, so a
  slow or unreachable GitHub never holds up the media counts or the radio flag. With an update
  published the version number stops being a dimmed label and becomes a lit button opening the new
  `UpdateModal`, which states the two versions, the release date and notes link, and the four steps of
  the replacement, together with the note that `db/`, `thumbnails/` and `covers/` sit *beside* the
  package rather than inside it — the fact that makes the swap safe. The release's assets are listed as
  download links with the ones matching the viewer's platform first and badged, matched by file name
  since a release carries nothing naming the target system; the match is advisory and by OS family
  only, so every asset stays listed and a wrong guess costs nothing but the ordering.

---

## [4.6.0] — 2026-08-24

- **Download an album.** `GET /api/music/albums/{albumId}/download` (`AlbumDownloadService`) hands an
  album's own files back as one zip named `artist - album (year).zip` from the `Album` row, offered
  from the Music view's edit modal under the same one-album condition as the cover upload. The
  library's files stay where the user put them, so an album's tracks are scattered under a library
  folder rather than sitting in a folder of their own — the album is the grouping only this application
  knows. Entries carry no directory prefix, since the album is already the archive, and the deflater is
  set to `NO_COMPRESSION` because audio files are already compressed and the zip is wanted for carrying
  the tracks together rather than for shrinking them.
- **The files are collected before the first byte is written,** so an album whose files have all gone
  missing is a 404 rather than an empty archive (a single missing file is skipped with a `WARN`, the
  rest of the album still being what was asked for), and the archive is written straight to the
  `HttpServletResponse` for the reason `RadioStreamProxyService` documents: a `StreamingResponseBody`
  would make the request asynchronous, which carries the container's timeout and a second,
  unauthenticated pass through the security filter chain.

---

## [4.5.1] — 2026-08-24

- **Support section on the Home view,** with the matching `multiLanguage.xml` entries.

---

## [4.5.0] — 2026-08-24

- **Radio favourites export and import.** Favourites are the one part of the radio feature that is not
  a mirror of the directory — a catalogue any installation rebuilds by crawling, against favourites
  that exist nowhere else — so they are the one part worth carrying out of an installation.
  `GET /api/radio/favorites/export` writes **every** user's favourites as one JSON file and
  `POST /api/radio/favorites/import` reads such a file back (`RadioFavoriteTransferService`), both
  admin-only through a `/api/radio/favorites/**` rule of their own, since the pair covers the whole
  installation rather than the caller and the blanket GET rule would otherwise hand one user's file to
  another. Both sit in the Library page's Radio Catalogue card.
- **Three decisions shape the format.** A favourite is a station uuid and a date and nothing else — the
  uuid is the directory's own identifier and the only value that means the same thing on every
  installation, so describing the station would only be a stale copy of what the importing catalogue
  holds; the cost is that a uuid the local crawl has not reached yet is counted as missing, and running
  the same file again once it has caught up picks it up. Users are named by **username**, matched
  against this installation's own users; an unknown one is reported, never created, because a user
  record carries a password and a role and inventing one from a file would be a way past the login. And
  the import **merges** — it adds what is missing and touches nothing else, so applying a file twice is
  a no-op the second time and an old file never costs a user a favourite marked since it was written —
  reporting everything it could not apply rather than failing over one of them. It is synchronous,
  unlike the library tag import: an installation's favourites are hundreds of rows, so there is no job
  to watch.

---

## [4.4.1] — 2026-08-24

- **A far fuller Help page,** rewritten to cover the application as it now stands — the media views and
  their filters, the music player and its shortcuts, radio, the library and its jobs — with some 675
  new `multiLanguage.xml` entries behind it.

---

## [4.4.0] — 2026-08-24

- **Support page** (`/support`) with contact and donation information, reachable from the top bar and
  translated like every other view, and the matching Support section in the public README.
- **The top bar collapses instead of wrapping.** Which links are shown depends on the user's role and
  on what the library holds, so a full menu was far wider than a minimal one and wrapped onto a second
  row on all but the widest screens. The rendered links are counted from the DOM — rather than
  repeating the template's per-link conditions in script, which would drift as links are added — and
  past a threshold they collapse behind a hamburger, independently of the viewport width, which
  collapses the whole menu on its own below the `lg` breakpoint. A click outside the nav closes it.

---

## [4.3.3] — 2026-08-23

- **Build and release.** The installer workflow publishes the platform jars alongside the packages, and
  the README and the packaging scripts were corrected on the distinction between a package (dropped
  into place) and an installer.

---

## [4.3.2] — 2026-08-23

- **Pseudo fullscreen in the music player (G).** A fixed overlay filling the browser *window* rather
  than the screen: the presentation an iPad's own player offers, with the browser's chrome and the OS
  bar left where they are. It exists because the real fullscreen is not always wanted — it takes over
  the whole display, and a browser may refuse it outright, granting it only from a user gesture and, on
  iOS, to a video element alone rather than to the stage around it. The two modes are mutually
  exclusive (entering either leaves the other, awaited so the `fullscreenchange` cannot arrive late and
  clear the overlay again), everything fullscreen-only is driven by whichever is active, the body is
  locked while the overlay is raised so a wheel or a swipe cannot move the page underneath, and unlike
  F it is answered on every page, the Radio view included: the retro dial is what the *screen* is
  handed over to there, while this only ever fills the window the page already occupies.
- **Infinite scroll for the photo grid,** with the intersection observer re-armed after each page so a
  page that arrives without moving the sentinel out of view still asks for the next one.
- **A public README,** covering the project overview, installation, and the API documentation.

---

## [4.3.1] — 2026-08-21

- **JavaFX on macOS in CI.** The workflow builds each platform's jar with `-Djavafx.platform=...`,
  overriding the property openjfx's own profiles set from the machine the build runs on. A jar per
  platform is not an artifact-naming nicety — the natives are platform specific, so it is the only
  reason the status window appears anywhere but the build's own operating system.

---

## [4.3.0] — 2026-08-21

- **A desktop status window for a packaged launch.** A launch from the macOS bundle, the Windows
  installation, or the AppImage is started by clicking an icon and has no terminal behind it, so until
  the browser is pointed at the right port there was nothing to say whether the server came up, failed,
  or was never started — nor any way to stop it short of ending the process by hand. `AppStatusUi`
  (`ui/`) shows a small JavaFX window reporting the serving address (clicking it opens a browser),
  whether startup succeeded, the data directory in use, the console output, and a Quit button.
- **It is shown only where it is wanted.** The packaged launcher is recognised by the
  `jpackage.app-path` property jpackage sets — the same one `anchorDataDirectory` already keys off — so
  a development run, a plain `java -jar`, and a headless deployment behave exactly as before; the
  `parrot.ui` system property overrides that decision in either direction. The window is created
  **before** Spring starts and shows a starting state until the server reports itself ready, because
  the seconds a cold start takes are precisely when a user has nothing to look at, and a startup
  failure is not fatal to it: `main` returns rather than rethrowing, and the window stays up on the
  console output that explains what went wrong.
- **It is best-effort by design.** `AppStatusUi` is JavaFX-free and reaches the toolkit only through
  `AppStatusWindow`, loaded lazily and wrapped in a `catch (Throwable)`, so a machine with no display
  or a build carrying no JavaFX natives costs nothing more than the window not appearing. JavaFX is
  pinned to 23.x, since 24 and later are compiled for Java 22+ and will not compile under the project's
  `release 21`; the natives load out of the fat jar through JavaFX's own extract-and-cache fallback,
  and both packaging scripts pass `--enable-native-access=ALL-UNNAMED` and
  `--sun-misc-unsafe-memory-access=allow` — guarded on the packaging JDK's feature version, since
  jpackage bakes `--java-options` into the launcher and an unrecognised option would stop the packaged
  application starting at all.
- **What the log view shows is the console output,** captured by `ConsoleLogBuffer` replacing
  `System.out`/`System.err` with streams that tee every byte into a 2000-line ring buffer —
  deliberately coarser than another logback appender, since what a user means by "the terminal logs" is
  everything that would have appeared on a console, here a mixture of logback output and plain
  `println` calls. Because logback's console appender resolves `System.out` once when it starts, the
  capture is the first statement in `main`, ahead of even the Derby system properties. Readers pull
  rather than subscribe, so the window can never hold up a thread that is writing to the console.
  Covered by a new `ConsoleLogBufferTest`.

---

## [4.2.7] — 2026-08-20

- **Release workflow.** Token validation and public-repository support for the installer build, pointed
  at the project's own release secret.

---

## [4.2.6] — 2026-08-20

- **Pagination without counting.** The folder and search endpoints now page through the new
  `UncountedPage` utility rather than running a count query per request: on a large library the count
  dominated the request while nothing on those views needed a total. The photo grid consumes the
  short-page signal instead of a page count, and the thumbnail endpoint gained the matching test
  coverage.

---

## [4.2.5] — 2026-08-20

- **`dateCreated` is when the file entered the library, not when it was taken.** `PhotoTagScanner` and
  `VideoTagScanner` no longer seed it from EXIF or the file's last-modified time: it is left unset and
  stamped when the record is written (by the entity's `@PrePersist` on the single-file path, by the
  batch insert's own timestamp on the other). The capture date is read into `dateTaken` and left null
  when the file carries none.
- **The video element's own fullscreen button is hidden in the Playlist view.** It sat a few pixels
  from the view's own fullscreen toggle and did something different (the bare video, without the rating
  stars or the playlist chrome). Chromium hides it from `controlslist`; WebKit ignores that and exposes
  it as a pseudo-element, so both are handled.

---

## [4.2.4] — 2026-08-16

- **Download a photo or a video.** `PhotoController` and `VideoController` gained download endpoints,
  offered from the photo and video detail panes and from the Slideshow, so a file can be taken back out
  of the library as it is.

---

## [4.2.3] — 2026-08-14

- **Album-wide batch edit from the album panel.** The panel's header carries an **edit** button opening
  the very same `MusicEditModal` the track list's multi-select opens — metadata fields, cover upload,
  and online cover lookup alike — on **every track of the album** rather than on ticked rows: a whole
  album is exactly the set a batch edit is usually aimed at, and reaching it through the list meant
  leaving the album panes, filtering by the album, and ticking its rows a page at a time. The view
  feeds the modal from an `editSongs` computed that reads the opened album's already-loaded tracks
  while `albumEditMode` is set and the checked rows otherwise, so one modal instance serves both. The
  only difference inside the modal is wording (`albumName`), since "all selected tracks" would be
  untrue of tracks the user never selected; the "group into the majority album" checkbox is naturally
  absent, every track already being in one album. Because the server writes an edited name and year
  onto the shared `Album` row as well, an album-wide edit patches the panel header and the album's tile
  in place (`applyAlbumEdit`), and `applyUpdate` patches the opened panel's own copies of the tracks
  alongside the list's.

---

## [4.2.2] — 2026-08-14

- **Cover candidates carry their resolution.** Neither provider reports a size, so `fillDimensions`
  measures each one: it fetches only as much of the image as `ImageIO` needs to read its header and
  closes the stream the moment the size is known, so a lookup costs a few kilobytes per candidate
  rather than a download of every one, and the probes run on virtual threads so the lookup waits for
  the slowest rather than the sum. It is deliberately measured rather than declared from the URL — the
  archive's "1200" thumbnail is 1200 on its longest side only. Probing goes through the same host
  allowlist, re-checked on every redirect, as the download does; a probe that fails leaves the size
  null, shown as an em dash and filled in from the browser's own `naturalWidth` if the candidate's
  preview is opened.
- **Candidates can be previewed full size** before one is picked, because a grid tile a few dozen
  pixels wide is not enough to choose a sleeve by. The modal's cover box states a resolution too — of
  the album's stored cover, or of the file or candidate about to replace it — so a candidate is
  compared against what is already there rather than in the abstract; that one needs no server work,
  the box displaying the stored cover itself, and only a picked candidate takes its size from the
  server's probe.

---

## [4.2.1] — 2026-08-14

- **Find a cover online.** For the many albums whose files carry no embedded artwork at all,
  `CoverLookupService` (`GET /api/music/albums/{id}/cover-candidates`) consults two providers in order:
  **MusicBrainz + the Cover Art Archive** first, because its images are contributed for reuse and a
  personal library is what they are there for, and the **iTunes Search API** only when the archive has
  nothing, because its coverage of commercial releases is what makes the feature useful rather than
  merely correct. Neither needs a key.
- **Four things about it are deliberate.** It runs **only on that button press** — nothing about the
  library reaches a provider on its own, which is what makes an outbound request acceptable here at
  all. It **never applies a match itself**: matching a release by name is guesswork and a plausible
  wrong sleeve is worse than none, so candidates come back as a grid for the user to pick from, and the
  pick is applied on Save through the same `replaceCover` path (and the same `editTagsOnFiles`
  embedding) as an upload — `POST /api/music/albums/{id}/cover-from-url`, where the **server**
  downloads the image rather than the browser. The search terms are cleaned first, since a scanned
  album name is not a catalogue title: bracketed groups naming a format or edition are dropped
  (`Dirty Blonde (flac)`) while a bracket that belongs to the name is kept (`Wish You Were Here
  (Live)`), and a credit naming several artists is reduced to the first — on a comma, semicolon, or
  "feat.", but never on an ampersand, which joins one act's name far more often than it separates two.
  And because the image URL comes back in a request body, the download is confined to an **allowlist**
  of the providers' own hosts, HTTPS only, with **every redirect re-checked** against that list rather
  than followed blindly — both providers redirect twice onto archive.org storage nodes, so following
  hops is required and re-checking them is what stops an allowed host being a way past the allowlist.
  MusicBrainz's one-request-a-second and descriptive `User-Agent` requirements are honoured server-side
  (`throttleMusicBrainz` paces the whole installation, not one tab).
- **Touch and wheel navigation for the album slider,** so the coverflow can be swiped on a phone and
  scrolled with a wheel rather than only arrowed.

---

## [4.2.0] — 2026-08-13

- **Merging albums.** Album identity is the (path, name) folder key, so one release whose tracks were
  downloaded into different folders — or whose embedded tags spell its name with different casing or
  punctuation — is scanned into several `Album` rows and shows as several tiles of the same record;
  nothing in the scan can tell that apart from two genuinely distinct same-named releases, which is why
  the folder grouping is deliberate and why putting them back together is an explicit user action. A
  toolbar toggle puts the **albums grid** into a **selection mode** where a click ticks a tile instead
  of opening it — the grid only, since a group tile is not an album and the slider centres one cover at
  a time — and `MusicAlbumMergeModal` asks which of the picked albums to keep, defaulting to the one
  holding the most tracks (ties to the lowest id), usually the one the release's real name and artwork
  were read from.
- **`POST /api/music/albums/merge`** (`AlbumMergeService`) reassigns every track of the others to it,
  writing the survivor's name onto each moved track's own `MusicTag.albumName` so the tag's copy does
  not contradict the album it now sits in, and deletes the rows they leave empty — the tracks must move
  first, since a row with tracks still pointing at it cannot go. The survivor keeps its own cover and
  adopts one from a merged-away album only when it has none, so a merge never replaces artwork the user
  already has nor costs them the one that was on screen; every other cover is deleted with its row
  rather than lingering as an orphan. It is a **database regrouping only** — no file is moved, renamed,
  or retagged — and it survives later scans, which only ever visit files carrying no `MusicTag` yet;
  only a deep clean undoes it. Covered by a new `AlbumMergeServiceTest`.
- **Clicking an album's name in the panel header leaves the album panes,** switching to the track list
  filtered by that album name — the very same field filter a click on an Album cell applies — so the
  album can be browsed with the list's own pager, sorting, multi-select, and per-row actions. The
  filter matches the name rather than the id, since that is the one album filter the list carries and
  can clear; an album whose name is already the active filter is not re-applied, a repeat click on the
  same value toggling that filter off and leaving an unfiltered list behind the request to see one
  album.

---

## [4.1.6] — 2026-08-11

- **The spectrum visualizer in the music player.** The same `AudioVisualizer` the retro tuner draws
  above its dial, reading the same `audioAnalyser` tap on the same media element, is now available in
  the player — toggled with **B** or its button, which sits with the play-queue button in a
  right-aligned row beneath the rating stars rather than on the transport bar, since neither drives
  playback and eight controls left that row unreadable in a 260px column. The toggle is persisted to
  `localStorage` (`musicVisualizer`), unlike the vinyl toggle, because switching it on is also what
  routes the audio through Web Audio and a user who asked for it once should not have to ask again
  every reload.
- **Windowed and fullscreen are opposites, deliberately.** Windowed it is an ordinary block at the foot
  of the player pane, drawn as the same black-glass LCD panel the tuner uses. Fullscreen the
  component's `transparent` prop drops the panel, the frame, and the glass's shine and draws the bare
  bars at a much lower opacity, fixed to the foot of the *screen* and painted **over** the tag overlay:
  a solid panel there would cut a band out of the artwork, so the display costs the stage no space at
  all and the track's tags read straight through the bars. The bar opacity is therefore not a styling
  detail to raise later — it is the only thing keeping the text legible — and the per-bar glow is
  dropped with the panel for the same reason. Only one of the two is ever mounted, and the panel is
  drawn independently of the `O`/`I` overlay toggles, since turning the tag overlay off is not a
  request to lose the visualizer with it. It is deliberately **library tracks only**: on a station the
  Radio view's dial already carries this display, and the player's `F` hands fullscreen over to that
  dial, so a copy here would draw the same spectrum twice.

---

## [4.1.5] — 2026-08-11

- **`RadioVisualizer` became `AudioVisualizer`,** carrying nothing radio-specific, in preparation for
  the music player hosting it too.
- **The player exposes `togglePlay`, `stop`, and `resume` on the store,** so a caller with no component
  reference of its own — the retro dial's play control — can end or restart playback the way the
  transport bar does: re-selecting an already-loaded station leaves the player's media key unchanged
  and so triggers no reconnect.
- **The Radio view's search filters persist,** and its filter dropdowns are sorted alphabetically,
  which the count-ordered vocabularies alone did not make browsable.
- **`MusicDetailsModal` shows its values in read-only inputs** rather than spans, so a long value can
  be selected and copied.

---

## [4.1.4] — 2026-08-09

- **Artist filtering in the Music view.** Because an artist tag routinely credits several people in one
  string, clicking an Artist cell whose value splits on "," or "&" into more than one name
  (`splitArtists` in `functions/artistNames.ts`) no longer filters straight away:
  `MusicArtistPickerModal` asks which of the credited artists to search for, offering the whole credit
  as a further choice, since the same separators appear inside single names ("Simon & Garfunkel"). The
  free-text field filters match as substrings, so filtering by one artist also finds the tracks where
  they appear among several.
- **A sleep timer** in the music player and toolbar (`MusicSleepTimerModal`), stopping a track or a
  live stream alike after a chosen number of minutes. It is held in the store as an end **timestamp**
  rather than a remaining duration, so a countdown can recompute on every tick without the store
  needing an interval of its own, and it is deliberately not persisted across a reload: it is a
  one-shot wind-down for the current listening session, not saved playback state.
- **At most one write folder per media kind.** `LibraryFolderService` now rejects a second one, since
  an upload destination is chosen by kind alone and two write folders of the same kind would leave that
  choice ambiguous. Covered by new `LibraryFolderControllerTest` cases.
- **A GitHub Actions workflow building the platform installers,** with the macOS job on
  `macos-15-intel` and the data-directory handling in `ParrotApp` reworked to match how a packaged copy
  is actually laid out.

---

## [4.1.3] — 2026-08-09

- **The catalogue crawl is paced more gently by default:** the batch interval moved from 30 minutes to
  60 and the cycle from 24 hours to 168 (weekly). A station list changes little day to day, and a
  tighter cycle only bought more chances for a mirror hiccup to surface as a failure.
- **Radio logging levels lowered** where a failed mirror or an unreadable stream title is an expected
  outcome rather than a fault, and the metadata-editing setting's description clarified on the settings
  page.

---

## [4.1.2] — 2026-08-08

- **Fullscreen is answered by exactly one of the two.** F is a shortcut of the *Radio view*, not of the
  player, when that view is mounted — the dial *is* what fullscreen means for radio, since a live
  stream has no video stage worth filling the screen with — so the player stands aside for as long as
  the view is on screen: both its shortcut handler and its fullscreen button check the same
  `retroAvailable` flag the dial's availability is published under, the button raising the dial instead
  of its stage on a station.
- **The retro tuner on a phone.** Held upright the cabinet has height and no width, so the display is
  given a height of its own instead of the leftover, the dial window shrinks with it, the stack is
  centred rather than pushed to the foot, and the panel becomes a two-row grid — logo and details
  above, stereo lamp, favourite star, and play control below — because in one row the details were
  squeezed to nothing and spilled off the edge. On its side it is the opposite: the panel is pared back
  to what its three rows need and every pixel that frees goes to the display. Both hold the panel to
  exactly three rows by printing the station's details as a single elided line.

---

## [4.1.1] — 2026-08-08

- **Station titles are reduced to the part a listener is meant to see.** A station's `StreamTitle` is
  whatever it chose to put there, and a good many (the iHeart networks in particular) pack their
  playout system's whole cue sheet into it — the song followed by a few hundred characters of media
  ids, an artwork URL, and a spot instance, which filled the player's pane and drowned the station's
  own details. `sanitize` cuts everything from the first appended `key="value"` field on (at the
  *first* field rather than by matching a well-formed run of them, since those values are unquoted and
  frequently malformed) and tells two cases apart: usually the sheet follows a complete title, which is
  simply kept, but some stations open it immediately after the artist and separator (`Artist -
  text="Song" ...`), where cutting would leave the artist dangling on its separator and lose the song,
  so the sheet's own `text` field is joined back onto the artist. Whatever survives is capped at 200
  characters on a word boundary; on the client the now-playing line is clamped to three lines with the
  whole of it on its tooltip.
- **The now-playing title on the fullscreen dial** is drawn as a third row of the panel readout beneath
  the station's name and details — fullscreen only, the inline dial's panel being the compact one, and
  filled only while the tuned station is the one actually streaming, because the needle wanders freely
  over the listing while the player stays where it was and printing one station's song under another's
  name would be plainly wrong. The row itself is drawn whether or not there is a title for it, an em
  dash standing in until one arrives, because a station's own details are on screen the moment it is
  tuned while its broadcast title follows seconds later, and a row that appeared with the title grew
  the panel under the user's eyes.
- **Visualizer styling:** gradient colours, the LCD panel effect, and the peak caps refined.

---

## [4.1.0] — 2026-08-08

- **A pre-scan safety check for an incomplete library.** A library folder is very often a network
  share, and a share whose mount did not come up is an *empty* directory rather than a missing one —
  the case that motivated the check being a container started before its mount finished. The Phase 1
  walk then finds nothing, every folder reads as changed, and the post-scan orphan and empty-folder
  cleanups delete the records of every file that is merely unreachable, taking their thumbnails, tags,
  ratings, play counts, and playlist entries with them. So after Phase 1 has counted what is on disk,
  `MediaScanService.findIncompleteLibraries` compares that count against
  `MediaFileRepository.countByLibraryFolder` and, when a folder is short by
  `scanMissingFilesThresholdPercent` (10 by default; `0` switches the check off) or more of what it has
  indexed, the scan **stops before Phase 2** — nothing is added and nothing is removed.
- **What it reports.** The background scan marks the job `FAILED` with the reason, which the Library
  page's scan card now renders (a refused scan's counters are all zero, so the badge alone would say
  nothing), the synchronous path returns it as the `ScanResult` message, and either way it is recorded
  as a `SCAN_ABORTED_INCOMPLETE_LIBRARY` warning on the Logs page.
- **Three details matter.** The comparison is **per library folder**, not library-wide, so one
  unmounted share is caught while the others are healthy. A folder with **nothing indexed yet** is
  always allowed through, since a first scan has nothing to lose. And the check is **skipped on
  cancellation**, where the walk stopped early and is short by definition. The cost of the guard is
  that a genuine bulk deletion outside the application is refused too, which is what the setting exists
  to let through. Covered by a new `MediaScanServiceMountGuardTest`.

---

## [4.0.2] — 2026-08-07

- **The retro tuner.** `RadioTuner.vue` lays the very same stations along a backlit amber scale in
  place of an analogue tuner's frequencies, switched from a toolbar button and remembered in
  `localStorage` (`radioViewMode`). Their names are printed **vertically**, as a real dial's legends
  are, because a dial has height to spare and no width at all. Three things follow from the catalogue
  rather than the styling: the **scale travels and the needle is fixed**, the reverse of a real
  receiver, because a page holds fifty stations and the dial is a few hundred pixels wide; the dial
  appends its own pages as the needle nears the end of what is loaded; and only a window of marks
  around the needle is rendered. Its palette is deliberately **not themed** — it imitates one
  particular object, and a 1970s receiver looks the same in a lit room as in a dark one.
- **Tuning plays, as it does on the real thing,** with no separate press of play. What keeps that
  usable is that it waits for the needle to **settle** — a sweep of the dial passes over dozens of
  stations, and starting each in turn would open a relay per station and give the listener a burst of
  half-second fragments — so a move only schedules the station under the needle and any further move
  within 450 ms replaces it, while a click on a station's name plays at once, being an explicit choice.
  A station already playing is left alone rather than torn down and reconnected, a click that merely
  ends a drag is ignored, and grabbing the dial again cancels whatever was about to start. Crucially,
  only genuine tuning schedules a station: the watchers that move the needle when the listing is
  refiltered, when the list shrinks, or when the player moves on all set the index directly.
- **The stream is relayed through the application.** A `MediaElementAudioSourceNode` fed by a
  cross-origin resource that was **not CORS-approved outputs silence** — not merely unreadable data —
  so attaching to a station fetched straight from its own server would have muted the player rather
  than just leaving the bars flat. `GET /api/radio/stations/{id}/stream`
  (`RadioStreamProxyService`) relays the audio so the browser fetches it same-origin, the media element
  carries `crossorigin="anonymous"`, and the app's wildcard CORS configuration makes it readable. It is
  spoken over a **raw socket** rather than the JDK's HTTP client, because a great many stations are
  Shoutcast v1 servers answering `ICY 200 OK`, which is not valid HTTP and which that client refuses
  outright; it asks for HTTP/1.0 so the body is never chunked, does not ask for ICY metadata (those
  blocks would be noise to a browser), and resolves the station from its uuid through the local
  catalogue rather than taking a URL from the caller. The audio is written **straight to the
  `HttpServletResponse`**: a `StreamingResponseBody` makes the request asynchronous, and Spring leaves
  that timeout at the container's default — Tomcat's being 30 seconds, which cut every live stream off
  after half a minute, its ending then dispatching back through the filter chain where Spring Security
  ran a second time with no authentication and answered Access Denied onto an already-committed
  response. The content type is the one piece of a station's response copied onto a header of the
  application's own, so it is checked against a plain `type/subtype` shape rather than trusted.
  `RadioStreamProxyServiceTest` drives it against a local server answering as each station type,
  checking the audio arrives byte for byte.
- **The spectrum display and the signal meter.** `functions/audioAnalyser.ts` taps the shared player
  through a Web Audio `AnalyserNode`; because a source may be created **once per element** and routing
  it is **permanent**, the graph is built at most once, never torn down, and attached lazily on mount,
  so a user who never opens the dial never has their playback routed through Web Audio at all. The FFT
  bins are mapped onto the bars **logarithmically**, each bar taking the loudest bin of its band;
  getting a spectrum that reads rather than saturates takes the analyser's dB window widened to
  -90..-10 dB, a bass tilt that is **added, not multiplied** (the byte an analyser reports is
  proportional to decibels and the bars are spaced by octave), and a display that **calibrates itself**
  to the signal, filling to 92% of the height whatever the station's level with a floor so silence
  stays quiet. The **signal meter** shows the honest equivalent of signal strength — how loud what is
  arriving actually is — read as the **RMS of the time domain**, mapped in **decibels** (-55 dBFS to -3
  dBFS, because a linear needle sits crushed against the left stop for everything short of a peak) and
  damped in the meter's own loop with a fast rise and slow fall; the needle therefore carries no CSS
  transition, an easing on top of a per-frame position only smearing it. Where no analyser can be had,
  the display falls back to a shaped noise animation and the meter to the tuned station's bitrate.
- **The dial goes fullscreen on F,** opening a *second* `RadioTuner` in a fixed overlay rather than
  moving the inline one, so the page underneath is left exactly as it was; the overlay covers the
  viewport by itself and the browser's fullscreen API is asked for on top of that, so a refusal costs
  nothing but the browser's own chrome. Fullscreen the cabinet is anchored to the **foot** of the
  screen and the dial window stays short, rather than growing tall enough to fill the height with empty
  glass. Three details keep it from colliding with the player: the dial **stops** the left/right keys
  rather than only preventing them, since those seek the loaded track; F and Escape are ignored while a
  filter input or select has focus, and any modifier combination is passed through; and the view's
  unmount cleanup exits fullscreen only when the overlay's own host is the fullscreened element,
  because the player may be fullscreened instead and it outlives the view.
- **The crawl walks a cycle in three stages.** Because a pass takes hours, what it fetches first is
  what the user has: `RadioSyncPhase` records the stage on the state row beside the offset (which
  restarts at 0 for each), with `PRIORITY_COUNTRY` taking one country's stations most-listened first
  (`radioPriorityCountry`, "Greece" by default), `PRIORITY_POPULAR` taking the directory's most-listened
  stations up to `radioPriorityStations` (5,000 by default), and only then `FULL` walking everything.
  Either stage is switched off by blanking or zeroing its setting; a disabled stage is stepped over
  before any request is made, and a capped stage's last batch asks for exactly the remainder so it
  cannot overshoot. The **full** stage is ordered by **name, not popularity**, because it spans hours
  and votes and clicks change constantly, so paging through a shifting ordering would skip stations and
  revisit others — the priority stages can afford `clickcount` precisely because each is only a batch
  or two long.
- **The catalogue's bulk writes are batched JDBC statements,** not repository saves, and that is not an
  optimisation to undo lightly: both entities have identity primary keys, and Hibernate cannot batch
  inserts of an identity-keyed entity because it must run each one on its own to read the generated key
  back, so mirroring 62,000 stations meant a statement per station and, on a spinning disk, minutes per
  batch. The crawl hands `RadioCatalogueWriter` flat `RadioStationRow` values, the writer asks in one
  query which uuids it already holds, and issues one batched `UPDATE` and one batched `INSERT` inside a
  single transaction.

---

## [4.0.1] — 2026-08-06

- **The whole radio feature is opt-in and off by default,** gated on the boolean `radioEnabled` setting
  (`SettingService.isRadioEnabled()`): it is the one part of the application that reaches a third party
  on its own — the catalogue crawl runs in the background whether or not anybody opens the page — so an
  installation that only wants its own library must make no request on its behalf. Being off is
  therefore not merely a hidden page: `RadioStationSyncService` refuses to run at every point a run can
  begin (the scheduled tick, the empty-catalogue startup fill, and `start()` — deliberately *not*
  inside `runBatch()`, so a caller that drives a batch on purpose still can), and every `/api/radio`
  endpoint answers **404** through `RadioController.requireEnabled()`, the same answer an unknown
  station gets, so a disabled installation reveals nothing about the catalogue behind it. The setting is
  read per request and per tick, so switching it on takes effect without a restart. On the client the
  flag is read once through `navVisibilityStore.loadRadioEnabled()` and gates the topbar link and home
  card (treating "not loaded yet" as *hidden*, since defaulting to visible would flash a Radio link
  onto most installations), the `meta.requiresRadio` route guard, and `musicPlayerStore.init()`, which
  skips restoring a stored station while radio is off.
- **The catalogue is mirrored into the database** rather than queried per request, because the
  directory is a volunteer-run pool of mirrors that is regularly slow or down: querying it per request
  meant a timing-out mirror surfaced as a broken page and put a remote request behind every keystroke
  of a search. `RadioStationSyncService` mirrors the **whole** directory — about 62,000 stations — but
  **paced**: one batch of `radioSyncBatchSize` stations (2,000 by default, so ten requests two seconds
  apart) every `radioSyncIntervalMinutes`, with the offset it reached persisted in the single-row
  `RadioSyncState` table so a batch resumes where the last one stopped, including across a restart. A
  full pass is about 31 batches; the next cycle begins `radioSyncCycleHours` after the last finished.
  Until a cycle has ever completed the batches run a minute apart instead, so a fresh installation has
  the whole catalogue within the hour rather than in a day. A batch is serialised on a single-thread
  executor behind an `AtomicBoolean`, `POST /api/radio/sync` (admin-only) brings the next one forward,
  and `GET /api/radio/sync` reports progress, which the Radio page and the Library page's new Radio
  Catalogue card show as a bar.
- **Two details of the crawl are load-bearing.** The offset advances by the **window asked for** rather
  than by the number of rows that came back, since the directory can return fewer for a window and
  advancing by the smaller number would re-read the same rows for ever. And stations that vanish from
  the directory are deactivated **at the end of a cycle** (`deactivateNotSeenSince(cycleStartedAt)`),
  never at the start: a paced crawl takes hours, and deactivating up front would leave the catalogue
  empty for all of them. They are **deactivated, never deleted**, so a favourite pointing at one
  survives. A batch that fails leaves the offset where it was, so the next one retries that window while
  the catalogue goes on serving everything it already holds.
- **The schema and the filters.** `RadioStation` mirrors the directory's JSON with three local
  additions: `active`, `dateSynced`, and the `tagsIndex`/`languagesIndex` columns, which hold the
  comma-separated tags and languages normalised to a lowercase, comma-**delimited** form (`,rock,pop,`)
  so an exact "has this tag" filter is a plain `LIKE '%,rock,%'` — the delimiters are what stop `rock`
  matching `rockabilly`, and truncating an overlong index cuts back to the last complete value. The
  filter dropdowns come from `RadioTerm` (kind/value/count), rebuilt from scratch rather than
  reconciled: countries group in SQL, but tags and languages are lists inside one column, so they are
  split and counted in Java — which is why the counts are precomputed rather than worked out per
  request. A rebuild rewrites tens of thousands of rows, so it runs when a cycle closes, when no
  vocabulary has ever been built, and every tenth batch in between. Paging is by row offset through
  `OffsetPageable`, since a short page from a filtered query leaves the next offset off any page
  boundary.
- **Favourites are per-user,** like playlists and the play queue: a `RadioFavorite` row links one
  `User` to one `RadioStation`, unique on the pair, and `RadioStationQueryService` builds every listing
  for one user — marking their favourites and, unless the listing is already restricted to them,
  ordering them first with a `CASE` over the user's favourite ids in the `ORDER BY` (the ids are
  already loaded to mark them, so an `IN` beats a correlated subquery; `RadioControllerTest` exercises
  it against Derby, which is the database that would refuse it). The toggles are granted to any
  authenticated user in `SecurityConfig`, a favourite being per-user data managed by its owner.
- **Now playing.** The title is broadcast **inside the audio stream** as ICY metadata, so
  `RadioNowPlayingService` opens its own short connection with an `Icy-MetaData: 1` header, reads the
  `icy-metaint` interval, skips that many audio bytes to the first metadata block, and parses
  `StreamTitle='Artist - Song'` out of it, closing the connection as soon as a title is in hand — a few
  tens of KB, not a running stream. It is another thing the frontend structurally cannot do: a media
  element exposes none of this to the page, the request needs a header a browser will not set, and the
  station would refuse the cross-origin read anyway. Up to three blocks are read, since a station that
  only emits on change can send an empty first one; results are cached for 10s so several listeners
  share a single read; and the station is resolved from its **uuid** through the local catalogue rather
  than taking a URL from the caller. The split of `Artist - Song` is on the first " - " **with spaces**,
  because a bare hyphen is far more often part of a name (`Blink-182`); a title that does not follow
  the convention is kept whole rather than guessed at. Everything about it is best-effort and answers
  200 with "nothing known" rather than an error. The client polls it every 20s but **only while a
  station is both loaded and playing**, and leaves the last known title on screen when playback stops.
- **A station is played by the same app-shell `MusicPlayer` the library uses.** The Radio view holds no
  media element and only hands the station to `musicPlayerStore.playStation`, so a stream keeps playing
  across navigation exactly as a track does, minimizing into the same bottom bar. The store keeps the
  station in a `station` ref **beside** `current` rather than squeezing it into a `MusicItem`, and
  while it is set it **takes precedence**: the player streams the station and the loaded track is left
  untouched underneath, so stopping the radio leaves that track ready for the next press of Play.
  Playing any track clears the station, so exactly one thing is ever playing, and the player switches
  between the two by watching a single `mediaKey` (`radio:<uuid>` or `track:<id>`) rather than the
  track id, so the one `<video>` element is never unmounted across the swap and keeps its iOS autoplay
  permission.
- **Everything track-shaped is gated on `isRadio`** and hidden or inert on a station — the seek bar (a
  live stream has no duration; a connection indicator stands in its place), rating, the metadata edit
  fields, play counting, shuffle, the play queue, and the playlist shortcuts. The fullscreen tag
  overlay has a radio form of its own (`--radio`) carrying only the bottom block — station name,
  country, and genres on the left, "Radio", stream quality, and the clock on the right — with the two
  overlays mutually exclusive, because the loaded track stays in place underneath a stream and would
  otherwise draw its rating, position, and tags on top of the station's. The keyboard shortcuts fall
  back to the handful that mean something live (Space, N/P, volume, F, H), and the help overlay lists
  that subset.
- **Radio is play/stop, not play/pause:** a paused live stream goes on buffering and comes back stale,
  so stopping drops the element's source outright to close the connection and starting re-assigns it to
  reconnect at the live edge — which is also why the source is assigned on every start rather than
  bound in the template. A dead stream is reported rather than skipped past, since the listed stations
  are a browse order, not a playlist. Previous/Next step through a snapshot of the view's listed
  stations (`setStationSequence`), the same way the track sequence snapshot outlives the Music view,
  and the store persists the whole station to `localStorage` (`radioCurrentStation`) — a station has no
  id to re-fetch it by — restoring it as loaded-but-not-started.

---

## [4.0.0] — 2026-08-05

- **The Radio view.** A new page browsing internet radio streams from the public radio-browser.info
  directory: a station search filtered by name substring plus exact country, tag, and language, ordered
  by clicks, votes, name, or bitrate, with the three filter vocabularies populating its dropdowns. The
  exact-match flags on country/tag/language are deliberate — those values come from the directory's own
  lists, so a substring match would only pull in unrelated neighbours ("Ireland" matching "Northern
  Ireland"). A stream is not library content: it has no `MediaFile`, tag, hash, thumbnail, or play
  count.
- **The directory is proxied rather than called from the browser,** for three reasons that all rule the
  frontend out: its mirror list is served over plain HTTP, which a page loaded over HTTPS refuses as
  mixed content; its usage policy asks for a descriptive `User-Agent`, which a browser will not let a
  page set; and the documented way to find a mirror is a reverse DNS lookup a browser cannot perform.
  `RadioBrowserService` discovers the mirror pool once from `all.api.radio-browser.info` over HTTPS,
  caches it for six hours in a **randomised** order so instances spread their load rather than all
  hitting whichever mirror is listed first, and retries the next mirror — rotating the failed one to
  the back of the pool — when one does not answer; only when every attempt fails does it throw. Two
  things about building that pool are not optional, both learned from it being useless in practice: the
  server list returns **one entry per IP address**, so a host with an A and a AAAA record appears twice
  and an un-deduplicated pool "fails over" from a dead mirror to itself; and discovery can legitimately
  return a single host, leaving nothing to fail over to, so the hard-coded fallbacks are **appended to
  whatever was discovered** rather than used only when discovery fails outright.
- **`POST /api/radio/stations/{id}/click`** reports a started stream back to the directory as its usage
  policy asks. It is granted to any authenticated user in `SecurityConfig`, since starting a stream is
  an ordinary listening action rather than an administrative one, and it is fire-and-forget on both
  sides, always answering 204, because playback has already begun by the time it runs.

---

## [3.13.8] — 2026-08-05

- **Video-frame screenshots from the music player.** While a music video is playing, the player's new
  **Z** shortcut saves the frame currently on screen: `MusicPlayer` draws the `<video>` onto an
  off-screen canvas at the video's own pixel dimensions (so the capture is the full-resolution frame,
  not the size the stage happens to be), encodes it as a JPEG, and posts it to the new
  `POST /api/uploads/screenshot` (`MediaUploadController` → `ScreenshotUploadService`). The frame is
  written into the writable `PHOTOS` library folder under a `year/month/day` directory, named
  `<artist> - <title> (<position>).jpg` so repeated captures of one video are told apart.
- **Screenshots are deliberately not indexed.** Unlike a music upload, the screenshot path writes the
  file and nothing else — no content hash, no `MediaFile` row, no tag scan, no `Folder` record —
  because a screenshot is a file the user asked to keep, not library content. The requested name is
  reduced to a bare, safe name (so it can never escape the destination), given an image extension when
  it lacks one, and never overwrites an existing file (a `(1)` suffix is appended). Non-images, empty
  files, uploads over 25 MB, and a missing writable photos folder are each rejected with an
  explanatory 400. The shortcut is a no-op for an audio track, and the outcome — confirmation or
  failure — is reported on the player stage, which now shows its flash message outside fullscreen too.
- **Tests.** A new `ScreenshotUploadServiceTest` integration suite covers the dated filing, the
  never-indexed guarantee, name sanitizing, collision suffixing, and every rejection.

---

## [3.13.7] — 2026-08-04

- **Regenerate Music Video Thumbnails.** For music videos whose thumbnail an earlier cleanup took, the
  Library page has a new button that starts `MusicVideoThumbnailJobService` through
  `POST /api/thumbnails/music-videos/regenerate` (with `.../cancel` and `GET .../status`, polled once
  a second and resumable by any page that opens later, exactly like the rehash and the tag import). The
  job walks the music videos carrying no thumbnail in ascending id order with a forward cursor
  (`findMusicFilesWithoutThumbnailAfterId` — a file whose frame cannot be decoded still has none, so
  offset pages would hand it back for ever) and calls `ThumbnailService.generateMusicVideoThumbnail` on
  each, reporting generated/skipped/missing/failed counts plus elapsed time and an ETA through
  `MusicVideoThumbnailJobState` / `MusicVideoThumbnailJobResponse`.
- **Guards and cancellation.** It refuses to start while a scan is running, since the scan makes these
  thumbnails itself, and cancellation is honoured between batches: the job only ever visits music
  videos that have no thumbnail, so a cancelled run is simply started again and carries on.

---

## [3.13.6] — 2026-08-04

- **Music files are renamed from their tags.** Draining a hash-refresh queue entry is now also where a
  music file's name is brought in line with the tags just written onto it: before the hash is
  recomputed, `MusicFileRenameService.renameFromTags(mediaFile)` renames the file to
  `<artist> - <title> (<year>)` plus its original extension — dropping the year part and its
  parentheses when neither the track's `Album` nor its `MusicTag` knows one — and stores the new name
  on `MediaFile.filename`. It happens here rather than on the edit request for the same reason the
  hashing does: by the time an entry is due, the user's edits have settled.
- **The rename is deliberately conservative,** because renaming a user's own files is destructive in a
  way a database edit is not: a file with no `MusicTag`, or whose tag names no artist or no title, is
  left untouched; illegal filename characters in a tag value become spaces; an existing file of the
  target name is never overwritten (a `(1)` suffix is appended, exactly as an upload does); and the
  file is moved **first** with the row written second, so a row that cannot be written moves the file
  back and disk and database never disagree. A failed rename is a `WARN` and does not cost the file its
  hash refresh — the two are independent repairs of the same edit, and a rename does not change the
  file's bytes. Covered by a new `MusicFileRenameServiceTest`.
- **Smart playlists export and import.** The smart-search modal can now export the playlist being
  edited to a `.json` file (`{ name, definition }`, with the name sanitized into the download filename)
  and import one back, accepting either that envelope or a bare rule group. Import validates the parsed
  structure before applying it, so a malformed or unrelated JSON file is reported rather than silently
  replacing the builder's contents.

---

## [3.13.5] — 2026-08-03

- **Deferred hash refreshes after a file is rewritten.** A hash describes the exact bytes it was taken
  from, so anything that rewrites a file invalidates it. `MediaHashService.refreshHash(MediaFile)`
  recomputes one file's hash with the kind-appropriate strategy and stores it when it differs,
  returning a `HashRefreshResult` (`UPDATED` / `UNCHANGED` / `MISSING` / `FAILED`); a file that is gone
  or unreadable keeps its stale hash, since an out-of-date hash is still better than none. That
  recomputation is never done on the request that rewrote the file — reading a file back the instant it
  was written makes an edit as slow as the file is big. Instead `MusicController` calls
  `HashRefreshQueueService.enqueue(mediaFile)` after `MusicFileTagWriter.writeTags` or `writeArtwork`
  reports that it actually committed a write (both now return a boolean for exactly this reason), and
  the reading is left to the new `HashRefreshJobService`.
- **The queue itself.** One `HashRefreshQueueItem` row per file (`media_file_id` unique, carrying a
  `queuedAt` timestamp): re-editing a file that is already queued only moves its `queuedAt` forward, so
  a burst of edits to one track is followed by a single rehash after the last of them. Two details are
  load-bearing: the re-queue is an **`UPDATE` first** (`touch`) with an insert only when no row was
  updated, since a read-then-insert lets two edits of the same file both insert and violate the unique
  key; and the write runs in a **separate transaction** (`HashRefreshQueueWriter`, `REQUIRES_NEW`) with
  the failure caught outside it, since a failed insert inside the edit's own transaction leaves the
  persistence context holding an un-insertable entity and turns an otherwise successful request into a
  500.
- **The job.** `HashRefreshJobService` is a `SchedulingConfigurer` running every
  `hashRefreshJobIntervalMinutes` (10 by default) that processes only entries older than
  `hashRefreshDelayMinutes` (5 by default) — both editable from the settings page. A run drains in
  batches of 50 until nothing is due, a 120-second budget is spent, or a scan starts, and is suppressed
  entirely while a scan or a library-wide rehash is running. An entry is dropped once handled whatever
  the outcome, so the queue always drains rather than retrying for ever.
- **Regenerate Hashes (library-wide rehash).** For files rewritten _outside_ the application, the
  Library page's new **Regenerate Hashes** button starts `MediaRehashJobService` through
  `POST /api/hashes/rehash` (with `POST /api/hashes/rehash/cancel` and `GET /api/hashes/rehash/status`,
  polled once a second and resumable by any page that opens later). It walks every `MediaFile` in
  ascending id order with a forward cursor (`findAllAfterId`, since a rehashed row still matches the
  query it was found by) and refreshes each one, reporting processed/updated/unchanged/missing/failed
  counts plus elapsed time and an ETA through `RehashJobState` / `RehashJobResponse`. It refuses to
  start while a scan, an import, or a deep clean is running, and cancellation is honoured between
  batches — the job is idempotent, so a cancelled run is simply started again.
- **A deleted thumbnail no longer becomes a permanent broken image.** `GET /api/thumbnails/{id}` now
  answers a bare **404** instead of throwing `NotFoundException`: a thumbnail is a regenerable cache
  artifact, and a page still holding an id the cleanup has just deleted would otherwise write one
  admin-visible error row per broken image tag. Alongside it, `ThumbnailService.clearStaleReferences`
  (and `clearStaleFolderReference` for folders) clears a media file's FK to a thumbnail record that no
  longer exists — every regeneration path short-circuits on a non-null thumbnail id, so without this a
  file would point at a dead id for ever — and the photo, video, music, and folder listing endpoints
  now run their pages through it, letting normal on-demand generation take over on the next view. The
  frontend views hide an image that fails to load rather than showing a broken tag.

---

## [3.13.4] — 2026-08-02

- **Tag import.** `POST /api/imports/tags` (multipart, `kind` + `file`, admin-only) uploads an export
  file whole and applies it with a background job (`MediaImportJobService` / `MediaImportService`),
  with `POST /api/imports/tags/cancel` and `GET /api/imports/tags/status`. An import of a large library
  reads and writes for minutes, so tying it to a page staying open made it the browser's job to keep
  going; started this way it is the application's, and the page only watches — including a page opened
  long after the import began. Entries are matched by **content hash**, so paths and database ids need
  not agree between installations.
- **What an import restores,** all reported as live counters (`ImportJobResponse`): matched and
  not-found entries, music tag fields and each item's `dateAdded`, per-user ratings/views/last view
  date, playlists created and playlist memberships added (per user, by name), and album covers restored
  from the export's covers directory — an existing cover is kept when it is at least as good, so a
  re-run never downgrades artwork. Per-user entries naming a user that does not exist here are counted
  and skipped.
- **Batch writing.** The new `BatchWriter` utility (`utilities/`) flushes accumulated entities in fixed
  chunks, so importing a large library writes in bounded batches instead of one ever-growing
  persistence context; the import services and their repositories were reworked around it.
- **`dateAdded` carried across.** Export and import DTOs now carry when an item was first indexed. It
  is the one scan-derived value a rescan cannot recover — a rescan stamps every record with the moment
  it ran — so it only survives by being exported.
- **Oversized uploads answer 413.** `GlobalExceptionHandler` now handles `MaxUploadSizeExceededException`
  with a 413 and a readable message instead of a 500 stack trace; the servlet container rejects such a
  request before any controller sees it. `SettingConfiguration` no longer declares a
  `MultipartConfigElement`, since that bean silently overrode the
  `spring.servlet.multipart.max-file-size` / `max-request-size` properties — the limits now live in
  `application.properties` alone.
- **Export refinements.** `MusicExportItemDTO` and `PhotoLibraryExportItemDTO` gained the file `path`
  and the per-user `lastViewDate` for interoperability with third-party tools, and dropped the
  thumbnail paths, which are regenerable and worth nothing across installations.

---

## [3.13.3] — 2026-07-29

- **Tag export.** `GET /api/exports/tags?kind=PHOTOS|MUSIC|MUSIC_VIDEOS` (`MediaExportController` →
  `MediaExportService`) returns every hashed item of one library kind keyed by its **content hash**,
  which is what makes an export portable: a later import matches by hash rather than by a path or a
  database id, neither of which survives a move to another installation. The envelope
  (`MediaExportDTO`) names the kind and reports what was left out — records with no hash yet, and
  records whose hash another item already holds (duplicate copies of the same content).
- **Only what a rescan cannot recover is exported.** `PhotoLibraryExportItemDTO` (images and videos of
  a photos library, told apart by `mediaKind`) carries identity plus the per-user ratings and views;
  `MusicExportItemDTO` adds the editable tag fields (title, artist, genre, album, year, live, source)
  and the album's cover path. Dimensions, capture date, camera, GPS, codec, duration, file size, and
  MIME type are all deliberately omitted — a rescan regenerates them, so re-importing them could only
  overwrite fresher values with stale ones. Per-user rating, view count, and playlist memberships are
  carried in `UserExportDataDTO`, keyed by username.
- **Export is admin-only.** An export carries every user's ratings and playlists, not only the
  caller's, so `SecurityConfig` restricts `/api/exports/**` to `ADMIN` rather than leaving it to the
  blanket rule that lets any authenticated user issue a GET.
- **Library page layout.** `ScanProgress` gained slots for library-wide actions and notices, so the
  Library page hosts the export/import controls (and the later hash and thumbnail jobs) in one place
  rather than scattering buttons; a library-kind modal picks which kind an export or import applies to.

---

## [3.13.1] — 2026-07-29

- **Nested write folders no longer double-index.** Only `READ` library folders are scan roots
  (`LibraryFolderService.getScannable()` / `findScannableForPath`); a `WRITE` folder is an upload
  destination and is never walked as a root of its own. A write folder nested inside a read folder
  (`.../music/download` inside `.../music`) previously had every file below it indexed twice — two
  `MediaFile` rows, two tags, and two folder trees per track. Files in a nested write folder are still
  indexed, as part of the read folder that contains them, and `MediaUploadService` records an upload
  against the enclosing read folder so the upload and the next scan agree on one owner. Covered by new
  `MediaScanServiceWriteFolderTest` integration tests.
- **Legacy import tool removed.** The one-off legacy SQL migration tool added in 3.9.2 has served its
  purpose and is gone: `LegacySqlImportService`, `LegacyMusicImportService`, `LegacyImportController`,
  `LegacyMusicMetadataDTO`, the frontend API module and its two Library page buttons, the
  `findAllByMediaFileFilenameIn` repository query, and the tool's `multiLanguage.xml` entries. The
  generically-useful helpers it introduced stay: `CoverService.resolutionOf`/`delete`,
  `AlbumService.attachOrUpgradeCover`, and `PlaylistService.findOrCreate`.

---

## [3.13.0] — 2026-07-29

- **Music upload into the library.** The Library page can now upload new music into a writable library
  folder through `POST /api/uploads/music` (`MediaUploadController`, multipart, admin-only), handled by
  the new `MediaUploadService`. Each file is routed by its own media kind to the `WRITE` folder whose
  `LibraryKind` matches — an audio track to the writable `MUSIC` folder, a video to the writable
  `MUSIC_VIDEOS` one — and is rejected with an explanatory message when no such folder is configured or
  the extension is not a recognised audio/video format. Files are accepted or rejected individually, so
  one rejection never holds back the rest.
- **Upload filing layout.** Within the destination folder a file is filed under `year/month/day` taken
  from the upload date — read once when the request starts and reused for the whole batch, so an upload
  running across midnight cannot split an album across two day directories — and an audio track
  additionally under an `artist - album (year)` directory read from its own tags, so every track of one
  album lands in one directory and resolves to a single `Album` row. Because the album must be known
  before the destination is chosen, the upload is staged under a dot-prefixed `.parrot-upload-tmp/`
  inside the destination folder (invisible to the scan), read there, and then moved. A submitted name is
  reduced to its last path segment so it can never escape the destination, and an existing name is never
  overwritten — a numeric suffix is appended instead.
- **Uploads are indexed exactly as a scan would.** A stored file is hashed when `generateHashesOnScanning`
  is enabled, saved as a `MediaFile`, tagged by the same `MediaTagScanner` the scan would dispatch it to
  (creating the album, cover, or music-video thumbnail), and its destination directory is recorded so the
  next scan skips it.
- **`MediaKindResolver`.** What counts as media at all is now decided in one place for both the scan and
  the upload (`utilities/MediaKindResolver`), which owns the shared image/video/audio extension sets.
- **Tests.** A new `MediaUploadServiceTest` integration suite covers routing, filing, rejection, and
  indexing.

---

## [3.12.8] — 2026-07-28

- **Stale error notifications cleared at login.** Starting a new session dismisses any error
  notifications left over from the previous one (`authStore`).
- **Smart search defaults.** The smart-search rule builder reorders its string operators and picks a more
  useful default field for a newly created rule.

---

## [3.12.7] — 2026-07-27

- **Album cover upload.** An album cover can be uploaded by hand from the Music view's edit modal:
  `POST /api/music/albums/{albumId}/cover` takes a multipart image (rejecting a non-image, an empty file,
  or one over 20 MB with a 400, and an unknown album with a 404) and `AlbumService.replaceCover` writes
  it, then deletes the superseded `Cover` row and file. It is the most permissive of the three attach
  modes — a user's explicit pick always wins over both the scan's and the resolution-based one. Because
  the cover hangs off the `Album`, one selected track sets it for the whole album, and the modal offers
  the upload only when every selected track is an audio track of the same album. With `editTagsOnFiles`
  enabled, `MusicFileTagWriter.writeArtwork` also embeds the new JPEG as front-cover artwork in **every**
  audio track of the album, and the endpoint reports how many files it reached as `taggedFiles`.
- **Hold-to-seek touch gesture.** Press and hold the player's skip controls to seek continuously, for
  touch devices where a scrub bar is awkward.
- **Configurable seek step.** A new setting defines how far the player's seek controls jump, read through
  `SettingService` and mirrored into the player store.

---

## [3.12.6] — 2026-07-26

- **Column sorting wins over album clustering.** `musicOrder` in `MusicController` now decides its primary
  sort key from whether the request carries a `sort` parameter at all. With **no** column chosen the list
  stays clustered by album, so an album's tracks never interleave with others'. With a column chosen —
  including date added, whose header click is therefore no longer a no-op — the primary value is the
  track's own value and the clustering is dropped, so one big album can no longer drag all of its tracks
  to wherever its top one sits. The clustering keys stay on as tiebreakers, so tied tracks still read
  album by album in sleeve order, and `GET /api/music/{id}/page` mirrors the same grouped/ungrouped choice.
- **A content hash now lives in exactly one place.** The duplicate `hash` column was dropped from
  `MusicTag`; `MediaFile.hash` is the single home for it, and `MusicDetailDTO` reads it through the tag's
  linked media file, exactly as the photo and video DTOs do. Since `ddl-auto=update` only ever adds schema
  objects, the new `MusicTagHashColumnMigrationRunner` issues the `ALTER TABLE music_tag DROP COLUMN hash`
  at startup — checking the database metadata first so a fresh schema and an already-migrated one are both
  no-ops, and swallowing failure as a `WARN` rather than aborting startup. It is the pattern to copy for
  any future column removal.
- **Deleting a music track cleans up after itself.** `PhotoService`'s deletion path now handles music
  tracks and every record hanging off them.
- **Album covers in the duplicates modal.** The Manage Duplicates modal never tries to generate a
  thumbnail for an audio track, which has no image to extract and would only fail. `DuplicateFileDTO`
  instead carries the `coverId` of the track's album — resolved in one batched query per page — and the
  modal renders that cover, falling back to a generic music icon when there is none.
- **Playlist removal feedback.** Removing a track that is not in the selected playlist now says so
  ("Not in playlist"), rather than failing silently.

---

## [3.12.5] — 2026-07-26

- **Server-side playlist pagination and sorting.** `GET /api/playlists/{id}/tracks` is now paginated by
  `page`/`size` and orders over the **whole** playlist by an optional `sort`/`dir` (reusing
  `MusicQueryTranslator.order`, combined with a correlated `EXISTS` over the playlist's entries since a
  `PlaylistTrack` has no association to `MusicTag`), defaulting to playlist order. `GET /{id}/random`
  draws a random position server-side for the player's shuffle, since the client holds only one page. In
  the Music view the pager now walks a whole loaded playlist and a header sort orders all of it rather
  than only the loaded rows.

---

## [3.12.4] — 2026-07-20

- **Range requests always answer 206.** `MusicController` and `VideoController` return
  `206 Partial Content` for every range request rather than occasionally buffering a whole file into
  memory to answer with a 200.
- **Smart search handling and pagination fixes** in the Music view.

---

## [3.12.3] — 2026-07-19

- **PWA support.** A service worker, web manifest, and app icons make ParrotApp installable, with offline
  handling for the app shell and continued playback handling in `MusicPlayer`.

---

## [3.12.2] — 2026-07-19

- **Playback stall reporting removed.** The stall-reporting endpoint and its client logic (added in
  3.11.3) are gone; the player's start watchdog timing was adjusted instead.

---

## [3.12.1] — 2026-07-18

- **Media Session API integration.** The player registers with the OS media session, so lock-screen and
  notification-shade controls, and the track metadata shown there, work as expected.

---

## [3.12.0] — 2026-07-18

- **Smart playlists.** A `SmartPlaylist` is a saved search rather than a fixed track list: a rule tree
  (field, operator, value, nested AND/OR groups) is stored per user as a JSON definition and evaluated on
  demand by the new `MusicQueryTranslator`, which turns it into a JPA `Specification`, so it always
  reflects the current library. `SmartPlaylistController`
  (`/api/smart-playlists`) is per-user CRUD like `PlaylistController`, and the frontend gained a rule
  builder (`MusicSmartSearchModal`, `SmartRuleGroup`, `smartSearch.ts`) reachable from the Music toolbar.
  Covered by a `SmartPlaylistControllerTest` integration suite; `DeepCleanService` clears the rows.
- **Smart search honours the toolbar filters.** A smart search additionally narrows by the kind, source,
  and rating filters.
- **App shell during backend outages.** `App.vue` degrades more gracefully when the backend is
  unreachable.

---

## [3.11.3] — 2026-07-18

- **Fine volume stepping near silence.** The player's volume control takes smaller steps at low volumes,
  where a linear step is perceptually huge.
- **Playback stall reporting.** Tracks that fail to start are logged as unplayable. (Removed again in
  3.12.2.)

---

## [3.11.2] — 2026-07-18

- **A music video keeps its album name.** `MusicTag.albumName` is the string counterpart of the `album`
  association and is now populated on every tag, mirroring `year`. A music video never gets an `Album`
  record, so `MusicDetailDTO` falls back to the tag's own `albumName` when `album` is null — which is how
  both single and batch edits can change a music video's album, where it was previously dropped for want
  of an `Album` row.

---

## [3.11.1] — 2026-07-17

- **HTML entities decoded in music metadata.** Titles, artists, albums, and genres carrying escaped HTML
  entities from a file's embedded tags are decoded for display and editing throughout the Music view
  (`functions/htmlEntities.ts`), instead of showing raw `&amp;`-style text.
- **Mobile layout for the Music page.** The player and content stack vertically on narrow screens, the
  toolbar's filters and action buttons reflow, and the track list refines its grid and hides
  non-essential columns for readability.

---

## [3.11.0] — 2026-07-16

- **Persistent, app-wide music player.** The music player is no longer owned by the Music view. A single
  `MusicPlayer` instance is mounted once at the app-shell level (`App.vue`) and driven by a new shared
  `musicPlayerStore` (`frontend/src/functions/musicPlayerStore.ts`). On the Music route it renders full
  size as that view's left column; on every other route the same instance restyles into a fixed,
  minimized bottom bar. Because it is only ever restyled — never moved in the DOM or unmounted —
  playback and its live media element **survive navigating away from the Music view**. The store owns all
  playback state: the loaded track, the ordered sequence snapshot the view hands it (so Previous/Next
  keep working after the view unmounts), the shuffle history, the active album playlist, the mirrored
  play queue, and the shuffle filters mirrored from the toolbar.
- **Track and shuffle persistence across sessions.** The store persists the loaded track id and the
  shuffle toggle to `localStorage` (`musicCurrentTrack`, `musicShuffle`). A reload restores the last
  track as loaded-but-not-started (browsers block autoplay without a user gesture, so it is ready for the
  next press of Play rather than auto-starting) and restores the shuffle mode.
- **Queue polling.** The per-user play queue is re-fetched every 5s, since there is no push channel, so
  another tab or device's queue changes are noticed by the current tab.
- **Field filtering resets, then restores, the toolbar filters.** Clicking a Title/Artist/Album/Genre/Year
  cell to field-filter the track list now resets the source and rating toolbar filters so the field search
  runs across the whole library instead of narrowing the already-filtered list (the kind filter is kept);
  cancelling the field filter — a repeat click or the toolbar's clear button — restores the source/rating
  filters to what they were before.
- **Mobile & touch improvements.** A mobile Play/Pause button was added with touch-friendly styling, and
  the fullscreen touch controls now key off pointer capability for more reliable behaviour on touch
  devices.

---

## [3.10.2] — 2026-07-16

- **Queue & playlist feedback notifications.** Adding to or removing from the play queue or a playlist now
  raises a confirmation notification, with the messages localised through `multiLanguage.xml`.

---

## [3.10.1] — 2026-07-15

- **Tracks clustered by album in the list.** Whatever the active sort, the track list now **always groups
  an album's tracks together** so they never interleave with other albums' tracks (`musicOrder` in
  `MusicController`), and `GET /api/music/{id}/page` computes a track's page from that same clustered
  order.
- **Music-video tags via the MP4 `ilst` atom.** `MusicTagScanner` and `MusicFileTagWriter` fall back to
  the low-level `Mp4TagReader`/`Mp4TagWriter` to read and write a music video's title/artist/genre
  directly from the standard MP4 `ilst` atom, which jaudiotagger's high-level API refuses
  (`CannotReadVideoException`); non-MP4 music videos still fall back to the filename, best-effort.
- **Bulk-delete confirmation and larger pages.** Deleting tracks (and their files) now shows a
  confirmation message, the track fetch page size was raised from 50 to 500, and a loading overlay covers
  page transitions.
- **Robustness & polish.** Error toasts were restyled and repositioned; the "current song not matching the
  active filters" case is handled gracefully; `[` and `]` are allowed literally in query strings for
  filtering; and audio/video media-element handling and double-tap-to-zoom were tuned for more reliable
  fullscreen touch control and autoplay.

---

## [3.10.0] — 2026-07-14

- **Improved fullscreen player.** Fullscreen gained overlay transport controls and a toggle, vinyl-mode
  stage styling, aspect-ratio-consistent disc sizing, and mobile touch controls (double-tap to reveal
  controls, swipe navigation).
- **Playlist keyboard shortcuts & cycling.** New A/D/E shortcuts add to, remove from, and cycle the
  toolbar's selected playlist, with feedback messages and translations; a track-details toggle overlay was
  added and localised. Shortcuts now key off `KeyboardEvent.code` for layout-independent handling.
- **Toolbar filter reset & persistence.** A reset-filters control clears the toolbar, filter state is
  persisted to `localStorage`, and applying a click-a-cell field filter resets the toolbar filters.
- **Bulk delete.** The track list gained multi-select bulk deletion of tracks and their files.

---

## [3.9.3] — 2026-07-13

- **Column sorting for the track list.** Tracks can be sorted by clicking a column header — title, artist,
  album, genre, year, rating, plays, or date added, ascending or descending (default date added
  descending) — implemented server-side in `MusicController` with sorting tests. The "Last played" label
  became "Date added".
- **Periodic queue polling & scan-completion refresh.** The Music view polls the play queue for real-time
  updates, and media stats refresh when a scan completes.
- **Album de-duplication safety.** Album retrieval returns a list and logs duplicates; legacy imports link
  tracks without albums and preserve existing ratings; unreadable media files get a sentinel hash so they
  are not re-hashed endlessly. A fullscreen toggle button was added to the player controls.

---

## [3.9.2] — 2026-07-12

- **Legacy metadata import.** A temporary legacy SQL import tool (`LegacyMusicImportService` /
  `LegacySqlImport`) extracts music metadata — ratings, play counts, album/year, playlist membership and
  album covers — from a previous database and applies it to matched tracks, with path normalisation, album
  source markers, year validation, and tests.
- **Rating filter & track details modal.** The Music API and UI gained a rating filter (rating value plus
  comparison operator), a read-only track details modal, fullscreen support with a details overlay, and a
  `year` field on `MusicTag`.
- **Packaging & data directories.** Build scripts package ParrotApp as a Linux AppImage and a macOS `.dmg`
  installer, and the thumbnail/cover data directories are anchored to `app.data.dir`.

---

## [3.9.1] — 2026-07-11

- **Play-count threshold setting.** A new play-percentage setting defines how much of a track must play
  before it counts as played, applied by the player's playback threshold logic.
- **Vinyl placeholder colours.** A track with no album cover shows a vinyl disc with randomly chosen
  placeholder colours.

---

## [3.9.0] — 2026-07-11

- **Music source filter.** The Music toolbar gained a **source filter** (all / regular / lossless / vinyl /
  CD rip) narrowing `GET /api/music/all` to one `MusicSource`, with a `REGULAR`-matches-null rule for
  legacy rows and an integration test for the track-page source filter.
- **Playlists.** Users can curate persistent, per-user **playlists** (`Playlist` / `PlaylistTrack`,
  `PlaylistController` at `/api/playlists`): create, rename, delete, and add/remove tracks, with the
  playlist controls grouped in the Music toolbar.
- **Navigation gating.** Sidebar/topbar navigation entries appear only when the corresponding media exists,
  and a processing-error log level was lowered from error to warn.

---

## [3.8.0] — 2026-07-11

- **Top-bar navigation.** The sidebar was replaced by a `TopBar` component carrying navigation and user
  controls, and navigation visibility follows the available media content.

---

## [3.7.3] — 2026-07-11

- **Library folder read/write modes.** Library folders gained READ/WRITE mode support in their management
  form, and the `TopBar` navigation component was introduced.

---

## [3.7.2] — 2026-07-10

- **Cover & metadata handling.** Album covers gained JPEG compatibility, the folder browser accepts an
  initial browsing path, and metadata backfill was extended to fill missing fields and album covers for
  audio tracks and photos.

---

## [3.7.1] — 2026-07-10

- **Album tiles & inline track list.** Album tiles show the release year and distinguish singles, an inline
  album track-list panel expands beneath the album panes, and filter tooltips display the specific filtered
  value.
- **Now-playing artwork.** The Music view shows album-cover backdrop artwork for the currently playing
  track and behind an album's tracks, plus a spinning vinyl mode for the album cover.
- **Album grouping on edit.** Editing selected tracks' metadata groups them into their majority album.

---

## [3.7.0] — 2026-07-09

- **Albums panes.** The Music view gained an **albums mode** with infinite scroll backed by
  `AlbumSummaryDTO`, an **album slider** (coverflow) with navigation controls, and album grouping, split
  into dedicated `MusicToolbar` and `MusicTrackList` components.

---

## [3.6.0] — 2026-07-09

- **Music-video thumbnails & source classification.** Music videos get a JCodec frame thumbnail at scan
  time, and audio tracks are classified by `MusicSource` (lossless auto-detected; vinyl/CD rip user-set).
- **Pagination & jump-to-track.** The track list gained a numbered pager, a `GET /api/music/{id}/page`
  endpoint returning a track's page index, and keyboard shortcuts for rating and the studio/live toggle.

---

## [3.5.2] — 2026-07-08

- **Cold-start playback.** Playback logic was hardened for cold starts, with improved "has next track"
  availability checks.

---

## [3.5.1] — 2026-07-08

- **Click-a-cell field filtering.** Clicking a track row's Title, Artist, Album, Genre, or Year cell adds
  one exact, case-insensitive field filter to `GET /api/music/all`, narrowing the list to that value.

---

## [3.5.0] — 2026-07-08

- **Metadata edits write back to the file (opt-in).** Correcting a track's metadata — through the
  single-track `PATCH /api/music/{id}` or the multi-select `PATCH /api/music/batch` — can now write the
  edited values back onto the music file's own embedded tags (`songName`→title, `artist`→artist,
  `genre`→genre, `album`→album, `year`→year) via the new `MusicFileTagWriter`, the write-side counterpart
  to `MusicTagScanner`'s read. A later rescan, or any external tool, therefore sees the correction rather
  than the original tags; a blank value deletes the tag from the file. The write covers audio tracks only
  (music videos are skipped) and is best-effort — an unsupported container, a missing file, or a corrupt
  tag is logged as a `WARN` and skipped so the database edit still succeeds.
- **New `editTagsOnFiles` setting.** Writing tags back to the original files is gated behind a new
  boolean setting on the Settings page, **off by default** so the user's files are never modified without
  an explicit opt-in. When off, a metadata edit changes only the database and leaves the files
  byte-for-byte untouched; when on, the file write above kicks in. Seeded by `SettingService` and read
  via `isEditTagsOnFiles()`.

---

## [3.3.0] — 2026-07-08

- **Music play queue.** The Music view gains a per-user **play queue** — an ordered list of tracks
  lined up to play next. Every track row now carries an **Add to queue** button beside its play
  button (which turns into a check and disables once the track is queued), and a new queue button
  beside the player's shuffle toggle opens a panel listing the upcoming tracks with a count badge.
  From the panel each entry can be played immediately or removed, and the whole queue cleared. While
  the queue holds tracks it **overrides both sequence and shuffle**: pressing Next (or a track ending)
  plays the head of the queue and removes it, and only once the queue is empty does playback fall back
  to the normal random/sequential order. A track can be queued at most once, so re-adding is a no-op.
- **Play-queue state and API.** The queue is persisted server-side and scoped to the user, so it
  survives reloads and follows the user across sessions. A new `QueueItem` entity (keyed uniquely on
  `(user, mediaFile)` with an ordering `position`), `PlayQueueService`, and `QueueItemRepository` back
  four endpoints on `MusicController`: `GET /api/music/queue` (read the ordered queue),
  `POST /api/music/queue/{id}` (append), `DELETE /api/music/queue/{id}` (remove — how playback advances
  past the head), and `DELETE /api/music/queue` (clear); each returns the resulting queue as an ordered
  array of `MusicDetailDTO`. Queue rows are cleared ahead of their media file in both the per-folder
  orphan cleanup and the full deep-clean reset, matching how `UserTag` is handled.

---

## [3.2.0] — 2026-07-07

- **Music shuffle playback.** The Music player gains a shuffle toggle in its transport controls that
  switches between sequential and random play of the current filter. Unlike the list's own paging,
  shuffle draws from the **whole** filtered library rather than only the page currently loaded: each
  advance (pressing Next or a track ending) asks the backend for a random track, excluding the one
  playing so a step never repeats it. Previous retraces the shuffled jumps via an in-session history,
  replaying tracks even when they came from another page. The mode honours the active
  Music / Music Videos kind filter and resets its history when the filter changes.
- **Random-track API.** A new `GET /api/music/random` endpoint returns one track chosen uniformly at
  random from the library, with optional `kind` (restrict to `MUSIC` or `MUSIC_VIDEO`) and `excludeId`
  (skip the track already playing) query parameters; it responds `404` when no eligible track remains.
- **Music player component.** The Music view's player pane (stage, transport, seek bar, and editable
  metadata fields) was extracted from `Music.vue` into a dedicated `MusicPlayer.vue` component,
  leaving the view responsible for the track list, filtering, and playback ordering. No behaviour
  change beyond the shuffle addition above.

---

## [3.1.0] — 2026-07-05

- **Application logging system.** Meaningful backend events are now persisted to a new `LogEntry`
  table and surfaced on an admin-only **Logs** page (sidebar entry, `/logs` route). Each entry has a
  level (`SUCCESS`, `INFO`, `WARN`, `ERROR`), a category, an optional action code, a message, the
  triggering username, an optional stack-trace detail, and a timestamp. Entries come from two sources:
  explicit calls to a central `LogService` for success and informational events (scan/backfill
  started and completed, thumbnail and media-hash job runs, deep clean, login/registration, and
  user / library-folder / setting changes), and a logback appender (`DatabaseLogAppender`) that
  automatically forwards every WARN and ERROR raised by the application's own loggers — so error
  cases are captured without wiring each one by hand. Writes run in their own transaction and never
  propagate a failure to the caller, so logging can never break a request.
- **Logs API.** A new `LogController` (`/api/logs`, ADMIN-only) exposes a paged, newest-first listing
  filterable by level, category, and a case-insensitive message search; a distinct-category listing
  (`/api/logs/categories`); and deletion (`DELETE /api/logs`, optionally `?olderThanDays=N`). The
  frontend page adds level/category/search filters, colour-coded level badges, expandable stack-trace
  details, pagination, refresh, and clear controls.
- **Consistent error bodies.** The global exception handler now also logs every handled error and
  standardizes the response body for `ResponseStatusException` and otherwise-unhandled exceptions to
  the same `{status, message, httpStatus, zonedDateTime}` shape (without leaking a stack trace to the
  client), and returns a proper `400` with the offending field for bean-validation failures.
- **Fixed: duplicate albums and covers.** Albums are now identified by the folder they live in plus
  their name — the `(path, name)` pair — instead of `(artist, name)`. Grouping by any artist tag split
  a single album into many rows whenever its tracks credited different or featured performers (or on
  compilations), producing duplicate album covers; folder-based grouping collapses all tracks in one
  directory under one album name into a single `Album` (and therefore a single `Cover`). The stored
  album `artist` is now informational only, and the former `UNIQUE(artist, name)` database constraint
  is removed (album resolution is serialised in `AlbumService` instead). Requires a rescan of existing
  music libraries to take effect.
- **Cover storage spread across more folders.** Album covers whose release year is known are now
  written under `covers/YYYY/BB/` (a random 1-12 bucket) instead of the current-time
  `covers/YYYY/MM/DD/HH/` path. A bulk scan previously concentrated nearly every cover into the handful
  of date folders it ran in; keying on the album year plus a bucket distributes them across many
  directories. Covers with no known year keep the date-partitioned layout.

## [3.0.0] — 2026-07-04

- **Music support (backend).** Library folders of kind `MUSIC` and `MUSIC_VIDEOS` are now scanned for
  music. Audio tracks (MP3, FLAC, WAV, AAC, OGG, M4A, WMA) and music videos (MP4, M4V, …) get a new
  `MusicTag` record holding song name, artist, genre, track time, track position, a live flag
  (defaulting to `false`), a content hash (full-file for audio, middle-slice for music videos), and —
  for music videos — pixel dimensions. Tags are read from the file's ID3/Vorbis/MP4 metadata via
  jaudiotagger; when a file has no readable tags the song name falls back to the filename and the
  remaining fields are left empty.
- **Albums and covers.** Scanning populates `Album` records (name, artist, year), deduplicated by
  `(artist, name)`, and links each track to its album. Embedded artwork from an audio track is stored
  as the album's `Cover` under a new `covers/` directory (date-partitioned like `thumbnails/`). Covers
  are one-per-album and, unlike thumbnails, are never regenerated or cleared automatically. Music
  videos rely on the existing on-demand video thumbnail instead of a cover.
- **Library-kind-aware tag dispatch.** The scan core now selects a `MediaTagScanner` from both the
  file's media kind and its library folder's kind, so a `.mp4` is tagged as a regular video in a
  `PHOTOS` library but as a music video in a `MUSIC_VIDEOS` library. Per-user ratings and plays
  continue to use the shared `UserTag`.
- **Music view.** A new **Music** page (sidebar entry, `/music` route) lists all tracks and music
  videos in a paginated table (title, artist, album, genre, year, rating, plays, last played) beside a
  player pane that streams the selected track — a `<video>` for music videos, the album cover with an
  `<audio>` element for audio — with play/pause, previous/next, a seek bar, star rating, and play-count
  tracking. Backed by a new `MusicController` (`/api/music`) exposing the list, per-track detail,
  range-supported streaming, album-cover serving, and rating/play endpoints.

## [2.1.7] — 2026-07-03

- The Geolocation map now respects the filter active in the Photos grid instead of always plotting
  every geotagged photo in the library: opening the map while browsing a folder scopes it to that
  folder's subtree, and opening it from an active search scopes it to the matching results,
  mirroring the existing Slideshow/Playlist scoping. `GET /api/photos/geo` gained optional
  `folderId` and `query` parameters with the same precedence as `GET /api/photos/batch`.

## [2.1.6] — 2026-07-03

- Fanning a small cluster open on the Geolocation map (spiderfying) now immediately generates
  thumbnails for its newly-revealed pins and refreshes the viewer's navigable photo set, instead of
  waiting until the fan is closed again.

## [2.1.5] — 2026-07-02

- Added multilingual entries for the Photos view's photo-editing and slideshow controls, so those
  labels are now fully localized rather than hard-coded English.

## [2.1.4] — 2026-07-02

- New **Help** section, reachable from the sidebar, explaining library folders and scanning, the
  Photos/Slideshow/Playlist views, and (for administrators) user and settings management. Backed by a
  new `Help.vue` view and route, with an entry point surfaced on the Home page.
- All Help content is fully localized via new `multiLanguage.xml` entries.

## [2.1.3] — 2026-07-02

- Small groups of co-located photos on the Geolocation map are no longer folded behind a single count
  badge. Clusters of at most five photos are now pulled out and their pins fanned into a ring at each
  location, so nearby photos are individually clickable without zooming in. Each marker's true
  coordinates are preserved so re-clustering on pan/zoom stays correct, and Leaflet's own spiderfy
  behaviour is left untouched for larger overlapping groups.

## [2.1.2] — 2026-07-02

- Photo locations now show a real place name (e.g. "Athens, Greece") instead of just raw
  coordinates. A new `GeocodingService` reverse-geocodes a photo's GPS coordinates via
  OpenStreetMap's Nominatim API the first time it's viewed, and caches the result on its
  `PhotoTag` (`locationName`) so later views never repeat the network call.
- The photo viewer overlays the resolved location as a pill-shaped label at the bottom-center of
  the image; the details sidebar shows it too, with the raw coordinates kept as a secondary line.
- `GET /api/photos/{id}` resolves and persists the name lazily; a failed or unresolved lookup
  simply leaves `locationName` null rather than failing the request.

## [2.1.1] — 2026-07-01

- Geolocation map images now load lazily as pins come into view, instead of generating every
  thumbnail up front, keeping large libraries responsive on the map.
- The map's photo viewer gained proper prev/next navigation across the pins visible at the
  current zoom level, and its visibility/cluster handling was reworked to keep the arrows in
  sync as the map is panned or zoomed.

## [2.1.0] — 2026-07-01

### Browsing & UI

- New **Geolocation** view mode on the Photos page plots geotagged photos on an interactive
  OpenStreetMap map (via Leaflet). Nearby photos are grouped into a single high-contrast cluster
  badge that expands as you zoom in, so a location with many photos collapses to one marker; each
  photo's thumbnail doubles as its marker, and clicking one opens the photo detail. A **Geolocation**
  toggle button switches between the folder grid and the map.
- Only photos carry GPS coordinates (videos have none), so the map shows images with both a latitude
  and a longitude recorded in their `PhotoTag`.
- Backed by a new `GET /api/photos/geo` endpoint returning lightweight `PhotoLocation` points
  (`id`, `filename`, `thumbnailId`, `latitude`, `longitude`) for every fully geotagged photo.

## [2.0.5] — 2026-07-01

- Duplicates modal now previews videos alongside photos and handles media-kind-specific actions
  when resolving duplicate groups (#39).

## [2.0.4] — 2026-06-30

- New **Clean Thumbnails** button on the Library view deletes every file thumbnail (database
  records and on-disk files) on demand, leaving folder thumbnails untouched. File thumbnails are
  regenerated as media is viewed, so it is a safe way to reclaim disk space or force regeneration
  after changing the thumbnail size (#37).
- Backed by a new `DELETE /api/thumbnails/files` endpoint and `ThumbnailService.cleanAllFileThumbnails()`,
  which shares its deletion logic with the existing scheduled stale-thumbnail cleanup.
- Orphan cleanup now also deletes `UserTag` and `VideoTag` records for removed media files, not
  just `PhotoTag`.
- Thumbnail regeneration delay adjusted to avoid conflicting with an in-progress media scan.
- Loading state added to the Photos view's next-photo navigation so the viewer doesn't appear
  stuck while the next page of photos is fetched (#38).
- Folder Browser modal now supports selecting a media kind when creating a library folder (#33).
- "Back to Library" button relabelled with localized text in the folder Edit component.
- `PhotoDetail` and `VideoDetail` now default their details sidebar to closed on open.

## [2.0.3] — 2026-06-28

- New metadata-backfill feature fills in missing EXIF data (e.g. GPS, camera info, date taken)
  for already-scanned photos whose tags were created before that metadata was extracted.

## [2.0.2] — 2026-06-28

- Library folders now support a `mediaKind` (`PHOTOS`, `MUSIC`, `MUSIC_VIDEOS`) so a folder can be
  categorized by the kind of media it holds; editable from the Library Folder edit form (#30).
- Configurable thumbnail retention setting added, wired through `ThumbnailService` and
  `ThumbnailJobService`.
- Responsive design improvements for the Settings page on smaller screens.

## [2.0.1] — 2026-06-28

- Deep-clean process optimized to use bulk deletes, reducing lock time on large libraries, and now
  also clears `VideoTag` records.
- Directory-skipping logic in `MediaScanService` improved during folder scans (#31).

## [2.0.0] — 2026-06-26

Video support across the whole stack — videos are now first-class media alongside photos.

### Scanning & metadata

- The scan now indexes videos (`mp4`, `mkv`, `avi`, `mov`, `wmv`, `flv`, `webm`, `m4v`) and
  populates a new `VideoTag` record per file with pixel dimensions, duration, frame rate, codec,
  bitrate, file size, and MIME type (read via `metadata-extractor`).
- The tag-scan phase is now strategy-driven: each `MediaTagScanner` owns its own "untagged" query, so
  images drain against `photo_tag` and videos against `video_tag` without the scanner core knowing
  the difference. New `VideoTagScanner` handles `MediaKind.VIDEO`.
- Content hashing is kind-aware. Videos are hashed from a small (~1 KB) slice taken from the middle of
  the file instead of digesting the whole file, which keeps hashing fast and de-duplication working
  for large videos. The scheduled hash job and inline hash-on-scan both drain images and videos.

### Browsing, playback & UI

- Videos appear in folders in the Photos view beside photos, each with a film badge and a play-button
  overlay. Filters, search, and the "Recent" view now match videos as well as photos.
- Clicking a video opens an inline player (`VideoDetail`) that streams the file with HTTP range
  support (seeking), shows video metadata, and supports rating and deletion.
- New **Playlist** page (sidebar + route), the video counterpart of the Slideshow: shuffle/sequential
  playback, auto-advance on end, rating, thumbnail strip, details sidebar, fullscreen, and keyboard
  shortcuts. Videos never play in the photo slideshow.
- Video thumbnails are real frames grabbed from the middle of the video (pure-Java JCodec), with a
  film-icon fallback when a frame cannot be decoded.

### API

- New `api/videos` endpoints: list, playlist batch, detail, range stream, thumbnail, rating, view,
  delete, and clear.
- `metadata-extractor` retained for video metadata; added `jcodec` / `jcodec-javase` for frame
  extraction. Project version bumped to `2.0.0`.

## [1.5.3] — 2026-06-25

- Dynamic scheduling for media hash generation with adaptive intervals and batch sizes.
- Calibrate interval and batch size from the first run's duration, keeping the values for subsequent runs.
- New media hash job interval setting with a default value, wired through related services.
- Improved log formatting in `MediaHashJobService`.

## [1.5.2] — 2026-06-25

- Tag export/import reworked to support user-specific data keyed by content hash (#27).

## [1.5.1] — 2026-06-25

- New API endpoint to retrieve the application version, displayed in the sidebar (#26).

## [1.5.0] — 2026-06-24

- Rating comparison functionality for photo search (e.g. ratings above/below a value) (#25).
- Photo details panel now shows album, rating, and view count.

## [1.4.0] — 2026-06-24

- "Remember me" functionality for user authentication and session management.
- "Recent" filter for photos, with localization support (#23).
- New endpoint for retrieving recently added photos, with pagination.
- Theme-aware loading spinner replacing the static loading image.
- Login and registration forms restyled with improved accessibility.
- Responsive image sizing in the Photos and Slideshow views (#24).

## [1.3.0] — 2026-06-22

- Settings management enhanced with metadata support (#22).

## [1.2.0] — 2026-06-22

- Rating system made user-specific — ratings now belong to individual users (#21).

## [1.1.0] — 2026-06-22

- Theme toggle with dark-mode support across the UI (#20).
- Database statistics endpoint integrated into the frontend for real-time display (#19).
- Sidebar and Home view refactored to surface admin settings navigation.

## [1.0.0] — 2026-06-19

- **First stable release.** Consolidates the full media library manager: scanning,
  thumbnails, slideshow, duplicate detection, ratings, scheduled jobs, and the new
  user authentication/registration system introduced in 0.4.1.

## [0.4.1] — 2026-06-19

- User authentication and registration system.

## [0.4.0] — 2026-06-18

- Off-canvas mobile sidebar with backdrop and toggle (#5).
- Responsive layout improvements for Photos, Slideshow, ScanProgress, and LibraryFolders.
- Folder browsing added to the LibraryFolder edit component.
- "Library Folders" renamed to "Library" across the UI.
- Loading state and keyboard-shortcut hint added to the Slideshow.
- First automatic scan delayed by 6 hours after application startup (#18).

## [0.3.9] — 2026-06-17

- Background duplicate-finding job with progress tracking and status API.
- Filesystem browsing API plus a frontend modal for directory selection (#16).
- Scheduled scanning job (#15).
- Duplicate handling refactored to support pagination, with multilingual bulk-deletion support (#14).
- Additional indexing to speed up duplicate detection.

## [0.3.8] — 2026-06-17

- Duplicate file detection and management features, with tests (#13).
- Duplicate management modal and full-size image preview (#14).
- Close button added to the slideshow details sidebar.
- README updated to reflect project details and features.

## [0.3.7] — 2026-06-16

- Hash-on-scan functionality for media files (#11).
- Folder deletion including all contents, with updated API docs (#9).
- Detailed photo information sidebar with file-size and date formatting (#6).
- Handling for missing photos and self-healing library records (#3).
- Empty-folder cleanup with tests (#2).
- Keyboard-shortcuts help modal with multi-language support.
- "Cleaning folders" phase added to the scan process (#12).

## [0.3.6] — 2026-06-15

- Comprehensive API documentation for ParrotApp.
- Setting to advance to the next photo after rating, plus rating advances the slideshow (#10, #4).
- Folder hash refresh after photo deletion (#8).
- Save-confirmation message for settings, in multiple languages.

## [0.3.5] — 2026-06-14

- Integration tests added for various controllers and models.
- Cleaning phase added to the scan process.
- Orphan cleanup now respects scan cancellation.
- `maven-surefire-plugin` configured with experimental Byte Buddy support.

## [0.3.4] — 2026-06-14

- Orphaned media-file cleanup service; scan results now track removed files.

## [0.3.3] — 2026-06-14

- Content hash generation for image files, with hash statistics exposed via API.

## [0.3.2] — 2026-06-13

- Cleanup of stale file thumbnails, detaching references from media files.

## [0.3.1] — 2026-06-13

- CSV import functionality removed (added briefly in 0.3.0, then reverted).

## [0.3.0] — 2026-06-13

- CSV import for photo ratings, with localization support.

## [0.2.1] — 2026-06-13

- Photo search with query and rating filters.
- Sorting for photo queries with customizable sort fields and directions.
- Photo navigation re-attaches the intersection observer to load more photos.
- UI tweaks to Home card dimensions and Photos scroll margins.

## [0.2.0] — 2026-06-12

- Shuffle mode toggle for the slideshow.
- Folder scoping for the slideshow and photo retrieval (per-folder slideshows).
- Folder chain retrieval and folder-aware routing.
- Scan cancellation with corresponding UI state.
- Search across folders and media files with pagination and rating filters.
- Bulk selection and deletion for photos.
- Photo rating and deletion from the PhotoDetail component.
- Slideshow fullscreen support for iOS devices.
- Reload functionality for router views.
- Legacy `Folders.vue` and related components removed.

## [0.1.10] — 2026-06-11

- Scanning optimizations with accurate progress denominator and elapsed-time display.
- Pagination for folder photos retrieval.
- Deep-clean functionality to permanently delete all library data and thumbnails.
- Infinite-scroll fix and layout/scrolling improvements in the Photos view.

## [0.1.9] — 2026-06-11

- Photo deletion with a confirmation modal.
- Import process refactor.

## [0.1.8] — 2026-06-11

- Display photos in the photos-browsing view.

## [0.1.7] — 2026-06-10

- Bulk import for tag data with progress tracking and optimized database queries.
- Random photo selection improved via ID shuffling.
- Index added to the `media_file` table for directory/filename lookups.
- Breadcrumb navigation with a home icon.

## [0.1.6] — 2026-06-10

- Maintenance release (version bump).

## [0.1.5] — 2026-06-10

- `metadata-extractor` dependency added; improved date handling in `PhotoTag` / `PhotoTagScanner`.
- Image preloading and release mechanisms to optimize slideshow memory usage.
- Optimized random photo selection (count + shuffle).
- Thumbnail regeneration with stale checks, a `dateUpdate` field, and weighted image selection.

## [0.1.4] — 2026-06-09

- Thumbnail generation for folders, with related service/component updates.

## [0.1.3] — 2026-06-09

- Folder scanning now skips hidden directories.
- Thumbnail generation for photos, including images from subdirectories; `thumbnailId` added to the media-file model.

## [0.1.1] — 2026-06-08

- Media file and folder handling updated to support a library-folder structure.
- Concurrent scanning and tagging in `MediaScanService`.
- Photo rating with hover effects in the Slideshow component.
- `Thumbnail` entity and service introduced, with a scheduled job to generate thumbnails for folders without one.
- New API endpoints for folder hierarchy and media-file retrieval.

## [0.0.1] — 2026-04-24 → 2026-06-08

Initial development. Foundations of the application:

- Project bootstrap, coding standards, and PRD.
- Media file and `PhotoTag` models; photo scanning feature (UI + backend).
- `MediaScanService` / `MediaTagScanner` architecture with `ScanContext`, multi-threaded
  (`maxThreads`) scanning and tagging, batch processing, and memory optimizations.
- Background library scan with REST API, progress tracking, and scan phases/metrics.
- `Folder` entity, repository, and service; folder management API/UI with localization,
  change detection, and level/finished tracking.
- Clear-library and clear-folders features.
- Photo detail retrieval and display with metadata.
- Tag export/import with DTOs.
- Slideshow feature: history-stack navigation, fullscreen, random-photo endpoints with
  prefetching, dynamic slideshow timing via a setting, photo rating and view increment.
- Internationalization via `multiLanguage.xml`; dynamic backend host resolution.

[1.5.3]: #153--2026-06-25-current
[1.5.2]: #152--2026-06-25
[1.5.1]: #151--2026-06-25
[1.5.0]: #150--2026-06-24
[1.4.0]: #140--2026-06-24
[1.3.0]: #130--2026-06-22
[1.2.0]: #120--2026-06-22
[1.1.0]: #110--2026-06-22
[1.0.0]: #100--2026-06-19
[0.4.1]: #041--2026-06-19
[0.4.0]: #040--2026-06-18
[0.3.9]: #039--2026-06-17
[0.3.8]: #038--2026-06-17
[0.3.7]: #037--2026-06-16
[0.3.6]: #036--2026-06-15
[0.3.5]: #035--2026-06-14
[0.3.4]: #034--2026-06-14
[0.3.3]: #033--2026-06-14
[0.3.2]: #032--2026-06-13
[0.3.1]: #031--2026-06-13
[0.3.0]: #030--2026-06-13
[0.2.1]: #021--2026-06-13
[0.2.0]: #020--2026-06-12
[0.1.10]: #0110--2026-06-11
[0.1.9]: #019--2026-06-11
[0.1.8]: #018--2026-06-11
[0.1.7]: #017--2026-06-10
[0.1.6]: #016--2026-06-10
[0.1.5]: #015--2026-06-10
[0.1.4]: #014--2026-06-09
[0.1.3]: #013--2026-06-09
[0.1.1]: #011--2026-06-08
[0.0.1]: #001--2026-04-24--2026-06-08

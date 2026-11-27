# Unique Singles

A [Lidarr](https://lidarr.audio) plugin that automatically detects and cleans up redundant singles when the same track already exists on a monitored album or EP.

When Lidarr monitors an artist, it tracks all release types — albums, EPs, and singles. This often leads to duplicate tracks: a single gets downloaded, and then the same track appears on a full album. Unique Singles detects these duplicates and removes the redundant single.

**Example:** You download Chappell Roan's single "HOT TO GO!" and later download the album "The Rise and Fall of a Midwest Princess". Unique Singles detects that "HOT TO GO!" on the single matches the album track, unmonitors the single, and deletes its files.

## What's different in this fork

This fork follows the original plugin and adds two changes:

- **Lidarr 4.x support.** Each release includes a .NET 10 build for Lidarr 4.x as well as the .NET 8 build for Lidarr 3.x. Lidarr installs the one that matches its own .NET version.
- **Quieter logs.** Per-track match details for singles that are skipped or kept log at Debug instead of Info, so a full-library scan no longer fills Lidarr's log files. See [Logs](#logs).

The fork's earlier fixes were merged into the original plugin ([#27](https://github.com/jtstothard/lidarr-plugin-uniquesingles/pull/27) to [#32](https://github.com/jtstothard/lidarr-plugin-uniquesingles/pull/32)). Fork releases are numbered 1.1.0.x and the original plugin's are 1.0.2.x, so Lidarr won't treat an original-plugin release as an update to this fork.

## Requirements

- **Lidarr nightly branch** — plugin support is not available in the stable release channel.
- Lidarr 3.x (.NET 8) or Lidarr 4.x (.NET 10).

## Installation

1. Open Lidarr and go to **System → Plugins** (or **Settings → General → Plugins** on some builds)
2. Paste the repository URL:

   ```
   https://github.com/matrix-runout/lidarr-plugin-uniquesingles
   ```

3. Click **Install**
4. When Lidarr reports that it installed the plugin, restart Lidarr. The plugin loads on the next start.

To update, install from the same URL again and restart Lidarr.

### Switching from the original plugin

This repository is a maintained fork of [jtstothard/lidarr-plugin-uniquesingles](https://github.com/jtstothard/lidarr-plugin-uniquesingles). Lidarr installs each plugin into a folder named after its GitHub owner and loads every plugin folder it finds, so the two versions do not replace each other. If the original plugin is installed, remove it before installing this one:

1. Go to **System → Plugins** and uninstall the `jtstothard` Unique Singles plugin.
2. Restart Lidarr.
3. Install this fork using the steps above.

Your Unique Singles entries under **Settings → Connect** and **Settings → Metadata** are stored by Lidarr, not by the plugin, so they are kept.

### Installed before the move to matrix-runout?

This fork moved to `matrix-runout/lidarr-plugin-uniquesingles` from its previous GitHub account. For the same folder reason, an update from the new location installs a second copy next to the old one. Reinstall once:

1. Go to **System → Plugins** and uninstall Unique Singles.
2. Install it again from `https://github.com/matrix-runout/lidarr-plugin-uniquesingles`.
3. Restart Lidarr.

Your Unique Singles settings are kept.

## Configuration

After installation, add a Unique Singles connection:

1. Go to **Settings → Connect**
2. Click the **+** button to add a new connection
3. Select **Unique Singles** from the list
4. Configure the settings (see below) and click **Save**

### Settings

| Setting | Default | Description |
|---------|---------|-------------|
| **Duration tolerance (ms)** | 3000 | Maximum duration difference for title + duration matching. Tracks within ±3 seconds are considered the same recording. |
| **Release types to compare** | `Album, EP` | Comma-separated release types used as comparison sources. The plugin checks whether single tracks appear on these release types. |
| **Title-only match action** | Flag for review | What to do when a single track matches an album track by title alone (no duration match). See matching tiers below. |
| **Exact FLAC analysis (report only)** | Off | Verify unresolved downloaded FLAC tracks locally; report complete duplicates without adding cleanup authority. |
| **Local FFmpeg executable** | `ffmpeg` | Operator-provided decoder inside the Lidarr runtime; no decoder is bundled. |
| **Exact FLAC cleanup** | Off | Separately opt into whole-release cleanup after full verification and source/content revalidation. Also enables analysis. |
| **Eagerly unmonitor redundant singles** | On | Unmonitor a monitored single with no downloaded files as soon as every catalog track is Tier-1/Tier-2 redundant against your downloaded library — at metadata refresh, album add/refresh, and monitoring changes, before anything is grabbed, and again on album imports and the scheduled scan. Purely preventive: never deletes files and never touches singles that already have downloads. |
| **Scan interval (minutes)** | 1440 (24 hours) | How often to run an automatic full-library scan. Minimum 60 minutes. Only read from the **UniqueSingles Scan** entry under **Settings → Metadata**; the value on the Connect entry is ignored. |

The same settings form appears in two places. The Connect entry controls import-triggered cleanup, and when it exists the scheduled scan uses its matching settings too. The Metadata entry controls the scan interval and supplies matching settings when no Connect entry is available; if neither supplies matching settings, the scan uses defaults.

See [exact FLAC operations](docs/exact-flac-analysis.md) for supported formats, decoder setup, evidence, limits and [disposable host acceptance](docs/acceptance/issue-42.md). Report-only analysis is not a dry run of existing metadata cleanup.

## Matching Tiers

Unique Singles uses a 3-tier cascade to match single tracks against album/EP tracks:

| Tier | Method | Confidence | Action |
|------|--------|------------|--------|
| **Tier 1** | MusicBrainz Recording MBID match | ~99.9% | Auto-clean (unmonitor + delete) |
| **Tier 2** | Title + duration match (within tolerance) | ~95% | Auto-clean (unmonitor + delete) |
| **Tier 3** | Title-only match | ~80% | Configurable: flag for review, skip, or auto-clean |

**Tier 1** matches are exact — the same MusicBrainz recording appears on both the single and the album. This is the highest confidence match.

**Tier 2** matches use normalized title comparison plus duration within the configured tolerance. Handles cases where MusicBrainz has different recording IDs for the same song (e.g., different mastering).

**Tier 3** matches find the same title but can't confirm duration. By default, these are flagged for manual review rather than auto-deleted. You can change this behavior with the **Title-only match action** setting.

## How It Works

- **Before the grab (eager):** When a single becomes grabbable — added by a metadata refresh (for example after switching an artist's metadata profile to include Singles), manually added, refreshed, or switched to monitored — Unique Singles immediately unmonitors it if it is fully redundant: every catalog track must be Tier-1/Tier-2 redundant against downloaded tracks on your monitored library releases, and the single must have no downloaded files. This happens before RSS or a search can grab it, so the redundant single is never downloaded in the first place. Eager unmonitors are recorded in Lidarr's History. Can be turned off with **Eagerly unmonitor redundant singles**.
- **On import:** When an album or EP is imported, Unique Singles checks whether any monitored singles contain tracks matching the imported tracks. Matching singles are unmonitored and their files deleted.
- **Scheduled scan:** A full-library scan runs periodically (configurable interval) to catch any singles that were missed by import-triggered cleanup — for example, singles added before the plugin was installed. The scan is off by default. To turn it on, go to **Settings → Metadata**, open **UniqueSingles Scan**, and tick **Enable**. Untick it to stop the scan. While it is enabled, the scan is listed under **System → Tasks → Scheduled** as **Unique Singles Scan**, where you can see its last and next run and start it manually.

Singles with downloaded files are checked with all three tiers, and redundant ones are unmonitored and their files deleted. Singles without downloaded files are unmonitored (never deleted) when every track has a Tier-1 or Tier-2 match against downloaded tracks on monitored releases of the configured comparison types (Album and EP by default): eagerly (see above), on an album import, and on the scheduled scan, which catches singles that existed before their album was downloaded. With **Eagerly unmonitor redundant singles** off, the import and scan paths fall back to exact MBID (Tier-1) matches only. Otherwise they are left alone; a single whose song is only on an album you haven't downloaded yet stays wanted. Only downloaded tracks on those monitored releases are used as comparison sources. Releases with a Live, Remix, DJ-mix, or Demo secondary type are never used as comparison sources, because their tracks often share a title and length with the studio recording.

Every path applies the same whole-release protection: a single with any catalog track that has no Tier-1/Tier-2 match — downloaded or not — is exclusive content and is never unmonitored or deleted, and title-only (Tier 3) matches never unmonitor a file-less single. See [eager unmonitor design](docs/eager-unmonitor.md) for the hook points and why a grab-time hook is not used.

When Unique Singles deletes a single's audio, it also removes the rest of the single's folder. Sidecar files such as `album.nfo`, `cover.jpg`, and `.lrc` lyrics go to Lidarr's recycle bin next to the deleted audio (or are deleted if no recycle bin is set), and then the empty folder is removed. The folder stays if it isn't inside the artist folder, has subfolders, holds files Lidarr tracks for another release, or still contains audio files Lidarr doesn't track.

## Logs

At Lidarr's default **Info** log level, Unique Singles logs the following. Search the logs for `UniqueSingles` to find them.

- An outcome line for each single it cleans up (`cleanup-approved`) or flags for review (`review-needed`, `outcome=unresolved-review`)
- Per-track match details (`track-evidence`) for approved cleanups and reviews
- Unmonitor and delete results
- A summary at the end of each scan, which counts the singles it kept (`keepExclusive=`) and skipped (`skipped=`)

Singles that are kept because they have exclusive tracks (`keep-exclusive`) or skipped (`cleanup-skipped`) log at **Debug**, with their per-track details. On a large library they are most of a scan. To see why the plugin skipped or kept a single, set **Settings → General → Logging → Log Level** to **Debug**. A full-library scan writes one line per candidate track at that level, so set it back to **Info** afterward.

## Known Issues

- **Requires Lidarr nightly branch.** Plugin support is not available in the stable release channel.
- **Tubifarry coexistence.** If Tubifarry's QueueCleaner is installed alongside UniqueSingles, a plugin dependency conflict may occur. Workaround: disable Tubifarry's QueueCleaner, or use only UniqueSingles for cleanup.
- **Scan settings live under Metadata.** Lidarr has no settings page for plugin tasks, so the scan is turned on and its interval is set under **Settings → Metadata → UniqueSingles Scan**, not on the Tasks page. Matching settings still come from the **Settings → Connect** entry.

## Related

- [Lidarr Issue #2671](https://github.com/Lidarr/Lidarr/issues/2671) — Unmonitor singles if on album
- [Lidarr Issue #1025](https://github.com/Lidarr/Lidarr/issues/1025) — Singles included with Albums

## Credits

Unique Singles was created by [Jay Stothard (jtstothard)](https://github.com/jtstothard). See [What's different in this fork](#whats-different-in-this-fork) for what this fork adds. If you find the plugin useful, you can support the original author on [Ko-fi](https://ko-fi.com/trshpotato).

## License

[MIT](LICENSE). The original copyright notice is kept in the license file.

### Optional exact FLAC reports

Enable **Exact FLAC analysis (report only)** in the matching settings to verify unresolved downloaded singles locally. It defaults off and requires an operator-provided FFmpeg executable in the Lidarr host/container. Complete verified releases are reported without additional cleanup. Existing metadata cleanup still runs. See [decoder, limits and deployment notes](docs/exact-flac-analysis.md).

### Keep and review outcomes

Scans and imports report `keep-exclusive` when any evaluated downloaded track has
no eligible match. This protects the whole release, even if another track has a
title-only match. Track diagnostics remain available, but that release does not
increase the review-needed count. “Exclusive” means unmatched against the current
eligible comparison sources, not proof that the recording exists nowhere else.

With **Tier 3 action** set to **Flag only**, releases whose remaining uncertainty
is title-only matching report `review-needed` (`outcome=unresolved-review`). Each
release counts once, even with several title-only tracks. Logs include candidate
release/track identity, matched source identity, and the reason for keeping or
reviewing the release. Existing cleanup settings, matching tolerance, source
eligibility, and no-file Tier-1 unmonitoring continue to apply.

Enable **Exact FLAC cleanup** separately to permit cleanup after complete exact
coverage and immediate source/content revalidation. It defaults off and also enables
analysis. Analysis alone remains report-only. Missing, changed or unverifiable files
protect the release; every track needs a retained eligible counterpart. See the
[cleanup contract](docs/exact-flac-analysis.md#explicit-exact-cleanup-41).

# Changelog

All notable changes to Unique Singles will be documented in this file.

## [Unreleased]

This release is published from the fork at jasonpatrickellykrause/lidarr-plugin-uniquesingles. Each fix below is also open as a pull request on the original repository (jtstothard/lidarr-plugin-uniquesingles #27 to #32).

### Added

- `CONTRIBUTING.md`, `SECURITY.md`, and `CODE_OF_CONDUCT.md`
- Issue and pull request templates
- Dependabot config for NuGet and GitHub Actions dependencies
- CodeQL security scanning workflow

### Changed

- The plugin now installs from and checks for updates against the fork. Uninstall the original plugin before installing this one; see the README.
- The scheduled full-library scan only runs when **UniqueSingles Scan** is enabled under Settings → Metadata. Previously it was scheduled on every install, and disabling or removing the Metadata entry did not stop it.
- Albums and EPs with a Live, Remix, DJ-mix, or Demo secondary type are no longer used as comparison sources. A live track with the same title and a similar length could previously cause the studio single to be deleted.
- Documented where the scheduled scan appears under System → Tasks. Documented that **Scan interval** is only read from the Metadata entry, that the scheduled scan uses the Connect entry's matching settings, and that singles without files are unmonitored on exact Tier 1 matches.
- Routine per-track and per-single log lines now log at Debug instead of Info: skipped and unmatched tracks, singles that are already unmonitored or have no files, and the per-artist scan start line. A full-library scan previously wrote one Info line per candidate track and filled Lidarr's log files. Approved cleanups, review-needed (Tier 3) tracks, unmonitor and delete results, and the final scan summary still log at Info.

### Fixed

- Release builds now carry the release version. Every build previously reported 1.0.2.0, so Lidarr always offered an update that never cleared after installing it (upstream #25).
- The scheduled scan no longer runs on every Lidarr restart. Lidarr removes plugin task rows at startup; the plugin now restores the previous run time instead of scheduling the scan as overdue.
- With `Single` in **Release types to compare**, every downloaded single matched its own tracks and was deleted. A single is now never compared against itself, and of two identical singles only one is removed.

## [1.0.2] - 2025-05-28

### Added

- Import-triggered redundant single cleanup — detects when a single's tracks already exist on a monitored album or EP
- 3-tier matching strategy: MusicBrainz Recording MBID (Tier 1), title + duration (Tier 2), title-only (Tier 3)
- Scheduled full-library scans with configurable interval
- Configurable settings via Lidarr Connect UI:
  - Duration tolerance for title + duration matching
  - Release types to use as comparison sources
  - Title-only match action (flag for review, skip, or auto-clean)
  - Scan interval
- Unmonitors redundant singles and deletes their files
- Conservative behavior: ambiguous matches are flagged for review, never auto-deleted by default

### Known Issues

- Requires Lidarr nightly branch (plugin support not in stable)
- Tubifarry QueueCleaner coexistence may cause a plugin dependency conflict
- Scheduled scan appears under Settings → Metadata due to Lidarr's limited plugin scheduling API

# CLAUDE.md — Project Context for AI-Assisted Development

> This file is read automatically by Claude Code at session start.
> Keep it updated after significant decisions — it's the project's memory.

---

## Project Overview

**Project name:** Comicarr
**GitHub:** https://github.com/andrew-i-maul/Comicarr

A ground-up rewrite of Mylar3 built on the Readarr codebase, targeting full compatibility with the *arr ecosystem (Prowlarr, download clients, notifications). Goal is a modern, polished comic book PVR that feels native alongside Sonarr/Radarr — same UI language, same operational conventions, same Docker-first deployment.

**Origin:** Forked from Readarr. Book/audiobook-specific logic is being stripped and replaced with comic-specific equivalents.

**Status:** Clean .NET 8 baseline build established and committed to GitHub. Codebase audit and strip-down is next. Development has moved fully into code-server.

---

## Environment

- **Dev server:** Debian LXC on Proxmox, code-server running as `comicarr` user on port 8680
- **Repo location:** `/home/comicarr/Comicarr`
- **Runtime:** .NET 8 (upgraded from Readarr's .NET 6)
- **Frontend tooling:** Node.js LTS + Yarn
- **Editor:** code-server (VS Code in browser)

### code-server Extensions (install if missing)
- C# Dev Kit (Microsoft) — IntelliSense, debugging, go-to-definition
- ESLint — frontend linting
- Prettier — code formatting
- GitLens — navigating inherited codebase
- Claude — Anthropic extension for Claude Code codebase awareness

---

## Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| Backend | C# / ASP.NET Core (.NET 8) | |
| Frontend | React (inherited from Readarr/Sonarr) | |
| Database | SQLite via Dapper | |
| Metadata | ComicVine API, ComicTagger | Comics metadata quality is poor vs TV/movies — consider Metron as an alternative; confirm any additional metadata sources with Andrew before implementing |
| Indexers | Prowlarr + native Newznab/Torznab | |
| Download clients | qBittorrent, SABnzbd, NZBGet, Deluge etc. (inherited) | |
| Deployment | Docker (Linux — Proxmox LXC target) | |

---

## Domain Model

Comics metadata is messier than TV or books. Key concepts:

- **Publisher** — e.g. Marvel, DC, Image
- **Series** — a named run, e.g. "The Amazing Spider-Man"
- **Volume** — a series can have multiple volumes (publishers restart numbering)
- **Issue** — individual issue within a volume, identified by number + year
- **Special / Annual / One-Shot** — non-standard issues, need explicit handling
- **Story Arc** — cross-series arcs (nice to have, not MVP)

ComicVine is the primary metadata source. Their IDs are the canonical identifiers throughout the system. If other metadata sources are identified, confirm with Andrew before taking on the additional scope.

---

## Architecture Decisions

### What We're Keeping from Readarr
- Core arr infrastructure: commands pipeline, event bus, health checks, notifications, scheduler
- Download client integrations (all of them)
- Indexer/RSS handling (Newznab/Torznab)
- SQLite + migration framework
- Docker packaging and update mechanism
- React frontend shell, routing, design system

### What We're Ripping Out
- All Goodreads / BookInfo / OpenLibrary integration
- Audiobook-specific logic
- Book/author domain model (Author, Book, Edition, BookFile)
- Calibre integration
- TagLibSharp-Lidarr (audio tagging — not needed)
- SixLabors.ImageSharp (book cover handling — replace with comic cover equivalent)
- PdfSharpCore (not needed)
- AZW/Kindle tag handling (`src/NzbDrone.Core/MediaFiles/AzwTag/`)

### What We're Building New
- ComicVine API client (`src/NzbDrone.Core/MetadataSource/ComicVine/`)
- Comic domain model: Publisher, Series, Volume, Issue, ComicFile
- CBR/CBZ file handling and parsing
- Comic-specific rename tokens and naming format
- Quality definitions appropriate for comics (scan quality, resolution, source)
- Issue monitoring logic (analogous to Sonarr episode monitoring)

### Indexers
Prowlarr handles indexer management. We support it as a first-class integration. Native Newznab/Torznab kept for users not running Prowlarr.

---

## ComicVine API

- Docs: https://comicvine.gamespot.com/api/documentation
- Rate limit: 200 requests/hour; higher with API key
- Auth: API key passed as query param `api_key`
- Key endpoints:
  - `/search` — series/issue search
  - `/volume/{id}` — series/volume detail
  - `/issues/?filter=volume:{id}` — all issues for a volume
  - `/issue/{id}` — individual issue detail
- Response format: JSON wrapped in `{error, limit, offset, number_of_page_results, number_of_total_results, status_code, results}`

**Known pain points:**
- Rate limits require careful request queuing
- Some series have multiple volumes with overlapping numbers — must expose volume selection in UI
- Annuals/specials are inconsistently categorised — handle gracefully, don't crash

---

## File Handling

Comic files are archives containing image sequences.

| Format | Extension | Notes |
|---|---|---|
| Comic Book RAR | `.cbr` | RAR archive |
| Comic Book ZIP | `.cbz` | ZIP archive — preferred |
| Comic Book 7-Zip | `.cb7` | Less common |
| PDF | `.pdf` | Lower priority, defer to post-MVP |

Quality detection based on image resolution inside the archive, not video bitrate.

Rename tokens to implement:
- `{Series Title}` `{Volume Number}` `{Issue Number}` `{Issue Title}` `{Year}` `{Publisher}` `{Quality Title}`

---

## Completed Work

- [x] Forked from Readarr
- [x] Upgraded from .NET 6 to .NET 8
- [x] Fixed all .NET 8 breaking changes (obsolete serialization constructors, ISystemClock, header API)
- [x] Updated vulnerable NuGet packages (MailKit, SixLabors.ImageSharp)
- [x] Updated all Microsoft.Extensions packages to 8.x
- [x] Clean build — 0 errors, 0 warnings
- [x] Committed to GitHub
- [x] Moved development into code-server

## Current Focus

- [ ] Install and configure code-server extensions (C# Dev Kit, ESLint, Prettier, GitLens, Claude)
- [ ] Codebase audit — catalogue what to remove
- [ ] Strip out Goodreads/BookInfo integration
- [ ] Strip out book/author domain model
- [ ] Scaffold ComicVine API client
- [ ] Define comic domain model

---

## Key Files & Locations

| What | Path |
|---|---|
| Core services | `src/NzbDrone.Core/` |
| API controllers | `src/Readarr.Api.V1/` |
| Frontend | `frontend/src/` |
| DB migrations | `src/NzbDrone.Core/Datastore/Migration/` |
| HTTP middleware | `src/Readarr.Http/` |
| Package versions | `src/Directory.Packages.props` |
| Docker | `./Dockerfile` |

---

## Conventions

### C# Backend
- Follow existing Readarr/Sonarr patterns — find the analogous class and mirror it
- Services in `src/NzbDrone.Core/`
- New feature areas get their own subfolder
- Constructor injection throughout
- All DB access via repositories — no raw SQL in services
- Commands are fire-and-forget via `IManageCommandQueue`
- StyleCop is enforced as errors — one blank line between members, trailing newline at end of file

### React Frontend
- Match Sonarr's component and styling conventions — non-negotiable for the "feels native" goal
- Don't introduce new state management libraries without discussion

### Git
- Feature branches off `develop`
- Conventional commits: `feat:` `fix:` `chore:` `refactor:` `docs:`

---

## Open Questions

- [ ] Volume selection UX — auto-select latest or expose choice in add-series flow?
- [ ] Story arc support — defer to post-MVP
- [ ] Mylar3 library import/migration tool — high value for adoption, scope TBD
- [ ] Additional metadata sources (e.g. Metron) — evaluate but confirm with Andrew before implementing

---

## Notes for Claude

- Treat this as Readarr with comics substituted for books — that mental model maps well
- When suggesting new classes/services, find the Readarr/Sonarr analogue first and note it
- The goal is something arr community contributors find immediately familiar — convention over cleverness
- Andrew has deep enterprise architecture and business process background (SAP, Oracle, supply chain) but is returning to hands-on development after ~25 years — prefer clear explanations over assumed modern framework knowledge
- StyleCop is enforced — all generated code must follow formatting rules (no double blank lines, trailing newlines, no blank line after opening brace)
- Dev environment is Debian LXC, all commands should be bash not PowerShell
- Confirm with Andrew before expanding scope — metadata sources, new dependencies, architectural changes

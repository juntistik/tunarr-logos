# Tunarr & Plex NAS Standards

Reference guide for channel creation, custom logo design & hosting, streaming/transcoding settings, and Synology NAS resource management (`192.168.86.40`).

---

## 1. Channel Lineup & Naming Standards

- **1–17 — US & UK Networks:** `1 HBO`, `2 NBC`, `3 FX`, `4 FOX`, `5 ABC`, `6 Comedy Central`, `7 Adult Swim`, `8 Food Network`, `9 Discovery & Travel`, `10 AMC`, `11 Syfy`, `12 BBC`, `13 UK Drama & Comedy`, `14 Showtime`, `15 PBS & Science`, `16 Toonami`, `17 Anime`
- **20–26 — Holiday & Franchises:** `20 Christmas TV`, `21 Christmas Movies`, `22 007 James Bond`, `23 Marvel`, `24 Star Wars`, `25 Harry Potter`, `26 DC`
- **30–41 — Movie Genres:** `30 Action Movies`, `31 Comedy Movies`, `32 Drama Movies`, `33 Crime Movies`, `34 Thriller Movies`, `35 Sci-Fi Movies`, `36 Horror Movies`, `37 Family & Animation`, `38 Documentaries`, `39 Romance Movies`, `40 Mystery Movies`, `41 Adventure Movies`
- **42–44 — Movie Decades:** `42 80s Movies`, `43 90s Movies`, `44 Classic Movies`
- **45–48 — Curated Movie Themes:** `45 Dad Movies`, `46 Date Night`, `47 Cult Movies`, `48 Midnight Movies`

### Smart Collection & Scheduling Standard
- Every channel uses a dedicated Tunarr Smart Collection named **`Channel: <Channel Name>`** so new Plex content matching the rules auto-populates.
- **Schedule Type:** `RandomSlotSchedule` (`maxDays: 3`, `flexPreference: "distribute"`, `randomDistribution: "uniform"`, `order: "shuffle"`).
- **Important:** Do not run `POST /api/channels/:id/programming` while Tunarr is actively scanning/indexing a library (`isLocked: true`), or Tunarr's worker pool may hit a 60s lock timeout.

---

## 2. Custom Logo Standards

1. **Format & Transparency:**
   - Always use **RGBA PNG with a transparent background** (flood-fill solid background from edges, plus clean any enclosed letter counters like `D`, `A`, `O`).
   - Keep strong outlines/contrast so logos remain legible on both dark and light guide themes.
2. **Dimensions & File Size:**
   - Resize to **`320px` max width** (`LANCZOS`, `optimize=True`) so guide icons load quickly across all Plex clients (~30–150 KB).
3. **HTTPS Hosting (Required for `app.plex.tv`):**
   - **Never** leave custom channel `icon.path` pointing at `http://192.168.86.40:8000/images/uploads/...` if you use `https://app.plex.tv` — browsers block `http://` images on `https://` pages as **Mixed Content**.
   - Host custom PNGs in the public GitHub repo **`https://github.com/juntistik/tunarr-logos`** and reference them via:
     `https://raw.githubusercontent.com/juntistik/tunarr-logos/main/<name>.png`
   - Standard network/genre logos come from `https://raw.githubusercontent.com/PseudoTV/PseudoTV_Logos/master/...`.
4. **Watermarks vs. Direct Streaming:**
   - `watermark.enabled` is **`false`** by default because `hls_direct` copies video without re-encoding (`-c:v copy`) and cannot burn overlays.
   - Enabling corner watermarks requires switching `streamMode` to `mpegts` or `hls` (software transcoding).

---

## 3. Streaming, Transcoding & NAS Hardware Constraints

- **NAS Hardware:** Synology NAS with **AMD Ryzen Embedded V1500B** (no Intel QuickSync / no `/dev/dri` hardware GPU) and **3.8 GiB RAM (`3,889 MB`)**.
- **Recommended `streamMode`:** **`hls_direct`**
  - Benchmark on Ch 3 (`FX`): `hls_direct` achieved **0.69s** time-to-first-byte (vs **2.13s** for `mpegts` and ~8–10s for full `hls` transcoding) because it avoids software `libx264` re-encoding on the V1500B CPU.
- **Default Transcode Config (`9c2d53ab-23c0-4306-b1c5-fa8f9814e920`):**
  - When transcoding is used, keep **`1280×720`**, **`videoPreset: "ultrafast"`**, and **`deinterlaceVideo: false`** (`1080p` `veryfast` + deinterlacing causes `Stream not ready yet. Retry` HTTP 500 timeouts on V1500B).

### RAM & Swap Management (3.8 GiB Limit)
- One `tunarr` container (Node + worker pool + embedded Meilisearch) uses **~400–755 MiB RAM** when idle/scanning.
- **Do NOT run duplicate stacks simultaneously** (`tunarr` + `tunarr-jellyfin` + native Jellyfin + Plex + Overseerr + Sonarr + Radarr + qBittorrent) on 3.8 GiB RAM — doing so pushed swap to **3.4 GB**, triggered `kswapd0` thrashing, and spiked disk I/O load to **22.04** (25 `D`-state processes).
- Keep `tunarr-jellyfin` stopped unless actively testing Jellyfin.

---

## 4. Plex Live TV & DVR Setup

- **Tuner Network Address:** `192.168.86.40:8000` *(never paste `/api/xmltv.xml` into the tuner address field)*
- **XMLTV Guide URL:** `http://192.168.86.40:8000/api/xmltv.xml`
- After updating Tunarr schedules or icons, trigger a **Refresh Guide** in **Plex Settings → Live TV & DVR** (`POST /livetv/dvrs/<id>/reloadGuide`) so Plex stays in sync with Tunarr's live playback clock.

---

## 5. Plex Library Hygiene Standards

- **Folder Structure:** Never nest one TV show inside another show's folder (e.g. `Survivorman/Suits`). Keep each show in its own top-level folder under `/volume1/data/media/tv/<Show Name>/Season XX/`.
- **Episode Naming:** Always use `SxxExx` (e.g. `S04E01`), avoiding `SE04E01` (parses as Season 264) or bare `206` / `701` episode numbers (parses as `S02E206` / `S12E701`).
- **Episode Ordering Overrides (`showOrdering`):**
  - **Futurama:** `tvdbProduction` (TheTVDB Production Order)
  - **Dragon Ball Z Kai:** `tvdbAbsolute` (TheTVDB Absolute Order)

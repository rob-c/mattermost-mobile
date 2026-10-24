# mm-stickers → Mattermost Mobile: scope & delivery plan

Status: draft for review · Owner: mobile fork (`rob-c/mattermost-mobile`) + plugin (`rob-c/mm-stickers`)
Target app base: upstream **v2.45.0** (tip of `upstream/release-2.45`, released 2026-10-14)

---

## 1. Summary

The plugin (`com.rob-c.mm-stickers`) today ships a full webapp experience and a *degraded* mobile
experience: every media post also attaches a plain Mattermost file so stock mobile clients render
"something". That fallback costs bandwidth, can only show static previews, and gives mobile users no
picker, no reactions, no soundboard, no video-message recording.

This plan turns the fork into a **first-class native client for the plugin**:

* full feature parity with the browser version (stickers, packs, recents, search, sticker reactions,
  GIF picker, soundboard, round video messages, admin/import flows, `/stickers`, `/sb`, `/gif`),
* **no format/platform hand-waving** — a defined renderer per format on iOS and Android, with an
  explicit, visible fallback when a codec genuinely is not available,
* **cheaper than the browser**: WS-driven invalidation (no polling), batched + conditional HTTP,
  delta pack sync, on-disk media cache, viewability-gated playback, upload compression/backoff,
* **backwards compatible in both directions**: fork ↔ old/new plugin versions, fork ↔ stock mobile,
  fork ↔ future upstream releases (small, guarded, isolated diffs).

Recommendation: build it as an isolated "product" module (`app/products/stickers/**`) behind a thin
`PluginBridge` seam, so the fork stays mergeable with upstream and the same seam can later host a
generic mobile plugin runtime (see D1).

---

## 2. Baseline (done)

| Item | State |
| --- | --- |
| Fork history before this work | pure mirror of upstream **v1.35.0-dev** (`2800de09`, 2020-08-21), zero local commits |
| Upstream history | rewritten for v2 (`22e82dfc` "Barebones code for v2", 2020-11-14) → **disjoint** from the v1 history, no merge base |
| Legacy preserved | `legacy-v1.35-2020` branch pushed to `origin` |
| New base | `master` reset to **v2.45.0** (= `upstream/release-2.45`), tracks `upstream/release-2.45` |
| Remotes | `origin` = `rob-c/mattermost-mobile`, `upstream` = `mattermost/mattermost-mobile` |
| Toolchain | Node **24.15.0** (`.nvmrc`; `engine-strict=true` blocks Node 26), npm 11 |
| Verified | `npm ci` (1609 pkgs) ✔ · `patch-package` 19/19 ✔ · `generate-compass-glyph-map` + `generate-assets` ✔ · `tsc --noEmit` **0 errors** ✔ · `eslint` clean ✔ · jest smoke ✔ |
| Not verified here | `pod install` / Gradle / device builds (no Xcode, Java or Android SDK in this environment) |

> Reproduce: `PATH=~/.local/node/node-v24.15.0-darwin-x64/bin:$PATH npm ci --ignore-scripts && npx patch-package && node scripts/generate-compass-glyph-map.mjs && node scripts/generate-assets.js`.
> `pod install` is required before an iOS build (`npm run pod-install`).

---

## 3. What the plugin already offers

**Post types** (`webapp/src/client.ts`): `custom_sticker`, `custom_video_message`, `custom_sound`, `custom_gif`.

**REST API** (`/plugins/com.rob-c.mm-stickers/api/v1`, session/token auth, see `server/api.go`):

```
GET    /config                                  capabilities + limits
GET    /packs            /packs/popular          catalog, curated install
POST   /packs/popular /packs/telegram /packs/discord /packs/upload
DELETE /packs/{pack_id}
GET    /packs/{pack}/stickers/{sticker}/file[?preview=1]
POST   /posts                                   create sticker post (root_id aware)
GET    /recents
GET    /reactions/{post_id}  POST /reactions  DELETE /reactions/{post}/{pack}/{sticker}
GET    /sounds  POST /sounds  DELETE /sounds/{id}  POST /sounds/{id}/post  GET /sounds/{id}/file
GET    /gifs/search|trending|local   POST /gifs/from_url|from_pack   GET /gifs/{id}/file
POST   /videos (multipart) | /videos/from_url    GET /videos/{id}/file
```

**WS events** (webapp `index.tsx`): `custom_com.rob-c.mm-stickers_open_picker|open_soundboard|open_gif|sticker_reaction_updated`.

**Already good for mobile** — `serveBytes` sets `Cache-Control: immutable`, `Accept-Ranges`, and handles
single-range requests for every binary endpoint (`server/api.go`), i.e. streaming/seeking already works.

**Weak spots for a native client**

1. `/reactions` is **per post** → the webapp issues one request per rendered post (N+1), and the WS
   `sticker_reaction_updated` event carries only `post_id`, so clients refetch.
2. `/packs` returns the **entire catalog** (all stickers, all packs) with no delta/pagination/ETag.
3. JSON endpoints have **no `ETag`/conditional GET**, so every foreground refresh re-downloads.
4. No thumbnail/rendition parameters for video posters or GIF grids (`?preview=1` exists for stickers only).
5. No `api_version`/`features` in `/config` → clients cannot negotiate capabilities.
6. `open_*` WS events exist so slash commands can pop UI on webapp; mobile has no handler yet.

---

## 4. What stock mobile does today (the "platform limitations")

| Capability | Today on mobile |
| --- | --- |
| Sticker post | message text + attached **static** file (animated → server picks `preview_key`) |
| Animated WebP / APNG / GIF sticker | static preview (not animated) |
| WebM sticker | Android: could play, but is served as file → inline video player, not a sticker; **iOS: unplayable** |
| TGS / Lottie sticker | static preview only |
| Sticker picker / packs / recents / search | none |
| Sticker reactions (pills, add/remove, who-reacted) | none |
| GIF picker | none |
| Soundboard | only as a generic audio attachment; no browse, no admin upload |
| Video messages | only as generic video attachment; **no recording**, no round bubble |
| `/stickers`, `/sb`, `/gif` | command output only; `open_*` WS events ignored |
| Import/admin (Telegram, Discord, ZIP, delete pack, sound upload) | none |
| Bandwidth | full media duplicated into channel file storage (extra upload + download) |

---

## 5. Goals / non-goals

**Goals**

1. Parity with the browser plugin for every user-facing feature, on iOS and Android.
2. One request per surface, zero polling, durable cache: opening a channel ≤ **3** plugin requests,
   scrolling a channel = **0**.
3. Every post type renders with a defined fallback chain — never a blank post, never a crash.
4. Capability negotiation so the app degrades cleanly against plugin 0.2.x/0.3.x and against servers
   where the plugin is absent or disabled.
5. Fork stays mergeable: all new code inside `app/products/stickers/**`, ≤ ~12 guarded edits elsewhere,
   documented upstream-sync runbook + CI check.
6. Mobile-only wins (not possible in the browser build): camera-roll/share ingest, hold-to-record round
   video, offline queue, PiP video messages, haptic quick-react, data-saver mode, on-device admin.

**Non-goals (this cycle)**

* Generic third-party plugin runtime / JS plugin loading on mobile (kept as seam, see D1).
* Server-side transcoding service (WebM→MP4, TGS→video) — evaluated separately (see §11).
* Replacing the plugin's file-attachment fallback (it stays, optionally flagged — stock clients need it).
* Calls/Boards/Playbooks interactions beyond not regressing them.

---

## 6. Parity matrix

Legend: ✅ parity · ➕ mobile-only · 🟡 degraded-with-reason · ⛔ out of scope

| # | Feature | Browser | Stock mobile | Target |
| --- | --- | --- | --- | --- |
| 1 | Sticker post render (static) | ✅ | 🟡 static file | ✅ |
| 2 | Sticker post render (animated webp/apng/gif) | ✅ | ⛔ static | ✅ |
| 3 | Sticker post render (webm) | ✅ | ⛔ | ✅ Android / 🟡 iOS (poster + "play in browser" + optional rendition) |
| 4 | Sticker post render (tgs/lottie) | ✅ | ⛔ | 🟡 poster default, ✅ via `lottie-react-native` (D4) |
| 5 | Send sticker from picker | ✅ | ⛔ | ✅ |
| 6 | Reply with sticker (post menu) | ✅ | ⛔ | ✅ |
| 7 | Sticker picker: pack tabs, search, recents | ✅ | ⛔ | ✅ + ➕ favourites, long-press preview |
| 8 | Packs manager: list/delete | ✅ | ⛔ | ✅ |
| 9 | Import: Telegram / Discord / ZIP / images | ✅ | ⛔ | ✅ (document picker + paste link) + ➕ paste from clipboard, camera-roll |
| 10 | Popular-pack catalog one-tap install | ✅ | ⛔ | ✅ |
| 11 | Sticker reactions: row, pills, counts, remove | ✅ | ⛔ | ✅ + ➕ haptics, long-press quick-react |
| 12 | Who-reacted sheet | ✅ | ⛔ | ✅ |
| 13 | Sticker reaction live updates | ✅ (refetch) | ⛔ | ✅ (payload pushed, no refetch) |
| 14 | GIF picker: search / trending / local | ✅ | ⛔ | ✅ + ➕ recent searches, keyboard-safe sheet |
| 15 | Send GIF post | ✅ | ⛔ | ✅ |
| 16 | Soundboard: browse + post | ✅ | ⛔ | ✅ + ➕ background-friendly player, one-audio-at-a-time |
| 17 | Soundboard: admin upload/delete | ✅ | ⛔ | ✅ + ➕ record from device mic |
| 18 | Video messages: playback (round bubble) | ✅ | 🟡 plain video | ✅ + ➕ PiP, viewability-gated autoplay |
| 19 | Video messages: record & send | ✅ webcam | ⛔ | ✅ native camera + ➕ hold-to-record, front/back, compress |
| 20 | `/stickers`, `/sb`, `/gif` commands | ✅ (+ UI popup) | 🟡 text only | ✅ + ✅ popups via WS |
| 21 | Slash autocomplete of stickers/sounds | ✅ | ✅ generic | ✅ (already server-driven) |
| 22 | Offline browse of cached packs | ⛔ | ⛔ | ➕ |
| 23 | Queue/send offline, auto-retry | ⛔ | ⛔ | ➕ |
| 24 | Share target: image/video → sticker (`share_extension`) | ⛔ | ⛔ | ➕ |
| 25 | Data-saver / Wi-Fi-only media policy | ⛔ | ⛔ | ➕ |

---

## 7. Architecture (fork side)

### 7.1 Module layout — mirrors the existing `products/*` convention

```
app/products/stickers/
  constants/plugin.ts        # PLUGIN_ID, post types, WS event names, API_VERSION, format policies
  constants/screens.ts       # merged into app/constants/screens.ts
  client/api.ts              # typed plugin REST client (config, packs, recents, reactions, sounds, gifs, videos)
  client/upload.ts           # multipart upload with progress/cancel/retry (video, sound, zip)
  database/schema.ts         # WatermelonDB tables (see §7.3)
  database/models/*.ts
  database/queries.ts
  actions/remote/*.ts        # sync packs (delta), reactions (batch), sounds, gifs, uploads
  actions/local/*.ts         # DB writes, optimistic updates
  actions/websocket/*.ts     # custom_* event handling
  components/post_content/   # StickerPost, GifPost, SoundPost, VideoMessagePost, UnknownPost
  components/media/          # StickerImage, AnimatedSticker, LottieSticker, RoundVideo, Poster
  components/reaction_row/   # pills, add-button, long-press menu
  components/picker/         # pack tabs, grid, search, recents, empty states
  components/audio/          # shared single-player coordinator (reuses ProgressBar + react-native-video)
  screens/                   # sticker_picker, packs_manager, gif_picker, soundboard, video_record, viewer
  hooks/                     # useStickersCapabilities, useVisibleItem, usePacksQuery
  utils/*.ts
  types/*.ts
  index.ts
```

Add `@stickers` alias in `babel.config.js` (module-resolver) and `tsconfig.json` `paths` — same 2 lines
that `@agents`, `@boards`, `@playbooks`, `@calls` already have.

Screen routes follow the existing pattern: `app/routes/(bottom_sheet)/sticker_picker.tsx` etc. as thin
re-exports of the product screens, so the router file list stays upstream-shaped.

### 7.2 Touchpoints in upstream files (deliberately tiny)

| File | Change | Risk |
| --- | --- | --- |
| `app/constants/screens.ts` | `...STICKERS_SCREENS` (1 line, like playbooks/agents) | none |
| `app/components/post_list/post/body/body.tsx` | one branch: `if (isStickersPost(post.type)) body = <StickersPost …/>` (playbooks precedent) | low |
| `app/components/post_list/post/body/content/content.tsx` | nothing, or hide duplicate attached file when custom renderer is active (`props.stickers_rendered=1`) | low |
| `app/components/post_draft/quick_actions/quick_actions.tsx` | 2–4 capability-gated quick actions | low |
| `app/screens/post_options/post_options.tsx` + `options/` | "React with sticker", "Reply with sticker", "Save sticker" | low |
| `app/actions/websocket/event.ts` | `default:` → `if (msg.event.startsWith('custom_')) handlePluginEvent(...)` (1 case, 3 lines) | low |
| `app/client/rest/index.ts` | `ClientStickers` mixin (uses existing `getPluginRoute(id)` + `doFetch`) | low |
| `app/database/schema/server/index.ts` + `migration/server/index.ts` | add stickers tables, bump version 21→22 | medium (conflict hotspot, additive) |
| `assets/base/i18n/en.json` | new keys (additive) | none |
| `app/actions/remote/entry/*` | optional: prefetch capabilities after entry sync | low |

Everything else is new files. A weekly `git merge --no-commit --no-ff upstream/main` dry-run in CI keeps
the conflict surface visible.

### 7.3 Data model (WatermelonDB, product-scoped like `@playbooks/database/schema`)

```
stickers_packs       id, name, description, author, source, created_at, updated_at, deleted_at, installed_at
stickers_stickers    id, pack_id, name, emoji, format, width, height, size, file_key, preview_key, position
stickers_reactions   post_id, user_id, pack_id, sticker_id, created_at          (index: post_id)
stickers_recents     user_id, pack_id, sticker_id, used_at                       (index: used_at)
stickers_sounds      id, name, content_type, size, duration_ms, created_at
stickers_capability  server_url, api_version, plugin_version, config_json, etag, fetched_at
```

All tables are additive and rebuildable (except `capability`, which is tiny). `stickers_reactions`
is the key to killing the N+1: hydrate once per visible channel window.

Alternatives considered: memory-only cache (loses offline + cold-start benefits), plain JSON files in
`expo-file-system` (no queries, no observable updates). DB wins; the migration file is the documented
conflict resolution point during upstream merges.

### 7.4 Network layer

* New `ClientStickers` mixin on `ClientBase`: plugin routes via existing `getPluginRoute(PLUGIN_ID)`
  and `doFetch` → inherits auth, CSRF, timeout, cancellation, telemetry from
  `@mattermost/react-native-network-client`.
* **Conditional GET**: store `ETag` per resource; send `If-None-Match`; `304` → keep cache, zero bytes.
* **Coalescing**: one in-flight request per key, subscribers share the promise; abort on unmount.
* **Delta sync**: `/packs?since=<ms>` (plugin ≥ 0.4) → upsert changed packs, delete `deleted` ids;
  fallback to full sync on older plugins (with `?stickers=0` catalog pass when available).
* **Batching**: `/reactions?post_ids=a,b,c` (plugin ≥ 0.4); fallback: paged per-post fetch limited to
  posts with a sticker post-type or a cached reaction row, 4-way concurrency.
* **Budget guard**: a debug-only counter (`requests/channel-open`, `bytes/window`, `cache-hit-rate`)
  wired into the existing `network_performance_manager` patterns.

### 7.5 WebSocket

Single new `default` case in `app/actions/websocket/event.ts` parsing `custom_<plugin id>_<event>`:

| Event | Handling |
| --- | --- |
| `sticker_reaction_updated` | upsert `stickers_reactions` from payload (0.4) or refetch that post (0.3) |
| `packs_changed` *(new in 0.4)* | schedule delta pack sync (debounced 2s, foreground only) |
| `sounds_changed` *(new in 0.4)* | refresh soundboard list cache |
| `open_picker` / `open_soundboard` / `open_gif` | open the matching bottom sheet (parity with webapp) |

No periodic timers; refresh only on foreground with a 60s floor and ETag revalidation.

### 7.6 Post rendering & fallback chain

`StickersPostContent` dispatches on `post.type`, then resolves a *render plan*:

```
custom_sticker:  props ≥ v0.3 → animated renderer (format policy)
                 props incomplete → attached file preview → message text
custom_video_message: poster + round player (Range stream) → attached mp4 → message text
custom_sound:    inline player via single-player coordinator → audio attachment → text
custom_gif:      image or video renderer by content_type → attached file → text
unknown custom_*: attached files (stock behaviour) + "Open in Stickers" affordance
```

Rules: never remove/rename existing props (forward compat); ignore unknown props; treat missing
`width/height` as unknown → measure on load; clamp to a max aspect ratio so a broken pack cannot blow
up the list layout.

### 7.7 Composer, pickers, reactions

* Quick actions (capability-gated): Sticker, GIF, Soundboard, Video message — bottom sheets, keyboard-safe,
  `FlashList` grids with `recyclingKey`, `?preview=1` thumbnails in grids, full file only in the viewer.
* Sending: sticker/GIF/sound post via `POST /posts` / `/sounds/{id}/post` / `/gifs/from_*`; attachments
  for video message via multipart upload then post. Reply-aware (`root_id`), optimistic pending post,
  retry with backoff, offline queue (M5).
* Reactions: pills with sticker thumbnails under the post (own component, not the emoji `Reactions`),
  optimistic add/remove, long-press → remove, tap → who-reacted sheet, WS-driven live updates,
  and a `React with sticker` entry in the post options sheet.
* Admin: packs manager (import/delete/popular), soundboard manager (upload via document picker or
  on-device recording, delete). All gated on `config.can_manage_packs` / admin flags returned by `/config`.

---

## 8. Plugin-side work — "mobile API v2" (plugin 0.4.0)

Backwards compatible by construction: new fields/endpoints only; old clients ignore them.

1. `/config` += `api_version: 2`, `plugin_version`, `features: [...]`, `post_types: {...}`,
   `max_upload_bytes`, `max_sticker_bytes`, `limits`.
2. `ETag` + `Cache-Control: private, max-age=30` on `/config`, `/packs`, `/sounds`, `/recents`,
   `/reactions/*`; honour `If-None-Match` → `304`.
3. `GET /packs?since=<ms>&stickers=0|1&page=&per_page=` → `{packs, deleted, cursor, etag}`.
4. `GET /reactions?post_ids=a,b,c` (max 100) → map of post → reactions; parallel KV reads,
   short-lived in-memory cache.
5. WS: `sticker_reaction_updated` carries `{post_id, reactions}`; add `packs_changed`, `sounds_changed`.
6. Media: `?poster=1` (JPEG poster for video), `?w=`/`?h=` thumbnail rendition for GIF/video grids,
   `HEAD` support, keep existing Range + immutable caching.
7. Post props: add `v` (props schema version), `duration_ms`, `poster` hint, `fallback: 1` marker so the
   fork can hide the duplicated attachment while stock clients keep using it.
8. Video upload: accept `mp4`/`mov`, client-reported `duration_ms`; keep size/duration limits from config.
9. Optional (M5): async import jobs (`202` + `GET /jobs/{id}`) so mobile can resume a big Telegram import.

---

## 9. Battery & efficiency design

Targets (device-measured, 30-post channel, 5 min soak):

| Metric | Browser today | Target |
| --- | --- | --- |
| Plugin requests on channel open | ~1 per post (+1 per reaction update) | **≤ 3** |
| Requests while scrolling | 0 (but refetch on every reaction WS event) | **0** |
| Bytes on warm open | full catalogs + media | **< 20 KB** JSON (304s) + cached media |
| Concurrent media decoders | unbounded | **≤ 1 video + 1 audio** |
| Autoplay | n/a | only ≥60 % visible, app active, screen focused |
| Cache hit rate (warm) | — | **> 90 %** media, 100 % catalog |

Techniques:

1. **No polling.** WS invalidation + foreground revalidation with `If-None-Match` and a 60s floor.
2. **Delta + pagination** for packs/stickers; catalog-only pass (`?stickers=0`) on cold start.
3. **Batching** for reactions (`post_ids`) and for capability probes (one per server per 24h).
4. **Media caching**: `expo-image` `memory-disk` cache with immutable URLs; grid thumbnails
   (`?w=`) in pickers; `priority: low` in lists, `high` on interaction; `recyclingKey` for FlashList.
5. **Playback policy**: a single global media coordinator guarantees at most one player; `paused`
   bound to `visible && appActive && screenFocused`; `progressUpdateInterval` only while playing;
   player released on unmount/background; posters instead of video on cellular when data-saver is on.
6. **Uploads**: compress before upload (round video: H.264 480p, ~1.5 Mbps, ≤ plugin `max_video_bytes`),
   multipart with progress + cancel + exponential backoff, Wi-Fi-only option, reuse
   `draft_upload_manager` retry semantics.
7. **Render hygiene**: `withObservables`/memoized queries, memoized post rows, no inline handlers in
   `post_list`, `drawDistance` capped, lottie only for on-screen items, `startTransition` for filter/search.
8. **Cold start**: zero plugin work before the first channel open; capability probe deferred and cached.
9. **Instrumentation**: request counters, byte counters, cache-hit ratio, dropped-frame counter, plus
   Xcode Energy Log / `adb shell dumpsys batterystats` runs in the release checklist.

---

## 10. Backwards compatibility

| Environment | Expected behaviour |
| --- | --- |
| Plugin **absent/disabled** (`404`/`403` on `/config`) | no sticker UI anywhere; posts render as today (attached file); negative cache for the session + "retry" affordance; **no** retry storms |
| Plugin **0.2.x** (no sound/gif) | sticker + video-message features only; no soundboard/GIF entries |
| Plugin **0.3.x** (current) | full read/write; per-post reaction fetch; full pack sync; no `?poster`/thumbnails |
| Plugin **≥ 0.4** | all optimisations (batch, delta, ETag, WS payloads, renditions) |
| Server older than plugin `min_server_version` | plugin not installable — treated as absent |
| Stock mobile client + new plugin | unchanged: attached-file fallback keeps working (`fallback: 1` marker is ignored) |
| Fork + stock server / other plugins | untouched: bridge only activates on the configured plugin id |
| Multi-server (fork) | capabilities, caches and WP state are keyed per `serverUrl` |
| Forward compat | unknown props/events/formats ignored; unknown `custom_*` → generic renderer + files |
| Upstream compat | new code isolated in `app/products/stickers/**`; ≤12 guarded edits; additive DB migration; documented sync runbook |

---

## 11. Format support matrix (target)

| Format | iOS | Android | Notes |
| --- | --- | --- | --- |
| PNG / JPG / WebP (static) | ✅ | ✅ | expo-image |
| GIF | ✅ | ✅ | expo-image animated |
| Animated WebP | ✅ (spike) | ✅ | verify SDWebImage/Glide animated coders; poster fallback if not |
| APNG | ✅ (spike) | ✅ | same spike; poster fallback |
| WebM (VP8/VP9) | 🟡 poster + "open in browser" | ✅ | `react-native-video`/ExoPlayer plays WebM; AVPlayer does not |
| TGS / Lottie | 🟡 poster → ✅ with `lottie-react-native` | same | D4 decides: bundle cost vs. true parity |
| MP4 (video messages) | ✅ | ✅ | Range streaming + poster |
| MP3 / M4A / WAV | ✅ | ✅ | react-native-video audio |
| OGG / Opus | 🟡 | ✅ | m4a preferred for clips recorded on iOS |

Optional (separate ticket, only if iOS WebM parity is a hard requirement): server-side rendition pass
(WebM → MP4/H.264, TGS → MP4) at import time, cached next to the original and advertised via
`config.features`.

---

## 12. Milestones

Estimates are dev-days for one experienced RN dev; parallelisable across 2 people.

| M | Scope | Key deliverables | Est. |
| --- | --- | --- | --- |
| **M0** | Fork hygiene *(done)* + guardrails | upstream base v2.45.0, legacy branch, CI job (`tsc`, `lint`, `jest`), upstream-sync runbook, this doc | 1 ✅ |
| **M1** | Plugin bridge & data plane | constants, `ClientStickers` (+ETag), capability probe/cache, DB schema+models+queries, delta pack sync, WS routing, offline read path | 3–4 |
| **M2** | Post rendering | 4 post renderers + fallback chain + duplicate-attachment suppression + fullscreen viewer + format spike (webp/apng) | 4–5 |
| **M3** | Compose & pick | sticker picker (tabs/search/recents), send + reply, GIF picker (Tenor/local), soundboard picker + player, video message via `launchCamera` | 5–6 |
| **M4** | Reactions | pills/row, post-menu quick-react, who-reacted sheet, optimistic updates, batch fetch, WS live updates | 3–4 |
| **M5** | Management + mobile extras | packs manager, imports (Telegram/Discord/ZIP), popular catalog, soundboard admin, share-sheet ingest, offline queue, PiP, data saver | 4–5 |
| **M6** | Hardening & release | perf/energy soak, detox e2e smoke (`send sticker`, `receive+render`, `react`), a11y, i18n, docs, release notes, upstream merge rehearsal | 2–3 |

**Critical path to a usable beta: M1 → M2 → M3 → M4 ≈ 15–19 dev-days.**
Plugin 0.4.0 (API v2) runs in parallel with M1/M4 and is a hard dependency only for the *optimised* paths.

---

## 13. Testing & metrics

* **Unit (jest)**: client (URL/ETag/304/coalescing), reducers/DB transformers, post renderers with
  fixture posts for every prop-shape (v0.1 → v0.4, missing props, unknown format), capability gating,
  WS handlers, upload retry/backoff.
* **Integration**: mock plugin API server (MSW-style or jest mocks) covering 0.3 vs 0.4 vs absent.
* **E2E (detox)**: send sticker, render received sticker, add/remove reaction, open GIF picker,
  record+send round video (simulator camera), offline queue replay.
* **Performance gate (CI nightly)**: JS bundle delta, cold-start time delta (< 100 ms), request counter
  assertions from the instrumentation harness, `tsc`/`lint` clean.
* **Manual matrix**: iOS 18/26 device + simulator, Android 11/15/16, dark/light, RTL, data-saver on/off,
  low-memory device, 200-pack catalog, 500-sticker pack, offline → online transition.

---

## 14. Risks

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Fork drift vs monthly upstream releases | rising merge cost | isolated product dir, ≤12 touchpoints, weekly merge dry-run in CI, sync runbook, product-scoped DB schemas |
| Upstream renames a touchpoint (e.g. post body) | broken build | tiny diffs → trivial resolution; CI catches immediately |
| WatermelonDB schema version conflicts | migration pain | additive tables only, one migration file, documented resolution |
| WebM/TGS on iOS | parity gap | poster-first UX + optional server rendition; explicit decision D4 |
| New native deps (camera/lottie) need pods & rebuild | release friction | MVP uses `launchCamera` (already bundled) to avoid new pods |
| Video autoplay energy regressions | battery complaints | single-coordinator playback policy + visibility gating + data-saver |
| Plugin KV reaction reads at scale | latency | batch endpoint with parallel reads + short TTL cache; cap posts per request |
| App Store review (UGC stickers) | rejection | admin delete (exists), report link, no private APIs, follow UGC guidelines |
| Test flakiness (network mocks, timers) | CI noise | deterministic fixtures, fake timers, no real network in unit tests |

---

## 15. Open decisions

* **D1 — Fork-native vs. generic mobile plugin host.** Recommend fork-native now, with a
  `PluginBridge` interface (id → capabilities, client, post renderers, WS handlers, screens) so a
  generic host can be layered later without rewriting the stickers module.
* **D2 — Camera.** MVP: `launchCamera` from `react-native-image-picker` (no new native deps) for
  round video; later `expo-camera`/`vision-camera` for hold-to-record + in-app preview + torch.
* **D3 — Cache.** WatermelonDB (offline, queryable) vs memory-only. Recommend DB as scoped in §7.3.
* **D4 — TGS/Lottie.** Add `lottie-react-native` (bundle + pod cost) for true parity, or poster-only.
* **D5 — Upstreaming.** Offer the `PluginBridge` + `custom_*` WS dispatch upstream as a small generic
  feature; if accepted, fork drift drops a lot. Low probability, high value.
* **D6 — Keep the server-side file fallback?** Recommend yes (stock clients), with `props.fallback=1`
  so the fork hides the duplicate attachment.

---

## 16. Immediate next steps (first two weeks)

1. M0 leftovers: CI workflow (`tsc`, `lint`, `jest`) + `docs/plugins/upstream-sync-runbook.md`.
2. Fork the plugin to 0.4.0-alpha with `/config` `api_version`, batch reactions, `packs?since`, ETag.
3. M1 skeleton: `app/products/stickers/{constants,client,database}` + `@stickers` alias + capability probe
   behind a debug flag (`STICKERS_PLUGIN_ID` in `.env`/config).
4. M2 spike (timeboxed 1 day): animated WebP + APNG on both platforms with `expo-image`; record findings
   in this doc's §11.
5. Decide D1/D3 (architecture) and D4 (lottie) — they gate M2/M3 scope.

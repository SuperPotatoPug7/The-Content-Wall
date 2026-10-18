# The Content Wall — Architecture Plan (v1)

A Chrome extension (Manifest V3) that wraps YouTube so that Shorts are consumed from a **finite, static wall** with a **global context-switch cooldown**, instead of from the infinite vertical feed.

This document turns the rough planning page into a build-ready plan: fixed decisions, precise behavioural rules, component architecture, YouTube integration surface, storage schema, project layout, milestones, and risks.

---

## 1. Design thesis

The harm model we are building against is not short-form video itself but two presentation mechanics:

| Mechanic | How YouTube Shorts does it | What we do instead |
|---|---|---|
| **Slot-machine scrolling** | Infinite vertical feed, one swipe = new random reward, no end state | A finite wall of thumbnails fixed per page load. No load-more, no refresh. New content only appears when you deliberately navigate to a new page. |
| **Rapid context switching** | Autoplay to the next video, zero-cost swipe, 5s clips chained indefinitely | A global cooldown (default 30s) that starts when a short starts playing. Shorts that are at least as long as the cooldown carry a ✓ and cost nothing extra; shorter ones carry a ⚠ and force a wait. Autoplay is disabled; the short loops instead. |

Everything in the extension serves one of those two rows. If a feature doesn't, it is out of scope.

**Why the cooldown starts at playback begin (not end):** combined with the ✓/⚠ rule this makes the penalty proportional to *switching speed*, not viewing. A 45s short is fully "paid for" by the time it ends. A 6s short leaves 24s of enforced pause. The user is nudged toward longer, more coherent clips without being punished for watching.

---

## 2. Scope

### v1 (this plan)
- Redirect every `/shorts/<id>` entry point (direct links, shared links, in-app clicks, SPA navigation) to `/watch?v=<id>`.
- Redirect the bare `/shorts` feed root to the homepage.
- Hide the "Shorts" entry in the left guide navigation.
- **Homepage:** replace the native Shorts shelf in place with our wall.
- **Watch page:** inject our wall at the top of the right-hand sidebar (fed by the homepage shelf, cached), keep native long-form recommendations beneath it, strip any native Shorts shelves from those recommendations.
- **Subscriptions page:** strip the native Shorts shelf (no wall there in v1).
- Wall item treatment: greyed + non-interactable while pending → duration badge + ✓/⚠ → single reorder (✓ first) → unlocked when the global cooldown is clear.
- Global cooldown: persisted, cross-tab, with overlay (timer icon, countdown, splash copy) on every wall.
- Hard enforcement: opening any YouTube-classified Short on `/watch` during cooldown shows a full-player interstitial and keeps the video paused until the timer clears.
- Short-form watch pages: autoplay-next suppressed, video loops.
- Options page: cooldown seconds (default 30), per-feature toggles.
- Chrome Web Store release quality: icons, listing, privacy policy, graceful degradation when YouTube changes.

### v2 (explicitly deferred)
- Treating Shorts grids on channel pages and search results (length badges + lock). These are a minor circumvention path with enough natural friction to defer.
- Firefox/Edge builds.
- Faster duration lookup (InnerTube player endpoint or YouTube Data API). Only worth the extra maintenance surface if the extension becomes popular; the `DurationService` interface is designed so this slots in without touching the wall.
- Local usage stats (shorts watched per day, time saved).

### Out of scope
- Any dedicated full-page wall at `/shorts`. The homepage shelf *is* the wall.
- Manual "refresh wall" or "load more". Finite means finite.
- Any change to long-form video behaviour beyond stripping embedded Shorts shelves.
- Mobile / YouTube Music / embedded players on third-party sites.
- Remote config, telemetry, accounts.

---

## 3. Decision log

| # | Decision | Rationale |
|---|---|---|
| D1 | Surfaces in v1: homepage shelf (in place) + watch-page sidebar. | Covers the two places users actually encounter Shorts organically. |
| D2 | Cooldown clock starts at **first `playing` event** of a new short-form video. | Penalises switching speed, not viewing (see §1). |
| D3 | ✓ iff `lengthSeconds >= cooldownSeconds`. Threshold and cooldown are the same number. | Single mental model: "is this clip long enough to pay for itself?" |
| D4 | Hard enforcement: interstitial on any short-form `/watch` during cooldown. | Soft locking only the wall leaves obvious holes (search, channel, pasted links). |
| D5 | Duration source: **scrape the watch page HTML** for `ytInitialPlayerResponse.videoDetails.lengthSeconds`, with a persistent per-video cache. | Minimise exposure to YouTube-update breakage. The watch page HTML is the most stable surface available (it is what browsers load); InnerTube endpoints and their keys churn, and the Data API adds a user-facing setup step. We would rather ship slower lookups than set an expectation of speed we can't maintain. Behind an interface so a faster source can be added if the extension gains enough users to justify the maintenance (v2). |
| D6 | Short-form watch page: suppress autoplay-next, set the video to loop. | Autoplay-next is a context switch we don't control; looping matches the Shorts player and keeps the end-screen from becoming a launchpad. |
| D7 | "Short-form" = YouTube-classified: arrived via `/shorts/` redirect, clicked from a wall, **or** player response has `isShortsEligible: true`. Fallback heuristic: vertical (`height > width`) and `lengthSeconds <= 180`. | A 45s horizontal clip from search is not a Short and shouldn't be gated. |
| D8 | Stack: TypeScript, Vite + CRXJS, vanilla DOM, Vitest. No UI framework. | Small UI surface; frameworks add bundle size and injection complexity into YouTube's Polymer page. |
| D9 | Wall is a fixed set per page load. No load-more, no refresh. | Core thesis. |
| D10 | Watch sidebar: wall on top, native long-form recommendations below, native Shorts shelves removed. | Keeps long-form discovery intact; one Shorts surface per page. |
| D11 | `/shorts` root → homepage. Hide "Shorts" guide button. Strip Shorts shelves from watch recs and Subscriptions. | Removes every path back into the feed. |
| D12 | Chrome Web Store release target, with options page. | Implies defensive selectors, feature kill-switches, privacy policy. |
| D13 | **Replace, don't mutate.** Native shelves are hidden via CSS and our own wall component is rendered from parsed data. | Mutating Polymer-rendered nodes is fragile (re-renders wipe changes). Rendering our own DOM from data is stable and gives one component for both mount points. |
| D14 | One reorder, after the whole duration queue settles (or a 10s timeout), not progressively. | Progressive shuffling is visually noisy and itself slot-machine-like. |
| D15 | Items whose duration lookup fails are badged `?`, grouped with ⚠ at the right, and remain clickable once cooldown clears. | Fail open on data, fail closed on cooldown. |
| D16 | After the interstitial countdown completes, playback auto-resumes. | The user already made the choice; the delay is the intervention. Revisit if it feels like it rewards waiting. |
| D17 | The homepage shelf is cached (default 10 min) and reused for watch-page walls. | Avoids a ~1MB homepage fetch on every watch page; less novelty is a feature, not a bug. |

---

## 4. Behavioural specification

### 4.1 Routing rules

| Incoming URL / action | Result | Mechanism |
|---|---|---|
| Full page load `youtube.com/shorts/<id>` | `302`-style redirect to `/watch?v=<id>&cw_src=shorts` | `declarativeNetRequest` static rule (regex redirect), fires before any HTML loads → no flash of the Shorts player |
| Full page load `youtube.com/shorts` or `/shorts/` (no id) | Redirect to `youtube.com/` | DNR static rule |
| SPA navigation to `/shorts/<id>` (click inside YouTube) | Intercepted; `/watch?v=<id>` pushed instead | (a) MutationObserver rewrites `a[href^="/shorts/"]` hrefs to `/watch?v=` so YouTube's own router goes to watch; (b) capture-phase click handler as belt-and-braces; (c) `yt-navigate-start` listener as last resort → `location.replace` |
| SPA navigation to `/shorts` root | Redirect to `/` | same as above |
| `cw_src=shorts` marker on `/watch` | Recorded as "arrived from Shorts", then stripped with `history.replaceState` | Watch-page controller |

### 4.2 Cooldown state machine (global, cross-tab)

```
state: { cooldownUntil: epochMs, startedByVideoId: string }

IDLE  ──(first `playing` of new short-form videoId)──▶  COOLING(until = now + cooldownSeconds*1000)
COOLING ──(now >= until)──▶ IDLE
COOLING ──(first `playing` of a DIFFERENT short-form videoId)──▶ COOLING(until = now + cooldown)   // only reachable via interstitial completing, see 4.5
```

- A "view" is the **first `playing` event for a given video ID after a navigation**. Loops/replays of the same ID do **not** restart the clock. Pausing and resuming does not restart the clock.
- Stored in `chrome.storage.local` so it survives restarts; every tab subscribes to `storage.onChanged` so a view in tab A locks the wall in tab B within ~50ms.
- Clock skew is irrelevant (single machine). Countdown rendering uses `cooldownUntil - Date.now()` on a 250ms tick, never a local decrementing counter.

### 4.3 Wall item state machine

```
PENDING   grey, no pointer events, no badge. Duration fetch queued.
   │ fetch ok
   ├──▶ RESOLVED_OK     badge "0:45 ✓"   (length >= cooldown)
   ├──▶ RESOLVED_SHORT  badge "0:12 ⚠"   (length <  cooldown)
   │ fetch failed / timed out
   └──▶ UNKNOWN         badge "?"
```

- Interactability = `state != PENDING && cooldown == IDLE`. Both conditions are computed on every render; the overlay and the per-item grey-out are driven from the same selector function.
- Order while anything is PENDING: original shelf order (stable). When the queue settles (all resolved or 10s global timeout): one stable sort `OK → SHORT → UNKNOWN`, preserving original order within each group.
- Duration badge appears as soon as that item resolves (no reorder yet). This gives the user information progressively without moving things.

### 4.4 Wall cooldown overlay

Rendered over the wall container (not per item) while COOLING:
- Timer icon, `mm:ss` countdown, one-line splash copy explaining *why* (rotating from a small set, e.g. "Short pause between shorts. Your attention will thank you.").
- `pointer-events: none` on items underneath; overlay itself swallows clicks.
- Fades out when IDLE; items un-grey if they are also resolved.

### 4.5 Watch-page interstitial (hard enforcement)

On `yt-navigate-finish` to `/watch` where the video is classified short-form (D7) **and** cooldown is COOLING **and** `startedByVideoId != this videoId`:
1. Pause the `<video>` immediately; re-pause on any `play` event until cleared (YouTube will try to autoplay).
2. Mount a full-player overlay inside `#movie_player` (so fullscreen/theatre modes still cover): same timer + copy as 4.4, plus the video's title.
3. On IDLE: unmount, call `video.play()` (D16). That `playing` event is the "view" that starts the next cooldown.

If the video is short-form and cooldown is IDLE → no interstitial; first `playing` starts the cooldown.

### 4.6 Short-form watch-page behaviour
- `video.loop = true` (re-asserted on `loadeddata` because YouTube swaps sources). With `loop` set the `ended` event never fires, so YouTube's autonav never triggers — no need to fight the autoplay toggle UI.
- Belt-and-braces: if YouTube's autonav toggle (`.ytp-autonav-toggle-button`) reports checked, click it once per session and remember.
- Long-form watch pages: untouched (except shelf stripping).

### 4.7 Removals (CSS-first)
- Guide "Shorts" entry: hidden.
- Native Shorts shelves on home, subscriptions, watch sidebar: hidden. On home the hidden shelf's slot is where our wall mounts.
- All removals are pure CSS where possible (injected stylesheet at `document_start`) so there is zero flash and zero dependence on script timing.

---

## 5. System architecture

```
┌──────────────────────────────── Chrome ────────────────────────────────┐
│                                                                        │
│  declarativeNetRequest (static rules)                                  │
│    /shorts/<id> → /watch?v=<id>&cw_src=shorts ;  /shorts → /           │
│                                                                        │
│  Background service worker (background.ts)                             │
│    • owns chrome.storage schema + migrations                           │
│    • DurationService: fetch+parse watch HTML, cache, concurrency=2     │
│    • ShelfService: fetch+parse homepage shelf, 10-min cache            │
│    • answers messages from content scripts; no DOM                     │
│                                                                        │
│  youtube.com tab                                                       │
│   ┌─ MAIN world (bridge.ts, document_start) ─────────────────────────┐ │
│   │ listens: yt-navigate-start / yt-navigate-finish                  │ │
│   │ relays event.detail (endpoint, response.playerResponse,          │ │
│   │ response.contents) to ISOLATED world via window.postMessage      │ │
│   └──────────────────────────────────────────────────────────────────┘ │
│   ┌─ ISOLATED world (content.ts, document_start) ────────────────────┐ │
│   │ Router ─ decides page kind (home / watch / subs / other)         │ │
│   │ ├ ShortsLinkRewriter (MutationObserver + click capture)          │ │
│   │ ├ Remover (CSS classes toggled by feature flags)                 │ │
│   │ ├ HomeController  → mounts <Wall> in native shelf slot           │ │
│   │ ├ WatchController → classify short-form, interstitial, loop,     │ │
│   │ │                   mounts <Wall> atop #secondary                 │ │
│   │ ├ CooldownStore (reactive wrapper over chrome.storage.local)     │ │
│   │ └ Wall component (render(data,state) → DOM; pure, testable)      │ │
│   └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  Options page (options.html/.ts) — cooldown seconds, feature toggles   │
└────────────────────────────────────────────────────────────────────────┘
```

### Why two content-script worlds
YouTube is a single-page app. Full-page `ytInitialData` is only present on the first load; every later navigation delivers its data via the `yt-navigate-finish` CustomEvent's `detail.response`. Content scripts in the default ISOLATED world receive the DOM event but **cannot read `detail`** from the page's JS world. A tiny MAIN-world script (`world: "MAIN"`, Chrome 111+) reads it and relays a sanitised JSON subset. This gives us, for free:
- on **home**: the shelf data with no extra fetch;
- on **watch**: `playerResponse.videoDetails.lengthSeconds`, `microformat.playerMicroformatRenderer.isShortsEligible`, and format dimensions → short-form classification without a scrape.

The MAIN-world script is kept to ~40 lines, has no access to extension APIs, and only ever *sends* data.

### Why a background worker for fetching
Content-script `fetch` to youtube.com works (same-origin, cookies attached), but centralising in the worker gives: one cache, one concurrency limiter across all tabs, no duplicate fetches when two tabs show the same wall, and keeps ~1MB HTML strings out of the page's memory.

---

## 6. Modules

| Module | File(s) | Responsibility | Depends on |
|---|---|---|---|
| **Manifest & DNR rules** | `manifest.json`, `rules/shorts-redirect.json` | Permissions, worlds, static redirects | — |
| **Bridge** | `src/content/bridge.main.ts` | Relay `yt-navigate-*` detail to isolated world | — |
| **Router** | `src/content/router.ts` | Normalise URL + nav events into `PageContext {kind, videoId, arrivedFromShorts}`; emits `pagechange` | Bridge |
| **ShortsLinkRewriter** | `src/content/rewriter.ts` | Rewrite `/shorts/` anchors, intercept clicks, guard `yt-navigate-start` | Router |
| **Remover** | `src/content/remover.ts`, `src/styles/remove.css` | Toggle body classes that drive hide rules per feature flag | Settings |
| **CooldownStore** | `src/shared/cooldown.ts` | Read/write `{cooldownUntil, startedByVideoId}`, `subscribe()`, `startView(videoId)`, `remaining()` | Storage |
| **Settings** | `src/shared/settings.ts` | Typed access to options + feature flags, defaults, `subscribe()` | Storage |
| **ShelfService** | `src/background/shelf.ts` | `getShelf(): ShelfItem[]` — from cache, else fetch home HTML → parse | Parsers |
| **DurationService** | `src/background/duration.ts` | `getDurations(ids): Map<id, number|null>` — cache → fetch watch HTML (concurrency 2, 8s timeout per item) | Parsers |
| **Parsers** | `src/shared/parse/initialData.ts`, `playerResponse.ts` | Pure functions: HTML → `ytInitialData` JSON; find Shorts shelf; `shortsLockupViewModel` → `ShelfItem`; player response → `{lengthSeconds, isShortsEligible, vertical}` | — |
| **Wall** | `src/content/wall/wall.ts`, `wall.css` | `mount(container)`, `render({items, states, cooldown})` → DOM. Pure given inputs. | CooldownStore |
| **HomeController** | `src/content/pages/home.ts` | On home: items from bridge payload (or ShelfService), find native shelf slot, mount Wall, run duration queue | Wall, services |
| **WatchController** | `src/content/pages/watch.ts` | Classify short-form; interstitial; loop/autonav; mount Wall atop `#secondary` | Wall, CooldownStore, services |
| **Interstitial** | `src/content/interstitial.ts` | Player overlay with countdown; pause-guard | CooldownStore |
| **Options UI** | `src/options/*` | Form bound to Settings | Settings |
| **Messaging** | `src/shared/messages.ts` | Typed request/response envelope (`{type:'GET_DURATIONS', ids}` etc.) with a single `send<T>()` helper | — |

Dependency rule: `shared/` has no DOM and no `chrome.*` except storage/runtime; `background/` has no DOM; `content/` is the only place that touches YouTube's DOM; YouTube selectors and JSON paths live **only** in `src/yt/adapters.ts` (see §7).

---

## 7. YouTube integration surface

All YouTube-specific knowledge is isolated in one adapter file so breakage is localised and testable against fixtures. Items marked **verified** were checked against live HTML on 2026-10-11; items marked **spike** must be confirmed in Milestone 0 against a **logged-in** session (the unauthenticated homepage serves no Shorts shelf).

### 7.1 Data (JSON) paths
| What | Path | Status |
|---|---|---|
| Shorts item model | `shortsLockupViewModel` | verified (search page: 25 items) |
| Item video id | `.onTap.innertubeCommand.reelWatchEndpoint.videoId` | verified |
| Item title / views | `.overlayMetadata.primaryText.content`, `.secondaryText.content` | verified |
| Item thumbnails | `.thumbnailViewModel.thumbnailViewModel.image.sources[]` | verified |
| Item accessibility text (title + views fallback) | `.accessibilityText` | verified |
| Shelf container (search) | `gridShelfViewModel.contents[]` | verified |
| Shelf container (home, logged in) | expected `richSectionRenderer.content.richShelfRenderer` with `contents[].richItemRenderer.content.shortsLockupViewModel`; title "Shorts" | **spike** |
| Shelf continuation items (the "show more" set) | expected inside same `richShelfRenderer.contents` beyond the visible row | **spike** |
| Watch sidebar Shorts shelf | expected `reelShelfRenderer` or a `shelfRenderer` wrapping `shortsLockupViewModel` | **spike** |
| Duration | `ytInitialPlayerResponse.videoDetails.lengthSeconds` (string) | verified; appears ~870KB into a 1.35MB page |
| Shorts flag | `ytInitialPlayerResponse.microformat.playerMicroformatRenderer.isShortsEligible` | verified (`true` on a Short) |
| Orientation | `streamingData.formats[0].width/height` (1080×1920 on a Short) | verified |
| Nav event payload | `yt-navigate-finish` → `detail.response.playerResponse` / `detail.response.contents` / `detail.endpoint` | **spike** (well-known, confirm shape) |

Parser strategy: never walk fixed paths. Use a generic deep-search (`findAll(obj, key)`) for `shortsLockupViewModel`, then `richShelfRenderer`/`gridShelfViewModel`/`reelShelfRenderer` as *containers* to attribute items to a shelf. Fixed paths break; key-search survives reshuffles.

Scrape note: the duration string sits past the halfway mark of the HTML, so streaming + early abort saves only ~35%. Implement it anyway (cheap), but size the cache (durations never change → cache indefinitely, LRU cap 5,000 ids) and concurrency (2) as the real controls.

### 7.2 DOM selectors (all **spike**, logged-in)
| Target | Candidate selector |
|---|---|
| Home Shorts shelf (mount slot) | `ytd-rich-section-renderer:has(ytd-rich-shelf-renderer[is-shorts])`, fallback `ytd-rich-shelf-renderer[is-shorts]` |
| Subscriptions Shorts shelf | same as above on `/feed/subscriptions` |
| Watch sidebar container | `#secondary #secondary-inner`, mount before `ytd-watch-next-secondary-results-renderer` |
| Watch sidebar Shorts shelf | `ytd-reel-shelf-renderer`, `yt-lockup-view-model:has(a[href^="/shorts/"])` ancestors |
| Guide "Shorts" entry | `ytd-guide-entry-renderer:has(a[title="Shorts"])`, mini-guide `ytd-mini-guide-entry-renderer[aria-label="Shorts"]` |
| Player root / video | `#movie_player`, `#movie_player video.html5-main-video` |
| Autonav toggle | `.ytp-autonav-toggle-button[aria-checked]` |
| Any Shorts link | `a[href^="/shorts/"], a[href*="youtube.com/shorts/"]` |

Every selector lives in `adapters.ts` with a `probe()` that returns which selectors currently match; the options page shows a "Compatibility" panel from it so users can report breakage precisely.

### 7.3 Events
- `yt-navigate-start`, `yt-navigate-finish` (document-level CustomEvents; the SPA lifecycle).
- `yt-page-data-updated` as a secondary "DOM is settled" signal for mounting.
- `<video>`: `playing`, `play`, `loadeddata`, `ended` (should never fire while looping).

---

## 8. Key data flows

### 8.1 Homepage load (full)
1. DNR: nothing to do. CSS at `document_start` hides native Shorts shelf + guide entry (no flash).
2. Bridge (MAIN) and content (ISOLATED) start. Router → `{kind:'home'}`.
3. HomeController parses `ytInitialData` from the inline `<script>` text (available to isolated world as DOM text) → `ShelfItem[]`. Also posts `SHELF_SEEN(items)` to the worker to refresh the shelf cache.
4. Waits for the native shelf slot to exist (MutationObserver, 5s timeout → fallback: mount at top of `ytd-rich-grid-renderer`).
5. Mounts Wall in `PENDING` state for all items; reads CooldownStore → overlay if COOLING.
6. Sends `GET_DURATIONS(ids)`; worker streams results back per item (`DURATION_RESOLVED`). Badges appear as they land. On `DURATIONS_SETTLED` (or 10s) → one reorder.
7. Subscribes to cooldown changes → overlay on/off, item interactability.

### 8.2 SPA navigation to a watch page (user clicks a wall item)
1. Wall item click → normal anchor navigation to `/watch?v=<id>` (YouTube router handles it). Item carries `data-cw-origin="wall"`; the click handler stores `{pendingOrigin:'wall', videoId}` in session memory.
2. Bridge relays `yt-navigate-finish` detail → Router → `{kind:'watch', videoId, arrivedFromShorts:true}`.
3. WatchController classifies short-form: origin says yes → short. (Otherwise consults bridge's `playerResponse` → `isShortsEligible` → fallback heuristic.)
4. If COOLING and started by a different id → Interstitial (4.5). Else, wait for first `playing` → `CooldownStore.startView(videoId)`.
5. Set `video.loop = true`; re-assert on `loadeddata`.
6. Mount Wall atop `#secondary-inner` using `GET_SHELF` from the worker (cache hit most of the time). Duration flow as 8.1 steps 6–7.
7. Remover hides any native Shorts shelf that appears in recommendations (CSS; selectors re-probed on `yt-page-data-updated`).

### 8.3 Cross-tab cooldown
Tab A starts a view → `storage.local.set({cooldown:{until, videoId}})` → `storage.onChanged` in tabs B, C → their CooldownStore emits → walls overlay instantly. Expiry is time-based in each tab; no message needed.

### 8.4 Duration fetch (worker)
```
GET_DURATIONS(ids)
  → split: cached vs missing
  → emit DURATION_RESOLVED for cached immediately
  → queue missing, concurrency 2, per-item 8s AbortController timeout
  → fetch(`https://www.youtube.com/watch?v=${id}`, {credentials:'include'})
  → stream body, regex /"lengthSeconds":"(\d+)"/ on a rolling window, abort on match
  → cache {id → seconds} ; emit DURATION_RESOLVED(id, seconds|null)
  → emit DURATIONS_SETTLED when queue drains
```

---

## 9. Storage schema (`chrome.storage.local`)

```ts
interface Storage {
  schemaVersion: 1;
  settings: {
    cooldownSeconds: number;         // default 30, min 5, max 600
    features: {
      redirectShorts: boolean;       // default true
      hideGuideEntry: boolean;       // default true
      homeWall: boolean;             // default true
      watchWall: boolean;            // default true
      stripSubscriptionsShelf: boolean; // default true
      stripWatchShelves: boolean;    // default true
      interstitial: boolean;         // default true
      loopShorts: boolean;           // default true
    };
  };
  cooldown: { until: number; videoId: string } | null;
  shelfCache: { fetchedAt: number; items: ShelfItem[] } | null;   // TTL 10 min
  durationCache: Record<string, number | null>;                   // LRU-capped at 5000, nulls expire after 1 day
}

interface ShelfItem {
  videoId: string;
  title: string;
  viewsText?: string;
  thumbnail: { url: string; width: number; height: number }[];
}
```

`chrome.storage.session` holds nothing persistent; per-tab transient state (pending click origin) lives in content-script memory.

---

## 10. Manifest (MV3) and permissions

```jsonc
{
  "manifest_version": 3,
  "name": "The Content Wall",
  "permissions": ["storage", "declarativeNetRequest"],
  "host_permissions": ["https://www.youtube.com/*"],
  "background": { "service_worker": "background.js", "type": "module" },
  "declarative_net_request": { "rule_resources": [{ "id": "shorts", "enabled": true, "path": "rules/shorts-redirect.json" }] },
  "content_scripts": [
    { "matches": ["https://www.youtube.com/*"], "js": ["bridge.main.js"], "world": "MAIN", "run_at": "document_start" },
    { "matches": ["https://www.youtube.com/*"], "js": ["content.js"], "css": ["remove.css"], "run_at": "document_start" }
  ],
  "options_ui": { "page": "options.html", "open_in_tab": true },
  "minimum_chrome_version": "111"
}
```

Deliberately **not** requested: `tabs`, `webNavigation`, `scripting`, `<all_urls>`. Store review is far smoother with two permissions and one host.

DNR rules (regex):
```jsonc
[
  { "id": 1, "priority": 2, "action": { "type": "redirect", "redirect": { "regexSubstitution": "https://www.youtube.com/watch?v=\\1&cw_src=shorts" } },
    "condition": { "regexFilter": "^https://www\\.youtube\\.com/shorts/([A-Za-z0-9_-]{11})", "resourceTypes": ["main_frame"] } },
  { "id": 2, "priority": 1, "action": { "type": "redirect", "redirect": { "url": "https://www.youtube.com/" } },
    "condition": { "regexFilter": "^https://www\\.youtube\\.com/shorts/?(\\?.*)?$", "resourceTypes": ["main_frame"] } }
]
```

---

## 11. Project layout

```
the-content-wall/
├── manifest.json
├── package.json            # vite, @crxjs/vite-plugin, typescript, vitest, @types/chrome, jsdom
├── vite.config.ts
├── tsconfig.json
├── rules/
│   └── shorts-redirect.json
├── public/icons/           # 16/32/48/128 png
├── src/
│   ├── background/
│   │   ├── index.ts        # message router, storage migrations
│   │   ├── shelf.ts        # ShelfService
│   │   └── duration.ts     # DurationService (queue, cache, streaming regex)
│   ├── content/
│   │   ├── bridge.main.ts  # MAIN world relay (tiny, no imports)
│   │   ├── index.ts        # boot: Router → controllers
│   │   ├── router.ts
│   │   ├── rewriter.ts
│   │   ├── remover.ts
│   │   ├── interstitial.ts
│   │   ├── pages/home.ts
│   │   ├── pages/watch.ts
│   │   └── wall/wall.ts, wall.css
│   ├── options/            # options.html, options.ts, options.css
│   ├── shared/
│   │   ├── settings.ts
│   │   ├── cooldown.ts
│   │   ├── messages.ts
│   │   ├── storage.ts      # typed get/set + schema migration
│   │   └── parse/initialData.ts, playerResponse.ts, findAll.ts
│   ├── yt/
│   │   └── adapters.ts     # ALL selectors + JSON container keys + probe()
│   └── styles/remove.css   # body.cw-hide-guide ytd-guide-entry-renderer… { display:none }
├── test/
│   ├── fixtures/           # saved ytInitialData / playerResponse JSON (sanitised), DOM snippets
│   ├── parse.test.ts
│   ├── cooldown.test.ts
│   ├── wall.test.ts        # jsdom render + state transitions + ordering
│   └── duration.test.ts    # queue, concurrency, abort, cache
├── docs/
│   ├── PRIVACY.md
│   └── STORE_LISTING.md
└── ARCHITECTURE.md (this file)
```

---

## 12. Build, test, QA

- **Dev loop:** `vite` with CRXJS → `dist/` loaded unpacked; HMR for content scripts and options page.
- **Unit (Vitest + jsdom):** parsers against fixtures (home/search/watch snapshots, captured in M0 and re-captured whenever adapters change); cooldown state machine with fake timers; wall render + ordering; duration queue with mocked `fetch` (concurrency, timeout, abort, cache hit path); router URL classification.
- **Integration (manual checklist, each milestone):** scripted walkthrough in a logged-in Chrome profile: cold load of `/shorts/<id>`, in-app click to a Short from search, home wall load, cross-tab lock, interstitial, loop, options change propagates live.
- **Optional (M6):** Playwright with a persistent profile for smoke tests of routing + wall mount. Kept optional because YouTube UI is non-deterministic and logged-in automation is brittle.
- **Lint/format:** eslint (typescript-eslint), prettier. CI: GitHub Actions running typecheck + unit tests + `vite build`.

---

## 13. Resilience strategy (Store-grade)

1. **Fail-safe defaults.** If a parser or selector fails, the native page is left alone *except* that CSS removals stay in place. Never leave the user with a broken YouTube: the worst case is "shorts shelf is hidden and no wall appeared".
2. **Kill-switches.** Every feature is a flag in Settings; the options page exposes them so a user can disable a broken module without uninstalling.
3. **Adapter probe.** `adapters.probe()` runs on every page and logs which selectors matched (debug mode only). The options page's Compatibility panel surfaces it for bug reports.
4. **Key-search parsing, not path parsing** (see §7.1).
5. **Mount timeouts.** Every "wait for element" has a timeout and a fallback mount point or a no-op.
6. **Idempotent controllers.** Navigations can fire twice; every mount checks for an existing `[data-cw-root]` and reuses it.
7. **Version-pinned fixtures.** Fixture snapshots carry a capture date; a failing parser test tells you *when* YouTube changed.

---

## 14. Milestones

Each milestone ends in a loadable, demoable extension.

| # | Milestone | Deliverable | Acceptance |
|---|---|---|---|
| **M0** | Spike & fixtures (≈1 day) | Logged-in captures of home `ytInitialData` (with shelf + continuation), watch `yt-navigate-finish` detail shape, selectors list; sanitised fixtures committed; `adapters.ts` first draft | All **spike** rows in §7 resolved to verified/changed |
| **M1** | Skeleton + routing | Vite/CRXJS project, manifest, DNR rules, bridge, router, rewriter, CSS removals, options page scaffold | `/shorts/<id>` never renders the Shorts player (cold load and in-app); `/shorts` → home; guide entry hidden; shelves hidden |
| **M2** | Cooldown core + watch page | CooldownStore, WatchController classification, loop/autonav, interstitial | Opening a Short starts a 30s cooldown visible in options debug; second Short within 30s shows interstitial, paused, auto-resumes; video loops |
| **M3** | Wall on home | Parsers, Wall component, HomeController, overlay | Home shows our wall in the shelf slot, greyed, with overlay during cooldown; clicking an item → watch page with origin flag |
| **M4** | Duration pipeline | DurationService, messaging, badges, settle-and-reorder | Badges appear within a few seconds; ✓ first after settle; cache makes a reload instant |
| **M5** | Wall on watch sidebar | ShelfService cache, mount atop `#secondary`, strip native sidebar shelves | Sidebar wall present on every watch page; recommendations intact beneath |
| **M6** | Hardening | Cross-tab tests, idempotency, probe panel, kill-switches, perf pass (no layout thrash, <50ms per nav), a11y (focus order, `aria-disabled`, reduced-motion) | Manual checklist green on two profiles; unit coverage on shared/ ≥ 80% |
| **M7** | Store release | Icons, listing copy, screenshots, privacy policy, version 1.0.0, CI | Submitted to Chrome Web Store |

Rough total: 3–4 weeks part-time. M2 and M3 are independent after M1 and can be parallelised.

---

## 15. Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| YouTube renames renderers / selectors | High (every few months) | Wall or removals break | Adapter isolation, key-search parsing, kill-switches, fixture tests that pinpoint the change |
| Scrape-based duration lookup is slow / bandwidth heavy (~1MB × N) | Medium | Pending state lingers; mobile hotspots suffer | Accepted trade-off (D5): stability over speed. Concurrency 2, streaming abort, permanent cache, 10s settle timeout. Faster source deferred to v2 behind the existing interface |
| YouTube rate-limits or serves consent/bot pages to rapid fetches | Low–Medium | Durations resolve to `?` | Treat as UNKNOWN (fail open), back off exponentially, cache negatives for 1 day |
| `yt-navigate-finish` detail shape differs from expectation | Medium | Classification falls back to heuristics | Heuristic fallback (vertical + ≤180s) plus origin flag; scrape fallback for playerResponse |
| MAIN-world script conflicts with YouTube's own JS | Low | Page errors | Script is ~40 lines, only adds listeners, no globals besides a namespaced relay |
| Interstitial fights YouTube's autoplay and causes stutter | Medium | Jank | Pause-guard on `play` events, overlay hides the video frame, resume only after cooldown |
| Store review rejects MAIN-world or DNR usage | Low | Delay | Minimal permissions, clear single-purpose description, privacy policy states no data leaves the device |
| User circumvents via channel pages / search (v2 gap) | Expected | Reduced efficacy, not breakage | Documented as v2; hard interstitial still catches the *watch* regardless of entry point |

---

## 16. Chrome Web Store checklist (M7)
- [ ] Single-purpose description matches behaviour exactly (routing, wall, cooldown).
- [ ] Privacy policy: no data collected or transmitted; all storage local; fetches go only to youtube.com with the user's own session.
- [ ] Permissions justification text for `storage`, `declarativeNetRequest`, host permission.
- [ ] Icons 16/32/48/128, 1280×800 screenshots (home wall, interstitial, options).
- [ ] Semantic versioning; CHANGELOG.
- [ ] Options page reachable from extension menu; all features toggleable.
- [ ] Reduced-motion and keyboard navigation verified.

---

## 17. Remaining small decisions (defaults chosen, change if you disagree)
1. **Splash copy set** for the overlay/interstitial: 4–6 short lines, rotating. Draft in M3.
2. **Badge format:** `m:ss` + icon; colour-blind safe (icon shape carries meaning, not just colour).
3. **Interstitial auto-resume** (D16) vs requiring a click. Default auto-resume; flip after dogfooding if it feels like a reward.
4. **Shelf cache TTL** 10 min (D17). Could be "until the next homepage visit" instead.
5. **Debug mode** flag in options that logs probe results and cooldown transitions to the console.

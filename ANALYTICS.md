# Analytics

Reference for the Google Analytics 4 event tracking in `index.html`.

Line numbers reference `index.html` as of this document's creation. The file is a
single-page monolith, so line references drift whenever code above them changes —
if a reference looks wrong, search for the event name string instead.

---

## Overview

### Tag systems

Two tag systems load on every page view:

| System | ID | Loaded at | Used by `Analytics.track()`? |
|---|---|---|---|
| Google Tag Manager | `GTM-MKNR82S2` | `index.html:5-11` (+ `<noscript>` iframe at `:2048`) | **No** |
| gtag.js (GA4) | `G-1J327MGMTX` | `index.html:59-68` | **Yes** |

`Analytics.track()` calls `gtag('event', ...)` exclusively. The GTM container is
loaded but never receives events from application code. See
[Known issues](#3-two-tag-systems-load-side-by-side).

gtag config options (`index.html:65-68`):

```js
gtag('config', 'G-1J327MGMTX', {
  anonymize_ip: true,
  cookie_flags: 'SameSite=None;Secure',
});
```

### The helper

All application events route through one helper — `Analytics.track()`, defined at
`index.html:2644-2655` and exposed as `window.Analytics` at `:2656`.

```js
const Analytics = {
  enabled: true,
  debug: false,
  track(eventName, parameters = {}) {
    if (!this.enabled) return;
    const enrichedParams = { ...parameters, app_version: '1.0.0' };
    if (typeof gtag === 'function') gtag('event', eventName, enrichedParams);
    if (this.debug) console.log('[Analytics]', eventName, enrichedParams);
  },
  enable()  { this.enabled = true;  console.log('[Analytics] Tracking enabled'); },
  disable() { this.enabled = false; console.log('[Analytics] Tracking disabled'); },
};
```

**There are no direct `gtag('event', ...)` calls anywhere else in the file.** The
only `gtag` invocation outside the config block is the one inside `track()` at
`:2650`. Keep it that way — adding a bypass means the consent gate no longer applies.

**Every event automatically carries `app_version: '1.0.0'`** (`:2649`). This is a
hardcoded string literal, not read from a manifest or build step. It will remain
`'1.0.0'` across releases unless someone edits that line, so do not use it to
segment by release.

### When events silently do not fire

Three conditions cause `track()` to no-op. None of them throw, log (in production),
queue, or retry — the event is simply lost.

| Condition | Mechanism | Effect |
|---|---|---|
| Consent declined | `Analytics.enabled === false` — set by `Analytics.disable()` from `initCookieConsent()` (`:9664`) when `localStorage['linefield_cookie_consent'] === 'declined'` | `track()` returns at `:2648` before doing anything |
| gtag script blocked | Ad blocker / CSP / network failure prevents `googletagmanager.com` from loading, so `typeof gtag !== 'function'` | Guard at `:2650` skips the call. Blocked visitors are **invisible**, not merely unattributed |
| — | `Analytics.debug` is `false` in production | No console output reveals either case above |

**Consent state is three-valued, not two.** `localStorage['linefield_cookie_consent']`
is `null` (never answered), `'accepted'`, or `'declined'`. Tracking is only disabled
on `'declined'`. See [Known issues](#2-tracking-is-active-before-consent-is-given).

Debug helpers available in the browser console:

```js
__linefield_analytics_status()   // { enabled, debug, consent, gtag_loaded }
__linefield_test_analytics()     // sets debug = true, fires test_event
__linefield_reset_consent()      // clears consent so the banner reappears
```

---

## Events

33 distinct event names across 43 call sites.

### Image handling

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `image_uploaded` | Image finishes decoding in `loadImageFile()` (`:3282`) | `image_format` (string) — raw MIME from `file.type`, e.g. `image/png`, `image/jpeg`, `image/webp`<br>`image_size_kb` (number)<br>`upload_method` (string) — `file_picker`, `drag_drop` | `image_format` is unbounded, not an enum — whatever MIME the browser reports. There is no paste-to-upload path. |
| `prepare_image_opened` | `openPrepModal()` (`:3940`) | `trigger` (string) — `upload`, `manual_button` | |
| `prepare_image_skipped` | Skip button in Prepare Image modal (`:4014`) | none | |
| `prepare_fullscreen_toggled` | Fullscreen toggle in Prepare Image modal (`:3693`) | `is_fullscreen` (boolean) | Value is the state **after** the toggle. |
| `background_removed` | Continue button in Prepare Image modal (`:4005`) | `strength` (number) — integer 0–100<br>`replacement_type` (string) — `transparent`, `white`, `black`, `custom`<br>`has_manual_edits` (boolean)<br>`mode` (string) — `remove`, `blur` | Only fires when `strength > 0` **and** a mask exists. Continuing with removal off is silent. |
| `manual_mask_brush_used` | `_setBrushTool()` (`:4184`) | `brush_type` (string) — `erase`, `restore` | Fires on tool **selection**, not on an actual brush stroke. Counts tool switches, not usage. |

### Presets

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `preset_applied` | Preset chip clicked — `applyPreset()` (`:9396`) | `preset_id` (string) — `fact_*` (9 factory) or `usr_*` (unbounded)<br>`preset_name` (string)<br>`preset_type` (string) — `factory`, `user` | Factory ids: `fact_blue_contour`, `fact_sparse_plot`, `fact_rainbow_spectrum`, `fact_green_grass`, `fact_velocity_line`, `fact_portrait_core`, `fact_topo_dense`, `fact_abstract_flow`, `fact_high_contrast`. User names are free text. |
| `preset_saved` | "Save current" chip → confirm (`:8702`) | `preset_name` (string) — free text | |
| `preset_exported_to_file` | Context menu → export `.linefield` (`:8925`) | `preset_name` (string) | |
| `preset_imported_from_file` | Single-preset import confirmed (`:9019`) | `is_pack` (boolean) — `false`<br>`preset_count` (number) — always `1` | |
| `preset_imported_from_file` | Preset **pack** import confirmed (`:9110`) | `is_pack` (boolean) — `true`<br>`preset_count` (number) — count actually imported | Same event name, two call sites. Split on `is_pack`. |

### Generation mode

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `contours_generated` | First generation producing a non-empty result after an image loads — `onWorkerMessage()` (`:3217`) | `gen_mode` (string) — `shape`, `isoluminance`<br>`base_shape` (string\|null) — `rhombus`, `circle`, `square`, `hexagon`, `triangle`, `star`; `null` when `gen_mode` is `isoluminance`<br>`contour_count` (number) — polylines actually produced<br>`levels` (number) — contours requested<br>`stroke_mode` (string) — `solid`, `per-line`, `along-line`<br>`from_preset` (boolean)<br>`time_to_first_generation_ms` (number) | **Once per loaded image, not per generation.** Generation reruns on every slider move; a one-shot flag (`_genTracked`) gates this. A generation yielding zero polylines does not consume the one-shot. `contour_count` vs `levels` is diagnostic — a large gap means parameters producing almost nothing. |
| `generation_session_ended` | User moves on from an image: a new image loads, the tab goes hidden, or the page unloads (`:3233`) | `regeneration_count` (number) — total recomputes for this image<br>`exported` (boolean)<br>`presets_tried` (number) — distinct preset ids applied<br>`session_duration_ms` (number) | Only fires if `contours_generated` already fired for that image. Guarded against double-firing. Listens on both `beforeunload` and `visibilitychange`→`hidden`, since `beforeunload` is unreliable on mobile. |
| `generation_mode_changed` | `setGenMode()` (`:7385`) | `new_mode` (string) — `isoluminance`, `shape` | Legacy values `hybrid` and `pure_shape` are rewritten to `shape` by `migrateParams()` and never reach GA. |
| `base_shape_changed` | Base shape dropdown (`:7562`) | `new_shape` (string) — `rhombus`, `circle`, `square`, `hexagon`, `triangle`, `star` | |
| `auto_tune_used` | `runAutoTune()` (`:7270`) | none | Fires on **start**, not completion. No success/failure signal. |
| `parameters_reset` | Reset-to-defaults button (`:7334`) | none | |

### Exports

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `export_completed` | Export SVG button (`:5096`) | `export_method`: `simple_svg`<br>`export_format`: `svg`<br>`has_watermark` (boolean)<br>`background_mode`: `photo`, `lines` | **Conversion event.** |
| `export_completed` | Export PNG button (`:5143`) | `export_method`: `simple_png`<br>`export_format`: `png`<br>`has_watermark` (boolean)<br>`background_mode`: `photo`, `lines` | **Conversion event.** |
| `export_completed` | Copy-to-clipboard button (`:5223`) | `export_method`: `simple_copy_clipboard`<br>`export_format`: `svg_clipboard` | **Conversion event.** No watermark/background params on this path. |
| `export_completed` | Advanced Export single-file download, via `_advDownload()` (`:6537`) | `export_method`: `advanced`<br>`export_tab`: `social`, `print`, `plotter`, `batch`<br>`export_format` (string) — filename extension<br>`watermark_position` (string)<br>`background_color` (string) — raw hex | **Conversion event.** Guarded by `if (!_advBatchRunning)` at `:6536`. |
| `export_completed` | Advanced Export batch ZIP complete (`:6668`) | `export_method`: `advanced`<br>`export_tab`: `batch` (literal)<br>`export_format`: `zip` (literal)<br>`watermark_position` (string) | **Conversion event.** Also fires the row above — see [Known issues](#1-batch-export-fires-export_completed-twice). |
| `advanced_export_opened` | `openAdvExport()` (`:5836`) | none | Modal opened, not an export. |

### Stroke coloring

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `stroke_color_mode_changed` | Stroke Coloring segmented control — `setStrokeMode()` (`:7678`) | `new_mode` (string) — `solid`, `per-line`, `along-line` | |
| `gradient_preset_selected` | Gradient preset dropdown `change` (`:7722`) | `gradient_name` (string) — `copper-fade`, `nostromo-fade`, `mono-fade`, `sunset`, `ocean`, `heat`, `spectrum`, `rainbow`, `duotone-cm` | Keys of `BUILTIN_GRADIENTS` (`:4763`). Applying a gradient via a saved user swatch is **not** tracked. |
| `gradient_inverted` | Invert-gradient toggle (`:7761`) | `is_inverted` (boolean) | |

### Background removal

Background-removal events are documented under [Image handling](#image-handling) —
`background_removed` and `manual_mask_brush_used` — since they fire from the
Prepare Image modal. The failure path is `error_occurred` with
`error_type: 'bg_removal_failed'`, documented under [Errors](#errors).

### Monetization

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `support_modal_shown` | Post-export celebration modal displayed (`:5459`) | `export_count` (number) — running total from localStorage<br>`frequency_mode` (string) — `always`, `smart`, `rare`, `off` | `frequency_mode` reads `MONETIZATION.supportModalFrequency`, currently `always` (`:2640`). |
| `support_clicked` | Any Buy-Me-A-Coffee link — 5 surfaces (`:5609`, `:5626`, `:5639`, `:5724`, `:9198`) | `source` (string) — `corner_button`, `post_export_modal`, `about_page`, `tour_modal`, `settings_panel` | |
| `support_dismissed` | Celebration modal closed without click-through (`:5614`, `:5618`, `:5622`) | `source` (string) — `post_export_modal` (only value)<br>`action` (string) — `close_x`, `click_outside`, `maybe_later` | |

### Social

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `instagram_clicked` | Any of the three Instagram links — top bar, post-export celebration modal, About modal (`:5603`) | `source` (string) — `top_bar`, `export_modal`, `about_modal` | **One call site, three sources.** The links are real `<a target="_blank">` elements, so the browser performs the navigation; the listener only records the source and must never `preventDefault`. Adding a fourth entry point means adding a `source` value here, not a new event. |

### Onboarding

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `tour_first_view` | Tour modal opens and `tourSeen` is falsy (`:5700`) | none | Selected by a ternary at the same line as `tour_reopened`. |
| `tour_reopened` | Tour modal opens and `tourSeen` is already set (`:5700`) | none | |
| `tour_completed` | "Start" button in the tour modal (`:5722`) | none | **Misleading name** — this is a dismissal, not a walkthrough completion. There are no per-step events. |
| `cookie_consent_accepted` | Accept in the consent banner (`:9677`) | none | No decline counterpart — `Analytics.disable()` runs first, so opt-out is structurally untrackable. |

### Color picker

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `color_picker_opened` | `_cpOpen()` — custom swatch picker opened (`:9856`) | `context` (string) — `stroke_color`, `background`, `gradient_stop`, `prep_bg_custom`, `adv_background`, `other` | Derived from the triggering input's element id. |
| `color_picker_native_used` | "More…" inside the picker → OS color dialog (`:9827`) | none | |

### Errors

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `error_occurred` | Background-removal inference `catch` block (`:3801`) | `error_type` (string) — `bg_removal_failed` (only value)<br>`error_message` (string) — truncated to 100 chars, `unknown` fallback | **Only one call site** despite the generic name. Other user-visible failures are untracked — see [Untracked gaps](#untracked-gaps). |

### Debug

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `test_event` | `window.__linefield_test_analytics()` from the console only (`:9694`) | `test_param` (string) — `hello` | **Not reachable through the UI.** Also flips `Analytics.debug = true` for the session. Exclude or filter in GA4. |

---

## Conversion events

**`export_completed` is the primary conversion metric.** It has 5 call sites and is
the only event representing a completed image output. `advanced_export_opened` is
intent, not conversion — the modal can be opened and abandoned.

Every parameter below must be registered as an event-scoped custom dimension in the
GA4 interface before it appears in reports.

| Parameter | Type | Full domain | Cardinality notes |
|---|---|---|---|
| `export_method` | string | `simple_svg`, `simple_png`, `simple_copy_clipboard`, `advanced` | 4 values. `advanced` covers both single-file and batch. |
| `export_format` | string | `svg`, `png`, `svg_clipboard`, `zip`, `jpg`, `webp`, `pdf`, `plt`, `gcode` | 9 values. On the advanced path this is `filename.split('.').pop()`, so PDF/X-1a, PDF/X-4 and plain PDF all collapse to `pdf`; a missing jsPDF library silently downgrades the file to `svg` (`:6493`); HPGL is written as `plt`. |
| `export_tab` | string | `social`, `print`, `plotter`, `batch` | Advanced exports only; absent on the three quick-export paths. |
| `watermark_position` | string | `NW`, `N`, `NE`, `W`, `C`, `E`, `SW`, `S`, `SE` | 9 values, default `S`. Advanced exports only. |
| `background_mode` | string | `photo`, `lines` | Quick SVG and PNG exports only. |
| `has_watermark` | boolean | `true`, `false` | Quick SVG and PNG exports only. |
| `background_color` | string | arbitrary hex, e.g. `#1A1814` | **Unbounded** — free color picker. Fires only from `:6537`. Bucket it rather than registering raw. |

Parameter presence varies by call site — no single `export_completed` carries all
seven. Reports that filter on `export_tab` will exclude all quick exports.

---

## Known issues

### 1. Batch export fired `export_completed` twice — RESOLVED

**Was: high — inflated the primary conversion metric.**

`_advExportBatch()` cleared `_advBatchRunning` *before* calling `_advDownload()`,
so the guard at `:6536` passed and the download fired its own `export_completed`
on top of the batch's. `_onExportComplete()` also ran twice, double-incrementing
`exportCount`.

Fixed by wrapping the function body in `try` / `finally` and releasing the flag in
the `finally`. This keeps the guard active through the batch's own download, and
also closes a second defect: three early `return` paths and any thrown error
previously left the flag stuck `true`, which silently suppressed
`export_completed` for every later advanced export in that session.

**GA4 note:** batch `export_completed` counts before this fix are inflated
roughly 2×. Segment on `export_tab = batch` when comparing across the fix date.

### 2. Tracking is active before consent is given

**Severity: high — compliance.**

`Analytics.enabled` initialises to `true` (`:2645`). `initCookieConsent()` (`:9664`)
only calls `Analytics.disable()` when stored consent is exactly `'declined'`:

```js
if (consent === null) {
  banner.classList.remove('hidden');   // banner shown, tracking still ON
} else if (consent === 'declined') {
  Analytics.disable();
}
```

On a first visit consent is `null` — the banner displays while tracking runs, so
every event fired before the visitor clicks anything is collected. Only an explicit
decline stops it.

**Fix:** default `enabled` to `false`, or call `Analytics.disable()` when
`consent === null` and enable only on accept.

### 3. Two tag systems load side by side

**Severity: medium — possible double-counting.**

GTM container `GTM-MKNR82S2` (`:5-11`) and gtag property `G-1J327MGMTX` (`:59-68`)
both load. Application code only calls `gtag()`. If the GTM container also holds a
GA4 configuration tag pointing at the same property, every event is counted twice.

**This cannot be determined from the source** — it depends on the container
configuration. Verify inside the GTM interface before trusting absolute numbers, and
either remove the GTM snippet or confirm it holds no GA4 tag for this property.

### 4. Silent failure when the script is blocked

**Severity: medium — systematic undercount.**

```js
if (typeof gtag === 'function') gtag('event', eventName, enrichedParams);  // :2605
```

If an ad blocker, CSP, or network failure prevents `googletagmanager.com` from
loading, the call is skipped with no error, no queue, and no retry. `Analytics.debug`
is `false` in production, so nothing surfaces in the console either.

Visitors with blockers are **invisible rather than merely unattributed**, which
biases every metric toward users who do not run blockers. Given the audience —
plotter and generative-art users — that skew is likely non-trivial.

**Mitigation:** consider a first-party measurement proxy, or at minimum accept and
document the bias when reporting absolute numbers.

---

## Untracked gaps

Significant user actions that currently emit nothing. Backlog for a future pass.

| Gap | Where | Why it matters |
|---|---|---|
| **Export failures** | Clipboard write failure (`:5225`), jsPDF-unavailable downgrade (`:6493`), print-too-large abort (`:6596`) | Each shows a toast or `alert()` but emits no `error_occurred`. Export failure rate is unmeasurable. |
| **Preset lifecycle beyond create** | `deletePreset()` (`:8749`), `renamePreset()` (`:8741`), `duplicatePreset()` (`:8717`), drag-reorder | Preset retention and churn cannot be measured. |
| **Undo / redo** | `historyUndo()`, `historyRedo()` (`:2779-2780`) | No signal for which controls users back out of. |
| **Consent decline** | `initCookieConsent()` (`:9679`) | No counterpart to `cookie_consent_accepted`; `Analytics.disable()` runs first. Opt-out rate is unmeasurable by design. |
| **License / unlock modal** | `LICENSE` module (`:2659`), unlock modal handlers | Dormant purchase flow has no instrumentation. Will matter if `MONETIZATION.mode` is switched to `purchase`. |
| **Watermark repositioning** | `_setWmPos()` (`:5236`) | Position is reported on export but changing it is not tracked. |
| **Individual parameter tweaks** | Sliders for `levels`, `weight`, `alpha`, `contrast`, etc. | Only mode-level switches are tracked. No signal on which controls drive engagement. |
| **User gradient swatches** | `renderGradUserSwatches()` click handler (`:8009`) | Applying a saved gradient is untracked, unlike `gradient_preset_selected` for built-ins. |
| **Image removal / replacement** | — | No event when a user discards or swaps the loaded image. |

---

## Maintenance

When adding a new event:

- [ ] **Add the call.** Use `Analytics.track('event_name', { ... })` — never call
      `gtag('event', ...)` directly, or the event bypasses the consent gate.
- [ ] **Name it `snake_case`**, consistent with the existing 30. Make the name
      describe what happened, not where the code lives (`tour_completed` is a
      cautionary example — it fires on dismissal).
- [ ] **Keep parameter values low-cardinality.** Prefer a fixed enum over free text
      or raw hex. If a value is unbounded, bucket it before sending.
- [ ] **Add a row** to the relevant table in this file: event name, trigger with
      line reference, every parameter with its type and full value domain, and any
      gotcha in Notes.
- [ ] **Add the name** to [All event names](#all-event-names) below, keeping the
      list alphabetical.
- [ ] **Register the event** in the GA4 interface (Admin → Events) if it needs to be
      a conversion or appear in custom reports.
- [ ] **Register each new parameter** as a custom dimension (Admin → Custom
      definitions → Custom dimensions), event-scoped. Unregistered parameters are
      collected but not reportable — and GA4 caps custom dimensions, so do not
      register parameters nobody will query.
- [ ] **Verify it fires.** Run `__linefield_analytics_status()` to confirm tracking
      is enabled, then use GA4 DebugView or the Network tab.

When adding a new **conversion** event, also update
[Conversion events](#conversion-events) with the full parameter domain.

---

## All event names

33 names. `test_event` is console-only and should be excluded or filtered in GA4.

```
advanced_export_opened
auto_tune_used
background_removed
base_shape_changed
color_picker_native_used
color_picker_opened
contours_generated
cookie_consent_accepted
error_occurred
export_completed
generation_mode_changed
generation_session_ended
gradient_inverted
gradient_preset_selected
image_uploaded
instagram_clicked
manual_mask_brush_used
parameters_reset
prepare_fullscreen_toggled
prepare_image_opened
prepare_image_skipped
preset_applied
preset_exported_to_file
preset_imported_from_file
preset_saved
stroke_color_mode_changed
support_clicked
support_dismissed
support_modal_shown
test_event
tour_completed
tour_first_view
tour_reopened
```

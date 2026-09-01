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
| Google Tag Manager | `GTM-MKNR82S2` | `index.html:5-11` (+ `<noscript>` iframe at `:2007`) | **No** |
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
`index.html:2599-2610` and exposed as `window.Analytics` at `:2611`.

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
`:2605`. Keep it that way — adding a bypass means the consent gate no longer applies.

**Every event automatically carries `app_version: '1.0.0'`** (`:2604`). This is a
hardcoded string literal, not read from a manifest or build step. It will remain
`'1.0.0'` across releases unless someone edits that line, so do not use it to
segment by release.

### When events silently do not fire

Three conditions cause `track()` to no-op. None of them throw, log (in production),
queue, or retry — the event is simply lost.

| Condition | Mechanism | Effect |
|---|---|---|
| Consent declined | `Analytics.enabled === false` — set by `Analytics.disable()` from `initCookieConsent()` (`:9511`) when `localStorage['linefield_cookie_consent'] === 'declined'` | `track()` returns at `:2603` before doing anything |
| gtag script blocked | Ad blocker / CSP / network failure prevents `googletagmanager.com` from loading, so `typeof gtag !== 'function'` | Guard at `:2605` skips the call. Blocked visitors are **invisible**, not merely unattributed |
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

30 distinct event names across 40 call sites.

### Image handling

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `image_uploaded` | Image finishes decoding in `loadImageFile()` (`:3178`) | `image_format` (string) — raw MIME from `file.type`, e.g. `image/png`, `image/jpeg`, `image/webp`<br>`image_size_kb` (number)<br>`upload_method` (string) — `file_picker`, `drag_drop` | `image_format` is unbounded, not an enum — whatever MIME the browser reports. There is no paste-to-upload path. |
| `prepare_image_opened` | `openPrepModal()` (`:3836`) | `trigger` (string) — `upload`, `manual_button` | |
| `prepare_image_skipped` | Skip button in Prepare Image modal (`:3910`) | none | |
| `prepare_fullscreen_toggled` | Fullscreen toggle in Prepare Image modal (`:3589`) | `is_fullscreen` (boolean) | Value is the state **after** the toggle. |
| `background_removed` | Continue button in Prepare Image modal (`:3901`) | `strength` (number) — integer 0–100<br>`replacement_type` (string) — `transparent`, `white`, `black`, `custom`<br>`has_manual_edits` (boolean)<br>`mode` (string) — `remove`, `blur` | Only fires when `strength > 0` **and** a mask exists. Continuing with removal off is silent. |
| `manual_mask_brush_used` | `_setBrushTool()` (`:4080`) | `brush_type` (string) — `erase`, `restore` | Fires on tool **selection**, not on an actual brush stroke. Counts tool switches, not usage. |

### Presets

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `preset_applied` | Preset chip clicked — `applyPreset()` (`:9243`) | `preset_id` (string) — `fact_*` (9 factory) or `usr_*` (unbounded)<br>`preset_name` (string)<br>`preset_type` (string) — `factory`, `user` | Factory ids: `fact_blue_contour`, `fact_sparse_plot`, `fact_rainbow_spectrum`, `fact_green_grass`, `fact_velocity_line`, `fact_portrait_core`, `fact_topo_dense`, `fact_abstract_flow`, `fact_high_contrast`. User names are free text. |
| `preset_saved` | "Save current" chip → confirm (`:8550`) | `preset_name` (string) — free text | |
| `preset_exported_to_file` | Context menu → export `.linefield` (`:8773`) | `preset_name` (string) | |
| `preset_imported_from_file` | Single-preset import confirmed (`:8867`) | `is_pack` (boolean) — `false`<br>`preset_count` (number) — always `1` | |
| `preset_imported_from_file` | Preset **pack** import confirmed (`:8958`) | `is_pack` (boolean) — `true`<br>`preset_count` (number) — count actually imported | Same event name, two call sites. Split on `is_pack`. |

### Generation mode

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `generation_mode_changed` | `setGenMode()` (`:7254`) | `new_mode` (string) — `isoluminance`, `shape` | Legacy values `hybrid` and `pure_shape` are rewritten to `shape` by `migrateParams()` and never reach GA. |
| `base_shape_changed` | Base shape dropdown (`:7431`) | `new_shape` (string) — `rhombus`, `circle`, `square`, `hexagon`, `triangle`, `star` | |
| `auto_tune_used` | `runAutoTune()` (`:7139`) | none | Fires on **start**, not completion. No success/failure signal. |
| `parameters_reset` | Reset-to-defaults button (`:7203`) | none | |

### Exports

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `export_completed` | Export SVG button (`:4990`) | `export_method`: `simple_svg`<br>`export_format`: `svg`<br>`has_watermark` (boolean)<br>`background_mode`: `photo`, `lines` | **Conversion event.** |
| `export_completed` | Export PNG button (`:5037`) | `export_method`: `simple_png`<br>`export_format`: `png`<br>`has_watermark` (boolean)<br>`background_mode`: `photo`, `lines` | **Conversion event.** |
| `export_completed` | Copy-to-clipboard button (`:5117`) | `export_method`: `simple_copy_clipboard`<br>`export_format`: `svg_clipboard` | **Conversion event.** No watermark/background params on this path. |
| `export_completed` | Advanced Export single-file download, via `_advDownload()` (`:6413`) | `export_method`: `advanced`<br>`export_tab`: `social`, `print`, `plotter`, `batch`<br>`export_format` (string) — filename extension<br>`watermark_position` (string)<br>`background_color` (string) — raw hex | **Conversion event.** Guarded by `if (!_advBatchRunning)` at `:6412`. |
| `export_completed` | Advanced Export batch ZIP complete (`:6540`) | `export_method`: `advanced`<br>`export_tab`: `batch` (literal)<br>`export_format`: `zip` (literal)<br>`watermark_position` (string) | **Conversion event.** Also fires the row above — see [Known issues](#1-batch-export-fires-export_completed-twice). |
| `advanced_export_opened` | `openAdvExport()` (`:5712`) | none | Modal opened, not an export. |

### Stroke coloring

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `stroke_color_mode_changed` | Stroke Coloring segmented control — `setStrokeMode()` (`:7547`) | `new_mode` (string) — `solid`, `per-line`, `along-line` | |
| `gradient_preset_selected` | Gradient preset dropdown `change` (`:7591`) | `gradient_name` (string) — `copper-fade`, `nostromo-fade`, `mono-fade`, `sunset`, `ocean`, `heat`, `spectrum`, `rainbow`, `duotone-cm` | Keys of `BUILTIN_GRADIENTS` (`:4657`). Applying a gradient via a saved user swatch is **not** tracked. |
| `gradient_inverted` | Invert-gradient toggle (`:7630`) | `is_inverted` (boolean) | |

### Background removal

Background-removal events are documented under [Image handling](#image-handling) —
`background_removed` and `manual_mask_brush_used` — since they fire from the
Prepare Image modal. The failure path is `error_occurred` with
`error_type: 'bg_removal_failed'`, documented under [Errors](#errors).

### Monetization

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `support_modal_shown` | Post-export celebration modal displayed (`:5352`) | `export_count` (number) — running total from localStorage<br>`frequency_mode` (string) — `always`, `smart`, `rare`, `off` | `frequency_mode` reads `MONETIZATION.supportModalFrequency`, currently `always` (`:2595`). |
| `support_clicked` | Any Buy-Me-A-Coffee link — 5 surfaces (`:5485`, `:5502`, `:5515`, `:5600`, `:9046`) | `source` (string) — `corner_button`, `post_export_modal`, `about_page`, `tour_modal`, `settings_panel` | |
| `support_dismissed` | Celebration modal closed without click-through (`:5490`, `:5494`, `:5498`) | `source` (string) — `post_export_modal` (only value)<br>`action` (string) — `close_x`, `click_outside`, `maybe_later` | |

### Onboarding

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `tour_first_view` | Tour modal opens and `tourSeen` is falsy (`:5576`) | none | Selected by a ternary at the same line as `tour_reopened`. |
| `tour_reopened` | Tour modal opens and `tourSeen` is already set (`:5576`) | none | |
| `tour_completed` | "Start" button in the tour modal (`:5598`) | none | **Misleading name** — this is a dismissal, not a walkthrough completion. There are no per-step events. |
| `cookie_consent_accepted` | Accept in the consent banner (`:9524`) | none | No decline counterpart — `Analytics.disable()` runs first, so opt-out is structurally untrackable. |

### Color picker

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `color_picker_opened` | `_cpOpen()` — custom swatch picker opened (`:9703`) | `context` (string) — `stroke_color`, `background`, `gradient_stop`, `prep_bg_custom`, `adv_background`, `other` | Derived from the triggering input's element id. |
| `color_picker_native_used` | "More…" inside the picker → OS color dialog (`:9674`) | none | |

### Errors

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `error_occurred` | Background-removal inference `catch` block (`:3697`) | `error_type` (string) — `bg_removal_failed` (only value)<br>`error_message` (string) — truncated to 100 chars, `unknown` fallback | **Only one call site** despite the generic name. Other user-visible failures are untracked — see [Untracked gaps](#untracked-gaps). |

### Debug

| Event name | Trigger | Parameters | Notes |
|---|---|---|---|
| `test_event` | `window.__linefield_test_analytics()` from the console only (`:9541`) | `test_param` (string) — `hello` | **Not reachable through the UI.** Also flips `Analytics.debug = true` for the session. Exclude or filter in GA4. |

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
| `export_format` | string | `svg`, `png`, `svg_clipboard`, `zip`, `jpg`, `webp`, `pdf`, `plt`, `gcode` | 9 values. On the advanced path this is `filename.split('.').pop()`, so PDF/X-1a, PDF/X-4 and plain PDF all collapse to `pdf`; a missing jsPDF library silently downgrades the file to `svg` (`:6369`); HPGL is written as `plt`. |
| `export_tab` | string | `social`, `print`, `plotter`, `batch` | Advanced exports only; absent on the three quick-export paths. |
| `watermark_position` | string | `NW`, `N`, `NE`, `W`, `C`, `E`, `SW`, `S`, `SE` | 9 values, default `S`. Advanced exports only. |
| `background_mode` | string | `photo`, `lines` | Quick SVG and PNG exports only. |
| `has_watermark` | boolean | `true`, `false` | Quick SVG and PNG exports only. |
| `background_color` | string | arbitrary hex, e.g. `#1A1814` | **Unbounded** — free color picker. Fires only from `:6413`. Bucket it rather than registering raw. |

Parameter presence varies by call site — no single `export_completed` carries all
seven. Reports that filter on `export_tab` will exclude all quick exports.

---

## Known issues

### 1. Batch export fires `export_completed` twice

**Severity: high — inflates the primary conversion metric.**

In `_advExportBatch()`:

```js
_advBatchRunning = false;              // :6537
_advDownload(zipBlob, `${root}.zip`);  // :6538  -> fires export_completed at :6413
_advSetProgress(-1);
Analytics.track('export_completed', {...});  // :6540  -> fires it again
```

Line 6537 clears the flag **before** the download call, so the guard
`if (!_advBatchRunning)` at `:6412` passes and `_advDownload()` fires its own
`export_completed`. Line 6540 then fires a second. The two payloads differ only in
that the first carries `background_color`.

`_onExportComplete()` also runs twice, so the localStorage `exportCount`
double-increments and the support-modal cadence drifts.

**Fix:** move `_advBatchRunning = false` to after the `_advDownload()` call, or drop
the redundant `:6540` call entirely.

### 2. Tracking is active before consent is given

**Severity: high — compliance.**

`Analytics.enabled` initialises to `true` (`:2600`). `initCookieConsent()` (`:9511`)
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
| **Contour generation** | `scheduleGenerate()` (`:4575`) | The app's core action has no event. The funnel jumps from `image_uploaded` straight to `export_completed`, so there is no visibility into how many generations happen between, or how many users generate but never export. Highest-value gap. |
| **Export failures** | Clipboard write failure (`:5119`), jsPDF-unavailable downgrade (`:6369`), print-too-large abort (`:6472`) | Each shows a toast or `alert()` but emits no `error_occurred`. Export failure rate is unmeasurable. |
| **Preset lifecycle beyond create** | `deletePreset()` (`:8597`), `renamePreset()` (`:8589`), `duplicatePreset()` (`:8565`), drag-reorder | Preset retention and churn cannot be measured. |
| **Undo / redo** | `historyUndo()`, `historyRedo()` (`:2734-2735`) | No signal for which controls users back out of. |
| **Consent decline** | `initCookieConsent()` (`:9526`) | No counterpart to `cookie_consent_accepted`; `Analytics.disable()` runs first. Opt-out rate is unmeasurable by design. |
| **License / unlock modal** | `LICENSE` module (`:2614`), unlock modal handlers | Dormant purchase flow has no instrumentation. Will matter if `MONETIZATION.mode` is switched to `purchase`. |
| **Watermark repositioning** | `_setWmPos()` (`:5130`) | Position is reported on export but changing it is not tracked. |
| **Individual parameter tweaks** | Sliders for `levels`, `weight`, `alpha`, `contrast`, etc. | Only mode-level switches are tracked. No signal on which controls drive engagement. |
| **User gradient swatches** | `renderGradUserSwatches()` click handler (`:7878`) | Applying a saved gradient is untracked, unlike `gradient_preset_selected` for built-ins. |
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

30 names. `test_event` is console-only and should be excluded or filtered in GA4.

```
advanced_export_opened
auto_tune_used
background_removed
base_shape_changed
color_picker_native_used
color_picker_opened
cookie_consent_accepted
error_occurred
export_completed
generation_mode_changed
gradient_inverted
gradient_preset_selected
image_uploaded
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

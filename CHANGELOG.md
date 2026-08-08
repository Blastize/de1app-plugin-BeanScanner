# Bean Scanner — Changelog

## v0.4.1 — the longest capture size no longer wraps

**Safety status: no change.** The only write remains the DYE next-shot
update after you press Accept. No database, no history files.

### Fixed

* **"Capture size" wrapped to a second line at `2048x1536`**, colliding with
  the row beneath it. Not a near-miss: the value column is what's left after
  the label column and the button are subtracted, and at `sec_label_w` 220 it
  came to 232 virtual units — about 121 physical px — while the longest value
  the UI can produce needs ~120 px at `font_primary`. It was always going to
  wrap at the largest capture size.

  `sec_label_w` is now 180, giving the value column 312 virtual units
  (~163 px). The longest label, "Overwrite existing", still fits comfortably.

Found by screenshotting the running settings page for the repo README.

### Verified on the tablet

v0.4.1 loads clean and the settings page renders with `2048x1536` on one
line. This run also confirms **v0.4.0's palette adoption on real hardware** —
the pages pick up the active skin's dark/amber colours rather than the stock
light grey, which v0.4.0's own entry still listed as untested.

## v0.4.0 - adopts the active skin's palette

The plugin no longer looks like a stock settings page pasted into a themed
skin.

**Safety status: no change.** The only write remains the DYE next-shot
update after you press Accept. No database, no history files.

### Added

- `_adopt_skin_palette`, called at the end of layout init. When the active
  skin publishes a colour array it maps those tokens onto the plugin's own
  and every page follows the skin - dark or light - instead of the fixed
  light grey.

Only `::lumen::C` is recognised today. The contract is just a plain array of
`#RRGGBB` values, so any other skin can opt in by exposing the same keys:
`bg glass glass_2 glass_brd ink ink_3 crema warn`.

### Why it is written defensively

Every lookup is guarded and only overwrites a token when the skin actually
provides a `#RRGGBB` value, so:

- no skin, or a skin without a palette -> the stock look, unchanged
- a partial palette -> only the tokens present are adopted

The plugin still paints its own page background rather than inheriting the
theme's, so contrast stays deterministic. It adopts *colours*, not the
theme's rendering.

Text tokens deliberately come from the skin's panel inks, not its page inks:
this plugin's body text sits on section cards, not on the page.

### Verified

Four palette states under `tclsh`: no skin, Lumen dark, Lumen light, and a
partial palette with a token missing (only that token falls back). Plus the
0.3.0 navigation scenarios, still passing.

Not yet tested on the tablet.

## v0.3.0 - sub-pages can return where you came from

Fixes a dead end when the plugin is entered directly at a sub-page, as the
Lumen skin does with its "Scan bag" button.

**Safety status: no change.** The only write remains the DYE next-shot
update after you press Accept. No database, no history files.

### Fixed

- `_settings_return_page` and `_entered_at_subpage` now have namespace-level
  defaults. `_exit_settings` reads the first WITHOUT a catch, so when the
  plugin was entered at a sub-page and nothing had assigned it yet, Done
  threw `can't read "_settings_return_page": no such variable` and silently
  did nothing - a button that looked dead, with no error on screen. Giving
  them defaults makes that failure impossible.

### Added

- `::plugins::BeanScanner::set_return_page <page>` - public entry point for
  callers that jump straight to a sub-page. Records where to go back to and
  marks that the settings page was skipped. Refuses empty, transient
  (espresso/steam/rinse/...) and BeanScanner_* pages, returning 0 so the
  caller can tell.
- `_exit_subpage` now returns to that page when the plugin was entered at a
  sub-page, instead of always handing back to `BeanScanner_settings`. Cancel
  from the camera is one tap home rather than two.

Entering through the settings page is unchanged: it clears the flag in its
own `show`, so its sub-pages still hand back to it as before.

### Verified

Six navigation scenarios exercised under `tclsh` with the framework stubbed:
deep-link then Cancel, settings-page entry then Cancel, settings-page entry
then Done, the old crash case with nothing ever set (now falls through to
`close_dialog` rather than throwing), a repeated Cancel, and rejection of a
transient return page.

Not yet tested on the tablet.

## v0.2.0 — 2026-07-25 (write semantics changed — read this entry)

**Safety status: the write path is unchanged in scope — still only
`::plugins::DYE::shots::source_next_from`, still only from the review page's
Accept button, still no SQL and no `history/` access. What changed is *what*
that write contains: with "Overwrite existing" on, an enabled field the bag
does not state is now written as EMPTY instead of being skipped.**

### Fixed

* **A blank field kept the previous bag's value.** Reported from the first
  real end-to-end scan: a bag with no roaster name printed on it inherited
  the previous bag's roaster, so the next shot recorded the new bean under
  the old roaster — a bag that never existed, saved into shot history.

  A scan describes one bag, so with "Overwrite existing" on every enabled
  field is now written, blanks included. Verified against DYE first:
  `source_next_from` assigns `settings(next_$field)` and `::settings($field)`
  unconditionally (`plugins/DYE/DYE.tcl:1615`), and its zero-coercion special
  case applies only to `number` fields — all five bean fields are
  category/text/long_text, so an empty value genuinely clears rather than
  being dropped or turned into 0.

  With "Overwrite existing" **off** the intent is the opposite (fill only
  what is blank), so nothing is ever cleared in that mode.

### Added

* **Guard against a destructive no-op:** if the scan recognized nothing at
  all, no write happens and the user is told to retake the photo — a failed
  read can never wipe the next-shot description.
* **The review page shows what will be cleared.** Each blank field that
  currently holds a value is marked "— will be cleared" in a warning colour,
  and the subtitle states which mode is active. Accept never clears anything
  silently.
* The applied-fields log line now names the cleared fields.

## v0.1.3 — 2026-07-25 (tablet-verified)

Layout fixes found by screenshotting the running pages. **No write
behavior exists beyond the DYE next-shot update behind Accept**, unchanged.

### Fixed

* **Capture page: status text overlapped the preview.** The status line sat
  just above the bottom bar, but the preview is a Tk photo whose height is
  in physical pixels and is not known at layout time, so nothing below it
  can be placed safely. Status now sits above the preview, in the toolbar
  band. (The "no overlap, ever" rule applies to images too, not just text
  and buttons.)
* **Diagnostics: dead space** between the title and the body text; the body
  now starts in the toolbar band instead of the list band.

### Verified on the tablet

Plugin loads and `main()` runs clean; settings page renders correctly;
Diagnostics reports `numcameras: 2`, `CAMERA: declared, granted=1`,
`DYE loaded: yes`; the capture page opens a live front-camera preview; and
leaving the page releases the camera (Android reports all devices `closed`).
Not yet exercised: capture → API → review → Accept end to end.

## v0.1.2 — 2026-07-25 (the button-render bugfix)

### Fixed

* **No button on any page had a background, and labels on the white cards
  were invisible** — the Scan card looked completely empty. Root cause found
  by instrumenting the running app rather than guessing: `dui aspect set`
  writes into the **current** theme, and by the time this plugin's `preload`
  runs the skin has switched the current theme to `DSx2` — but the pages are
  created with `-theme default`. Aspect lookup falls back *from* a named
  theme *toward* `default`, never the other way, so the `bsc_btn` style was
  never found at render time: `shape` resolved empty (no background drawn)
  and the label fell back to the theme default white. The diagnostic build
  logged `theme=DSx2` outright. Fix: register the style with
  `dui aspect set -theme default`, matching the pages.
  *(GrindAdvisor's identical `-style` pattern only appears to work because
  its style is equally ignored and the unstyled `default.dbutton.*` fallback
  happens to look right — it loads before the skin switches themes.)*
* **Page background is now painted by the plugin** (`_page_bg`, first item on
  every page) instead of inheriting the skin theme's, which is dark under
  DSx2 and made the dark-on-light text unreadable. Contrast is now
  deterministic on any skin, and all colours route through tokens
  (`page_bg` / `fg_*` / `on_card_*`) with no literal left below the token
  block.

## v0.1.1 — 2026-07-25 (first tablet run: visual fixes)

Bugfix pass from the first on-tablet render. No behavior or safety change:
**no write behavior was added or altered in this version** — the only write
remains the DYE next-shot update behind the review page's Accept button.

### Fixed

* **Invisible buttons.** Every `dbutton` rendered without a background, and
  its label was white-on-white inside the section cards — the "Scan" card
  looked completely empty. Cause: dui resolves a button's fill as
  `[dui aspect get {dbutton shape} fill -style $style]`
  (`de1app-core/dui.tcl:10070`), and a custom style defining only
  `shape`/`radius` resolves fill to **empty** rather than falling back to
  `default.dbutton.fill`. The `bsc_btn` style now sets `fill`,
  `disabledfill`, and a matching `dbutton_label` fill explicitly.
* **Near-invisible text on the page background.** The fpdialog background is
  dark under this skin's theme, but the page titles, subtitles, status text,
  review rows, diagnostics and help body were all coloured for a light
  background (`#2b2b2b` / `#444444` / `#666666`). Introduced explicit colour
  tokens — `fg_title` / `fg_body` / `fg_muted` for the dark page, and
  `on_card_title` / `on_card_label` / `on_card_value` for the white section
  cards — and routed every item through them. No colour literal remains
  below the token block.
* **Wrapped value text.** A row with no button reserved button width anyway,
  so a long value (the model id `claude-opus-5`) wrapped onto a second line
  and pushed into the next row. Button-less rows now span to the card edge.

## v0.1.0 — 2026-07-25 (superseded by v0.1.1)

First version. Camera capture, AI vision recognition, review page, and a
confirmed write into DYE's next shot.

**Safety status:** the only write this version performs is
`::plugins::DYE::shots::source_next_from`, reached solely from the review
page's Accept button. There is no SQL of any kind, no `history/` or
`history_v2/` access, no direct `::settings` write, and no raw sensor data is
read or displayed. The plugin writes one file only if the user types an API
key into the settings page (the framework's own plugin settings store).

### Added

* **Camera capture** via AndroWish `borg camera` — open / start / live
  preview polled into a Tk photo / `takejpeg` + `jpeg` byte retrieval, with
  configurable preview and capture resolution, and a front/back/auto camera
  preference (front by default).
* **Import fallback** — "Use Latest Photo" picks the newest JPEG from
  `/sdcard/DCIM/Camera`, so the feature still works on app builds whose
  manifest does not declare `android.permission.CAMERA`.
* **Two switchable vision providers** — Anthropic Messages API
  (`/v1/messages`, base64 image block) and OpenAI Chat Completions
  (`/v1/chat/completions`, `image_url` data URI). Each keeps its own model id
  and API key.
* **Asynchronous HTTPS** — `http::geturl -command` over TLS 1.2 so the Tk
  event loop keeps running and the tablet UI does not freeze during a
  multi-megabyte upload.
* **Strict JSON extraction** — the prompt forbids guessing and requires
  `null` for anything not legible; the parser strips markdown fences and
  surrounding prose, tolerates a leading thinking block in Anthropic
  responses, and maps `null` / "not printed" / "unknown" to empty.
* **Review page** — shows roaster, beans, roast date, roast level and
  composed notes before anything is written. Accept / Rescan / Cancel.
* **DYE integration** — builds on `::plugins::DYE::shots::get_next`, sets
  `clock` to 0 (DYE compares it numerically and `get_next` leaves it empty),
  and applies only enabled, non-empty fields; with "Overwrite existing" off,
  already-filled next-shot fields are left alone.
* **API key from file** — `api_key_anthropic.txt` / `api_key_openai.txt`
  next to the plugin, so a long key can be pushed instead of typed.
* **Diagnostics page** — camera probe (`numcameras`, `state`, and whether
  `android.permission.CAMERA` is declared and granted), provider / model /
  key source, DYE availability, last capture size, last HTTP result, last
  error, and a truncated last raw response.
* **Help page** covering setup, capture technique and the permission
  fallback.

### Notes

* Navigation uses the app's own mechanism: the true return page is captured
  in each page's `show{}` callback, transient machine-state pages and this
  plugin's own pages are skipped, and Done loads a verified registered page
  or falls back to `dui page close_dialog`. Failures are logged via `msg`,
  never swallowed.
* The capture page's `hide{}` callback always releases the camera, so a
  flush / rinse / steam interruption cannot leave it held.
* No output-token cap is sent to OpenAI: the parameter name differs across
  their model generations and the expected reply is a short JSON object.

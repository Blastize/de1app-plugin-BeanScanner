# Bean Scanner

A plugin for the [Decent Espresso DE1app](https://github.com/decentespresso/de1app).
Photograph a bag of coffee with the tablet camera, an AI vision model reads
the label, and — after you confirm what it read — the roaster, bean, roast
date, roast level and notes go into DYE's next shot. No more typing bag
details on a tablet keyboard.

![Bean Scanner settings page](docs/settings-page.png)

*The settings page, adopting the active skin's palette.*

## How it works

1. **Scan Bean Bag** — the camera preview opens. Hold the printed side of the
   bag up to the camera so it fills the frame, then tap **Capture**.
2. The photo goes to Claude or GPT, which is told to read only what is
   actually printed and never to guess.
3. A **review page** shows what came back. **Accept** writes it into DYE's
   next shot; **Rescan** takes another photo; **Cancel** throws it away.

Nothing is written anywhere until you press Accept.

## Before you use it

Three things you should know, because this plugin sends data off your tablet
and spends your money:

1. **Your photo is uploaded to a third party** — Anthropic or OpenAI,
   whichever provider you select. That includes whatever else is in frame.
   The camera on a mounted tablet often catches you and the room behind you,
   so point it at the bag and check the preview before capturing.
2. **You pay for each scan** on your own API key. It's one small image plus a
   short reply — a fraction of a cent per scan — but it is billed to you.
3. **The API key is stored in plaintext on the tablet**, either in a text file
   in this folder or in the plugin's settings file. A DE1 tablet is usually a
   shared kitchen or café appliance. Treat the key accordingly, and use a key
   scoped to just this purpose if your provider supports it.

## Requirements

* The **DYE** (Describe Your Espresso) plugin, enabled in
  Settings → App → Extensions. DYE owns the next-shot definition and persists
  it into shot history; Bean Scanner only fills it in.
* An **API key** for Anthropic or OpenAI.
* A tablet camera. If the app can't reach it, there's a fallback — see
  [If the camera won't start](#if-the-camera-wont-start).

### About the API key

A ChatGPT Plus or Claude Pro **subscription is not API access**. Create a key
with pay-as-you-go credit at
[platform.openai.com](https://platform.openai.com) or
[console.anthropic.com](https://console.anthropic.com).

Two ways to supply it:

1. **A file next to the plugin** (recommended) —
   `de1plus/plugins/BeanScanner/api_key_anthropic.txt` or
   `api_key_openai.txt`. Far easier than typing 100+ characters on a tablet.
2. **Settings → API key**, and type it in. The entry sits in the top half of
   the screen so the Android keyboard doesn't cover it.

The settings entry wins when both exist. Key files are gitignored and are
never uploaded anywhere except to the provider you selected.

## Install

Clone into the app's plugin folder on the tablet:

```bash
cd /sdcard/de1plus/plugins
git clone https://github.com/Blastize/BeanScanner.git BeanScanner
```

Or download the ZIP and extract it so the files land in
`de1plus/plugins/BeanScanner/` — **not** in a nested
`BeanScanner/BeanScanner/`.

Then restart the app and enable **Bean Scanner** in
Settings → App → Extensions.

## Settings

| Setting | What it does |
|---|---|
| Provider | Anthropic or OpenAI. Each keeps its own model name and key. |
| Model | The model id sent to the provider. Change the default in `plugin.tcl`. |
| API key | Opens the key entry page for the selected provider. |
| Camera | Front / Back / Auto. Front is the default (the tablet faces you). |
| Capture size | 640x480 up to 2048x1536. Bigger reads small print better but uploads slower. |
| Include bag notes | Whether origin / process / varietal / tasting notes are written to `bean_notes`. |
| Overwrite existing | **On (default):** the scan replaces the whole bean identity — fields the bag doesn't state are **cleared**. **Off:** only currently-empty fields are filled in, and nothing is ever cleared. |

### Why blank fields are cleared

A scan describes **one bag**. If a bag prints no roaster name and the blank
were simply skipped, the previous bag's roaster would sit next to the new
bag's bean name — and that combination gets saved into your shot history as a
record of a bag that never existed.

So with "Overwrite existing" on, every enabled field is written, blanks
included. The review page marks each field that is about to be cleared, and
nothing is written at all if the scan recognized nothing — a failed read
can't wipe your next shot. If you'd rather keep what's there and only fill
gaps, turn "Overwrite existing" off.

## If the camera won't start

Open **Diagnostics** and tap **Probe Camera**. It reports the camera count,
the camera state, and whether `android.permission.CAMERA` is declared in this
build of the app.

Runtime permission requests cannot add a permission the APK's manifest
doesn't declare. If Diagnostics says the permission is **not declared**, this
build can't drive the camera directly. Use the fallback instead:

* Take the photo with the tablet's own camera app.
* Back in Bean Scanner, tap **Use Latest Photo** — it picks the newest JPEG
  from the import folder (default `/sdcard/DCIM/Camera`).

## For skin authors

* **Palette adoption** — if your skin publishes a colour array, Bean Scanner
  maps it onto its own tokens so its pages follow your skin instead of the
  stock light grey. Expose an array of `#RRGGBB` values with the keys
  `bg glass glass_2 glass_brd ink ink_3 crema warn`. Every lookup is guarded:
  no palette, or a partial one, falls back cleanly.
* **Deep links** — call
  `::plugins::BeanScanner::set_return_page <your_page>` before jumping
  straight to a Bean Scanner sub-page, and Cancel/Done will return to your
  page instead of the plugin's settings page. It returns 0 if it refuses the
  page (empty, transient, or a `BeanScanner_*` page).

## What it writes

Only DYE's next-shot definition, through DYE's own
`::plugins::DYE::shots::source_next_from`, and only these fields:

`bean_brand` (roaster), `bean_type` (bean), `roast_date`, `roast_level`,
`bean_notes` (origin / process / varietal plus tasting notes).

## Safety

* No SQL of any kind. The shot database is never opened.
* `history/` and `history_v2/` are never read, written, moved or deleted.
* No raw sensor data (pressure, flow, temperature, weight, series) is read,
  displayed or modified.
* `::settings` is never written directly — the app-level bean fields are set
  by DYE as a consequence of the next-shot update, exactly as they are when
  you type the same values into DYE by hand.
* The camera is released whenever the capture page goes away, including when
  a flush, rinse or steam screen interrupts it.
* The photo is sent only to the provider you selected, and only when you
  press Capture (or Use Latest Photo). The plugin does not store it.

## Tested on

DE1PRO, app v1.46.1.1, DSx2 skin v3.30, Samsung Galaxy Tab A9 (SM-X110),
1340x800.

Versions up to v0.2.0 are verified on that hardware end to end: capture, API
call, review and Accept. v0.4.1 is confirmed on the tablet to load cleanly
and render its pages, including adopting the active skin's palette (that's
the screenshot above) and the v0.3.0 deep link — the Lumen skin's "Scan bag"
button opens the camera directly and Cancel returns to the skin's home page
in one tap. See [CHANGELOG.md](CHANGELOG.md).

## License

GPLv3 — see [LICENSE](LICENSE). Same license as the DE1app and DYE.

## Not affiliated

This is an independent community plugin. It is not affiliated with, endorsed
by, or supported by Decent Espresso, Anthropic, or OpenAI.

# Modern Dark — a dark theme for Redmine 7

A pure-CSS dark theme for Redmine 7.x. No JavaScript, no build step, no
dependencies beyond the self-hosted JetBrains Mono font shipped in this repo.
Every colour is a CSS custom property defined in one place, so the whole
theme can be re-hued by changing two numbers.

## Install

This repo's root *is* the theme folder — clone it directly into your
Redmine install's `themes/` directory, naming the target folder
`modern_dark` (that folder name is what Redmine displays as the theme name,
so keep it as `modern_dark` unless you want it to show up differently):

```bash
git clone https://github.com/FrostyKitten02/modern-dark-redmine-theme.git <redmine>/themes/modern_dark
```

You should end up with:
```
<redmine>/themes/modern_dark/stylesheets/application.css
<redmine>/themes/modern_dark/fonts/JetBrainsMono-*.woff2
```

Then:
1. Restart Redmine (themes are scanned once at boot). In a production
   deployment, also run `bin/rails assets:precompile` — or Redmine will
   recompile automatically on the next boot once it notices the new files
   are newer than its asset manifest.
2. Administration → Settings → Display → Theme → **Modern dark**.
3. Hard-refresh your browser (the old light-theme CSS may be cached).

## Re-skinning

Everything lives in the `:root` block at the top of
`stylesheets/application.css`. The two knobs that matter most:

```css
--hue-n: 262;   /* neutral surface/text tint (0-360, small effect — keep it subtle) */
--hue-a: 265;   /* accent hue: links, buttons, selected states, focus ring */
```

Change `--hue-a` alone to get a different accent colour (e.g. `150` for
green, `25` for red/orange, `292` for violet) — every accent-derived token
(`--accent-solid`, `--accent-fg`, `--focus-ring`, `--color-current-marker`,
the Open Color `blue-*`/`indigo-*` remap) is computed from it via `oklch()`.

Below the knobs, `:root` has three layers:
1. **Primitives** — the surface ladder (`--bg-sunken` … `--bg-active`),
   border and text scales, all derived from `--hue-n`.
2. **Semantic tokens** — accent variants, `--danger-fg`/`--success-fg`/
   `--warning-fg`/`--info-fg`, the code/syntax-highlighting palette.
3. **The Open Color bridge** — every `--oc-*` custom property Redmine's own
   `application.css` reads (Redmine 7 themes it through
   [Open Color](https://open-color.style/)), remapped from primitives/semantic
   tokens above. This is what actually re-themes the shipped HTML/CSS.

If you just want new colours, edit layers 1–2 and leave the bridge alone —
it derives from them already.

### How the bridge was built

Redmine's core CSS uses ~72 distinct `--oc-*` tokens. Each one was
classified by its actual role (grep'd out of the real 7.0-stable source,
not guessed) into one of four buckets per hue family:

| Role | Meaning | Example use |
|---|---|---|
| `tint` | subtle wash background | flash message bg, highlight chip |
| `border` | thin border/divider | annotate blame stripe, alert border |
| `solid` | filled badge/bar (needs light text on top) | avatar chip, gantt bar, SCM icon |
| `fg` | text/icon colour on a dark surface | links, badge text, alert titles |

Every `solid` value is contrast-checked so white/`--on-solid` text on it is
≥4.5:1, and every `fg` value is checked at ≥4.5:1 against `--bg-surface`
(most land at 7:1+, i.e. AAA). Neutral text tokens (`--text-primary` …
`--text-disabled`) are checked against all six surface levels. A couple of
tokens are pinned to a literal value instead of the bridge (`--oc-white` →
`--bg-surface` as a safety-net default, `--oc-black` left at `#000000`)
because the original token is used for two incompatible purposes (surface
*and* text-on-solid) — see the "stays bright" section in the CSS for the
~20 selectors that needed an explicit pin because of this.

## Typography

JetBrains Mono, self-hosted (`fonts/`, OFL-1.1 licence, see
`LICENSE-JetBrainsMono.txt`), used for the entire UI — not just code. Five
static weights are shipped (Regular, Italic, Medium, SemiBold, Bold); the
upstream release only ships a variable font as `.ttf`, and converting it to
`.woff2` would have pulled in a font-build dependency, so static weights it
is. Change `--font-ui` in `:root` to swap fonts entirely.

## What's covered

The bridge alone fixes the large majority of Redmine's UI, since ~90% of
core's CSS already reads Open Color variables. On top of that, this theme
explicitly overrides:
- Chrome text (top bar / header) that sits on a background no longer tied
  to the generic gray ramp.
- ~20 selectors that need guaranteed-light text on a solid accent/semantic
  fill (badges, avatars, selected context-menu row, SCM change icons, the
  calendar "today" marker).
- A handful of literal (non-variable) `white` hardcodes in core.
- Gantt: the "todo" milestone marker (was invisible — same color as the
  bridge's default dark surface), overdue-link colour, the today-line,
  and a best-effort fix for the relation-line SVG colours.
- jQuery UI (dialogs, datepicker fallback, tooltip, autocomplete) — a
  separate, un-themed stylesheet with its own hard-coded palette.
- Rouge syntax highlighting (`.syntaxhl`) — also pure hex in core, fully
  re-themed with a dedicated 10-colour code palette.
- Scrollbar, text selection, focus ring, native form control colours
  (`color-scheme: dark`, `accent-color`).

## Known limitations (please report what you see)

- **Chart.js canvases** (Reports, repository statistics) draw with
  hard-coded `rgba()` colours in JS — CSS can't reach canvas pixels. A
  `filter: invert(...)` is applied as a best-effort fix; it won't look as
  clean as a native dark palette would.
- **A few small legacy icons** (search magnifier in autocomplete fields,
  external-link icon, up-arrow, exclamation mark) are drawn as dark
  background-image glyphs that CSS can't recolour directly. A small light
  halo is placed behind each so they stay visible, rather than trying to
  redraw the icons pixel-for-pixel.
- **Wiki toolbar icons** (bold/italic/table/etc. in the edit view) — I
  could not confirm from source alone whether these render as inline SVG
  (which would already pick up theme colours) or as background images
  (which wouldn't). Please check and report.
- **The wiki-syntax cheatsheet page** (`/help/...`) uses its own separate
  stylesheet (`wiki_syntax.css`) which this theme does not currently ship
  a replacement for — the bridge's `:root` variables still apply, but a
  handful of hard-coded Rouge colours on that one page may not match.
- **User-authored wiki/Markdown content** can set arbitrary inline
  `color`/`background-color` — no theme can safely override user content.
- Tested against source only, not a live instance (by agreement — see
  below). Please install and report anything that looks wrong.

## Testing checklist

Please check these and report anything that looks off — that's the fastest
path to fixing it:

- Issues: list, detail, new/edit, filters, context menu
- Projects, roadmap/versions, project overview
- Wiki: view, edit, toolbar, preview
- Gantt (today-line, relation colours, todo/done/late bars), calendar
- Time entries / time report, boards/forums, news, search results
- Reports/charts, activity, my page
- Admin pages (incl. workflow transitions, field permissions), login
- Modals, datepicker, autocomplete, flash messages, user dropdown
- Mobile width (≤899px / ≤599px)

## Requirements

Redmine 7.x only (relies on the Open Color CSS variables introduced in
Redmine 7 core; Redmine 6.x has almost no CSS variables and would need a
much larger override sheet).

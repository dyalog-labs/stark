# Documentation Assets (Zensical)

Stylesheets, fonts and images for Dyalog project documentation built with
[Zensical](https://zensical.org/) — the successor to Material for MkDocs.

This is the Zensical counterpart to
[`documentation-assets`](https://github.com/Dyalog/documentation-assets), which
targets Material for MkDocs. The palette here is taken from the [Dyalog brand
guidelines](https://dyalogprod.gos.dyalog.com/about/brand-guidelines/); the
fonts and images are carried over from the MkDocs asset set.

## What's here

| Path                | Contents                                                     |
| ------------------- | ------------------------------------------------------------ |
| `css/dyalog.css`    | The whole theme: fonts, brand tokens, light + dark schemes    |
| `fonts/APL387`      | [APL387](https://github.com/Dyalog/APL387) — the APL glyphs, used for `apl` code fences |
| `fonts/IBMPlexSans` | Body text, weights 400/700 plus italics                       |
| `fonts/JetBrainsMono` | Non-APL code (font files included; not referenced by `css/dyalog.css` yet) |
| `images`            | Favicon / logo cube                                           |

Only the font weights the stylesheet actually declares are included, rather
than every weight of every family as in the MkDocs asset set.

### APL387

APL387 replaces APL385 Unicode — it is a redrawn and extended version of the
same design, and it is what Dyalog is standardising on. It ships as `woff2`
(68 KB, against 212 KB for the APL385 TrueType), taken from the daily release
tagged `2026-08-17` and released under the Unlicense.

The release also carries variants with OpenType features baked in — `ss01`
(dotless zeros), `ss02` (centred tilde in `⍱⍲⍫`) and `cv01` (diaeresis and dot
composition in `∵∴`). The base font keeps all three as optional features, so
they can be switched on from CSS with `font-feature-settings` instead of
shipping a second file. None are on by default.

APL385 Unicode is still named in the font stack as a `local()` fallback, so
readers who have it installed keep working glyphs if `woff2` fails to load.

### JetBrains Mono

General code — everything that is not an `apl` fence — is set in JetBrains Mono
2.304 (`woff2`, SIL OFL 1.1), replacing Go Mono. It advances 0.600 em, exactly
matching APL387, so the two faces hold the same columns and a block that mixes
them stays aligned. Go Mono had no glyphs at all for `⌶ ⍎ ⍙ ⎕`, which appear in
unlabelled fences; JetBrains Mono covers those and the rest of the common APL
set. APL387 is still chained after it so rarer glyphs never reach a system
font.

## Using it

Zensical only copies files from inside `docs_dir`, and it does **not** follow
symlinks, so these assets have to live under `docs/`:

```shell
git submodule add https://github.com/Dyalog/documentation-assetsz docs/documentation-assetsz
```

Then in `zensical.toml`:

```toml
[project]
extra_css = ["documentation-assetsz/css/dyalog.css"]

[project.theme]
variant = "classic"   # the modern variant barely uses the primary colour
logo = "documentation-assetsz/images/dyalog-logo_white.svg"
favicon = "documentation-assetsz/images/dyalog-ide-cube-2025.2.svg"
font = false          # fonts are self-hosted here, skip Google Fonts
features = ["content.code.copy", "navigation.sections"]

[[project.theme.palette]]
media = "(prefers-color-scheme: light)"
scheme = "default"
primary = "custom"    # colours come from dyalog.css, not the built-in palette
accent = "custom"
toggle.icon = "lucide/sun"
toggle.name = "Switch to dark mode"

[[project.theme.palette]]
media = "(prefers-color-scheme: dark)"
scheme = "slate"
primary = "custom"
accent = "custom"
toggle.icon = "lucide/moon"
toggle.name = "Switch to light mode"
```

## Design notes

The brand's primary colours are ten-step scales (10 lightest → 100 darkest)
rather than single hexes, so `dyalog.css` defines every step once as a custom
property and each scheme picks the step that works on its own background. That
keeps the light and dark rules to one short block apiece.

Colours are assigned by the **role** the guidelines give them, not by taste:

| Role              | Colour   | Used for                        |
| ----------------- | -------- | ------------------------------- |
| Action            | Orange   | The header bar, links, buttons  |
| Neutral Backbone  | Gunmetal | Page and code surfaces (dark)   |
| Accent            | Lavender | Hover, focus, selection         |

The header bar is Retina Searing Orange at its base in both schemes. White text
on it is only **2.87:1**, which fails at any size, so the bar's foreground and
logo are Raisin instead — 5.53:1, and the escalation the guidelines set out for
the logo: orange, then white, then Raisin, then black.

Orange also fails as *text* on white (2.87:1), so links take step 70 on light
(5.62:1) and step 30 on dark (8.05:1) rather than the base.

In dark mode the header sits one step *lighter* than the page (Gunmetal 80 over
Gunmetal 90) so raised surfaces read as raised, and code wells recess to step
100. Every applied pair clears WCAG AA; most clear AAA.

The logo cannot be orange on an orange bar, and white reaches only 2.87:1, so
the wordmark is Raisin. Orange and white variants are kept alongside it for
surfaces where they do carry. The mobile nav drawer repeats the logo over a light title bar, so
`dyalog.css` gives that bar the header colour at the drawer breakpoint.

Status colours are defined as tokens but not wired into admonitions; Zensical's
built-in admonition styling already works in both schemes. Note from the brand
guidelines that the base Error red passes on white but fails on Raisin, so a
dark-mode admonition would need the tint rather than the base.

## Licences

Fonts keep their original licences — see the `LICENCE` file in each font
directory. `LICENCE` at the root covers the rest.

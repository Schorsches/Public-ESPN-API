# NFL Team Logo Sprite

All 32 NFL team logos as a retina-ready CSS spritesheet, keyed by **ESPN team abbreviation**.

| File | Size | What it is |
|---|---|---|
| `nfl-sprite.png` | 147 KB | 1x sheet, 512 × 256 |
| `nfl-sprite@2x.png` | 386 KB | 2x sheet, 1024 × 512 |
| `nfl-sprite.css` | 2.9 KB | Base class + one modifier per team |
| `nfl-sprite.json` | 3.5 KB | Manifest: grid metadata and per-team offsets |
| `preview.html` | — | Renders all 32 on light and dark to eyeball the alpha |

## Use

```html
<link rel="stylesheet" href="nfl-sprite.css">
<i class="nfl-logo nfl-logo--kc" role="img" aria-label="Kansas City Chiefs"></i>
```

Class suffix is the lowercased ESPN abbreviation, so `nfl-logo--wsh` and `nfl-logo--lar`
(**not** `was` or `la`). The keys match `nfl_teams.csv` and `nfl_reference.json`, so a team
row joins straight to its logo class with no lookup table.

`background-size` is pinned to the **1x** dimensions and the retina sheet is selected via
`image-set()`, so the 2x sheet scales correctly and browsers without `image-set()` fall back
to the plain `background-image` declared just above it.

To change the display size, scale `background-size` and the element box by the same factor —
e.g. for 32 px tiles use `width/height: 32px` and `background-size: 256px 128px`.

## Layout

8 columns × 4 rows, ordered alphabetically by abbreviation, so positions are stable across
rebuilds. Row 1 is ARI–DAL, row 4 ends at WSH.

## How it was built

Sources are `https://a.espncdn.com/i/teamlogos/nfl/500/{abbr}.png` — 8-bit RGBA PNGs, no SVG
available. Two things shape the build:

**The Jets are served at 4096×4096**, not 500×500, so every canvas is normalised to 500×500
before scaling.

**Logos are not trimmed.** 42–73% of each source is transparent padding, but ESPN already
normalises every logo to the same canvas-relative width (~92% for 28 of 32) while heights vary
from 0.29 to 0.93. Content aspect ratios run 0.81 to 3.18. Trimming to the alpha box and
fitting each logo to its tile would render the Jets wordmark at a third the height of the
Steelers roundel. Keeping the square canvas preserves ESPN's own cross-team sizing.

**Cells are pasted without a mask.** `paste(img, pos, img)` double-applies alpha and corrupts
anti-aliased edges — measured worst per-channel error 254 versus 0 for a plain paste.

## Verified

- Sheets are exactly 512 × 256 and 1024 × 512, both RGBA with transparent backgrounds
- **All 32 cells on both sheets are byte-exact** against independently resized sources —
  worst per-channel difference 0
- No empty cells
- 32/32 CSS classes present, every offset agreeing with the manifest

These are trademarked club logos, kept local rather than hotlinked, for private
non-commercial use.

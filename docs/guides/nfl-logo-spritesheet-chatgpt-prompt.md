# NFL Logo Spritesheet — ChatGPT Prompt

> Paste everything below the line into ChatGPT.

---

## TASK

Build me a retina-ready CSS spritesheet of all 32 NFL team logos, sourced from ESPN's public
CDN, for use in a web app.

**Environment note:** your code sandbox probably has no internet access. So unless you can
verify you can reach `a.espncdn.com`, do **not** try to download inside the sandbox. Instead
give me one complete, runnable Python script I execute locally, plus the CSS. If you *can*
reach the network, run it and hand me the finished files as well.

## SOURCE

Logos live at a predictable URL keyed by the team abbreviation, lowercased:

```
https://a.espncdn.com/i/teamlogos/nfl/500/{abbr}.png
```

The 32 ESPN abbreviations, which are the canonical keys — use these exact strings as CSS class
suffixes and JSON keys:

```
ARI ATL BAL BUF CAR CHI CIN CLE DAL DEN DET GB HOU IND JAX KC LAC LAR LV MIA MIN NE NO NYG NYJ PHI PIT SEA SF TB TEN WSH
```

Two of these differ from the abbreviations used elsewhere in football data: ESPN uses **WSH**
(not WAS) for Washington and **LAR** (not LA) for the Rams. Do not "correct" them.

These are trademarked club logos. This is for a private, non-commercial app; keep the assets
local rather than hotlinking ESPN's CDN.

## VERIFIED FACTS — trust these over your own assumptions

I checked all 32 files directly. Do not re-derive or second-guess:

1. Every file is a genuine **PNG, 8-bit RGBA with a real alpha channel**. Confirmed by magic
   bytes and `Content-Type: image/png`. **Never flatten onto a white background** — transparency
   must survive into the sprite.
2. **There is no SVG.** Both plausible `.svg` paths return 404. PNG is the only format, so
   downscale from the 500 px source and never upscale.
3. **The Jets are an anomaly.** `nyj.png` is served at **4096×4096**, not 500×500, and is 128 KB
   against 7–40 KB for everyone else. Normalise every source canvas to 500×500 before compositing,
   or the Jets will blow up your memory use and land wrong in the grid.
4. **Do NOT trim to the alpha bounding box.** This is the one that will tempt you. Each logo sits
   letterboxed inside its square canvas, and 42–73% of each file is transparent padding — so
   trimming looks like an obvious win. It is not. ESPN has already normalised every logo to the
   **same canvas-relative width** (~92% for 28 of 32, standard deviation 0.04), while heights vary
   enormously (0.29 to 0.93 of the canvas). Content aspect ratios run from 0.81 (vertical) to 3.18
   (the Jets wordmark). Trimming and fitting each logo to the tile would make a wide wordmark
   render at a third of the height of a round logo, destroying the cross-team consistency ESPN
   already built in. **Keep the square canvas intact and scale it as a whole.**
5. **Paste without a mask.** In Pillow, `sheet.paste(img, pos, img)` — passing the RGBA image as
   its own mask — is the pattern most sprite tutorials use and it is **wrong here**. It applies
   the alpha a second time, corrupting every anti-aliased edge. I measured it: worst-case
   per-channel error of **254** across the 32 cells, versus **0** for a plain
   `sheet.paste(img, pos)` onto an already-transparent canvas. Use the plain paste, or
   `Image.alpha_composite`. This one is silent — the sheet still looks roughly right at a glance.
6. The `500-dark/` variants are frequently **byte-identical** to the default (verified for KC and
   PHI; the Jets genuinely differ), so a dark sheet is mostly wasted bytes. Skip it unless I ask.

## REQUIREMENTS

**Grid.** 32 logos as **8 columns × 4 rows**, ordered alphabetically by abbreviation so positions
are deterministic and stable across rebuilds.

**Sizing.** Default tile **64 CSS px**, so:

- `nfl-sprite.png` — 1x, 512 × 256
- `nfl-sprite@2x.png` — 2x, 1024 × 512

Make the tile size a variable at the top of the script so I can change it once.

**Image handling.**
- Resize with Lanczos resampling.
- Keep RGBA end to end; the sprite background must be fully transparent.
- Downscale only. The 500 px source comfortably covers a 128 px @2x tile.
- Composite each logo centred in its cell.

**Outputs**, all written to an `out/` directory:
1. `nfl-sprite.png` and `nfl-sprite@2x.png`
2. `nfl-sprite.css`
3. `nfl-sprite.json` — manifest mapping each abbreviation to its x/y offset, plus grid and tile
   metadata
4. The script itself

**CSS shape.** One base class plus a modifier per team. Use `image-set()` so the browser picks the
retina sheet, with a plain `background-image` fallback first. Set `background-size` to the **1x**
sheet dimensions so the 2x sheet scales correctly:

```css
.nfl-logo {
  display: inline-block;
  width: 64px; height: 64px;
  background-image: url("nfl-sprite.png");
  background-image: image-set(url("nfl-sprite.png") 1x, url("nfl-sprite@2x.png") 2x);
  background-size: 512px 256px;
  background-repeat: no-repeat;
}
.nfl-logo--ari { background-position: -0px -0px; }
/* …one per team… */
```

**Script quality.**
- Cache downloads locally and skip files already present, so re-runs are cheap.
- Sequential requests with a small delay and a descriptive User-Agent — this is an undocumented
  API being used as a guest.
- Fail loudly if any of the 32 is missing or is not a valid PNG. A silently incomplete sprite is
  worse than a failed build.
- Optionally run `oxipng` or `pngquant` if present, but never at the cost of the alpha channel.

## VERIFY BEFORE YOU HAND IT OVER

State the actual result of each, do not just assert success:

- 32 of 32 logos downloaded, each verified as PNG by magic bytes
- Output sheets are exactly 512×256 and 1024×512
- Both sheets are RGBA and the background is transparent, not white
- Every one of the 32 CSS classes exists and its offset matches the manifest
- **Byte-exactness check:** for every one of the 32 cells, crop it back out of the @2x sheet and
  diff it against the independently resized source. The worst per-channel difference must be
  **exactly 0**. Anything above 0 means the alpha was double-applied — see gotcha 5. Report the
  measured worst-case number, not a pass/fail.
- Report the final byte size of both sheets. For reference, my own run of this spec produced
  ~149 KB at 1x and ~389 KB at 2x with 64 px tiles.

Finally, give me a minimal HTML page that renders all 32 tiles in a grid so I can eyeball it.

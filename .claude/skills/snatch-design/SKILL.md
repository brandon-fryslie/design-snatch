---
name: snatch-design
description: Capture the design language of the currently-focused Chrome page and emit a reusable Claude skill that builds NEW sites in that aesthetic. Use when the user is browsing a design (via the design-snatch gallery index) and says "snatch this design", "/snatch-design", "capture this page", "grab this design", "make a skill from this design", or points at a site whose look they want to reproduce. Requires a Chrome instance connected over CDP with the target page open (see the design-snatch README for launching via `crom`).
---

# snatch-design

Capture the **design language** of the page currently open in Chrome and turn it into a
reusable skill under `skills/<slug>/`. Fidelity target is the *transferable aesthetic* —
palette, type, spacing, layout, motion — **not** a pixel clone of that one page.

The captured screenshot (`reference.png`) and the extracted `tokens.md` are the
authoritative record of the design. `[LAW:one-source-of-truth]` The generated `SKILL.md`
prose is *derived* from them, so the written description can never drift from the design
it claims to describe.

## Preconditions — fail loud, never guess

Before capturing, confirm a Chrome page is actually connected and focused:

1. Call `list_pages` (chrome-devtools MCP).
2. If it errors or returns no pages, **STOP** and tell the user Chrome isn't connected —
   they need to launch it first (see README: `crom` + the CDP port). Do **not** silently
   launch Chrome to a blank page or navigate anywhere on your own. `[LAW:no-silent-failure]`
3. Identify the target page (the selected/active one). Read back its URL and title so the
   user can confirm you're capturing what they're looking at.

## Procedure

### 1. Screenshot the page (the visual source of truth)

Slugify the target's hostname into `<slug>` (e.g. `stripe.com` → `stripe`; add a short
descriptor only if disambiguation is needed, e.g. `stripe-pricing`). Then:

- `take_screenshot` with `fullPage: true`, `format: "png"`,
  `filePath: "skills/<slug>/reference.png"`.

### 2. Extract design tokens (the measured source of truth)

Run this exact function with `evaluate_script`. It samples the rendered DOM and returns
JSON — the measured facts of the design, not a guess from the image:

```js
() => {
  const norm = v => (v || "").trim();
  const tally = () => new Map();
  const bump = (m, k, w = 1) => { if (k) m.set(k, (m.get(k) || 0) + w); };
  const top = (m, n = 12) => [...m.entries()].sort((a, b) => b[1] - a[1]).slice(0, n)
    .map(([k, c]) => ({ value: k, count: c }));

  const colors = tally(), bgs = tally(), families = tally(), sizes = tally(),
        weights = tally(), radii = tally(), shadows = tally(), spacing = tally(),
        letters = tally(), lineHeights = tally();

  const skip = new Set(["SCRIPT", "STYLE", "NOSCRIPT", "META", "LINK", "HEAD"]);
  const els = [...document.querySelectorAll("body *")].slice(0, 4000);
  for (const el of els) {
    if (skip.has(el.tagName)) continue;
    const r = el.getBoundingClientRect();
    if (r.width === 0 || r.height === 0) continue; // ignore invisible nodes
    const s = getComputedStyle(el);
    const area = Math.max(1, r.width * r.height);
    const wt = Math.log2(area + 2); // weight tokens by visual footprint
    bump(colors, norm(s.color), wt);
    if (s.backgroundColor && s.backgroundColor !== "rgba(0, 0, 0, 0)") bump(bgs, norm(s.backgroundColor), wt);
    bump(families, norm(s.fontFamily), wt);
    bump(sizes, norm(s.fontSize));
    bump(weights, norm(s.fontWeight));
    if (s.borderRadius && s.borderRadius !== "0px") bump(radii, norm(s.borderRadius));
    if (s.boxShadow && s.boxShadow !== "none") bump(shadows, norm(s.boxShadow));
    if (s.letterSpacing && s.letterSpacing !== "normal") bump(letters, norm(s.letterSpacing));
    bump(lineHeights, norm(s.lineHeight));
    for (const p of ["paddingTop", "paddingBottom", "marginTop", "marginBottom", "gap"]) {
      const v = s[p];
      if (v && v !== "0px" && v !== "normal") bump(spacing, v);
    }
  }

  // CSS custom properties declared on :root — the site's own token names, if any.
  const rootVars = {};
  try {
    for (const sheet of document.styleSheets) {
      let rules; try { rules = sheet.cssRules; } catch { continue; } // cross-origin
      for (const rule of rules || []) {
        if (rule.selectorText === ":root" && rule.style) {
          for (const name of rule.style) {
            if (name.startsWith("--")) rootVars[name] = rule.style.getPropertyValue(name).trim();
          }
        }
      }
    }
  } catch {}

  return {
    url: location.href,
    title: document.title,
    viewport: { w: innerWidth, h: innerHeight, dpr: devicePixelRatio },
    bodyBackground: norm(getComputedStyle(document.body).backgroundColor),
    colors: top(colors), backgrounds: top(bgs),
    fontFamilies: top(families), fontSizes: top(sizes, 16), fontWeights: top(weights),
    lineHeights: top(lineHeights), letterSpacing: top(letters),
    borderRadii: top(radii), boxShadows: top(shadows, 8), spacing: top(spacing, 16),
    cssCustomProperties: rootVars,
  };
}
```

### 3. Read the screenshot and synthesize

`Read` the saved `reference.png`. Combine what you *see* (layout structure, visual
hierarchy, density, imagery treatment, mood, any motion cues visible in the DOM) with the
*measured* tokens from step 2. The numbers tell you the palette and scale; the image tells
you how they're composed.

### 4. Emit the captured skill

Write three files under `skills/<slug>/`:

```
skills/<slug>/
  SKILL.md        how to build NEW sites in this style (durable artifact)
  reference.png   already saved in step 1
  tokens.md       the measured tokens from step 2, written up readably
```

**`tokens.md`** — the measured facts, organized: palette (with hex + role: bg/ink/accent),
type (families, the size scale you observed, weights, line-height, letter-spacing), spacing
rhythm, radii, shadows, and any `--custom-properties` the site exposed. This is derived
directly from step 2's JSON; do not invent values it didn't report.

**`SKILL.md`** — use this template:

```markdown
---
name: <slug>-style
description: Build web pages/components in the "<short aesthetic name>" design language
  captured from <hostname>. Use when the user wants a site/landing page/section that looks
  like <hostname> or asks for the "<slug>" / "<aesthetic name>" style. <one distinctive
  visual phrase, e.g. "high-contrast editorial with oversized serif display and dotted texture">.
---

# <aesthetic name> (captured from <hostname>)

Reference: `reference.png` · measured tokens: `tokens.md`.

## The thesis
<2-3 sentences: what makes this design read the way it does — the one idea a builder must
get right, and why. This is the "why", derived from the reference, not a caption of it.>

## Palette
<roles + hex from tokens.md: background, ink, muted, accent(s), borders>

## Type
<families and the intended pairing (display vs body), the size scale, weights, the
line-height and letter-spacing character>

## Layout & spacing
<grid/container width, section rhythm, density, alignment habits, radii, shadow language>

## Motion & interaction
<hover/scroll/transition character if observable; omit if the design is static>

## Build guidance
- Do: <the moves that make it feel like the original>
- Avoid: <the moves that break the aesthetic — be specific>
```

### 5. Report

Tell the user the slug, the three files written, and a one-line summary of the captured
aesthetic. Offer to build a sample page with the new skill so they can verify it composes.

## What this skill does NOT do

- It does not launch or navigate Chrome — the user drives the browser to the design first.
- It does not clone the page's copy or scrape its assets — it captures the *design language*.
- It does not write anywhere but `skills/<slug>/`.

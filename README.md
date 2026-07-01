# design-snatch

Browse curated galleries of high-craft web design, capture a design you like, and
turn it into a reusable Claude **skill** that builds new sites in that design language.

## Workflow

1. Open a Claude Code session in this directory.
2. Ask Claude to **launch Chrome** — it starts Chrome via `crom` (CDP port open) and
   connects through the `chrome-devtools` MCP, opening `index.html`.
3. Browse the galleries, navigate to a design you like.
4. Run **`/snatch-design`**. Claude screenshots the current page, measures its design
   tokens (computed styles), and analyzes it.
5. It emits a skill under `skills/<slug>/` reproducing the design *language*.

## The two skill trees

- `.claude/skills/snatch-design/` — **the tool.** The `/snatch-design` capture action.
- `skills/<slug>/` — **the products.** One folder per captured design.

Tool and product are separate parts with separate lifecycles, so they never blur together.

## The gallery index

`index.html` is a static, local-only launch page. The site list is a single `SITES`
array in its source — the one source of truth. Add a gallery by appending one entry;
the page renders itself from the data.

## What a captured skill contains

Fidelity target: **design language, not a pixel clone.** The skill teaches how to build
*new* sites in the captured aesthetic. Each capture produces:

```
skills/<slug>/
  SKILL.md        how to build in this style (the durable artifact)
  reference.png   the captured screenshot (the source of truth for the tokens)
  tokens.md       palette, type scale, spacing rhythm, layout grid, motion
```

`reference.png` + `tokens.md` are the authoritative record; `SKILL.md`'s prose is
derived from them, so the description can't drift from the design it describes.

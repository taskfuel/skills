---
name: build-brand-system
description: Create a brand identity from a product brief — pick a visual direction, write the spec (palette, type, logo, voice, usage rules), and generate one 3x3 board that shows the system in use, paid per-call through taskfuel.ai. Use when the user wants a brand, visual identity, style guide, brand system, design language, or "what should this look like" for a product, app, or company. Not for applying an already-approved brand to one asset.
license: MIT
---

# Build Brand System

Turn a product brief into a brand a developer can ship from. Two deliverables,
nothing else:

- **`brand/BRAND.md`** — the spec. Exact hex values, real licensed fonts, logo
  construction, voice, usage rules. This is the source of truth.
- **`brand/brand-board.png`** — one 3x3 image showing the system applied. This
  is a *showcase*, not a spec: it proves the direction reads, and it is what the
  user looks at to say yes.

Write the spec first, then generate the board from it. Never read colours or
type off a generated image — the image inherits its values from `BRAND.md`, not
the other way round.

## Prerequisites

- `taskfuel` CLI connected (`taskfuel whoami`), balance ≥ ~$0.20 (each board is
  ~$0.05; budget 2–3 attempts).
- `blockrun.ai` in the gateway's allowed domains (`taskfuel discover`).
- Base `taskfuel` skill covers payment mechanics — read it first if you haven't.

If the environment has its own image generation, use that instead and skip the
paid call. Without either, still produce `BRAND.md` and hand over the board
prompt — say plainly that no image was generated.

## Workflow

### 1. Brief

Infer everything you can from the conversation, an attached moodboard, or the
repo *if the user pointed at one* — don't treat an unrelated cwd as evidence.
Then ask at most three questions, and only ones that change the output:

- what the product is and who it's for
- anything that must survive (existing name, mark, colour, tone)
- where it has to work (web app, CLI, print, packaging, dark mode)

Don't ask for a preferred aesthetic. Proposing is the job. If you have enough,
state your assumptions and start.

### 2. Kernel — hold this in your head, not in a file

Before any visual choice, pin down four things and say them back in two or
three sentences:

- **Job**: who needs this and what progress they're making.
- **Promise**: what the brand should make credible in one second.
- **Category stance**: which conventions to embrace, which to resist.
- **Three tensions**, each as *desired → avoid*: "calm authority → institutional
  coldness", "technical precision → developer-only signalling". Adjective piles
  ("modern, clean, bold") are not a kernel — they can't reject anything later.

If you can't use the kernel to reject an attractive-but-wrong direction, it
isn't specific enough yet.

### 3. Three directions → user picks one

Propose **three** directions in text. Cheap, fast, no images yet. Each gets four
lines:

- **Idea** — one strategic argument derived from the kernel.
- **System** — type logic, geometry, colour logic, imagery treatment.
- **Feels like** — the one-second read.
- **Risk** — where it fails. A direction with no risk is vague.

They must differ in *at least four* of: metaphor, category relationship,
emotional temperature, type logic, geometry, colour logic, imagery, materiality.
Three palettes of the same layout are one direction, not three. Never name real
brands or living designers as shorthand — describe observable behaviour.

**Stop here and get an explicit pick.** Don't merge directions, don't generate a
board, don't touch frontend code. If the user asks to see them before choosing,
board the top two (~$0.05 each) and present side by side. Separate image quality
from system quality — a well-rendered weak idea should not win.

### 4. Write `brand/BRAND.md`

Every value is authored by you and exact. Sections:

- **Brand idea** — 2–3 sentences: the kernel, resolved.
- **Palette** — named roles with hex: ink, surface, surface-alt, primary,
  accent, plus semantic success/warning/danger if the product needs them. Give
  dark-mode counterparts. Verify body text hits **4.5:1** against its surface
  and UI/large text **3:1** — compute the ratios and state them; fix the value,
  don't hedge. Say what each colour is *for*, and name the one that must stay
  rare.
- **Typography** — real, licensable families (default to Google Fonts unless the
  user wants to buy type), one display and one text face, plus mono if it's a
  dev product. Record weights, the type scale, tracking on display sizes, and
  the fallback stack.
- **Logo** — construction in words: mark idea, geometry, clear space, min size,
  lockups, what breaks it. The board's mark is a sketch; the real one gets
  redrawn in vector.
- **Graphic language** — grid, radii, borders, elevation, iconography style,
  motion character. Concrete values, not moods.
- **Imagery** — photography/illustration/texture treatment, and what's banned.
- **Voice** — three rules with a good and bad example each.
- **Do / don't** — the five rules most likely to be broken.
- **Open questions** — what's unresolved. Say it rather than inventing it.

Keep it one file. If the project later needs machine-readable tokens, generate
them from this file — don't fork the truth.

### 5. Generate the board

One image, 3x3, 9 panels. Fixed roles so the board covers the whole system:

| | | |
|---|---|---|
| 1 logo lockup on primary field | 2 palette in proportion | 3 type specimen: display + text |
| 4 core product surface | 5 icon + control set | 6 secondary surface (mobile/dark) |
| 7 imagery treatment | 8 graphic pattern / motif | 9 real-world application |

Swap panel 9 (and only 9) for whatever fits — packaging, signage, merch, a
social card. Build the prompt straight out of `BRAND.md`:

```
One single image: a 3x3 grid brand board, 9 equal panels, thin neutral
gutters, no captions or numbers anywhere.
PALETTE: use exactly #RRGGBB (ink), #RRGGBB (surface), #RRGGBB (primary),
#RRGGBB (accent, sparingly) in every panel.
TYPE: <display family>, <text family>, <weights>.
SYSTEM: <geometry, radii, iconography, imagery treatment from BRAND.md>.
PANELS: 1 <role>. 2 <role>. ... 9 <role>.
AVOID: <the avoid list — clichés, effects, motifs the direction rejects>.
```

**Pick the model deliberately** — don't inherit a default. List what's on offer
and read the prices:

```sh
curl -s https://blockrun.ai/api/v1/models | jq -r '.data[]
  | select(.categories[]? == "image")
  | "\(.id)\t$\(.pricing.per_image)\t\(.description)"'
```

Default to **`openai/gpt-image-2`** ($0.06) for brand boards. A board is nine
panels of *typography* — a wordmark, a type specimen, button labels — and image
models differ enormously here. Verified 2026-07-30 on the same prompt:
`google/nano-banana` ($0.05) rendered garbled pseudo-text in every text panel
across two attempts, while gpt-image-2 produced a correctly-spelled wordmark, a
legible a–z specimen, and accurate button labels first try. One cent buys the
entire difference between a usable board and an unusable one.

Build the body with `jq -Rs` so the prompt's quotes and newlines survive, then
quote before paying. `--body` takes **inline JSON only** — there is no `@file`
support, so pass `"$(cat …)"`:

```sh
jq -Rs '{model: "openai/gpt-image-2", prompt: .}' prompt.txt > board.json
taskfuel call https://blockrun.ai/api/v1/images/generations \
  --method POST --body "$(cat board.json)" --quote      # confirm ~$0.064
taskfuel call https://blockrun.ai/api/v1/images/generations \
  --method POST --body "$(cat board.json)" > resp.json
```

**Two response shapes.** Fast models return the image inline as `data[0].url`.
Slower ones — gpt-image-2 included — return `status: "queued"` with a `poll_url`;
`taskfuel call "https://blockrun.ai<poll_url>"` every ~20s until
`status: "completed"`, which takes about a minute. Payment settles on completion,
so a queued response already means you've been charged.

Either way the URL is an ordinary hosted PNG (not a data URI) — `curl -sSL` it to
`brand/brand-board.png`. The response embeds an unquoted `tx_hash`, so `jq` can
fail to parse it; `grep -o 'https://blockrun.ai/api/media[^"]*\.png'` is the
reliable extraction. Save the final prompt in a fenced block at the bottom of
`BRAND.md` so the board can be regenerated.

Keep rendered text minimal. Image models mangle small copy — treat any type in
the board as *indicative of attitude*, never as a specimen to measure.

### 6. Check, revise once at a time, deliver

Read the generated image back and grade it against `BRAND.md`:

- Are the palette colours the ones you specified, in roughly the intended
  proportion?
- Is the type attitude right (weight, contrast, density)?
- Is the geometry consistent across all nine panels, or did three of them drift
  into a different system?
- Does it survive the kernel's tensions, or did it land on the "avoid" side?

Fix **one** identified failure per regeneration, naming it in the prompt, and
state the correction *and* its opposite — "a continuous chart LINE, not a solid
filled mound", reinforced by adding the wrong reading to the AVOID list. Naming
only the thing you want reliably gets you the thing you don't. If drift keeps
coming back, the fix is in `BRAND.md` — sharpen the spec, don't pile on
adjectives. Two or three attempts is normal; more means the direction is
underspecified.

Observed failure modes worth pre-empting in the first prompt: the model fills
any "hill/curve/area" language into a solid opaque shape; it picks label colours
on filled buttons at random, so **state the label colour per button**; and a
"sticker sheet" panel leaks its white outline into every other panel unless you
scope it ("the ONLY panel where creatures have outlines"). All three are fixed by
naming the constraint *and* its opposite — a per-panel scope reliably holds.

**Know when to stop.** Some spec items no image model will hit — per-letter
baseline rotation in a wordmark failed three times across two models here. That's
not a prompt problem, and a fourth attempt is just money. Logo geometry is on the
"rebuild in real tools" list anyway, so note the gap and move on. Stop when the
only remaining deltas are things `BRAND.md` already declares the board
non-authoritative for.

Then show the user the board and the spec, and say in one line what's directional
(the mark, any rendered type) versus what's exact (hex, families, values).

## Rules that keep this honest

- **The board never overrides `BRAND.md`.** If they disagree, the file is right
  and the board gets regenerated.
- **Rebuild in real tools before shipping**: logos as vector, type as licensed
  webfonts, UI in code. Generated raster is final only for texture, imagery, and
  campaign art.
- **Don't build frontend to compare directions.** That's what the board is for.
- **Don't rerun this skill for routine assets.** Read the existing `BRAND.md`
  and apply it. Rerun only for a rebrand, or when the system fails a new
  requirement.
- Preserve anything the user said must survive, even when a direction would be
  stronger without it — surface the tension, let them decide.

## Costs (observed 2026-07, quote to confirm)

| Step | Price |
|---|---|
| Brand board (`blockrun.ai`, nano-banana t2i) | ~$0.05/board |
| Extra directions boarded for comparison | ~$0.05 each |
| Revision pass | ~$0.05 each |

A normal run lands around $0.10–0.20. Kernel, directions, and `BRAND.md` cost
nothing — only images are paid.

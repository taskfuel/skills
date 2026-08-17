---
name: storyboard-to-video
description: Turn a single N-frame storyboard image (e.g. a 3x3 grid) into a multi-shot AI video with seamless cuts, paid per-call through taskfuel.ai/x402. Use when the user has a storyboard/grid image and wants it animated, wants a storyboard generated from scratch and then animated, mentions "storyboard to video", or wants multi-shot AI video with character consistency.
license: MIT
---

# Storyboard → Video

Turn one storyboard image containing N frames into a multi-shot film. Anchor
frames from the storyboard bound each shot (first/last-frame video
generation), and adjacent shots share a boundary frame so cuts line up by
construction. Every paid step goes through the `taskfuel` CLI (quote first,
then pay).

**Why a single storyboard image:** one image-model generation gives you a
consistent character, consistent world, and a deliberate color script across
all panels — sidestepping the hardest problems in multi-shot AI video.

## Prerequisites

- `taskfuel` CLI connected (`taskfuel whoami`), balance ≥ the chosen
  model's per-shot price plus a margin for re-rolls.
- `stableupload.dev` and `blockrun.ai` must be in the gateway's allowed
  domains (`taskfuel discover` lists them). If stableupload is missing, the
  domains list needs a deploy — stop and tell the user.
- `ffmpeg` locally.

## Workflow

### 1. Storyboard gate — validate it, or generate one

The whole pipeline stands on the storyboard: a board that fails the checks
below cannot produce seamless cuts no matter how good the shot prompts are,
and each bad clip costs ~$0.49 vs ~$0.05 for a new board. So this step is a
**hard gate before any video spending**.

**No storyboard yet?** Offer to generate one: sketch the shot chain with the
user (which panels will be cut points), write a prompt with the template
below, then generate and run the same gate on the result:

```sh
# body: {"model": "google/nano-banana", "prompt": "<from template>"}
taskfuel call https://blockrun.ai/api/v1/images/generations --method POST --body @...  # ~$0.05/board
```

**Have a storyboard?** Read the image and grade it — panel by panel, and
anchor-pair by anchor-pair — against this checklist. All must pass:

- Regular grid with straight gutters **and equal-size panel cells** —
  verify by measuring the actual gutter positions (step 3's detection
  procedure), never by assuming W/C × H/R. Panel cells differing by more
  than a couple of pixels fail the gate: unequal anchors produce different
  output aspect ratios per shot (breaking lossless concat), and the
  measured-vs-assumed mismatch leaks gutter slivers into the anchors.
- Same character, world, and rendering style in every panel (face, costume,
  palette).
- **Static set elements hold still in scene space.** Camera moves between
  panels are fine — judge the world, not the frame. For each planned anchor
  pair, reconstruct the scene layout from each panel (what sits next to
  what, relative sizes, which side of the room) and check the two
  reconstructions describe the same world. A crate seen from a new angle is
  a camera move; a crate against a different wall, landmarks that swapped
  relative positions, or set pieces that appeared/vanished are the *set*
  moving — the video model will animate that as furniture sliding/warping
  mid-clip. Only the subject and objects the beats say it moves may change
  place in the world. When a pair implies a camera move, the shot prompt
  must state that move explicitly (it happens mid-shot; the camera is still
  at rest at each boundary, per step 5).
- **Every anchor pair must be bridgeable by on-screen action — and that
  includes the state of the environment, not just objects.** Compare the
  whole environment between the two anchors. Any change that is not
  specifically part of the story's on-screen action is a defect: the model
  is forced to conjure the difference from nothing, which leads to
  inconsistencies, unwanted cuts, and unrealistic output regardless of
  prompt wording (verified across three paid takes on one such pair — a
  pristine floor in anchor A vs an excavated trapdoor pit in anchor B, far
  more change than the scripted single hand-brush could produce). Fix by
  re-planning the anchors (option 2 below), or with an i2i edit that makes
  the earlier panel foreshadow the later state (e.g. a faint dusty outline
  of the trapdoor). Scope: this rule is about what the anchors *visibly*
  assert — content outside a panel's framing is unpinned, not
  contradicted. A prop or set piece the story needs may live off-frame
  (the unseen end of a desk, behind the camera) and be revealed mid-shot
  by a camera move or narrated action; that requires no board edit. Judge
  "does the anchor contradict it?", never "does the anchor show it?".
- **Enough rest-point panels to chain the shots.** Every panel usable as a
  shot boundary must show the subject at rest — "cut on stillness" (see the
  shot-plan rules in step 2). A board whose panels are all mid-action has no
  valid cut points.
- ≤2 story beats per 5 seconds of each shot's planned duration (a wider
  anchor span is fine if its shot gets a proportionally longer duration).
- No ambiguous poses/props on intended boundary panels (a half-open door
  reads as opening *or* closing).
- Ideally no baked-in text at all. Small corner numbers are tolerable but
  force a per-panel i2i cleanup (step 3) that risks panel-to-panel drift;
  captions or speech bubbles across panels fail the gate.

Report the verdict to the user panel-by-panel, naming which panels can serve
as boundaries. On failure, present the user with the recovery options below —
with the context (which panels/pairs failed, and why) and a recommendation
for which option fits this board — and let the user choose; never pick one
unilaterally:

1. **Regenerate the storyboard** (~$0.05): rewrite their prompt with the
   template below, explicitly fixing the named defects, and re-gate. Iterate
   here freely — several board attempts still cost less than one bad clip.
2. **Re-plan around the passing panels**: fewer or different shots that only
   use panels that pass. Skipping a mid-chain panel as an anchor is often
   the cleanest fix for an unbridgeable pair: anchors only pin the
   endpoints, so a wider span (P1→P4 instead of P1→P2→P4) leaves the
   in-between unpinned and lets the model render the set change
   continuously as part of the action. Budget that shot's duration for the
   extra beats it now carries (step 2) — the skipped panel still earns its
   keep as world/style reference in the board. Skipping panels changes the
   shot plan: **confirm the new chain with the user before proceeding.**
3. **Proceed with mitigation** (only if the user insists on a mid-action
   boundary): the boundary panel itself must encode motion (blur, dust
   trails, limb drag), and warn that the cut may read as visible.

#### Storyboard prompt template

Whether generating from scratch or helping the user improve their own
prompt, build it from this template. Decide the shot chain *first*, so the
rest poses land exactly where the cuts will be:

```
[GRID]      "One single image: a CxR storyboard grid of N panels, thin white
             gutters, no text or numbers anywhere." (Panel identity comes
             from grid position — numbers would only force a cleanup step.)
[CHARACTER] one fixed description sentence, repeated verbatim everywhere the
             subject appears.
[WORLD/STYLE] setting, lighting, rendering style, one color script across
             all panels.
[PANELS]    numbered beats, one line each. Write every intended boundary
             panel as an explicit rest pose: "panel 5: she stands still at
             the cliff edge, feet planted, arms at her sides."
[RULES]     "Panels <boundary list> show the subject completely at rest, no
             motion blur. Same character, outfit and style in every panel.
             The world layout is consistent: environment elements keep the
             same positions relative to each other in every panel that
             shares a location — the camera angle may change between
             panels, but the set itself never rearranges; only the subject
             and objects it moves change place. No captions, text, or
             numbers anywhere."
```

### 2. Shot plan and model choice

With a board that passed the gate, agree a shot plan and a video model with
the user before spending:

- Each shot = a pair of anchor panels (first frame, last frame).
- **Adjacent shots must share their boundary panel** (e.g. panels 1→5, 5→8,
  8→9) — that shared frame is what makes the cut seamless.
- **Cut on stillness.** Boundaries belong at rest points (standing, landed,
  apex of a jump) — never mid-fall or mid-swing. First/last anchoring
  guarantees *position* continuity at a cut but not *velocity*: two clips
  will invent different speeds/directions for the same mid-action pose and
  the cut will look fake. If the story forces a mid-action boundary, the
  panel itself must encode motion (blur, dust trails, limb drag).
- **≤2 story beats per 5s shot.** More gets compressed into nonsense.
- **Watch for ambiguous anchor poses** (a half-open door reads as opening
  *or* closing). Prefer an unambiguous panel — fix the board (regenerate or
  i2i) rather than trying to disambiguate in the prompt; at most add one
  clarifying clause to the shot's narration.

#### Pick the video model (ask the user)

Don't assume a model — list what blockrun offers *right now* and let the
user trade price against quality (cheapest, best, or a specific one). The
catalog endpoint is free and unauthenticated:

```sh
curl -s https://blockrun.ai/api/v1/models | jq -r '
  .data[] | select(.categories | index("video"))
  | [.id, "$\(.pricing.per_second)/s", "max \(.pricing.max_duration_seconds)s",
     .description] | @tsv'
```

Present the video models with an estimated per-shot and whole-film price
(`per_second × shot length × shot count`), then confirm the pick fits the
workflow before committing:

- **First/last-frame support is required** (`image_url` + `last_frame_url`).
  Not every listed video model supports it — verify with a `--quote` call on
  the intended body; if it's rejected, report back and re-pick.
- The model's default/max duration must fit the planned shot length.
- `per_second × duration` must fit the gateway's per-call cap — the quote
  response's `per_call_cap_usd` field states the current cap (observed
  $10, 2026-07-25); quotes above it are rejected.

The same catalog lists image models (`categories` contains `"image"`,
`pricing.per_image`) if the user wants a say in the storyboard/cleanup
model too; `google/nano-banana` is the tested default there.

### 3. Crop the anchor panels (clean only if annotated)

Only the anchor panels are needed, not all N.

**Never crop at assumed W/C × H/R fractions.** Boards have gutters (thin
near-white lines) between panels, real grids are rarely pixel-uniform, and
fraction-boundary crops leak gutter slivers — or worse, slivers of the
neighboring panel — into the anchors (observed: white lines baked into
anchors and therefore into every clip generated from them; the actual
gutters sat 3–11px away from the assumed boundaries). Measure, then crop:

1. **Look at the board first.** There is no standard for gutters — they may
   be white, black, colored, or absent entirely (panels touching edge to
   edge). View the image and note what actually separates the panels
   before writing any detection code.
2. **Detect the measured boundaries from the pixels**, using what you saw:
   gutter bands are full-width rows / full-height columns that are
   near-uniform (low variance) and match the observed gutter color —
   compute per-row/per-column statistics (ffmpeg rawvideo + a few lines of
   python) and find those bands. If there are no gutters, find the
   boundaries by content discontinuity between adjacent panels instead.
3. **Panels are the spans between the measured boundaries** (and the image
   edges). Crop each needed panel to its measured span, inset a couple of
   pixels so no boundary pixel survives.
4. **Verify or reject**: all panel spans must be equal within a couple of
   pixels — otherwise the board fails the step-1 grid check (report it,
   don't silently crop unequal anchors: unequal anchors → unequal output
   aspect per shot → concat needs re-encode). After cropping, check each
   panel's outermost rows/columns against the observed gutter color — a
   matching edge band means a leaked gutter; fix the bounds and re-crop.
5. **View every cropped panel before hosting.** A defect in an anchor is
   inherited by every clip that uses it.

**Annotation-free panels (all generated boards) need no cleanup — use the
crop as-is** and go straight to hosting. There is no upscale step either:
video models output at their native resolution regardless of anchor size.

**Never edit an existing image without the user's explicit prior
approval of the specific change — regardless of price.** Boards and panels
are the user's creative material: an edit is a creative decision the user
owns, not a cost decision, so the ~$0.05 price never makes it
pre-approved. Before proposing an edit at all, re-check whether it is
actually needed (e.g. the off-frame scope rule in step 1); then describe
exactly what would change and wait for the OK.

Only if a user-supplied panel has baked-in text/numbers, remove them with a
minimal i2i edit — a last resort, because each edit is an independent
generation and per-panel edits can drift the character/palette apart,
eroding the single-image consistency the workflow is built on. Keep the
edit surgical, and clean a shared boundary panel **once**, reusing the same
file in both shots. Compress to JPEG first — the CLI body limit is ~128KB
and the image goes in as a base64 data URI:

```sh
ffmpeg -i panel$n.png -q:v 4 panel$n.jpg   # ~15-50KB
# body: {"model": "google/nano-banana", "prompt": "Remove the small number
#   digit from the top-left corner. Fill seamlessly. Change nothing else.",
#   "image": "data:image/jpeg;base64,<base64 -w0 panel$n.jpg>"}
taskfuel call https://blockrun.ai/api/v1/images/image2image --method POST --body @... # ~$0.05/panel
```

The response's `data[0].url` is a base64 data URI — decode to `panel$n-clean.png`.
**Visually verify every cleaned panel before proceeding** — check both that
the annotation is gone and that nothing else changed; compare cleaned
panels side-by-side against the original board for character/style drift.

### 4. Host the panels publicly (stableupload.dev)

Video APIs take image *URLs*, not uploads. Buy a short-term slot per file
($0.005, 7-day retention), then POST the file to the returned presigned S3
form (`curlExample` in the response shows the exact command); the file is
then live at `publicUrl`:

```sh
taskfuel call https://stableupload.dev/api/upload --method POST \
  --body '{"filename": "panel1.png", "contentType": "image/png", "tier": "short-10mb"}'
# → { uploadUrl, postFields, publicUrl, curlExample, ... } — run the curlExample with your file
```

Verify each `publicUrl` returns HTTP 200 before generating.

### 5. Write the shot prompts

Plan each shot with a boundary contract, but understand what it is: a
**planning artifact between humans, defining the interface between adjacent
shots — it does NOT go into the generation payload.**

```
[BOUNDARY IN]  exact position + velocity + camera at frame 1 —
               verbatim copy of the previous shot's BOUNDARY OUT.
[BEATS]        1-2 numbered actions, in order.
[BOUNDARY OUT] exact position + velocity + camera at the last frame.
```

Rules: duplicate the boundary-contract sentence **verbatim** in both adjacent
shots' plans; pin the camera static at every boundary (camera moves happen
mid-shot); cut on stillness.

**The generation prompt is a few short sentences of scene narration — and
nothing else.** The anchor frames already carry the character, the set, the
palette, and the framing; the prompt's only job is to describe what happens
*between* them. Hard rules, each learned from paid failures:

- **Never re-describe anything visible in the anchor images.** No character
  appearance, no room inventory, no color script, no restating the start or
  end pose. Every re-described detail is redundant at best and a second,
  conflicting authority at worst. The same goes for asserting environment
  details the anchors contradict: prompting "sweeps the dust away" against
  a visibly clean floor doesn't get ignored — the model goes hunting for
  somewhere in the frame the detail *could* be true and rewrites the story
  around it (observed: a "hidden under dust" beat redirected onto a crate,
  turning a floor-trapdoor shot into a wardrobe-door shot).
- **Short and to the point.** Subject's actions in order, the camera's
  physical motion, one brief style/mood line. A good shot prompt is 3–5
  sentences (~50–80 words). If it has paragraphs, it's wrong.
- **No editorial/frame vocabulary.** "At the first frame", "the clip ends
  on this frame", "take", "cut" — the anchors already define the endpoints,
  so these sentences are redundant scaffolding. Talk about the scene, not
  the video.
- **Camera motion as slow, smooth, physical movement**: "the camera dollies
  in slowly and smoothly the whole time" — a described *move*, not a
  described *framing change* ("pushes in to a medium-close" reads as a
  target state and gets resolved as a cut).
- **Tie object appearances to causes.** If something is visible in the last
  anchor but absent in the first (dust cleared, door opened, hole dug),
  narrate the action that produces it ("sweeps the dust away, gradually
  uncovering the ring") — an object with a reason to appear can be rendered
  continuously; one that must simply materialize gets popped in.

Example of a complete shot prompt (a 5-beat, wide-span shot — budget ~10s):

```
The robot sits slumped and alone under the hanging lamp. It notices
something under the dust of the floor, crawls over, sweeps the dust away,
and uncovers a hidden trapdoor with a metal pull-ring. It heaves the
trapdoor open, warm light pours up from below, and it jumps in feet-first,
falling through open space as the gray world gives way to warm color, then
lands softly on its knees in the sand of a vast sunlit sandbox. The camera
follows it the whole way. Handmade miniature diorama, Pixar-style 3D
animation, loneliness turning to joy.
```

**Known limit — prompts cannot bridge an invalid anchor pair.** When the
anchors demand a change the on-screen action can't plausibly produce (the
step-1 bridgeability check), the model pays the gap as a hidden cut or
object pop-in regardless of prompt wording — verified: verbose, guarded,
and minimal prompts all cut on the same shot, and the pair was the cause.
If a clip cuts despite good prompts, suspect the anchor pair first and
re-check it against step 1; then check the beat/duration budget (step 2's
≤2 beats per 5s). Prompt-wording tweaks are the weakest lever of the
three.

**Write one prompt file per shot and have the user review before spending.**
Put each shot in its own `shotN-prompt.md` containing the exact request body
(model, `image_url`/`last_frame_url` with the real hosted URLs, duration)
and the exact prompt string — the literal payload, nothing left to
substitute at send time — and hand the files to the user for review before
any generation call. Keep the boundary contracts in the planning notes,
byte-identical between adjacent shots (verify mechanically with grep +
string-compare, not by eye).

### 6. Generate the shots

Use the model chosen in step 2, first/last-frame mode. Always quote before
paying. The recipe below shows the tested default
(`bytedance/seedance-1.5-pro`, 5s ≈ $0.49); other models may expect
slightly different body params — check the quote/error response before
assuming the recipe transfers.

```sh
# body: {"model": "bytedance/seedance-1.5-pro", "input_type": "first_last_frame",
#        "image_url": "<first publicUrl>", "last_frame_url": "<last publicUrl>",
#        "duration_seconds": 5, "prompt": "<step-5 scene narration>"}
taskfuel call https://blockrun.ai/api/v1/videos/generations --method POST --body @... --quote
taskfuel call https://blockrun.ai/api/v1/videos/generations --method POST --body @...   # submit
```

The submit returns `{id, poll_url}`. Generation takes ~60–180s (premium
models can take longer): **wait ~170s, then poll** via
`taskfuel call "https://blockrun.ai<poll_url>"`, re-polling at ~60s
intervals — don't hammer. Billing (verified 2026-07-25): the submit and
in-progress polls debit $0.00; exactly one settlement at the quoted price
happens on the first poll that finds the job completed, and an unclaimed
job stays claimable ~48h if your client times out. Report actual debits to
the user. The completed poll returns the video URL; download it.

**Quality-gate each clip before generating the next**: extract 2–3 frames
(`ffmpeg -vf "select='eq(n\,60)'"`) and check the action actually happened
(models take shortcuts on hard or ambiguous beats) and that static set
elements didn't
slide, warp, or teleport between the anchors. **Also run automated cut
detection — sampled frames routinely miss hidden cuts** (a clip can look
fine at 4 sampled timestamps and still contain a hard cut between them):

```sh
ffmpeg -i clip.mp4 -vf "select='gt(scene,0.1)',metadata=print" -f null - 2>&1 \
  | grep -oE 'pts_time:[0-9.]+|scene_score=[0-9.]+' | paste - -
```

Any hit is a suspected cut — extract frames just before/after the reported
timestamp and eyeball them (scores ≥~0.4 are near-certain hard cuts; ~0.1–0.3
catches softer angle jumps; expected fast action like an explosion can
false-positive). A clip with an invented cut fails the gate. Before
re-rolling, re-check the shot's anchor pair against step 1's bridgeability
rule — an invalid pair was the verified root cause of repeated cuts, and
no prompt fixes it; re-plan the anchors or extend the duration instead of
piling on anti-cut language (tried, doesn't work). A bad take costs a full
clip to redo; keep bad takes as v1 files, never delete or overwrite them.

**Re-rolls are new paid generations: confirm with the user first, and never
submit in parallel a fix you haven't proven.** When a prompt change is meant
to fix a failure shared by several clips, re-roll ONE clip, gate it, and
show the user the result before spending on the rest — if the fix doesn't
work, a parallel batch multiplies the waste by the batch size. This
overrides any urge to save wall-clock time.

### 7. Assemble

All Seedance clips come back matching (h264, same resolution, 24fps) —
concat losslessly; re-encode only if params differ:

```sh
printf "file '%s'\n" /abs/path/clip*.mp4 > list.txt   # absolute paths
ffmpeg -f concat -safe 0 -i list.txt -c copy film.mp4
```

To splice into an existing video (e.g. an ambient hero loop), extract its
first frame and generate a transition shot: last storyboard anchor → that
frame, prompt a camera pull-back/push-in. The final frame will land nearly
pixel-identical and the hard cut disappears.

## Costs (observed 2026-07, quote to confirm)

| Step | Price |
|---|---|
| Storyboard generation (nano-banana t2i) | ~$0.05/board — iterate until it passes the gate |
| Panel cleanup (nano-banana i2i, only if a user-supplied board is annotated) | ~$0.05/panel |
| Hosting (stableupload short-10mb, 7 days) | $0.005/file |
| 5s shot (tested default: Seedance 1.5 Pro first/last) | ~$0.49 quoted — settles once at the quoted price (submit and in-progress polls are $0). Other models: `per_second × duration` from the live catalog (step 2) |
| Crop/stitch (ffmpeg) | free |

A 3-shot film lands around $3. Spending rules apply: quote every endpoint
first, get explicit user OK before every paid generation — video shots and
image calls (board generations, i2i edits) alike, regardless of price —
report a running cost total, and stop-and-report on any failed paid call —
never blind-retry.

## Rejected alternative: whole-board reference generation (tested)

Seedance 2.0 `reference_image_urls` — **tested, not recommended for
multi-shot films.** Feeding all storyboard panels as references with one
continuous prompt (15s, 5 panels, $4.78) did produce the full story with a
consistent character, but the model invented its own hidden cuts and jumps
between beats: with no anchor frames pinning shot boundaries, you lose
exactly the continuity control the first/last-frame chain exists to
provide — and a bad result costs an entire film, not one clip.

**The per-shot first/last-frame chain with a human quality-gate between
clips remains the recommended workflow** — per-clip anchoring plus per-clip
review beats one big generation. Reference mode may still suit single-scene
shots wanting appearance-only guidance (≤2 beats, no cut points).

Practical facts (observed): `input_type: "reference"` +
`reference_image_urls` (max 9) is mutually exclusive with
`image_url`/`last_frame_url`; seedance-2.0 rejects `duration_seconds` < 4;
max 15s; a 15s reference job takes ~6–8 min, not the documented 60–180s;
output follows the reference aspect ratio.

## Upgrade paths (untested)

- Seedance 2.0 `reference_videos` (r2v) can condition on the previous clip's
  tail for true motion continuity across cuts — ~$0.32/s.

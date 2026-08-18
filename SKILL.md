---
name: analyzing-video-content
description: Use when analyzing a video (YouTube, live replay, reel, screencast, lecture) for its information content — extracting claims, methods, prompts, code, slides, dashboards, charts or demonstrations — especially when part of the content is only shown on screen and never spoken ("as you can see here", silent screencasts, slide decks, UI walkthroughs), or when a transcript-only analysis feels incomplete.
---

# Analyzing Video Content

## Overview

A video carries information in three layers: **free text around it** (description,
pinned comments, linked resources), **the audio** (transcript), and **the screen**
(what is shown but never said). Complete analysis = mine them in that order,
because each layer is ~10× cheaper than the next.

The goal is to give the analyst SIGHT. A human watches at 24 fps but *attends*
selectively; full-rate capture is unaffordable, so attention is the substitute —
**except when full capture IS affordable, in which case take it** (see modes).

Core principle: **narrow first, read last.** Every vision read costs ~1-1.5k tokens.
The whole craft is deciding WHICH few frames deserve eyes.

Works for any niche — trading screens, startup metric dashboards, medical lecture
slides, cooking steps, code tutorials, fitness form demos. See "Adapting to your
niche" at the end.

## Choosing the depth — need × cost (pick a mode, say which)

Before any layer, state three facts: the NEED (what completeness does the task
require — "capture everything" vs "find X" vs "worth a deeper look?"), the COST
side (duration × scene density, from the zero-token probe in Layer 3), and the
BUDGET if one is set. Then pick a mode and announce it:

| Mode | When (observable predicates) | What you do |
|---|---|---|
| **EXHAUSTIVE** — full sight | short video (≲20 min), OR need says "everything" and estimated cost fits: `ceil(scene_frames/8) × ~1.5k tokens` | Extract EVERY scene-change frame, sheet ALL of them, read every sheet, full-res dive on every info-bearing screen (budget ~1 dive per distinct screen seen on the sheets). Nothing sampled, nothing skipped. |
| **STANDARD** — selective attention | default | Sweep + triggers + targeted dives (the rest of this skill). |
| **MINIMAL** — recon | budget ≲3k tokens, or need = "should we look deeper?" | Layer 1 + reduced transcript + probe verdict only; report what a deeper pass would cover. |

Exhaustive is not a failure of economy: a 10-min screencast with 40 scene changes
costs ~5 sheet reads to see *in full* — cheaper than missing the one overlay that
mattered. The economy machinery below is for when full capture does NOT fit.

## Layer 1 — Free text (always first, ~0 cost)

```bash
yt-dlp --print description URL          # creators often link the very thing they show
```
Check: description, pinned comment, linked pastebin/gist/Notion/docs. If the target
content is there, you are done — no frames, no transcript needed for that part.

## Layer 2 — Transcript

```bash
yt-dlp --skip-download --write-auto-subs --sub-langs "en,fr" -o base URL
```
Auto-subs VTT are bloated with rolling duplicates (~10:1). Reduce before reading:
strip tags/timestamps, drop consecutive duplicate lines, join. A 230 KB VTT becomes
~25 KB of clean prose. Analyze THAT, never the raw VTT.

While reading the transcript, collect two trigger lists:
- *Pointing phrases* → the speaker references the screen: "as you can see",
  "this chart", "right here", "on the screen", "let me show you", "look at this"
  (add your language's variants).
- *Content gaps* → the transcript announces something it doesn't contain ("the
  full prompt is on screen"). Each gap is a visual extraction target with a
  timestamp.

## Layer 3 — The screen

### Step 0 — coverage probe, then sweep (never rely on triggers alone)

Triggers tell you where to ZOOM; they must not decide WHETHER you look — content
shown without being announced would slip through. So:

```bash
# zero-token probe: how visually rich is this video?
ffmpeg -i video.mp4 -vf "select='gt(scene,0.30)',metadata=print" -f null - 2>&1 | grep -c pts_time
```
- Low scene count (talking head) → no visual pass needed, done.
- High scene count → MANDATORY coverage sweep: sample 8-16 frames across the FULL
  duration, tile into 1-2 contact sheets, read them. The sweep is your guarantee:
  any major on-screen content appears in the grid and earns a targeted dive, even
  if no one ever said "as you can see".

### Choosing the entry recipe (for the dives)

| Situation | Recipe |
|---|---|
| Speech + pointing phrases | Frames in ±20s windows around each trigger timestamp |
| Silent / no useful speech (screencast, live session) | Time-sample the whole span, contact-sheet it |
| Long video / live VOD (downloads stall or are huge) | **Don't download** — grab frames straight from the stream URL |
| Short video already downloaded | Scene-detect: `ffmpeg -vf "select='gt(scene,0.30)'" -vsync vfr` |

### Frame grabs without downloading (works on multi-hour live VODs)

```bash
URL=$(yt-dlp -g -f "b[height<=720]/bv*[height<=720]" VIDEO_URL | head -1)
ffmpeg -y -ss 3600 -i "$URL" -frames:v 1 -q:v 3 f_3600.jpg   # ~3s per frame
```
Loop over sampled timestamps (e.g. every 25-30 min across a 7 h session). This
bypasses YouTube's throttling of long live-VOD downloads entirely.

### Contact sheets — map many frames in ONE read

```bash
ffmpeg -y -start_number 1 -i seq_%02d.jpg -vf "scale=640:360,tile=4x2" -q:v 3 sheet_%d.jpg
```
One 4×2 sheet ≈ one vision read, and tells you which frames deserve full
resolution. Rule: about to Read more than ~10 images? Stop — sheet them first.
Then re-read only the 1-3 frames per distinct screen at full size.

### The sheet is the classifier — tag tiles, route by type

A sheet read is not just "looking": TAG every tile with a content type, then apply
that type's recipe. This is how the skill adapts to any subject — the types are
universal, only their value ranking shifts per niche:

| Tile type | Value | Recipe |
|---|---|---|
| TABLE / LIST (ingredients, positions, specs, rankings) | high | full-res single frame, transcribe verbatim, confirm on neighbor frame |
| DENSE TEXT (prompt, code, recipe, slide bullets) | high | full-res at 720p+ (crop+upscale if needed); scrolling → first/last frame, then 2s sampling if they differ |
| CHART / GRAPH | high | full-res; extract axes, annotations, drawn levels; if it evolves → 2-3 timestamps stacked in one image |
| UI / DASHBOARD | high | crop the info panels, zstack across timestamps |
| TEXT OVERLAY (lower-third, caption, quantity) | medium | overlays flash in/out — sample ±5-10s around the tile to catch the full text |
| FACE / TALKING HEAD | zero — the info is in the transcript | skip, never spend a read |
| B-ROLL / AMBIANCE (hands, pan, scenery) | context only | skip unless the question is about the gesture/technique itself |

### Budget checkpoints — degrade declared, never silent

If a budget is set, checkpoint at every sheet read: tokens spent vs. % of duration
covered. Coverage lagging spend (e.g. 80% spent, 50% covered) → stop unit
full-res reads, finish coverage of the unseen spans with sheets only, then put any
remainder into the single highest-value full-res dive. Degradation order (drop
first → last):

1. full-res re-reads of content already legible on a sheet
2. B-roll / context tiles
3. arc-tracking (same panel across multiple timestamps)
4. NEVER drop: the coverage sweep of unseen spans, and verbatim targets named by
   transcript content gaps.

The report always states what was NOT covered and what a bigger budget would add.

### Resolution floor — and the crop+upscale escape hatch

720p minimum for DIRECT reading of dense on-screen text (dashboards, prompts,
code). For thumbnails in sheets, 640×360 tiles keep large UI numbers readable;
drop to 4×2 (not 4×3) tiling when the source UI is dense.

When only low-res is obtainable (360p stream while 720p stalls): **crop the
panels of interest, upscale ×3, and stack several timestamps into one image** —
one vision read then covers the same panel across time:

```bash
ffmpeg -y -i f_1.jpg -i f_2.jpg -i f_3.jpg -filter_complex \
  "[0]crop=320:180:0:540,scale=960:540[a];[1]crop=320:180:0:540,scale=960:540[b];\
   [2]crop=320:180:0:540,scale=960:540[c];[a][b][c]vstack=3" zstack.jpg
```

Caveat: report upscaled-360p numbers with a last-digit reliability warning, and
confirm key figures at two reading scales.

## Validation

- Static overlays: transcribe from one frame, confirm on a neighbor (animations
  truncate lines mid-transition).
- Scrolling/paged content: compare first and last frame of the window; if they
  differ, sample every 2s and stitch.
- If a free-text source for the same content surfaces later, diff against it and
  report discrepancies — this also measures your own vision accuracy.

## Cost Heuristics (measured on real cases)

| Approach | Vision reads | Tokens |
|---|---|---|
| Naive: read every scene-change of a 20-min video one-by-one | 150-200 | 200k+ |
| Triggers → direct frame reads | 4-8 | 6-10k |
| Sampling → contact sheets → targeted full-res | 3-6 | 5-8k |
| EXHAUSTIVE mode via sheets (estimate, not the naive trap) | `ceil(scene_frames/8)` sheets + dives | `sheets × ~1.5k` + dives |

## Adapting to your niche

Before analyzing, state (or ask for) the user's niche, then build two lists:

1. **Pointing phrases** in the languages of the channels you follow.
2. **Screen targets** — what matters on-screen in that niche:
   - *Trading*: position tables (side/size/average price), P&L overlays, chart
     annotations and levels, order tickets, margin ratios.
   - *Entrepreneurship*: metric dashboards (MRR, churn), pricing pages, funnel
     screenshots, spreadsheet models.
   - *Health/science*: lecture slides, protocol tables, study screenshots, dosage
     charts.
   - *Code/tutorials*: prompts, config files, terminal output, architecture
     diagrams.

The extraction machinery is identical; only the trigger vocabulary and the target
list change.

## Common Mistakes

- Skipping Layer 1 and burning frames on content that was one
  `--print description` away.
- Skipping the coverage sweep because "the transcript has no pointing phrases" —
  that is exactly when unannounced visual content slips through.
- Grabbing the frame AT the pointing phrase: overlays appear 2-15s off — use a
  window.
- Reading frames one-by-one to "explore" — exploration is what sheets are for.
- Fighting a stalled live-VOD download instead of switching to URL frame grabs.
- Analyzing the raw VTT (10× token waste) instead of the reduced transcript.
- Trusting a single frame of animated/scrolling text.
- Spending reads on FACE/talking-head tiles — that information is in the transcript.
- Economizing on a short video where EXHAUSTIVE mode was affordable — missing
  content to save tokens that were available is the worse failure.
- Running out of budget mid-video silently instead of checkpointing and finishing
  coverage with sheets.

# analyzing-video-content — a Claude Code skill

**Extract what videos SHOW but never SAY.**

Transcript-based video analysis misses everything displayed on screen without being
spoken: the prompt a tutorial creator "puts on screen", the position book of a
silent day-trader, the slide a lecturer points at, the dashboard in a startup
breakdown. This skill teaches an AI agent to mine all three information layers of
a video — surrounding free text, transcript, and the screen itself — at ~30× less
token cost than naively reading frames.

Niche-agnostic: tell the agent your niche (trading, entrepreneurship, health,
coding, cooking…) and it adapts its trigger vocabulary and screen targets.
See "Adapting to your niche" inside the skill.

## Measured results (real cases, August 2026)

| Case | Result | Cost |
|---|---|---|
| 22-min tutorial, 3 long prompts shown on screen, never read aloud | All 3 extracted **verbatim** (validated letter-perfect against the creator's own pastebin) | 5 vision reads, ~7k tokens |
| 7-hour silent trading live | Full characterization: short-seller, ~12 simultaneous positions, hour-by-hour P&L arc, behavior — **without downloading the video** | 3 vision reads, ~6k tokens |
| Blind validation: fresh agent + this skill + a 4-hour FX live it had never seen | Autonomous report: broker, position sizes, pivot levels, P&L timeline, averaging-down detected | 4 vision reads |

## Install

```bash
mkdir -p ~/.claude/skills/analyzing-video-content
cp SKILL.md ~/.claude/skills/analyzing-video-content/
```

Prerequisites: `ffmpeg` and `yt-dlp` on PATH (for YouTube, a recent yt-dlp
nightly + a JS runtime like deno may be needed: `pip install -U --pre yt-dlp`).

## Key techniques inside

- **Need × cost modes**: EXHAUSTIVE (short video or "capture everything" — every
  scene change gets seen, via sheets), STANDARD (selective attention), MINIMAL
  (recon). The agent announces its mode and adapts depth to the actual need.
- **3-layer cost ladder**: free text → reduced transcript → targeted frames.
- **Content-type routing**: each contact-sheet tile is tagged (table, dense text,
  chart, UI, overlay, face, b-roll) and routed to a per-type extraction recipe —
  faces cost zero reads, tables get verbatim full-res transcription.
- **Budget checkpoints**: spend vs. coverage tracked at every sheet; a declared
  degradation order guarantees coverage is finished with sheets before budget dies —
  never silent truncation.
- **Coverage guarantee**: a zero-token ffmpeg scene-density probe decides IF the
  video is visually rich; a full-duration contact-sheet sweep guarantees no major
  on-screen content goes unseen — keyword triggers only decide where to zoom.
- **Frame grabs straight from the stream URL**: analyze multi-hour live VODs
  without downloading them (bypasses throttling entirely, ~3s per frame).
- **Contact sheets**: map 8-16 frames in a single vision read.
- **Crop + upscale ×3 + stack**: make low-res panels readable when HD stalls.

## Known limits

- Digits read from upscaled 360p can be off by one on the last digit — the skill
  mandates flagging this and cross-checking at two scales.
- YouTube changes its defenses regularly; keep yt-dlp fresh.
- The ~6-7k tokens/video figure assumes contact-sheet discipline.

Built with a TDD-for-documentation process (baseline test without the skill →
write → blind re-test with a fresh agent → refactor), on real videos.

MIT license.

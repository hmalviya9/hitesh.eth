---
name: data-motion-video
description: Build 30-second data motion-graphic videos and render them as deliverable MP4 files (no manual screen recording). Use this skill any time a user wants to create a short video, motion graphic, animated stat reveal, social media video, X/Twitter video, Instagram Reel, launch trailer, promo film, or any short-form animated content from data, dashboards, or numbers. Trigger on phrases like "make a 30-sec video", "render an MP4", "turn this into a video", "animate these stats", "make a Reel", "X video", "launch trailer for [project]", "data video", "promo for [project]", "social cut", or whenever the user has data and wants it as a video. Default to this skill for ANY video-from-data request, even if the user doesn't say "motion graphic" — if the deliverable is a short MP4 of typography, numbers, or animated data with a hook + reveal + CTA, this skill applies. Output is always a real MP4 file rendered server-side, never just a treatment.
---

# Data Motion-Graphic Video → Rendered MP4

You produce 30-second data motion-graphic videos as actual MP4 files. The user doesn't screen-record anything — you build a self-playing HTML animation, then render it to an MP4 via headless Chromium and ffmpeg, and hand them the file.

This skill exists because the alternative (writing a treatment and asking the user to record their browser) is bad. The user wanted the video, not homework.

## What this is for

- Promo videos / launch trailers for projects, dashboards, sites, products
- Data motion-graphics — taking a striking statistic and giving it a 30-second arc
- Hype reels for social media — X (16:9), Reels/TikTok (9:16), or 1:1 square
- Short editorial films built around typography and numbers, not stock footage

This skill is **not** for: AI-generated cinematic footage (use `avant-garde-music-video` for that), 60-second IG-data-viz Reels with the snappy Gen-Z vibe (use `data-viz-reels` if it's installed), or longer-form video editing. It's specifically for short, typography-led data films delivered as a final MP4.

## The pipeline

```
data + hook  →  single-file HTML animation  →  headless render  →  MP4
                  (auto-plays on load)         (Playwright + ffmpeg)
```

You build the HTML. The render script does the rest.

## Step 1 — Find the hook and the arc

Before opening any code, decide:

1. **The hook** — one number, fact, or claim that earns the first 3 seconds. Without a hook the viewer scrolls past. Examples that work: "70% of Indians lived below the poverty line in 1947." "Only 4% of Indian startups survive past year 7." "The dashboard tracks 22 manifesto promises." Bad hooks: "We built a thing." "Welcome to our project." A hook is a stat that makes the viewer want to know more.

2. **The arc** — the 30 seconds is structured as **hook → reveal → punchline → CTA**. Sketch a beat sheet in seconds before writing HTML:

```
0:00 – 0:03   Cold open — the hook (one big number, one caption)
0:03 – 0:08   Setup — build context, animate the data
0:08 – 0:21   Reveal — the surprising / contrarian / interesting bit
0:21 – 0:25   Punchline — the thesis stated plainly (often inverted bg for impact)
0:25 – 0:30   CTA — brand + URL
```

Specific timings vary per project, but every video has these four moves. **A 30-second video has 4–8 scenes total.** More than 8 means it'll feel like a slideshow.

3. **The thesis** — what the viewer should feel/believe by the end. State it in one sentence. The video exists to deliver this sentence with maximum punch.

## Step 2 — Build the self-playing HTML

Read `references/html-template.md` for the full reference template. Key rules:

- **Single file**, no external dependencies except Google Fonts.
- **Fixed design canvas** of 1920×1080 (or 1080×1920 for vertical), scaled to viewport via CSS transform on a `#stage` div. This gives pixel-perfect typography at any preview size and clean output at render time.
- **Editorial typography stack**: Fraunces (display serif, italic capable), IBM Plex Sans (UI), JetBrains Mono (data/labels). This combo always looks good with restraint.
- **Color palette** — prefer warm cream `#FAFAF7` background with stone-900 `#1C1917` text. Use accent colors sparingly for status (forest green `#2D5F3E`, gold `#B58A2C`, maroon `#8B2D2D`, slate `#4A6B82`). Match the source dashboard's palette if there is one.
- **Each scene is an absolutely-positioned div**. Show/hide via an `.active` class controlled by JS timeline.
- **The timeline is a JS array** of `{id, start, duration}` objects. A single `tick()` function reads `requestAnimationFrame` timestamps, calculates elapsed time, and switches scenes accordingly.
- **Scene entry animations** are CSS keyframes with delays (e.g., `animation: fadeUp 0.7s 0.2s forwards`). Keep them subtle — opacity + 20px slide is enough.
- **Persistent corner UI** — small brand mark + clock counter (`00:00 / 00:30`) in opposite corners. This lends professionalism and tells the viewer the run-time.
- **Progress bar** at bottom of stage, fills left-to-right over the 30 seconds.
- **Replay button** that's hidden during playback and appears at the end. Crucial: hide it via CSS during render so it doesn't show in the MP4.

The template in `references/html-template.md` has all of this scaffolded — copy it, then customize the scenes for the specific project.

## Step 3 — Test the HTML in browser before rendering

Once the HTML is built, do a quick visual sanity check by capturing 2-3 screenshots at key timestamps using Playwright. This catches issues like:

- Typography overflow at 1920×1080
- Wrong PM/year/value combinations in animated data
- Animation timing being off
- Scenes not transitioning cleanly

```js
const { chromium } = require('playwright');
const browser = await chromium.launch();
const ctx = await browser.newContext({ viewport: { width: 1920, height: 1080 }, deviceScaleFactor: 1 });
const page = await ctx.newPage();
await page.goto('file:///path/to/video.html');
await page.waitForTimeout(3000); // mid scene 1
await page.screenshot({ path: 'check-cold-open.png' });
await page.waitForTimeout(20000); // mid resolution
await page.screenshot({ path: 'check-resolution.png' });
await browser.close();
```

Fix any issues before rendering.

## Step 4 — Render the MP4

Run `scripts/render_mp4.js` to produce the final MP4. It handles:

1. Launching headless Chromium at 1920×1080 (or whatever you specify)
2. Recording video via Playwright's `recordVideo` to webm
3. Triggering animation playback at a known offset
4. Detecting the precise frame where animation visually starts (brightness drop)
5. Trimming webm and converting to H.264 MP4 at 30fps with X-friendly settings (`yuv420p`, `+faststart`, no audio)

```bash
node scripts/render_mp4.js \
  --input  /path/to/video.html \
  --output /path/to/output.mp4 \
  --duration 30 \
  --width 1920 \
  --height 1080
```

Default values: 30 seconds, 1920×1080. For Reels add `--width 1080 --height 1920`.

The script outputs an MP4 ready to post directly. Typical specs: ~2 MB file size, H.264 codec, 30fps CFR, no audio track. X autoplays muted anyway, and silent files leave the user free to add their own music in post.

## Step 5 — Deliver

When complete, present the MP4 file using `present_files`. Include:

- The MP4 itself
- The source HTML (so they can edit and re-render later — change the URL, swap a stat, etc.)
- A brief caption suggestion for the platform they're posting to

Optionally include a treatment markdown if the user wants to pass it to a real video editor for refinement, but the MP4 is the hero deliverable.

## Aspect ratio cheat sheet

| Platform | Aspect | Resolution | Notes |
|----------|--------|------------|-------|
| X / Twitter | 16:9 | 1920×1080 | Plays inline, no crop. Default. |
| Instagram Reels / TikTok | 9:16 | 1080×1920 | Vertical, plays full-screen on mobile |
| Instagram Feed | 1:1 | 1080×1080 | Square, less common now but safe |
| YouTube | 16:9 | 1920×1080 | Same as X, can also do 4K (3840×2160) |
| LinkedIn | 16:9 or 1:1 | 1920×1080 or 1080×1080 | LinkedIn prefers shorter for feed |

If the user doesn't specify, ask. If they're vague, default to 16:9 for X.

## Constraints to remember

- **30 seconds is the default** — that's the X auto-play attention span. Longer if explicitly requested, but push back gently. Anything past 60 seconds for social is a different format.
- **No music in the rendered MP4 by default**. Silence works on X (which mutes autoplay). If the user wants music, render silent and tell them to add audio in post (CapCut, Premiere, even iMovie). Avoid generating music server-side — copyright is messy, and the user usually has preferences.
- **No real public figures' faces or voices**. Typography-led design avoids this entirely. If the brief mentions "show Modi/Trump/whoever" — redirect to typography or symbolic representation.
- **Delivered MP4 must be under 50 MB** for easy posting. Default settings (1080p, 30fps, CRF 18) produce 1-3 MB videos — well under any platform cap.
- **Keep transitions restrained**. Hard cuts and short fades. No swooshes, no after-effects-style 3D, no glitter. Editorial register.

## What good looks like

The video should feel like a New Yorker piece set to motion: confident typography, restrained animation, one clear thesis, ends with the brand. If a frame from your video could appear in a Times Square ad for a payday loan company, it's wrong — strip it down.

The viewer should walk away with one fact and one feeling. The fact is the data. The feeling is "I should look at this thing properly."

## Common mistakes to avoid

- **Too many scenes.** 4–8 is the limit. Each scene is a load-bearing image, not filler.
- **Hook delayed past 3 seconds.** Brand-first openings ("Welcome to...") kill social reach. Lead with the data.
- **Animations that distract from the type.** The numbers and words ARE the actors. Decorative motion competes for attention.
- **Forgetting to hide the replay button before rendering.** It will show up in the final MP4 if you don't inject `display: none` via the render script. The included script does this — but if you customize, double-check.
- **Skipping the verification screenshots.** Always screenshot at least 2-3 key moments before doing the full render. Render time is ~45 seconds; iteration cost adds up.
- **Wrong trim point.** The webm includes ~3 seconds of page-load before animation starts. The render script handles this automatically by detecting the brightness drop, but if you write a custom variant, you must trim accurately or the MP4 will start with a blank cream frame.

## When NOT to use this skill

- The user wants a longer-form video (60s+). That's a different beast — likely needs real footage, music sync, voice-over.
- The user wants AI-generated cinematic footage (people, places, weird imagery). Use the avant-garde-music-video skill instead.
- The user wants you to actually pick music, narrate, or anything beyond visual + typography. Tell them the rendered MP4 is silent and they should add audio in post.
- The user wants a static social card (single image). Just use a Visualize tool or build an SVG, no video needed.

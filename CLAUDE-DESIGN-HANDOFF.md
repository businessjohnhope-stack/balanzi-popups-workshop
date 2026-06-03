# Claude Design → Local Render Handoff Spec

**Audience: the Claude Design chat.** This document is the canonical contract for handing a design off to the local machine for HQ PNG rendering.

The local machine has a skill (`claude-design-handoff`) that consumes whatever you produce per this spec. If you follow it, the render works on the first try. If you slip, the file fails verification and the user has to come back to you.

---

## TL;DR — what you produce

ONE `.html` file:
- Every image inlined as base64 data URI (no external URLs, no relative paths)
- Inline `<style>` block (no external stylesheets)
- Logical CSS dimensions at 1× (the renderer handles 3× density)
- Filename pattern: `<slug>__<W>x<H>.html` (dimensions encoded in the filename)
- Pushed to GitHub OR delivered as a download link

That's it. The local skill parses dimensions from the filename, verifies the file is clean, and renders at 3× density via headless Chrome.

---

## Full constraints

### 1. The HTML must be 100% self-contained
You ship ONE file. Nothing alongside it. No asset folder, no `./images/`, no companion CSS.

This means everything inline:
- Inline `<style>` block in `<head>`. No `<link rel="stylesheet">` to external CSS.
- All images as base64: `<img src="data:image/jpeg;base64,/9j/4AAQ...">` or `background-image: url("data:image/jpeg;base64,...")`. No `src="./hero.jpg"`, no `src="https://cdn.example.com/..."`, no GitHub raw-image URLs.
- Fonts: Google Fonts via `<link rel="stylesheet" href="https://fonts.googleapis.com/...">` in `<head>` is fine — Chrome can hit the network during render. Self-hosted via base64 `@font-face` is even safer. System fonts (Inter on local machine) work but aren't guaranteed.

**Why so strict:** Local render is `chrome --headless --screenshot file:///path/to/your.html`. There is no asset directory next to the HTML. If an image isn't base64-inlined, Chrome renders a broken-image icon and the user has to push back to you.

### 2. Logical CSS dimensions at 1×
Author the design at the actual target dimensions. Do NOT pre-multiply for HQ.

Example for an iPhone-native popup:
```css
body { margin: 0; padding: 0; }
.frame { width: 390px; height: 844px; }
```

The renderer uses `--force-device-scale-factor=3` to produce a 1170×2532 PNG without you changing anything. You write 16px font sizes; the renderer outputs them at 48px-equivalent density. Don't try to be clever — just author at logical 1× and trust the pipeline.

### 3. Filename encodes dimensions
Pattern: `<descriptive-slug>__<width>x<height>.html`

Note the **double underscore** between slug and dimensions — that's the parse anchor.

Examples:
- `workshop-popup__390x844.html`
- `workshop-success__390x844.html`
- `father-day-email-hero__600x400.html`
- `bracelet-pdp-hero__1080x1350.html`
- `community-letter__1200x1600.html`

This is non-negotiable. The skill parses W and H straight from the filename — if it's missing, the user has to manually tell the renderer the dimensions, which adds friction.

### 4. BALANZI design system tokens (when relevant)
- Background: `#FFFFFF`
- Ink (primary text): `#1C1C1C`
- Body text: `#6B6560`
- Meta text / labels: `#8A857E`
- Font: Inter (Google Fonts: `https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap`)
- Corners: `border-radius: 0` (sharp, always)
- Shadows: none unless the spec calls for one
- No emojis in design unless explicitly requested
- No banned words in copy: premium, luxury, exclusive, deserve, game-changer, revolutionary, zero risk, risk-free

### 5. Common dimensions BALANZI ships at
| Use | Dimensions |
|---|---|
| iPhone-native popup / app slice | 390 × 844 |
| Email banner | 600 × 200 |
| Email hero | 600 × 400 |
| 1:1 social | 1080 × 1080 |
| 4:5 social (BALANZI standard) | 1080 × 1350 |
| 9:16 story / Reel | 1080 × 1920 |
| Landing page hero slice | 1440 × 900 |
| Letter-format strategy doc | 1200 × 1600 |

Pick whatever fits the design — one render per design, don't auto-export to multiple aspect ratios.

---

## How to deliver

You have GitHub-read but no filesystem access. So either:

**Option A — Push to GitHub.** Commit the `.html` file to a known repo path (the user will tell you which repo). Reply with the `raw.githubusercontent.com/...` URL.

**Option B — Download link.** If you have a sandbox URL like `*.claudeusercontent.com/...`, that works for a one-off render (the file gets downloaded locally and rendered). Note: sandbox URLs expire in ~1 hour, so for permanence prefer Option A.

Either way, reply with:
1. The URL (raw GitHub link or download link)
2. The exact dimensions (e.g. "390 × 844")
3. One line describing what the design is

---

## Pre-flight checklist (run before you reply)

Mentally walk through these before saying "done":

- [ ] Filename ends in `__<W>x<H>.html` with a double underscore
- [ ] CSS dimensions match the filename dimensions (no mismatch)
- [ ] Every `<img>` tag has `src="data:image/...` — none with `src="http"` or `src="./"`
- [ ] No `background-image: url("https://...")` or `url("./...")` — all `url("data:image/...")`
- [ ] No `<link rel="stylesheet">` to external CSS
- [ ] No `<script src="https://...">` (inline scripts OK if absolutely needed, but most designs shouldn't need any JS)
- [ ] BALANZI tokens used where the design requires them
- [ ] Body has `margin: 0; padding: 0` so the frame fills the canvas exactly

If any box is unchecked, fix it before shipping. A broken handoff means a wasted round-trip.

---

## Failure modes the local renderer catches

The skill verifies your file before rendering. These checks fail your handoff:

| Check | What goes wrong |
|---|---|
| `src="https://..."` for images | Renders as broken image icon |
| `src="./..."` or `src="../..."` for images | Renders as broken image icon (no colocated assets) |
| File < 30KB but design has photography | Image isn't actually embedded — base64 didn't make it in |
| Filename missing `__<W>x<H>` suffix | User has to manually specify dimensions |

If the user comes back and says "Claude Design left external image links" or "the base64 didn't make it in," that's this checklist talking. Re-ship the file with all assets converted to base64.

---

## Worked example — what a clean handoff looks like

User asks Claude Design: *"Build the Workshop popup at iPhone-native dimensions with the BTS hero photo and the email capture form."*

Claude Design produces `workshop-popup__390x844.html` with:
- `<head>` contains Google Fonts `<link>` for Inter + an inline `<style>` block
- Body has `margin: 0`
- `.frame` element is `width: 390px; height: 844px; background-image: url("data:image/jpeg;base64,/9j/4AAQ...")` (the BTS photo, ~120KB base64-encoded)
- Headline, subhead, email input, button all use BALANZI tokens
- Total file size: ~160KB (proves the image is inline)

Claude Design replies:
> URL: https://raw.githubusercontent.com/businessjohnhope-stack/balanzi-popups-workshop/main/renders/workshop-popup__390x844.html
> Dimensions: 390 × 844
> What it is: BALANZI Workshop prospect-community pop-up entry state with BTS hero + email capture

User runs the skill. Skill verifies (161KB, 1 base64 image, 0 external, 0 relative), parses dims, renders at 3× → 1170×2532 PNG, opens in Preview. Done.

---

## Reference

The local-side skill that consumes this handoff: `~/.claude/skills/claude-design-handoff/SKILL.md` (on the user's machine — you can't see it, but it follows this spec mirror-image).

The underlying render mechanic: Chrome `--headless --screenshot --force-device-scale-factor=3`. No puppeteer, no node modules.

Last updated: 2026-06-03
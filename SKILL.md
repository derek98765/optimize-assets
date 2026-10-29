---
name: optimize-assets
description: Shrink the images and videos in this project's public and src/assets folders without visible quality loss. Images mode converts .png/.jpg/.jpeg to .webp, downscaling oversized ones to a safe size based on how large they actually render. Video mode converts .mov to web-safe .mp4 (H.264) and compresses oversized .mp4 files at near-original quality. Use when the user wants to reduce image or video file sizes, improve page load speed, or when the asset-optimization hook asks for it. Not for SVG or WebM, and not for general "optimize this" requests about code or performance.
allowed-tools: Read Glob Grep Bash
---

# Optimize assets

Two modes share one workflow. Pick the mode (or both), read its file, then follow
the shared steps below.

## Pick the mode

| Asked to optimize | Mode | Read |
|---|---|---|
| images, PNG/JPG/JPEG, "convert to webp" | **images** | `references/images.md` |
| videos, `.mov`/`.mp4`, "compress videos" | **video** | `references/video.md` |
| "assets", or both | both — run images, then video | both files |

Read only the file for your mode. The mode file covers detection, the safety
rules, the tool commands and verification; this file covers what both modes do
the same way.

## Scope

Look in **`public`** and **`src/assets`** only. These skills assume a Vite-style
app with `yarn`. Never touch `.svg` (already vector) or `.webm` (already
web-friendly), or files that are already `.webp`.

## Shared workflow

### 1. Detect

List the candidate files with their properties (the mode file has the exact
commands). Note what is already optimized and what's too small to be worth it.

### 2. Find every usage

The two locations are referenced differently, so search for both:

- **`public`**: string paths (`"/images/..."`, `"/videos/..."`, `<video src>`,
  `<source src>`, template literals, JSON data, CSS `url(...)`).
- **`src/assets`**: ES imports (`import hero from "@/assets/.../hero.png"`,
  relative `../assets/...`), `new URL(...)`, `require(...)`, and CSS `url(...)`
  in `src/**/*.css`. Resolve each import to the identifiers and JSX where it is
  used.

For every usage, find the largest size it renders at (CSS px) across all
breakpoints: `w-[..]`/`h-[..]`/size tokens, `md:`/`lg:` overrides, `max-w-*`,
parent container widths, `object-fit`/`aspect-*` crops and `background-size`.
Responsive `w-full` means the widest container it can fill, not the file's own
width. **If a file is used in several places, use the largest rendered size among
them.** Unknown or dynamic usage (unused, passed through props, API data, CSS you
can't trace) means: don't resize, only convert, and report it.

### 3. Show the plan, then change things

Show the user a table of what you intend to do, per file, with the reason (the
mode file lists the columns), **before changing anything**.

### 4. Never lose the only copy

Write results to a temp name or alongside the original first. Replace or delete
an original only after the new file is verified (smaller, or passed the quality
check) **and** its references are updated.

### 5. Update source references

- Static hardcoded paths: swap the extension directly.
- ES imports (`src/assets`): update the import specifier. Also `new URL(...)`,
  `require(...)` and CSS `url(...)`.
- Dynamic paths built from data: swap the extension at runtime with a
  `.replace(...)` on the variable, as the mode file shows.
- Vite fingerprints imports, so a missed one fails the build. Run `yarn build`
  (or `yarn tsc --noEmit`) afterwards to catch dangling imports.

### 6. Clean up and verify

Delete converted originals only after steps 4 and 5. Then search for leftovers:
nothing converted should remain, and nothing should still reference the old
name. Finish with a before/after table (file, old size, new size, % saved, plus
the mode's quality figure).

## Always

- **Quality first.** A result that's visibly softer, blockier or stuttery than the
  source is a failure. When in doubt, keep more pixels or bitrate.
- Never upscale. Never change aspect ratio.
- Anything that may be zoomed, downloaded, shown fullscreen, used as an OG/social
  image or used as a master keeps its original size.
- Use a checklist (TodoWrite) for the run.

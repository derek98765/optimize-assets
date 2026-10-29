# Video mode — .mov → .mp4 and mp4 compression

Read after `optimize-assets/SKILL.md` (shared scope, usage tracing, reference
updates and cleanup there).

Scan **`public`** and **`src/assets`** for videos and:

1. **Convert every `.mov` to `.mp4`** (H.264 + AAC, web-safe on Chrome, Safari
   and Firefox).
2. **Compress `.mp4` files that are larger than they need to be**, at
   near-original visual quality.

A result that is visibly softer, blockier or stuttery than the source is a
failure. When in doubt, keep more bitrate.

## Prerequisites

```bash
brew install ffmpeg   # provides ffmpeg and ffprobe
```

## 1. Detect

```bash
find public src/assets -type f \( -iname "*.mov" -o -iname "*.mp4" \) | sort | while read -r f; do
  info=$(ffprobe -v error -select_streams v:0 \
    -show_entries stream=codec_name,profile,width,height,r_frame_rate,pix_fmt,bit_rate:format=duration,bit_rate,size \
    -of csv=p=0 "$f" | tr '\n' ' ')
  a=$(ffprobe -v error -select_streams a:0 -show_entries stream=codec_name -of csv=p=0 "$f")
  echo "$(du -h "$f" | cut -f1)  $f  [$info]  audio=${a:-none}"
done
```

Record per file: video codec, resolution, fps, pixel format, duration, bitrate,
whether it has audio, and whether it is HDR/10-bit (`pix_fmt` like
`yuv420p10le`, or `color_transfer` of `smpte2084`/`arib-std-b67`). Leave `.webm`
and already-tiny files alone.

## 2. Usage notes (on top of the shared trace)

- **Audio needed?** If every usage is `muted` (autoplay loops, backgrounds,
  transitions), the audio track can be dropped with `-an`. If any usage plays
  sound, or usage is unknown, keep audio.
- The largest rendered size is used only to decide whether the video is oversized
  (rule 3 below).

## 3. Decide the action per file

| File | Action |
|---|---|
| `.mov` | **Always convert** to `.mp4`. |
| `.mp4` | **Compress only if** it passes the rules below. Otherwise leave it. |

Rules for `.mp4` compression (all must hold):

1. **Skip tiny files**: under ~1 MB, or a bitrate already below ~1 Mbps at
   1080p-equivalent.
2. **Skip already-efficient files**: an H.264 encode at roughly CRF 20+ won't
   shrink meaningfully. The real test is the encode itself (step 5): **keep the
   result only if it is at least 15% smaller**, otherwise discard it.
3. **Resolution:** never upscale and never change aspect ratio. Only downscale
   when the longest side is more than **1.5×** (largest rendered size × 2),
   rounded up to an even number. Within 1.5×, or with unknown/dynamic usage,
   keep the original resolution.
4. **Frame rate:** keep the original fps. Never drop frames.
5. Anything downloadable, fullscreen or a master/source asset keeps its original
   resolution.

Plan table columns: file, codec, resolution, size, bitrate, action, reason.

## 4. Encoding settings (near-original quality)

Constant-quality H.264, so each video gets the bits it needs and no more:

```bash
ffmpeg -hide_banner -y -i "$in" \
  -map 0:v:0 -map 0:a:0? \
  -c:v libx264 -crf 18 -preset slow -profile:v high -pix_fmt yuv420p -tag:v avc1 \
  -c:a aac -b:a 160k \
  -movflags +faststart \
  "$out"
```

- **CRF 18** is visually transparent for almost all content. For `.mp4`
  compression use **CRF 18–20** (go to 20 only for very high-bitrate sources
  such as screen recordings or clean gradients). Never go above 22.
- `-preset slow` compresses better at the same quality (`veryslow` for short
  clips if time allows).
- `-pix_fmt yuv420p`, `-tag:v avc1` and `+faststart` are required for
  Safari/iOS compatibility and instant playback.
- Add `-an` only when step 2 showed every usage is muted. Otherwise keep audio,
  re-encoding to AAC only if it isn't already AAC (`-c:a copy` when it is).
- Add `-vf "scale=<w>:-2:flags=lanczos"` only for the downscale case in rule 3.
- Don't set `-r` or `-vsync`, and don't trim. Duration and frame rate must match
  the source.

**`.mov` shortcut (no re-encode):** if the `.mov` video is already H.264
(`yuv420p`) and the audio is AAC or absent, remux, which is lossless and instant:

```bash
ffmpeg -hide_banner -y -i in.mov -c copy -movflags +faststart -tag:v avc1 out.mp4
```

Then apply the 15% compression check only if the result is large.

**HEVC / ProRes / 10-bit / HDR sources** (typical for iPhone `.mov`): re-encode
to H.264 with the settings above. For 10-bit or HDR sources, **do not** just
force `yuv420p`, which washes out colors. Tonemap to SDR first
(`-vf zscale=t=linear:npl=100,format=gbrpf32le,zscale=p=bt709:t=bt709:m=bt709:r=tv,format=yuv420p`)
or flag the file and skip it. Also copy the color tags:
`-colorspace bt709 -color_primaries bt709 -color_trc bt709`.

**iPhone rotation / variable frame rate:** ffmpeg applies rotation metadata
automatically. For variable-frame-rate footage keep `-fps_mode passthrough`.

## 5. Encode to a temp file, then verify

Encode to `name.optimized.mp4` (never in place), then check:

```bash
# Duration within 0.1s and same resolution/fps (unless deliberately downscaled)
ffprobe -v error -show_entries format=duration -of csv=p=0 "$in" "$out"

# SSIM should be >= 0.98 (PSNR > 40 dB) vs the source
ffmpeg -hide_banner -i "$out" -i "$in" -lavfi "[0:v][1:v]scale2ref=flags=bicubic[a][b];[a][b]ssim" -f null - 2>&1 | grep SSIM
```

Replace the original only if **all** pass:
- duration matches (±0.1 s) and frame count/fps match,
- SSIM ≥ 0.98 (if lower, re-encode with a lower CRF, e.g. 16–17),
- for `.mp4` compression, the output is **≥ 15% smaller**; otherwise delete the
  temp file and leave the original.

For `.mov` conversions the size gate doesn't apply (the point is the
container/codec), but report it if the `.mp4` came out larger than the `.mov`.
Spot-check a frame when the video has fine detail, text or gradients:

```bash
ffmpeg -ss 00:00:01 -i "$out" -frames:v 1 /tmp/check.png
```

## 6. References and cleanup

- **`.mov` → `.mp4`:** update static paths, ES import specifiers, `new URL(...)`,
  `require(...)` and CSS `url()`. Update `<source type="video/quicktime">` to
  `type="video/mp4"`.
- **Dynamic paths:** `` `/videos/${clip.file.replace(/\.mov$/i, ".mp4")}` ``
- **Compressed `.mp4` files keep the same name**, so no reference changes.
- Rename `name.optimized.mp4` to the final name, delete converted `.mov`
  originals only after their `.mp4` verified, and delete leftover temp files.

```bash
find public src/assets -type f \( -iname "*.mov" -o -iname "*.optimized.mp4" \)
grep -rnE "\.mov" src --include="*.jsx" --include="*.tsx" --include="*.js" --include="*.ts" --include="*.css"
```

Both should come back empty, aside from `.mov` files deliberately skipped, which
must be reported. Add SSIM to the final before/after table.

## Notes

- Always encode to a temp file first. A bad encode must never overwrite the only
  copy.
- Don't convert `.webm`.
- Don't re-encode an already-compressed `.mp4` repeatedly: every generation loses
  quality.
- If a file fails the quality check even at CRF 16, leave the original and tell
  the user.
- Prefer a larger-than-needed result over a visibly degraded one.

# Images mode — PNG/JPG/JPEG → WebP

Read after `optimize-assets/SKILL.md` (shared scope, usage tracing, reference
updates and cleanup there).

Convert all `.png` / `.jpg` / `.jpeg` images in `public/images` and `src/assets`
to `.webp`. Oversized images are downscaled first, but only to a *safe* size
derived from how large each image actually renders, so it stays clear at every
size and screen density.

## Prerequisites

```bash
brew install webp   # provides cwebp
```

## 1. Detect

```bash
find public/images src/assets -type f \( -iname "*.png" -o -iname "*.jpg" -o -iname "*.jpeg" \) | sort | while read -r f; do
  dims=$(sips -g pixelWidth -g pixelHeight "$f" 2>/dev/null | awk '/pixel/{printf $2" "}')
  size=$(du -sh "$f" | cut -f1)
  echo "$size  $dims  $f"
done
```

## 2. Compute the safe resize target

Don't use fixed tiers. Per image, after the shared usage trace:

```
required_px = max_rendered_css_px × density_factor
```

- `density_factor` = **3** for small fixed elements (icons, avatars, logos
  ≤ ~100 CSS px, cheap and the most visible when soft), otherwise **2**
  (standard retina). Use 3 for any image that is full-bleed on mobile.
- `max_rendered_css_px` is measured on the dimension that drives the render (the
  longest rendered side). If the box crops the image (`object-cover`), size so
  the *scaled* image still covers the box on both axes.
- Round `required_px` **up** to the next multiple of 16.

**Safety rules** (all must hold before resizing):

1. **Never upscale.** If the image is already ≤ `required_px`, leave it.
2. **Skip-if-close guard:** only resize when the longest side is **more than
   1.5× `required_px`**. Within 1.5×, keep the dimensions and just convert.
3. **Unknown or dynamic usage: don't resize.** Convert only, and report these.
4. **Preserve aspect ratio** and resize the longest side. Never crop.
5. **Never go below the largest rendered size × 2.**
6. Anything zoomable, lightboxed, downloadable or used as OG/social keeps its
   original size.

Plan table columns: file, original px, largest render px, `required_px`, action.

## 3. Resize (only images that passed the rules)

```bash
sips -Z <required_px> path/to/image.png --out path/to/image.png
```

## 4. Convert to WebP

Per folder (both `public/images/**` and `src/assets/**`):

```bash
for f in *.png; do
  [ -f "$f" ] || continue
  cwebp -q 100 "$f" -o "${f%.*}.webp"
done
# JPEG separately (zsh errors on *.jpg with no matches)
for f in *.jpg *.jpeg; do
  [ -f "$f" ] || continue
  cwebp -q 100 "$f" -o "${f%.*}.webp"
done
```

Quality 100 avoids visible compression artifacts, at the cost of larger files
than a lower setting. **Keep a `.webp` only if it is smaller than the original**;
otherwise delete the new `.webp` and leave the original and its references
untouched.

## 5. References and cleanup

Follow the shared steps. For dynamic paths built from API data
(`app.image = "finance_buddy.png"`):

```js
// Before
`/images/app-icons/${app.image}`
// After
`/images/app-icons/${app.image.replace(/\.(png|jpe?g)$/i, ".webp")}`
```

Verify:

```bash
find public/images src/assets -type f \( -iname "*.png" -o -iname "*.jpg" -o -iname "*.jpeg" \)
grep -rnE "\.(png|jpe?g)" src --include="*.jsx" --include="*.tsx" --include="*.js" --include="*.ts" --include="*.css"
```

## Notes

- Never convert `.svg`. Already-`.webp` files need no action.
- `sips -Z <n>` resizes the longest side and keeps the aspect ratio.
- If the filename comes from an external API you don't control, use the runtime
  `.replace()` rather than renaming files.
- Prefer a larger-than-needed result over a soft one.

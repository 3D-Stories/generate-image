---
name: generate-image
description: >-
  Generate images for websites, presentations, and marketing materials using AI
  models on Replicate (Nano Banana 2, FLUX Schnell) with optional background
  removal for transparent PNGs. Use this skill whenever the user asks to create,
  generate, or make images, illustrations, icons, hero images, OG images, logos,
  favicons, or any visual asset. Also use when implementing issues that require
  visual assets, or when the user says "generate an image", "create an
  illustration", "make me an icon", "I need a visual for...", "remove
  background", "make transparent", or references image generation in any context.
  If a project needs visual assets and you're about to use placeholder text/emoji
  instead, use this skill to generate real assets.
---

# Image Generation Skill

Generate production-ready visual assets using AI models on Replicate. Handles model selection, prompt engineering, background removal, and file management.

**Before proceeding, verify you have tools available starting with `mcp__replicate__`. If not, tell the user: "Replicate MCP not detected. Run `/generate-image:setup` to configure it." and stop.**

**No AI image model generates true transparency.** When a transparent background is needed, always use a two-step pipeline: generate the image with Nano Banana 2 (on a solid white background), then remove the background with `recraft-ai/recraft-remove-background`. This pipeline is automatic — do not ask the user whether to remove the background when transparency is requested or implied (icons, illustrations, logos).

## Models

> Models last verified: 2026-04-05. If a model ID returns an error, check Replicate for the current model name and file an issue at 3D-Stories/generate-image.

| Model | ID | Best For | Cost | Speed |
|-------|----|----------|------|-------|
| **Nano Banana 2** | `google/nano-banana-2` | Icons, illustrations, complex scenes, text in images, high quality | ~$0.04/image | ~20s |
| **FLUX Schnell** | `black-forest-labs/flux-schnell` | Quick iterations, drafts, bulk low-cost generation | ~$0.003/image | ~0.5s |
| **Background Remover** | `recraft-ai/recraft-remove-background` | Transparent PNGs from any image | ~$0.01/image | ~5s |

### Model Selection

- **Default to Nano Banana 2** for production assets — the quality difference over FLUX is significant for icons and illustrations. NB2 produces cleaner, more coherent compositions with better color adherence.
- **Use FLUX Schnell** only for quick drafts, rapid iteration when exploring styles, or bulk generation where cost matters more than quality.
- **Neither NB2 nor FLUX can generate transparent backgrounds.** Any request for transparency (explicit or implied — icons, illustrations, logos) requires the automatic two-step pipeline: generate on white -> recraft bg removal. Do not attempt to prompt NB2/FLUX for transparency; it does not work.

### Supported Aspect Ratios

**Nano Banana 2:** `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9`, `1:4`, `4:1`, `1:8`, `8:1`

**FLUX Schnell:** `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9`, `9:21`

| Use Case | Aspect Ratio | Model Notes |
|----------|-------------|-------------|
| Icon / Favicon source | 1:1 | Both models |
| Hero image | 16:9 | Both models |
| OG image (social) | 21:9 | Generate at 21:9, crop to 1200x630 |
| Instagram post | 1:1 | Both models |
| Instagram story | 9:16 | Both models |
| Twitter/X header | 21:9 | Closest to 3:1; crop if needed |
| App store screenshot | 9:16 | Both models |
| Email header | 21:9 | Closest to 3:1; crop if needed |
| Tall banner | 1:4 | NB2 only |

For use cases needing ratios not in the supported list, generate at the closest supported ratio and crop to the target dimensions.

## Workflow

### Step 1: Check for a Style Guide

Before generating, look for `style-guide.md` in the project root. If it exists, read it and incorporate the style directives (colors, visual style, mood) into your prompts. If it doesn't exist and this is the first generation for the project, suggest creating one with `/superpowers:brainstorm`.

### Step 2: Construct the Prompt

Build prompts in layers. The background color choice is critical for later bg removal:

1. **Background:** Always use `"on solid white background"` — white backgrounds produce dramatically cleaner bg removal than black. Black backgrounds cause dark halos and artifacts around edges.
2. **Style:** From the style guide (e.g., "Flat vector illustration icon")
3. **Subject:** Be specific: "a gold coin with a dollar sign embossed in the center" not "a coin"
4. **Colors:** Reference hex codes: "gold hex fbbf24", "purple hex a78bfa"
5. **Composition:** "Centered, clean minimal design"
6. **Constraints:** "suitable for 64px display. No text labels, no border, no other objects."

**Template:**
```
[Style]. [Subject with specific details]. On solid white background. [Color hex references]. Clean minimal design suitable for [size]px display. [Constraints]. Centered.
```

**Example:**
```
Flat vector illustration icon on solid white background. A golden trophy cup with two handles on a pedestal, gold hex fbbf24 with subtle highlights. Clean minimal flat design suitable for 64px display. No text, no border, centered.
```

### Step 3: Generate via Replicate MCP

Use `mcp__replicate__create_models_predictions`:

**Nano Banana 2:**
```
model_owner: "google"
model_name: "nano-banana-2"
Prefer: "wait"
input: {"prompt": "...", "aspect_ratio": "1:1", "number_of_images": 1}
```

**FLUX Schnell:**
```
model_owner: "black-forest-labs"
model_name: "flux-schnell"
Prefer: "wait"
input: {"prompt": "...", "num_outputs": 1, "aspect_ratio": "1:1", "output_format": "png"}
```

**Recraft Remove Background:**
```
model_owner: "recraft-ai"
model_name: "recraft-remove-background"
Prefer: "wait"
input: {"image": "https://..."}
```
The `image` input accepts any HTTP URL — including Replicate output URLs from a prior generation step. This means you can chain generation -> bg removal without downloading/re-uploading.

Download each output URL with `curl -sL`.

### Error Handling

- **Prediction fails:** Check the error message from Replicate. Common causes: invalid API token (re-run `/generate-image:setup`), model temporarily unavailable (try again in a minute), malformed input (check parameter names and types).
- **Output URL empty or 404:** Retry the prediction once. If it fails again, report the prediction ID to the user for debugging: "Prediction `<id>` returned no output. Check status at replicate.com/predictions/<id>."
- **Cost guard:** Before starting a batch operation, calculate the estimated cost. If the estimated cost exceeds $1, show the breakdown and ask the user for confirmation before proceeding. Example: "This batch of 30 icons will cost approximately $1.50 (30 x $0.05 per NB2 + bg removal). Proceed?"

### Step 4: Remove Background (automatic for transparent images)

When the user requests a transparent background — or when the use case implies it (icons, illustrations, logos) — this step runs automatically after generation. Do not ask the user; just do it.

1. **Download and save the original** from Step 3's output URL to an `originals/` subdirectory
2. **Run bg removal via MCP** — pass the generation output URL directly as the `image` input (no re-upload needed):
   ```
   mcp__replicate__create_models_predictions:
     model_owner: "recraft-ai"
     model_name: "recraft-remove-background"
     Prefer: "wait"
     input: {"image": "<GENERATION_OUTPUT_URL>"}
   ```
3. **Download the transparent PNG** result with `curl -sL`

**For existing local images** (not from a prior generation), upload first using `mcp__replicate__create_files`, then pass the returned URL as the `image` input to bg removal.

For batch processing, launch all bg removal MCP calls in parallel — they complete in ~5s each.

**When to skip bg removal** (opaque background is intentional):
- Full-bleed hero images or section backgrounds
- OG images (social sharing — needs solid background)
- Images where the background IS part of the design

### Step 5: Save the Output

**Determine the output directory:**

1. Search the project for existing image files (`*.png`, `*.svg`, `*.jpg`). If found, use the most common parent directory — this respects the project's established conventions.
2. If no existing images, check for common directories: `public/images/`, `assets/images/`, `static/images/`, `src/assets/images/`. Use the first that exists.
3. If nothing matches, ask the user: "Where should I save generated images?"

**Save files:**

1. **Originals:** Save to an `originals/` subdirectory within the chosen output directory (before bg removal)
2. **Final files:** Save to the output directory
3. **Filename:** Descriptive kebab-case: `icon-gold-coin.png`, `hero-phone-mockup.png`
4. **Format:** PNG for transparent, WebP for opaque hero/background images

### Step 6: Report

```
Generated: <output-path>/icon-gold-coin.png
Model: google/nano-banana-2 + recraft-ai bg removal
Prompt: [prompt used]
Size: [file size]
Background: transparent
```

## Smart Defaults

| Use Case | Model | Background | BG Removal | Aspect Ratio | Post-Processing |
|----------|-------|-----------|------------|-------------|-----------------|
| Icon | NB2 | White | Yes | 1:1 | — |
| Hero image | NB2 | Scene bg | No | 16:9 | — |
| OG image | NB2 | Designed bg | No | 21:9, crop to 1200x630 | — |
| Section illustration | NB2 | White | Yes | 16:9 | — |
| Logo cleanup | — | — | Yes (existing image) | 1:1 | — |
| Quick draft | FLUX | White | Optional | 1:1 | — |
| Favicon | NB2 | White | Yes | 1:1 | ICO conversion (see below) |

### Favicon ICO Conversion

After generating and bg-removing a favicon source image, convert to multi-size ICO format.

Check for available conversion tools in this order:

1. **Pillow** — check: `python3 -c "from PIL import Image"`
   ```python
   python3 -c "
   from PIL import Image
   img = Image.open('favicon.png')
   img.save('favicon.ico', format='ICO', sizes=[(16,16),(32,32),(48,48),(64,64),(128,128),(256,256)])
   "
   ```

2. **ImageMagick** — check: `convert --version`
   ```bash
   convert favicon.png -define icon:auto-resize=256,128,64,48,32,16 favicon.ico
   ```

3. **Neither available** — tell the user: "ICO conversion requires Pillow (`pip install Pillow`) or ImageMagick (`apt install imagemagick`). Install one and re-run."

## Batch Generation

For multiple related assets (icon sets, etc.):

1. Use NB2 for all icons in a batch — consistency matters more than speed
2. Generate ALL on white backgrounds with the same style prefix
3. Keep all originals in `originals/` subdirectory
4. Run bg removal as a batch after all generation completes — launch all MCP calls in parallel, they complete in ~5s each
5. Show the user 2-3 results before completing the full batch — catch style issues early
6. **Cost guard:** Calculate total estimated cost before starting. If over $1, show the estimate and confirm.

## Background Removal for Existing Images

This skill also handles removing backgrounds from existing images (logos, photos, user-provided assets). The same `recraft-ai/recraft-remove-background` model works on any image:

1. **Upload the local image** using `mcp__replicate__create_files` (provides an HTTP URL for the file)
2. **Run bg removal** using `mcp__replicate__create_models_predictions` with `model_owner: "recraft-ai"`, `model_name: "recraft-remove-background"`, `Prefer: "wait"`, and `input: {"image": "<FILE_URL>"}`
3. **Download** the transparent PNG output with `curl -sL`

## Troubleshooting

- **Dark halos after bg removal:** The source image was generated on a black background. Regenerate on white and re-run bg removal.
- **BG removal ate part of the image:** The subject color was too close to the background. Try generating on a contrasting background color.
- **Images look too small on page:** AI generates at 1024x1024 but display size matters. For icons on dark backgrounds, 84-126px display width reads well. Don't go smaller than 64px for feature cards.
- **Wrong colors:** Be explicit with hex codes. "gold hex fbbf24" works better than "golden".
- **Quality not good enough:** Switch from FLUX to NB2. The quality difference is worth the 13x cost increase for production assets.

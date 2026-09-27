# controlnet-workflow

A ComfyUI SDXL graph (`image_sdxl_controlnet_v1.json`) for generating a
consistent character across different poses and expressions — built as
source material for training a character LoRA (face/skin/build identity,
not tied to any specific pose or action).

The graph is edited directly as JSON (litegraph format), not through the
ComfyUI UI. ComfyUI itself runs on a remote instance (endpoint/API key in
the gitignored `.env`).

## Requirements

- Checkpoint: `epicrealismXL_pureFix.safetensors`
- ControlNet: `controlnet-union-sdxl-1.0.safetensors` (xinsir)
- Custom nodes: rgthree-comfy (Power Lora Loader, Fast Groups Bypasser),
  RvTools (Latent Switch)
- A `poses` subfolder in ComfyUI's own `input` directory, populated with
  pre-rendered OpenPose skeleton images (body keypoints only — see
  gotchas below)

## Graph modes

One graph, three modes, toggled by hand — nothing auto-switches:

1. **Plain txt2img** — "ControlNet" and "Img2Img" rows both bypassed on
   the Fast Groups Bypasser (node 57), Latent Switch (node 45) set to
   `2` (EmptyLatentImage).
2. **txt2img + ControlNet pose** — "ControlNet" row enabled, "Img2Img"
   still bypassed, switch still on `2`. Node 54
   (`LoadImageDataSetFromFolder`) points at the `poses` subfolder and,
   because it outputs a list, one Queue Prompt click runs the whole
   folder — one generation per pose image.
3. **img2img variation pass** — "Img2Img" row enabled (LoadImageOutput →
   VAEEncode → RepeatLatentBatch), Latent Switch set to `1`.
   `RepeatLatentBatch` (node 58) fans one source image out into several
   samples per click (currently 6) so you get multiple variants of the
   same edit — e.g. different emotional expressions from one neutral
   base image — in a single queue.

Bypassing a group and setting the Latent Switch are two separate manual
steps. Don't put node 45 (the switch) inside a bypass group — bypass
forwards `input1` regardless of the switch's own setting, so it can't
safely fall back to the other input on its own.

## Gotchas learned the hard way

- **`denoise` only means "already this noisy" for real img2img.**
  Applying `denoise < 1.0` to a latent from `EmptyLatentImage` (pure
  noise, plain txt2img) starves the sampling schedule and produces
  garbled/hallucinated results. Only lower it when the latent came from
  an actual `VAEEncode`d image.
- **`controlnet-union-sdxl-1.0` was not trained on hand/face
  annotations.** Don't feed it face-only OpenPose keypoints for
  expression control — use Canny, Lineart, or Depth instead (with an
  actual preprocessor node, since this graph currently just loads
  pre-rendered images directly).
- **Body-only pose skeletons have no hand keypoints**, so an
  outstretched/raised hand can get a hallucinated prop (a bag, flowers,
  etc.) — the model fills in "what a hand there usually holds." Add
  exclusions to the negative prompt (`holding object, bouquet, bag`) and
  `empty hands` to the positive if it recurs.
- **LoRA slots in node 27 are placeholders** (`lora 1`..`lora 4`) —
  swap in real filenames locally when needed, don't commit real ones
  back (this repo is public).

## Files

- `image_sdxl_controlnet_v1.json` — the graph.
- `CLAUDE.md` — session notes for AI-assisted editing of this graph.
- `.env` (gitignored) — remote ComfyUI endpoint/API key.

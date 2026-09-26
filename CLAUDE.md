# controlnet-workflow

ComfyUI SDXL workflow: base positive/negative prompting + manual openpose
injection via `controlnet-union-sdxl-1.0.safetensors`. The graph lives in
`image_sdxl_controlnet_v1.json` and is edited directly as JSON (litegraph
format), not through the ComfyUI UI, in this repo.

ComfyUI itself runs remotely (see `.env`, gitignored — holds real
endpoints/API keys, never commit it).

## Status as of 2026-09-26

Fixed a broken ControlNet branch and added a one-click enable/disable
toggle. Three commits so far:

1. `0aab325` — initial import of the workflow JSON.
2. `e234d86` — root-cause fix: `controlnet-union-sdxl-1.0` is a
   multi-task model and the graph was missing a `SetUnionControlNetType`
   node (id 56) telling it which task to run. Inserted it between
   `ControlNetLoader` (id 55) and `ControlNetApplyAdvanced` (id 38), set
   to `"openpose"`. (Nodes 54/38 being in Bypass mode was *not* a bug —
   that's the user's intentional manual toggle for running plain
   txt2img/img2img without ControlNet.)
3. `d6ef4c6` — replaced the manual per-node bypass toggling with a
   single control: grouped nodes 54 (pose image loader), 55
   (`ControlNetLoader`), 56 (`SetUnionControlNetType`), 38
   (`ControlNetApplyAdvanced`) into a group titled "ControlNet", and
   added node 57, `Fast Groups Bypasser (rgthree)` (titled "ControlNet
   On/Off"), which bypasses/enables that whole group with one click.

Current default state: ControlNet branch is enabled (mode 0).

## Open items / to verify next session

- User is about to test-run the graph in ComfyUI. Need to confirm:
  - The "ControlNet On/Off" node (id 57) actually shows a toggle row for
    the "ControlNet" group when the graph loads — `Fast Groups Bypasser`
    builds its toggle list dynamically by scanning groups, so if it
    doesn't show, try a right-click refresh on the node.
  - Whether `controlnet-union-sdxl-1.0` actually needs a preprocessed
    OpenPose skeleton image (it does) — user confirmed their "poses"
    images are already pre-rendered skeleton maps, not raw photos, so no
    preprocessor node was added. Revisit if that's not actually true for
    all images they plan to use.
  - Node 38's `strength` widget is `1.5` (widgets_values: `[1.5, 0, 1]`
    = strength, start_percent, end_percent) — flagged as possibly too
    strong; may need to drop toward 0.6–1.0 depending on results.
- No other plumbing conflicts found between the ControlNet branch and
  the base prompting/latent path — when the group is bypassed, litegraph
  cleanly passes `positive`/`negative` conditioning straight through to
  the KSampler unchanged.

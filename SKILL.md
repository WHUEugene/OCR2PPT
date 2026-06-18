---
name: ocr2ppt
description: Convert slide screenshots or rendered PPT page images into editable PPTX with OCR-guided text rebuild, clean-base generation, visual element extraction, and reconstruction QA. Use for OCR2PPT, img-to-pptx, screenshot-to-editable-PPT, slide image reconstruction, OCR text removal, PPT element extraction, SAM/BEN2/BiRefNet cutout experiments, and step-by-step visual QA.
---

# OCR2PPT

OCR2PPT reconstructs editable PowerPoint decks from slide images. Treat the page as layers, not as one image:

1. clean background/base image
2. visual elements and simple geometry
3. editable OCR text boxes
4. reconstruction manifest and visual QA

Always generate step visualizations before claiming a better pipeline.

## Default Workflow

For each page:

1. **Use the best source image**
   - Start from the original rendered slide PNG whenever available.
   - Avoid low-resolution screenshots unless no better source exists.

2. **Run GPU OCR and text/graphic split**
   - Use GPU inference when OCR/model work is needed.
   - Current tested OCR: PaddleOCR PP-OCRv6 medium detection/recognition.
   - Classify OCR hits into `text_rebuild`, `text_review`, and `graphic_element`.
   - Keep logo/brand OCR hits as graphics; do not erase or rebuild logo text as ordinary slide text.

3. **Create the text-removal mask**
   - Default mask: OCR text bbox/poly for `text_rebuild` and approved `text_review` only.
   - Exclude `graphic_element` OCR boxes from text removal.
   - Use OCR bbox/poly when clean removal matters most.
   - Use refined glyph masks only when OCR boxes leave unacceptable rectangular artifacts.

4. **Generate the clean base with nearest-background fill**
   - Default erasure is **nearest unmasked background color fill with light feathering**.
   - For each masked pixel, copy/interpolate from the nearest non-masked pixel, then feather the edge.
   - This is the primary path for PPT-like white/red/gray text regions because it avoids LaMa-style dirty texture, warped pure-color bars, and AI hallucinated marks.
   - Keep LaMa/MAT/ZITS/MIGAN/FcF only as fallbacks for genuinely complex image backgrounds.

5. **Extract visual elements**
   - Do not use SAM automatic masks as the sole source of truth.
   - Run SAM/SAM2 on the clean base only as candidate generation for non-text visual objects.
   - For crop-level foreground extraction, prefer tested candidates:
     - `BEN2`: often better for cards/icons and fuller foreground crops.
     - `BiRefNet`: often better for larger foreground/whole-flow crops.
   - Avoid `BiRefNet_HR-matting` and `RMBG-1.4` as defaults for PPT pages; they were too fragmentary in tests.
   - Remember: `BEN2` and `BiRefNet` are element-cutout alternatives, not text-erasure/LaMa replacements.

6. **Reconstruct geometry**
   - Rebuild simple pure-color rectangles, bars, arrows, panels, and capsules as native PPT shapes where practical.
   - Do not force these through SAM or matting models; model masks often produce wavy edges on simple geometry.

7. **Rebuild editable text**
   - Reinsert OCR text as editable PowerPoint text boxes.
   - OCR does not reliably provide font color; estimate text color from original pixels and verify visually.
   - Make text boxes long enough. Many bad layouts come from short boxes that force wrapping.

8. **Assemble PPTX**
   - Layer order: clean base, native shapes/cutout images, editable text.
   - Write a manifest mapping each reconstructed item to source page, bbox, method, and asset path.

## Required QA Contact Sheet

For every pipeline change, output a side-by-side contact sheet containing at least:

1. source page
2. OCR/text-graphic split
3. OCR color/text sampling overlay if available
4. text-removal mask
5. clean base after text removal
6. baseline clean base if comparing methods
7. visual element candidates
8. rebuild plan overlay
9. rendered/reinsert preview

Do not report that a method is better without showing this visual comparison.

## Validated Decisions From 2026-06-18

Validated on `/Users/linyong/Downloads/ppt_generated_slides/03.png`:

- OCR bbox + nearest-background fill produced a cleaner PPT base than LaMa for regular slide layouts.
- LaMa, MAT, ZITS, MIGAN, and FcF were compared with the same mask; none clearly beat LaMa, and FcF damaged nearby logo content.
- The core failure was mask strategy and treating PPT text removal as generic inpainting, not simply choosing the wrong inpainting model.
- OCR box fill removes text cleanly but can leave rectangular artifacts on colored bars.
- Glyph masks reduce rectangular artifacts but are harder to make complete and can leave residual strokes.
- Best current default: OCR text boxes constrain removal, nearest-background fill creates the clean base, then OCR text is rebuilt as editable text.

Local reference artifacts:

- Nearest-background full-pipeline contact sheet:
  `/Users/linyong/vscode/AIPPT_260103/outputs/manual-20260611-sam-full-elements/experiments/nearest_bg_full_pipeline_03_20260618/nearest_bg_full_pipeline_03_contact_sheet.jpg`
- OCR box fill comparison:
  `/Users/linyong/vscode/AIPPT_260103/outputs/manual-20260611-sam-full-elements/experiments/new_img2pptx_pipeline_try_generated03/ocr_box_fill/ocr_box_fill_contact_sheet.jpg`
- Inpainting model comparison:
  `/Users/linyong/vscode/AIPPT_260103/outputs/manual-20260611-sam-full-elements/experiments/inpaint_model_compare_03_20260618/inpaint_model_compare_03_contact_sheet.jpg`
- Cutout model comparison:
  `/Users/linyong/vscode/AIPPT_260103/outputs/manual-20260611-sam-full-elements/experiments/cutout_model_compare_20260618/cutout_model_selected_comparison_4models.jpg`

## GPU Server Notes

Before using remote machines, read:

`/Users/linyong/.codex/CODEX_RESOURCES.md`

Known GPU server:

`user2@219.223.194.233`

Observed working roots:

- OCR: `/data2/user2_dir/lyj/aippt_ocr_20260618`
- IOPaint/inpainting models: `/data2/user2_dir/lyj/aippt_text_erase_20260618`
- Cutout model tests: `/data2/user2_dir/lyj/aippt_cutout_models_20260618`

Keep large weights and intermediate model outputs on `/data2`. Copy back only contact sheets, manifests, and selected assets needed for local review.

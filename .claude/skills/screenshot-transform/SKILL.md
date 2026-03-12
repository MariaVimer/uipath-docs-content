# Screenshot Transform

Transform a raw UI screenshot into a documentation-ready image following the UiPath Automation Cloud screenshot standards.

## Usage

```
/screenshot-transform
```

## Variables

| Variable | Description | Example |
|---|---|---|
| `INPUT` | Path to the raw screenshot file | `images/_raw/my-screenshot.png` |
| `TEMPLATE` | Template type to apply | `modal`, `overview`, `step`, `config-panel`, `table`, `auto` |
| `ANNOTATE` | Comma-separated callout definitions | `"1:Save button,2:Name field"` |
| `BLUR` | Data types to auto-blur | `email`, `username`, `tenant`, `token` |
| `NAME` | Override for the output filename | `orchestrator-add-queue-dialog.webp` |
| `OUTPUT_DIR` | Output directory for the processed image | `images/orchestrator/` |

## Instructions

### Step 1: Collect Variables

Ask for any missing required variables. `INPUT` is always required. For `TEMPLATE`, if not provided use `auto` and classify in Step 2. For `BLUR`, default to `email,username,tenant` if not specified.

### Step 2: Analyze the Screenshot

Use your vision capability to analyze the raw screenshot at `INPUT`. Produce a structured analysis:

```
page_type:         [modal | config-panel | table | step | overview]
page_title:        [detected page or modal title]
product_area:      [orchestrator | automation-cloud | action-center | apps | insights]
primary_elements:  [list of buttons, fields, dropdowns with approximate positions]
sensitive_regions: [list of regions containing email, username, tenant, token data]
suggested_crop:    [description of the ideal crop region]
```

If `TEMPLATE` is `auto`, classify based on these rules:
- Overlay with darkened background → `modal`
- Side panel occupying 25–40% of width → `config-panel`
- Tabular data with header row → `table`
- Single button/field focus < 400px → `step`
- Full-page layout with nav → `overview`

### Step 3: Define the Crop

Apply the crop rule for the selected template:

| Template | Crop Rule | Padding |
|---|---|---|
| `modal` | Modal border | 16px all sides |
| `step` | Target element bounds | 24px all sides, min 400px wide |
| `overview` | Page content boundary | 0px |
| `config-panel` | Panel boundary | 0px |
| `table` | Table + toolbar boundary | 0px |

Always exclude:
- Browser chrome (address bar, tabs, window controls)
- OS UI (taskbar, dock)
- Unrelated left or top navigation (unless `overview` template)
- Empty space larger than 24px on any edge

### Step 4: Generate the Sharp Script

Write a Node.js script using `sharp` to:

1. **Crop** the image to the region defined in Step 3
2. **Blur sensitive regions** — for each region in `BLUR` or detected in Step 2:
   - Apply Gaussian blur sigma 8 to the region
   - Use a rounded rectangle composite (4px radius)
3. **Export** as WebP at 90% quality, scaled to 1× (halve pixel dimensions if captured at 2×)

```js
// Example structure — fill in actual coordinates from Step 2 analysis
const sharp = require('sharp');

const crop = { left: X, top: Y, width: W, height: H };

sharp('INPUT')
  .extract(crop)
  // blur sensitive regions via composite
  .webp({ quality: 90 })
  .toFile('OUTPUT_DIR/FILENAME.webp');
```

Provide the complete, runnable script with all coordinates filled in from the analysis.

### Step 5: Generate the Filename

If `NAME` is not provided, generate a filename using the pattern:

```
{product-area}-{action-or-feature}-{optional-step}.webp
```

Rules:
- All lowercase, hyphens only
- `product-area`: from detected page/product area (e.g., `orchestrator`, `automation-cloud`)
- `action`: from modal title, page heading, or primary button label
- `step`: include only if `ANNOTATE` contains sequential numbered callouts

Examples:
```
orchestrator-create-queue.webp
automation-cloud-invite-user-step1.webp
action-center-assign-task-modal.webp
```

### Step 6: Define Annotations

For each entry in `ANNOTATE` (format `"N:element label"`):

Generate SVG or canvas draw calls to place:

**Callout circle:**
```
Shape:    Filled circle, 24×24px
Color:    #FA4616 fill, white text
Font:     Inter Bold 13px
Position: Top-left of target element, offset -12px x, -12px y
           If overlap: try top-right, bottom-left, bottom-right
Connector: 2px line #FA4616 from callout center to element center
            (only if callout is >20px from element edge)
```

**Highlight box** (when `--highlight` is used or template is `step`/`modal`):
```
Shape:    Rectangle, border-radius 4px
Border:   2px solid #FA4616
Fill:     rgba(250, 70, 22, 0.08)
Bounds:   element.x - 4px, element.y - 4px,
          element.width + 8px, element.height + 8px
```

Never mix annotation styles. Use callout circles for sequences, highlight boxes for single-element focus.

### Step 7: Produce Output Summary

Present the following to the user before they run the script:

```
─────────────────────────────────────────────
  Screenshot Transform Summary
─────────────────────────────────────────────
  Template:     [template]
  Crop region:  [x, y, width, height]
  Blur regions: [count] detected ([types])
  Annotations:  [count] callouts
  Output file:  [OUTPUT_DIR/FILENAME.webp]
  Est. size:    [rough KB estimate]
─────────────────────────────────────────────
  Next: Run the generated script, then commit:

  git add [OUTPUT_DIR/FILENAME.webp]
  git commit -m "docs(screenshot): add [FILENAME]"
─────────────────────────────────────────────
```

### Step 8: Commit Guidance

After the user confirms the output image looks correct, provide the exact git commands:

```bash
git add [OUTPUT_DIR/FILENAME.webp]
git commit -m "docs(screenshot): add [FILENAME]"
```

If the image is a replacement for an existing screenshot, use:
```bash
git commit -m "docs(screenshot): update [FILENAME]"
```

## Style Reference

All outputs must conform to these standards:

| Property | Value |
|---|---|
| Annotation color | `#FA4616` (UiPath Red) — only this color |
| Font | Inter Bold 13px for callouts |
| Highlight border | 2px solid `#FA4616` |
| Highlight fill | `rgba(250, 70, 22, 0.08)` |
| Blur style | Gaussian sigma 8, rounded rect mask 4px radius |
| Export format | WebP, 90% quality |
| Naming | `{product-area}-{action}-{step}.webp` |
| Resolution | 1× export (half of 2× capture) |

Do not use yellow, blue, or any color other than `#FA4616` for annotations. Do not use black rectangles for blur.

## Notes

- Full style guide: `docs/screenshot-standards/01-style-guide.md`
- Templates reference: `docs/screenshot-standards/02-templates.md`
- Example transformations: `docs/screenshot-standards/05-example-transformations.md`

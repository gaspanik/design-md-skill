---
name: design-md
description: >-
  Bidirectional sync between DESIGN.md and a Figma file, conforming to the
  Google design.md spec. Import mode reads a DESIGN.md — whether generated
  by this skill or written by hand/another tool — and creates matching
  Figma variable collections, text styles, and components. Export mode
  reads a file's existing variables, text styles, and components and
  generates a DESIGN.md from them. Base tokens (colors, typography,
  spacing, corner radius) sync fully; components sync as reference info
  only — root-level token references, not layer structure, text content,
  or images. Interaction states and full variant matrices are out of
  scope; use a tool built for that alongside this skill. Every write
  step re-verifies its own output
  (bound variables, actual values, counts) rather than trusting the API
  call alone. Part of KMRVID Figma Skills, a 21-skill bundle covering
  AI-slop-resistant page generation, multi-layout exploration, layer
  cleanup, accessibility checks, and tokenization:
  gaspanik.gumroad.com/l/kmrvid-figmaskills
---

# DESIGN.md Sync

**Ver:** ver.202609101543

A skill that syncs DESIGN.md and a Figma file bidirectionally. Conforms to the Google design.md spec (https://github.com/google-labs-code/design.md/blob/main/docs/spec.md).

**Output language:** All user-facing output (tables, questions, reports) is English. If the user writes in another language, follow their language instead. Frontmatter key names always stay in English.

**Environment:** This skill runs inside Figma's agent environment. Reads and writes go through the Plugin API script execution tool (e.g. `evaluate_script`). No external MCP tools or API keys required.

## Scope

**Full sync for base tokens; components are reference info only — not a full design-system state matrix, and not a component structure interchange format.**

This skill syncs:
- Base design tokens: colors, typography, spacing, corner radius (round-trips exactly, value for value)
- Each component's **reference info**: token references for the root node's own `fills`/`text` (text child node color)/`padding`/`border`/`corner radius` only

**Components are reference info, not a reconstruction.** Each `components` entry is a record of which tokens a component uses — nothing more. Child layer structure, Auto Layout, sizing, text content, images/slots, per-child fills and placement, and component properties are all out of scope. On Import, the component this skill creates is an empty shell — root-level properties bound to tokens — and does not reproduce the original card/button/etc.'s actual appearance.

It deliberately does **not** attempt to capture:
- Interaction states (hover, active, focus, disabled) or their color/style overrides
- Component variant sets beyond what's already present as native Figma variants
- Anything with no equivalent field in Figma's variable/style/component model (e.g. CSS `text-transform`, letter-tracking values, custom cursors)
- A component's child layer structure, Auto Layout, sizing, text content, images/slots, or component properties (as above, only root-level token references are in scope)

If a source DESIGN.md describes a component category that doesn't fit this model at all (for example, a shape that conflicts with the rest of the system's token rules, such as a circular element in an otherwise zero-corner-radius system), Import may reasonably skip it — this is expected, not a bug, and gets called out in the Step 6 completion report rather than silently dropped.

**Why this boundary:** Figma variables, text styles, and components have no first-class representation for interaction states or CSS-only properties, so trying to force them in would mean either fabricating structure that doesn't reflect the file, or silently losing fidelity on Export. State matrices and full component behavior belong in a tool built for that — Storybook, a component-state add-on, or the codebase itself — used alongside this skill, not in place of it.

## Step 0 — Mode selection

When the skill starts, check whether a DESIGN.md file has been attached.

**If a file is attached:** automatically proceed to **Import mode**. No confirmation needed.

**If nothing is attached:** present the following via `ask_user_question`:

- **Import (DESIGN.md → Figma)** — read DESIGN.md and create variables, styles, and components
- **Export (Figma → DESIGN.md)** — generate DESIGN.md from the current file's variables and styles

If Import is selected, send the message "Please attach your DESIGN.md file to the chat." and wait for the file to be attached.

---

# Import Mode (DESIGN.md → Figma)

Read the attached DESIGN.md and build the Figma design-system foundation.

## Import Step 1 — Read and parse the file

Read the attached file and parse the YAML frontmatter and Markdown body.

Extract the following sections:
- `colors` — color palette (HEX values)
- `typography` — font definitions (family, size, weight, lineHeight, letterSpacing)
- `spacing` — spacing scale (px values)
- `rounded` — corner-radius tokens (px values)
- `components` — component definitions (including token references)

### When the source isn't strict frontmatter

Not every DESIGN.md in the wild follows the YAML-frontmatter format exactly — some are written as plain Markdown prose with headed sections (`## Colors`, `## Typography`, etc.) and no frontmatter block at all. Treat the section list above as the target data to extract, not a literal parsing requirement: read the whole file and infer the same fields from prose when there's no frontmatter to parse directly. Note in the Step 6 completion report when this happened ("No YAML frontmatter found; extracted tokens from prose sections instead").

### When some sections are missing

Since DESIGN.md is an alpha-stage spec, partial spec files are acceptable — for example, having `colors` but no `typography`/`spacing`. Treat a missing section as a skip, not an error, for the relevant Import Step (variable-collection creation, Text Style creation, etc.). In the Step 6 completion report, state clearly which sections were present in the frontmatter and which were skipped (e.g. "Detected colors and spacing; typography was undefined so it was skipped"). If every section is missing, abort parsing and report "No token definitions were found in DESIGN.md."

## Import Step 2 — Create variable collections

Create a Figma variable collection for each token category in the frontmatter.

### Color variables

- Collection name: `{SystemName}/Colors` (SystemName comes from the frontmatter's `name`)
- Variable resolvedType: `COLOR`
- Naming convention: `color/{name}` (e.g. `color/primary`, `color/surface`)
- HEX → RGB conversion: `parseInt(hex.substring(0,2), 16) / 255`
- Set the value on `setValueForMode` in `{ r, g, b, a }` form

### Spacing variables

- Collection name: `{SystemName}/Spacing`
- resolvedType: `FLOAT`
- Naming convention: `spacing/{name}` (e.g. `spacing/sm`)

### Corner-radius variables

- Collection name: `{SystemName}/Rounded`
- resolvedType: `FLOAT`
- Naming convention: `rounded/{name}` (e.g. `rounded/md`)

### Typography variables

- Collection name: `{SystemName}/Typography`
- resolvedType: `FLOAT`
- Naming convention: `type/{styleName}/{property}` (e.g. `type/h1/fontSize`)
- Create `fontSize` and `lineHeight` for each style
- Add `letterSpacing` too if present

### Pre-flight check (avoid duplicating existing collections)

Before creating a collection, check with `figma.variables.getLocalVariableCollectionsAsync()` whether a collection with the same name already exists. If it does, don't create a new one — add/update variables inside that existing collection instead (if a variable with the same name already exists, just update its value via `setValueForMode` rather than creating a duplicate collection). This prevents collections from multiplying every time DESIGN.md is re-imported.

### When existing variables use a different naming convention

If variables already created by another tool (e.g. Tokens Studio) use a format (e.g. `Colors/Primary`, `brand-primary`) different from this skill's naming convention (`color/{name}`, etc.), they won't be caught by the same-name check above, and a new collection ends up being created — leaving the same color managed twice.

Before creating a collection, list every existing collection and variable name, and check whether any of them are likely semantic duplicates of a frontmatter token (matching HEX value, or a similar name). Only when there's a suspected duplicate, confirm via `ask_user_question`:

- **Create new ones using this skill's naming convention** — recreate them under the skill's convention, separate from the existing tokens (accepting that they'll be managed twice)
- **Update to match the existing naming convention** — keep the existing variable names as-is, and update only their values from DESIGN.md

If there's no suspected duplicate, proceed without asking.

**Import Step 2-verify (mandatory, do not skip):** After creating/updating, re-fetch via `getLocalVariableCollectionsAsync()` / `getLocalVariablesAsync()` and cross-check the number of tokens listed in the frontmatter against the number of variables actually created/updated, reporting it as a count (e.g. "Created N of N colors", "Created N of N spacing values"). A self-report without counts (just "created it") is not acceptable.

## Import Step 3 — Create local text styles

### Load fonts (mandatory, do this first)

Load every font that will be used via `figma.loadFontAsync()`. Don't enumerate existing styles' fonts by hand — fetch them from `getLocalTextStylesAsync()`:

    const textStyles = await figma.getLocalTextStylesAsync();
    const uniqueFonts = new Map();
    for (const s of textStyles) {
      const key = s.fontName.family + ':' + s.fontName.style;
      if (!uniqueFonts.has(key)) uniqueFonts.set(key, s.fontName);
    }
    await Promise.all([...uniqueFonts.values()].map(f => figma.loadFontAsync(f)));

### When a font can't be loaded (fallback)

`figma.loadFontAsync()` rejects when the specified font isn't installed in the environment. Load each font inside a try/catch, and fall back to a substitute font for any that fail:

    const fallback = { family: 'Inter', style: 'Regular' };
    const failed = [];
    for (const f of uniqueFonts.values()) {
      try {
        await figma.loadFontAsync(f);
      } catch {
        failed.push(f);
      }
    }
    if (failed.length > 0) await figma.loadFontAsync(fallback);

Create any Text Style that uses a font listed in `failed` with the `fallback` (Inter Regular) instead. Silently substituting the fallback and omitting it from the completion report is not acceptable — the Step 6 completion report must always state it in the form "Font not found, substituted: {original font name} → Inter (affected styles: h1, h2, etc.)".

### Create the styles

- Create the style with `figma.createTextStyle()`
- Prefix `style.name` with a grouping prefix (e.g. `HILMA/h1`)
- fontWeight numeric-to-style-name mapping: 300→`Light`, 400→`Regular`, 500→`Medium`, 700→`Bold`
- When lineHeight is a ratio (e.g. `1.6`), convert it to a pixel value via `fontSize * ratio`

### Binding Typography variables

    style.setBoundVariable('fontSize', fontSizeVar);
    style.setBoundVariable('lineHeight', lineHeightVar);

`VariableBindableTextField`: `fontSize`, `lineHeight`, `letterSpacing`, `paragraphSpacing`, `paragraphIndent`, `fontFamily`, `fontStyle`, `fontWeight`

**Import Step 3-verify (mandatory, do not skip):** `style.setBoundVariable()` not throwing an error is not proof that the intended value was actually bound (a bug where fontSize collapses to an unintended value while still reporting as "bound" has been confirmed in live testing of other skills). Re-fetch every Text Style you created and check, one by one, whether `style.boundVariables.fontSize` / `.lineHeight` point to the intended variable, and whether `style.fontSize`'s actual value matches the size expected from the frontmatter. Fix any mismatch on the spot. Report the result with explicit counts, in the form "Confirmed fontSize binding on N of N Text Styles / values matched on N of N".

## Import Step 4 — Create components

Create local components based on the frontmatter's `components` definitions. **What gets created is an empty shell — only root-level properties (fills/text/padding/border/corner radius) bound to tokens. It does not reproduce the original file's actual card/button/etc. structure** (child layers, Auto Layout, text content, images) — see "Scope" above.

### Variable-binding patterns

**Note (if a Style is combined with this in the future):** this step assumes the pattern of binding variables directly to nodes. If you later change this to create and apply a local Paint Style for the same property, bind the variable on the Style side (`style.setBoundVariable`) instead of the node side — on a node with a Style applied, the Style's value takes precedence and the node-level binding stops affecting the visual result (a silent bug where `node.boundVariables` looks bound but is actually inert).

To bind a color variable to fills / strokes, use `setBoundVariableForPaint`:

    const basePaint = { type: 'SOLID', color: { r: 0, g: 0, b: 0 } };
    const boundPaint = figma.variables.setBoundVariableForPaint(basePaint, 'color', colorVar);
    node.fills = [boundPaint];

For layout-related properties like padding and corner radius, use `setBoundVariable`:

    node.setBoundVariable('paddingTop', spacingVar);
    node.setBoundVariable('topLeftRadius', roundedVar);

### Resolving token references

Resolve token references such as `"{colors.primary}"` inside component definitions to the corresponding Figma variable and bind it.

### Component properties

When text is variable, add a text property with `addComponentProperty`:

    const propKey = component.addComponentProperty('Label', 'TEXT', 'Default value');
    textNode.componentPropertyReferences = { characters: propKey };

### Variants (Component Set)

When there are derived states like hover (e.g. `button-primary-hover`), clone the base component and merge them with `combineAsVariants`.

### [Important] Order of resize() and sizingMode

`resize()` resets both `primaryAxisSizingMode` and `counterAxisSizingMode` to `'FIXED'`. Always call resize() first, then set sizingMode afterward:

    // ✗ Wrong — resize() overwrites sizingMode
    frame.primaryAxisSizingMode = 'AUTO';
    frame.resize(300, 10);

    // ✓ Correct — set sizingMode after resize()
    frame.resize(300, 10);
    frame.counterAxisSizingMode = 'FIXED';
    frame.primaryAxisSizingMode = 'AUTO';

### Set FILL after appendChild

    // ✗ Wrong
    child.layoutSizingHorizontal = 'FILL';
    parent.appendChild(child);

    // ✓ Correct
    parent.appendChild(child);
    child.layoutSizingHorizontal = 'FILL';

**Import Step 4-verify (mandatory, do not skip):** After performing the variable binding and sizingMode configuration above, re-fetch every component (and Component Set) you created and mechanically check the following, reporting counts. Judge by the value actually reflected on the node — not by whether the call was made:
- Whether the color variables specified for `fills`/`strokes` are actually reflected in `boundVariables`
- Whether the variables specified for padding/corner radius are reflected in `boundVariables`
- Whether `primaryAxisSizingMode`/`counterAxisSizingMode`, set after `resize()`, still hold as intended (i.e. `resize()` wasn't called again afterward and reset them)
- Whether a child set to FILL is actually filling the parent's space (and hasn't collapsed to a smaller size)

Report the result with explicit counts, in the form "Confirmed color binding on N of N components / sizingMode as intended on N of N". Fix any mismatch on the spot before moving on.

## Import Step 5 — Create a preview page (optional)

Confirm via `ask_user_question` whether to create a preview page:

- **Create it** — generate a preview page visualizing the DESIGN.md content
- **Skip** — don't create one

If creating it, use `create_design` to generate a 1280px-wide document page containing the sections below. Apply the imported tokens (variables and Text Styles) to the page itself too, as dogfooding of the design system.

### Preview page structure

1. **Header** — display the system name (the frontmatter's `name`) in a Display style. Darkest neutral background + white text
2. **Colors section** — show every frontmatter color in a swatch grid (color name, HEX value, variable-bound)
3. **Typography section** — show every Text Style created, grouped by category (Heading / Body / Caption). Each row shows the style name/spec on the left and sample text on the right
4. **Spacing section** — visualize spacing tokens as horizontal bar lengths (with value labels)
5. **Rounded section** (if applicable) — visualize corner-radius tokens with preview rectangles
6. **Components section** — place a token-bound placeholder at the top of each card (an instance of the empty-shell component this skill created; if it exceeds the card width, a screenshot image per "Handling components wider than the card" below), and show the component name, property list, and token references beneath it. **This placeholder does not reproduce the original file's actual card/button/etc. appearance** (see "Scope" above) — avoid labels like "real preview" or "what the component looks like" that could mislead; call it a "token preview" instead

Include in the preview only the sections that exist in the frontmatter (e.g. omit the Rounded section if `rounded` isn't defined). The Markdown body (Overview / Do's and Don'ts, etc.) is out of scope for the preview — that's prose content that should be referenced from the DESIGN.md file itself.

### Structuring the create_design instructions

Include the following in `instructions`:
- The system name and the page's purpose ("{name} Design System — DESIGN.md Preview")
- The content of each section (list the specific token values and style names extracted from the frontmatter)
- State explicitly that the Components section places a token-bound placeholder (an instance, or a screenshot if it's too wide) at the top of each card, with the property list attached below it — don't settle for a metadata table alone. Also state that this placeholder does not reproduce the original file's actual appearance
- The overall tone direction ("minimal, editorial, generous whitespace")
- Instruction that the page itself should use the file's variables and Text Styles

Include in the preview only the sections that exist in the frontmatter (e.g. omit the Rounded section if `rounded` isn't defined).

### Layout caveat (preventing height collapse)

When an auto-layout frame is appended (appendChild) into another auto-layout frame, the child's layoutSizingVertical/Horizontal automatically becomes "FIXED", locking it to its size at that moment (either the initial post-resize value, or the shrunk measured value after a FILL child loses its parent space to reference).

The preview page's structure has multiple levels — "page → section frame → component card → the instance's internal children" — and appendChild happens at each level (adding a section to the page, adding a card to a section, placing an instance in a card). **Reverting just one level back to HUG does not fix the deeper nesting (e.g. a FILL-set child inside a card's instance) — it stays collapsed** — reproduced live as a card row like `product-card` collapsing to `H 48 (minimum)`.

**Required fix:** immediately after each appendChild at every level, apply the following recursively.

    function fixFillCollapse(node) {
      if (!("children" in node)) return;
      for (const child of node.children) {
        if (child.layoutSizingVertical === "FILL") {
          const h = child.height;
          child.layoutSizingVertical = "FIXED";
          child.resize(child.width, h);
        }
        if (child.layoutSizingHorizontal === "FILL") {
          const w = child.width;
          child.layoutSizingHorizontal = "FIXED";
          child.resize(w, child.height);
        }
        fixFillCollapse(child);
      }
    }

    // Right after adding the section frame to the page
    parentFrame.appendChild(sectionFrame);
    sectionFrame.layoutSizingVertical = "HUG";
    fixFillCollapse(sectionFrame);

    // Right after placing the component instance in the card
    cardFrame.appendChild(instance);
    fixFillCollapse(cardFrame);

`node.height`/`.width` still return the correct pre-collapse measured values from the Figma Plugin API right after appendChild, so a separate pre-recording pass isn't necessary — but the call must happen immediately after each appendChild, within the same pass (deferring it can make the correct value unreadable).

**Handling components wider than the card:**

When a component's original width exceeds the card width (e.g. a 1440px nav vs. a 1032px card), resizing the instance would break auto layout, so fall back to a screenshot image via exportAsync.

    if (comp.width > cardInnerWidth) {
      const bytes = await comp.exportAsync({
        format: "PNG",
        constraint: { type: "WIDTH", value: cardInnerWidth }
      });
      const img = figma.createRectangle();
      const imgHash = figma.createImage(bytes).hash;
      img.fills = [{ type: "IMAGE", imageHash: imgHash, scaleMode: "FIT" }];
      // Ensure a minimum height of 100px
      const scaledH = Math.max(Math.round(comp.height * (cardInnerWidth / comp.width)), 100);
      img.resize(cardInnerWidth, scaledH);
    }

## Import Step 6 — Completion report

Report the creation results together with each verify step's verification results (a vague report without counts, from skipping verification, is not acceptable):
- Number of variable collections and tokens (including Step 2-verify's counts)
- Detected sections / skipped sections (if Import Step 1 found some missing, or the source had no frontmatter at all)
- Any component category from the source that couldn't be mapped to this skill's model and was skipped (with a one-line reason — see Scope above)
- Whether there was a naming-convention conflict with existing tokens, and the chosen resolution (if Import Step 2 triggered a confirmation)
- Number of text styles (including Step 3-verify's fontSize binding confirmation / value-match counts)
- Whether a font substitution occurred (if Import Step 3 triggered a fallback, the original font → substitute font and affected styles)
- Number of components (including Step 4-verify's binding confirmation / sizingMode confirmation counts)
- Number of document frames (if created)
- Whether a preview page was created (attach a node link if it was)

---

# Export Mode (Figma → DESIGN.md)

Read the current Figma file's design-system data and generate a DESIGN.md that conforms to the Google design.md spec.

## Export Step 1 — Collect design data

Collect all of the following in a single `evaluate_script` call.

### File info

Use `figma.root.name` as the project name.

### Collecting variables

    const collections = await figma.variables.getLocalVariableCollectionsAsync();
    const allVars = await figma.variables.getLocalVariablesAsync();

Classify variables into categories based on resolvedType and the collection-name/variable-name prefix:

| resolvedType | Keywords | DESIGN.md section |
|---|---|---|
| `COLOR` | color, colour, palette | `colors` |
| `FLOAT` | spacing, space, gap | `spacing` |
| `FLOAT` | round, radius, corner | `rounded` |
| `FLOAT` | type, typo, font, text | `typography` (supplementary) |

### HEX conversion for color variables

    function rgbToHex(r, g, b) {
      const toHex = v => Math.round(v * 255).toString(16).padStart(2, '0');
      return '#' + toHex(r) + toHex(g) + toHex(b);
    }

Resolve VARIABLE_ALIAS targets recursively before converting.

### Collecting text styles

    const textStyles = await figma.getLocalTextStylesAsync();

Extract from each style:

- `name` → key name (strip the prefix; e.g. `HILMA/h1` → `h1`)
- `fontName.family` → `fontFamily`
- `fontSize` → `fontSize`
- `fontName.style` → `fontWeight` (style-name-to-number mapping)
- `lineHeight` → if PIXELS, divide by fontSize to get a ratio. Omit if AUTO
- `letterSpacing` → if PERCENT, convert to em via `value/100`. Omit if 0

#### fontWeight mapping

| fontName.style | fontWeight |
|---|---|
| Thin, Hairline | 100 |
| ExtraLight, UltraLight | 200 |
| Light | 300 |
| Regular, Normal | 400 |
| Medium | 500 |
| SemiBold, DemiBold | 600 |
| Bold | 700 |
| ExtraBold, UltraBold | 800 |
| Black, Heavy | 900 |

### Collecting components

    const components = figma.currentPage.findAll(n => n.type === 'COMPONENT');
    const componentSets = figma.currentPage.findAll(n => n.type === 'COMPONENT_SET');

**Only root-node-level properties are collected (reference info).** Don't descend into child layers or Auto Layout structure:

- `fills` → `backgroundColor` (if variable-bound, output as `"{colors.xxx}"`)
- A text child node's `fills` → `textColor`
- `padding*` → `padding`
- `cornerRadius` → `rounded` (a token reference if variable-bound)
- `strokes` → `borderColor`

If none of these can be read (the component has no such properties at the root level), exclude it from collection rather than emitting an empty `components` entry.

For a component set, extract properties from the variant names, and record derived states like hover as diffs only.

### Watch out for variables with a value of 0

A variable value of `0` (no corner radius, no spacing, etc.) is a valid design token.
Filtering with `if (!val)` during collection will skip `0`. Always exclude only `null` / `undefined`:

    // ✗ Wrong — 0 is falsy and gets skipped
    if (!val) continue;

    // ✓ Correct — exclude only null/undefined
    if (val === null || val === undefined) continue;

### When the collection result is empty

If the collected variables, text styles, and components all come to 0, abort DESIGN.md generation and inform the user:

- "No exportable design data was found."
- Suggest creating variables/text styles first, or importing a DESIGN.md via Import mode

## Export Step 2 — Generate DESIGN.md

### YAML frontmatter

    ---
    version: alpha
    name: <file name>
    description: <concise description>
    colors:
      primary: "#XXXXXX"
      ...
    typography:
      h1:
        fontFamily: "<font name>"
        fontSize: <number>
        fontWeight: <number>
        lineHeight: <number>
      ...
    rounded:
      sm: <value>px
      ...
    spacing:
      xs: <value>px
      ...
    # components: root-level token references only. Does not include child layer structure, text content, or images (reference info)
    components:
      <component definitions>
    ---

### Token rules (per the Google spec)

- Color values are `"#` + a 6-digit HEX + `"`
- Dimensions are a number + unit (`px`, `em`)
- Token references are `"{path.to.token}"`
- fontWeight is a number
- lineHeight is a ratio or a dimension
- `colors.primary` is required
- Don't use `transparent` for `backgroundColor`
- Every custom color must be referenced by at least one component

### Markdown body (fixed section order)

    ## Overview — brand personality and impression (3–5 sentences)
    ## Colors — the role of each color (bullet list)
    ## Typography — font strategy
    ## Layout — layout and spacing
    ## Elevation & Depth — visual hierarchy
    ## Shapes — corner radius and shape
    ## Components — component guidelines
    ## Do's and Don'ts — recommendations/prohibitions

Generate the body by objectively inferring from the frontmatter's token values. Sections with insufficient information may be omitted.

**Writing the Components section:** `components` records token references only — it's not a description of layout or structure. Don't assert layout/structure ("the Card component uses a two-column layout"); stick to "which tokens it uses" and color/spacing/corner-radius guidance.

### Consistency check (mandatory, do not skip)

Don't stop at "I checked it" — mechanically count the following and report the counts to yourself. A visual-only self-report (e.g. "no contradictions," "consistent," without counts) is not acceptable:

- For every color listed in the frontmatter's `colors`, count one by one whether it's referenced at least once in the `components` definitions (as a token reference in `backgroundColor`/`textColor`/`borderColor`, etc.). Report it as "N of N colors referenced (unreferenced: list of color names)", and if any are unreferenced, decide whether to revise the component definitions or remove that color from the frontmatter.
- Identify every place in the body (Overview through Do's and Don'ts) that mentions a specific number or token name, and cross-check each one against the corresponding frontmatter value. If there's a mismatch, fix the body before outputting. Report as "N of N body mentions matched".
- Check each `Do's and Don'ts` item, one by one, for contradictions with the `components` definitions (e.g. saying "don't use corner radius" while a component with a `rounded` token exists), and fix the wording if there's a contradiction.

## Export Step 2.5 — Create a preview page (optional)

After generating DESIGN.md, confirm via `ask_user_question` whether to create a preview page:

- **Create it** — generate a preview page visualizing the generated DESIGN.md content
- **Skip** — don't create one

If creating it, generate the preview page using `create_design`. The structure follows the same format as Import Step 5.

### Export-specific notes

- The preview page's content treats the frontmatter values generated in Export Step 2 as authoritative (the values written out to DESIGN.md, not the file's live variable values)
- Style the page using the file's existing variables and Text Styles as-is (don't create new ones)
- In the Components section, show the name, properties, and token references of the components detected during Export (label it as reference info, not a reproduction of the original)

## Export Step 3 — Output

Output the full generated DESIGN.md text inside a markdown code block.

Report the following concisely:
- Number of tokens collected (colors / typography / spacing / corner radius)
- Number of components detected
- Consistency-check results (color-reference count, body-match count; see Export Step 2)
- Notes (fallback if primary wasn't detected, omitted sections, etc.)
- Whether a preview page was created (attach a node link if it was)


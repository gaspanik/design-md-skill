---
name: design-md
description: >-
  Bidirectional sync between DESIGN.md and a Figma file, conforming to the
  Google design.md spec. Import mode reads a DESIGN.md — whether generated
  by this skill or written by hand/another tool — and creates matching
  Figma variable collections and text styles; components stay as
  reference info (root-level token references) in the frontmatter
  data, never as fabricated Figma nodes. Export mode reads a file's
  existing variables, text styles, and components and generates a
  DESIGN.md from them. Base tokens sync fully; components sync as
  reference info only. Interaction states and full variant matrices
  are out of scope; use a tool built for that alongside this skill.
  Every write step re-verifies its own output
  (bound variables, actual values, counts) rather than trusting the API
  call alone. Part of KMRVID Figma Skills, a 21-skill bundle covering
  AI-slop-resistant page generation, multi-layout exploration, layer
  cleanup, accessibility checks, and tokenization:
  gaspanik.gumroad.com/l/kmrvid-figmaskills
---

# DESIGN.md Sync

**Ver:** ver.202609101723

A skill that syncs DESIGN.md and a Figma file bidirectionally. Conforms to the Google design.md spec (https://github.com/google-labs-code/design.md/blob/main/docs/spec.md).

**Output language:** All user-facing output (tables, questions, reports) is English. If the user writes in another language, follow their language instead. Frontmatter key names always stay in English.

**Environment:** This skill runs inside Figma's agent environment. Reads and writes go through the Plugin API script execution tool (e.g. `evaluate_script`). No external MCP tools or API keys required.

## Scope

**Full sync for base tokens; components are reference info only — not a full design-system state matrix, and not a component structure interchange format.**

This skill syncs:
- Base design tokens: colors, typography, spacing, corner radius (round-trips exactly, value for value)
- Each component's **reference info**: only these 7 fields — `fills` (background), `strokes` (border), `padding`, `corner radius`, `width`, `height` (the component's own root node properties), and **one direct text child node's `fills`** (as `textColor` — the sole "read a child" exception)

**`width`/`height` are collected only when variable-bound (never a raw measured pixel value).** The Google design.md spec explicitly allows `size`/`width`/`height` as component token properties, and simple components like buttons, chips, and pills often do have their size tokenized. But writing an unbound size value would be fabrication just like everything else here, so only emit `width`/`height` when `node.boundVariables.width`/`.height` exists — otherwise omit those keys entirely for that component (the other 5 fields still apply normally).

**If even one of these 7 fields is readable, include the component.** `textColor` is a first-class collection target on the same footing as the root-level properties — don't treat it as "a child layer" and exclude it. Many components have no fill of their own and are defined entirely by their text color; excluding `textColor` drops essentially every text-driven component (cards, rows, list items) from collection entirely.

**Components are reference info, not a reconstruction.** Each `components` entry is a record of which tokens a component uses — nothing more. Beyond the 7 fields above — child layer structure, Auto Layout, the text content itself, images/slots, fills/placement of any child other than the one text node, and component properties — are all out of scope.

**Import does not create components as Figma nodes.** Even when width/height are present, child layer structure, the text content itself, and images are still out of scope, so building a node still only produces an empty shell, not the original card/button's actual appearance. And most complex components (e.g. product cards) won't have width/height bound to a variable at all, so they'd just be same-size empty boxes with no way to tell them apart. So `components` data lives only in the DESIGN.md frontmatter; if Import builds a preview page, components appear there as reference-info text (name + property list + token references), never as a Figma node with real substance.

It deliberately does **not** attempt to capture:
- Interaction states (hover, active, focus, disabled) or their color/style overrides
- Component variant sets beyond what's already present as native Figma variants
- Anything with no equivalent field in Figma's variable/style/component model (e.g. CSS `text-transform`, letter-tracking values, custom cursors)
- A component's child layer structure, Auto Layout, sizing, the text content itself, images/slots, or component properties (the direct text child's `fills` — i.e. `textColor` — is the one exception; no other child is in scope)

If a source DESIGN.md describes a component category that doesn't fit this model at all (for example, a shape that conflicts with the rest of the system's token rules, such as a circular element in an otherwise zero-corner-radius system), Import may reasonably skip it — this is expected, not a bug, and gets called out in the Step 6 completion report rather than silently dropped.

**Why this boundary:** Figma variables, text styles, and components have no first-class representation for interaction states or CSS-only properties, so trying to force them in would mean either fabricating structure that doesn't reflect the file, or silently losing fidelity on Export. State matrices and full component behavior belong in a tool built for that — Storybook, a component-state add-on, or the codebase itself — used alongside this skill, not in place of it.

## Step 0 — Mode selection

When the skill starts, check whether a DESIGN.md file has been attached.

**If a file is attached:** automatically proceed to **Import mode**. No confirmation needed.

**If nothing is attached:** present the following via `ask_user_question`:

- **Import (DESIGN.md → Figma)** — read DESIGN.md and create variables and styles (components stay as reference info, not Figma nodes — see Scope)
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

## Import Step 4 — Review component reference info

Read the frontmatter's `components` definitions and take note of them. **Do not create any component nodes in Figma** (see "Scope" above). Why: `components` only records fills/textColor/padding/border/cornerRadius/width/height (width/height only when variable-bound), never children. Most complex components (e.g. product cards) won't have width/height bound at all, so building a node from just this data produces identically-sized empty boxes for components that are actually quite different -- and even for the simple atoms that do have width/height, child layer structure and text content are still out of scope, so the node would still be an empty shell.

This step only does two things:
- Confirm that each component definition's token references (e.g. `"{colors.primary}"`) resolve correctly to the variables created in Step 2 (call out any reference that doesn't resolve in the Step 6 completion report)
- Hold on to the component names and property lists for use in Step 5 (preview page, if created) and Step 6 (completion report)

## Import Step 5 — Create a preview page (optional)

Confirm via `ask_user_question` whether to create a preview page:

- **Create it** — generate a preview page visualizing the DESIGN.md content
- **Skip** — don't create one

If creating it, use `create_design` to generate a 1280px-wide document page containing the sections below. Apply the imported tokens (variables and Text Styles) to the page itself too, as dogfooding of the design system.

### Preview page structure

1. **Header** — display the system name (the frontmatter's `name`) in a Display style. Darkest neutral background + white text
2. **Colors section** — show every frontmatter color in a swatch grid (color name, HEX value). **Each swatch's fill must be bound to the actual corresponding Figma variable (`setBoundVariableForPaint`).** Don't bind to a different variable just because it looks similar or shares the same HEX (e.g. two colors that are the same `#1a1a1a` but differ only in opacity) — match by variable name/id, not by visual similarity. After binding, verify each swatch's `boundVariables` points to the intended variable and its opacity matches that variable's value; report as "Confirmed variable binding on N of N colors"
3. **Typography section** — show every Text Style created, grouped by category (Heading / Body / Caption). Each row shows the style name/spec on the left and sample text on the right. **The sample text must have the actual Text Style applied via `textNode.textStyleId = style.id`.** Don't approximate the font/weight/size independently by eye (`create_design` has a known bug of substituting an unrelated "similar-looking" font). After applying, verify each sample text's `textStyleId` points to the intended Text Style; report as "Confirmed application on N of N text styles"
4. **Spacing section** — visualize spacing tokens as horizontal bar lengths (with value labels)
5. **Rounded section** (if applicable) — visualize corner-radius tokens with preview rectangles
6. **Components section (text only in Import mode)** — Import Step 4 doesn't create any Figma node, so place no visual placeholder or instance. For each component, lay out its name, property list (only the keys that actually exist among `backgroundColor`/`textColor`/`padding`/`border`/`rounded`/`width`/`height`), and token references as a text-only card. **Only list properties that actually exist in that component's `components` definition.** Don't add `spacing` (gap) or anything else not in the `components` definition — writing information that isn't there makes it look like DESIGN.md captured something it didn't. (Export mode's preview differs from this — it may place a real instance of the actual component, since it exists in the live file; see Export Step 2.5)

Include in the preview only the sections that exist in the frontmatter (e.g. omit the Rounded section if `rounded` isn't defined). The Markdown body (Overview / Do's and Don'ts, etc.) is out of scope for the preview — that's prose content that should be referenced from the DESIGN.md file itself.

### Structuring the create_design instructions

Include the following in `instructions`:
- The system name and the page's purpose ("{name} Design System — DESIGN.md Preview")
- The content of each section (list the specific token values and style names extracted from the frontmatter)
- State explicitly that the Components section is text-only — no instance, placeholder rectangle, or other visual element; just each component's name, property list, and token references laid out as text cards
- State explicitly that the property list must only include keys that actually exist in that component's `components` definition. For a component whose definition has no `width`/`height`, explicitly prohibit reading them (or `spacing`/gap) from the live Figma file and adding them to the list — `create_design` has access to the live file, so without this constraint it will fill gaps on its own
- **State explicitly that each Colors swatch must be bound to its actual Figma variable (not chosen by visual similarity), and that opacity must match that variable's value** — two colors sharing the same HEX but different opacity must not be conflated
- **State explicitly that each Typography sample text must have the actual created Text Style applied, not an independently-chosen approximate font/weight/size**
- The overall tone direction ("minimal, editorial, generous whitespace")
- Instruction that the page itself should use the file's variables and Text Styles

Include in the preview only the sections that exist in the frontmatter (e.g. omit the Rounded section if `rounded` isn't defined).

### Layout caveat (preventing height collapse)

When an auto-layout frame is appended (appendChild) into another auto-layout frame, the child's layoutSizingVertical/Horizontal automatically becomes "FIXED", locking it to its size at that moment (either the initial post-resize value, or the shrunk measured value after a FILL child loses its parent space to reference).

The preview page's structure has multiple levels — "page → section frame → card → the card's text elements" — and appendChild happens at each level (adding a section to the page, adding a card to a section, placing a text element in a card). **Reverting just one level back to HUG does not fix the deeper nesting — it stays collapsed** — reproduced live as a card row collapsing to its minimum size.

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

    // Right after adding a card to the Components section
    sectionFrame.appendChild(cardFrame);
    fixFillCollapse(sectionFrame);

`node.height`/`.width` still return the correct pre-collapse measured values from the Figma Plugin API right after appendChild, so a separate pre-recording pass isn't necessary — but the call must happen immediately after each appendChild, within the same pass (deferring it can make the correct value unreadable).

## Import Step 6 — Completion report

Report the creation results together with each verify step's verification results (a vague report without counts, from skipping verification, is not acceptable):
- Number of variable collections and tokens (including Step 2-verify's counts)
- Detected sections / skipped sections (if Import Step 1 found some missing, or the source had no frontmatter at all)
- Any component category from the source that couldn't be mapped to this skill's model and was skipped (with a one-line reason — see Scope above)
- Whether there was a naming-convention conflict with existing tokens, and the chosen resolution (if Import Step 2 triggered a confirmation)
- Number of text styles (including Step 3-verify's fontSize binding confirmation / value-match counts)
- Whether a font substitution occurred (if Import Step 3 triggered a fallback, the original font → substitute font and affected styles)
- Number of component reference-info entries confirmed in Step 4 (state explicitly that they were not created as Figma nodes)
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

**Only the following 7 fields are collected (reference info).** Don't descend into any other child layer or Auto Layout structure:

- `fills` → `backgroundColor` (if variable-bound, output as `"{colors.xxx}"`)
- **One direct text child node's `fills` → `textColor`** (the sole "read a child" exception — don't exclude this as "a child layer"; many components have no fill of their own and are defined entirely by text color, so excluding `textColor` causes mass collection drop-out)
- `padding*` → `padding`
- `cornerRadius` → `rounded` (a token reference if variable-bound)
- `strokes` → `borderColor`
- `boundVariables.width` if present → `width` (as a token reference; if not bound, omit the `width` key entirely for that component — never write a raw measured pixel value)
- `boundVariables.height` if present → `height` (same rule)

The Google design.md spec explicitly allows `size`/`width`/`height` as component tokens, and simple components (buttons, chips, pills) often have their size tokenized. Complex composite components (product cards, rows) usually won't — that's expected, not a bug; just omit `width`/`height` for those.

**Include the component if even one of these 7 fields is readable.** Only exclude it from collection (rather than emitting an empty `components` entry) if *all seven* are unreadable — no fills/strokes/padding/cornerRadius/width/height on the root AND no direct text child. Before excluding, check each of the 7 individually — don't mistake "only `textColor` was found" for "nothing was found."

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
- `colors.primary` is required. If a variable in the file already resolves to `primary` after stripping its prefix (e.g. `brand/primary` → `primary`), use it as-is. **If none exists, don't substitute a different color as `primary` to fill the gap** — instead, report in the Step 6 completion report that no `primary`-named variable was found, and either leave `colors.primary` unset or ask the user which color to assign
- Don't use `transparent` for `backgroundColor`
- **A color variable's output key is exactly its Figma name with the collection/group prefix stripped, nothing else.** Never change a color's key, or substitute it for another color's key, based on whether it's referenced from `components`

**Colors always sync in full** (see "Scope" above). A color variable that no `components` entry happens to reference is still kept in the frontmatter's `colors` — never dropped or renamed. The only case where a color is removed or renamed is when it no longer exists as a Figma variable. "Unreferenced" is reported as a count in the consistency check below, nothing more.

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

- For every color listed in the frontmatter's `colors`, count one by one whether it's referenced at least once in the `components` definitions (as a token reference in `backgroundColor`/`textColor`/`borderColor`, etc.). Report it as "N of N colors referenced (unreferenced: list of color names)". **Do not remove unreferenced colors from the frontmatter** — colors sync in full regardless of whether a component happens to reference them (see "Token rules" above). Report the count only.
- Identify every place in the body (Overview through Do's and Don'ts) that mentions a specific number or token name, and cross-check each one against the corresponding frontmatter value. If there's a mismatch, fix the body before outputting. Report as "N of N body mentions matched".
- Check each `Do's and Don'ts` item, one by one, for contradictions with the `components` definitions (e.g. saying "don't use corner radius" while a component with a `rounded` token exists), and fix the wording if there's a contradiction.

## Export Step 2.5 — Create a preview page (optional)

After generating DESIGN.md, confirm via `ask_user_question` whether to create a preview page:

- **Create it** — generate a preview page visualizing the generated DESIGN.md content
- **Skip** — don't create one

If creating it, generate the preview page using `create_design`. The Colors/Typography/Spacing/Rounded sections follow the same format as Import Step 5. **The Components section differs** (see below) — Import is text-only since it creates no nodes, but Export's target components genuinely exist in the live file, so a real instance is fine to show.

### Export-specific notes

- The preview page's content treats the frontmatter values generated in Export Step 2 as authoritative (the values written out to DESIGN.md, not the file's live variable values)
- Style the page using the file's existing variables and Text Styles as-is (don't create new ones)
- **The Components section may place an instance of the real, exported component at the top of each card** (screenshot fallback via exportAsync if it's wider than the card). This is simply showing what genuinely exists in the file — not fabrication. The property list and token references beneath it should still be labeled as reference info, understood as a restatement of what's written to DESIGN.md
- **The properties table may only list keys that actually appear in the generated DESIGN.md's `components` entry.** The original component being exported is real and lives in the live file, so `spacing` (gap) is always readable, and `width`/`height` are readable even when unbound — but any key not actually in the `components` definition (including an unbound width/height) stays out of the preview too. Adding a value to the preview alone that isn't in DESIGN.md implies DESIGN.md captured information it didn't

## Export Step 3 — Output

Output the full generated DESIGN.md text inside a markdown code block.

Report the following concisely:
- Number of tokens collected (colors / typography / spacing / corner radius)
- Number of components detected
- Consistency-check results (color-reference count, body-match count; see Export Step 2)
- Notes (fallback if primary wasn't detected, omitted sections, etc.)
- Whether a preview page was created (attach a node link if it was)


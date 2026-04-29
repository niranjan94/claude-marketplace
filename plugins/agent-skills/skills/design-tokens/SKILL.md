---
name: design-tokens
description: >
  Create or update a DESIGN.md design-system document (machine-readable YAML token front matter plus
  human-readable markdown rationale) so that design agents and tools share a single source of truth for
  colors, typography, spacing, shapes, and components. Invoke this skill whenever the user wants to
  generate, scaffold, refine, audit, validate, or update a DESIGN.md file; whenever they describe a
  visual identity, brand, palette, or design system in natural language ("a playful coffee app with warm
  colors", "make it feel like Stripe", "extract a design system from this URL"); whenever they paste a
  brand URL, screenshot, Figma reference, brand PDF, or color palette and ask for a structured design
  system; or whenever they mention "design tokens", "design system", "DESIGN.md", "Stitch", "tokens.json",
  "Tailwind theme", "Figma variables", "style guide", or "brand guidelines" -- even if they don't use
  those exact words. Also use when an existing DESIGN.md needs to be edited, extended with new tokens or
  components, validated against the spec, or migrated from an ad-hoc style guide.
---

# DESIGN.md Design Tokens

Create or update a `DESIGN.md` file -- a plain-text design system document with two layers:

1. **YAML front matter** at the top, holding machine-readable design tokens (exact hex values, font properties, spacing scales, component definitions). Agents read this for precise enforcement.
2. **Markdown body** below, holding human-readable rationale organized into `##` sections. Humans (and agents) read this for the *why*.

Think of `DESIGN.md` as the design counterpart to `AGENTS.md` or `README.md`: a living, version-controlled artifact that any design or coding agent can consume to keep the visual identity consistent across screens.

## Why this matters

Without a `DESIGN.md`, every screen a design agent generates stands alone. Colors drift, spacing flexes, button radii pick themselves. With one, the agent has a single source of truth: the YAML gives precise values, the prose gives intent, and the file lives in the repo where humans and agents can both edit it.

The format is a **foundation, not a prescription**. Unknown sections and custom token names are accepted, not rejected -- the spec gives a shared vocabulary while leaving room for domain-specific extensions.

## Workflow

### Step 1: Determine the task

Figure out which of these the user is asking for, and ask if it's genuinely ambiguous:

| Path | Trigger | What to do |
|---|---|---|
| **Generate from a vibe** | User describes the aesthetic in natural language ("playful coffee app with warm colors") | Translate intent into concrete tokens. Pick a coherent palette, type system, spacing scale, and component patterns that match the description. |
| **Derive from a brand source** | User provides a URL, screenshot, image, brand PDF, Figma file, or existing site to imitate | Extract the actual palette, typography, radii, and component patterns. Use available tools (web fetch for URLs, image inspection for screenshots, file reading for PDFs) before guessing. |
| **Author by hand** | User wants to dictate exact values | Take the user's stated values and structure them into a spec-conformant file. Don't invent values they didn't ask for; do flag missing required pieces. |
| **Update an existing DESIGN.md** | A `DESIGN.md` already exists and the user wants to edit, extend, or fix it | Read it first. Preserve unknown sections and custom tokens (they are valid per the spec). Make the smallest change that satisfies the request. |
| **Validate / audit** | User asks "is my DESIGN.md correct?" or "does this match the spec?" | Run the validation checks in Step 6 and report findings. Don't auto-fix unless asked. |

If existing brand or design context is available in the repo (a `README.md`, screenshots, a `tailwind.config.js`, a `tokens.json`, CSS variables, etc.), read those before generating tokens from scratch -- they almost always carry the design intent already and prevent you from inventing values that contradict the codebase.

### Step 2: Locate the file

The default location is `DESIGN.md` at the repo root, alongside `README.md` and `AGENTS.md`. Use the existing one if present. If the user specifies a different path, honor it.

For updates, **always read the file first**. The spec explicitly preserves unknown sections, unknown color/typography token names, and unknown component properties -- so don't strip anything you don't recognize.

### Step 3: Decide the tokens

Tokens are the normative values. Choose them deliberately:

- **`name`** (required): A short, evocative name for the design system (e.g., `"DevFocus Dark"`, `"Daylight Prestige"`). Not a filename.
- **`version`** (optional): Currently `alpha`. Include it if you're being explicit about spec compliance.
- **`description`** (optional): A one-sentence summary. The Overview section already covers this in prose, so it's only useful for tooling that reads YAML alone.
- **`colors`**, **`typography`**, **`rounded`**, **`spacing`**, **`components`**: see the schema reference below.

When the user is vague, prefer fewer, well-considered tokens over a sprawling palette. A good first DESIGN.md typically has:

- 4-6 color tokens (primary, secondary, surface, on-surface, error, optionally tertiary or neutral)
- 6-12 typography levels (display/headline/body/label, with a couple of size variants each)
- A 3-5 step radius scale
- A 5-7 step spacing scale (often based on a 4px or 8px grid)
- A handful of components covering the loudest UI atoms (buttons, inputs, cards)

Don't pad. The spec accepts extension later; an over-specified first draft is harder to refine.

### Step 4: Write the front matter

The YAML block must begin with a line containing exactly `---` and end with a line containing exactly `---`. Place it at the very top of the file -- anything before it breaks the front matter.

Quote hex colors so YAML doesn't try to parse them as numbers or comments:

```yaml
colors:
  primary: "#1A1C1E"   # quoted -- # would otherwise start a comment
```

Use dimensions with units (`px`, `em`, `rem`) for sizes, and prefer unitless multipliers for `lineHeight` (`1.6`, not `24px`). Numeric `fontWeight` values (`400`, `700`) are easier for downstream tooling than strings (`"semibold"`).

Token references use curly braces with a dotted path: `"{colors.primary}"`, `"{rounded.md}"`. **References must be quoted** in YAML, otherwise the braces are read as a flow-style mapping. Most groups (colors, rounded, spacing, typography) require references to point at primitive values; only the `components` group permits references to composite values like `"{typography.label-md}"`.

### Step 5: Write the markdown body

Sections use `##` headings and follow this canonical order. Skip any section that genuinely doesn't apply, but don't reorder.

| # | Section | Aliases | Purpose |
|---|---|---|---|
| 1 | Overview | Brand & Style | Brand personality, target audience, emotional tone. Foundational fallback context. |
| 2 | Colors | | What each palette is for, with semantic role descriptions. |
| 3 | Typography | | Roles for each level (headline, body, label), tone of voice in type. |
| 4 | Layout | Layout & Spacing | Grid model, spacing scale, containment principles. |
| 5 | Elevation & Depth | Elevation | How hierarchy is conveyed -- shadows, tonal layers, borders. |
| 6 | Shapes | | Corner radii, edge treatments, overall shape language. |
| 7 | Components | | Style guidance per component atom. |
| 8 | Do's and Don'ts | | Practical guardrails for generation. |

An optional `#` title may appear above the first `##`; it's not parsed as a section.

The body's job is the *why*. Don't restate the YAML values verbatim -- explain the role each token plays, when to reach for it, and how it relates to the others. Prose may use evocative color names ("Deep Ink", "Warm Limestone") that map to systematic token names; the tokens stay normative.

A duplicate `##` heading (e.g., two `## Colors`) is a hard error per the spec -- it should never be written or left in place.

### Step 6: Validate before declaring done

Walk these checks once the file is written. They catch the failures most likely to break downstream tooling:

1. **Front matter delimiters**: file starts with `---` and the block ends with `---`. Nothing precedes the opening delimiter.
2. **YAML parses**: no tabs in indentation, hex values quoted, references quoted, dimensions have units.
3. **`name` is present** in the front matter.
4. **`colors.primary` is defined** if any colors are defined at all (the spec recommends at least the primary palette).
5. **All token references resolve**: every `{path.to.token}` points at a value that actually exists in the YAML tree.
6. **Section order matches the canonical sequence** for any sections that are present. Custom sections (e.g., `## Iconography`, `## Motion`) are allowed; place them after the canonical ones unless they fit naturally elsewhere.
7. **No duplicate `##` headings.**
8. **Typography composites are well-formed**: `fontFamily` is a string, `fontSize` is a dimension, `fontWeight` is numeric, `lineHeight` is a unitless number or a dimension.
9. **Component property names** are from the standard set where possible (`backgroundColor`, `textColor`, `typography`, `rounded`, `padding`, `size`, `height`, `width`). Custom properties are accepted by consumers with a warning -- mention any you've added.

If you find issues during validation, fix them in place rather than reporting and waiting, unless the user explicitly asked for an audit-only pass.

### Step 7: Summarize the change

After writing or editing, give the user a short summary:

- Path to the file
- The system's `name`
- How many tokens of each type were defined (or changed)
- Any deliberate omissions or extensions (custom sections, unknown component properties)
- Anything the user should review or fill in (e.g., "I picked a placeholder error color -- swap if you have a brand value")

## Schema reference

### Front matter shape

```yaml
---
version: alpha            # optional
name: <string>            # required, short evocative system name
description: <string>     # optional one-liner
colors:
  <token-name>: <Color>
typography:
  <token-name>:
    fontFamily: <string>
    fontSize: <Dimension>
    fontWeight: <number>
    lineHeight: <Dimension | number>
    letterSpacing: <Dimension>
    fontFeature: <string>
    fontVariation: <string>
rounded:
  <scale-level>: <Dimension>
spacing:
  <scale-level>: <Dimension | number>
components:
  <component-name>:
    <property>: <string | TokenReference>
---
```

### Token types

| Type | Format | Example |
|---|---|---|
| Color | `#` + sRGB hex (3, 6, or 8 chars) | `"#1A1C1E"`, `"#fff"` |
| Dimension | number + unit (`px`, `em`, `rem`) | `48px`, `-0.02em`, `1.5rem` |
| Token reference | `{path.to.token}`, quoted in YAML | `"{colors.primary}"` |
| Typography | composite object (see above) | see the example file |

### Recommended token names

The spec doesn't mandate names, but these are the conventions design agents recognize most reliably:

- **Colors**: `primary`, `secondary`, `tertiary`, `neutral`, `surface`, `on-surface`, `error`. Tonal scales are commonly suffixed (`primary-10`, `primary-60`, `primary-90`).
- **Typography**: `headline-display`, `headline-lg`, `headline-md`, `body-lg`, `body-md`, `body-sm`, `label-lg`, `label-md`, `label-sm`. (Older systems often use `h1`-`h6`; both are valid.)
- **Rounded**: `none`, `sm`, `md`, `lg`, `xl`, `full` (where `full: 9999px` for pills/circles).
- **Spacing**: `xs`, `sm`, `md`, `lg`, `xl`, plus task-specific names like `gutter`, `margin`, `base`.

Custom names are accepted by consumers; just stay internally consistent within a file.

### Component properties

Standard properties consumers recognize natively:

| Property | Type |
|---|---|
| `backgroundColor` | Color |
| `textColor` | Color |
| `typography` | Typography (literal or `{typography.body-md}`-style reference) |
| `rounded` | Dimension |
| `padding` | Dimension |
| `size` | Dimension |
| `height` | Dimension |
| `width` | Dimension |

Variants for interaction states are written as separate keys with a related name, not nested:

```yaml
components:
  button-primary:
    backgroundColor: "{colors.primary-60}"
    textColor: "{colors.primary-20}"
    rounded: "{rounded.md}"
    padding: 12px
  button-primary-hover:
    backgroundColor: "{colors.primary-70}"
  button-primary-pressed:
    backgroundColor: "{colors.primary-80}"
```

## Worked example

A minimal but complete `DESIGN.md` for a dark productivity app:

```markdown
---
version: alpha
name: DevFocus Dark
colors:
  primary: "#2665fd"
  secondary: "#475569"
  surface: "#0b1326"
  on-surface: "#dae2fd"
  error: "#ffb4ab"
typography:
  headline-md:
    fontFamily: Inter
    fontSize: 24px
    fontWeight: 600
    lineHeight: 1.25
    letterSpacing: -0.01em
  body-md:
    fontFamily: Inter
    fontSize: 16px
    fontWeight: 400
    lineHeight: 1.6
  label-sm:
    fontFamily: Inter
    fontSize: 12px
    fontWeight: 500
    lineHeight: 1
    letterSpacing: 0.08em
rounded:
  sm: 4px
  md: 8px
  lg: 12px
  full: 9999px
spacing:
  xs: 4px
  sm: 8px
  md: 16px
  lg: 24px
  xl: 40px
components:
  button-primary:
    backgroundColor: "{colors.primary}"
    textColor: "{colors.on-surface}"
    rounded: "{rounded.md}"
    padding: 12px
    typography: "{typography.label-sm}"
  button-primary-hover:
    backgroundColor: "#1f56e0"
  input-base:
    backgroundColor: "{colors.surface}"
    textColor: "{colors.on-surface}"
    rounded: "{rounded.sm}"
    padding: 10px
---

# Design System

## Overview
A focused, minimal dark interface for a developer productivity tool.
Clean lines, low visual noise, high information density. The aesthetic
should feel like a code editor, not a marketing site.

## Colors
- **Primary (#2665fd):** the only saturated color in the system. Reserved
  for the single most important action on a screen and for active states.
- **Secondary (#475569):** muted slate for chips, secondary controls, and
  metadata that needs to recede.
- **Surface (#0b1326):** page and panel backgrounds.
- **On-surface (#dae2fd):** primary text on dark backgrounds. Maintains
  WCAG AA against `surface`.
- **Error (#ffb4ab):** validation errors and destructive confirmations.

## Typography
- **Headlines** use Inter Semi-Bold with tight tracking for a confident,
  technical voice.
- **Body** is Inter Regular at 16px with a 1.6 line height for long-form
  reading without strain.
- **Labels** are uppercase Inter Medium with positive tracking for section
  headers and metadata callouts.

## Layout
A strict 8px spacing scale. Content lives in a 1200px max-width container
on desktop and a fluid grid on mobile.

## Elevation & Depth
Depth comes from tonal layers and 1px borders, not shadows. Cards sit on
the page via a slightly lighter surface tint and an on-surface border.

## Shapes
Soft 8px corners on interactive elements, 4px on inputs, fully rounded
(9999px) on pills and avatars. Avoid sharp corners.

## Components
- **Buttons**: 8px radius, primary uses brand blue with on-surface text.
- **Inputs**: 4px radius, surface background with a 1px on-surface border.
- **Cards**: 8px radius, no shadow, 1px border at 20% on-surface.

## Do's and Don'ts
- Do reserve the primary color for one CTA per screen.
- Don't mix rounded and sharp corners in the same view.
- Do maintain WCAG AA contrast (4.5:1) for body text.
- Don't introduce a third font family.
```

## Edge cases and gotchas

- **Hex colors in YAML**: always quote them. `primary: #1A1C1E` is parsed as a YAML comment after `#`. `primary: "#1A1C1E"` is correct.
- **Token references**: always quote them. `"{colors.primary}"`, never `{colors.primary}` -- bare braces start a flow mapping.
- **`lineHeight`**: prefer unitless multipliers (`1.6`) over fixed dimensions (`24px`). Multipliers scale with font size; fixed values don't.
- **Composite references in non-component groups**: not allowed. You can do `{colors.primary-60}` from a component, but not `{colors.primary}` -> a typography object.
- **Tonal scales**: the spec doesn't mandate them, but tools like Stitch generate `primary-10` through `primary-90` automatically. If you write components that reference them (`{colors.primary-60}`), make sure the scale is actually defined in the YAML.
- **Custom sections**: accepted. `## Iconography`, `## Motion`, `## Voice & Tone` are all fine. Place them after the canonical sections unless there's a strong reason otherwise.
- **Custom component properties**: accepted but consumers may warn. Use them when no standard property fits (e.g., `borderColor`, `borderWidth`); flag them to the user so they know.
- **Updating without clobbering**: when extending an existing file, preserve unknown sections, unknown token names, and unknown component properties. The spec explicitly accepts them; deleting them is data loss.
- **Duplicate headings**: `## Colors` twice is rejected by the spec. If a file has them, merge them; never silently leave both.

## Anti-patterns

- **Inventing values to fill the schema.** If the user said "warm and friendly" but didn't pick a font, propose one and call it out -- don't pretend the choice was inevitable.
- **Restating the YAML in the prose.** The body is the *why*, not a JSON dump. If the only sentence you can write about a token is its hex value, the prose isn't earning its place.
- **Over-specification on a first pass.** A 200-line palette is harder to maintain and harder for an agent to apply consistently than a 6-color core. Start small; the spec accepts extension later.
- **Reordering sections "to make sense."** The canonical order exists so consumers can find sections without searching. Keep it.
- **Skipping validation because "it looks right."** Hex quoting and reference quoting are the most common mistakes; check them every time.

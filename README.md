# DESIGN.md Sync — Figma ⇄ design.md bridge (Figma Design Agent edition)

Register this skill in Figma's custom skill feature and run it from the chat. It syncs a Figma file's design tokens bidirectionally with a DESIGN.md file that follows [Google's design.md spec](https://github.com/google-labs-code/design.md/blob/main/docs/spec.md) — Import builds variables, text styles, and components from a DESIGN.md; Export reads a file's existing tokens and writes a DESIGN.md.

> *日本語 README はこちら: [README.ja.md](README.ja.md)*

> **About the paid course**: This skill is identical in content — not a feature-trimmed edition — to one of the 21 skills bundled in the paid course "[KMRVID Figma Skills](https://kmrvid-claude-skills.gaspanik.workers.dev/figma/)". Full catalog: [KMRVID Figma Skills README](https://fragrant-edam-563.notion.site/KMRVID-Figma-Skills-README-md-3ad5dae25dd280d69c12c69b277d90b6)

> **Course preview**: See the skills in action on [YouTube](https://www.youtube.com/@kmrvid/videos) or the course's [free preview](https://kmrvid.com/apps/ldt-course/course/6922c0ad336ff5545bbff61d/6a6b17c49dc4e0183716f605?locale=en) (no signup required).

---

## What this is

The design.md spec describes a design system as a single Markdown file — YAML frontmatter for tokens (colors, typography, spacing, corner radius, components), prose for the reasoning behind them. This skill is the bridge between that file and a real Figma file:

- **Import** — attach a DESIGN.md (yours, a teammate's, or a third-party template) and this skill creates matching variable collections, text styles, and components in Figma, so the tokens are immediately usable in design work.
- **Export** — already have variables and text styles set up in Figma? This skill reads them back out and generates a DESIGN.md, including a consistency check that flags any color defined but never referenced by a component.

```
Attach a DESIGN.md?
        │
   ┌────┴────┐
  yes         no → ask: Import or Export?
   │
   ▼
Import: parse tokens (frontmatter or prose) → create variable
collections → create text styles (with font fallback) → create
components → optional preview page → completion report with counts

Export: collect variables/text styles/components → generate YAML
frontmatter + prose body → consistency check (every color referenced?
body matches tokens?) → optional preview page → output
```

Every creation/binding step re-verifies its own output — re-fetching what was just created and checking the actual bound value, not just whether the API call succeeded — before reporting a count.

---

## Scope

- **Base tokens and default component states only.** This isn't a full design-system state manager — it syncs colors, typography, spacing, corner radius, and each component's default appearance. Hover/active/focus/disabled states and full variant matrices are out of scope; pair this with a tool built for that (Storybook, etc.) for the rest.
- **Handles non-strict DESIGN.md too.** Not every real-world DESIGN.md follows the YAML-frontmatter format exactly — Import also reads prose-only Markdown (headed sections, no frontmatter) and infers the same token data from it.
- **A component category that doesn't fit the model may be skipped.** For example, a circular shape in an otherwise zero-corner-radius system — Import calls this out in its completion report rather than forcing a bad fit.
- **`colors.primary` is required by the spec.** If a source file has no `primary` defined, Export infers a reasonable stand-in (the color that's actually doing that job in the file) and states so in its report.

---

## Requirements

- **Figma (Design Agent / custom skill feature).** Reads and writes go through Figma's built-in Plugin API script execution tool (e.g. `evaluate_script`). No external MCP servers or API keys required — register `SKILL.md` as a custom skill and it runs.

---

## Repo structure

```
skills/
  design-md/
    SKILL.md          — skill definition to register in Figma's custom skill feature
    LICENSE
```

---

## Getting started

**Once published on Figma Community**, you'll be able to add this skill directly from the [AI Skills library](https://www.figma.com/community/ai-skills) — no download needed. Until then, register it from source:

**From source:** clone this repo and register the skill file yourself.

```bash
git clone https://github.com/gaspanik/design-md-skill
```

**1. Register `skills/design-md/SKILL.md`** in Figma's custom skill feature.

**2. Run it from the chat**

```
/design-md
```

```
Import this DESIGN.md and build out the variables and components
```

```
このファイルの変数・コンポーネントからDESIGN.mdを書き出して
```

Attach a DESIGN.md file to go straight into Import mode, or run with nothing attached to be asked which direction to sync.

---

## Runtime flow

1. **Mode check** — a DESIGN.md attached to the chat goes straight to Import; nothing attached asks Import vs. Export
2. **Import** — parse tokens from frontmatter (or infer from prose if there's none) → create variable collections → create text styles with font-load fallback → create components with variables bound → optional preview page
3. **Export** — collect every local variable, text style, and component → generate YAML frontmatter + prose body → run the consistency check (every color referenced by a component? does the body match the tokens?)
4. **Verification** — every write step re-fetches what it just created and checks the actual bound value before reporting a count; no step reports success from the API call succeeding alone
5. **Completion report** — token/component counts, any skipped section or component category with a reason, font substitutions, and whether a preview page was created

---

## Example output

```
Import 完了報告 — RawBlock Design System

検出セクション: Colors ✓ / Typography ✓ / Spacing ✓ / Border Radius ✓ / Components ✓

変数コレクション: 4コレクション作成、変数34個 全数一致
テキストスタイル: 8個作成（fontSizeバインド 8/8確認、フォント代替なし）
コンポーネント: 11個作成（カラーfillsバインド 8/10、Ghost・Inputは意図的にtransparent）
```

---

## Part of a larger set

This skill is one of a 21-skill bundle, **KMRVID Figma Skills**, covering AI-slop-resistant page generation, multi-layout-pattern exploration, layer cleanup, contrast/accessibility checks, tokenization, component audits, ALT text suggestions, and more: [gaspanik.gumroad.com/l/kmrvid-figmaskills](https://gaspanik.gumroad.com/l/kmrvid-figmaskills)

---

Built by Masaaki Komori - [@cipher](https://x.com/cipher) · Skill for [Figma](https://www.figma.com/)

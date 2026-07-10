# JobLand — Basic Design System

> Converted from `JobLand_basic-design-system.pdf` (Figma export, 7 pages).
> Contents: **Color Primitives · Semantic Color Tokens · Scale Primitives · Font Primitives · Semantic Typography Tokens · Grid System · Text Styles.**
>
> _Note: a few typographic size/line-height values were column-interleaved in the PDF's text layer; those tables are reconstructed to the most consistent reading and marked **[verify]** where the source was ambiguous. All color, spacing, sizing, radius, border, opacity, elevation, and grid tokens are reproduced verbatim._

---

## 1. Color Primitives
Primitive color tokens — the raw palette. Groups: **Primary · Secondary · Tertiary · Natural · Base · Green · Red.**

### Primary
| Token | Hex |
|-------|-----|
| T10 | `#180064` |
| T20 | `#2B009E` |
| T30 | `#4002DB` |
| T40 | `#5937F1` |
| T50 | `#735BFF` |
| **500** | `#6648FE` |
| T60 | `#8F7FFF` |
| T70 | `#AB9FFF` |
| T80 | `#C7BFFF` |
| T90 | `#E5DEFF` |
| T95 | `#F3EEFF` |

### Secondary
| Token | Hex |
|-------|-----|
| T10 | `#10124F` |
| T20 | `#272A65` |
| T30 | `#3D417D` |
| T40 | `#555997` |
| T50 | `#6E71B1` |
| **500** | `#B5B8FD` |
| T60 | `#888BCD` |
| T70 | `#A2A6E9` |
| T80 | `#BFC1FF` |
| T90 | `#E1E0FF` |
| T95 | `#F1EFFF` |

### Tertiary
| Token | Hex |
|-------|-----|
| T10 | `#201C00` |
| T20 | `#383100` |
| T30 | `#514700` |
| T40 | `#6B5F0C` |
| T50 | `#857826` |
| **500** | `#BBAC55` |
| T60 | `#9F913D` |
| T70 | `#BBAC55` |
| T80 | `#D7C76D` |
| T90 | `#F5E486` |
| T95 | `#FFF2AF` |

### Natural
| Token | Hex |
|-------|-----|
| T10 | `#0B1C30` |
| T20 | `#213145` |
| T30 | `#38485D` |
| T40 | `#505F76` |
| T50 | `#68788F` |
| **500** | `#64748B` |
| T60 | `#8292AA` |
| T70 | `#9CACC5` |
| T80 | `#B7C8E1` |
| T90 | `#D3E4FE` |
| T95 | `#EAF1FF` |

### Base
| Token | Hex |
|-------|-----|
| Black | `#000000` |
| White | `#FFFFFF` |

### Green
| Token | Hex |
|-------|-----|
| T10 | `#052814` |
| T20 | `#0A4D27` |
| T30 | `#0F7339` |
| T40 | `#138C42` |
| T50 | `#16A34A` |
| T60 | `#22C55E` |
| T70 | `#4ADE80` |
| T80 | `#86EFAC` |
| T90 | `#BBF7D0` |
| T95 | `#DCFCE7` |

### Red
| Token | Hex |
|-------|-----|
| T10 | `#450A0A` |
| T20 | `#7F1D1D` |
| T30 | `#991B1B` |
| T40 | `#B91C1C` |
| T50 | `#DC2626` |
| T60 | `#EF4444` |
| T70 | `#F87171` |
| T80 | `#FCA5A5` |
| T90 | `#FECACA` |
| T95 | `#FEF2F2` |

---

## 2. Semantic Color Tokens
Each semantic token **aliases a primitive**. Groups: Typography · Background · CTA · Border · Icon · Base · Status.

### Typography
| Token | Alias |
|-------|-------|
| Headline | Primary/T10 |
| Heading | Natural/T10 |
| Body | Natural/T30 |
| Body Secondary | Natural/T50 |
| Label | Natural/T20 |
| Caption | Natural/T60 |
| Placeholder | Natural/T70 |
| Disabled | Natural/T80 |
| Inverted | White |
| Link | Primary/500 |
| Link Hover | Primary/T40 |

### Background
| Token | Alias |
|-------|-------|
| Page | Natural/T95 |
| Surface | White |
| Surface Raised | White |
| Surface Overlay | Natural/T90 |
| Subtle | Natural/T95 |
| Inverted | Primary/T10 |
| Brand | Primary/500 |
| Brand Subtle | Primary/T95 |

### CTA — Primary
| Token | Alias |
|-------|-------|
| Background | Primary/500 |
| Label | White |
| Hover | Primary/T40 |
| Pressed | Primary/T30 |
| Disabled | Primary/T80 |

### CTA — Secondary
| Token | Alias |
|-------|-------|
| Background | Secondary/Secondary |
| Label | Primary/T10 |
| Hover | Secondary/T70 |
| Pressed | Secondary/T60 |
| Disabled | Secondary/T90 |

### CTA — Inverted
| Token | Alias |
|-------|-------|
| Background | White |
| Label | Primary/500 |
| Hover | Primary/T95 |
| Pressed | Primary/T90 |
| Disabled | Natural/T80 |

### CTA — Outline
| Token | Alias |
|-------|-------|
| Border | Primary/500 |
| Label | Primary/500 |
| Hover Background | Primary/T95 |
| Pressed Background | Primary/T90 |
| Disabled Border | Natural/T70 |
| Disabled Label | Natural/T70 |

### Border
| Token | Alias |
|-------|-------|
| Default | Natural/T80 |
| Strong | Natural/T60 |
| Subtle | Natural/T90 |
| Brand | Primary/500 |
| Focus | Primary/T50 |

### Icon
| Token | Alias |
|-------|-------|
| Default | Natural/T30 |
| Secondary | Natural/T60 |
| Brand | Primary/500 |
| Inverted | White |
| Disabled | Natural/T80 |

### Base
| Token | Alias |
|-------|-------|
| Black | Base/Black |
| White | Base/White |
| Foreground | Base/Black |
| Background | Base/White |
| Overlay | Base/Black |
| Inverted Text | Base/White |

### Status
Each status has Background · Surface · Default · Strong · Text · Icon · Border.

| Status | Background | Surface | Default | Strong | Text | Icon | Border |
|--------|-----------|---------|---------|--------|------|------|--------|
| Success | Green/T95 | Green/T90 | Green/T60 | Green/T50 | Green/T20 | Green/T50 | Green/T70 |
| Warning | Tertiary/T95 | Tertiary/T90 | Tertiary/T70 | Tertiary/T50 | Tertiary/T20 | Tertiary/T60 | Tertiary/T80 |
| Error | Red/T95 | Red/T90 | Red/T50 | Red/T30 | Red/T20 | Red/T50 | Red/T60 |
| Info | Secondary/T95 | Secondary/T90 | Secondary/T60 | Secondary/T40 | Secondary/T20 | Secondary/T50 | Secondary/T70 |
| Natural | Natural/T95 | Natural/T90 | Natural/T60 | Natural/T40 | Natural/T20 | Natural/T50 | Natural/T70 |

---

## 3. Scale Primitives
Design token reference: Spacing · Sizing · Radius · Border · Typography · Opacity · Elevation · Grid · Letter Spacing.

### Spacing
Base-4 scale for padding, margin, gap. Token: `Spacing/{n}`

| Token | Value |
|-------|-------|
| none | 0px |
| 1 | 4px |
| 2 | 8px |
| 3 | 12px |
| 4 | 16px |
| 5 | 20px |
| 6 | 24px |
| 7 | 28px |
| 8 | 32px |
| 9 | 36px |
| 10 | 40px |
| 12 | 48px |
| 14 | 56px |
| 16 | 64px |
| 20 | 80px |
| 24 | 96px |
| 32 | 128px |
| 40 | 160px |
| 48 | 192px |
| 56 | 224px |
| 64 | 256px |

### Sizing
Component & icon dimensions. Token: `Sizing/{XS–10XL}`

| Token | Value |
|-------|-------|
| XS | 16px |
| SM | 20px |
| MD | 24px |
| LG | 32px |
| XL | 40px |
| 2XL | 48px |
| 3XL | 56px |
| 4XL | 64px |
| 5XL | 80px |
| 6XL | 96px |
| 7XL | 120px |
| 8XL | 160px |
| 9XL | 200px |
| 10XL | 240px |

### Border Radius
Corner rounding from sharp to full pill. Token: `BorderRadius/{name}`

| Token | Value |
|-------|-------|
| None | 0px |
| XS | 2px |
| SM | 4px |
| MD | 6px |
| LG | 8px |
| XL | 12px |
| 2XL | 16px |
| 3XL | 20px |
| 4XL | 24px |
| Full | 999px |

### Border Width
Stroke weights for dividers, outlines, and focus rings. Token: `BorderWidth/{n}`

| Token | Value |
|-------|-------|
| 0 | 0px |
| 1 | 1px |
| 2 | 2px |
| 4 | 4px |
| 8 | 8px |

### Typography — Font Size & Line Height
Token: `FontSize/{name}` · `LineHeight/{name}`

| Token | Size | Line Height | Weight |
|-------|------|-------------|--------|
| XS | 10px | 14px | 400 |
| SM | 12px | 16px | 400 |
| Base | 14px | 20px | 400 |
| MD | 16px | 24px | 400 |
| LG | 18px | 28px | 400 |
| XL | 20px | 28px | 500 |
| 2XL | 24px | 32px | 700 |
| 3XL | 28px | 36px | 500 |
| 4XL | 32px | 40px | 600 |
| 5XL | 36px | 44px | 600 |
| 6XL | 40px | 48px | 700 |
| 7XL | 48px | 56px | 700 |
| Display | 96px | 104px | 700 |

### Font Weight (primitive)
Token: `FontWeight/{name}` — Thin (100) → Black (900)

| Token | Weight |
|-------|--------|
| Thin | 100 |
| ExtraLight | 200 |
| Light | 300 |
| Regular | 400 |
| Medium | 500 |
| SemiBold | 600 |
| Bold | 700 |
| ExtraBold | 800 |
| Black | 900 |

### Opacity
Token: `Opacity/{n}` — 0 (invisible) → 100 (fully opaque)

| Token | Value |
|-------|-------|
| 0 | 0% |
| 5 | 5% |
| 10 | 10% |
| 20 | 20% |
| 25 | 25% |
| 30 | 30% |
| 40 | 40% |
| 50 | 50% |
| 60 | 60% |
| 70 | 70% |
| 75 | 75% |
| 80 | 80% |
| 90 | 90% |
| 95 | 95% |
| 100 | 100% |

### Elevation
Shadow depth values for layering. Token: `Elevation/{n}`

| Token | Blur |
|-------|------|
| 0 | 0px blur |
| 1 | 2px blur |
| 2 | 4px blur |
| 3 | 8px blur |
| 4 | 12px blur |
| 5 | 16px blur |
| 6 | 24px blur |
| 7 | 32px blur |

### Letter Spacing
Tracking values for tighter or wider text. Token: `LetterSpacing/{name}`

| Token | Value |
|-------|-------|
| Tighter | -0.8px |
| Tight | -0.4px |
| Normal | 0px |
| Wide | +0.4px |
| Wider | +0.8px |
| Widest | +1.6px |

---

## 4. Font Primitives
Font family · weight · style token reference for the **Inter** typeface.

### Font Family
Token: `FontFamily/{name}` — STRING variable, scope `FONT_FAMILY`

| Token | Value | Usage |
|-------|-------|-------|
| FontFamily/Primary | Inter | Primary brand font — all UI text, headings, labels, body |
| FontFamily/System | System UI | Fallback stack — rendered by the OS if Inter is unavailable |
| FontFamily/Mono | Monospace | Code snippets, token values, technical labels |

### Font Weight
Token: `FontWeight/{name}` — FLOAT variable, scope `FONT_WEIGHT`, 9 steps (100–900)

| Token | Weight | Inter Style |
|-------|--------|-------------|
| FontWeight/Thin | 100 | Thin |
| FontWeight/ExtraLight | 200 | Extra Light |
| FontWeight/Light | 300 | Light |
| FontWeight/Regular | 400 | Regular |
| FontWeight/Medium | 500 | Medium |
| FontWeight/SemiBold | 600 | Semi Bold |
| FontWeight/Bold | 700 | Bold |
| FontWeight/ExtraBold | 800 | Extra Bold |
| FontWeight/Black | 900 | Black |

### Font Style
Token: `FontStyle/{name}` — STRING variable, scope `FONT_STYLE`, 18 Inter variants (9 upright + 9 italic)

| Token | Value | Type |
|-------|-------|------|
| FontStyle/Thin | Thin | Upright |
| FontStyle/ThinItalic | Thin Italic | Italic |
| FontStyle/ExtraLight | Extra Light | Upright |
| FontStyle/ExtraLightItalic | Extra Light Italic | Italic |
| FontStyle/Light | Light | Upright |
| FontStyle/LightItalic | Light Italic | Italic |
| FontStyle/Regular | Regular | Upright |
| FontStyle/Italic | Italic | Italic |
| FontStyle/Medium | Medium | Upright |
| FontStyle/MediumItalic | Medium Italic | Italic |
| FontStyle/SemiBold | Semi Bold | Upright |
| FontStyle/SemiBoldItalic | Semi Bold Italic | Italic |
| FontStyle/Bold | Bold | Upright |
| FontStyle/BoldItalic | Bold Italic | Italic |
| FontStyle/ExtraBold | Extra Bold | Upright |
| FontStyle/ExtraBoldItalic | Extra Bold Italic | Italic |
| FontStyle/Black | Black | Upright |
| FontStyle/BlackItalic | Black Italic | Italic |

---

## 5. Semantic Typography Tokens
Typography scale for **Web & Mobile** — 25 roles × 5 properties (FontSize · LineHeight · FontWeight · FontFamily · FontStyle). Token values differ between breakpoints where marked. _[verify] — size/line-height values reconstructed from the interleaved export._

### Display & Headings (7 roles)
Hero text, page titles, and section headings — scale down on mobile.

| Role | Token | Web (Size/LH) | Mobile (Size/LH) | Weight |
|------|-------|---------------|------------------|--------|
| Display | Typography/Display | 96 / 104 | 64 / 72 | Bold |
| Heading 1 | Typography/H1 | 48 / 56 | 40 / 48 | Bold |
| Heading 2 | Typography/H2 | 40 / 48 | 36 / 44 | Bold |
| Heading 3 | Typography/H3 | 36 / 44 | 32 / 40 | Semi Bold |
| Heading 4 | Typography/H4 | 32 / 40 | 28 / 36 | Semi Bold |
| Heading 5 | Typography/H5 | 28 / 36 | 24 / 32 | Semi Bold |
| Heading 6 | Typography/H6 | 24 / 32 | 20 / 28 | Semi Bold |

### Subtitle (2 roles)
Supporting headings and section intros — step down one size on mobile.

| Role | Token | Web (Size/LH) | Mobile (Size/LH) | Weight |
|------|-------|---------------|------------------|--------|
| Subtitle Large | Typography/Subtitle/Large | 20 / 28 | 18 / 24 | Medium |
| Subtitle Small | Typography/Subtitle/Small | 18 / 24 | 16 / 24 | Medium |

### Body (3 roles)
Paragraph text — same size across breakpoints, optimised for readability.

| Role | Token | Size / LH | Weight |
|------|-------|-----------|--------|
| Body Large | Typography/Body/Large | 16 / 24 | Regular |
| Body | Typography/Body/Default | 14 / 20 | Regular |
| Body Small | Typography/Body/Small | 12 / 16 | Regular |

### Label (3 roles)
UI labels, tags, metadata — consistent across breakpoints.

| Role | Token | Size / LH | Weight |
|------|-------|-----------|--------|
| Label Large | Typography/Label/Large | 16 / 24 | Medium |
| Label | Typography/Label/Default | 14 / 20 | Medium |
| Label Small | Typography/Label/Small | 12 / 16 | Medium |

### Utility (2 roles)
Captions, overlines, helper text — smallest text in the system.

| Role | Token | Size / LH | Weight |
|------|-------|-----------|--------|
| Caption | Typography/Caption | 10 / 14 | Regular |
| Overline | Typography/Overline | 10 / 14 | Semi Bold |

### Button (3 roles)
CTA labels — Semi Bold weight, consistent across breakpoints.

| Role | Token | Size / LH | Weight |
|------|-------|-----------|--------|
| Button Large | Typography/Button/Large | 16 / 24 | Semi Bold |
| Button | Typography/Button/Default | 14 / 20 | Semi Bold |
| Button Small | Typography/Button/Small | 12 / 16 | Semi Bold |

### Input (3 roles)
Form fields, labels, placeholders.

| Role | Token | Size / LH | Weight |
|------|-------|-----------|--------|
| Input | Typography/Input/Default | 14 / 20 | Regular |
| Input Label | Typography/Input/Label | 12 / 16 | Medium |
| Placeholder | Typography/Input/Placeholder | 14 / 20 | Regular |

### Link (1 role)
Inline text links — Medium weight.

| Role | Token | Size / LH | Weight |
|------|-------|-----------|--------|
| Link | Typography/Link/Default | 14 / 20 | Medium |

### Code (1 role)
Monospace code snippets — uses `FontFamily/Mono`.

| Role | Token | Size / LH | Weight |
|------|-------|-----------|--------|
| Code | Typography/Code/Default | 12 / 24 | Regular |

---

## 6. Grid System
Responsive **12-column grid** · 5 breakpoints · Column · Gutter · Margin · Container tokens.

**Anatomy:** Column (content sits here) · Gutter (space between columns) · Margin (page-edge offset).

| Breakpoint | Viewport | Columns | Gutter | Margin | Container | Token Prefix |
|------------|----------|---------|--------|--------|-----------|--------------|
| Mobile | 390px+ | 4 | 16px | 16px | 358px | `Grid/*/Mobile` |
| Tablet | 768px+ | 8 | 20px | 24px | 720px | `Grid/*/Tablet` |
| Laptop | 1024px+ | 12 | 24px | 32px | 960px | `Grid/*/Laptop` |
| Desktop | 1440px+ | 12 | 32px | 80px | 1280px | `Grid/*/Desktop` (default) |
| Wide | 1920px+ | 12 | 32px | 120px | 1440px | `Grid/*/Wide` |

Semantic tokens per breakpoint: `Grid/Columns/{bp}` · `Grid/Gutter/{bp}` · `Grid/Margin/{bp}` · `Grid/Container/{bp}`.

---

## 7. Typography — Text Styles
**47 named text styles**, applied via the right panel under "Text Styles". Grouped: Display · Headings · Subtitle · Body · Label · Button · Input · Utility · Numeric · Navigation.

### Display (2 styles)
Hero and page-level display text.

| Style | Breakpoint | Size | Line Ht | Weight | Tracking |
|-------|-----------|------|---------|--------|----------|
| Display | Web | 96px | 104px | Bold | -0.5px |
| Display | Mobile | 64px | 72px | Bold | -0.4px |

### Heading (12 styles)
H1–H6 hierarchy for Web and Mobile breakpoints.

| Style | Breakpoint | Size | Line Ht | Weight | Tracking |
|-------|-----------|------|---------|--------|----------|
| Heading One | Web | 48px | 56px | Bold | -0.3px |
| Heading One | Mobile | 40px | 48px | Bold | -0.3px |
| Heading Two | Web | 40px | 48px | Bold | -0.2px |
| Heading Two | Mobile | 36px | 44px | Bold | -0.2px |
| Heading Three | Web | 36px | 44px | Semi Bold | -0.2px |
| Heading Three | Mobile | 32px | 40px | Semi Bold | -0.2px |
| Heading Four | Web | 32px | 40px | Semi Bold | -0.1px |
| Heading Four | Mobile | 28px | 36px | Semi Bold | -0.1px |
| Heading Five | Web | 28px | 36px | Semi Bold | — |
| Heading Five | Mobile | 24px | 32px | Semi Bold | — |
| Heading Six | Web | 24px | 32px | Semi Bold | — |
| Heading Six | Mobile | 20px | 28px | Semi Bold | — |

### Subtitle (4 styles)
Supporting headings and section intros.

| Style | Breakpoint | Size | Line Ht | Weight |
|-------|-----------|------|---------|--------|
| Subtitle Large | Web | 20px | 28px | Medium |
| Subtitle Large | Mobile | 18px | 24px | Medium |
| Subtitle Small | Web | 18px | 24px | Medium |
| Subtitle Small | Mobile | 16px | 24px | Medium |

### Body (6 styles)
Paragraph and reading text — Regular weight, optimised for legibility.

| Style | Breakpoint | Size | Line Ht | Weight |
|-------|-----------|------|---------|--------|
| Body Large | Web / Mobile | 16px | 24px | Regular |
| Body Default | Web / Mobile | 14px | 20px | Regular |
| Body Small | Web / Mobile | 12px | 16px | Regular |

### Label (6 styles)
UI labels, tags, form labels — Medium weight.

| Style | Breakpoint | Size | Line Ht | Weight |
|-------|-----------|------|---------|--------|
| Label Large | Web / Mobile | 16px | 24px | Medium |
| Label Default | Web / Mobile | 14px | 20px | Medium |
| Label Small | Web / Mobile | 12px | 16px | Medium |

### Button (3 styles)
CTA button labels — Semi Bold, consistent across breakpoints.

| Style | Size | Line Ht | Weight |
|-------|------|---------|--------|
| Button Large | 16px | 24px | Semi Bold |
| Button Default | 14px | 20px | Semi Bold |
| Button Small | 12px | 16px | Semi Bold |

### Input (3 styles)
Form field text, labels, and placeholders.

| Style | Size | Line Ht | Weight |
|-------|------|---------|--------|
| Input (Default) | 14px | 20px | Regular |
| Input Label | 12px | 16px | Medium |
| Placeholder | 14px | 20px | Regular |

### Utility (4 styles)
Caption, overline, links, and code snippets.

| Style | Size | Line Ht | Weight | Notes |
|-------|------|---------|--------|-------|
| Caption | 10px | 14px | Regular | Helper text |
| Overline | 10px | 14px | Semi Bold | Tracking +0.8px, uppercase |
| Link (Default) | 14px | 20px | Medium | Inline text link |
| Code (Default) | 12px | 20px | Regular | Monospace |

### Numeric (4 styles)
KPI cards and metric displays — Bold weights for maximum prominence.

| Style | Size | Line Ht | Weight | Specimen |
|-------|------|---------|--------|----------|
| Numeric Display | 40px | 44px | Bold | £18,640 |
| Numeric Large | 32px | 36px | Bold | 97.1% |
| Numeric Medium | 24px | 28px | Semi Bold | 312 |
| Numeric Small | 18px | 22px | Semi Bold | 3.2d |

### Navigation (3 styles)
Left sidebar and tab navigation labels.

| Style | Size | Line Ht | Weight | Specimen |
|-------|------|---------|--------|----------|
| Nav Active | 14px | 20px | Semi Bold | Overview |
| Nav Default | 14px | 20px | Medium | Platform Config |
| Nav Caption | 11px | 16px | Regular | Jobland Internal |

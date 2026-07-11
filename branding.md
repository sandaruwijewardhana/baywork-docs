# BayWork — Brand Guidelines
### Sri Lanka's First Digital Garage Management Platform
**v1.0 — May 2026**

---

## 1. Brand Essence

**Name:** BayWork
**Tagline:** *Where the bay work gets done.*
**Promise:** Replace paper job cards, lost invoices, and guesswork with a system built for your garage.

**Personality**
- **Confident, not loud.** A workshop foreman, not a salesman.
- **Built, not designed.** Looks like engineered software, not a marketing template.
- **Sri Lankan-first.** Reads like it belongs in Colombo — not translated from San Francisco.
- **Bay over Buzzwords.** No "synergize," no "AI-powered." We say what it does.

**Voice rules**
- Active verbs. Short sentences. No throat-clearing.
- Numbers and specifics beat adjectives. ("10-minute setup" not "fast setup.")
- LKR, not $. Sri Lankan plate formats. Local cities.
- Never use "drivers." Say "customers" or "garage owners."

---

## 2. Logo

### Wordmark
Two-tone wordmark: **Bay** in Ink, **Work** in Amber. Set in Space Grotesk 700, tightly tracked (-0.02em).

### Mark
A solid amber diamond (square rotated 45°) with a smaller ink diamond cut inside it. Reads as a service bay viewed from above, or a stylized "B." The mark always sits to the left of the wordmark with 12px of clear space between them.

### Clear space
Minimum clear space on all sides of the locked-up logo: the height of the lowercase "a" in "Bay."

### Don't
- Don't recolor the mark outside the official palette.
- Don't outline the wordmark.
- Don't add a tagline lockup smaller than 12px.
- Don't stretch or skew either element.

---

## 3. Color System

All four palettes share the same role structure so the product UI can switch between them without breaking. Names use OKLCH-friendly tone families.

### 3.1 Workshop (default)
Deep, premium, confident. The flagship palette.

| Role | Token | Hex | Use |
|------|-------|-----|-----|
| Ink | `--ink` | `#0A0E1A` | Type on light, deep backgrounds |
| Ink Soft | `--ink-soft` | `#15243B` | Cards on dark, footer |
| Ink Mid | `--ink-mid` | `#2A3A57` | Borders on dark, hover |
| Amber | `--accent` | `#FFB12B` | Primary CTA, accent lines, brand mark |
| Amber Deep | `--accent-deep` | `#E08F00` | Pressed states, contrast text |
| Cream | `--surface` | `#F4EFE6` | Light section background (warm, not cold) |
| Paper | `--paper` | `#FFFFFF` | Card bodies |
| Steel | `--steel` | `#5B6473` | Secondary body text |
| Steel Light | `--steel-light` | `#9AA3B2` | Captions, placeholders |
| Hairline | `--hairline` | `#E5DED1` | Dividers on cream |
| Signal | `--signal` | `#22C972` | "Ready", success |
| Alert | `--alert` | `#FF5747` | Overdue, error |

### 3.2 Ceylon (alt)
Earthy, distinctly Sri Lankan. Deep teal + saffron.

| Role | Hex |
|------|-----|
| Ink | `#0E2620` |
| Ink Soft | `#143A30` |
| Accent | `#F2A33A` |
| Accent Deep | `#C77D14` |
| Surface | `#F1ECDF` |
| Steel | `#566962` |

### 3.3 Racing (alt)
Motorsport. High-contrast black/red/white.

| Role | Hex |
|------|-----|
| Ink | `#0A0A0C` |
| Ink Soft | `#16171B` |
| Accent | `#EF233C` |
| Accent Deep | `#B81632` |
| Surface | `#F5F5F7` |
| Steel | `#5C5F66` |

### 3.4 Electric (alt)
After-hours workshop. Charcoal + electric lime.

| Role | Hex |
|------|-----|
| Ink | `#0E1410` |
| Ink Soft | `#1A211B` |
| Accent | `#C7FF3D` |
| Accent Deep | `#94D013` |
| Surface | `#EEF0EC` |
| Steel | `#5A625B` |

### Usage rules
- **Accent ≤ 20% of any composition.** Use Amber for CTAs and one highlight word — not whole headlines.
- **Cream beats white** for light sections. White is reserved for cards on cream.
- Status colors (Signal / Alert) are constant across all four palettes.

---

## 4. Typography

### Type system
| Role | Family | Weight | Use |
|------|--------|--------|-----|
| Display | **Space Grotesk** | 700 | H1, H2, big numbers |
| Subheading | Space Grotesk | 600 | H3, card titles, navigation |
| Body | **Inter** | 400 / 500 | All running text |
| UI Caption | Inter | 600 uppercase, +0.08em | Eyebrow labels, chips |
| Mono | **JetBrains Mono** | 500 | IDs (`JC-2026-0041`), plates, code |

### Scale (desktop / mobile)
- Hero H1: 84 / 44
- Section H2: 56 / 34
- Card H3: 22 / 20
- Body L: 19 / 17
- Body: 16 / 15
- Caption: 13

### Rules
- Headlines: **tracking -0.025em**. Never positive tracking on display.
- Long-form body: `text-wrap: pretty`, 1.55 line-height.
- Never use Inter at 700+ — that's Space Grotesk's job.
- Numbers in stats are always **Space Grotesk 700**, never Inter.

---

## 5. Iconography & Imagery

### Icons
Lucide, 1.75px stroke, square caps. **Always 24px in product, 20px in marketing.** No filled icons.

### Imagery direction
**Bold illustrations + geometric shapes. No real photos in Phase 1.**
- Use SVG geometric compositions (stripes, grids, plates, simple shapes) as hero/section accents.
- Use **product screenshots** for the hero visual and the "see it in action" section.
- Where a photo would normally go, use a striped placeholder card with a monospace caption — that becomes a deliberate visual style, not a missing image.

### The number-plate motif
A recurring visual: Sri Lankan-format plates (yellow background, black border, JetBrains Mono lettering, "SRI LANKA" microtype on top) used to:
- Display job numbers, stat counts, registration numbers in hero.
- Anchor the pricing tier identifier.
- Appear in the empty/loading states of the product.

This is the brand's most ownable visual element. Use it.

---

## 6. Motion

- **Easing:** `cubic-bezier(0.2, 0.7, 0.2, 1)` — physical, decisive.
- **Durations:** 200ms (UI hover), 400ms (entrance), 800ms (hero reveals).
- **Stagger:** 50ms between siblings.
- **Hero kanban loop:** cards translate column-to-column every ~3s; cards never overlap, never reset abruptly — it loops cleanly.
- **No bouncy springs.** No `ease-in-out` everywhere. The product is industrial — motion should feel mechanical.

---

## 7. Layout Primitives

- **Max content width:** 1200px.
- **Section padding (desktop):** 120px top + bottom for hero/anchor sections, 80px for in-between.
- **Grid gutter:** 24px.
- **Border radius:** 6 (chips) / 12 (inputs, small cards) / 20 (large cards) / 28 (hero panel).
- **Shadows:** Use sparingly. One light shadow on cards (`0 1px 2px ink/4%, 0 8px 24px ink/6%`). One dramatic shadow on the hero product mock (`0 40px 80px ink/30%, 0 0 0 1px ink/40%`).

---

## 8. Copywriting templates

### Hero headline pattern
> Sri Lanka's First **{Promise Word}** Garage Management System.

The promise word ("Digital", "Smart", "Real") is the only word in Amber. Everything else is Ink (light bg) or Paper (dark bg).

### Section eyebrow chip
> `{LABEL}` — uppercase, +0.12em letter-spacing, 12px, in Amber with `Amber@8%` background. Pill, 6px x 12px padding.

### CTA verbs
Use, in order: **Get Started** (free) → **Start Free Trial** (after trial visible) → **Select Plan** (pricing context). Never "Sign Up Now," never "Click Here."

### Reassurance row format
> ✓ No setup fee &nbsp; ✓ Cancel anytime &nbsp; ✓ Works on mobile

Always green check, always three items, never more.

---

## 9. Don'ts

- ❌ Gradient backgrounds across full sections. (One gradient max, in the final CTA banner.)
- ❌ Emoji in product UI. (Only on landing reassurance row, where ✓ reads as a checkmark.)
- ❌ Stock photos of generic mechanics giving thumbs up.
- ❌ "Powered by AI" copy in Phase 1.
- ❌ Comparison tables that bash competitors by name.
- ❌ Testimonials with fake names. (Use real ones or leave the section out until Phase 2.)

---

*BayWork — Where the bay work gets done.*
*Brand v1.0 · May 2026 · Maintained by Design.*

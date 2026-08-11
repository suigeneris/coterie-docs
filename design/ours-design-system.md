# Ours Design System

> A calm, premium design language for co-owning luxury items.

**Status:** reconciled 2026-08-10. The 14 image plates in `assets/` are the source of truth;
this document has been rewritten to match them. Where the plates were silent, incomplete or
demonstrably wrong, the resolution is recorded inline under **Reconciliation**.

**Naming:** *Ours* is the design system's name. *Coterie* is the engineering codename used
throughout the repositories (`coterie-ios`, `coterie-backend`, `coterie-docs`, bundle ID
`com.coterie.app`). The final product name is undecided. Nothing in this document depends on
which is chosen.

**Provenance:** the plates are stamped *Version 1.0, May 2025*. The previous markdown
(`ours-design-system-v2-ORIGINAL.md`, retained for reference) was a later expansion whose
colour and type values diverged from the plates. On a direct conflict, **the plates win**.

---

## Contents

1. Colour · 2. Typography · 3. Spacing & Layout · 4. Shape / Corner Radius · 5. Buttons ·
6. Cards & Surfaces · 7. Elevation & Shadows · 8. Iconography · 9. Imagery ·
10. Navigation · 11. Forms & Inputs · 12. Motion · 13. Accessibility · 14. States & Feedback

---

## 01 Colour

A calm, confident palette inspired by heritage, nature and timeless luxury.

### Brand

| Token | Hex | Role |
|---|---|---|
| `forest` | `#0F3D34` | Primary brand colour. Key actions, active states, brand expression |
| `cream` | `#F6F3EE` | App background and soft warm surfaces |
| `lavender` | `#E9E6FF` | Highlights, feature surfaces and moments of delight. Used sparingly |
| `yellow` | `#F2E074` | Attention badges and small accents |
| `periwinkle` | `#C9D3F3` | Secondary feature surface, paired with `lavender` in layered compositions |

### Neutrals

| Token | Hex | Use |
|---|---|---|
| `ink` | `#111111` | Primary text |
| `slate` | `#4B4B4F` | Secondary text |
| `stone` | `#6E6E6E` | Tertiary text, metadata, placeholders |
| `mist` | `#E5E5E7` | Borders and dividers |

### Surfaces

| Token | Hex | Use |
|---|---|---|
| `surfaceBase` | `#FFFFFF` | Cards, sheets, navigation bar |
| `surfaceAlt` | `#F6F3EE` | Grouped and secondary backgrounds (same value as `cream`) |
| `backgroundPrimary` | `#F6F3EE` | Main app background |

### Semantic

| Token | Hex | Use |
|---|---|---|
| `success` | `#1E7A5E` | Success and confirmation |
| `warning` | `#8A6410` | Warning and caution |
| `error` | `#C62828` | Errors and destructive actions |
| `info` | `#507582` | Neutral informational messages |

### Usage

- **Forest is the primary brand colour** — key actions, active states, brand expression.
- **Neutrals build hierarchy and ensure clarity.** Keep surfaces light and calm; let content
  and imagery shine.
- **Lavender is used sparingly**, for highlights and moments of delight.
- **Yellow is for attention badges only** — availability counts, "Popular", "Limited".
- **Do not use forest as the semantic success colour.** They are different jobs.

### Contrast

Every pairing below is computed, not estimated. Body text requires **4.5:1**, large text and
UI components **3:1**.

| Pairing | Ratio |
|---|---|
| `ink` on `cream` / `surfaceBase` | 17.06 / 18.88 |
| `slate` on `cream` / `surfaceBase` | 7.85 / 8.68 |
| `stone` on `cream` / `surfaceBase` | 4.61 / 5.10 |
| `forest` on `cream` / `surfaceBase` | 10.92 / 12.09 |
| `white` on `forest` | 12.09 |
| `ink` on `lavender` / `yellow` | 15.49 / 14.11 |
| `forest` on `lavender` / `yellow` | 9.92 / 9.03 |
| `success` / `warning` / `error` / `info` on `cream` | 4.74 / 4.85 / 5.08 / 4.51 |

**Lavender, periwinkle and yellow are fill colours only.** Yellow measures **1.34:1** against white — it
can never be text, an icon, or a thin line. Always place `ink` or `forest` on top of them.

> **Reconciliation.** Three plate tokens failed the WCAG AA commitment made in plate 13 and
> have been darkened along their own hue to the minimum passing value against `cream`:
> `stone` `#A3A3A3` → **`#6E6E6E`** (was 2.28:1 on cream), `warning` `#D4A017` → **`#8A6410`**
> (was 2.38:1 on white), `info` `#557C8A` → **`#507582`** (was 4.09:1 on cream). `info` at
> 4.51:1 is marginal; darken further if you want headroom.
>
> **Yellow** is used in plates 03 and 14 but never named or given a hex; `#F2E074` was
> supplied directly. **Periwinkle** was dropped during reconciliation because no plate named
> it, then **reinstated 2026-08-11** at `#C9D3F3` — the approved onboarding design pairs it
> with `lavender` in the layered card stack. Fill only, and never used to distinguish a state
> from `lavender`: they measure 1.02:1 against each other.
>
> **Do not sample colours from the plate images.** They are downscaled screenshots and every
> labelled swatch disagrees with its own pixels: forest `#0F3D34` samples as `#0B4335`, error
> `#C62828` as `#B61E25`, lavender `#E9E6FF` as `#D8C7FA`. The labels are the design intent.
>
> **Greyscale ramp.** Plate 01 carries a 900→0 ramp
> (`#0D0D0D`, `#1F1F1F`, `#2E2E2E`, `#4B4B4F`, `#6E6E73`, `#8F8F94`, `#B3B3B8`, `#D1D1D6`,
> `#E5E5E7`, ?, `#F2F2F4`, `#F7F7F8`, `#FFFFFF`). The `50` step could not be read reliably and
> is omitted. The named neutrals above are what the UI should use; the ramp is reference only.

![Colour](assets/01-colour.png)

---

## 02 Typography

**Instrument Serif + Inter.** Instrument Serif brings character and elegance; Inter ensures
clarity and functionality. Both are open-source, so there is no licensing cost.

Instrument Serif is for headlines, feature titles and editorial moments. Inter is for body
text, UI, labels, buttons and all functional content. **The impact of the serif comes from
its contrast with Inter — use it selectively.**

### Type scale (iOS)

| Style | Font | Weight | Size | Line height | Use |
|---|---|---|---|---|---|
| `display` | Instrument Serif | Regular | 40 | 48 | Hero headlines |
| `h1` | Instrument Serif | Regular | 32 | 40 | Page and section titles |
| `h2` | Instrument Serif | Regular | 24 | 32 | Sub-section titles |
| `h3` | Inter | Semibold | 20 | 28 | Card titles, strong labels |
| `body-lg` | Inter | Regular | 17 | 24 | Body (large) |
| `body` | Inter | Regular | 15 | 22 | Body (default) |
| `body-sm` | Inter | Regular | 13 | 18 | Supporting text, labels |
| `caption` | Inter | Regular | 11 | 16 | Captions, metadata |
| `button` | Inter | Semibold | 16 | — | Button labels |

### Usage

- Keep headlines short and impactful.
- **Use sentence case.** Avoid all caps except for small labels.
- Maintain strong contrast for accessibility.
- All functional typography must support **Dynamic Type**.

> **Reconciliation.** The plate omits a `button` token, but plate 05 specifies button labels
> as Inter Semibold 16pt — recorded above. Labels use `body-sm` (13pt); no separate token.
> The plate's `display` size reads `40 / 44`; plate 03 shows onboarding headlines at
> **40–48pt**, so treat 40 as the token and 48 as the permitted hero maximum.

![Typography](assets/02-typography.png)

---

## 03 Spacing & Layout

A **4pt base grid**, with **8pt increments as the default rhythm**.

| Token | Value | Use |
|---|---|---|
| `space2xs` | 4pt | Tiny gaps, icon spacing, hairlines |
| `spaceXs` | 8pt | Closely related elements, small gaps |
| `spaceSm` | 12pt | Label to supporting text, tight grouping |
| `spaceMd` | 16pt | Standard component padding, default gaps |
| `spaceLg` | 24pt | Groups of related content, card padding |
| `spaceXl` | 32pt | Section spacing |
| `space2xl` | 48pt | Large section separation |
| `space3xl` | 64pt | Hero and editorial spacing |

**8, 16, 24 and 32pt cover 80%+ of the UI.** When in doubt, round to the nearest token.

### Screen margins

| Layout | Margin |
|---|---|
| Dense / utility | 16pt |
| Standard (default) | 24pt |
| Editorial / onboarding | 24pt, with 32–48pt between sections |

### Vertical rhythm

| From → to | Space |
|---|---|
| Heading → content | 16pt |
| Section → next section | 32pt |
| Major editorial separation | 48pt |

### Component spacing

- **Button** — 56pt height, 24pt horizontal padding, radius 16pt
- **Card** — 16–24pt internal padding
- **Avatar group** — 8pt between avatars, trailing `+` affordance

### Onboarding screen spec

Taken directly from plate 03 and used by the intro carousel:

| Element | Value |
|---|---|
| Screen margins | 24pt |
| Top → headline | 40–48pt |
| Headline → body | 16pt |
| Body → card | 32pt |
| Card → page dots | 32pt |
| Page dots → primary button | 24pt |
| Button → bottom safe area | 24pt |
| Primary button | 56pt tall, full width |
| `Skip` | top-right, tertiary style |

**Onboarding copy** (plate 03, supersedes the product spec's carousel copy):

> **luxury, together**
> Access the pieces you love by co-owning with people you trust.

![Spacing and layout](assets/03-spacing-layout.png)

---

## 04 Shape / Corner Radius

| Token | Value | Use |
|---|---|---|
| `radiusXs` | 8pt | Small elements, inputs, chips |
| `radiusSm` | 12pt | Buttons, small cards, alerts |
| `radiusMd` | 16pt | Standard cards, inputs, sheets |
| `radiusLg` | 20pt | Large cards, modals |
| `radiusXl` | 28pt | Hero cards, onboarding cards |

Use `radiusMd` (16pt) for the majority of surfaces. Reserve larger radii for elevated or
expressive moments where it adds impact.

![Corner radius](assets/04-corner-radius.png)

---

## 05 Buttons

All buttons: **56pt height, `radiusMd` (16pt), 24pt horizontal padding, Inter Semibold 16pt**,
minimum touch target **44×44pt**. Every type defines **default, pressed, disabled and loading**.

| Type | Default | Pressed | Use |
|---|---|---|---|
| **Primary** | `forest` fill, white label | Darker fill | The main action on a screen |
| **Secondary** | Transparent, `forest` outline and label | Outline thickens | Important but not primary |
| **Tertiary / text** | No container, label + chevron | Label darkens | Optional actions and links |
| **Destructive** | Pale red fill, `error` label | Solid `error` fill, white label | Irreversible or removing actions |

Loading state replaces the label with a spinner and keeps the container.
Disabled uses `disabled`/`mist` fill with `stone` label.

**Avoid multiple primary buttons in the same decision group.**

### Icon buttons (square)

| Size | Use |
|---|---|
| 32pt | Inline actions — favourite, share, bookmark |
| 40pt | Prominent actions |
| 56×56pt | Floating action |

![Buttons](assets/05-buttons.png)

---

## 06 Cards & Surfaces

All cards use `radiusMd` (16pt) unless stated. Internal padding 16–24pt. Prefer clean
surfaces over heavy borders, and don't stack card layers when spacing alone creates hierarchy.

| Card | Contents |
|---|---|
| **Product (vertical)** | Image, title (serif), ownership line, avatar group, price in `forest`, availability pill, favourite action |
| **Collection (horizontal)** | Image, name, item count and co-owner count, avatar group, chevron |
| **Ownership** | `lavender` surface, ownership fraction in a circle, status, next payment, chevron |
| **Information** | Quiet neutral surface, icon, title, supporting detail, chevron |

Ownership cards are the one place brand colour carries a whole surface. Information cards stay
visually quiet.

![Cards and surfaces](assets/06-cards-surfaces.png)

---

## 07 Elevation & Shadows

Mostly flat. Elevation exists only to clarify hierarchy.

| Token | Shadow | Border | Use |
|---|---|---|---|
| `elevationNone` | none | `mist` 1pt | Backgrounds, sections, dividers |
| `elevationSm` | `0 2px 8px rgba(0,0,0,0.06)` | `mist` 1pt | Product cards, inputs, small controls |
| `elevationMd` | `0 8px 24px rgba(0,0,0,0.10)` | none | Sheets, modals, dropdowns, overlays |

- Use the lowest elevation that achieves clarity.
- Prefer `elevationSm` for cards and interactive controls.
- Reserve `elevationMd` for overlays, sheets and floating elements.
- Avoid stacking multiple elevated elements.
- **Never use shadow as the primary visual style.**
- In dark mode, shadows remain subtle and reduce in intensity.

> **Reconciliation.** Plate 06 lists the card shadow as `0 4pt 16pt rgba(0,0,0,0.06)`, which
> conflicts with plate 07's `elevationSm`. Plate 07 wins.

![Elevation and shadows](assets/07-elevation-shadows.png)

---

## 08 Iconography

**SF Symbols** is the primary icon set on iOS.

| Style | Use |
|---|---|
| Outlined | Default |
| Filled | Selected / active state |
| Duotone | Special cases only |

| Size | Use |
|---|---|
| 16pt | Very small supporting indicators |
| 20pt | Small inline icons in text |
| 24pt | **Default** standard UI icon |
| 32pt | Feature or prominent use |
| 44pt+ | Minimum interactive touch area, not glyph size |

Use `forest` for active and selected states, `ink` for normal functional icons. Icons support
meaning — they should not become decoration.

**Common set (24pt):** home, search, heart, bag, user, settings, message, notification, share,
bookmark, calendar, card, lock, check, plus, minus, close, chevron right, filter, more.

![Iconography](assets/08-iconography.png)

---

## 09 Imagery & Product Photography

Great imagery builds trust and desirability. Photography is clean, consistent, and lets the
product shine.

### Types

- **Product** — clean, studio-style, accurate colour, full focus on the item, minimal props.
  Used on product pages, cards, search and listings.
- **Lifestyle** — editorial and aspirational, warm natural tones, human or environmental
  context, never over-styled. Used for collections, discovery and brand content.

### Aspect ratios

| Ratio | Use |
|---|---|
| `1:1` | Product (default) — grids, cards, gallery thumbnails |
| `4:5` | Editorial — collection cards, feature stories |
| `16:9` | Hero / banner — home hero, promotions |
| `9:16` | Stories / full-screen — onboarding, immersive content |

### Cropping

Keep products centred with balanced breathing room. **The product should occupy roughly
60–70% of the frame.** Avoid tight crops and cutting off important edges.

### Backgrounds

White (default) · cream (soft warm alternative) · light stone (subtle texture acceptable) ·
lavender (campaigns and branded moments only) · lifestyle (editorial only).

Keep backgrounds clean and consistent across similar content. Avoid busy patterns, harsh
shadows and strong colour casts.

### Overlays

Favourite action top-right with a 40×40pt tap target · semantic badges · gradient behind text
overlays for readability · clear, concise price labels. Avoid clutter.

### Galleries

First image is the hero. Include key angles — front, back, side, detail, interior. Keep
lighting and background consistent. Tap thumbnail to switch.

### Quality

Minimum **2000px on the long edge**. True-to-life colour, soft even lighting, no heavy
filters, no distortion, no watermarks or added text.

### User uploads

Allow JPG, PNG and HEIC. Provide crop guidance and aspect options. Validate quality,
orientation and inappropriate content. Optimise file size for performance.

![Imagery and product photography](assets/09-imagery-product-photography.png)

---

## 10 Navigation

Familiar iOS patterns with Ours styling. Navigation should feel invisible.

### App shell

| Pattern | Use |
|---|---|
| **Bottom tab bar** | Primary navigation |
| **Navigation bar** | In-page hierarchy |
| **Sheet** | Additional or contextual options |
| **Full screen** | Immersive or multi-step flows |
| **Modal** | Focused tasks and decisions |

### Bottom tab bar

- **5 tabs maximum.** The centre action is for primary creation or add flows.
- Active tab uses `forest` and a filled icon.
- Minimum touch target 44×44pt.
- Hidden during full-screen flows.

> The plate's example tabs (home · collection · **+** · deals · profile) are illustrative
> only. The product's real information architecture is defined by the product spec, not here.

### Navigation bar

Title in sentence case, centred. Height 44pt. Background `surfaceBase`. **Back** for
hierarchy, **Close (X)** for modals and full-screen. Contextual actions trailing.

### Sheets

| Size | Height |
|---|---|
| Small | up to ~25% |
| Medium | up to ~60% |
| Large | up to ~90% |

Peek height by default, dismiss always visible, contextual actions only. **Avoid deep
hierarchies and nested sheets.**

### Full-screen flows

Hide the bottom tab bar. Close (X) to exit. Keep content focused. **Show progress steps for
multi-step flows** — "1 of 3".

### Back & close patterns

| Pattern | Use |
|---|---|
| Back (push) | Going back one level in the hierarchy |
| Close (X) | Dismiss modal or full-screen, return to previous context |
| Dismiss (drag handle) | Swipe down or tap outside to dismiss sheets |
| Back to top | Long lists |

### Hierarchy

App shell (always visible) → **Section level** (top-level areas) → **List level** (browse and
find) → **Detail level** (view and act on specific items).

![Navigation](assets/10-navigation.png)

---

## 11 Forms & Inputs

Clear, minimal and easy to complete.

### Text fields

States: **default · focused · filled · disabled · error · success**. Focused uses a lavender
border. Error uses an `error` border with a message beneath. Success shows a check and a short
confirmation. Support leading icons and trailing icon buttons such as password reveal.

Helper text sits below the field. Show a character count where a limit applies.

### Search

Familiar search field with optional clear action. Used for global search and filtering.

### Selection controls

Radio buttons · checkboxes · single-select chips · multi-select chips · segmented controls ·
toggles. Selected chips use `forest` fill with a white label.

### Validation

- Prefer **inline validation close to the affected field**.
- Explain what is wrong and how to fix it.
- Support field-level and form-level errors.
- **Never rely on colour alone** — pair with icon and text.

### File upload

Dashed lavender drop zone. State supported formats and size limit. Show the uploaded file with
name, size and a remove action.

### Layout

Single column on mobile · group related fields · clear labels above fields · helpful
placeholder and helper text · primary action at the end · keep it scannable.

**Don't** use ambiguous labels, hide important actions, rely on placeholders as labels, or
collect unnecessary data.

![Forms and inputs](assets/11-forms-inputs.png)

---

## 12 Motion & Animation

Subtle, purposeful and consistent. Motion communicates change, guides attention and reinforces
the premium feel.

| Token | Duration | Use |
|---|---|---|
| `motionQuick` | ~150ms | Micro-interactions and feedback |
| `motionStandard` | ~250ms | Most UI changes and transitions |
| `motionExpressive` | ~400ms | Emphasis and complex motion |

Use standard iOS easing curves — ease in, ease out, ease in-out.

### Specified interactions

| Interaction | Motion |
|---|---|
| Button press | Scale to 0.98 · 150ms · ease out |
| Icon tap (heart) | Scale + fade · 150ms · ease out |
| Toggle switch | Thumb slides · 200ms · ease out |
| Card lift | Lift + shadow · 250ms · ease out |
| Push / pop navigation | Slide · 250ms · ease out / ease in |
| Tab switch | Icon scale + fade · 150ms |
| Modal / sheet | Slide up + fade · 300ms · ease out |
| Skeleton loading | Fade in · 250ms · ease in |
| Progress bar fill | 250ms · linear |
| Success check | Scale in + fade · 300ms · ease out |
| Toast | Slide up + fade · 250ms, auto-dismiss 2.5s |
| Confetti (milestone) | Burst + fade · 400ms · ease out |
| Favourite / save | Scale + burst · 250ms · ease out |
| Ownership progress | Arc draw · 300ms · ease out |
| Image transition | Crossfade · 250ms · ease in-out |

### Do not animate

Auto-playing content · backgrounds or decorative elements · layout changes without user
action · too many elements at once · anything longer than necessary.

### Reduce Motion

When enabled: instant state changes, no parallax or complex effects, short crossfades only,
and essential feedback preserved.

![Motion and animation](assets/12-motion-animation.png)

---

## 13 Accessibility

> **Luxury should never come at the expense of accessibility.**

- **Dynamic Type** supported up to XXXL. Layouts adapt without loss of content or
  functionality. No clipped text; containers expand to fit.
- **Touch targets** minimum 44×44pt for all interactive elements.
- **Contrast** meets WCAG 2.1 AA — 4.5:1 normal text, 3:1 large text and UI.
- **Never communicate information by colour alone** — pair with icon, text or shape.
- **VoiceOver** — clear labels, values and hints; logical reading order; complex components
  grouped and navigable in order. Product images get meaningful descriptions; decorative
  images are hidden.
- **Forms** — labels, hints and errors are announced and connected to their field.
- **Focus management** — focus moves to the right element in modals, sheets and errors, and is
  trapped within a presented sheet until dismissed.
- **Respect** Reduce Motion, Increase Contrast, Reduce Transparency and Bold Text.
- **Haptics** supplement feedback and never replace it — light for selection, medium for
  success, heavy for error, soft for notification.
- **Language** — simple and clear. "Invite co-owners to share this item", not "utilise shared
  ownership mechanisms".

> **Reconciliation.** The contrast figures printed on plate 13 are incorrect — none of the four
> matches a computed value (claimed 14.6 / 7.3 / 8.2 / 4.7; actual 12.09 / 15.49 / 14.77 /
> 5.62). The verified table in §01 supersedes them.

![Accessibility](assets/13-accessibility.png)

---

## 14 States & Feedback

Use the **least disruptive feedback that communicates the result clearly**.

### Feedback hierarchy

1. UI change · 2. Inline message · 3. Toast / banner · 4. Dialog · 5. Full screen

### Loading

**Skeletons** for structured content — lists, cards, galleries. **Spinners** for short isolated
actions. **Blur load** for single image transitions. Don't block the screen unnecessarily.

### Empty

Short editorial heading, concise explanation, one clear action. Keep illustrations light and
supportive.

### Error

Explain what happened and what to do next. Prefer contextual errors over full-screen. Offer a
retry. Pair colour with icon and text.

### Offline

Keep cached content visible, indicate it may be out of date, and provide retry when an action
requires connectivity.

### Success

Prefer subtle UI updates, toasts, checkmarks or haptics. **Reserve full-screen success for
meaningful milestones** such as becoming a co-owner — where a more expressive treatment,
including confetti, is permitted.

### Confirmation

**Do not confirm routine or reversible actions.** Use dialogs only for destructive, expensive
or irreversible decisions.

### Destructive

Clearly state the consequence. Use specific labels — **Remove item**, **Leave collection** —
never Yes/No. Present as an action sheet where appropriate.

### Progress

Multi-step flows show step progress — "2 of 4". Task progress uses a linear bar with a label.

### Badges

Short and semantic. Availability — `1/4 available`, `2/4 available`, `Sold out`.
Status — `Verified`, `New`, `Sold`, `Archived`. Highlights — `Popular`, `Limited`, `Featured`.
Yellow supports attention-oriented badges; semantic states use semantic colours.

### Payment & transaction

Keep the user informed throughout — authorising, successful, failed with a retry. Payment is
the highest-stakes flow in the product and gets the clearest possible messaging.

![States and feedback](assets/14-states-feedback.png)

---

## Principles

- Keep the interface **calm, spacious and product-led**.
- Use brand colour **intentionally**, not evenly.
- **Prefer native iOS behaviour** when custom behaviour adds no meaningful value.
- Maintain **consistent tokens** rather than one-off values.
- Use the **least visual weight** necessary to establish hierarchy.
- Keep premium aesthetics **compatible with accessibility**.
- Treat this as a **living foundation** — extend it only when repeated product needs justify a
  new token, component or pattern.

---

## Open questions

1. **Greyscale `50` step** — could not be read reliably from plate 01. Confirm the value or
   confirm the ramp is reference-only and can be dropped.
2. **`info` at 4.51:1 on `cream`** is the computed minimum and leaves no headroom. Darken
   further, or restrict `info` text to white surfaces.
3. ~~Dark mode~~ — **decided 2026-08-11: the app is light-only.** No dark palette will be
   defined for now. The app sets `UIUserInterfaceStyle: Light`, so it renders light even on a
   device in dark mode. Without that lock the tokens would stay light while system chrome —
   text fields, sheets, the keyboard, default label colours — flipped dark, giving a mixed
   result worse than either mode. Plate 07's note about dark-mode shadows is therefore
   dormant, not lost. Revisit before public launch: light-only is increasingly conspicuous.
4. **Onboarding copy** — plate 03's *"luxury, together"* now supersedes the three-slide
   carousel copy in the product spec §6. The product spec needs updating to match, or a
   decision that the plate copy applies only to the first slide.

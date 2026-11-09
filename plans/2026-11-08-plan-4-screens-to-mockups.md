# Coterie Plan 4 — Bring the coterie screens to the final mock-ups

> **For agentic workers:** work task by task, in order. Each task lists what changes from plan 3, the exact copy, and a checklist. The screenshots in `design/mockups/plan-4/` are the visual reference; this document wins where a screenshot and the text disagree. Do not take colours, sizes or copy from the HTML canvas: use Ours tokens and the strings below.

**Goal:** the Discover, Coterie detail, buy, create and Coteries screens built in plan 3 (Tasks 9–11) match the final mock-ups approved on 2026-11-08, using the redesigned `OursOwnershipCard` already merged in `coterie-design` (#8).

**Architecture:** no new data, tables or repository methods. Every field used here already exists on `Pool` and `PoolOwner`. The work is presentation: one small package task (Task 1) for two missing components, then screen changes on top of plan 3's views and view models. The view models, routing (`AfterBuy`), verification flow and photo handling from plan 3 stay as they are unless a task says otherwise.

**Reference:**
- Screenshots: `design/mockups/plan-4/01-…` to `23-…` (iPhone 402×874pt, exported at 2×).
- Design doc: `design/ours-design-system.md`, especially §05 (compact button), §06 (ownership card) and §14 (status in words).
- Live canvas, for people rather than agents: "Coterie: Join & Create" on claude.ai.

---

## Decisions this plan rests on

Made with the user while reviewing the mock-ups (2026-11-07 and 2026-11-08).

| Decision | Result here |
|---|---|
| No funding deadline, no term, no day count | No deadline, "12 months" or "121 days" anywhere. "Your use: Equal time on a shared calendar" stays |
| Funding copy | "N of M filled" on Discover cards, the detail screen and "You're in". Never "committed" |
| New items only | No condition field in the wizard |
| Size badge | A new, larger size badge: a yellow strip attached under the thumbnail on Discover cards (never laid over the photo, which real products fill), and beside the name on the detail screen (Task 1) |
| Ownership card | `OursOwnershipCard` as merged in `coterie-design` #8: share ring, one status line in words, co-owner avatars, no badges, no money |
| Badges | Status in words; a badge only asks for action or flags a problem. The detail screen keeps its "Full" and "Your share" badges as mocked |
| Item price | Shown as "Item price £2,000" wherever the total appears: Discover cards, detail, buy sheet, review. Never "RRP": the creator types it unchecked, and an RRP label implies a reference price (CMA and ASA rules) |
| Payment | Not built. Every place payment will go shows the dashed placeholder box (Task 5) |
| Dark mode | None. The app stays locked to light |

## Global constraints

- Ours tokens and components only. Any value not given here comes from a token named in the design doc, never a literal.
- British English and £. The group is a "coterie" in every string; never "pool" in UI copy.
- Every interactive element is at least 44×44pt. Every decorative image and icon is hidden from VoiceOver; every icon-only button has a label.
- `make generate` after adding or removing files. `make test` and `make lint` must pass before each commit.
- Run `make refresh-design` once before starting: Task 1 and the ownership card live in the package.

## Review focus

- The copy matches this document exactly, including punctuation ("Full · we're buying it" uses a middle dot).
- Nothing in plan 3's buy, leave or verification logic is lost while the screens change around it (Tasks 4 and 5).
- The ownership card's `onOpen` and its Book action are separate targets.

---

### Task 1: Size badge and chip sizes (`coterie-design`)

**Why:** the mock-ups use two things the package doesn't have: the larger shoe size badge, and the bigger chips in the create wizard.

**Files (in `coterie-design`):**
- Create: `Sources/OursDesignSystem/Components/OursSizeBadge.swift`
- Modify: `Sources/OursDesignSystem/Components/OursChip.swift`
- Test: `Tests/OursDesignSystemTests/ComponentTests.swift`

**Interfaces:**

```swift
public struct OursSizeBadge: View {
    public enum Emphasis: Sendable { case strip, detail }
    public init(_ size: String, emphasis: Emphasis = .strip)   // "UK 5"
}

public enum OursChipSize: Sendable { case regular, large, tile }
// OursChip(_:systemImage:isSelected:size:action:) gains `size: OursChipSize = .regular`.
```

**Spec:**
- **Size badge:** `yellow` fill, `ink` text (14.1:1), `radiusXs`. A "SIZE" label in Inter Semibold 11, uppercase, tracking +0.06em, then the value in Inter Bold. `.strip`: 17pt, full width of its container, 4pt top and bottom, centred, square corners: it sits under a thumbnail and the two share one tile (below). `.detail`: 24pt, padding 8×14, `radiusXs`. VoiceOver reads "Size UK 5".
- **Chip `.large`:** 56pt tall, value in `h3`, equal widths when placed in a grid. Used for the number of owners.
- **Chip `.tile`:** 128pt tall, `radiusMd`, a 32pt symbol above the label in `button` style. Used for the category.
- All three chip sizes keep `OursChip`'s colours: resting `surfaceBase` with a 1pt `outline` border, selected `forest` with a white label.

- [ ] Write failing tests: the size badge's text pair meets 4.5:1; `.large` and `.tile` chips are at least 44pt tall; selected and unselected colours are the same at every size.
- [ ] Run `xcodebuild test -scheme OursDesignSystem -destination 'platform=iOS Simulator,name=iPhone 17 Pro'` and see them fail.
- [ ] Implement. No existing call site changes: both additions default to today's behaviour.
- [ ] Run the tests. Expected: all pass. CI runs them on the PR.
- [ ] Commit: `feat: size badge and larger chip sizes`. Open the PR, merge it, then `make refresh-design` in `ios`.

### Task 2: Discover

**Screenshots:** 01 (loaded), 02 (loading), 03 (empty), 04 (failed).

**Changes from plan 3, Task 9:**
- **Header:** title "Discover" (`h1`, `forest`) as before, now with a subtitle under it in `body` `slate`: "Co-own a bag or a pair of shoes with up to three other people." Shown in every state except failed.
- **Card:** keep `OursProductCard`'s layout (112pt photo left, brand, serif name, price), with these changes:
  - Price reads `£666.67` in `body-lg-strong` `forest`, followed by "per share" in `body-sm` `slate`. Under it, in `body-sm` `stone`: "Item price £2,000" (`targetAmount`, formatted without pence when whole).
  - The size badge moves off the footer and **under the photo**: a 112pt-wide column of the photo then `OursSizeBadge(size, emphasis: .strip)`, clipped together to one `radiusSm` tile with a 1pt `mist` hairline, so photo and strip read as one piece. It never overlaps the photo. Shoe cards are about 32pt taller than bag cards; that's expected. Bags show none.
  - The footer is one row and a bar: "N of M filled" (`body-sm` `slate`) on the left, `OursAvatarGroup(people:, showsOpenPlace: !isFull)` on the right, then the 8pt funding track full width underneath. Build it with `OursFundingBar`'s track and the label row above it.
- **End of list:** after the last card, centred, `body-sm` `stone`: "That's every coterie open right now." Only when the list has loaded and isn't empty.
- **Empty:** symbol `bag` in the 72pt `lavenderDeep` disc. Title "Nothing to join yet" (`h2`). Body "Be the first. Start a coterie for a bag or a pair of shoes you love, and others can join you." Primary "Start a coterie" calls `startCreating()`.
- **Failed:** unchanged (`OursErrorScreen`). Copy: title "Something went wrong", message "We couldn't load coteries. Check your connection and try again.", button "Try again".
- **Loading:** unchanged: three skeleton cards in the card's shape.

- [ ] Update `PoolPresentationTests` for any changed strings, and see them fail.
- [ ] Implement.
- [ ] `make test`, `make lint`. Expected: pass.
- [ ] Capture against screenshots 01–04, plus a shoes card with a 20-character size and a 120-character name.
- [ ] Commit: `feat(pools): Discover matches the final mock-ups`

### Task 3: Coteries tab and the new ownership card

**Screenshots:** 21 (owned), 22 (empty).

**Changes from plan 3, Task 9:**
- **Header:** title "Coteries" (`h1`, `forest`) with the subtitle "Everything you co-own, and how each one is going." (`body` `slate`).
- **Row:** replace the old `OursOwnershipCard(fraction:title:detail:showsChevron:status:)` with the merged card:

```swift
OursOwnershipCard(
    title: pool.itemName,
    status: pool.ownershipStatus,            // below
    places: pool.ownerCount,
    filled: pool.slotsFilled,
    coOwners: pool.coOwners(excluding: currentUserID),
    onOpen: { coteriesPath.append(pool.id) },
    media: { PoolPhotoView(url: pool.photoURLs.first) }
)
```

  For `.active`, use the `action:` initialiser with `OursButton("Book", kind: .secondary, size: .compact, systemImage: "calendar") { coteriesPath.append(pool.id) }`. Booking isn't built, so for now Book opens the detail screen. The bookings plan replaces that.
- **Add to `PoolPresentation.swift`:**

```swift
extension Pool {
    var ownershipStatus: String { get }
    func coOwners(excluding userID: UUID) -> [OursAvatarGroup.Person]   // owners minus the viewer, in joinedAt order
}
```

  | `status` | `ownershipStatus` |
  |---|---|
  | `.funding` | "1 place left" or "N places left" (`ownerCount - slotsFilled`) |
  | `.funded` | "Full · we're buying it" |
  | `.active` | "Ready to book" |
  | `.closed` | "Closed" |
  | `.cancelled` | "Cancelled" |
  | `.unknown` | "" (the line is omitted) |

  The old `statusBadge` and `ownedFraction` are no longer used by this row. Delete them if nothing else uses them.
- **Empty:** symbol `person.2` in the 72pt `lavenderDeep` disc. Title "You're not in a coterie" (`h2`). Body "Join one from Discover, or start your own for something you'd love to share." Primary "Browse coteries" switches to Discover; secondary "Start a coterie" calls `startCreating()`.

- [ ] Write failing tests: `ownershipStatus` for every status, including 1 and 2 places left; `coOwners(excluding:)` drops the viewer, keeps order, and returns `[]` when the viewer is alone.
- [ ] Run `make test` and see them fail. Implement. Run `make test` and `make lint`.
- [ ] Capture against 21–22, plus a coterie where you're alone ("Just you so far"), co-owners with and without avatar URLs, and Accessibility Large.
- [ ] Commit: `feat(pools): Coteries uses the new ownership card`

### Task 4: Coterie detail

**Screenshots:** 05 (can buy, full scroll), 06 (shoes), 07 (full), 08 (owner), 09 (not yet verified).

**Layout, top to bottom (replaces plan 3's list):**
1. Hero `PoolPhotoView`, full width, 1:1. A 44pt floating back button over its top-left (white circle, mist hairline, `chevron.left`), in place of the navigation bar.
2. Thumbnail row if there's more than one photo: 64pt, `radiusSm`, 8pt apart. The selected one has a 2pt `forest` ring, the others a `mist` hairline. Tapping switches the hero.
3. `brand` in `body-sm` `stone`, then `itemName` in `h1`. For "Full", the "Full" (`success`) badge sits right-aligned on the brand row; for an owner, "Your share" (`feature`).
4. Shoes only: `OursSizeBadge(size, emphasis: .detail)` with "Every owner wears this size. Only join if it's yours." beside it in `body-sm` `slate`.
5. `pricePerShare` in `h3` `forest`, then "per share" in `body` `slate`. Under it, in `body-sm` `stone`: "Item price £2,000" (`targetAmount`).
6. Funding: "N of M filled" (`body-sm` `slate`) and the track.
7. **Co-owners** (`body-lg-strong`): one row per owner, a 32pt avatar and their `displayName` (`body`). The creator (`creatorID`) also gets "Started this coterie" in `body-sm` `stone`. Then one row per open place: the open-place avatar and "Open place" in `body` `stone`. Under the rows: a `checkmark.shield` icon in `forest` and "Everyone in a coterie has verified their identity." (`body-sm` `slate`).
8. **About this item** (`body-lg-strong`) and `description` in `body` `slate`, if not blank.
9. A white card (mist hairline, `radiusMd`, 16pt side padding) of fact rows: "Your share" → "£666.67 (1 of 3)"; "Your use" → "Equal time on a shared calendar"; "Per-use fee" → "£5 per booking" (from `perUseFee`); and plan 3's "Product page" row if `safeSourceURL` is set. Plan 3's "Item price" row is removed: point 5 says it.
10. Under the card, `body-sm` `slate`: "Each time you book the {bag|shoes}, £5 covers its cleaning and insurance." ("bag" or "shoes" from `category`.)
11. Info banner: "Nothing is charged today. Payment is taken once all N places are filled." Not shown when full.

**Action area** (white, a `mist` hairline on top, 24pt sides, in `.safeAreaInset(edge: .bottom)`):

| Viewer | Places left | Full |
|---|---|---|
| Has no share | Primary "Buy this share" | "All N places are taken." (`body` `slate`, centred) and secondary "Browse other coteries" (back to Discover) |
| Has a share | "You own 1 of N shares. Nothing is charged until the last place fills." (`body-sm` `slate`, centred), secondary "View in Coteries", and plan 3's tertiary "Leave this coterie" under it | "You own 1 of N shares." and secondary "View in Coteries" |

**Not yet verified:** the action area still shows "Buy this share". Tapping it while `primaryAction == .verifyFirst` presents a sheet (`.presentationDetents([.medium])`, drag indicator) instead of plan 3's inline card. The sheet has the 72pt `lavenderDeep` disc with `checkmark.shield` in `forest`, then:

| `VerifyPrompt` | Title (`h2`) | Body (`body` `slate`) | Primary | Tertiary |
|---|---|---|---|---|
| `.verify` | Verify your identity first | Everyone in a coterie verifies who they are, so you always know you're sharing with real people. It takes about two minutes with a photo ID. | Verify identity | Not now |
| `.retry` | Verify your identity first | (as above) | Try again | Not now |
| `.pending`, `.refused`, `.checking`, `.failed` | Plan 3's copy, unchanged | | Plan 3's button, if any | Not now |

"Verify identity" starts `verification.startVerification()` exactly as plan 3 does. "Not now" dismisses. Plan 3's reload-on-return logic is unchanged.

- [ ] Update `PoolPresentationTests` for the new strings (the item line, the fee line by category, the owner lines), and see them fail.
- [ ] Implement. Plan 3's `PoolDetailRoutingTests` must still pass unchanged.
- [ ] `make test`, `make lint`.
- [ ] Capture against 05–09, plus every verify prompt, one photo (no thumbnail row), no description, and an owner who can leave.
- [ ] Commit: `feat(pools): coterie detail matches the final mock-ups`

### Task 5: Buy sheet and "You're in"

**Screenshots:** 10 (confirm), 11 (working), 12 (failed), 13 (you're in), 14 (took the last place).

**Buy sheet (changes from plan 3, Task 10):**
- Keep the title, the item row (64pt photo, name, brand, size badge), and the fact rows, adding **"Item price"** (`targetAmount`) as the first row, above "Your share (1 of N)".
- **New: payment placeholder** before the banner. A box with a 1.5pt dashed `stone` border, `radiusMd`, white, a `creditcard` symbol, "Payment" (`body`, semibold) and "Card details go here in the payments release." (`body-sm` `slate`). Make it its own view, `PaymentPlaceholder(message:)`. The payments plan replaces it.
- Info banner copy: "Nothing is charged today. Your card is charged once all N places are filled."
- **Failed:** `actionError` shows as an error banner in place of the info banner. Copy for a network failure: "We couldn't add you to this coterie. Nothing was charged. Check your connection and try again." The primary becomes "Try again", running the same `buy()`.
- Working: unchanged (primary with `isLoading`).

**"You're in" (`JoinedScreen`), changes:**
- No X. The two buttons are the only ways out.
- A 72pt `forest` disc with a white `checkmark` (not `symbolMark`). Title "You're in" in `display` (`ink`, not `forest`).
- Body (`body-lg` `slate`): "You're a co-owner of the {brand} {itemName}. We'll tell you when the last place is filled."
- A white card: 64pt photo, `itemName` in `body-lg-strong`, "N of M filled" right-aligned, the funding track below.
- Under it, `body-sm` `stone`: "Nothing has been charged yet."
- Primary "Go to Coteries", **tertiary** (was secondary) "Keep browsing". Same routing as plan 3.
- **Filled variant:** body "You took the last place, so the {brand} {itemName} is now full. We'll buy it and let you know when it's ready to book." The card shows the "Full" badge instead of the count, with "N of N filled" under the full track, then `PaymentPlaceholder(message: "Payment confirmation goes here in the payments release: every owner's card is charged now.")` in place of the "Nothing has been charged" line.

- [ ] Implement. Plan 3's routing tests must pass unchanged.
- [ ] `make test`, `make lint`.
- [ ] Capture against 10–14.
- [ ] Commit: `feat(pools): buy sheet and You're in match the final mock-ups`

### Task 6: Create wizard

**Screenshots:** 15 (category), 16 (details, bag), 17 (details, shoes, size missing), 18 (price and owners), 19 (review), 20 (live).

**Frame (changes from plan 3, Task 11):** a 44pt row: back chevron (steps 2–4; empty on step 1), "N of 4" centred in `body-sm` `slate`, close X. Under it, the four 8pt segments (`OursStepProgress`'s track, without its own label). Step titles are `h1` (was `h2`). The pinned button sits on `cream` with a `mist` hairline above it. The discard dialog is unchanged.

| Step | Title | Subtitle |
|---|---|---|
| 1 | What are you sharing? | A coterie is for one bag or one pair of shoes. |
| 2 | Item details | Tell people what they'd be sharing. |
| 3 | Price and owners | Set what the item costs and how many people share it. |
| 4 | Review and publish | Check everything before others can join. |

**Step 1:** two `OursChip(size: .tile)` side by side, equal width: "Bag" (`bag`) and "Shoes" (the shoe symbol). Under them, an info icon and "Everyone in a shoes coterie wears the same size, so you'll set it in the next step." (`body-sm` `slate`).

**Step 2:**
- Photo tiles at 104pt, 12pt apart. The add tile reads "Add photo" with "N of 4" under it (photos added so far, for example "2 of 4"). The rest of plan 3's photo handling is unchanged.
- Under the row: a frame icon and "Square photos on a plain background work best, with the item filling about two-thirds of the frame." (`body-sm` `slate`).
- Fields, in this order:

| Label | Field |
|---|---|
| Item name | `OursTextField("e.g. Puzzle Bag")` |
| Brand | `OursTextField("e.g. Loewe")` |
| Size (shoes only) | `OursTextField("e.g. 5")` with a "UK" prefix |
| Description | `OursTextArea("What makes it special? Colour, material, size.", limit: 2000)` |
| Product link (optional) | Unchanged from plan 3 |

- Size error copy: "Add the size. Everyone in a shoes coterie wears the same one."

**Step 3:**
- "Item price": the field with a "£" prefix, and helper text "What it costs to buy new, including delivery." (`body-sm` `stone`).
- "Number of owners": three `OursChip(size: .large)` in an equal-width grid, 12pt apart.
- Share preview, on a white card with a mist hairline: "Each share" (`body-sm` `slate`), the amount in `h1` `forest`, then "£2,000 split 3 ways" (`body-sm` `stone`), adding ", rounded up to the penny" only when the split isn't exact. A hairline, then "You'll pay one share too: you're the first owner." (`body-sm` `slate`). With no price yet: "Enter a price to see each share".

**Step 4:**
- A white card: the first photo (64pt), `itemName` (`body-lg-strong`), "{brand} · N photos" (`body-sm` `stone`), and a tertiary "Edit" that goes back to step 2. Under it, fact rows: Category; Product link ("Added" or "Not added" in `stone`); Item price; Owners; Each share.
- A second white card: "Your share" (`body-lg-strong`) with "You're the first owner, 1 of N" under it, and the share in `h3` `forest` on the right.
- `PaymentPlaceholder(message: "Card details go here in the payments release. Nothing is charged until the coterie is full.")`.
- Primary "Publish coterie". `publishError` and publishing are unchanged.

**"Your coterie is live" (`CoterieLiveScreen`):** a 72pt `forest` disc with a white check. Title in `display` `ink`. Body (`body-lg` `slate`): "You're the first owner. Others can find it in Discover and join now." Then the **Discover card** for the new coterie (Task 2's layout), with a "You" avatar (white on `forest`) and the open place. Primary "View coterie"; tertiary "Done" closes the flow. The old `OursOwnershipCard` preview is removed.

- [ ] Update `CreatePoolViewModelTests` if the share-preview wording comes from the model, and see them fail.
- [ ] Implement.
- [ ] `make test`, `make lint`.
- [ ] Capture against 15–20, for bags and shoes, the size error, four photos, an exact and an inexact split, and the discard dialog.
- [ ] Commit: `feat(pools): create wizard matches the final mock-ups`

### Hand checks once Tasks 2–6 are in

Run against the local stack, as in plan 3.

- [ ] Each screen side by side with its screenshot at the same size: spacing, type, colours and copy match.
- [ ] Buy as an unverified user: the verify sheet appears, "Not now" closes it, and verifying returns you to a working "Buy this share".
- [ ] Join, take the last place, and see both "You're in" variants and the Coteries status change to "Full · we're buying it".
- [ ] Create a shoes coterie, hit the size error, fix it and publish. The live screen shows the Discover card.
- [ ] VoiceOver on Coteries reads each card as one sentence: "Puzzle Bag. 1 place left. With Amara O."
- [ ] Accessibility Large on every screen: nothing clipped, nothing overlapping.

---

## Out of scope

Booking (Book opens the detail screen until the bookings plan), payments (placeholders only), profile photos (avatars show initials when there's no photo), the vertical Discover card for grids, Discover filters and search, favourites, chat, sharing and deep links, and dark mode.

## Not yet decided

- What Book does once bookings exist: a calendar sheet, or a booking screen.
- Whether "Closed" and "Cancelled" coteries stay in the Coteries list, or move to a "Past" section.

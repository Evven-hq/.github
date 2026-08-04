# Evven — "Receipt / Passbook / Tab" Concepts
### Implementation notes

Reference build: `evven-concepts-v2.html`
Base tokens: pulled directly from `app/globals.css` (`--evven-*` custom properties) — nothing new introduced except where called out under **New tokens** per section.

---

## Shared foundations

| Token | Value | Used for |
|---|---|---|
| `--bg` | `#faf8f5` | page background |
| `--card` | `#fefefe` | card/paper surfaces |
| `--border` | `#c4bfb7` | hairlines, dashed rules |
| `--text` | `#1a1816` | primary ink |
| `--muted` | `#8b8480` | captions, labels |
| `--accent` | `#2d5a4f` | Evven deep green — ink, seals, stamps |
| `--accent2` | `#e8dcc8` | tan — tape, secondary fills |
| `--error` | `#c0392b` | debit / "you owe" |

Fonts: `Xanh Mono` (headings, not used further here), `Baskervville` italic (display serif — store name / account holder name), `JetBrains Mono` (all tabular figures, receipt/ledger body copy), `Inter` (buttons, UI chrome).

All three concepts are **full-width responsive sections**, not phone-locked — they reflow via `max-width` containers and a single breakpoint at `640px`, matching the rest of the app's `sm:` convention.

---

## 1. Receipt

**Concept:** the post-hero section reads like the itemized thermal-printer slip from a group meal.

**Structure:**
```
.receipt (max-width: 420px, centered)
  .r-store         → "Evven" wordmark + date, centered
  .r-meta          → order # / time, flex justify-between
  .r-div.thick     → 2px solid divider
  balance summary  → .r-total-row × 3 (owed / owe / NET)
  .r-div.thick
  itemized list    → .r-line × n (dotted leader between name and amount)
  .r-stamp         → rotated circular "SETTLED %" mark
  .r-barcode       → decorative bars
  .r-foot          → micro-copy
```

**Key CSS techniques:**

- **Torn/perforated edge** — a pseudo-element strip of repeated circles colored as the *page* background, sitting on top of the card's edge:
  ```css
  .receipt::after{
    content:''; position:absolute; left:0; right:0; bottom:-11px; height:12px;
    background-image: radial-gradient(circle at 10px 10px, var(--page-bg) 9px, transparent 10px);
    background-size: 20px 20px;
    background-repeat: repeat-x;
  }
  ```
  `::before` is the same, flipped with `transform: scaleY(-1)`, for the top edge. **This only works if the surrounding page background is a flat color** — if this ships on a page with texture/imagery behind it, swap to an SVG mask (`mask-image: url(zigzag.svg)`) instead.

- **Dotted price leader** — classic menu/receipt alignment trick:
  ```css
  .r-line{ display:flex; align-items:baseline; gap:6px; }
  .r-line .leader{ flex:1; border-bottom:1px dotted #b8b1a7; transform:translateY(-3px); }
  ```
  Name and amount are two children with the leader `<span>` between them soaking up remaining width.

- **Rubber stamp** — rotated circle, double ring, `mix-blend-mode: multiply` so it reads as ink pressed onto paper rather than a flat sticker:
  ```css
  .r-stamp{ border:2.5px solid var(--accent); border-radius:50%; transform:rotate(-14deg); mix-blend-mode:multiply; opacity:.85; }
  .r-stamp::before{ content:''; position:absolute; inset:5px; border:1px solid var(--accent); border-radius:50%; }
  ```

- **Barcode** — pure CSS, no image asset:
  ```css
  .r-barcode{ background: repeating-linear-gradient(90deg, var(--text) 0 2px, transparent 2px 5px, var(--text) 5px 6px, transparent 6px 9px, ...); }
  ```

**Data mapping:**
- `.r-total-row` × 3 → `analytics` derived: owed / owe / net (net = owed − owe)
- itemized list → `personalExpenses` (or merged group expenses), each row needs `title`, `group name`, `amount`, sign
- stamp % → settled ratio (settled settlements / total balances)

**React componentization:** `<ReceiptCard balance={} items={} settledPct={} />`, with `ReceiptLine` as a mapped sub-component. The stamp and barcode are presentational-only (no props needed beyond `settledPct`).

---

## 2. Passbook

**Concept:** an old bank passbook / ledger book — DATE · PARTICULARS · DEBIT · CREDIT columns, a red ledger margin, a wax-seal net balance.

**Structure:**
```
.passbook
  .pb-tabs       → month tabs (folder-tab look), active = filled accent, others = ghost
  .pb-body       → accent-green panel: account holder name + .seal (net balance medallion)
  .pb-ledger
    .pb-row-head → column headers, 2px bottom border
    .pb-rows     → relative wrapper, ::before = red margin rule
      .pb-row × n → grid: date | particulars | debit | credit
  .pb-foot       → tagline + "Settle up" CTA
```

**Key CSS techniques:**

- **Red ledger margin** — a single absolutely-positioned line behind all rows, not per-row, so it stays continuous:
  ```css
  .pb-rows{ position:relative; }
  .pb-rows::before{ content:''; position:absolute; left:22px; top:0; bottom:0; width:2px; background:#c9463f; opacity:.35; }
  ```

- **Column grid** — every row (including the header) shares one `grid-template-columns: 64px 1fr 74px 74px` so figures stay aligned regardless of content length. Debit/credit are never both filled in the same row (single-entry feel); empty side renders an em dash.

- **Wax-seal medallion:**
  ```css
  .seal{ border:2px solid rgba(255,255,255,.55); border-radius:50%;
    background: radial-gradient(circle at 35% 30%, rgba(255,255,255,.12), transparent 60%); }
  ```

- **Folder tabs** — plain `border-radius: 8px 8px 0 0` on siblings, active tab filled, others "ghost" (`background:#e3ddd0`).

**Data mapping:**
- `.seal .v` → net balance (`owed − owe`)
- each `.pb-row` → one expense/settlement: `date`, `particulars` (title + sub as group/payer), `debit` (you owe this) or `credit` (you're owed this) — mutually exclusive per row
- month tabs → group expenses by month, most recent = active

**React componentization:** `<Passbook accountName={} netBalance={} months={} rows={} activeMonth={} onMonthChange={} />`. The debit/credit split logic (which column a row's amount lands in) belongs in a small mapper: `toLedgerRow(expense, currentUserId)`.

---

## 3. Tab (pinboard)

**Concept:** a corkboard of pinned IOU notes, one per group — casual, tactile, "who owes who" whiteboard energy.

**Structure:**
```
.board (dotted corkboard texture background)
  .board-title    → handwritten-style label
  .board-grid     → 2-col grid ≥640px, 1-col below
    .note × n
      .pin / .washi → decorative anchor (alternate per note)
      .who          → group name, handwriting font
      .what         → member count / description
      .tally        → SVG tally marks + expense count
      .owe-line     → amount (color-coded) + action button
```

**Key CSS techniques:**

- **Torn/pinned paper feel** — each `.note` gets a small fixed rotation via `:nth-child`, alternating sign so the board doesn't look uniform:
  ```css
  .note:nth-child(1){ transform:rotate(-2deg); }
  .note:nth-child(2){ transform:rotate(1.5deg); }
  ```
  In production this should be **randomized per-item with a seeded hash of the group id**, not hardcoded by position, so it's stable across re-renders but still feels organic.

- **Pin** — radial gradient for a glossy pushpin head:
  ```css
  .pin{ background: radial-gradient(circle at 35% 30%, #e7857a, #b5433c); box-shadow:0 3px 5px rgba(0,0,0,.35); }
  ```

- **Washi tape** — diagonal repeating stripe, used on alternating notes instead of a pin:
  ```css
  .washi{ background: repeating-linear-gradient(45deg, #e6c9a0, #e6c9a0 4px, #dcb98c 4px, #dcb98c 8px); transform:rotate(6deg); }
  ```

- **Tally marks** — hand-drawn via inline SVG, four verticals + one diagonal strike per group of 5, count driven by `Math.floor(expenseCount / 5)` full groups + remainder:
  ```html
  <svg viewBox="0 0 40 22"><g stroke="#1a1816" stroke-width="2.4" stroke-linecap="round">
    <line x1="3" y1="2" x2="3" y2="20"/> <!-- repeat for each of first 4 -->
    <line x1="1" y1="20" x2="26" y2="1"/> <!-- 5th = diagonal strike -->
  </g></svg>
  ```
  For counts that aren't clean multiples of 5, render the tally SVG as a small generator function rather than hardcoding paths (see below).

- **New font:** `Caveat` (Google Fonts, handwriting) — used **only** for `.who` and `.board-title`, nowhere else. Justified narrowly by the corkboard-note conceit; don't let it leak into body copy or numerals.

**Data mapping:**
- one `.note` per group (`groups`, capped to a reasonable count with a "+N more" note or scroll)
- `.tally` count → that group's `expense_count`
- `.owe-line` amount + sign → per-group balance for the current user
- button label: `"Remind"` if you're owed, `"Settle"` (filled accent) if you owe

**React componentization:** `<Pinboard groups={} />` → maps to `<PinNote group={} index={} />`. Tally rendering extracted as `<TallyMarks count={n} />`, generating `Math.floor(n/5)` full five-groups + `n % 5` loose lines, each group offset by a fixed pitch so they read left-to-right.

```tsx
function TallyMarks({ count }: { count: number }) {
  const full = Math.floor(count / 5);
  const rem = count % 5;
  // render `full` five-bundles (4 verticals + 1 diagonal strike), then `rem` loose verticals
}
```

---

## Porting checklist

- [ ] Extract each concept as its own component under `components/dashboard/concepts/`
- [ ] Move the balance/ledger/net-balance derivation logic out of markup into a shared `useDashboardSummary()` hook so all three concepts (and future ones) consume the same shape
- [ ] Rotation values in **Tab** → seed from `group.id` (e.g. `hashCode(id) % 5 - 2` degrees) instead of nth-child
- [ ] Confirm `mix-blend-mode: multiply` (Receipt stamp) renders correctly over the app's actual card background, not just `#fefefe`
- [ ] `Caveat` font: add to the Google Fonts `<link>` in `app/layout.tsx` only if Tab ships; don't load it app-wide otherwise
- [ ] All three are decorative-heavy — audit for `prefers-reduced-motion` / no motion currently used, but check hover states before shipping

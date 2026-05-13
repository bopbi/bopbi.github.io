# warga.jp — Design System

Design decisions locked in through the Claude Design process. Read this before adding any new UI surface or editing existing layouts.

## Product & Audience

**warga.jp** is a community platform for Indonesian citizens in Japan. Content types: panduan (guides) and komunitas (community listings). Admin moderates every submission before it publishes. No user accounts in production.

**Audience:** Indonesians 22–32 on TG (technical intern), internship, or working visa. Mobile-first. Scrolls Instagram/TikTok/LINE daily. Speaks casual Indonesian mixed with Jepang terms (zairyu, Kokuho, apato, tenkyo, kakutei shinkoku). Not a Craigslist-era interface; not a magazine-editorial either — closer to Mercari JP, Notion, current-day Reddit.

## Visual Direction — Locked

### Type system

- **Plus Jakarta Sans** throughout (designed in Jakarta, cultural fit without kitsch)
- **JetBrains Mono** for metadata only — timestamps, prefecture codes, counts, dates, small IDs
- **No display serif anywhere.** No italic `<em>` accents inside headings. No drop caps, no kicker→big-serif-headline magazine pattern.

### Color tokens

Defined in `static/base.css` as CSS custom properties (shared by public and admin). The canonical token list:

```css
--paper: #fafaf7      /* warm paper, body background */
--paper-2: #f3f2ec    /* slightly darker surface, card fills */
--paper-3: #e9e7de    /* pressed/active surface */
--ink: #141414        /* primary text */
--ink-2: #444         /* secondary text */
--ink-3: #7a7a75      /* muted / placeholders */
--rule: #e1ded4       /* 1px card borders */
--rule-soft: #eeece4  /* inner dividers, list separators */

--merah: #c62828      /* Pico red — CTAs, primary actions, current-state ONLY */
--merah-ink: #9f1f1f  /* darker merah for text on merah-soft bg */
--merah-soft: #fcebea /* light merah tint — nasional badges, pin-link bg */

--amber: #f5c542      /* ⭐ Rekomendasi section-head underline */
--amber-soft: #fff8e1 /* light amber tint — prefecture label bg */
--amber-ink: #8a6400  /* amber text on amber-soft bg */

--blue: #2196f3       /* 🆕 Panduan Terbaru (informational/latest) */
--blue-soft: #e3f2fd  /* light blue tint — terbaru row hover */
--blue-ink: #0d47a1   /* blue text on blue-soft bg */
```

**Red is functional, not decorative.** Never use `--merah` as a kicker color, eyebrow color, or generic accent. Reserved for: primary buttons, "current" nav state, `.btn-cta`, national content badges (`.nasional`). Exception: the masthead and hero/contrib full-bleed panels (structural surfaces, not decorative).

### Component patterns

**Cards** — 1px `--rule` border on `#fff` or `--paper-2` fill, 10–14px radius, soft hover lift (`translateY(-1px)` + subtle shadow + `--ink` border).

**Section heads** — `h2` + meta text on one line, underlined with a 2px stripe in the section's accent color (amber / blue / ink). Not 2px all-around borders, not cutout label boxes.

**Buttons** — rounded 6–8px (or full 99px for pill forms like `.apex-search`). Primary = red fill / white text. Secondary = transparent / 1px border. `.btn-cta` = ink fill / paper text, hover → red.

**Metadata pills** (prefecture code, dates, counts) — mono font, 11px, muted color, optional `--paper-2` soft-background fill.

**Search bar** — `.apex-search`: pill-shaped (border-radius 99px), white background, ink border on focus. Button inside is ink-fill, hover → red. Used on `/panduan` and `/komunitas` apex listing pages.

**Nasional rail** — `.nasional-rail`: 2-column CSS grid (`1.4fr 1fr`, 1-column on mobile). Each column is a `.nas-col` panel (white bg, border, border-radius). Column header (`.nas-col header`) has a title + `.more` count link on a `--paper-2` stripe. Rows are `.nas-list` items — 2-col grid of `.nas-title` + `.nas-meta`. Footer (`.nas-foot`) is muted mono. Hover on rows → `--merah-soft` background.

**Region accordion** — `.region` is a `<details>` element (not `<div>`). `.region-head` is the `<summary>` — flex row with `h4` + `.n` meta caption. All regions start `open`. Clicking the summary collapses/expands natively with no JS.

**Region-pin** — The national entry at the top of each `.wilayah-grid` uses `<details class="region region-pin">`. It spans the full grid width (`grid-column: 1 / -1`) and carries a 3px `--merah` left-accent bar via `::before`. The link inside uses `.pin-link` (merah-soft fill → full merah on hover). `.pin-note` is a muted mono caption beside the link.

**`.nasional` badge modifier** — When a panduan or komunitas originates from `national.warga.jp`, its prefecture label badge gets the `.nasional` class (e.g. `.rek-pref.nasional`, `.t-pref.nasional`, `.km-pref.nasional`). This switches the color from muted-gray/amber to `--merah-ink` on `--merah-soft` background (or just `--merah-ink` with `font-weight: 600` for inline spans). The intent: national content stands out as a different tier without being loud.

**Komunitas tipe taxonomy** — Community listings carry a `category` field (one of 10 canonical slugs). The canonical list lives in `domain/community/model/category.go`. Each tipe has a slug, label, emoji, and tone (the slug itself is used as the CSS modifier class):

| Slug | Label | Emoji |
|---|---|---|
| `ppi` | PPI / Alumni | 🎓 |
| `keagamaan` | Keagamaan | 🙏 |
| `daerah` | Paguyuban daerah | 🏘️ |
| `olahraga` | Olahraga | ⚽ |
| `kuliner` | Kuliner & masak | 🍳 |
| `seni` | Seni & budaya | 🎨 |
| `profesional` | Profesional & karir | 💼 |
| `keluarga` | Keluarga & anak | 👨‍👩‍👧 |
| `bantuan` | Bantuan warga | 🤝 |
| `hobi` | Hobi lainnya | 🎮 |

Apply tipe color via `.tipe-{slug}` modifier on `.kla-tipe` / `.kom-tipe` / `.km-tipe` / `.tb-chip` / `.ts-chip`. Colors are soft-hue + ink tokens defined in `static/public.css`.

**Prefecture emoji composition** — `Prefecture.Emoji` and `Prefecture.Label` are separate fields. Templates compose them at the point of use: visual chrome (chips, dropdowns, cards) writes `{ pref.Emoji } { pref.Label }`; `<title>` tags, `<meta name="description">`, JSON-LD names, sitemap, and email subjects use `{ pref.Label }` only (no emoji). `Prefecture.Display()` is available for admin contexts that need the combined string but are not templ templates.

**Highlights row** — On the apex homepage, "Panduan Terbaru" and "Komunitas" are merged into one `.highlights-row` (2-col grid: 1.55fr 1fr). Each column has an `.hl-subhead` (eyebrow-style h3 + `.more` link). Right column uses `.kom-rail` > `.kom-mini` compact cards showing `.km-tipe` chip.

**Tipe-browse strip** — `.section.section-tight` > `.tipe-browse` (2-col: 220px label + chip area). Chips are `.tb-chip` linking to `/komunitas?tipe={slug}#list`. Each chip shows emoji, label, and count (`.n`). Only rendered when `vm.CategoryCounts` is non-empty.

**Lap-layout (report pages)** — 2-column grid (280px sidebar + 1fr main). Sidebar: `.lap-back` link, `.lap-side-h` subheadings, `.lap-preview` item card, `.lap-flow` numbered ol, `.lap-note` muted block. Main: `.lap-eyebrow` (red mono tag), `h1`, `.lede`, `.form-card`. Collapses to single column at ≤900px (sidebar moves below main).

**Reason chips** — `.reason-chips` > `<input type="radio" name="reason_kind">` + `<label class="rc">` pairs. CSS-only active state via `input[type="radio"]:checked + label.rc` (ink fill). No JavaScript. Hidden radio inputs are positioned off-screen (`position: absolute; opacity: 0; width: 1px; height: 1px`). The selected `reason_kind` value is posted alongside the free-text `reason` field.

**Tipe filter (apex `/komunitas`)** — `.tipe-strip` chips are **multi-select**; each click toggles a slug in/out of the URL (`?tipe=a&tipe=b#list`) via `tipeToggleURL` in `view/helpers.go`. Active chips get `.cur` class (ink fill). `.filter-active` notice (flex row with `.hapus` clear-all link) shows `selectedTipesLabel` (e.g. "🏘 Daerah + ⚽ Olahraga") + result count. `.empty-tipe` card renders when filter returns zero results; CTA links to `/komunitas/tambah` (no tipe preselect for multi). `noindex` set when any filter is active. `SelectedTipes []string` in `CommunityIndexViewModel`; `GetApprovedCommunityByCategories` in the repo.

**Prefecture hero enhancements** — `h1` in `.pref-hero` may include `<span class="jp">` with the kanji prefecture name (e.g. 東京都). `.pref-dek` paragraph shows a 1–2 sentence factual description. "Ibu kota" fact row added to `dl.facts`. Static data in `domain/prefecture/model/meta.go`.

**Footer social** — `.foot-social` block sits between `.foot-links` and `.foot-copy` in `SiteFooter` (`presentation/shared/component/chrome.templ`). On the ink panel footer the pill reads `rgba(250,250,247,0.78)` text/border; hover → white. To add another platform later (LINE, X, etc.), drop in another `<a>` sibling. **Templ gotcha:** `@` characters in literal text are parsed as component invocations; write handles as `{ "@warga.jp" }` (Go string interpolation) to escape.

**Chrome theme — "direction 2"** — locked as of the 2026-05 design session:
- **Masthead** — `background: var(--merah)`; white wordmark; amber dot; nav links `rgba(255,255,255,0.78)`, active `rgba(255,255,255,0.16)` fill; CTA white-on-red (`background: #fff; color: var(--merah)`). No bottom border (flows flush into red hero on apex).
- **Hero (apex only)** — full-bleed red panel via negative-margin breakout (`margin: 0 calc(50% - 50vw); padding: 72px calc(50vw - 50% + 24px) 64px`). White type, amber `.accent` on H1, white search pill with ink button (hover → `#000`), ghosted "Populer" chips (`rgba(255,255,255,0.12)` bg). `.hero-compact` variant (used by apex search-results mode) resets to cream via explicit overrides. Prefecture pages use `.pref-hero` (unrelated class) — not affected.
- **Contribute band** — same full-bleed breakout (`margin: 16px calc(50% - 50vw) 0`). Red bg, white type, `.btn-outline` inside overridden to white-on-red.
- **Footer** — `background: var(--ink)`, light type on near-black, no top rule. Inner-page margin-top stays 56px; `.contrib + .foot { margin-top: 0 }` fires when contrib is the direct footer predecessor.

### Layout

- Max content width: 1120px (via `.page` container)
- Mobile-first. Body 15.5px / 1.55. H1 ~48px desktop, ~30px mobile.
- Sections separated by padding (48px top) + accent-underline section head. No hairline dividers between sections.
- `@media (max-width: 860px)` is the main mobile breakpoint for apex-head and grid layouts.

## Don'ts

These patterns will get bounced back:

- **Fake social signals.** No user counts ("230.689 warga"), no "aktif sekarang", no real-time reply counts. Accounts don't exist; these are lies.
- **Vaporware in nav.** Only link features that actually exist in the router. Current live nav: **Beranda · Panduan · Komunitas · Kontak** + `＋ Bagikan ke warga` CTA. Diskusi/Lapak/Lowongan/Kuliner/Jasa are commented out — don't add them to nav or footer.
- **Editorial furniture.** No "Edisi", "Vol.", "No." metadata. No drop caps. No italic em accents in headings. No 2px all-around borders with cutout labels. No display serif headlines.
- **Decorative red.** Kickers, eyebrows, section labels use amber / blue / ink / mono-gray, never red.
- **CDN-loaded JS, inline `onclick=`, `<script>` tags** beyond the pinned `admin.js` pattern. Strict CSP (`script-src 'self'`) will block it in production.
- **Starting a section with `<em>` or italic serif accents.** Just say the words plainly.
- **Design-canvas variations for minor tweaks.** One option per decision unless explicitly asked for variations.

## Copywriting Tone

Casual Indonesian, second-person friendly. **"Kamu"** not "Anda". Mix Jepang terms where they're the real words people use (zairyu card, Kokuho, ward office, apato, gaimen kirikae, furusato nozei). Avoid marketing-speak and puffery ("revolusioner", "terbaik di Jepang"). Brand voice: **"warga yang udah ngelewatin duluan"** — a neighbor telling you how it works, not a startup pitching you.

CTA verbs:
- **"Bagikan"** not "Pasang" / "Tulis" — sets the admin-moderation expectation (submission, not instant publish)
- **"Ajukan"** on the form itself
- **"Cari"** for search (never "Search", never "Find")

## Section Accent Color System

| Section | Accent | Class |
|---|---|---|
| ⭐ Rekomendasi | Amber | `.section-head.amber` |
| 🆕 Panduan Terbaru + 🤝 Komunitas (combined `.highlights-row`) | Blue / Ink | `.hl-subhead` (no section-head underline) |
| 🗂 Telusuri komunitas (tipe-browse) | — | `.section.section-tight > .tipe-browse` (no section-head) |
| 🇮🇩 Berlaku di seluruh Jepang | Ink / neutral | `.section-head` (no modifier) |
| 🗾 Pilih prefektur (unified dual-count grid) | Ink | `.section-head` (no modifier) |
| Hasil pencarian | Ink / neutral | `.section-head` (no modifier) |

## When Starting a New Design Task

1. Read this file first.
2. Check `docs/rules/ui-pico-conflicts.md` for Pico.css conflict patterns, `docs/rules/ui-tokens.md` for design tokens, and `docs/rules/ui-layouts.md` for layout component signatures. Check `docs/rules/homepage.md` for homepage section patterns and their CSS classes.
3. Check `docs/rules/seo.md` for layout signatures (`LayoutGeneric`, `LayoutPrefecture`, etc.).
4. Don't invent features — match what's in `docs/CONTEXT.md` and the live router.
5. Numbers must be real. If the backend can't produce a number, don't show it. "butuh kontributor" beats a fake count.
6. One clean option per decision unless variations are explicitly requested.

## Related Docs

| Doc | What it covers |
|---|---|
| `docs/rules/ui-layouts.md` | Static assets, layouts, search, footer, chrome theme, root font-size |
| `docs/rules/ui-pico-conflicts.md` | Pico.css conflict patterns |
| `docs/rules/ui-tokens.md` | Design token reference |
| `docs/rules/homepage.md` | Homepage sections (Rekomendasi, Terbaru, Nasional Rail), Contribute CTA |
| `docs/rules/apex-listing-pages.md` | Apex listing pages (`/panduan`, `/komunitas`) CSS classes, data flows, `?tipe=` filter |
| `docs/rules/seo.md` | Layout templ signatures, noindex policy, canonical URLs |
| `docs/rules/forms.md` | Form field design, label markers, contact privacy |
| `docs/CONTEXT.md` | Full route list, tech stack, environment variables |

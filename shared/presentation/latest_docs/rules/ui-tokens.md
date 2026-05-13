# UI — Design Token Reference

All custom properties are declared in `:root` in `static/base.css`. Key tokens:

| Token | Value | Use |
|---|---|---|
| `--sans` | Plus Jakarta Sans, system fallback | Body and UI text |
| `--mono` | JetBrains Mono, monospace fallback | Code, labels, dates |
| `--merah` | `#c62828` | Primary red — CTA buttons, accents |
| `--merah-soft` | `#fcebea` | Red tint backgrounds |
| `--ink` | `#141414` | Primary text |
| `--ink-2` | `#444` | Secondary text |
| `--ink-3` | `#7a7a75` | Muted text, hints |
| `--paper` | `#fafaf7` | Body background |
| `--paper-2` | `#f3f2ec` | Card/section backgrounds |
| `--rule` | `#e1ded4` | Borders |
| `--rule-soft` | `#eeece4` | Inner dividers |
| `--amber` | `#f5c542` | Callout accent |
| `--amber-soft` | `#fff8e1` | Amber tint backgrounds |
| `--blue` | `#2196f3` | Informational/latest accent |
| `--radius` | `8px` | Default border radius |
| `--radius-sm` | `6px` | Small elements |
| `--radius-lg` | `12px` | Cards |

**Red is functional, not decorative.** Reserve `--merah` for: primary buttons, "current" nav state, `.btn-cta`, national content badges, and the four locked structural surfaces (`.masthead`, apex `.hero`, apex `.contrib`). Never use it as a kicker color or generic accent inside content. See `docs/DESIGN.md` and `rules/ui-layouts.md` § Chrome theme for the full visual direction.

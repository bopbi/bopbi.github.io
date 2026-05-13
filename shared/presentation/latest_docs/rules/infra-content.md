# Infrastructure — Legal Pages & Changelog

## Legal Pages

> **Every change to `privacy.templ` or `terms.templ` requires four things done in the same commit — see the checklist below. Skipping any step leaves the site in an inconsistent state.**

### Checklist for every legal page update

When changing `privacy.templ` or `terms.templ`:

1. Update the page content in the `.templ` file.
2. Update the corresponding date constant in `presentation/shared/legal/dates.go`.
3. **Prepend** a new entry to `Entries` in `presentation/shared/changelog/entries.go` — `/weblogs` is the user-facing record of legal changes and must stay current.
4. Run `templ generate` on the affected view package.

All four in the same commit.

The same applies to any other major platform change (new features, significant behaviour changes, etc.) — step 3 only.

### Updating the "last updated" date

The Privacy Policy (`/kebijakan-privasi`) and Terms & Conditions (`/syarat-ketentuan`) each show a "Terakhir diperbarui" date. These dates live in one file:

```
presentation/shared/legal/dates.go
```

```go
const PrivacyUpdated = "19 April 2026"  // update when changing privacy.templ
const TermsUpdated   = "18 April 2026"  // update when changing terms.templ
```

Do not edit the date inside the `.templ` files directly — the constants are the single source of truth.

The `.templ` files import this package and render the value via `{ legal.PrivacyUpdated }` / `{ legal.TermsUpdated }`.

### Announcing changes in the changelog

Prepend to the `Entries` slice in `presentation/shared/changelog/entries.go` — newest first, never append:

```
presentation/shared/changelog/entries.go
```

```go
var Entries = []Entry{
    // newest first — always prepend, never append
    {
        Date:        "19 April 2026",
        Kind:        "kebijakan",   // "feat" | "kebijakan" | "" (omit chip)
        Title:       "Pembaruan Kebijakan Privasi — Pencatatan Kata Kunci Pencarian",
        Description: "Kebijakan Privasi diperbarui untuk mencerminkan bahwa kata kunci yang dimasukkan di kolom pencarian kini dicatat secara anonim di server kami.",
    },
}
```

`Kind` controls the coloured chip on the `/weblogs` timeline. Allowed values:

| Value | Label | Colour |
|---|---|---|
| `"feat"` | Fitur | Blue |
| `"kebijakan"` | Kebijakan | Amber |
| `""` | _(no chip)_ | — |

Use `"kebijakan"` for privacy policy / terms changes. Use `"feat"` for new public features. Leave `Kind` empty for minor operational entries that don't fit either category.

**Rule:** The slice is capped at 45 entries (`MaxEntries` in `entries.go`). When the slice reaches 45, remove the oldest entry (the last element) in the same commit as the new prepend. The `GetEntries()` helper enforces this cap at runtime as a safety net, but the slice itself should never exceed 45.

# Admin JavaScript — `static/admin.js`

All admin-side behaviour is in `static/admin.js`, served from the same origin and allowed by the `script-src 'self'` CSP. It is embedded in the binary via `static/static.go` and served from `/static/admin.js`.

Add `<script src={ "/static/admin.js?v=" + version.Build } defer></script>` to the `<head>` of any admin template that needs JS.

**Always include `defer`.** Without it the script executes synchronously during parsing, before the DOM exists — listeners never attach and nothing works in production. Do not wrap code in `DOMContentLoaded` — with `defer` it is redundant.

## Current behaviours wired in `admin.js`

| Behaviour | How it is declared in HTML | How `admin.js` handles it |
|---|---|---|
| Prefecture `<select>` auto-navigates on change | `id="prefecture-select"` on the `<select>`, form `action` is the target URL | `change` listener reads `sel.form.action + '?prefecture=' + value` and assigns `window.location.href` |
| Confirm dialog before destructive action | `data-confirm="…"` on the `<button>` | `click` listener calls `window.confirm(el.dataset.confirm)` and cancels if declined |
| Clipboard copy with brief feedback | `data-copy="<text>"` on any `<button>` | `click` listener runs `execCommand('copy')` synchronously first (always works in a direct click handler), then fires `navigator.clipboard.writeText` as a parallel best-effort override; shows `Tersalin!` for 1.5 s |

## Templates that currently load `admin.js`

| Template | Behaviours used |
|---|---|
| `presentation/admin/panduan/view/list.templ` | `data-confirm`, `#prefecture-select`, `data-copy` |
| `presentation/admin/panduan/view/detail.templ` | `data-copy` |
| `presentation/admin/community/view/list.templ` | `data-confirm`, `#prefecture-select`, `data-copy` |
| `presentation/admin/community/view/detail.templ` | `data-copy` |
| `presentation/admin/report/view/list.templ` | `data-confirm` |
| `presentation/admin/featured/view/featured.templ` | `data-confirm` |

> **Common mistake:** adding `data-copy` or `data-confirm` but forgetting `<script src=...admin.js...>` in the template's `<head>`. The button silently does nothing.

> **`data-copy` pitfall — local vs production discrepancy:** `navigator.clipboard` is only available in secure contexts (HTTPS). On `machine.local` (plain HTTP) the object is `undefined`, so the code always takes the synchronous `execCommand` path — which is reliable and always works. On `warga.jp` (HTTPS) `navigator.clipboard` exists, so async `writeText` is attempted; if it rejects (tab not focused, permission denied, etc.) any fallback that runs inside the `.catch()` callback is no longer a direct user gesture and `execCommand` silently fails. **Rule:** for `data-copy`, always run `execCommand` synchronously first inside the click handler, then fire the clipboard API as a parallel best-effort call — never as an async fallback. Bugs in the async-only path are invisible on `machine.local` and only surface in production.

## Pattern for a prefecture selector (do not use `onchange`)

```templ
<form method="GET" action="/admin/panduan" class="form-filter">
    <select id="prefecture-select" name="prefecture">
        for _, pref := range vm.Prefectures {
            if pref.Value == vm.SelectedPrefecture {
                <option value={ pref.Value } selected>{ pref.Label }</option>
            } else {
                <option value={ pref.Value }>{ pref.Label }</option>
            }
        }
    </select>
</form>
```

## Pattern for a confirm-before-submit button (do not use `onclick`)

```templ
<form method="POST" action={ templ.URL(...) } class="form-inline">
    <button type="submit" data-confirm="Yakin ingin menghapus?">Hapus</button>
</form>
```

If adding new external script sources, update `Caddyfile.prod`; no Go code changes are needed.

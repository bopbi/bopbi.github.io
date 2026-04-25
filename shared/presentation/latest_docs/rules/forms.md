# Forms

## Markdown preview / edit-lagi — token and state preservation

Panduan forms (public `/panduan/tambah` and admin `/admin/panduan/tambah`, `/admin/panduan/{id}/edit`) support a **preview → edit → submit** cycle. The handler checks `r.FormValue("action")` and short-circuits into `preview` or `edit` branches **before** rate limiting and captcha verification.

**Key rules:**

1. **Do NOT rotate the CSRF token on preview/edit.** The viewmodel reuses `r.FormValue("csrf_token")` so the cookie set on the original GET still matches. Rotating would invalidate the cookie and break the final submit.
2. **Do NOT re-verify captcha on preview/edit.** The captcha question and HMAC token are preserved in hidden inputs (`captcha_question`, `captcha_answer`) and the user's answer in `captcha_user`. They are only verified on `action=submit`.
3. **Do NOT count preview/edit against the submit rate limiter.** These branches return before `limiter.Allow()` is called, so users can iterate freely without consuming their 5/hour submission budget. However, the **preview** branch is independently rate-limited (30/hour/IP via a separate `previewLimiter`) because `markdown.RenderMarkdown` is CPU-bound — unmetered previews would be a DoS vector even though CSRF is enforced. The **edit** branch just re-renders the editor (no markdown work), so it doesn't need its own limit. Admin forms don't need a preview limiter (admins are authenticated and trusted).
4. **Preserve all form state in hidden inputs.** When the template is in preview mode, every user-entered field (email, whatsapp, line, prefecture, category, title, content, captcha_user) must be carried in `<input type="hidden">` or a hidden `<textarea>` so clicking "Ajukan"/"Simpan" submits the complete form.

**Viewmodel constructors:**
- `NewPanduanPreviewViewModel(r, previewHTML)` — reuses all form values from the request, sets `PreviewHTML` to flip the template into preview mode.
- `NewPanduanEditViewModel(r)` — same preserved fields, but `PreviewHTML` is empty so the template renders the textarea editor.

**Handler pattern** (runs after CSRF check, before rate limit + captcha):
```go
if action := r.FormValue("action"); action == "preview" {
    html := markdown.RenderMarkdown(r.FormValue("content"))
    vm := viewmodel.NewPanduanPreviewViewModel(r, html)
    adminRepository.SetCSRFTokenCookie(w, vm.CSRFToken)
    templ.Handler(view.SubmitScreen(vm)).ServeHTTP(w, r)
    return
} else if action == "edit" {
    vm := viewmodel.NewPanduanEditViewModel(r)
    adminRepository.SetCSRFTokenCookie(w, vm.CSRFToken)
    templ.Handler(view.SubmitScreen(vm)).ServeHTTP(w, r)
    return
}
```

**Template pattern** — the template branches on `vm.PreviewHTML != ""`:
- **Preview mode:** rendered markdown in a `.markdown-preview` box, hidden inputs for all fields, "Edit Lagi" + "Ajukan" buttons.
- **Editor mode:** full form with textarea, "Pratinjau" + "Ajukan" buttons.

## Form Design

### One field per data point
Every form input must map to exactly one DB column, and every DB column must map to at most one form input. Before adding a new field to a form or a new column to a table, answer both questions:

1. **Is this data already captured by another field?** If yes, remove the old field or reuse it — don't add a new one alongside it.
2. **Does this column already exist under a different name?** Check the table schema before writing the migration.

Failing this check is what created the `contact` / `submitter_*` overlap in `community_info`: a generic free-text "Kontak" field was added next to structured `email`, `whatsapp`, and `line` fields that already captured the same information. The `contact` column was removed in migration 9 to resolve this.

### Checklist before adding a form field
- [ ] Read the existing form — is a field with the same purpose already there?
- [ ] Read the model struct — is there already a field that holds this value?
- [ ] Read the DB schema (`PRAGMA table_info(table_name)`) — is there already a column for it?
- [ ] If you find a duplicate, remove the old one; do not keep both.

### Contact fields (community_info and generic_info_post)
The canonical submitter contact fields for all submission forms are:
- `email` (required) → `submitter_email`
- `whatsapp` (optional) → `submitter_whatsapp`
- `line` (optional) → `submitter_line`

Do **not** add any other contact-style field (e.g. a generic "Kontak" text input). These three fields are stored for admin moderation only.

**Submitter contact lifecycle — stored → reviewed → deleted:**
1. **On submission (pending):** fields are stored in plaintext — visible to the admin for moderation.
2. **On approval:** `ClearCommunitySubmitterContact` / `ClearGenericInfoPostSubmitterContact` immediately NULLs all three columns. The admin sees empty cells in the detail view after approval.
3. **On rejection:** contact is intentionally preserved — admin may need it for a courtesy follow-up.
4. **Backfill (migration v3):** the one-time `migrateNullApprovedSubmitterContact` migration (per-prefecture DB v3, runs once on boot) wiped all existing approved rows on first deploy.

They must **never** be rendered on any public-facing template (listing cards, detail pages, search results, OG tags, JSON-LD, sitemap, RSS, etc.) at any status. This is a permanent rule — do not re-introduce a public "Kontak & link" card that exposes them. If a future feature needs public community contact (e.g. a group URL, Instagram handle), add a separate dedicated column (see `TODO.md` → "Community social links") rather than repurposing the submitter fields.

**Admin UX note:** because approval immediately NULLs the contact, admins who want to send a thank-you or follow-up must copy the email/WhatsApp **before** clicking Setujui. There is no recovery after approval.

Submission form copy must also make this clear to the user: label each submitter input with the `<span class="opt">… · privat</span>` marker (e.g. `wajib · privat`, `opsional · privat`) and place them under a section whose heading and sub-copy states that these fields are for moderator verification only, are not displayed publicly, and are deleted on approval.

### Mandatory / optional indicators on every input

**Every `<label>` on a public submission form must carry a `<span class="opt">` marker that tells the user whether the field is required.** No unlabeled fields; no exceptions for "obvious" ones. This prevents users from guessing and reduces bounce on server-side validation errors.

Marker vocabulary (kept consistent across forms):

| Marker | When to use |
|---|---|
| `wajib` | Required field — handler rejects the submission if empty. |
| `opsional` | Optional field — handler accepts empty values. |
| `wajib · privat` | Required submitter-only field (email). |
| `opsional · privat` | Optional submitter-only field (WhatsApp, Line ID). |
| `wajib · Markdown` | Required field that accepts Markdown (panduan content). |

Pattern:

```templ
<label for="title">Judul panduan <span class="opt">wajib</span></label>
<label for="whatsapp">WhatsApp <span class="opt">opsional · privat</span></label>
```

Rules:
- The marker must match the handler's actual behavior — if the handler returns `"Field X wajib"` when empty, the label must say `wajib`; if not, it must say `opsional`.
- The **captcha** input is the one exception: its label is the captcha question itself, so no marker is added (the input's purpose is self-evident from the prompt).
- Radio groups that are UI-only (not persisted or validated server-side, e.g. komunitas `tipe`) should be marked `opsional`.
- When you add a new field, update its label marker in the **same** commit as the handler validation — never let the two drift.

This is the counterpart to the honeypot / CSRF / rate-limit patterns in `security.md`: a form that asks for data must also tell the user what it expects.

### Inline feedback messages (ErrorMsg / SuccessMsg)

**Every page that accepts user input** — submission forms (komunitas/tambah, panduan/tambah) and contact forms (/kontak) — must display feedback as a styled alert box. This applies to any future form as well.

Use exactly this markup (CSS classes from `static/base.css / public.css` — no inline styles):

```templ
if vm.ErrorMsg != "" {
    <div class="alert alert-error">
        <div class="alert-title">Terjadi Kesalahan</div>
        { vm.ErrorMsg }
    </div>
}

if vm.SuccessMsg != "" {
    <div class="alert alert-success">
        <div class="alert-title">Berhasil!</div>
        { vm.SuccessMsg }
    </div>
}
```

Rules:
- Place the message block **above the form** so it is visible without scrolling.
- Use a bold label (`Berhasil!` / `Terjadi Kesalahan`) as a heading inside the box so the type of message is immediately clear.
- Put the message text directly inside the box div using `{ vm.ErrorMsg }` / `{ vm.SuccessMsg }` — do **not** wrap in `<strong>`.
- Do **not** use `style="..."` attributes — inline styles are blocked by the `style-src 'self'` CSP. Add classes to `base.css / public.css` instead.
- The viewmodel must always expose both `ErrorMsg string` and `SuccessMsg string` fields, even if only one is used, so the pattern stays consistent.

## Reports (Laporan)

### Status flow
| Status | Meaning | How reached |
|---|---|---|
| `pending` | Not yet actioned | Created on visitor submission |
| `resolved` | Admin acted on the content | Via Nonaktifkan or Hapus Konten |
| `dismissed` | Admin closed without action | Via Abaikan |

There is no intermediate "reviewed" state — every action either resolves or dismisses the report. This avoids a dead-end state where reports are acknowledged but never actioned.

### Admin actions
- **Nonaktifkan** — sets the content's `status` to `rejected` (hides it from public pages) then marks the report `resolved`. Use when the content is inaccurate or violates guidelines but should be kept for audit purposes.
- **Hapus Konten** — permanently deletes the content row then marks the report `resolved`. Use when the content should be fully removed. A browser `confirm()` guards against accidental clicks.
- **Abaikan** — marks the report `dismissed` without touching the content. Use when the report is unfounded.

### Dual-action handler pattern
Action handlers that affect both a report and its referenced content must:
1. Look up the report by ID to retrieve `content_type` and `content_id`.
2. Dispatch to the correct repository based on `content_type` (`"komunitas"` → `communityRepository`, `"panduan"` → `prefectureRepository`).
3. Act on the content first, then update the report status — so if the content action fails, the report stays `pending`.

```go
report, _ := reportRepository.GetReportByID(db, reportID)
switch report.ContentType {
case "komunitas":
    communityRepository.UpdateCommunityStatus(db, report.ContentID, "rejected")
case "panduan":
    prefectureRepository.UpdateGenericInfoPostStatus(db, report.ContentID, "rejected")
}
reportRepository.UpdateReportStatus(db, reportID, "resolved")
```

See `presentation/admin/report/handler/handler.go` for the full implementation.

## Email Notifications

Admin email notifications are sent from `lib/notify`. Configuration is loaded once at startup via `notify.LoadConfig()` (reads env vars) and threaded through `Router()` into the relevant handlers.

### Functions in `lib/notify`

| Function | Used by | Purpose |
|---|---|---|
| `notify.Submission(cfg, type, prefecture, title)` | komunitas + panduan submit POST handlers | Alerts admin that new content is pending review |
| `notify.Contact(cfg, senderEmail, message)` | `/kontak` POST handler | Forwards visitor message; sets `Reply-To` so admin can reply directly |

### Adding a notification to a new submission handler

1. Accept `emailCfg notify.Config` in the handler factory signature.
2. After the content is successfully saved to the DB, fire in a goroutine:
   ```go
   go notify.Submission(emailCfg, "komunitas", prefectureLabel, content.Name)
   ```
3. Pass `emailCfg` from `router.go` when registering the handler.

The call is always fire-and-forget (`go ...`) — SMTP latency must never block the HTTP response. `notify.Submission` logs failures but does not return an error, so a misconfigured SMTP server will never cause a 500.

### Environment variables

| Var | Example | Required |
|---|---|---|
| `SMTP_HOST` | `smtp.gmail.com` | Yes (disables notifications if missing) |
| `SMTP_PORT` | `587` | No (defaults to `587`) |
| `SMTP_USER` | `you@gmail.com` | Yes |
| `SMTP_PASS` | app password | Yes |
| `NOTIFY_TO` | `you@gmail.com` | No (defaults to `SMTP_USER`) |

If `SMTP_HOST` is empty, all `lib/notify` calls are no-ops — safe for local dev without any SMTP setup.

### Adding future notification channels (webhook, WhatsApp)

Add a new function to `lib/notify/` (e.g. `webhook.go`) and call it alongside `notify.Submission` in the same goroutine or in its own. Keep `lib/notify` as the single location for all outbound notification logic.

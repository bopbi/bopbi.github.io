# Article Edit Policy

## Who can edit articles
Only admins can edit articles (panduan and komunitas) after submission. There is no public user account system — submissions track contact info only (email, WhatsApp, Line), not a registered user identity. Creator editing is therefore not feasible without a separate auth system.

Edit handlers: `presentation/admin/panduan/handler/handler.go` (`PanduanEditHandler`, `PanduanEditPostHandler`) and `presentation/admin/community/handler/handler.go` (`CommunityEditHandler`, `CommunityEditPostHandler`).

**Rule:** Admin edit handlers must always fetch the row with `GetGenericInfoPostByIDAnyStatus` / `GetCommunityByIDAnyStatus` — never the approved-only `GetGenericInfoPostByID` / `GetCommunityByID`. Using the approved-only variant causes a 404 when an admin tries to edit a pending or rejected entry.

## Approval side-effect: submitter contact is permanently deleted
Clicking **Setujui** (approve) on a komunitas or panduan calls `ClearCommunitySubmitterContact` / `ClearGenericInfoPostSubmitterContact` immediately after `UpdateStatus`. All three submitter contact columns (`submitter_email`, `submitter_whatsapp`, `submitter_line`) are NULLed in the same request. **This is irreversible.** If you need to contact the submitter (thank-you, follow-up), copy their email/WhatsApp from the detail view *before* clicking Setujui. Rejection (`Tolak`) intentionally leaves contact intact.

## Status after admin edit
Approved articles **stay approved** after an admin edit. Admin edits are trusted — no re-review cycle is triggered.

## Admin edit indicator (`admin_edited_at`)
Both `generic_info_post` and `community_info` tables have an `admin_edited_at DATETIME` nullable column (added in migration version 2). It is set **only** by `UpdateGenericInfoPost` and `UpdateCommunity` — not by `UpdateGenericInfoPostStatus` / `UpdateCommunityStatus` (approve/reject).

This distinction matters: `updated_at` is also touched on approve/reject, so it cannot be used as a reliable "was this content edited?" signal. `admin_edited_at` is the authoritative flag.

When non-null, the public detail page shows **"✏ Diperbarui admin: [date]"** below the creation date. The formatted value is computed in the detail viewmodel (`AdminEditedAtFormatted`) using `lib/formatter.FormatDate`.

**Rule:** Never set `admin_edited_at` from a status-change function. Only set it from a function that modifies the article content itself.

# Admin Dashboard

## Admin dashboard — action queue, pending counts must link to the manage page

`/admin` is an action queue: the "Per Prefektur — Butuh Perhatian" table only shows prefectures where `KomunitasPending + PanduanPending + ReportsPending > 0`. Rows with no pending work are hidden. Rows are sorted descending by total pending (most urgent first). When no prefecture has pending work an empty-state `.msg-success` banner is shown instead.

Every pending count shown in the table must be a link that navigates directly to the relevant manage page with the prefecture pre-selected. A bare number with no link forces the admin to manually switch the selector — that is the exact UX issue this rule exists to prevent.

**Pattern** (each cell shows pending only, or an em-dash when zero):
```templ
<td>
    if p.KomunitasPending > 0 {
        <a href={ templ.URL("/admin/komunitas?prefecture=" + p.Value) }>
            <span class="status-pending">{ fmt.Sprintf("%d", p.KomunitasPending) } pending</span>
        </a>
    } else {
        { "—" }
    }
</td>
```

**Filter + sort live in `presentation/admin/dashboard/viewmodel/viewmodel.go`**: the `sort.Slice` call right after `BuildPrefectureSummaries`. The template gates the empty state on `vm.PendingKomunitas+vm.PendingPanduan+vm.PendingReports > 0`. If a new content type is added to the table, add its pending field to the sort key, the template row condition, and the empty-state check.

## VPS health indicator (`/admin/sistem`)

`/admin/sistem` shows a live snapshot of the host: CPU 1m load, RAM, disk used by `dbs/`, goroutines, Go heap, DB file sizes, and build info — each metric tagged with a green/amber/red severity badge using the same `.status-approved` / `.status-pending` / `.status-rejected` classes used elsewhere in the admin UI. The dashboard (`/admin`) carries a compact "VPS" tile that summarises the worst severity and links to the detail page.

**Sampler:** `lib/sysmetrics.Sample()` calls `github.com/shirou/gopsutil/v4` (`load.Avg`, `mem.VirtualMemory`, `disk.Usage("dbs/")`) plus `runtime.ReadMemStats` and `os.Stat` for each DB file. One sample per request; no background goroutine; no metrics DB. Cross-platform — works the same on Linux production and macOS dev so the page can be QA'd locally.

**Thresholds** live in `presentation/admin/system/viewmodel/viewmodel.go` and mirror `docs/SCALING.md` § Trigger Thresholds. Update both files together if a threshold changes.

## Kontak (contact form) tracking

Every `/kontak` submission is saved to `dbs/admin.db` (`contact_messages`) before the notification email is sent, so messages are never lost even if SMTP is misconfigured.

**Statuses:** `pending` (default) → `replied` (admin clicks *Tandai dibalas*) → `archived` (soft-delete / spam). `archived` rows can be restored to `pending` via *Kembalikan*. `replied_at` and `archived_at` timestamps are set automatically on transition.

**Dashboard card** shows the current `pending` count. Anything above zero means messages are waiting for a reply.

### Reply procedure

Cloudflare Email Routing is **receive-only** — it forwards `kontak@warga.jp` to your personal inbox but cannot send. Replying directly from that forwarded mail would expose your personal email address to the visitor. Instead:

1. Open `/admin/kontak`, find the message, click **Lihat** to read the full body.
2. Click **Balas via Email** — this opens a `mailto:` link pre-addressed to the sender.
3. **Before sending**, check that the Gmail **From** field is set to `kontak@warga.jp` (the "Send mail as" alias backed by Resend SMTP — see `docs/rules/forms.md` § Environment variables). Never send from the default personal address.
4. After replying, return to `/admin/kontak` and click **Tandai dibalas** to record `replied_at` and drop the dashboard pending count.

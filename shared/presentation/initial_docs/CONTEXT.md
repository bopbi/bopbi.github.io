# Warga.jp - Project Context

## Overview
Japanese-Indonesian community platform for each prefecture. Users can browse communities, marketplaces, and other features organized by Japanese prefecture. Users can also submit new content which requires admin approval before being displayed.

## Tech Stack
- **Language:** Go
- **Router:** github.com/go-chi/chi/v5
- **Templating:** github.com/a-h/templ (Templ)
- **Database:** SQLite (github.com/mattn/go-sqlite3)
- **Auth:** Session-based with bcrypt password hashing
- **CSS:** Pico.css (via CDN)
- **Markdown:** github.com/gomarkdown/markdown

## Architecture Pattern
```
domain/          # Business logic, models, repositories
presentation/    # HTTP handlers, views, viewmodels
migrations/      # Database schema
dbs/             # SQLite database files
lib/             # Utilities (formatters, navigation, markdown)
constant/        # Constants (domain name)
router/          # Route definitions
```

## Database Structure
- `dbs/admin.db` - Admin users & sessions
- `dbs/user.db` - User accounts & sessions
- `dbs/<prefecture>.db` - Prefecture-specific data (tokyo.db, osaka.db, etc.)

### Key Tables
- `admins` - Admin user accounts
- `admin_sessions` - Admin login sessions
- `users` - Regular user accounts
- `sessions` - User login sessions
- `community_info` - Community listings with status and submitter info
- `generic_info_post` - Information/guide posts with status and submitter info
- `marketplace` - Buy/sell listings
- `schema_migrations` - Tracks applied migrations

### Table Schema Changes (Versioned)
Both `community_info` and `generic_info_post` tables have been extended with:
- `status` - Submission status: `pending`, `approved`, `rejected`
- `submitter_email` - Submitter's email
- `submitter_whatsapp` - Submitter's WhatsApp number
- `submitter_line` - Submitter's Line ID

## Key Routes

### Public Routes
- `/` - Homepage with all prefectures
- `/komunitas` - Community list (per prefecture subdomain)
- `/komunitas/{id}` - Community detail
- `/komunitas/tambah` - Submit new community (GET/POST)
- `/panduan/tambah` - Submit new panduan/guide (GET/POST)

### Admin Routes
- `/admin/login` - Admin login
- `/admin/logout` - Admin logout
- `/admin/komunitas` - Manage communities (with prefecture selector)
- `/admin/komunitas/tambah` - Add community form
- `/admin/komunitas/{id}/edit` - Edit community form
- `/admin/komunitas/{id}/hapus` - Delete community
- `/admin/komunitas/{id}/setujui` - Approve pending community
- `/admin/komunitas/{id}/tolak` - Reject pending community
- `/admin/panduan` - Manage panduan/guide posts (with prefecture selector)
- `/admin/panduan/tambah` - Add panduan form
- `/admin/panduan/{id}/edit` - Edit panduan form
- `/admin/panduan/{id}/hapus` - Delete panduan
- `/admin/panduan/{id}/setujui` - Approve pending panduan
- `/admin/panduan/{id}/tolak` - Reject pending panduan

### Subdomain Routes
- `<subdomain>.warga.jp` - Prefecture-specific content
- Example: `tokyo.warga.jp/komunitas`

## User Submission Workflow

### Features
1. **Math Captcha** - Simple addition captcha to prevent spam
2. **Privacy Policy** - Notice that personal data won't be used for commercial purposes
3. **Status Workflow** - `pending` → `approved` or `rejected`
4. **Contact Info** - Email (required), WhatsApp, Line ID (optional)
5. **Motivational Messaging** - Contextual messages with prefecture name encourage contributions

### Flow
1. User visits `/komunitas/tambah` or `/panduan/tambah`
2. Reads motivational message with prefecture context
3. Fills form with contact info, content, and math captcha
4. Submission saved with `pending` status
5. Admin reviews in admin panel (`/admin/komunitas` or `/admin/panduan`)
6. Admin clicks "Setujui" (Approve) or "Tolak" (Reject)
7. Approved content appears publicly

### Motivational Messaging
Motivational messages are displayed on submission and listing pages to encourage user contributions:
- Messages include the prefecture name for contextual relevance
- Styled with blue accent box (background: #e3f2fd; border-left: 4px solid #2196f3)
- Example: "Setiap komunitas yang terdaftar di Tokyo adalah hasil kontribusi warga yang peduli..."

Pages with motivational messaging:
- `/komunitas` - Community listing page (per prefecture subdomain)
- `/komunitas/tambah` - Community submission page
- `<subdomain>/` - Panduan listing page (per prefecture)
- `/panduan/tambah` - Panduan submission page

## Current Features Status
- [x] Prefecture listing (homepage)
- [x] Community pages (komunitas)
- [x] Community user submission with approval workflow
- [x] Admin authentication
- [x] Admin community management with approve/reject
- [x] Prefecture selector for community management
- [x] Panduan (guide) pages
- [x] Panduan user submission with approval workflow
- [x] Panduan admin management with approve/reject
- [x] Markdown rendering for panduan content
- [x] Math captcha for user submissions
- [x] Privacy policy notice for submissions
- [x] Motivational messaging with prefecture context
- [ ] Culinary (kuliner)
- [ ] Services (jasa)
- [ ] Marketplace (lapak)
- [ ] Jobs (loker)
- [ ] Recycling (lungsuran)
- [ ] Discussion (diskusi)

## Environment Variables
- `APP_ENV` - "development" or "production"
- Domain: "machine.local" (dev) or "warga.jp" (prod)

## Styling
- Uses Pico.css (pico.red theme)
- Custom styles in `presentation/shared/component/style.templ`
- Grid layout: `.main-grid` with `.main-aside` sidebar

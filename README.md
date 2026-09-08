# Get Levrg — Project Wishlist

A single-file web app for capturing client project wishlists, scoping them
internally, sharing read-only views, and inviting clients to fill them out.
Front end is one self-contained `index.html`; all data and auth live in Supabase.

---

## 1. What it is

| URL | Surface | Auth |
|-----|---------|------|
| `/` | **Admin app** — all wishlists, scope mapping, exports, user & client management | Team login |
| `/?form` | **Customer wishlist form** — the 5-step wizard | Access phrase (no login) |
| `/?w=<token>` | **Read-only shared wishlist** | Team login |

One file serves all three via URL routing.

## 2. Tech stack

- **Front end:** React 18 (loaded from esm.sh), JSX transpiled in-browser by Babel Standalone, Tailwind (Play CDN), framer-motion, lucide-react. No build step.
- **Backend:** Supabase — Postgres (data + row-level security), Auth (team accounts), Vault (secrets), `pg_net` (outbound email from triggers).
- **Email:** Resend (verified domain `mailgetlevrg.com`) for app emails; Supabase Auth SMTP for auth emails (see §7).
- **Hosting:** any static host. Deploy `index.html` at the site root.

## 3. Features

**Customer form (`/?form`)**
- Branded access-phrase gate (the Get Levrg "V", animated).
- 5 steps: Customer Details → Your Goals → Measure Success → Wishlist Projects → Review & Submit.
- Save draft (browser localStorage); anonymous submit.

**Admin app (`/`)**
- All Wishlists with **Active / Deleted** tabs, **soft delete + restore**, delete confirmation modal.
- Scope-mapping view: read-only client context + per-project capability mapping.
- **New Wishlist** (same wizard) and **Edit** (updates in place — no duplicates).
- Excel + Google-Sheets CSV export (formula-injection sanitized).
- **Copy link to wishlist** on each row and on the open wishlist → read-only share link.
- **User Management** (admins): magic-link invites, role changes, remove members.
- **Send Client Invite** (admin/sales): emails a client the form link + access phrase.

**Roles:** `admin`, `sales_am`, `project_manager`, `csm`, plus a hidden **super admin** (sole maintainer).

**Super admin (sole maintainer)**
- Identified by `SUPER_ADMIN_EMAIL` (UI) and an `is_super` flag (database).
- **Super Admin** panel visible only to the maintainer; account hidden from User Management and protected at the database level (see §6).
- Login accepts the maintainer's private email (e.g. Gmail) in addition to `@getlevrg.com`.

## 4. Configuration (in `index.html`)

Near the top of the script block:

| Constant | Purpose |
|----------|---------|
| `SUPABASE_URL` / publishable key | Supabase project (publishable key is safe client-side) |
| `CLIENT_PAGE_PASSWORD` | Access phrase for the customer form. Current: `GLwishlist@Submit` |
| `SUPER_ADMIN_EMAIL` | The sole-maintainer account email. **Set to your real address** (Gmail is fine) |

`SUPER_ADMIN_EMAIL` must match the email flagged `is_super` in the database (§6).

## 5. Database — SQL migrations

Run once each in the Supabase SQL editor, **in this order**. All are idempotent.

1. **`schema-v5.sql`** — base schema: `wishlists` and `profiles` tables, roles, row-level security, `current_user_role()`, role-escalation guard, `pending_invites`, and the signup trigger that creates a profile and applies a pre-authorized role.
2. **`wishlist-soft-delete.sql`** — adds `wishlists.deleted_at` (powers the Deleted tab / Restore). *Without this the wishlist list reads empty.*
3. **`wishlist-share-links.sql`** — adds `wishlists.share_token` + `get_shared_wishlist(token)` (login-only) for read-only share links.
4. **`client-invites.sql`** — `client_invites` table + Resend trigger that emails clients the form link + phrase.
5. **`wishlist-email-trigger.sql`** — emails admins on each new wishlist via Resend.
6. **`super-admin-protection.sql`** — adds `profiles.is_super`, restrictive RLS to hide/lock the maintainer row, and a trigger so only the service role can set the flag. **Edit the final `update … where lower(email) = …` to your Gmail before running.**

> Rule of thumb: if a front-end change starts reading a new column, add the column in Supabase *first*, or the whole query fails and lists read as empty.

## 6. Secrets (Supabase → Vault)

Set these once (used by the email triggers):

| Secret | Value |
|--------|-------|
| `RESEND_API_KEY` | Resend API key (`re_…`) |
| `WISHLIST_FROM_EMAIL` | `Get Levrg <noreply@mailgetlevrg.com>` |
| `WISHLIST_NOTIFY_TO` | fallback admin recipient (optional) |
| `WISHLIST_BASE_URL` | `https://gl-wishlist.netlify.app` (used to build links) |

## 7. Email — two separate paths

- **Resend via DB triggers** — new-wishlist notifications and client invites. Configured entirely in SQL + Vault.
- **Supabase Auth SMTP** — password reset, team magic-link invites, and login links. Configure under **Authentication → Emails → SMTP Settings**. Point it at Resend so everything uses one sender:
  - Host `smtp.resend.com`, Port `465`, Username `resend`, Password = your Resend API key, Sender `noreply@mailgetlevrg.com`.
  - (A Google *App Password* also works if you keep Gmail SMTP with 2FA on — a normal Gmail password will be rejected with `534 5.7.9 Application-specific password required`.)

**Deliverability:** Resend keeps a **suppression list**; an address that bounced or marked spam is silently dropped on future sends. Clear it under Resend → Suppressions for valid addresses, and keep DMARC set on the sending domain.

## 8. Deploy — from scratch

1. Set `SUPABASE_URL`, key, `CLIENT_PAGE_PASSWORD`, and `SUPER_ADMIN_EMAIL` in `index.html`.
2. Run the SQL migrations (§5) in order; set Vault secrets (§6).
3. Configure Auth SMTP (§7).
4. Create the maintainer account: **Authentication → Users → Add user** (your Gmail + password), then run the `update … is_super = true` line for that email.
5. Deploy `index.html` at the site **root** (rename/drag the single file). Admin = `/`, form = `/?form`.
6. Embed the form in WordPress (§9).

**Applying an update:** if the change is front-end only, deploy `index.html` and hard-refresh. If it references a new column/table, run that migration first.

## 9. WordPress embed (customer form)

Paste into a **Custom HTML** block; change both URLs for a custom domain.

```html
<div style="max-width:1200px;margin:0 auto;">
  <iframe id="gl-wishlist-iframe" src="https://gl-wishlist.netlify.app/?form"
    title="Project Wishlist" loading="lazy" scrolling="no"
    style="width:100%;border:0;display:block;min-height:640px;overflow:hidden;"></iframe>
</div>
<script>
(function () {
  var f = document.getElementById('gl-wishlist-iframe');
  var allowed = 'https://gl-wishlist.netlify.app';
  window.addEventListener('message', function (e) {
    if (e.origin !== allowed) return;
    if (e.data && e.data.type === 'gl-wishlist-height' && e.data.height) f.style.height = (e.data.height + 24) + 'px';
  });
})();
</script>
```

## 10. Security notes

- The publishable key is safe client-side; all enforcement is **RLS**, not the browser.
- No service key ever ships in `index.html`. Server-side actions (invites, notifications) run from DB triggers using Vault secrets.
- The maintainer account is hidden and locked at the **database** level (restrictive RLS + protected `is_super`), not just the UI.
- Keep **2FA** on the maintainer email — it's the highest-privilege account.
- Rotate the Resend API key if it was ever shared, then update the `RESEND_API_KEY` Vault secret.

## 11. Editing the code

`index.html` is one file with a single `<script type="text/babel">` block. To validate a change compiles before deploying:

```bash
npm install @babel/standalone
node -e "const b=require('@babel/standalone'),fs=require('fs');const h=fs.readFileSync('index.html','utf8');const m=h.match(/<script[^>]*text\/babel[^>]*>([\s\S]*?)<\/script>/i);b.transform(m[1],{presets:['react']});console.log('OK');"
```

## 12. File manifest

- `index.html` — the entire app.
- `schema-v5.sql`, `wishlist-soft-delete.sql`, `wishlist-share-links.sql`, `client-invites.sql`, `wishlist-email-trigger.sql`, `super-admin-protection.sql` — database migrations.
- `gl-wishlist-wordpress-embed.html` — the embed snippet.

# Testing Wad Almardi (storefront + admin)

This skill covers running and testing Wad Almardi Market locally — both the customer-facing storefront and the admin dashboard.

## Stack at a glance
- **Frontend** — Next.js 14 (App Router) + TS + Tailwind, in `frontend/`. Bilingual ar/en RTL via next-intl. Admin section at `/admin/*` is Arabic-only and bypasses next-intl middleware.
- **Backend** — Laravel 11 + PHP 8.2 + Sanctum + spatie/laravel-permission, in `backend/`.
- **DB** — MySQL 8 in Docker (compose service `mysql`, exposed on host port `3307`, credentials `wdalmardy/wdalmardy`, database `wdalmardy`).
- **PDF** — `mpdf/mpdf` (NOT dompdf — see gotcha below).

## Bring the local stack up
```bash
# 1) MySQL + phpMyAdmin
cd /path/to/Wdalmardy
docker compose up -d

# 2) Backend (always migrate:fresh --seed on a clean test, so the seeded admin user / sample data exist)
cd backend
php artisan migrate:fresh --seed --force
php artisan serve --port=8000 > /tmp/laravel.log 2>&1 &

# 3) Frontend
cd ../frontend
npm install      # only the first time
npm run dev > /tmp/next.log 2>&1 &
```

Verify both are up:
```bash
curl -s -o /dev/null -w "backend: %{http_code}\nfrontend: %{http_code}\n" \
    http://localhost:8000/api/categories http://localhost:3000/admin/login
```
Both should return `200`.

## Test accounts (seeded by `AdminSeeder`)
All passwords are `password` (local dev only; safe to put in this skill).

| Role          | Email                          |
|---------------|--------------------------------|
| admin         | `admin@wadalmardi.com`         |
| accountant    | `accountant@wadalmardi.com`    |
| driver        | `driver@wadalmardi.com`        |

The admin frontend stores the Sanctum token in `localStorage` under key `wdalmardy_admin_token`. A 401 from any admin API call auto-redirects to `/admin/login`.

## Useful URLs
- Storefront (Arabic, RTL): `http://localhost:3000/ar`
- Storefront (English, LTR): `http://localhost:3000/en`
- Admin login: `http://localhost:3000/admin/login`
- Admin dashboard: `http://localhost:3000/admin`
- phpMyAdmin: `http://localhost:8081` (server `mysql`, user `wdalmardy`, password `wdalmardy`)

## Regex / format conventions
- Order numbers — `^WD-\d{6}-\d{3}$` (e.g. `WD-260428-001`). Format is `WD-YYMMDD-NNN` and resets per day.
- Currency — render as `ج.س` (e.g. `3,500 ج.س`).
- Order status ENUM — `['new','preparing','shipped','delivered','cancelled']`. Default is `'new'`.

## Gotchas (real bugs we hit; check these first if anything looks weird)

### 1. Arabic PDF rendering
The invoice PDF endpoint (`GET /api/admin/orders/{id}/invoice`) MUST use `mpdf/mpdf` — `barryvdh/laravel-dompdf` does not implement Arabic shaping and will produce garbled (reversed, unconnected) Arabic. The Blade template in `backend/resources/views/invoices/order.blade.php` should declare `font-family: dejavusans` (mPDF's native font alias), not `"DejaVu Sans"`.

If the PDF text looks like `تكرام يضرملا دو` instead of `ود المرضي ماركت`, the controller is using dompdf again — swap it back to mPDF.

### 2. Missing `is_active` in API resources
`ProductResource` and `CategoryResource` MUST expose `'is_active' => (bool) $this->is_active`. The admin frontend filters product/category lists on this field; if it's missing the frontend treats every row as inactive ("غير نشط") even when the DB says otherwise.

### 3. Order status ENUM mismatch
The `orders.status` migration ENUM and the `Order` model's default MUST agree. The current migration uses `['new','preparing','shipped','delivered','cancelled']` with default `'new'`. If you see `Data truncated for column 'status'` on order creation, the migration drifted — fix the migration and re-run `php artisan migrate:fresh --seed --force`.

## Quick smoke test (≈ 5 minutes)
Use this whenever you change anything in admin or storefront:

1. **Login** — `/admin/login` → submit prefilled creds → land on `/admin`. KPIs should be numeric, not `—` or `NaN`.
2. **Storefront order** — open `/ar/store` → add any product → checkout → submit COD. Note the `WD-YYMMDD-NNN` order number.
3. **Dashboard refresh** — back to `/admin`, reload. "مبيعات اليوم" should reflect the new order; recent orders list should contain it.
4. **Order detail** — `/admin/orders` → click the order. Click "طباعة الفاتورة" → PDF must download (NOT open as HTML in a new tab) and Arabic must read RTL with connected letters. Click "إرسال واتساب" → `api.whatsapp.com/send/?phone=...` opens with Arabic body containing order # + total.
5. **Block customer** — `/admin/customers` → click ⛔ on the customer → confirm. Badge → `محظور`, stat "محظورون" increments. F5 — must persist.
6. **Storefront regression** — `/ar` should still load (next-intl middleware excludes `/admin`).

If all six pass, the admin layer is healthy.

## Recording during E2E tests
- Always maximize Chrome before `recording_start` (`wmctrl -r :ACTIVE: -b add,maximized_vert,maximized_horz` if available, otherwise the GUI's own maximize).
- Use `annotate_recording` with `setup` / `test_start` / `assertion` types — group related checks into a single assertion ("Sidebar collapsed", not 5 individual icon assertions).
- The video slows down around annotations, so keep them concise.

## Devin Secrets Needed
None for local testing. Production WhatsApp Business API + payment gateway will need secrets later, but storefront uses public `wa.me` / `api.whatsapp.com` URLs (no key required) and COD does not need a gateway.

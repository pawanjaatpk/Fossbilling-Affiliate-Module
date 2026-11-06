# Affiliate module for FOSSBilling

A complete referral / affiliate program: tracked links, signup attribution, one-off or recurring commissions,
hold periods, refund clawbacks, payouts (PayPal, bank, crypto, other, or instant account credit), admin tooling,
and a client-area dashboard that works with the default theme and with [Tide](https://github.com/getnamingo/tide).

Requires FOSSBilling 0.8+ (PHP 8.3+), MySQL/MariaDB.

## Preview

<img width="1201" height="475" alt="Screenshot 2026-11-06 232008" src="https://github.com/user-attachments/assets/cb0a1808-9c90-4a4b-a3fd-3e9c8e488528" />

## Install

```bash
cp -a Affiliate /var/www/modules/Affiliate
chown -R www-data:www-data /var/www/modules/Affiliate
```

1. Admin → **Extensions → Overview**: find **Affiliate** and click **Install/Activate**.
   (Tables `affiliate_*` are created; e-mail templates are registered.)
2. **Run cron once** (or press *Run cron* in admin). FOSSBilling discovers hook handlers during cron, which is what
   activates signup attribution, commission creation and refund clawbacks.
3. Admin → **Affiliates → Settings**: configure rates, hold period, payout methods.
4. Add a link to `/affiliate` in your client menu (see *Tide* below).

> Uninstalling the extension drops all `affiliate_*` tables.

## How it works

| Step | What happens |
|------|--------------|
| Click | `https://site/affiliate?ref=CODE` (optional `&to=/order/hosting`; short forms `/a/CODE`, `/ref/CODE` also work) records every real click (bots, prefetches and a refresh within 30 s are ignored; unique clicks are counted per visitor cookie, not per IP), sets `fb_affiliate` + `fb_aff_vid` cookies (HttpOnly, SameSite=Lax) and redirects with no-cache headers. |
| Signup | `onAfterClientSignUp` ties the new client to the cookie's affiliate. If the cookie is missing, a click from the same IP in the previous 24 h is used (setting *Match signups by IP*). Self-referral by the same account is always blocked; the same-IP block is optional (off by default). |
| Payment | `onAfterAdminInvoicePaymentReceived` (plus a self-healing sweep of paid invoices on cron and on traffic) calculates the commission from invoice items (per-product overrides → affiliate override → program default; deposits excluded optionally; converted to the default currency). One commission per invoice (idempotent). |
| Hold | Commission is *pending* for `hold_days`, then cron moves it to *approved* (or an admin approves it manually in *manual* mode). |
| Refund | `onAfterAdminInvoiceRefund` on a **fully refunded** invoice reverses unpaid commissions, or creates a negative adjustment if already paid/in a payout. Partial refunds: use *Manual adjustment*. |
| Payout | Affiliate requests payout of the full available balance (≥ minimum). Admin marks paid (with reference), or rejects (funds return to balance). *Account credit* payouts are added to the client's balance automatically. |

## Options

**Program:** enable/disable, registration (open / admin approval / closed), terms URL + acceptance, description text.
**Commission:** percentage or fixed; first invoice only / N months / lifetime; signup bonus; exclude deposits; refund clawback;
per-affiliate custom rate; per-product overrides (0 = excluded).
**Tracking:** cookie days, first- or last-click attribution, landing path, self-referral block, deep links.
**Payouts:** hold days, auto/manual approval, minimum payout, enabled methods, e-mail notifications.
**Admin:** dashboard stats, affiliate list/approve/suspend/reject/edit/delete, manual adjustments, referral & commission &
payout lists with filters, staff permissions `manage_payouts` and `manage_settings`.

## Tide theme

Tide is a Tabler-based client theme; the client page uses standard `card`/`table` markup and works without changes.

* Dashboard card: copy `integration/tide/dashboard.widgets.twig` to
  `themes/tide/html/custom/dashboard.widgets.twig` (survives Tide upgrades).
* Menu link: add `<a href="{{ 'affiliate'|link }}">Affiliate</a>` wherever your navigation lives (e.g. your own
  override in `html/custom/`, or a Custom Pages link).
* Override the page: place `mod_affiliate_index.html.twig` in the theme's `html_custom/` folder.

## API

`admin/affiliate/*`: `get_stats`, `get_list`, `get`, `create`, `update`, `set_status`, `delete`, `referral_get_list`,
`commission_get_list|approve|reject|batch_approve|create`, `payout_get_list|mark_paid|reject|create`,
`product_rate_get_list|set|delete`, `settings_get|update`.
`client/affiliate/*`: `get_overview`, `join`, `payout_settings_update`, `referral_get_list`, `commission_get_list`,
`payout_get_list`, `payout_request`, `payout_cancel`.
`guest/affiliate/get_program`: public terms for marketing pages.

## Notes & limitations

* Amounts are stored in the system default currency; account-credit payouts are converted to the client's currency.
* Commission base is the invoice item subtotal (before tax).
* Attribution needs the `/affiliate?ref=CODE` link (cookie based). Clients who clear cookies or switch devices are not attributed.
* Tested: PHP syntax, all Service logic (tracking, attribution, commissions, overrides, holds, cron, payouts, clawbacks)
  against MariaDB with a mocked container, and Twig parsing of every template. **Not yet run inside a live
  FOSSBilling instance** - test on staging first (signup via link → paid invoice → payout) and check
  `Settings → Email templates` lists the three `mod_affiliate_*` templates.

## Changelog

### 1.2.0 - accurate purchase counting
- **Purchase ledger (`affiliate_invoice`):** every paid invoice of a referred client is now recorded as a purchase, even when its commission is 0 (excluded product/deposit, "first invoice only" renewals, 100% discount). Before, such invoices left no row, were re-processed on every sync and - being the oldest - filled the 500-invoice sweep window, which **blocked newer paid invoices from ever being counted**. Existing commissions are back-filled into the ledger automatically.
- **Purchase stats** (clicks/signups/purchases/sales/paying referrals, admin list, dashboard) now come from the ledger instead of the commission table, so they no longer depend on commission amount or status.
- **Discounts:** invoice-level discounts (coupons) are spread over the lines, so sales and percentage commissions are no longer overstated.
- **Only `paid` invoices** are processed by the hook (it could previously count an invoice in another state).
- **Refunds:** fully refunded invoices are removed from purchase counts even when clawback is off, and a cron/traffic sweep (`reconcileRefunds`) catches refunds whose hook was missed.
- **Unique clicks:** a new browser now gets a random visitor id. The old fingerprint (IP + user agent) was written into the cookie, so every visitor behind one proxy/CDN/NAT with the same browser shared one id and counted as one unique visitor.
- If a commission insert fails, the invoice claim is released so the next sync retries instead of losing it.
- After upgrading: press *Sync now* in *Affiliates -> Settings* (use a large window, or the *days* parameter of `admin/affiliate/sync`) to record past purchases.

### 1.1.0 - accurate clicks, signups and purchases
- **Clicks:** unique visitors were de-duplicated by IP for 10 minutes, so everyone behind a shared IP / proxy / CDN counted as one click. Now every real click is stored, uniqueness uses a visitor cookie (with an IP+UA fingerprint fallback), and only refreshes within 30 s are ignored. Empty user agents, HEAD requests and browser prefetches are not counted. The redirect sends `no-store` headers so a CDN or browser cannot cache it and hide repeat clicks. `Secure` cookie flag now also works behind a reverse proxy.
- **Signups:** if the cookie is missing, the signup is matched to a click from the same IP in the previous 24 h (toggle: *Match signups by IP*). The same-IP self-referral block is now **off by default** (it silently dropped signups made from the affiliate's own network while testing). Admins can attach a client manually (`admin/affiliate/referral_assign`).
- **Purchases:** commissions no longer depend on hooks alone. `sync()` (cron, and throttled on affiliate traffic and dashboard loads) creates commissions for paid invoices of referred clients that were missed, and attributes clients missed at signup. Concurrent processing of one invoice is safe (unique key + guarded insert). The signup bonus is no longer paid on a deposit-only first invoice.
- Hooks are registered on install/update (`connectHooks()`); new **Tracking health** panel in *Affiliates → Settings* with *Register event hooks* and *Sync now*.
- New stats: unique clicks, paid invoices, sales total, purchase rate; shown in admin and client area. Schema 1.1.0 is applied automatically (new columns `visitor_id`, `is_unique`, `source`, `visit_id`).

### 1.0.8
- Fixed referral links (`/affiliate?ref=CODE`) redirecting to `/<domain>` and returning 404.
  Cause: the configured `url` had no `https://`, so the redirect `Location` header was relative.
  `getSystemUrl()` now always returns an absolute URL (adds the scheme, handles `//host` and path-only values,
  honours `X-Forwarded-Proto`, falls back to the request host).

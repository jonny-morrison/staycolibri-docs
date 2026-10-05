# Stay Colibri — Site Documentation
**Last Updated:** October 2026 (v1.3)  
**Platform:** WordPress Multisite (HivePress/RentalHive)  
**URL:** https://staycolibri.com  
**Subsite prefix:** `wp_14_`  
**Server:** Bitnami AWS Lightsail (Ubuntu)

> **Security:** this file is kept in GitHub. Never put credentials, secrets, client IDs or webhook URLs in it — write `[IN PASSWORD MANAGER]` instead.

---

## Table of Contents
1. [System Architecture](#1-system-architecture)
2. [WordPress Data Structure](#2-wordpress-data-structure)
3. [WPCode Snippets](#3-wpcode-snippets)
4. [Third Party Integrations](#4-third-party-integrations)
5. [Host Ledger and AWS S3](#5-host-ledger-and-aws-s3)
6. [Cron Jobs](#6-cron-jobs)
7. [Zapier Flows](#7-zapier-flows)
8. [Go-Live Checklist](#8-go-live-checklist)
9. [Troubleshooting](#9-troubleshooting)
10. [Known Issues and Open Items](#10-known-issues-and-open-items)

---

## 1. System Architecture

### Stack
- **WordPress Multisite** — main network at `2uw.uk`, subsite at `staycolibri.com` (blog ID 14)
- **HivePress** — listing/booking framework
- **RentalHive** — HivePress extension for rental bookings
- **WooCommerce** — payment processing (legacy post-based order storage; HPOS **not** enabled)
- **WPCode** — all custom PHP/JS snippets
- **Cloudflare** — DNS and proxy
- **WP Mail SMTP** — email via Zoho SMTP
- **Next3Cloud** — media files offloaded to Peasoup Cloud S3
- **PhpSpreadsheet** — installed at `/opt/bitnami/wordpress/vendor/autoload.php`

### Money flow (summary)
- Guest pays accommodation + 10% service fee via WooCommerce.
- **Cash hosts:** each paid booking adds an entry to `sc_cash_balance_entries`. On the first UK working day of each month, eligible entries (checkout + 5 UK working days before the batch date) plus pending adjustments are paid as USD cash in $50 multiples. The remittance fee (11% + €3.00) is charged on the delivered amount only; the remainder rolls over gross.
- **Tropipay hosts:** a payout record is created at checkout; the transfer runs automatically 5 UK working days after checkout, net of 3.5% + €0.50, with any pending adjustments applied.
- **Host Ledger:** an append-only SQL ledger records every financial event. It is reconciled nightly against the payout system and rendered to XLSX in S3.

### Key URLs — public
| Page | URL |
|---|---|
| Home | https://staycolibri.com |
| Properties | https://staycolibri.com/properties/ |
| Coming Soon | https://staycolibri.com/coming-soon/ |
| Why Colibri (hosts) | https://staycolibri.com/whycolibri/ |
| Contact | https://staycolibri.com/contact/ |
| Payment Setup | https://staycolibri.com/payment-setup/ |
| Host Dashboard | https://staycolibri.com/account/vendor/dashboard/ |
| Booking Details | https://staycolibri.com/make-booking/details/ |
| Verification Pending | https://staycolibri.com/verification-pending/ |
| Submit Listing | https://staycolibri.com/submit-listing/ |

### Key URLs — admin tools (must be logged in as admin)
| Tool | URL | Snippet |
|---|---|---|
| Ledger admin (backfill, render, reconcile status) | `/?sc_ledger_admin=1` | 3.15 |
| Full ledger for a host (live from SQL) | `/?sc_ledger_view={user_id}` | 3.24 |
| Download host XLSX (as of last render) | `/?sc_ledger_download={user_id}` | 3.24 |
| Render XLSX now | `/?sc_rebuild_ledger=1&host_id={user_id}` or `host_id=all` | 3.15 |
| Pending cash balance | `/?sc_cash_balance={user_id}` | 3.24 |
| Reconcile cash balance | `/?sc_cash_reconcile={user_id}` | 3.24 |

All of these are also linked from the **Stay Colibri — Financial Tools** box on the Edit Host screen.

### Admin Access
- WordPress Admin: https://staycolibri.com/wp-admin
- phpMyAdmin: via Bitnami server local access
- WPCode Snippets: WordPress Admin → Code Snippets
- Host financial tools: WordPress Admin → Hosts → edit host → **Stay Colibri — Financial Tools** box

---

## 2. WordPress Data Structure

### Database Tables (subsite prefix `wp_14_`)
| Table | Purpose |
|---|---|
| `wp_14_posts` | All posts including listings, bookings, payouts, orders |
| `wp_14_postmeta` | Post metadata |
| `wp_14_terms` | Taxonomy terms (areas, categories etc.) |
| `wp_14_term_taxonomy` | Taxonomy definitions |
| `wp_14_term_relationships` | Links posts to taxonomy terms |
| `wp_usermeta` | User metadata (network-level, no prefix) |
| `wp_14_sc_ledger` | **Host Ledger** — append-only financial lines (see 3.15) |
| `wp_14_sc_ledger_status` | Current status of each booking (the only mutable ledger data) |
| `wp_14_sc_ledger_status_log` | Append-only history of every status change |

**Database triggers** (`wp_14_sc_ledger_no_update`, `wp_14_sc_ledger_no_delete`, `wp_14_sc_ledger_status_log_no_update`, `wp_14_sc_ledger_status_log_no_delete`) reject any UPDATE or DELETE on the ledger and status log tables with the error "Stay Colibri ledger is append-only". `TRUNCATE` and `DROP` are not blocked by triggers; the ledger admin Reset uses `TRUNCATE` and is disabled once the ledger is marked live.

### Post Types
| Post Type | Description |
|---|---|
| `hp_listing` | Property listings |
| `hp_booking` | Booking records (trashed = cancelled) |
| `hp_vendor` | Host (vendor) profile; `post_author` is the host's WordPress user ID |
| `hp_payout` | Payout records (created by cron) |
| `shop_order` | WooCommerce orders |

### User Meta Keys
| Meta Key | Values | Description |
|---|---|---|
| `hp_acctype` | `107` = Guest, `108` = Host | Account type |
| `hp_verified` | `1` = verified | Didit verification status |
| `hp_phone` | string | Phone number (from Didit) |
| `hp_carne` | string | Cuban ID number (from Didit) |
| `first_name` | string | First name (from Didit) |
| `last_name` | string | Last name (from Didit) |
| `sc_payment_model` | `tropipay` or `cash` | Host payout method |
| `sc_payment_model_locked` | `1` = locked | Prevents host changing payout method |
| `sc_tropipay_email` | email | Host's Tropipay account email |
| `sc_cash_address` | string | Host's cash delivery address |
| `sc_tropipay_deposit_account_id` | integer | Tropipay beneficiary ID for this host |
| `sc_payouts_locked` | `1` = paused | Pauses all payout processing for the host |
| `sc_cash_balance_entries` | serialised array | Cash host working balance (structure below) |
| `sc_cash_balance_entries_backups` | serialised array | Last 10 snapshots taken by the Reconcile tool before each write |
| `sc_cash_adjustments` | serialised array | Admin balance adjustments, both payout methods (structure below) |
| `sc_ledger_rendered_at` | `Y-m-d H:i:s` UTC | When the host's XLSX was last rendered to S3 |

**`sc_cash_balance_entries` — one array per entry**
```
order_id       WooCommerce order ID; 0 = rollover (carried balance)
amount         EUR, gross accommodation (rollovers may be negative)
checkout_date  Unix timestamp (rollovers: batch day minus 1 second)
status         'pending' or 'paid' (paid = consumed by a batch or a carry-forward)
note           optional, e.g. 'Rollover from November 2026'
batch_ref      set when paid (V3.4+): '#{payout_id}' or 'rolled-forward-YYYY-MM'
payout_ref     on rollovers created by a cash batch (V3.4+): the batch's payout ID
```

**`sc_cash_adjustments` — one array per adjustment**
```
adj_id      ADJ-XXXXXXXX (unique reference)
amount      float, positive (credit) or negative (deduction)
reason      string, sent to the host
date        j F Y
admin_id    WordPress user ID of the admin (0 = System)
admin_name  display name ('System' for cron-created carries)
status      'pending' or 'paid'
batch_ref   payout reference when paid; 'rolled-forward-YYYY-MM' if consumed in a no-delivery month;
            '#{payout_id} (zero payout)' if consumed by a Tropipay payout that netted to zero
```

### Order Meta Keys
| Meta Key | Description |
|---|---|
| `_sc_host_user_id` | WordPress user ID of the host |
| `_sc_host_payment_model` | Payment model at time of booking |
| `_sc_accommodation_amount` | Current accommodation amount (reduced by partial refunds) |
| `_sc_original_accommodation_amount` | Accommodation amount at payment — never changed (V3.5+) |
| `_sc_checkout_date` | Guest checkout timestamp |
| `_sc_payout_processed` | `0` = no payout record yet, `1` = payout record created (Tropipay) |
| `_sc_vendor_id` | Vendor post ID |
| `_sc_payout_post_id` | ID of associated hp_payout record (Tropipay) |

### Payout Post Meta Keys — Tropipay
| Meta Key | Description |
|---|---|
| `hp_amount` | Net amount transferred (after Tropipay fee) |
| `sc_payout_method` | `Tropipay Transfer` |
| `sc_order_id` | Associated WooCommerce order ID |
| `sc_checkout_date` | Guest checkout timestamp |
| `sc_payment_date` | Unix timestamp of scheduled transfer date (checkout + 5 UK working days) |
| `sc_base_accommodation_amt` | Accommodation amount before adjustments (V3.4+) — adjustments are always applied to this |
| `sc_accommodation_amt` | Accommodation amount after pending adjustments (written by Section 7) |
| `sc_adjustment_total` | Adjustments applied to this payout |
| `sc_tropipay_fee` | Tropipay fee (3.5% + €0.50 on the adjusted amount) |
| `sc_tropipay_transfer_id` | Tropipay transfer ID (after successful payout) |
| `sc_tropipay_transfer_reference` | Tropipay transfer reference |
| `sc_tropipay_transfer_state` | Tropipay transfer state at creation |

### Payout Post Meta Keys — Cash batch
| Meta Key | Description |
|---|---|
| `hp_amount` | **EUR deducted from the host balance** = delivered EUR × 1.11 + €3.00 |
| `sc_payout_method` | `USD Cash Delivery` |
| `sc_total_eur` | Gross monthly balance (eligible entries + adjustments) |
| `sc_booking_total` | Eligible booking entries (incl. rollovers) |
| `sc_adjustment_total` | Pending adjustments included |
| `sc_payable_usd` | USD delivered (multiple of $50) |
| `sc_payable_eur` | EUR equivalent of the USD delivered (V3.4+) |
| `sc_fee_amount` | Remittance fee: 11% of delivered EUR + €3.00 (V3.4+; **pre-V3.4 batches stored 11% of the whole balance + €3.00**) |
| `sc_deduction_eur` | Same as `hp_amount` (V3.4+) |
| `sc_remainder_eur` | Carried forward to next month, gross (V3.4+) |
| `sc_eur_usd_rate` | EUR/USD rate used |
| `sc_rate_fallback` | `1` if the exchange-rate API failed and 1.08 was used |
| `sc_calc_version` | `3.4` for batches calculated by `sc_cash_batch_calc()` |
| `sc_net_eur`, `sc_net_usd`, `sc_remainder_usd` | **Legacy, superseded.** Still written by V3.4+ (as payable EUR, payable USD, and carried EUR × rate) for backwards compatibility. Do not use in new code |

### Options
| Option | Description |
|---|---|
| `sc_cash_batch_last_run` | `YYYY-MM` of the last cash batch run — prevents a second run in the same month |
| `sc_ledger_db_version` | Ledger schema version (`2.0`) |
| `sc_ledger_triggers` | `active`, or the names and errors of any missing triggers |
| `sc_ledger_backfilled` / `sc_ledger_backfilled_at` | `1` and timestamp once the one-time backfill has run (ledger hooks are inactive until then) |
| `sc_ledger_live` | `1` once the ledger is marked live (Reset disabled permanently) |

### Transients
| Transient | Lifetime | Description |
|---|---|---|
| `sc_uk_bank_holidays` | 24 h | UK bank holidays from gov.uk |
| `sc_dash_eur_usd_rate` | 6 h (15 min on API failure) | EUR/USD rate for the host dashboard estimate |
| `sc_cancel_reason_{booking_id}` | 5 min | Host cancellation reason (3.22), read by 3.22 and the ledger |

### Taxonomies
| Taxonomy | Description |
|---|---|
| `hp_listing_area` | Hierarchical location taxonomy. Parent terms = provinces, child terms = areas |
| `hp_listing_category` | Listing categories (Homestay, Apartments, Villas) |
| `hp_listing_electricity` | Electricity supply attribute |
| `hp_listing_pets` | Pets policy attribute |
| `hp_listing_tags` | Listing tags (City, Beach, Countryside) |

### hp_listing_area Term IDs
| ID | Name | Parent |
|---|---|---|
| 121 | La Habana | — |
| 122 | La Habana Vieja | 121 |
| 123 | Vedado | 121 |
| 124 | Miramar | 121 |
| 125 | La Habana del Este | 121 |
| 127 | Pinar del Río | — |
| 128 | Artemisa | — |
| 129 | Matanzas | — |
| 130 | Cienfuegos | — |
| 131 | Villa Clara | — |
| 132 | Sancti Spiritus | — |
| 133 | Ciego de Ávila | — |
| 134 | Camagüey | — |
| 135 | Las Tunas | — |
| 136 | Granma | — |
| 137 | Holguín | — |
| 138 | Guantánamo | — |
| 139 | Santa Clara | 131 |

---

## 3. WPCode Snippets

### Snippet inventory — financial snippets
All are PHP snippets set to **Run Everywhere**. Each has its version in its header comment; increment it on every change.

| Section | Snippet | Current version |
|---|---|---|
| 3.6 | Payout System (cron) | **V3.5** |
| 3.7 | Dashboard Balance Block | **V3.9** |
| 3.15 | Host Ledger | **V2.0.4** |
| 3.24 | Admin Financial Tools | **V2.3** |

**Retired — must be deactivated** (function names clash with the snippets above): V1 Host Ledger, V1.4 Admin Financial Tools (formerly V1.3 Admin Balance Adjustments), V1.0 Cash Balance Reconcile, V1.1 Cash Balance Admin Tool.

**Dependencies:** 3.7, 3.15 and 3.24 call functions defined in 3.6. 3.24 calls functions defined in 3.15. All cross-snippet calls are guarded with `function_exists()`.

---

### 3.1 Didit Verification Endpoint
**Type:** PHP  
**Hook:** REST API endpoint  
**Trigger:** Zapier webhook on Didit approval  
**Endpoint:** `POST /wp-json/custom/v1/verify-user`  
**What it does:**
- Sets `hp_verified = 1` on the user
- Writes `first_name`, `last_name`, `hp_phone`, `hp_carne` from Didit webhook payload

**Troubleshooting:**
- If verification is not working, check Zapier Zap 3 is active
- Check the webhook URL in Zapier matches the site URL
- Verify the user exists in WordPress before the webhook fires

---

### 3.2 Submit Listing Access Control
**Type:** PHP  
**Hook:** `template_redirect`  
**What it does:** Intercepts all `/submit-listing*` URLs and redirects based on user status:
- Not logged in → `/whycolibri/`
- Logged in as guest (107) → `/guest-host/`
- Logged in as unverified host (108, `hp_verified` not 1) → `/verification-pending/`
- Verified host → allowed through

**Troubleshooting:**
- If hosts cannot access submit listing, check `hp_verified` is set to `1` in user meta
- Check `hp_acctype` is `108` for host accounts

---

### 3.3 Listing Page Map Notice
**Type:** PHP  
**Hook:** `wp_footer`  
**What it does:** Injects a JS notice above the Leaflet map on listing pages explaining that the pin shows an approximate location.

---

### 3.4 Payment Setup Page (V3)
**Type:** PHP  
**Shortcode:** `[sc_payment_setup_form]`  
**Page:** https://staycolibri.com/payment-setup/  
**What it does:**
- Shows a form for hosts to choose their payout method
- Option 1: Tropipay Transfer — collects Tropipay email
- Option 2: USD Cash Delivery — collects delivery address
- Form is locked after submission (`sc_payment_model_locked = 1`)
- To unlock for a host: WordPress Admin → Users → edit host → uncheck Form Locked

**Admin fields (visible on user edit page):**
- Payment Model (dropdown)
- Form Locked (checkbox)
- Tropipay Email
- Cash Delivery Address

**Dashboard notice:** If a host has not completed payment setup, a warning notice appears on all `/account/` pages.

**Troubleshooting:**
- If a host needs to change their payment method, uncheck Form Locked in their user profile
- If the form is not showing, check the shortcode `[sc_payment_setup_form]` is on the payment-setup page

---

### 3.5 V3 Gateway Routing
**Type:** PHP  
**Hook:** `woocommerce_available_payment_gateways`  
**What it does:**
- At checkout, reads the host's payment model from `sc_payment_model`
- If tropipay → hides Mollie gateway, shows Tropipay
- If cash → hides Tropipay gateway, shows Mollie

**Function:** `sc_get_cart_host_payment_model_v3()` reads host payment model fresh from DB on every call.

**Troubleshooting:**
- If wrong gateway shows at checkout, check `sc_payment_model` on the host's user profile
- Check the WooCommerce session is not caching a stale value

---

### 3.6 V3.5 Payout System (MAIN CRON SNIPPET)
**Type:** PHP  
**This is the most critical snippet on the site.** It defines the helper functions used by 3.7, 3.15 and 3.24.

**Section 1 — `woocommerce_payment_complete` (priority 10)**  
Stores order meta: `_sc_host_user_id`, `_sc_host_payment_model`, `_sc_accommodation_amount`, `_sc_original_accommodation_amount`, `_sc_vendor_id`, `_sc_checkout_date`, `_sc_payout_processed = 0`. For cash hosts, adds an entry to `sc_cash_balance_entries`. Accommodation amount = the order item's `_line_total`. Assumes one booking per order.

**Section 2 — Cron schedules**  
Registers the `sc_daily` interval (86400 s). Both `sc_daily_payout_cron` and `sc_monthly_payout_cron` are scheduled **daily at 06:00 UTC**. V3.4 migrated `sc_monthly_payout_cron` off the old 30-day `sc_monthly` interval, which drifted away from the 1st of the month and silently skipped batches. On first load, it sets `sc_cash_batch_last_run` to the current month so no batch fires mid-month.

**Section 3 — `sc_get_uk_holidays()` / `sc_is_uk_working_day($ts)`**  
UK bank holidays from https://www.gov.uk/bank-holidays.json, cached 24 h in `sc_uk_bank_holidays`. Weekends and bank holidays are non-working days.

**Section 4 — `sc_add_uk_working_days($ts, $days)`**  
Adds N UK working days.

**Section 5 — `sc_first_uk_working_day_of_month($ts = null)`**  
First UK working day of the month containing `$ts` (defaults to the current month). Before V3.4 the argument was ignored.

**Section 5d — `sc_cash_batch_calc($gross, $rate)` — single source of truth for cash maths**  
Used by Section 8 and the dashboard (3.7), so the two cannot drift apart.
- The fee (11% + €3.00) is charged on the **delivered amount only**.
- Delivery = the largest $50 multiple such that `round(payable_usd / rate, 2) × 1.11 + 3.00 ≤ gross`. It starts one $50 step above the exact cap `(gross − 3) / 1.11 × rate` and steps down until the cent-rounded deduction fits.
- Remainder carried forward **gross** = gross − deduction.
- Returns `scenario` (`payout` | `under50` | `negative`), `payable_usd`, `payable_eur`, `fee_amount`, `deduction_eur`, `remainder_eur`.
- Tested against 200,000 random balances and rates: it never over-draws the balance and never leaves a deliverable $50 behind.

**TOTP — `sc_generate_totp($secret)`**  
Generates the 6-digit TOTP from `SC_TROPIPAY_TOTP_SECRET` (wp-config.php). Used as `securityCode` on live Tropipay payouts.

**Section 5b — `woocommerce_order_refunded` (priority 10)**  
Handles host front-end and wp-admin refunds.
- The service fee is `order total − _sc_original_accommodation_amount`. Before V3.5 the already-reduced amount was used, so a second partial refund over-deducted the host.
- New accommodation = full refund ? 0 : `(order total − total refunded) − service fee`.
- **Post-checkout** (today > checkout date): emails jonny@auno.uk and touches no payout records.
- **Tropipay, no payout record yet:** updates `_sc_accommodation_amount` so Section 6 pays the reduced amount (V3.5; before that the refund was ignored and the full amount paid). A full refund also sets the order to Refunded, which Section 6 excludes.
- **Tropipay, pending payout:** full refund deletes the payout; partial refund recalculates `hp_amount`, `sc_accommodation_amt`, `sc_base_accommodation_amt` and `sc_tropipay_fee`.
- **Tropipay, payout already transferred:** emails jonny@auno.uk (manual clawback).
- **Cash, pending entry:** full refund removes the entry; partial refund sets it to the new amount.

**Section 5c — Payout pause**  
`sc_payouts_locked = 1` skips the host in Sections 6, 7 and 8. Set in the Financial Tools box (3.24). The dashboard shows "Pagos en pausa / Payouts paused", and ledger statuses show "Payouts paused".

**Section 6 — Daily: create Tropipay payout records**  
Finds Tropipay orders (processing/completed, `_sc_payout_processed = 0`) whose `_sc_checkout_date` equals midnight today or tomorrow. Creates a pending `hp_payout` with net amount (accommodation − 3.5% − €0.50), `sc_payment_date` = checkout + 5 UK working days and `sc_base_accommodation_amt`, then emails the host in Spanish. **Known issue — see section 10:** orders whose checkout day is missed by the cron never get a payout record.

**Section 7 — Daily: Tropipay transfers**  
For pending Tropipay payouts with `sc_payment_date ≤ today`:
- Pending adjustments are added to the **base** amount (`sc_base_accommodation_amt`), never to the previously adjusted figure. This fixes the pre-V3.4 bug where a failed transfer double-applied adjustments on retry. The fee is then recalculated.
- Adjusted amount ≤ 0: the payout is deleted, the adjustments are marked paid (`#{id} (zero payout)`), and any negative remainder is carried forward as a new system adjustment (with an `adj_id` from V3.5). jonny@auno.uk is emailed.
- Otherwise: gets a token, finds or creates the beneficiary, and transfers via `POST /operations/payout` with the TOTP `securityCode`. On success the payout is published, the transfer ID, reference and state are stored, and the adjustments are marked paid. On failure jonny@auno.uk is emailed.

Sandbox/live is controlled by `SC_TROPIPAY_SANDBOX` in wp-config.php.

**Section 8 — Monthly cash batch**  
The hook ticks daily. The batch runs **once per month, on or after the first UK working day**: it claims the month in `sc_cash_batch_last_run` before processing, and catches up if the batch day was missed. The batch date is the eligibility cut-off, so a late run pays exactly what was due on batch day.
- Eligible entries: `status = pending` and checkout + 5 UK working days < batch date. Pending adjustments are added. Gross total = sum.
- Calculation by `sc_cash_batch_calc()`.
- **No delivery** (gross ≤ 0, or under $50 after fee): entries and adjustments are marked paid with `batch_ref = rolled-forward-YYYY-MM`, the whole gross is carried forward as a rollover entry, and no fee is charged.
- **Delivery:** creates a pending `hp_payout` (meta in section 2), marks entries paid with `batch_ref = #{payout_id}`, adds a rollover entry with `payout_ref` for the remainder, and marks adjustments paid.
- EUR/USD from Frankfurter. If the API fails, 1.08 is used and the email subject and body are flagged **RATE FALLBACK**.
- Emails jonny@auno.uk with, per host: delivered USD, EUR equivalent, fee, deduction, carried forward and delivery details, plus a list of hosts carried forward with no delivery. Late runs are flagged.

To force a catch-up run for the current month, set `sc_cash_batch_last_run` to the **previous** month (for example `2026-09`). Deleting the option just re-creates it as the current month.

**Section 9 — Cash payout published by admin (`save_post`, priority 999)**  
For V3.4+ batches only (`sc_calc_version` set): if `hp_amount` was edited before publishing, re-syncs **only the rollover entry linked to that payout** (`payout_ref`) to `sc_total_eur − hp_amount`. If that rollover was already consumed by a later batch, it emails jonny@auno.uk with the difference to add as an adjustment. Pre-V3.4 batches are left alone. (V3.3 deleted every "Rollover from" entry, which could wipe another month's carried balance.)

**Troubleshooting:**
- Tropipay payouts not firing: check `SC_TROPIPAY_SANDBOX`; payout records exist with `post_status = pending` and `sc_payout_method = Tropipay Transfer`; `sc_payment_date ≤ today`; host not paused.
- Cash batch did not run: WP Crontrol should show `sc_monthly_payout_cron` as **Once Daily**. Check `sc_cash_batch_last_run` in `wp_14_options`; if it already holds the current month, the batch has run.
- UK bank holidays are auto-fetched; if the API is down, the transient may be empty and only weekends are excluded.
- To trigger manually: temporary snippet with `do_action('sc_daily_payout_cron')` or `do_action('sc_monthly_payout_cron')`.

---

### 3.7 V3.9 Dashboard Balance Block
**Type:** PHP  
**Hook:** `wp_footer` on `/account/vendor/dashboard/`

**Tropipay hosts — "Fondos Pendientes"** = total upcoming revenue, with no time horizon (Tropipay pays per booking):
- all pending Tropipay payout records (base amount), plus all paid orders with `_sc_payout_processed = 0` (any checkout date), each net of 3.5% + €0.50;
- plus pending adjustments × 0.965 (Section 7 adds adjustments before the 3.5% fee), with an "Incluye ajuste de €X" line.

**Cash hosts — "Próxima Entrega en Efectivo"** = estimated USD cash delivery at the next batch:
- Next batch = next first UK working day of the month (`sc_dash_first_working_day()`, self-contained).
- Eligible = pending entries with checkout + 5 UK working days before that date, plus pending adjustments. The calculation is done by `sc_cash_batch_calc()` at today's rate (Frankfurter, cached 6 h in `sc_dash_eur_usd_rate`, fallback 1.08).
- Headline: "$650 USD". Sub-line: "de un saldo mensual de €X" with an amber (i) button opening the breakdown modal. Entries eligible after the next batch are shown as "Reservas posteriores: €X".
- Modal (Spanish): Saldo trasladado, Ingresos de este mes, Ajustes, **Saldo mensual**, Entrega en efectivo (USD), Equivalente en EUR (with rate), Comisión de remesa (11% + €3,00, on the delivered amount), **Trasladado al próximo mes** (gross EUR, identical to the rollover entry Section 8 writes). Notes: estimate at today's rate; $50 multiples; fee on delivered amount only; no-delivery explanation where relevant.
- Under the balance: "Próximo pago: {date}".

**Both:** the Request a Payout button is hidden and replaced. Payouts paused shows an amber bilingual notice. Negative amounts display as "−€X".

**Order lookup:** uses `sc_ledger_find_orders()` (3.15), with a direct SQL fallback. **Do not use `wc_get_orders()` with `meta_query` on this site** — with legacy order storage it ignores the meta filter and returns every order (V3.3–V3.8 summed every unprocessed order on the site into each Tropipay host's balance).

**Troubleshooting:**
- Wrong cash figure: compare with the "Next batch balance" on `/?sc_cash_balance={id}`, which uses the same rule.
- Modal date wrong: check `sc_is_uk_working_day()` is available (3.6 loaded).
- "Call to undefined function": 3.6 not loaded — check WPCode snippet status.

---

### 3.8 Coming Soon System
**DEACTIVATE AT LAUNCH**  
**Type:** PHP  
**What it does:**
- template_redirect: Redirects all non-logged-in visitors to `/coming-soon/` except `/coming-soon/`, `/whycolibri/`, `/contact/`, REST and AJAX
- Shortcode [sc_coming_soon]: Standalone coming soon page with email signup (Brevo list 3) and host CTA
- Registration locked to Host (108) only
- `/whycolibri/` gets Sign Up as a Host CTA

**Troubleshooting:**
- If logged-in users are being redirected, check they are actually logged in
- If the email signup is not working, check Brevo API key in the snippet

---

### 3.9 Why Colibri Page
**Type:** PHP  
**Hook:** wp_head and wp_footer  
**What it does:**
- Hides HivePress hero banner, Home/Properties/News nav items, footer, List a Property button
- Fixes logo to S3 URL
- List a Property button opens register modal
- Marketing page content in Spanish

---

### 3.10 Account Settings Page Modifications
**Type:** PHP  
**What it does:**
- Guests: First/last name mandatory, carne field hidden
- Hosts: First/last/carne disabled with note to contact info@staycolibri.com
- Account type dropdown replaced with plain text (hidden input preserves value)
- Server-side: re-saves `hp_acctype` on every update

---

### 3.11 Register Modal Warning (/whycolibri/ only)
**Type:** PHP  
**Hook:** `wp_footer` on `/whycolibri/`  
**What it does:** Injects bilingual (Spanish/English) hosts only warning into register modal.

---

### 3.12 Contact Form Styling
**Type:** PHP  
**Hook:** `wp_head` on `/contact/`  
**What it does:**
- Styles Contact Form 7 form as a card
- Amber focus states matching site colours
- Bold red/green success/error messages
- Hides auto-populated user-name field

---

### 3.13 Contact Page Nav Restriction
**DEACTIVATE AT LAUNCH**  
**Type:** PHP  
**Hook:** `wp_head` on `/contact/`  
**What it does:** Hides Home/Properties/News nav items, footer, hero banner, List a Property button. Fixes logo to S3 URL. Matches `/whycolibri/` nav experience during pre-launch.

---

### 3.14 Payout Cancel Button Hidden
**Type:** PHP  
**Hook:** `wp_footer` on `/account/vendor/payouts/`  
**What it does:** Hides `.hp-payout__action--cancel` button so hosts cannot cancel their own payouts.

---

### 3.15 V2.0.4 Host Ledger
**Type:** PHP  
**Replaces:** V1 Host Ledger (which regenerated each XLSX from orders on every event). **Deactivate V1 first.**

#### Principles
- **The ledger is the table `wp_14_sc_ledger`.** Lines are inserted once and never updated or deleted (enforced by database triggers). Corrections are new lines.
- The **only mutable data is each booking's status**, held in `wp_14_sc_ledger_status`. Every change is appended to `wp_14_sc_ledger_status_log`.
- **The XLSX in S3 is a rendered view**, re-rendered nightly for hosts with changes, or on demand.
- **Duplicate protection:** every line has a unique `source_key` (e.g. `booking:957`, `refund:1023`, `payout:1100`, `adj:ADJ-3F9A21C0`), so hooks firing twice cannot duplicate lines.
- **Tamper evidence:** each line stores `prev_hash` and `line_hash` (SHA-256 per host). `sc_ledger_verify_chain($host)` reports "Verified" or the first broken line.
- **Reconciliation:** ledger balance (sum of `host_amount`) must equal the payout system balance (pending cash entries + pending adjustments + Tropipay pending payout base amounts + unprocessed Tropipay orders). This is checked nightly, with an email to jonny@auno.uk on mismatch.
- **Fees are recorded when actually charged**, on payout lines. Booking lines are gross, so the running balance is exactly what the host is owed.
- English only (back-of-house).

#### Line types
| Type | Written when | Host amount |
|---|---|---|
| BOOKING | `woocommerce_payment_complete` (priority 30, after 3.6) | + accommodation |
| REFUND | Each WooCommerce refund, one line per refund ID (priority 5, before 5b) | − host's share; €0 with a flag when 5b does not change the balance (post-checkout, Tropipay already transferred, cash already paid) |
| CANCELLATION | Booking trashed (`trashed_post`, priority 5, before 3.22) | €0 — notes include the host's reason if recorded |
| ADJUSTMENT | Adjustment added (detected on write to `sc_cash_adjustments`) | ± amount |
| ADJUSTMENT REVERSAL | Pending adjustment deleted (with the admin's reason) or its amount changed | Opposite of the original |
| PAYOUT — TROPIPAY | Successful transfer (`sc_tropipay_transfer_state` written) | − adjusted gross; fee and EUR delivered shown |
| PAYOUT — CASH BATCH | Batch created (after Section 8) | − deduction; fee, EUR and USD delivered, carried amount in notes |
| NO DELIVERY | Cash month with no delivery | €0 — amount carried in notes |
| PAYOUT NETTED | Tropipay payout deleted by Section 7 because adjustments netted it to ≤ 0 | €0 — carried amount in notes |
| CORRECTION | Cash balance entry changed outside the payout system (Pending Cash Balance delete, Reconcile apply/add/restore, Section 9 rollover re-sync); opening correction at backfill; record of a paid adjustment removed | ± difference |
| ORDER DELETED | Paid order deleted (`before_delete_post` / `woocommerce_before_delete_order`), or found missing by the nightly sweep | €0 — balance changes only when its entry or payout record is removed; email sent |

**How changes to user meta are captured:** the ledger watches `add/update/delete_user_meta` for `sc_cash_adjustments` and `sc_cash_balance_entries` before the write. It ignores changes made inside `woocommerce_payment_complete`, `woocommerce_order_refunded` and `sc_monthly_payout_cron`, because those are already recorded as BOOKING, REFUND and PAYOUT lines.
- Order entries: the ledger is brought to the new state, so CORRECTION = new entry amount − ledger total for that order. For orders with no BOOKING line (pre-ledger orders whose amount came in through the opening correction), the CORRECTION is the change itself (V2.0.2).
- Rollover entries: CORRECTION = change in the total pending rollover amount.
- System carry adjustments created by Section 7 ("Negative rollover from payout #…") are not recorded as new money; the netting is shown as PAYOUT NETTED.

#### Booking statuses
| Status | Meaning |
|---|---|
| Booked | Paid, before check-in |
| In stay | Check-in ≤ today < check-out |
| Checked out | Within 5 UK working days of checkout (detail: eligible-from date) |
| Eligible — awaiting payout | Cash: waiting for the batch (detail: batch date). Tropipay: transfer scheduled, or "payout record not yet created" |
| Payout processing | Tropipay transfer due/retrying; cash batch created, awaiting delivery confirmation |
| Paid out | Tropipay transferred, or cash batch published |
| Carried forward | Cash: consumed in a no-delivery month; follows the carry chain to Paid out |
| Settled | Netted to zero against adjustments |
| Payouts paused | Host paused (detail: underlying stage) |
| Partially refunded — {stage} | Prefix on any stage |
| Refunded | Fully refunded |
| Cancelled — refunded / partially refunded / no refund | Booking cancelled |
| Not in cash balance | Cash order with no balance entry — check Reconcile |
| Order in trash / Order deleted | Order trashed or deleted |

Statuses are updated immediately by event hooks and swept nightly for time-based changes (In stay, Checked out, Eligible).

#### XLSX layout (S3 `staycolibri-ledgers`, key `ledgers/{user_id}_{slug}.xlsx`)
- **Sheet "Ledger"**, rows 1–10: host details, ledger balance, payout system balance, Reconciled (Yes / NO with difference), Hash chain (Verified / BROKEN), line counts.
- Row 11 headers, row 12+ one row per line in recording order: Line, Recorded (UTC), Event date (UTC), Type, Status (booking lines only), Order #, Booking #, Reference, Guest user ID, Property, Check-in, Check-out, Nights, Guest paid (€), Service fee (€), Host amount (€), Payout fee (€), Delivered (€), Delivered ($), Host balance (€) (running `=SUM` formula), Notes.
- **Order #, Booking # and Guest user ID are hyperlinks** to the order, booking and user pages in wp-admin. Cancelled bookings link to the bookings trash list. Deleted items are plain text. Guest checkouts show "Guest checkout".
- Colour by type; imported lines in italics; payout lines bold.
- **Sheet "Status history":** every status change, with order/booking links.

#### Nightly job — `sc_ledger_nightly`, 03:00 UTC
Payout scan (records any payout not yet in the ledger), then status sweep (also detects orders deleted without a hook), then reconciliation (email on mismatch), then render the hosts whose lines or statuses changed since their last render (email if an S3 upload fails).

#### Admin page — `/?sc_ledger_admin=1`
Shows backfill status, line count, trigger status, mode (pre-launch/live), next nightly run, and hosts awaiting render. Per host: lines, ledger €, system €, Reconciled, Hash chain, last rendered, Render now. Actions: **Run one-time backfill** (only while the table is empty), **Render all XLSX now**, **Scan payouts + refresh statuses**, **Re-check tables / triggers**; pre-launch only: **Reset ledger** (type RESET) and **Mark ledger live** (permanently disables Reset).

#### One-time backfill
Imports existing orders, refunds, cancellations, adjustments and payouts per host in date order (marked `[Imported]`). It then adds an **opening balance CORRECTION** so each host starts reconciled; this absorbs pre-ledger deletions, legacy rollovers and test data. Ledger hooks are inactive until the backfill has run.

#### Shared functions used by other snippets
`sc_ledger_ready()`, `sc_ledger_host_balance()`, `sc_ledger_system_balance()`, `sc_ledger_verify_chain()`, `sc_ledger_refresh_host_statuses()`, `sc_ledger_find_orders($meta, $statuses)` (order lookup by meta that works on legacy storage), `sc_ledger_adj_key()`, `sc_ledger_order_admin_url()`, `sc_ledger_booking_admin_url()`, `sc_ledger_user_admin_url()`, `sc_ledger_s3_key()`, `sc_s3_download()`. Filter: `sc_ledger_adjustment_delete_reason` (used by 3.24 to put the deletion reason on the reversal line).

**Troubleshooting:**
- "Triggers: Not active — already exists": two page loads raced on install. Click **Re-check tables / triggers** (V2.0.1+ serialises install and reads the status from `information_schema`).
- Triggers fail with a privilege error: the ledger still works. Create the triggers as root in phpMyAdmin.
- Host shows **Reconciled: No**: open `/?sc_ledger_view={id}` and look for recent changes made outside the tools. Typical causes are a direct database edit or a change made before the latest version was deployed. Correct with an adjustment, or (pre-launch) Reset + backfill.
- Hash chain broken: a line was altered outside WordPress. Investigate before doing anything else.
- XLSX not updating: it renders at 03:00 UTC only when something changed. Use Render now. If the upload fails, check AWS credentials and the PhpSpreadsheet install (`ls /opt/bitnami/wordpress/vendor/phpoffice/phpspreadsheet`).

---

### 3.16 Home Page Location Dropdowns
**Type:** PHP  
**Hook:** `wp_footer` on home page  
**What it does:**
- Replaces Mapbox location search with two cascading dropdowns
- Province dropdown → Area dropdown (populated dynamically)
- On area selection: redirects to /?post_type=hp_listing&area={term_id}
- Keywords field hidden

**Taxonomy:** `hp_listing_area` (hierarchical — provinces are parent terms, areas are children)

**Troubleshooting:**
- If dropdowns do not appear, check the Mapbox location field selector `.hp-form__field--location` still matches
- If search returns no results, check the `area` URL parameter matches the term ID

---

### 3.17 Properties Page Fixes
**Type:** PHP  
**Hook:** `wp_footer` on `/properties/` and `?post_type=hp_listing`  
**What it does:**
- Location placeholder Location → Address (MutationObserver)
- Map open by default on desktop only (window.innerWidth >= 768)

---

### 3.18 Inclusive Pricing — Properties Page
**Type:** PHP  
**Hook:** `wp_footer` on `/properties/` and `?post_type=hp_listing`  
**What it does:** Multiplies all listing card nightly rates by 1.1 to show inclusive price (includes 10% service fee). Uses setInterval to catch dynamically loaded cards.

**DMCC Act 2024 compliance:** All displayed prices must include the mandatory 10% service fee.

---

### 3.19 Inclusive Pricing — Listing Page
**Type:** PHP  
**Hook:** `wp_footer` on single listing pages  
**What it does:**
- Multiplies displayed price by 1.1
- Shows includes 10% service fee note with (i) info button
- Modal explains service fee in full
- Uses setInterval to catch price updates when dates are selected

**Service fee modal content:**
- Fee covers: maintaining the platform, payment processing, host identity verification
- No fees charged to Cuban hosts
- Non-refundable if guest cancels; refunded if host cancels

---

### 3.20 Booking Details Page — Service Fee Display
**Type:** PHP  
**Hook:** `wp_footer` on `/make-booking/details/`  
**What it does:** Adds two rows below the Price field:
- Service Fee — 10% of accommodation price with (i) modal
- Total Price — accommodation plus service fee in bold

---

### 3.21 Service Fee Info Modal Content
The same modal content appears on the listing page and booking details page:

Stay Colibri charges a 10% service fee on all bookings. This fee covers maintaining the platform, payment processing, and host identity verification, ensuring a safe and secure booking experience for both guests and hosts.

Stay Colibri does not charge any fees to Cuban hosts — the service fee is paid by guests only.

The service fee is added to the accommodation cost and will be included in your total at checkout.

If you choose to cancel your booking, the service fee is non-refundable. If your host cancels, the service fee will be refunded in full.

---

### 3.22 Host Cancellation Flow
**Type:** PHP + JS  
**Hook:** `trashed_post` on `hp_booking`, `wp_footer` on `/account/bookings/`, `wp_ajax_sc_store_cancel_reason`

**Part 1 — Cancel modal (JS):**
Injects a reason dropdown and optional/mandatory text field into the HivePress cancel booking modal (`#booking_cancel_modal`) on the host bookings page. The dropdown has six options:
- Property unavailable — maintenance or damage
- Force majeure / emergency (flood, hurricane etc.)
- Double booking / calendar error
- Host illness or family emergency
- Guest behaviour concerns (prior to check-in)
- Other (please specify)

For options 1-5 the text field is optional. For "Other" it is mandatory. The form cannot be submitted without a reason selected. Before the HivePress DELETE request fires, the reason is stored via AJAX in a WordPress transient keyed to the booking ID (5 minute TTL).

**Part 2 — AJAX handler (PHP):**
Receives the cancellation reason via `wp_ajax_sc_store_cancel_reason`, validates the nonce, and stores it in a transient `sc_cancel_reason_{booking_id}`.

**Part 3 — trashed_post hook (PHP):**
Fires when HivePress trashes the booking post. Reads the stored reason from the transient. If no reason is found (e.g. admin-initiated cancellation) the hook exits silently. The Host Ledger (3.15) also reads this transient, at priority 5 before this hook, and records the reason on its CANCELLATION line.

Data gathered:
- Listing title from `$post->post_parent` (hp_listing)
- Host name and email from listing post_author
- Check-in / check-out from booking post meta `hp_start_time` / `hp_end_time`
- Order found via WooCommerce order item meta `hp_booking`
- Guest name and email from WooCommerce billing fields
- Order total, accommodation amount, platform fee (order total minus accommodation amount)
- Extras from order item meta `hp_price_extras`

**Admin email** sent to `jonny@auno.uk` containing:
- Cancellation reason
- Property name, check-in, check-out, extras
- Host name and email
- Guest name and email
- Refund breakdown: order total / host can refund / platform fee (admin must action)
- Link to WooCommerce order in wp-admin
- Link to front-end order page

**Guest email** sent to guest billing email (bilingual Spanish/English) containing:
- Booking reference, property, dates
- Full refund amount and 3 working day processing commitment
- Note that refund may arrive in two separate payments
- Note that refunds may take up to 14 business days depending on their bank and Stay Colibri cannot affect these timescales

**Important notes:**
- For ALL host cancellations the guest is entitled to a full refund including the platform fee
- The host can only refund the accommodation amount via the front-end order page
- Admin must manually refund the platform fee via the WooCommerce order in wp-admin
- Each refund appears as its own REFUND line in the Host Ledger
- HivePress may also send its own cancellation notification — this is in addition to it

**Troubleshooting:**
- If admin email is not received, check the transient was stored — add `error_log` to the AJAX handler to confirm
- If check-in/check-out show as Unknown, check `hp_start_time` and `hp_end_time` exist on the booking post in phpMyAdmin
- If the reason dropdown does not appear, check the snippet is firing on `/account/bookings/` URLs
- If the form submits without a reason, check the JS event listener is not being blocked by another script

---

### 3.23 Retired
V1.1 Cash Balance Admin Tool, V1.3/V1.4 Admin Balance Adjustments and V1.0 Cash Balance Reconcile are merged into **3.24 Admin Financial Tools** and must be deactivated.

---

### 3.24 V2.3 Admin Financial Tools
**Type:** PHP  
**Access:** Admin only (`manage_options`)  
**Requires:** 3.6 V3.5+, 3.15 V2.0.4+

#### Edit Host screen — "Stay Colibri — Financial Tools" box
Shown on the `hp_vendor` edit screen. The host's user ID is the vendor post's `post_author`.

**1. Overview:** payout method; ledger balance; payout system balance with ✓ reconciled / ✗ difference; ledger integrity (hash chain); XLSX last rendered; for cash hosts, whether the balance entries match the orders (changes proposed by Reconcile). Buttons: **View Full Ledger**, **Download XLSX**, and for cash hosts **Pending Cash Balance** and **Reconcile Cash Balance**.

**2. Pause Payouts:** checkbox for `sc_payouts_locked`. Toggling it refreshes ledger statuses immediately.

**3. Balance Adjustments — the only place to add or delete adjustments**
- Table of all adjustments: ID, date, amount, reason, added by, status, batch ref. Paid adjustments are locked.
- **No editing.** To correct one, delete it and add a new one; the ledger records both.
- **Delete** (pending only): ticking Delete reveals a **required reason** field. Without a reason the deletion is refused with an admin notice. The reason is recorded on the ledger's ADJUSTMENT REVERSAL line, and the host is emailed "Ajuste retirado / Adjustment withdrawn" (bilingual: reference, amount, original reason, original date, reason for withdrawal, updated balance). Deletions are matched by `adj_id`.
- **Add:** amount (positive = credit, negative = deduction) and reason (required, sent to the host). The hidden `sc_adj_id` (`ADJ-` + 8 hex) prevents duplicates when `save_post_hp_vendor` fires more than once. The host is emailed "Ajuste en tu cuenta / account adjustment" (bilingual, with updated balance).
- **Email balance figures** match the host dashboard: cash = entries eligible for the next batch (gross) + adjustments; Tropipay = all upcoming revenue net of fees + adjustments × 0.965.
- Admin notices confirm each add/withdrawal or explain a refusal.

#### Full ledger viewer — `/?sc_ledger_view={user_id}`
Live from SQL: summary (balances, reconciliation, hash chain, last render); buttons for Render XLSX now, Download XLSX, Pending Cash Balance, Reconcile, Edit Host; every ledger line with status, running balance and type colours; status history (newest first). **Order, booking and guest ID link to wp-admin** (new tab); deleted targets show "(deleted)"; guest checkouts show "Guest checkout".

#### XLSX download — `/?sc_ledger_download={user_id}`
Streams the host's file from S3 as of the last render. If none exists yet, it offers Render now.

#### Pending Cash Balance — `/?sc_cash_balance={user_id}`
The payout system's working balance (the permanent record is the ledger).
- **Booking balance entries:** order (linked), amount, checkout, eligible-from date, status ("Pending — next batch (date)", "Pending — later batch", "Paid — #ref"), note.
- **Delete** a pending entry: nonce-protected link with a confirmation prompt. It refuses if the entries changed since the page loaded. Paid entries are locked. **No bulk clear.** Each deletion becomes a ledger CORRECTION. Deleting an entry means the host is not paid it and the guest is not refunded — use a refund or an adjustment for business decisions. Reconcile will propose re-adding an entry whose order is still paid.
- **Adjustments:** read-only, with a link to Edit Host.
- **Totals:** pending entries (of which next batch / later), pending adjustments, grand total, **next batch balance** (matches the dashboard), ledger balance with reconciliation status.

#### Reconcile Cash Balance — `/?sc_cash_reconcile={user_id}`
Dry-run reconciliation of `sc_cash_balance_entries` against WooCommerce orders, with apply.
- Never touches paid entries, rollover entries (`order_id 0`) or adjustments.
- **Removes** pending entries whose order is deleted, trashed/cancelled/refunded/failed, belongs to another host, is duplicated, or also has a paid entry.
- **Adds** missing paid cash orders only if their eligibility date is on or after the last possible batch run. Earlier ones go to a manual review list with per-order "Add as pending" (confirm first that it was not paid in a batch).
- Amount, checkout, model and status mismatches are shown as warnings only.
- Apply is nonce-protected and refuses if the entries changed since preview. Every write first backs up to `sc_cash_balance_entries_backups` (last 10), with a one-click restore. Every write is recorded in the ledger as CORRECTION lines, and ledger statuses are refreshed.

**Troubleshooting:**
- Box missing: snippet inactive, or an old snippet still active causing a fatal error — check WPCode and the debug log.
- "No WordPress user linked to this vendor": the vendor post has no `post_author`.
- Viewer says ledger not active: run the backfill at `/?sc_ledger_admin=1`.

---

## 4. Third Party Integrations

### 4.1 Tropipay
**Purpose:** Automated payouts to hosts with Tropipay accounts  
**API Version:** v3  
**Sandbox:** https://sandbox.tropipay.me/api/v3  
**Live:** https://www.tropipay.com/api/v3  

**wp-config.php constants** (values are in the password manager — never in this file or the repository):

    define('SC_TROPIPAY_CLIENT_ID',     '[IN PASSWORD MANAGER]');
    define('SC_TROPIPAY_CLIENT_SECRET', '[IN PASSWORD MANAGER]');
    define('SC_TROPIPAY_SANDBOX',       false);   // true = sandbox
    define('SC_TROPIPAY_ACCOUNT_ID',    80001);   // live; sandbox account ID in password manager
    define('SC_TROPIPAY_TOTP_SECRET',   '[IN PASSWORD MANAGER]');

Sandbox client ID, client secret and account ID: password manager.

**Payout flow:**
1. Daily cron gets access token via POST /access/token (client_credentials)
2. Finds or creates beneficiary via GET /deposit_accounts + POST /deposit_accounts
3. Generates TOTP via `sc_generate_totp()` using `SC_TROPIPAY_TOTP_SECRET`
4. Executes payout via POST /operations/payout with `securityCode` = TOTP
5. Stores `sc_tropipay_deposit_account_id` in user meta to avoid recreating beneficiary

**TOTP:** Live credentials require a TOTP `securityCode` on every payout. The secret is stored in `SC_TROPIPAY_TOTP_SECRET` in wp-config.php (never in GitHub). `sc_generate_totp()` is defined in the Payout System snippet (3.6).

**Security note:** Live Tropipay credentials were rotated in October 2026 after accidental GitHub exposure. Credentials, client IDs and secrets must never be committed to the repository — including sandbox ones.

---

### 4.2 Didit Identity Verification
**Purpose:** Verifies host identity before they can list properties  
**Flow:**
1. Host registers → Zapier creates Didit verification session → emails host
2. Host completes Didit verification
3. Didit sends webhook to Zapier on approval
4. Zapier POSTs to /wp-json/custom/v1/verify-user
5. WordPress sets hp_verified = 1, stores name/carne/phone

**Troubleshooting:**
- Check Zapier Zap 3 is active
- Check the Didit webhook URL in Zapier is correct
- Manually set hp_verified = 1 in user meta if webhook fails

---

### 4.3 Brevo CRM
**Purpose:** Email marketing and CRM  
**Lists:**
- List 3: Pre-launch signups (ACCOUNT_TYPE = PreLaunch)

**Contact attributes set on registration:**
- Guest signup → ACCOUNT_TYPE = Guest
- Host signup → ACCOUNT_TYPE = Host
- Pre-launch signup → ACCOUNT_TYPE = PreLaunch

**Zapier integration:** Zap 1 (host signup) and Zap 2 (guest signup) update Brevo via webhook.

---

### 4.4 Frankfurter API
**Purpose:** Live EUR/USD exchange rate  
**URL:** https://api.frankfurter.app/latest?from=EUR&to=USD  
**Used in:**
- Section 8 cash batch — rate on batch day. On failure, 1.08 is used and the batch email is flagged **RATE FALLBACK**; check amounts before paying.
- Dashboard (3.7) — cached 6 h in `sc_dash_eur_usd_rate` (15 min after a failure), fallback 1.08.

---

### 4.5 UK Bank Holidays API
**Purpose:** Determines UK working days for payout scheduling  
**URL:** https://www.gov.uk/bank-holidays.json  
**Cached:** 24 hours via WordPress transient `sc_uk_bank_holidays`  
**Used in:** `sc_is_uk_working_day()`, `sc_add_uk_working_days()`, `sc_first_uk_working_day_of_month()`, dashboard, ledger statuses, Pending Cash Balance

---

## 5. Host Ledger and AWS S3

### Architecture
- **Source of truth:** the SQL ledger (`wp_14_sc_ledger`), written at the moment of each event. See 3.15.
- **S3 holds rendered XLSX views only**, re-rendered nightly at 03:00 UTC for changed hosts, or on demand. Losing or corrupting a file in S3 loses nothing — re-render it.
- Rendering no longer happens during checkout (V1 uploaded to S3 inside `woocommerce_payment_complete`).

### Credentials (wp-config.php)

    define('SC_AWS_ACCESS_KEY_ID',     '[IN PASSWORD MANAGER]');
    define('SC_AWS_SECRET_ACCESS_KEY', '[IN PASSWORD MANAGER]');
    define('SC_AWS_BUCKET',            'staycolibri-ledgers');
    define('SC_AWS_REGION',            'eu-west-2');

### IAM User
- Username: staycolibri-ledger-writer
- Policy: AmazonS3FullAccess
- Bucket: staycolibri-ledgers (private, all public access blocked)
- **Versioning:** should be enabled on the bucket so every rendered version is kept (see go-live checklist)

### File Format
- One XLSX file per host: `ledgers/{user_id}_{slug}.xlsx`
- Generated using PhpSpreadsheet
- Two sheets: Ledger, Status history (layout in 3.15)

### Manual Render

    https://staycolibri.com/?sc_rebuild_ledger=1&host_id=13
    https://staycolibri.com/?sc_rebuild_ledger=1&host_id=all

Must be logged in as admin. Also available as **Render now** on the ledger admin page and the ledger viewer.

---

## 6. Cron Jobs

### Scheduled events
| Hook | Schedule | Handlers (priority) |
|---|---|---|
| `sc_daily_payout_cron` | Daily 06:00 UTC (`sc_daily`) | 3.6 Section 6 — create Tropipay payout records (10); 3.6 Section 7 — Tropipay transfers (10); 3.15 — record payouts in ledger + status sweep (40) |
| `sc_monthly_payout_cron` | Daily 06:00 UTC (`sc_daily`) | 3.6 Section 8 — cash batch, runs only once per month on/after the first UK working day (10); 3.15 — record batch/no-delivery lines + status sweep (20) |
| `sc_ledger_nightly` | Daily 03:00 UTC (`daily`) | 3.15 — payout scan, status sweep, reconciliation email, render changed XLSX |

### Checking Cron Status
Install WP Crontrol to view scheduled events. `sc_monthly_payout_cron` must show **Once Daily**. The `sc_monthly` (30-day) schedule is legacy and unused.

### Note on WP-Cron
WP-Cron only runs when the site receives a request. Section 8's once-per-month claim and catch-up make a missed batch day safe for cash. Section 6 is not safe (see section 10). Consider triggering `wp-cron.php` from the server's crontab for reliable timing.

---

## 7. Zapier Flows

### Zap 1 — Host Registration
**Trigger:** WordPress user register webhook (URL stored in password manager — Zapier catch hook)  
**Filter:** hp_acctype = 108  
**Actions:**
1. Create Didit verification session
2. Send verification email to host
3. Add/update Brevo contact with ACCOUNT_TYPE = Host

### Zap 2 — Guest Registration
**Trigger:** Same WordPress webhook  
**Filter:** hp_acctype = 107  
**Action:** Add/update Brevo contact with ACCOUNT_TYPE = Guest

### Zap 3 — Didit Approval
**Trigger:** Didit webhook on approval  
**Action:** POST to /wp-json/custom/v1/verify-user

### WordPress Webhook Snippet
Fires 10 seconds after user registration. Sends to Zapier:
- email — user email
- acctype — HivePress account type (107/108)
- meta — all WordPress user meta

---

## 8. Go-Live Checklist

### Done
- [x] Tropipay API — 2FA resolved via server-side TOTP
- [x] Live Tropipay credentials in wp-config.php (rotated October 2026)
- [x] Booking cancellations — host cancellation flow (3.22), refunds and cancellations recorded in the Host Ledger
- [x] Append-only Host Ledger with backfill (3.15)

### Must do before launch
- [ ] Didit verification — live testing with real Didit account
- [ ] **Fix Section 6 checkout window** (section 10) — first check the ledger for bookings showing "Eligible — awaiting payout · Tropipay payout record not yet created"
- [ ] **Clean start for the ledger:** remove test orders, payouts, cash balance entries and adjustments → `/?sc_ledger_admin=1` → **Reset ledger** → **Run one-time backfill** → confirm every host reconciles → **Mark ledger live**
- [ ] Enable **Versioning** on S3 bucket `staycolibri-ledgers`
- [ ] Check any order partially refunded before V3.5 (its refund split may be wrong — section 10)
- [ ] Deactivate coming soon snippet — removes redirect to /coming-soon/
- [ ] Deactivate contact page nav restriction snippet — restores full nav on /contact/
- [ ] Payment setup page — add Tropipay confirmation checkbox

### Recommended
- [ ] Server crontab for `wp-cron.php` (reliable cron timing)
- [ ] Migrate credentials to AWS Secrets Manager
- [ ] WordPress admin 2FA; IP restriction on wp-admin via Cloudflare
- [ ] Add Max Guests attribute to listings

### wp-config.php for live

    define('SC_TROPIPAY_CLIENT_ID',     '[IN PASSWORD MANAGER]');
    define('SC_TROPIPAY_CLIENT_SECRET', '[IN PASSWORD MANAGER]');
    define('SC_TROPIPAY_SANDBOX',       false);
    define('SC_TROPIPAY_ACCOUNT_ID',    80001);
    define('SC_TROPIPAY_TOTP_SECRET',   '[IN PASSWORD MANAGER]');

---

## 9. Troubleshooting

### Site Gives Critical Error
1. Enable debug logging in wp-config.php:

    define('WP_DEBUG',         true);
    define('WP_DEBUG_LOG',     true);
    define('WP_DEBUG_DISPLAY', false);

2. Check log: tail -100 /opt/bitnami/wordpress/wp-content/debug.log
3. "Cannot redeclare function sc_…": a retired snippet is still active alongside its replacement (see 3.23 / snippet inventory)
4. Disable snippets one by one in WPCode until error stops

### Host Dashboard Crashes
Most likely cause: a WPCode snippet calling a function defined in another snippet that has not loaded yet.
- Check for Call to undefined function in debug log
- Wrap the call in function_exists() check
- Adjust snippet load priority in WPCode

### Tropipay Payouts Not Processing
1. Check SC_TROPIPAY_SANDBOX value in wp-config.php
2. Check payout records exist: WordPress Admin → HivePress → Payouts
3. Check sc_payment_date <= today's timestamp in postmeta
4. Check host has sc_tropipay_email set in user meta and is not paused
5. Check sc_tropipay_deposit_account_id — if empty, beneficiary creation may be failing
6. No payout record at all for a checked-out booking: see the Section 6 issue in section 10

### Cash Batch Missing or Wrong
1. WP Crontrol: `sc_monthly_payout_cron` must be Once Daily
2. `sc_cash_batch_last_run` in `wp_14_options` — current month = already run
3. Batch email flagged RATE FALLBACK — the API failed and 1.08 was used; check amounts before paying
4. Compare with the dashboard modal and `/?sc_cash_balance={id}` — all use `sc_cash_batch_calc()` and the same eligibility rule

### Ledger Does Not Reconcile
1. `/?sc_ledger_admin=1` shows the host and difference; the nightly email lists the same
2. Open `/?sc_ledger_view={id}` and check the latest lines against recent admin actions
3. Changes made directly in the database (phpMyAdmin) are not seen by the ledger — avoid; if unavoidable, add a matching adjustment
4. Pre-launch: Reset + backfill

### Ledger Not Updating (XLSX)
1. Lines are written live; the XLSX renders at 03:00 UTC or via Render now
2. Check AWS credentials in wp-config.php
3. Check PhpSpreadsheet: ls /opt/bitnami/wordpress/vendor/phpoffice/phpspreadsheet
4. Check S3 bucket permissions in AWS IAM console

### Prices Not Showing Inclusive of Service Fee
- Properties page: check inclusive pricing properties snippet is active
- Listing page: check inclusive pricing listing snippet is active
- Booking details: check booking details fee snippet is active

### Coming Soon Redirect Not Working
- Check the coming soon snippet is active in WPCode
- Check the allowed slugs array includes any new pages that should be publicly accessible

### WP_DEBUG Already Defined Warning
wp-config.php has WP_DEBUG defined twice. Remove the duplicate definition. The warning is harmless but clutters the error log.

### Brute Force Login Attempts
The server has been receiving brute force attempts targeting WordPress login. Recommended actions:
- Install Limit Login Attempts Reloaded plugin
- Add Cloudflare firewall rules to block repeated login failures
- Enable two-factor authentication on admin accounts

---

## 10. Known Issues and Open Items

| Issue | Impact | Status |
|---|---|---|
| **Section 6 checkout window** — Tropipay payout records are only created for orders whose `_sc_checkout_date` is exactly midnight today or tomorrow | If the daily cron misses that window (no site traffic, outage), the booking **never gets a payout record** and the host is not paid. The ledger shows it as "Eligible — awaiting payout · Tropipay payout record not yet created" | Fix planned: match `≤ tomorrow`. Check for stale unprocessed test orders first |
| **`wc_get_orders()` ignores `meta_query`** on this site's legacy order storage | Returns every order. Caused the V3.3–V3.8 dashboard Tropipay bug | Fixed in 3.7/3.15/3.24; always use `sc_ledger_find_orders()` for meta lookups |
| **Orders partially refunded before V3.5** | 5b's refund split used the already-reduced amount; a second partial refund over-deducted the host; Tropipay partial refunds before payout creation were ignored | Fixed in V3.5; check historical partially-refunded orders manually |
| **One booking per order** assumed | 3.6 Section 1 and the ledger read the first booking item only | Holds for current checkout flow |
| **Pre-V3.4 cash batches** | `sc_fee_amount` holds the old whole-balance fee; no `batch_ref`/`payout_ref` | Ledger derives the fee from `hp_amount`; statuses show "pre-V3.4 batch" |
| **Direct database edits** | Not seen by the ledger watchers; reconciliation will flag them | Use the admin tools |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | September 2026 | Initial documentation |
| 1.1 | October 2026 | Added host cancellation flow (snippet 3.22), updated go-live checklist, added security risks section |
| 1.2 | October 2026 | V3.3 Payout Cron: refund hook (5b), payout lock (5c), adjustment handling in sections 7 and 8, TOTP live. V3.1 Dashboard Balance Block: payout lock notice, adjustment total display. Added snippets 3.23 (V1.1 Cash Balance Admin Tool) and 3.24 (V1.3 Admin Balance Adjustments). Updated Tropipay section: live credentials, TOTP, credential rotation note |
| 1.3 | October 2026 | **Payout System V3.5:** `sc_cash_batch_calc()` (fee on delivered amount, corrected delivery cap, cent-rounding guard); monthly hook now daily with once-per-month claim and catch-up; `sc_first_uk_working_day_of_month()` honours its argument; Section 9 re-syncs only the linked rollover; `batch_ref`/`payout_ref`; base amount for Tropipay adjustments; rate-fallback flag; 5b original-amount and pre-payout Tropipay refund fixes; `adj_id` on system adjustments. **Dashboard V3.9:** cash shows estimated USD delivery with breakdown modal; Tropipay shows total upcoming revenue; order-lookup fix. **Host Ledger V2.0.4:** append-only SQL ledger, status tracking with history, hash chain, triggers, nightly reconciliation and render, backfill, admin page, hyperlinks. **Admin Financial Tools V2.3:** merges V1.4 Financial Tools, V1.0 Reconcile and V1.1 Cash Balance; overview with reconciliation; ledger viewer and XLSX download; adjustments add/delete only on Edit Host with required deletion reason and host withdrawal email; Pending Cash Balance with protected deletes and no bulk clear. Credentials removed from this file. Added section 10 |

Documentation last updated October 2026. Update this file whenever snippets are added, modified or removed.

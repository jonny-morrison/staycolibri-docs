# Stay Colibri — Site Documentation
**Last Updated:** October 2026 (v1.19)  
**Platform:** WordPress Multisite (HivePress/RentalHive)  
**URL:** https://staycolibri.com  
**Subsite prefix:** `wp_14_`  
**Server:** Bitnami AWS Lightsail (Ubuntu), London region

> **Security:** this file is kept in GitHub. Never put credentials, secrets, client IDs or webhook URLs in it — write `[IN PASSWORD MANAGER]` instead.

---

## Table of Contents
1. [System Architecture](#1-system-architecture)
2. [WordPress Data Structure](#2-wordpress-data-structure)
3. [WPCode Snippets](#3-wpcode-snippets)
4. [Third Party Integrations](#4-third-party-integrations)
5. [Host Ledger and AWS S3](#5-host-ledger-and-aws-s3)
6. [Cron Jobs](#6-cron-jobs)
7. [Zapier Flows (retired)](#7-zapier-flows-retired)
8. [Go-Live Checklist](#8-go-live-checklist)
9. [Troubleshooting](#9-troubleshooting)
10. [Known Issues and Open Items](#10-known-issues-and-open-items)

---

## 1. System Architecture

### Stack
- **WordPress Multisite** — main network at `2uw.uk`, subsite at `staycolibri.com` (blog ID 14)
- **HivePress** — listing/booking framework
- **RentalHive** — HivePress extension for rental bookings
- **WooCommerce** — payment processing (legacy post-based order storage; HPOS **not** enabled). Order Attribution is **on**, gated behind Analytics consent (section 4.8)
- **WPCode** — all custom PHP/JS snippets
- **Cloudflare** — DNS, proxy and WAF (free plan; section 4.6)
- **Cloudflare Turnstile** — human check on wp-login, password reset and contact forms (Simple CAPTCHA Alternative with Cloudflare Turnstile plugin) and on HivePress's login, register and password forms (3.34)
- **Pressidium Cookie Consent** — cookie banner, consent records and script blocking (section 4.8)
- **WP Mail SMTP** — outgoing email via **Amazon SES** (`eu-west-1`, Ireland; check Settings → Mailer for SMTP vs API)
- **Next3Cloud** (Next3 Offload) — media files offloaded to Peasoup Cloud S3. Its image-size URL filter is broken; 3.33 corrects it (section 4.7)
- **Backups** — at server level (not by a WordPress plugin)
- **PhpSpreadsheet** — installed at `/opt/bitnami/wordpress/vendor/autoload.php`
- **Didit** — host identity verification (direct API + signed webhook; no Zapier)
- **Brevo** — CRM contacts for guests and hosts (direct API; no Zapier)
- **Email design** — one shared HTML layout, snippet 3.31, for emails to hosts and guests. Admin emails stay plain text

### Money flow (summary)
- Guest pays accommodation + service fee (HivePress buyer commission, currently 10% — HivePress → Settings → Users → Commission) via WooCommerce. All prices shown to guests include the fee (DMCC Act 2024, no drip pricing).
- **Cash hosts:** each paid booking adds an entry to `sc_cash_balance_entries`. On the first UK working day of each month, eligible entries (checkout + 5 UK working days before the batch date) plus pending adjustments are paid as USD cash in $50 multiples. The remittance fee (11% + €3.00) is charged on the delivered amount only; the remainder rolls over gross. Cash is delivered by **Transpaygo Limited** (company 08936601; FCA-authorised payment institution, FRN 722528; HMRC MSB 12579512), which receives the host's name, delivery address and amount. The public privacy policy describes it only as "an FCA-authorised money transfer company".
- **Tropipay hosts:** a payout record is created at checkout; the transfer runs automatically 5 UK working days after checkout, net of 3.5% + €0.50, with any pending adjustments applied.
- **Host Ledger:** an append-only SQL ledger records every financial event. It is reconciled nightly against the payout system and rendered to XLSX in S3.

### Host onboarding flow (summary)
1. Host registers → HivePress sends the email-confirmation email (HivePress's own "Registration Complete" email is suppressed for hosts — 3.27).
2. Host confirms their email → 3.27 creates a Didit session (`vendor_data` = WordPress user ID) and emails the Didit link (Spanish, designed with 3.31); 3.28 creates the Brevo contact.
3. Host completes Didit → Didit redirects the browser to the home page; 3.25 sends an unverified host to `/verification-pending/`.
4. Didit's signed webhook reaches 3.26 → on Approved: `hp_verified = 1`, names and carné stored, Brevo updated (3.28). Approval after a manual review also emails the host (Spanish). Declined / In Review / Abandoned / Expired email the admin.

### Key URLs — public
| Page | URL |
|---|---|
| Home | https://staycolibri.com |
| Properties | https://staycolibri.com/properties/ |
| Coming Soon | https://staycolibri.com/coming-soon/ |
| Why Colibri (hosts) | https://staycolibri.com/whycolibri/ |
| Contact | https://staycolibri.com/contact/ |
| Privacy and cookie policy | https://staycolibri.com/privacy/ (cookies section anchor `#cookies`) |
| Payment Setup | https://staycolibri.com/payment-setup/ |
| Host Dashboard | https://staycolibri.com/account/vendor/dashboard/ |
| Booking Details | https://staycolibri.com/make-booking/details/ |
| Verification Pending | https://staycolibri.com/verification-pending/ |
| Submit Listing | https://staycolibri.com/submit-listing/ |

### Key URLs — machine endpoints
| Endpoint | Purpose | Snippet |
|---|---|---|
| `POST /wp-json/custom/v1/didit-webhook` | Didit `status.updated` webhooks (HMAC-signed) | 3.26 |
| `POST /wp-json/hivepress/v1/users/login` | HivePress login (modal) — Turnstile token required | HivePress; 3.34 |
| `POST /wp-json/hivepress/v1/users` | HivePress registration — Turnstile token required | HivePress; 3.34 |
| `POST /wp-json/hivepress/v1/users/request-password` | HivePress forgot password — Turnstile token required | HivePress; 3.34 |
| `POST /wp-json/hivepress/v1/users/reset-password` | HivePress password reset — Turnstile token required | HivePress; 3.34 |

### Key URLs — admin tools (must be logged in as admin)
| Tool | URL | Snippet |
|---|---|---|
| Ledger admin (backfill, render, reconcile status) | `/?sc_ledger_admin=1` | 3.15 |
| Full ledger for a host (live from SQL) | `/?sc_ledger_view={user_id}` | 3.24 |
| Download host XLSX (as of last render) | `/?sc_ledger_download={user_id}` | 3.24 |
| Render XLSX now | `/?sc_rebuild_ledger=1&host_id={user_id}` or `host_id=all` | 3.15 |
| Pending cash balance | `/?sc_cash_balance={user_id}` | 3.24 |
| Reconcile cash balance | `/?sc_cash_reconcile={user_id}` | 3.24 |
| Verification / CRM event log (Didit webhooks, link emails, Brevo syncs) | `/?sc_didit_log=1` | 3.26 |

The financial tools are also linked from the **Stay Colibri — Financial Tools** box on the Edit Host screen. Verification and Brevo tools are on the **Edit User** screen (boxes "Stay Colibri — Didit verification" and "Stay Colibri — Brevo").

### Admin Access
- WordPress Admin: https://staycolibri.com/wp-admin
- phpMyAdmin: via Bitnami server local access
- WPCode Snippets: WordPress Admin → Code Snippets
- Cookie consent: WordPress Admin → Settings → Cookie Consent (section 4.8)
- Host financial tools: WordPress Admin → Hosts → edit host → **Stay Colibri — Financial Tools** box
- Host verification and Brevo: WordPress Admin → Users → edit user → **Stay Colibri — Didit verification** / **Stay Colibri — Brevo** boxes
- Didit: https://business.didit.me (separate **sandbox** and **live** applications)
- Cloudflare: https://dash.cloudflare.com (zone `staycolibri.com`, free plan)

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
| `hp_verified` | `1` = verified | Didit verification status (set by 3.26 on Approved) |
| `hp_phone` | string | Phone number from the sign-up form, including country code. Not touched by Didit |
| `hp_carne` | string | Cuban ID number from Didit (3.26): `personal_number`, else `document_number` — confirm on the first real host (section 10) |
| `first_name` | string | Hosts: from Didit on approval, title case (3.26 V1.0.4). Locked for hosts (3.10) |
| `last_name` | string | Hosts: from Didit on approval, title case; both Cuban surnames together. Locked for hosts (3.10) |
| `sc_didit_first_name_raw` / `sc_didit_last_name_raw` | string | Names exactly as Didit read them off the document (admin reference, e.g. cash delivery) |
| `sc_payout_details_log` | serialised array | Last 20 payout-detail changes (3.4 V3.1): time, field, old, new, by, where, IP |
| `hp_email_verify_key` | string | HivePress: present until the user confirms their email |
| `sc_didit_session_id` | UUID | Host's current Didit session. Webhook events for any other session are ignored (3.26). Written by 3.27 |
| `sc_didit_session_created` | Unix timestamp | When the current session was created |
| `sc_didit_status` | Didit status string | Last status received (`Not Started`, `In Progress`, `In Review`, `Approved`, `Declined`, `Resubmitted`, `Abandoned`, `Expired`, `Kyc Expired`) |
| `sc_didit_status_at` | Unix timestamp | When that status was received |
| `sc_didit_verified_at` | Unix timestamp | When the host was approved |
| `sc_didit_approval_emailed` | session UUID | Session for which the post-review approval email was sent (prevents duplicates) |
| `sc_didit_link_sent_at` | Unix timestamp | Last time the Didit link was emailed |
| `sc_didit_link_sent_count` | integer | Times the link has been emailed |
| `sc_didit_last_error` | string | Last session-creation / link-email error (cleared on success) |
| `sc_brevo_synced_at` | Unix timestamp | Last successful Brevo sync (3.28) |
| `sc_brevo_last_error` | string | Last Brevo error, or a note such as "synced without: SMS (Invalid phone number)" |
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

The Didit verification link is **never** stored in WordPress or logged — it acts as a credential.

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
| `_wc_order_attribution_*` | WooCommerce Order Attribution (traffic source, landing page, pages, user agent). Only populated when the guest accepted Analytics cookies (section 4.8); otherwise the order's Origin shows "Unknown" |

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
| `sc_didit_webhook_log` | Last 100 verification/CRM events, newest first (3.26 log; 3.27 and 3.28 also write to it). No names, ID numbers, phone numbers or links. Not autoloaded |
| `cfturnstile_key` / `cfturnstile_secret` | Simple Cloudflare Turnstile plugin's site key and secret. 3.34 falls back to these only if the wp-config constants are not defined |
| `pressidium_cookie_consent_settings` | Pressidium Cookie Consent settings (section 4.8). Change through the plugin screen or Import Settings, not in the database |

### Transients
| Transient | Lifetime | Description |
|---|---|---|
| `sc_uk_bank_holidays` | 24 h | UK bank holidays from gov.uk |
| `sc_dash_eur_usd_rate` | 6 h (15 min on API failure) | EUR/USD rate for the host dashboard estimate |
| `sc_cancel_reason_{booking_id}` | 5 min | Host cancellation reason (3.22), read by 3.22 and the ledger |
| `sc_didit_evt_{md5(event_id)}` | 14 days | Didit event already processed (3.26) — Didit reuses `event_id` on retries |

### Image sizes used by the theme
Registered by WordPress, HivePress, HiveTheme and WooCommerce; generated on upload and offloaded to S3 by Next3Cloud (file name pattern `{original-name}-{w}x{h}.{ext}`, in the same folder as the original).
| Size | Dimensions | Used for |
|---|---|---|
| `hp_landscape_large` | 800 × 600, cropped | Listing page gallery slides (RentalHive `listing-images.php`) |
| `large` | 1024 max, uncropped | Listing gallery zoom (`data-zoom`) |
| `hp_landscape_small` | 400 × 300, cropped | Listing cards (properties page, favourites, host profile) |
| `hp_square_small` | 400 × 400, cropped | HivePress square thumbnails |
| `ht_landscape_small` / `ht_cover_large` | 400 × 300 / 1600 × 800 | HiveTheme |

Since the crops are 4:3, portrait and landscape photos appear at the same height in the gallery and cards **only if the size URLs resolve correctly** — see 3.33.

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

### Snippet inventory
All are PHP snippets set to **Auto Insert → Run Everywhere** (a header/footer location loads too late for hooks such as `template_redirect` and `rest_api_init` and silently does nothing). Each has its version in its header comment; increment it on every change. **WPCode title format:** `3.xx — Name` (version in the code header only). Numbers of retired snippets are not reused.

| No. | Snippet | Version | Notes |
|---|---|---|---|
| 3.0 | Config | V1.0 | |
| 3.2 | Submit Listing Access Control | V1.0 | Stays active after launch |
| 3.3 | Listing Page Map Notice | V1.0 | |
| 3.4 | Payment Setup Page | **V3.2** | Host change email on the 3.31 design |
| 3.5 | Gateway Routing | V3 | |
| 3.6 | Payout System (cron) | **V3.6** | Section 6 Tropipay email on 3.31; new 6b cash checkout email and 10 cash delivery confirmation (Spanish) |
| 3.7 | Dashboard Balance Block | V3.9 | WPCode title said v3.8 — code is V3.9 |
| 3.8 | Coming Soon System | **V1.1** | Deactivate at launch |
| 3.9 | Why Colibri Page | V1.0 | |
| 3.10 | Account Settings Page Modifications | **V1.1** | Server-side field lock |
| 3.11 | Register Modal Warning (hosts only) | V1.0 | Deactivate at launch |
| 3.12 | Contact Form Styling | V1.0 | |
| 3.13 | Contact Page Nav Restriction | V1.0 | Deactivate at launch |
| 3.14 | Payout Cancel Button Hidden | V1.0 | |
| 3.15 | Host Ledger | V2.0.4 | |
| 3.16 | Home Page Location Dropdowns | V1.0 | |
| 3.17 | Properties Page Fixes | V1.0 | |
| 3.18 | Guest Pricing Display | V3.2 | |
| 3.22 | Host Cancellation Flow | **V1.2** | Hosts only; guest email on the 3.31 design (ES + EN) |
| 3.24 | Admin Financial Tools | **V2.4** | Host adjustment emails on the 3.31 design (Spanish) |
| 3.25 | Unverified Host Lockdown | **V1.0.3** | Deactivate at launch |
| 3.26 | Didit Verification Webhook | **V1.0.5** | Host approval email on the 3.31 design |
| 3.27 | Didit Verification Session + Email | **V1.8** | Email built with 3.31; carné only; logo now held by 3.31 |
| 3.28 | Brevo Contact Sync | V1.3 | |
| 3.29 | Listing Photo Limit | **V1.1** | 30 photos per listing |
| 3.30 | Show User ID Shortcode | V1.0 | `[current_user_id]` |
| 3.31 | Email Layout | **V2.6** | Shared design for host and guest emails (Zunzún palette); hero text may be empty |
| 3.32 | HivePress Email Wrapper | **V1.3** | Puts HivePress's own emails in the 3.31 design (English); suppresses `booking_cancel_vendor`, `order_receive`, `order_complete` to hosts and `payout_complete` to cash hosts |
| 3.33 | Image Size URL Fix (Next3Cloud) | **V1.0** | Restores image size URLs; Next3Cloud returns the original for every size. Keep active while Next3Cloud is in use |
| 3.34 | Turnstile on HivePress Account Forms | **V1.1** | Server-side Turnstile check on HivePress login/register/password REST endpoints + widget in the modals. Stays active after launch |
| 3.35 | Mapbox Storage Block | **V1.0.1** | Stops Mapbox GL JS storing `mapbox.*` identifiers in browser storage (PECR). Maps, address search and map-load billing unaffected. Stays active after launch |
| 3.36 | Cookie Settings Menu Link | **V1.0** | Menu items with URL `#cookie-settings` open the cookie settings modal. Stays active after launch |

**Retired — must be deactivated** (function names clash, prices would be multiplied twice, or data would be sent to a retired service): V1 Host Ledger, V1.4 Admin Financial Tools (formerly V1.3 Admin Balance Adjustments), V1.0 Cash Balance Reconcile, V1.1 Cash Balance Admin Tool, Inclusive Pricing — Properties Page, Inclusive Pricing — Listing Page, Booking Details Page — Service Fee Display, Guest Pricing Display V2.0, **3.1 Didit Verification Endpoint** (trashed October 2026), **WordPress → Zapier registration webhook** (section 7), **`/wp-json/custom/v1/get-user-by-email`** helper endpoint (deleted October 2026 — Zapier-era, admin-only), **TEMP — Image size diagnostic** (`/?sc_img_diag=`, used once in October 2026 to diagnose the Next3Cloud bug; deleted), **TEMP — customer-post-data trap** (a `document.cookie` watcher, if it was created during the October 2026 cookie audit; delete it).

**Dependencies:** 3.7, 3.15 and 3.24 call functions defined in 3.6. 3.24 calls functions defined in 3.15. 3.18 calls functions defined in 3.0. 3.27 and 3.28 use the log and helpers in 3.26, and 3.28 listens for 3.26's `sc_didit_host_approved` action. 3.27 builds its email with the blocks in 3.31 (a plain fallback email is sent if 3.31 is inactive), and 3.31's `sc_email_send()` embeds the logo using `sc_didit27_logo_png()` from 3.27. 3.34 reads the Turnstile keys from wp-config, falling back to the Simple Cloudflare Turnstile plugin's settings. 3.36 needs the Pressidium Cookie Consent plugin. All cross-snippet calls are guarded with `function_exists()`. **Deploy 3.31 before 3.27.**

---

### 3.0 V1.0 Stay Colibri — Config
**Type:** PHP — Run Everywhere  
**Purpose:** single source of truth for the guest service fee. The rate is **not** stored here — it is read from the HivePress buyer commission:
- `hp_user_commission_rate` — percent (currently `10`)
- `hp_user_commission_fee` — fixed amount per booking (currently empty)

Set in **HivePress → Settings → Users → Commission**. HivePress Marketplace uses the same options to charge the fee at checkout (`get_cart_fee()`: (cart total − tax − security deposit) × rate / 100 + fixed fee), so displayed and charged figures cannot drift. If the rate option is blank, the functions fall back to 10%.

**PHP functions:** `sc_service_fee_percent()` (10), `sc_service_fee_rate()` (0.10), `sc_service_fee_fixed()` (0), `sc_service_fee($acc)`, `sc_guest_price($acc)`, `sc_service_fee_label()` ("10%", or "10% + €2.00" if a fixed fee is set).

**Browser:** `window.SC_CONFIG` printed at the top of `<head>` (`wp_head` priority 1): `feeRate`, `feePercent`, `feeFixed`, `feeLabel`, `multiplier`, `serviceFee(a)`, `guestPrice(a)`.

**Shortcode:** `[sc_service_fee]` → "10%". **Use it in page content wherever the fee percentage is stated** (FAQs, terms, Why Colibri) so the text follows the setting. It does not work in page titles, menu labels, SEO meta, emails or images — those must be edited by hand if the rate changes.

**Changing the rate:** change it in HivePress only. New bookings, displayed prices, the modal and page text follow automatically. Existing orders are unaffected — payouts, refunds and the ledger use each order's actual totals, never the rate.

---

### 3.1 Retired — Didit Verification Endpoint
`POST /wp-json/custom/v1/verify-user` received Didit approvals relayed by Zapier Zap 3, matched hosts by email and was protected only by a shared `X-Secret-Key` header. **Replaced by 3.26** (direct, signed Didit webhook) and **trashed in October 2026**. The WordPress application password used by Zap 3 was deleted at the same time.

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

### 3.4 V3.1 Payment Setup Page
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

**Admin save (V3.1):** the admin fields are shown and saved only for hosts (108); saving another user's profile no longer writes empty payout meta.

**Payout-detail change log (V3.1):** any change to `sc_payment_model`, `sc_tropipay_email` or `sc_cash_address` — host form, admin profile or any code (caught on `add/update_user_metadata` and `delete_user_meta`) — is:
- recorded in user meta `sc_payout_details_log` (last 20: time UTC, field, old, new, by whom, where, IP) and shown under **Payout detail changes** on the Edit User screen;
- emailed to the admin with old and new values ("Stay Colibri — payout details changed (host #…)");
- emailed to the host in Spanish, naming the changed details but **not** the values ("Se han actualizado tus datos de pago").
Emails go out once per request (`shutdown`). Unchanged saves send nothing.

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

### 3.6 V3.6 Payout System (MAIN CRON SNIPPET)
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
- To trigger manually: temporary snippet with `do_action('sc_daily_payout_cron')` or `do_action('sc_monthly_payout_cron')`. (Not `wp cron event run` — WP-CLI crashes while Next3Cloud is active; section 10.)

---

**Section 6b — Cash hosts: checkout-day email (V3.6, Spanish)**  
Daily (`sc_daily_payout_cron`, priority 20). Cash orders (processing/completed) whose `_sc_checkout_date` is today or up to 3 days ago and that have no `_sc_cash_checkout_emailed` get one email: the booking's EUR amount (the host's pending balance entry for that order, so partial refunds are reflected) and the monthly batch ("lote de pagos") it belongs to, worded as the date the batch is processed, with the cash arriving a few days later, from `sc_cash_batch_date_for_checkout()`: the first batch date strictly after checkout + 5 UK working days (Section 8's rule). Includes booking number, listing, checkout date, a button to the dashboard, and short notes on $50 multiples, the exchange rate and the 11% + €3,00 fee. If payouts are paused, the email says so and gives no delivery date. The order is flagged before sending, so it never sends twice; no pending entry (refunded or removed) = no email. Needs 3.31 V2.5+ (otherwise nothing is sent).

**Section 10 — Cash payout marked Completed: delivery confirmation (V3.6, Spanish)**  
When a `USD Cash Delivery` payout (with `sc_calc_version`) changes to Completed (`publish`) in a request, `save_post` priority 1000 (after Section 9 and after the admin's edited `hp_amount` is stored) emails the host once (`sc_host_completion_emailed`): cash sent in USD, amount deducted from the balance, and the final calculation with the dashboard modal's lines: Saldo trasladado and Ingresos de este mes (split from the entries with `batch_ref` `#ID`; one combined line if they do not add up), Ajustes, Saldo mensual, Entrega en efectivo, Equivalente en EUR (tasa), Comisión de remesa, a "Corrección al confirmar el pago" line if `hp_amount` differs from `sc_deduction_eur`, and Trasladado al próximo mes = `sc_total_eur − hp_amount` (Section 9's formula). Also reference, confirmation date, name and delivery address. Months with no delivery (under $50 or negative) create no payout, so no email. Needs 3.31 V2.5+.

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

### 3.8 V1.1 Coming Soon System
**DEACTIVATE AT LAUNCH**  
**Type:** PHP  
**What it does:**
- template_redirect: Redirects all non-logged-in visitors to `/coming-soon/` except:
  - WordPress pages `coming-soon`, `whycolibri`, `contact` (`is_page()`)
  - HivePress account routes, matched by URL path because they are not WordPress pages: `/account/verify-email/` (email confirmation link), `/account/reset-password/` (password reset link), `/account/login/` (V1.1)
  - REST and AJAX requests
- Shortcode [sc_coming_soon]: Standalone coming soon page with email signup (Brevo list 3, `ACCOUNT_TYPE` = 3 PreLaunch) and host CTA
- Registration locked to Host (108) only (front-end JS)
- `/whycolibri/` gets Sign Up as a Host CTA

**Brevo key:** read from `SC_BREVO_API_KEY` in wp-config.php (V1.1; shared with 3.28). V1.0 had the key hard-coded and sent `ACCOUNT_TYPE` as the text "PreLaunch", which Brevo ignored — pre-launch contacts created before V1.1 may have no account type.

**Privacy page:** `/privacy/` is not on the allowlist, so logged-out visitors are redirected away from it while 3.8 is active. Add `privacy` to the allowed pages if the policy must be reachable before launch (the cookie banner links to it).

**Troubleshooting:**
- If logged-in users are being redirected, check they are actually logged in
- Email confirmation or password reset link lands on `/coming-soon/`: the HivePress route path changed — update the path list in section 4 of the snippet
- If the email signup is not working, check `SC_BREVO_API_KEY` in wp-config.php

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

### 3.10 V1.1 Account Settings Page Modifications
**Type:** PHP  
**What it does (settings page, browser):**
- Guests: First/last name mandatory, carne field hidden
- Hosts: First/last/carne shown read-only with note to contact info@staycolibri.com
- Account type dropdown replaced with plain text (hidden input preserves value)

**Server-side lock (V1.1):**
- During the save request (REST), the locked fields are removed from HivePress's `user_update` / `user_update_profile` forms (`hivepress/v1/forms/…`, priority 1000 — after HivePress adds attribute fields at 100), so submitted values are ignored. HivePress saves only fields the form contains.
- Locked: **account type** for every non-admin; **first name, last name, carné** for hosts.
- wp-admin → Profile: hosts' first/last name are kept (`insert_user_meta` filter).
- Administrators are not restricted — correct a host's name or carné in wp-admin. The Didit webhook (3.26) writes names with `update_user_meta` and is not affected.
- V1.0's "re-save `hp_acctype` on update" guard did nothing (HivePress fires `hivepress/v1/models/user/update` after the save) and was removed.

---

### 3.11 Register Modal Warning (/whycolibri/ only)
**DEACTIVATE AT LAUNCH** (guests can register after launch)  
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

### 3.18 V3.2 Guest Pricing Display
**Type:** PHP — Run Everywhere  
**Requires:** 3.0 Config  
**Replaces:** the three former snippets 3.18 Inclusive Pricing — Properties Page, 3.19 Inclusive Pricing — Listing Page, 3.20 Booking Details — Service Fee Display (and V2.0). Sections 3.19–3.21 are retired into this one.

**DMCC Act 2024 — no drip pricing:** every price a guest sees before checkout includes the mandatory service fee, and the total at checkout equals the prices shown from the start. The host's **stored** prices are never changed — HivePress Marketplace adds the buyer commission on top of them at checkout, which is exactly what is displayed.

**1. Listing price — server-side** (`hivepress/v1/fields/currency/display_value`, priority 20)  
HivePress passes every field's display value through this filter (`HivePress\Fields\Field::display()`). The snippet acts only on the front end and only on fields named `price`, and returns `raw price × (1 + rate)` formatted by HivePress's `format_price()`. The inclusive price is therefore in the HTML from the first byte on every template that displays the listing price: property cards, single listing, map popups, favourites, host profile, vendor listings, and daily prices (`booking_ranges` sub-field, also named `price`). Other currency fields — security deposit (`booking_deposit`, excluded from the fee), payout amounts, balances, order totals — are untouched. wp-admin shows the host's own figure.

**2. Extras (and tiers) — server-side** (`hivepress/v1/forms/booking_make`, priority 20)  
HivePress Bookings builds the extras checkbox labels in `alter_booking_make_form()` (priority 10) with `format_price($item['price'])` directly, outside the field filter. The snippet rebuilds those labels from the listing's own data, using HivePress's translatable formats ("price extra format", "per day" / "per booking") and the inclusive price — e.g. Breakfast €5.00 per day → "Breakfast (€5.50 per day)". The listing is obtained exactly as HivePress does: `$form->get_model()->get_listing()`. Tiers are handled the same way (option currently disabled).

> **Developer note:** HivePress model getters (`get_listing()`, `get_price_extras()`, …) are magic `__call` methods. `method_exists()` returns **false** for them — never guard model calls with it (this was the V3.1 bug).

**3. Presentation only — JavaScript** (`wp_footer`)
- **Listing page:** "includes 10% service fee" note with (i) button under the (already inclusive) price; re-added by a MutationObserver if HivePress re-renders the price block.
- **Booking details page** (`/make-booking/details/`): two rows after Price — **Service Fee (10%)** (price × rate + fixed fee, with (i) button) and **Total Price** (bold). Itemising here is permitted because the total equals the inclusive prices shown earlier.
- **Service fee modal** (both pages), text built from `SC_CONFIG.feeLabel`:
  - Stay Colibri charges a 10% service fee on all bookings. This fee covers maintaining the platform, payment processing, and host identity verification, ensuring a safe and secure booking experience for both guests and hosts.
  - Stay Colibri does not charge any fees to Cuban hosts — the service fee is paid by guests only.
  - The service fee is already included in the prices shown on the site and will be shown separately in your total at checkout.
  - If you choose to cancel your booking, the service fee is non-refundable. If your host cancels, the service fee will be refunded in full.
- If `SC_CONFIG` is missing, the JS falls back to 10% and logs a console warning.

**Fixed fee (if ever set):** a per-booking amount cannot be expressed in a nightly price; it is included in the booking details breakdown and checkout. Listing cards would then need wording such as "+ €X booking fee".

**Verifying compliance:** use **View Source** (not Inspect) on `/properties/` and a listing — the inclusive prices must be in the raw HTML. Then book a test stay with an extra: the booking details total and the checkout total must equal nights × displayed nightly price + displayed extras.

**Troubleshooting:**
- Prices **exclusive** everywhere: 3.18 inactive or not Run Everywhere.
- Prices **multiplied twice** (e.g. €60.50 for a €50 host price): an old JS pricing snippet (former 3.18/3.19/3.20 or V2.0) is still active — deactivate it.
- Extras exclusive but listing price inclusive: HivePress Bookings changed `alter_booking_make_form()`; check the filter name and option keys.
- Guest calendar shows no prices: expected — see Backlog.

---

### 3.22 V1.1 Host Cancellation Flow
**Type:** PHP + JS  
**Hosts only (V1.1):** the reason dropdown is added only for hosts (108), and the AJAX handler stores a reason only if the logged-in user is the author of the booking's listing (or an administrator). V1.0 accepted a reason from any logged-in user, so a guest cancelling their own booking could trigger the host-cancellation emails (full refund including the service fee). Guest cancellations now store no reason and this flow does nothing for them.  
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

### 3.25 V1.0.3 Unverified Host Lockdown (Pre-launch)
**DEACTIVATE AT LAUNCH**  
**Type:** PHP — WPCode location must be **Auto Insert → Run Everywhere**  
**Hook:** `template_redirect`, priority 1  
**What it does:** Logged-in hosts (`hp_acctype = 108`) without `hp_verified` are redirected to `/verification-pending/` from **every** front-end page except the allowlist:
- `/verification-pending/` (the destination — prevents a redirect loop)
- `/contact/` (so they can ask for help)
- any path containing `/logout`

**Not affected:** logged-out visitors (handled by 3.8 Coming Soon), guests (107), verified hosts, administrators (`manage_options`). Because it runs on `template_redirect`, `wp-login.php`, wp-admin, admin-ajax and the REST API are untouched, so login/logout and the Didit webhook endpoint (3.26) keep working. Verification takes effect on the next page load after 3.26 sets `hp_verified = 1`.

**Versions:** V1.0 (priority 5) was what ran until October 2026; V1.0.2 (priority 1, WPCode location note) had not been deployed. V1.0.3 = V1.0.2 + comments updated for 3.26.

**Relationship to the Didit callback:** Didit redirects the host's browser to the **home page** after verification (set by 3.27). This snippet then sends a host who is not yet verified — including one whose webhook has not arrived yet — to `/verification-pending/`.

**Relationship to 3.2:** 3.2 Submit Listing Access Control stays active after launch and keeps unverified hosts out of `/submit-listing`. This snippet only extends the restriction to the whole site during pre-launch.

**`/verification-pending/` page content:** it is the only page an unverified host can see, so it must explain that the Didit link was emailed (check spam), that review can take up to 3 working days, give a contact route, and include a log-out link (`wp_logout_url()` / `wp-login.php?action=logout`). Written in *tú*, matching the host emails.

**Privacy page:** `/privacy/` is not on the allowlist, so an unverified host cannot open the privacy policy from the cookie banner while 3.25 is active. Add `/privacy/` to `$allowed`.

**Troubleshooting:**
- Snippet has no effect at all: check the WPCode location is **Run Everywhere**. A Site Wide Header/Footer location loads the code after `template_redirect` has fired, so it silently does nothing.
- Verified host still redirected: check `hp_verified` is `1` in `wp_usermeta`, and the host's line on `/?sc_didit_log=1` (3.26); clear any page cache.
- Redirect loop: the `/verification-pending/` slug changed — update the allowlist.
- To let unverified hosts reach another page, add its path to `$allowed`.

---

### 3.26 V1.0.4 Didit Verification Webhook
**Type:** PHP — Run Everywhere  
**Replaces:** Zapier Zap 3 and 3.1 `/verify-user`  
**Endpoint:** `POST /wp-json/custom/v1/didit-webhook` (registered on blog 14 only)

**wp-config.php:**

    define('SC_DIDIT_WEBHOOK_SECRET', '[IN PASSWORD MANAGER]'); // the Didit destination's signing secret
    define('SC_DIDIT_ENVIRONMENT',    'sandbox');               // 'sandbox' or 'live'
    define('SC_ADMIN_EMAIL',          'jonny@auno.uk');         // optional

**Security checks, in order:**
1. Config present — otherwise **503** (Didit retries while it is fixed).
2. `X-Timestamp` within 5 minutes — otherwise **401** (replay protection).
3. HMAC-SHA256 signature: `X-Signature` on the raw body, falling back to `X-Signature-V2` on canonical JSON (keys sorted, objects kept as objects, unescaped slashes/Unicode/line terminators). **`X-Signature-Simple` is never accepted** — it does not cover the decision body. Failure → **401**, with a diagnostic in the log (headers present, body length, secret length and an 8-character fingerprint, and whether the envelope-only Simple signature matches — "no match" = wrong secret, "MATCHES" = body altered in transit).
4. Try Webhook test deliveries (`X-Didit-Test-Webhook: true`) are logged as `test_webhook_ok` with the ID-result field names, and change nothing.
5. Only `webhook_type = status.updated` from the configured environment is processed (others → 200, ignored).
6. `event_id` processed once (`sc_didit_evt_*` transient, 14 days).
7. `vendor_data` must be the WordPress user ID of a host (108) who is a member of blog 14, **and** `session_id` must equal the host's `sc_didit_session_id`. An Approved event matching no host emails the admin.

**Status handling** (Didit statuses are exact, case-sensitive strings):
| Status | Action |
|---|---|
| Approved | `hp_verified = 1`, `sc_didit_verified_at`; `first_name`, `last_name` (**title case**, below) and `hp_carne` written **only when Didit returns a non-empty value** (number from `personal_number`, else `document_number`; the log records which); Didit's original names kept in `sc_didit_first_name_raw` / `sc_didit_last_name_raw`. Then `wp_update_user(ID)` fires `profile_update` so **HivePress sets the display name from its own setting** (HivePress → Settings → Users → Display Name; currently *Full name*). Missing values → admin email. Fires `sc_didit_host_approved` (3.28 Brevo). If the approval followed a manual review, the host is emailed (below) |
| Declined, In Review, Kyc Expired | Admin email with Didit's warning codes. `hp_verified` is not changed (a Declined session can still become Resubmitted) |
| Abandoned, Expired | Admin email; fires `sc_didit_session_inactive`. Host needs a new link (3.27 **Send verification link**) |
| Not Started, In Progress, Resubmitted, Awaiting User | Status recorded only |

**Names (V1.0.4):** Didit reads names off the document in capitals. `sc_didit_title_case()` converts them before storing: lowercase then capitalise each word; Spanish/Portuguese/Italian/Dutch particles **inside** a name stay lowercase (de, del, la, las, los, y, e, da, das, do, dos, di, van, von); the first word of each field is always capitalised; each part of hyphenated and apostrophe names is capitalised. Examples: `JOSÉ MARÍA` → José María; `PÉREZ DE LA CRUZ` → Pérez de la Cruz; `DEL CASTILLO RUÍZ` → Del Castillo Ruíz; `MARÍA-JOSÉ` → María-José. Dual Cuban surnames stay together in `last_name`. Nothing splits names on spaces. Brevo receives the stored (title-case) names via 3.28. An admin can correct an unusual result in wp-admin.

**ID data from Didit (v3 payload):** `decision.id_verifications[]` — the first item with status Approved is used, otherwise the first item. A manual approval can leave the ID item at "In Review" while the session is Approved; this is handled.

**Host approval email** (Spanish, "Stay Colibrí — Tu cuenta de anfitrión está verificada", link to `/submit-listing/`, Reply-To info@staycolibri.com): sent **only when the approval followed a manual review** — previous status In Review / Resubmitted / Declined, or the payload shows a reviewer (`trigger` manual_review / manual_step_update, or `decision.reviews` not empty). Automatic approvals send nothing (the host sees the result on screen). Once per session (`sc_didit_approval_emailed`). First name title-cased for the greeting only. **Not yet on the 3.31 design** — see the rollout item in the checklist.

**Admin and host emails** are sent from scheduled single events (`sc_didit_admin_notice`, `sc_didit_host_approval_email`), never inside the webhook request (Didit times out after 5 seconds).

**Log:** `/?sc_didit_log=1` (admins) — last 100 events: time (UTC), outcome, status, environment, user, session, event, signature method, detail. Also used by 3.27 and 3.28. Never contains names, ID numbers, phone numbers or verification links.

**Log outcomes:** `approved_verified`, `approved_verified_after_review`, `admin_notified`, `status_recorded`, `no_matching_host`, `ignored_session_mismatch`, `ignored_wrong_environment`, `ignored_type_*`, `duplicate_ignored`, `test_webhook_ok`, `rejected_bad_signature`, `rejected_stale_or_missing_timestamp`, `rejected_bad_json`, `config_missing`.

**Troubleshooting:**
- `rejected_bad_signature` with "simple: no match (secret wrong)": the secret in wp-config is not the one for **this destination in this application** (each destination has its own; the API key is different). Regenerate the destination secret and paste it immediately. This happened during setup in October 2026.
- No log lines at all: check the destination's **Deliveries** tab in Didit. A 403 means Cloudflare blocked Didit (deliveries come from `18.203.201.92`, user agent `DiditWebhook/2.0`) — add an allow rule for the webhook path.
- `ignored_session_mismatch`: the event belongs to an older session (after a resend), or the session was not created by 3.27 (e.g. a Didit console link). Expected in both cases.
- `no_matching_host`, user 0: session has no `vendor_data` — created in the Didit console, not by the site.
- Didit retries a failed delivery only twice (~1 min, then ~4 min) and then drops it. If a host is approved in Didit but not verified here, check Deliveries and replay the event from the Didit console.

---

### 3.27 V1.7 Didit Verification Session + Email
**Type:** PHP — Run Everywhere  
**Requires:** 3.26; 3.31 V2.4+ (builds and sends the email; a plain fallback is sent without it)  
**Replaces:** Zapier Zap 1 (Didit session + email steps)

**Versions (October 2026):** V1.3 first redesign (logo header, VPN warning, numbered steps, CID logo). V1.4 email rebuilt on 3.31 (amber hero, coral VPN band, coloured step tiles) and subject changed to one without a dash. V1.5 link colour read from the 3.31 palette. V1.6 step 1 says the carné only (passports are not accepted). V1.7 logo replaced with the 700 px file, trimmed to 606 × 200. Only `sc_didit27_link_email_html()`, the subject and the logo data changed; session, retry and admin logic is as V1.2.

**wp-config.php:**

    define('SC_DIDIT_API_KEY',     '[IN PASSWORD MANAGER]'); // API key of the sandbox or live Didit application
    define('SC_DIDIT_WORKFLOW_ID', '[IN PASSWORD MANAGER]'); // workflow in the SAME application

The API key, workflow ID and webhook destination (3.26) must all belong to the same Didit application, and `SC_DIDIT_ENVIRONMENT` must match it.

**Trigger:** the host **confirms their email address** — HivePress fires `hivepress/v1/models/user/verify` (also on admin confirmation, or straight after registration if HivePress email verification is turned off; an email *change* does not fire it). The job is queued as a single event (`sc_didit_verification_job`) and runs on the next page load, so the host's redirect is not held up. Hosts never receive two verification emails at once.

**Job:**
1. Checks the user is an unverified host (108) on blog 14; if `hp_acctype` is not saved yet, retries.
2. Skips if a link was already sent (the automatic job only).
3. `POST https://verification.didit.me/v3/session/` with `workflow_id`, `vendor_data` = user ID, `callback` = home page, `language` = `es`. Didit returns the host's **existing unfinished session** if there is one, so retries and resends never create duplicates; an expired/abandoned/declined session gets a new one.
4. Stores `sc_didit_session_id` (and resets `sc_didit_status` to Not Started for a new session).
5. Emails the link (Spanish, subject "Verifica tu identidad en Stay Colibrí"; built by `sc_didit27_link_email_html( $logo_src, $url )` from the 3.31 blocks). Top to bottom: four-colour stripe, logo, amber hero ("¡Casi listo! Falta comprobar que eres tú."), **coral band "Antes de empezar, apaga la VPN."** (verification fails with a VPN, so it sits above the button), button "Verificar mi identidad" with "Mejor desde el teléfono y con buena luz.", three steps with coloured number tiles (**carné de identidad only — passports are not accepted**; scan on a flat surface in good light; follow the on-screen steps), "usually immediate; manual review can take up to 3 working days and we'll email you", the copy-paste link, help/expired-link contact, sign-off "Un abrazo, El equipo de Stay Colibrí", footer with company number 15560964. Hidden inbox preview line: "Apaga la VPN antes de empezar. Con ella conectada, la verificación no funciona." Reply-To info@staycolibri.com. The **logo is embedded** as an inline CID attachment (`cid:sc-logo`, held as base64 in `sc_didit27_logo_png()`, added via a one-off `phpmailer_init` hook), so nothing is fetched from S3 and it shows even when remote images are blocked — this relies on WP Mail SMTP passing PHPMailer's complete message to Amazon SES (true for the SES SMTP mailer; confirm by test for the SES API mailer — if the logo is missing, the email shows "Stay Colibrí" as text). Logo details are under 3.31.
6. Retries after 5 min, 30 min and 2 h on network errors, 429, 5xx or email failure. 400/401/403 (bad workflow ID or key) are not retried. Final failure → admin email "Didit link NOT sent" (plain text).

**Suppressed HivePress email:** hosts no longer receive HivePress's "Registration Complete" email ("Thank you for registering, now you can access website areas…") — its body is emptied via the `hivepress/v1/emails/user_register` filter, so HivePress skips it. Guests still receive it. Logged as `welcome_email_suppressed`.

**Edit User screen — "Stay Colibri — Didit verification" box** (hosts only): verified (and when), Didit status, session (and when created), link last emailed and count, last error, **Send verification link** — same link if the current session is still live, otherwise a new session. Use it after Expired / Abandoned / Declined, when a host says the email never arrived, for accounts created by hand in wp-admin (which never trigger the job), or to test the email design.

**Log outcomes:** `session_link_sent`, `session_link_resent`, `session_job_failed`, `session_resend_failed`, `session_job_skipped`, `welcome_email_suppressed`.

**Troubleshooting:**
- "HTTP 400" / "HTTP 403": workflow ID from another application (e.g. the live workflow with the sandbox key), unpublished workflow, or wrong key.
- No email and no log line: WP-Cron has not run — load any page; set up a server crontab (checklist).
- Email arrives as a short plain message: 3.31 is inactive or not Run Everywhere (see 3.31 troubleshooting).

---

### 3.28 V1.3 Brevo Contact Sync
**Type:** PHP — Run Everywhere  
**Requires:** 3.26, 3.27  
**Replaces:** Brevo steps of Zapier Zaps 1 and 2

**wp-config.php:**

    define('SC_BREVO_API_KEY',         '[IN PASSWORD MANAGER]'); // Brevo v3 API key
    define('SC_BREVO_GUEST_LIST_ID',   3);                        // optional, default 3
    define('SC_BREVO_PHONE_ATTRIBUTE', 'PHONE');                  // optional, default PHONE
    define('SC_BREVO_SEND_SMS',        true);                     // optional, default true
    define('SC_PHONE_META_KEY',        'hp_phone');               // optional, default hp_phone

**When:** a user confirms their email (guests and hosts — unconfirmed addresses never reach Brevo); Didit approves a host (`sc_didit_host_approved`); an admin clicks **Sync to Brevo now** (Edit User screen, "Stay Colibri — Brevo" box: last synced, last error). Queued as `sc_brevo_sync_job`; retries 5 min / 30 min / 2 h; final failure → admin email.

**What is sent** (`POST /v3/contacts`, `updateEnabled: true`, built from current WordPress data on every sync):
| Attribute | Host | Guest |
|---|---|---|
| `ACCOUNT_TYPE` (category) | 1 (Host) | 2 (Guest) |
| `ID_VERIFIED` (Boolean) | `hp_verified` | false |
| `HIVEPRESS_USER` / `HIVEPRESS_USER_ID` | nickname / user ID | nickname / user ID |
| `FIRSTNAME` / `LASTNAME` | from Didit, once verified (as on the document) | `LASTNAME` = nickname (as Zap 2 did) |
| `SMS` (built-in) and `PHONE` (Text) | `hp_phone`, cleaned | `hp_phone`, cleaned |
| List | none | 3 |

**Never sent:** carné / ID number, address, payout details.

**Phone:** cleaned (spaces, dashes, dots, brackets removed; leading `00` → `+`); no country code is guessed (the sign-up form collects it). `SMS` is validated by Brevo (real numbering plan, correct length — e.g. Cuban mobiles are +53 then 8 digits starting with 5) and must be **unique across contacts**; `PHONE` has no rules, so it always holds the number for customer service. A 400 response is retried without `SMS`, then without `PHONE`, so the contact always syncs; the drop is logged and shown in "Last error".

**Brevo attributes required:** `ACCOUNT_TYPE` (category: 1 Host, 2 Guest, 3 PreLaunch), `ID_VERIFIED` (**Boolean** — recreated from multiple-choice in October 2026), `HIVEPRESS_USER`, `HIVEPRESS_USER_ID`, `FIRSTNAME`, `LASTNAME`, `PHONE` (Text), `SMS` (built-in). Brevo silently ignores attributes that do not exist or values of the wrong type.

**Log outcomes:** `brevo_synced` (detail lists the attributes sent, or why no phone was sent), `brevo_failed`.

**Troubleshooting:**
- Attribute empty in Brevo but listed in "sent:": the attribute does not exist in Brevo, or has a different type.
- "synced without: SMS (Invalid phone number)": wrong length or not a valid mobile range — `PHONE` still has it.
- "synced without: SMS (… already associated with another Contact)": shared number — `PHONE` still has it.
- "no phone in hp_phone": the sign-up form stores the phone under another key — set `SC_PHONE_META_KEY`.
- Brevo shows a multiple-choice attribute as a count ("+1", "+2"), not its value.

---

### 3.29 V1.1 Listing Photo Limit
**Type:** PHP — Run Everywhere  
**Hook:** `hivepress/v1/models/listing` (priority 100)  
**What it does:** sets the listing `images` field's `max_files` to **30** (V1.0, the undocumented "Increase Number of Photos", set 100). HivePress enforces it on upload. Caps per-listing storage on the media bucket.

---

### 3.30 V1.0 Show User ID Shortcode
**Type:** PHP — Run Everywhere  
**What it does:** `[current_user_id]` prints the logged-in user's own WordPress user ID ("Not logged in" otherwise). Shows only the viewer's own ID. Formerly undocumented ("Show User ID").

---

### 3.31 V2.6 Email Layout
**Type:** PHP — Run Everywhere (a header/footer location would load too late for `wp_mail` callers)  
**Requires:** nothing. Holds the logo itself (`sc_email_logo_png()`, 606 × 200 PNG, same bytes as before); `sc_email_send()` embeds it as `cid:sc-logo`  
**Used by:** 3.4 (host change notice), 3.6 Section 6 (payout on its way), 3.22 (guest cancellation), 3.24 (adjustment and withdrawn), 3.26 (host approval), 3.27 (verification link). Each keeps its previous plain email as a fallback if 3.31 is inactive. **Admin emails stay plain text.** HivePress's own emails are not yet covered.

**V2.5 additions:** `sc_email_ledger( $rows, $title )` (two-column calculation like the dashboard balance modal: label left, amount right, `'total'` and `'strong'` lines) and `sc_email_month_es()` ("noviembre de 2026"). **V2.4 additions:** `sc_email_logo_png()` (moved from 3.27); `sc_email_amount( $label, $amount, $note )` (one big figure on a gold block with a hard edge); `sc_email_facts( $rows, $title )` (label above value, teal rule on the left); `sc_email_date_es()` (Spanish dates); `sc_email_signoff()`, `sc_email_footer()` and `sc_email_shell()` take a language, `'es'` (default) or `'en'`. **Language rule:** hosts get Spanish emails and guests get English emails, never both in one email. Only the guest cancellation email (3.22) is English; all host emails (3.4, 3.6 §6, 3.24, 3.26, 3.27) are Spanish.

**Purpose:** one shared design for transactional emails, so each new email is a few block calls and all of them look the same. Colours, fonts and logo size live in one function, `sc_email_tokens()`.

**House rules for email design** (set October 2026):
- No centred hero text with a glowing badge or "The future of…" tagline.
- No row of three floating feature cards with thin grey borders and generic icons.
- No pill buttons with soft drop shadows.
- No uniform card grids where every box has the same height and padding.
- Copy follows the same spirit: plain Spanish in *tú* for hosts, no filler greetings, no em dashes, every sentence carries a fact (see the "human-copy" rules: no vague praise, no "serves as", no triplets for effect).

**Look:** a four-colour stripe (deep green, emerald, gold, ruby-pink, in uneven widths) over the logo on white; then full-width colour bands (emerald hero with dark text, ruby-pink alert band with white text, pale-green body, deep-green footer); Verdana (fallbacks Geneva, DejaVu Sans, Tahoma; Trebuchet MS until V2.6); a 560 px left-aligned column inside each band; a button with a hard gold edge (a solid offset, not a soft shadow); numbered steps, each number in its own coloured tile (emerald, gold, ruby-pink, repeating).

**Palette "Zunzún"** (the Cuban bee hummingbird; chosen from five options in October 2026):
| Token | Hex | Used for | Contrast |
|---|---|---|---|
| `primary` | `#0a5c52` | button, footer, links, first stripe segment | white on it 7.9:1 |
| `hero` | `#36c2a8` | hero band, second stripe segment, step tile 1 | dark text 6.6:1 |
| `alert` | `#cf2f68` | alert band, last stripe segment, step tile 3 | white on it 4.9:1 (the first choice, `#e0457b`, gave 3.96:1 and was darkened) |
| `accent` | `#f5c542` | button edge, stripe, step tile 2 | dark text 9.0:1 |
| `ink` | `#0b2e2a` | main text | |
| `cream` | `#f3fbf6` | body background | muted text on it 5.1:1 |
| `muted` | `#55706a` | small print | |
| `footer_text` | `#cfeadf` | text on the footer | 6.2:1 |
Rule: any new colour used for text needs at least 4.5:1 against its background.

**Changing the colours, fonts or logo size:** edit `sc_email_tokens()` only (`primary`, `hero`, `alert`, `accent`, `ink`, `cream`, `muted`, `footer_text`, `font`, `logo_width`, `tiles`). Every block reads from it. Do not hard-code a hex value in an email; use `sc_email_tokens()['name']`.

**Blocks** (inputs are escaped HTML except where noted):
| Function | What it makes |
|---|---|
| `sc_email_shell( $logo_src, $preheader, $bands )` | The whole document: stripe, logo, your bands, footer. Preheader is the hidden inbox preview line (plain text) |
| `sc_email_hero( $kicker, $headline, $text_html )` | Amber/emerald hero band. Kicker and headline are plain text |
| `sc_email_alert( $title, $text_html )` | The one warning the reader must not miss. Title plain text |
| `sc_email_section( $inner, $pos )` | Pale body band. `$pos`: `'top'` (button), `'mid'` (steps), `'end'` (closing text) or none |
| `sc_email_button( $url, $label, $note )` | Button with hard edge, optional small line under it. Label plain text |
| `sc_email_steps( $items, $title )` | Numbered steps with coloured tiles. Title plain text |
| `sc_email_p( $html, $small )` | Paragraph; `$small` for secondary text |
| `sc_email_signoff()` | "Un abrazo, El equipo de Stay Colibrí" |
| `sc_email_footer()` / `sc_email_header()` / `sc_email_stripe()` / `sc_email_band()` | Parts used by the shell; `sc_email_band( $bg, $inner, $pad )` makes any custom full-width band |
| `sc_email_send( $to, $subject, $html, $reply_to )` | `wp_mail` with the logo embedded as `cid:sc-logo` for that message only; default Reply-To info@staycolibri.com; returns true/false |
| `sc_email_logo_src()` | Returns `cid:sc-logo` |

**Example** (how 3.27 builds its email, shortened):

    $bands  = sc_email_hero( 'Kicker', 'Titular', 'Texto del hero.' );
    $bands .= sc_email_alert( 'Aviso', 'Texto del aviso.' );
    $bands .= sc_email_section( sc_email_button( $url, 'Botón', 'Nota' ), 'top' );
    $bands .= sc_email_section( sc_email_steps( array( 'Uno', 'Dos' ), 'Qué vas a hacer' ), 'mid' );
    $bands .= sc_email_section( sc_email_p( 'Texto final.' ) . sc_email_signoff(), 'end' );
    $html   = sc_email_shell( sc_email_logo_src(), 'Texto de vista previa', $bands );
    sc_email_send( $to, 'Asunto', $html );

**Logo:** 606 × 200 PNG, about 7 KB, held as base64 in `sc_didit27_logo_png()` in 3.27 and attached to each email as `cid:sc-logo`. It is the 700 px `logo2_small-2.png` (S3 `colibrimedia/2026/10/`) with its empty transparent margin trimmed, so the wordmark lines up with the text edge. Shown at `logo_width` = 200 px (about 3× pixel density, sharp on any phone). To change it, replace the data in that function; keep `logo_width` under about half the file's pixel width. The original 350 px file (4 KB) was replaced because it was soft above about 175 px.

**Email client notes:** table layout and inline styles throughout (what Gmail and Outlook support). In Outlook desktop, rounded corners show square and the button's hard edge may be flat; nothing breaks. No web fonts. The design is for light mode; a client that force-inverts colours may alter it. Previews were rendered headless at 390 px (phone) and 900 px (desktop); **not yet checked in Outlook desktop or the Gmail app** (checklist).

**Deployment order:** 3.31 V2.4 first, then 3.4, 3.6, 3.22, 3.24, 3.26 and 3.27 in any order (3.27 V1.8 no longer holds the logo, so it needs 3.31 V2.4+ to show it). Earlier note: 3.31 first, then 3.27. 3.27 V1.5+ reads `sc_email_tokens()['primary']`, so it needs 3.31 V2.1+ (the V2.0 token names differ); V1.7 needs V2.3 for the logo width.

**Troubleshooting:**
- The email arrives as a short plain message: 3.31 is inactive or its WPCode location is not Run Everywhere.
- Logo missing and "Stay Colibrí" shown as text: the mailer did not pass PHPMailer's attachment (possible with the SES API mailer), or 3.27 is inactive so `sc_didit27_logo_png()` does not exist.
- "Undefined array key" notices mentioning `navy`, `amber`, `teal` or `primary`: 3.31 and 3.27 are different generations. Update both to the versions in the inventory.
- Colours look different in one client: that client inverts or adjusts colours in dark mode.

### 3.32 V1.3 HivePress Email Wrapper
**Type:** PHP — Run Everywhere  
**Requires:** 3.31 V2.4+ (nothing is wrapped without it)  

**Purpose:** HivePress sends its 25 emails as a bare HTML body with no wrapper, through `wp_mail`. The text comes from HivePress → Emails (all are currently the HivePress defaults, in English; the site is English only). 3.32 wraps each one in the 3.31 design without changing HivePress or its text.

**How it works:** HivePress fires `hivepress/v1/emails/{name}/send` just before `wp_mail()`. 3.32 records the event name and subject, then a `wp_mail` filter wraps that one message: small label (from the event name) and the subject as headline in the hero; the body text below; the first bare link in the body becomes a button (label per event, for example "Verify my email", "Reset my password", "Read the message"), with the URL repeated in small print; English sign-off and footer; logo embedded as `cid:sc-logo` for that message only. A message is wrapped only if it is HTML, is not already a full document, and matches the announced email (same subject, within 5 seconds); everything else (WooCommerce, our snippets, admin emails) passes through untouched.

**Suppressed (V1.1):** `booking_cancel_vendor` (host cancels), because 3.22 sends the detailed email to the guest; and `order_receive` ("Order Received", to the host when a booking is paid), because the host already gets "Booking Confirmed". The body is emptied so HivePress sends nothing, the same method 3.27 uses for `user_register`. The list is `sc_hpmail_suppressed()`. **Host-only suppression (V1.3, `sc_hpmail_suppressed_for_hosts()`):** `order_complete` ("Order Completed") is not sent to any host, because 3.6 sends its own checkout emails (Section 6 Tropipay, Section 6b cash); `payout_complete` ("Payout Completed") is not sent to cash hosts, because 3.6 Section 10 sends the cash delivery confirmation (Tropipay hosts still get it). Checked at send time against the recipient's `hp_acctype` and `sc_payment_model` and stopped with `pre_wp_mail`; a guest or admin recipient still receives them.

**Changing the look or text:** colours and layout live in 3.31. To change wording, edit the email in HivePress → Emails (tokens as usual); the wrapper keeps working. Button labels and the small label above the headline are in `sc_hpmail_button_label()` and `sc_hpmail_events()`.

**Troubleshooting:** an email arrives unwrapped (plain HivePress look): 3.31 inactive, 3.32 inactive, or the email is not in the event list in 3.32 (add its name). A guest still gets a HivePress cancellation email on top of 3.22's: check that the email is `booking_cancel_vendor` (the name shown in HivePress → Emails may differ) and add the right name to `sc_hpmail_suppressed()`.

---

### 3.33 V1.0 Image Size URL Fix (Next3Cloud)
**Type:** PHP — Run Everywhere  
**Hook:** `wp_get_attachment_image_src`, priority 99 (after Next3Cloud's own filter at 10)  
**Keep active for as long as Next3Cloud is in use.** If Next3 fixes its bug, 3.33 returns the same URLs Next3 would, so it is harmless to leave on.

**The problem (found October 2026):** on listing pages, portrait photos rendered much taller than landscape ones. The cause is not the theme: RentalHive's gallery template (`themes/rentalhive/hivepress/listing/view/page/listing-images.php`, same as HivePress's own) asks for `hp_landscape_large` (800 × 600, cropped 4:3) for each slide and `large` for the zoom. WordPress generates those crops on upload, Next3Cloud offloads them to the bucket, and the attachment metadata lists all of them correctly. But **Next3Cloud's `wp_get_attachment_image_src` filter returns the original file for every requested size, with 150 × 150 as the dimensions** (even for `full`). So every image on the site — gallery slides, listing cards, thumbnails — was served at full size (1600 px wide or more): portrait photos kept their 2:3 shape in the gallery, and each listing page downloaded several MB of images.

Diagnosed with a temporary admin page (`/?sc_img_diag={attachment_id}`, deleted afterwards) that showed: sizes present in the raw and filtered metadata; `wp_get_attachment_url` correct; every size resolving to the original at 150 × 150; Next3Offload\Modules\Compatiblility hooked on `wp_get_attachment_url`, `wp_get_attachment_image_src`, `wp_get_attachment_metadata` and `image_get_intermediate_size`. Reported to Next3 in October 2026 (section 4.7).

**What it does:** rebuilds the size URL the way WordPress core does — the folder of `wp_get_attachment_url()` plus the size's filename from the attachment metadata — with the real width and height from the metadata.
- Named sizes (`hp_landscape_large`, `large`, …): taken from `$meta['sizes'][ $size ]`.
- Array sizes (`array( w, h )`): via `image_get_intermediate_size()`.
- `full`, or a size with no file (image smaller than the size): the original, with its real dimensions.
- Non-image attachments (e.g. listing videos) and attachments without metadata are left alone.

**Checking it works:** a listing gallery's slides should all be the same height and load `…-800x600.jpg` files; the zoom loads `…-683x1024.jpg` (portrait) or `…-1024x683.jpg` (landscape); `/properties/` cards load `…-400x300.jpg` files. In the browser console on a listing:

    [...document.querySelectorAll('.hp-listing__images img')].map(i => i.src.split('/').pop() + ' ' + i.naturalWidth + 'x' + i.naturalHeight)

should show 800 × 600 for every slide.

**Troubleshooting:**
- Slides different heights again / originals loading: 3.33 inactive, or Next3 changed its filter priority above 99 — raise 3.33's priority.
- An image is broken (404) after 3.33: that size file is not in the bucket — check Next3Cloud is set to offload all image sizes, not just the original.
- Note on file names: some older uploads end in `.jpg.jpg` (an artefact of the WebP-to-JPEG converter used on those photos, unrelated to the site). The size files follow the same pattern (`…-scaled.jpg-800x600.jpg`) and work normally.

---

### 3.34 V1.1 Turnstile on HivePress Account Forms
**Type:** PHP — Run Everywhere  
**Stays active after launch.**  
**Hooks:** `rest_pre_dispatch` (server-side check), `wp_footer` priority 100 (widget, logged-out visitors only)

**The problem:** the Simple CAPTCHA Alternative with Cloudflare Turnstile plugin protects `wp-login.php`, WordPress password reset and the contact form, but HivePress's front-end login, register, forgot-password and reset-password forms do not use those pages — they post to HivePress's REST API. Both the modal forms and direct calls to the API (scripted brute force) bypassed Turnstile.

**wp-config.php:**

    define('SC_TURNSTILE_SITE_KEY',   '[IN PASSWORD MANAGER]');
    define('SC_TURNSTILE_SECRET_KEY', '[IN PASSWORD MANAGER]');

If these are not defined, 3.34 uses the Turnstile plugin's own settings (`cfturnstile_key` / `cfturnstile_secret` options). Same keys as the plugin.

**1. Server side — the actual protection.** Every `POST` to these routes must carry a valid Turnstile token in `cf-turnstile-response`:
| Route | Form |
|---|---|
| `/hivepress/v1/users/login` | Login (modal and `/account/login/`) |
| `/hivepress/v1/users` | Register |
| `/hivepress/v1/users/request-password` | Forgot password |
| `/hivepress/v1/users/reset-password` | Reset password page (from the email link) |

Routes confirmed against `plugins/hivepress/includes/controllers/class-user.php` (October 2026). The check matches the REST route itself, so it also covers the `/?rest_route=…` form of the URL. The token is verified with Cloudflare siteverify (`remoteip` from `CF-Connecting-IP`), and the hostname in the response must be the site's. Rejected requests get HTTP 400 with HivePress's own error shape (`['error' => ['code' => 400, 'errors' => [['message' => …]]]]`, as `hivepress/includes/helpers.php` `rest_error()`), so the form shows **"Please complete the security check and try again."** HivePress never runs for a rejected request.

**Fails closed:** no token, an invalid or reused token, missing keys, or Cloudflare unreachable → rejected (an unreachable siteverify is written to the PHP error log). Admin login through `wp-login.php` is not affected by 3.34 and remains available.

**2. Front end — the widget.** On pages for logged-out visitors, finds HivePress forms (`form.hp-form`) whose `data-action` (HivePress's AJAX URL) or `action` points at one of the routes above and renders a Turnstile widget (managed mode, flexible width) above the submit button.
- Rendered only once the form is **visible**, and rendered afresh each time a modal opens: the modal script moves the form into its container, and moving an iframe reloads it, which left V1.0's widgets blank.
- Tokens are single-use: after every submit the widget is reset (after HivePress has read the form data), so a wrong password followed by a retry works.
- Loads Cloudflare's `api.js` (explicit render) unless the Turnstile plugin has already loaded it on that page.

**Versions:** V1.0 matched forms on `action` only, so no widget appeared (HivePress uses `data-action`); the server-side check worked from V1.0. V1.1: `data-action` matching, visibility-based rendering, re-render on modal open.

**Options:** for an invisible widget unless Cloudflare is suspicious, add `appearance: 'interaction-only'` to the render options. To fail open when siteverify is unreachable (not recommended: disrupting that call would then bypass the check), make `sc_ts_verify()` return true on a network error.

**Testing:**
- Server side: `curl -s -X POST https://staycolibri.com/wp-json/hivepress/v1/users/login -d username_or_email=test -d password=test` → HTTP 400 with the security-check message (before 3.34, HivePress's "invalid username or password").
- In a private window: login modal shows the widget; close and reopen it — still there; wrong password then right password → logged in. Repeat for Register, Forgot password and the reset-password page.

**Troubleshooting:**
- Every attempt says "Please complete the security check": the widget is not rendering. With the modal open, run in the console:

      [...document.querySelectorAll('form')].map(f => [f.className, f.getAttribute('action'), f.getAttribute('data-action')]).concat([['turnstile loaded', typeof turnstile]])

  The target forms must show a `data-action` ending in one of the routes above, and `turnstile` must be `object`.
- Widget shows but always fails: site key and secret are from different Turnstile widgets, or the hostname is not on the widget's allowed list in Cloudflare.
- Widget blank inside a modal: the modal was opened in a way the observer did not catch — check the click/MutationObserver triggers in the snippet.
- Nobody can log in on the front end after a key change: 3.34 fails closed. Log in via `/wp-login.php`, fix the keys.

---

### 3.35 V1.0.1 Mapbox Storage Block
**Type:** PHP — Run Everywhere  
**Stays active after launch.**  
**Hooks:** `wp_head` and `admin_head`, priority 1 (must print before `mapbox-gl.js` loads)

**The problem (October 2026 cookie audit):** Mapbox GL JS (v2.7.0, loaded by HivePress Geolocation from `api.mapbox.com`) writes two items to `localStorage` whenever a map loads: `mapbox.eventData.uuid` (a persistent device ID) and `mapbox.eventData:{base64 account name}` (last usage-event time and the account username). They exist for Mapbox's own usage statistics. On v2+ billing is per map load, counted without them, and the map draws without them. So they are not strictly necessary, and storing them without consent would breach PECR. Gating the map behind consent was rejected: maps run the whole location system.

**What it does:** a small inline script patches `Storage.prototype.setItem` / `getItem` so any key starting `mapbox.` is never written or read, and removes any such keys already stored. Mapbox then behaves as it does in private browsing: it keeps the values in memory for the page and still sends its usage event with the access token. Maps, address search and billing are unaffected. Other storage keys are untouched.

**Versions:** V1.0 printed the script with `?>` … `<?php`; WPCode rejected it ("Unclosed '{'"). V1.0.1 prints it from a nowdoc string (the closing `SCMAPBOXJS;` line must start at column 1).

**Licence note:** the Mapbox GL JS v2+ licence forbids modifying "marked portions of the code related to billing, accounting, and anonymized data collection". 3.35 changes no Mapbox code and stops no request; it reproduces the storage-unavailable condition the library already handles. Low risk, but review if Mapbox objects.

**Checking it works:** DevTools → Application → Local Storage → delete any `mapbox.*` keys; load the properties page and a listing and use the address search. No `mapbox.*` keys may reappear, and `events.mapbox.com` requests must still appear in the Network tab. Check the Submit Listing map still places the pin.

**Troubleshooting:** `mapbox.*` keys reappear: Mapbox changed its storage method (e.g. sessionStorage via another API) or 3.35 prints after `mapbox-gl.js` — check it is Run Everywhere at priority 1, then extend the key check.

---

### 3.36 V1.0 Cookie Settings Menu Link
**Type:** PHP — Run Everywhere  
**Stays active after launch.**  
**Hook:** `nav_menu_link_attributes`

**Why:** the Pressidium floating settings button is off, so a footer link is the site's only way to reopen cookie settings. Withdrawing consent must be as easy as giving it (UK GDPR / PECR), so this link is required. The WordPress menu editor cannot add the `data-cc` attribute Pressidium needs.

**What it does:** any menu item whose URL ends in `#cookie-settings` gets `data-cc="c-settings"` and `role="button"` in the page HTML. Pressidium (the Cookie Consent JS library) binds to that attribute when the page loads and opens the settings modal on click.

**Setup:** Appearance → Menus → footer menu → Custom Link: URL `#cookie-settings`, text `Cookie settings`.

**Troubleshooting:** the link only jumps to `#cookie-settings`: Inspect it — if `data-cc` is missing, the theme builds that menu without the standard filter (move the item to a standard menu, or extend the snippet); if present, check the Pressidium plugin is active and loaded on that page.

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

**Plugin note:** the Tropipay WooCommerce plugin logs `Undefined variable $hidden` at `wc-tropipay.php` line 322 on every load. Harmless, but it fills `debug.log` (section 10).

---

### 4.2 Didit Identity Verification
**Purpose:** Verifies host identity before they can list properties (hosts only; guests are not verified)  
**Console:** https://business.didit.me  
**API base:** https://verification.didit.me/v3/  
**Contracting entity:** Didit Identity Spain, S.L. (CIF B22929327, Barcelona) — the MSA assigns UK/EU/EEA/Swiss clients to the Spanish entity. Didit is our processor for the checks and an independent controller for its own security, abuse prevention and legal compliance.  
**Applications:** two — a **sandbox** application (test mode: mocked checks, sample documents, outcome picker, no billing) and the **live** application. Each has its own API key, workflow and webhook destination with its own signing secret. The site uses whichever set is in wp-config.php, and `SC_DIDIT_ENVIRONMENT` must match.

**Current mode: LIVE** (switched October 2026). Sandbox values are kept in the password manager for future testing; the sandbox webhook destination is disabled (it shares the site URL, and its events would now be rejected).

**Accepted document:** the Cuban carné de identidad only; passports are not accepted (the verification email says so). Check that the live workflow only allows that document type, otherwise a host could still submit a passport (see section 10).

**Data protection settings (October 2026):**
- **Retention:** live application set to **10 years** (Didit's default is unlimited). Rationale, as stated in the privacy policy: the ID and face images are evidence of who received our payments, needed for as long as payment records may be challenged — financial records run until 6 years after a host's last payout, so 10 years from the check covers a host active for about 4 years. Hosts active longer should be re-verified before their images expire (backlog).
- **Consent:** the live workflow shows Didit's own consent screen before the camera turns on (biometric data, UK GDPR Article 9). Didit makes this the client's responsibility, so keep it enabled.
- **Model training:** Didit uses verification data to train its models by default. The opt-out is not in the console; **ask Didit support to opt the organisation out** (checklist). The privacy policy discloses training until the opt-out is confirmed.

**Live API key:** permission **Sessions write** only (the site makes one call, create session — no decision or media access, so a leaked key cannot read hosts' ID images), and **restricted by IP to the server**. Requirements:
- the server's outgoing IP must be **static** (Lightsail: static IP attached);
- if the server sends over **IPv6**, its IPv6 address must be allowlisted too (symptom: "HTTP 403" on session creation);
- **server migration (OVH): add the new server's IP to the key before switching DNS**, otherwise new hosts get no verification link (admin is emailed "Didit link NOT sent").
If a future feature needs to read decisions (e.g. a sweep for dropped webhooks), add **Decisions read** then.

**Flow:** see "Host onboarding flow" in section 1, 3.27 (session + email) and 3.26 (webhook).

**Didit configuration:**
- Webhook destination: URL `https://staycolibri.com/wp-json/custom/v1/didit-webhook`, version **v3**, event **`status.updated`** only. Secret → `SC_DIDIT_WEBHOOK_SECRET`.
- Session callback (browser redirect after the flow): the home page, sent by 3.27 per session.
- Session expiry: configured per workflow in the console; expired sessions are reported (3.26) and a new link is sent from the Edit User screen.
- Cloudflare: allow Didit's delivery IP `18.203.201.92` on the webhook path if bot or WAF rules challenge it.

**Testing without a real ID:** in the sandbox application, run the hosted flow with a sample document and choose the outcome in the scenario picker, or approve/decline manually in the console. Sandbox extracted data is fabricated (the sample document's number arrives as `document_number`). **Try Webhook** (API & Webhooks) sends signed sample payloads to a destination.

**Troubleshooting:** see 3.26 and 3.27. If a webhook was dropped, replay it from the destination's Deliveries tab. Manually setting `hp_verified = 1` remains possible but leaves names, carné and Brevo unset — prefer replaying the event.

---

### 4.3 Brevo CRM
**Purpose:** CRM for guests and hosts (customer service) and email marketing  
**API:** https://api.brevo.com/v3/ (key in `SC_BREVO_API_KEY`)  
**Lists:**
- List 3: Pre-launch signups (ACCOUNT_TYPE = PreLaunch), plus guests synced by 3.28

**Contact attributes:** see 3.28. `ACCOUNT_TYPE` values: 1 = Host, 2 = Guest, 3 = PreLaunch.

**Integration:** 3.28 (direct API) for guests and hosts; 3.8 Coming Soon for pre-launch signups. Both use `SC_BREVO_API_KEY`.

**Consent:** guests are added to list 3 without a marketing opt-in (as Zap 2 did). A consent checkbox is a pre-launch to-do (section 8); the privacy policy (section 9 of the policy) already assumes it exists.

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

### 4.6 Cloudflare (DNS, proxy, WAF, Turnstile)
**Plan:** Free. `staycolibri.com` and `2uw.uk` are **separate zones** — rules on one do not apply to the other.

**Turnstile:** one widget (site key + secret, in the password manager) used by:
- the **Simple CAPTCHA Alternative with Cloudflare Turnstile** plugin — `wp-login.php`, WordPress password reset, contact form;
- **3.34** — HivePress login, register, forgot-password and reset-password (REST API + modals).

**WAF custom rules** (free plan: 5 rules, all actions except Log, no regex):
| Rule | Expression | Action | Notes |
|---|---|---|---|
| Block XML-RPC | URI Path equals `/xmlrpc.php` | Block | Added October 2026 (both zones). The dashboard showed about 4,300 matching requests over 30 days — bot traffic. **Jetpack is installed and uses XML-RPC to talk to WordPress.com** — the block breaks its connection unless Jetpack is removed or its IP ranges are allowed (checklist) |

If bot or WAF rules ever challenge Didit's webhook deliveries (3.26), add a Skip rule for source IP `18.203.201.92` on `/wp-json/custom/v1/didit-webhook`.

**Rate limiting rule** (free plan: 1 rule; Path field only, counted per IP, 10 s period, 10 s block):
| Expression | Rate | Action |
|---|---|---|
| URI Path is in `/wp-json/hivepress/v1/users/login`, `/wp-login.php` | 5 requests per 10 s per IP | Block for 10 s |

Kept at 5 deliberately: the free plan cannot filter by method (page loads of `/wp-login.php` count) and cannot allow for shared IPs, and Cuban users reach the internet through ETECSA carrier-grade NAT, so many hosts can share one public IP. A lower limit would block real users for almost no gain. The rule matches on path only, so `/?rest_route=/hivepress/v1/users/login` is not rate-limited — 3.34 still requires Turnstile on that form of the URL, which is why 3.34 is the protection and the Cloudflare rules are a second layer.

---

### 4.7 Next3Cloud (Next3 Offload)
**Purpose:** offloads the media library to S3-compatible storage (Peasoup Cloud, `eu-west-1`, bucket `colibri-media-001215`, folder `colibrimedia/YYYY/MM/`). New uploads and all generated image sizes are offloaded; URLs point at `https://s3.eu-west-1.peasoup.cloud/colibri-media-001215/colibrimedia/…`.

**Known bugs (reported to Next3, October 2026):**
1. **Every image size resolves to the original file** (`wp_get_attachment_image_src` filter returns the original URL with 150 × 150 dimensions). Worked around by **3.33** — keep 3.33 active.
2. **WP-CLI fatal error:** `array_key_exists(): Argument #2 ($array) must be of type array, null given` in `next3-offload/functions.php:1073` (`next3_offload_row()`), reached through Next3's `wp_get_attachment_url` filter when WooCommerce's CLI runner loads the REST index (`get_site_icon_url()`). Any `wp` command that loads WordPress with WooCommerce active fails. Consequences: no `wp eval`, `wp media regenerate` or `wp cron event run` while Next3 is active; use temporary admin-only snippets instead, and `curl`/`php wp-cron.php` for the server crontab (section 6). The front end is not affected.

**When Next3 replies with a fix:** update the plugin, then check `wp eval 'print_r( wp_get_attachment_image_src( 1004, "hp_landscape_large" ) );' --url=staycolibri.com` runs and returns the `-800x600` URL with 800 × 600. 3.33 can stay either way.

---

### 4.8 Pressidium Cookie Consent (cookie banner) and privacy policy
**Plugin:** Pressidium Cookie Consent 1.9.1 (built on the Cookie Consent JS library by Orest Bida). Settings: WordPress Admin → Settings → Cookie Consent. Settings are kept as JSON and exported/imported with **Export Settings / Import Settings** — the current file is `pressidium-settings-staycolibri-analytics.json` (keep a copy with this repository's working files, not in GitHub if it ever holds keys; it holds none today).

**Legal basis (October 2026 audit):** PECR as amended by the Data (Use and Access) Act 2025 (cookie changes in force 5 February 2026; PECR fines up to £17.5m) and the ICO's storage and access technologies guidance (April 2026). Opt-in is used for every non-essential item, for all visitors: EU guests get no equivalent of the UK analytics exemption, and Order Attribution data is stored per order, so it would not qualify anyway.

**Settings in force:**
| Setting | Value | Why |
|---|---|---|
| Mode / Autorun | Opt-in / on | Nothing optional runs before a choice |
| Auto-clear cookies | On | `sbjs_*` deleted when Analytics is withdrawn |
| Page scripts | On | Allows `type="text/plain" data-cookiecategory="…"` tagging |
| Force consent | Off | Blocking the page is not required |
| Cookie expiration | 182 days | Re-ask interval |
| Revision | 4 | **Bump on every change to cookies or purposes** so visitors are asked again |
| Banner buttons | Accept all / Reject all, equal style | Reject must be as easy as accept |
| Floating settings button | Off | Replaced by the footer link (3.36) |
| Record consents | On | Proof of consent (UK GDPR) |
| Hide empty categories | On | Targeting and Preferences are empty and hidden |
| Links | `/privacy/#cookies` | The policy page; the "8. Cookies" heading has HTML anchor `cookies` |

**Categories and inventory** (from DevTools scans of home, properties, listing, login, checkout and the Mollie round trip):

*Strictly necessary:* `pressidium_cookie_consent` (6 months); `^wordpress_logged_in_`, `^wordpress_sec_` (session, or 14 days with Remember me); `wordpress_test_cookie`; `^wp-settings-` (1 year); `woocommerce_cart_hash`, `woocommerce_items_in_cart` (session); `^wp_woocommerce_session_` (2 days, 7 days logged in); `customer-post-data` (session; see section 10); `ally-supports-cache` (localStorage, accessibility feature cache). Cloudflare Turnstile stores on Cloudflare's own domain (security, necessary). Mapbox stores nothing (3.35).

*Analytics (opt-in):* `^sbjs_` — WooCommerce Order Attribution (sourcebuster.js): traffic source, landing page, pages viewed and user agent, attached to the order as `_wc_order_attribution_*`. Session cookies; `sbjs_session` 30 minutes.

**Blocked Scripts tab (added by hand — not in the import file):**
| Script source | Category | Regex |
|---|---|---|
| `sourcebuster` | Analytics | ✓ |
| `order-attribution` | Analytics | ✓ |

Re-check both rows after every settings import.

**Reopening settings:** footer menu item "Cookie settings" (URL `#cookie-settings`, 3.36).

**Privacy and cookie policy:** published at `/privacy/` (draft kept as a Claude doc, "Stay Colibri — Privacy and Cookie Policy"). Controller: Inta Group Ltd, company 15560964, Suite RA01, Wood Street, London E17 3NU. Named recipients: Mollie, Tropipay, Didit. Other suppliers are described by service and location (hosting/email: UK and Ireland; media storage: UK; network security: US/worldwide; maps: US; CRM: France; cash delivery: "an FCA-authorised money transfer company"), with an offer to name any on request. Retention: closed accounts 12 months; enquiries 2 years; consent records 2 years; server logs 30 days; financial records 6 years after the financial year; host verification result, name and carné 6 years after last payout; Didit images 10 years.

**Testing (incognito):**
1. Before any choice and after Reject all: no `sbjs_*` cookies on any page, including checkout.
2. Accept all: `sbjs_*` cookies appear (immediately or on the next page).
3. Footer "Cookie settings" → switch Analytics off → Save: `sbjs_*` cookies deleted.
4. A test order after accepting shows an Origin (e.g. "Direct") on the order screen.
5. Settings → Cookie Consent → Consent Records shows the choices.

**Troubleshooting:**
- `sbjs_*` before consent: the Blocked Scripts rows are missing or the script URLs changed (check DevTools → Network for `sourcebuster` / `order-attribution`). Fallback: a snippet that enqueues the two WooCommerce scripts only when the consent cookie contains `analytics`.
- Banner links land on the coming-soon page: 3.8 is redirecting `/privacy/` (see 3.8).
- Footer link does nothing: see 3.36.

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

### Single events (queued on demand, run on the next page load)
| Hook | Queued by | What it does |
|---|---|---|
| `sc_didit_verification_job` (user, attempt) | 3.27 on email confirmation; its own retries | Creates the Didit session and emails the link |
| `sc_didit_admin_notice` (user, status, session, detail) | 3.26 | Admin email for Declined / In Review / Abandoned / Expired / Kyc Expired / incomplete or unmatched approvals |
| `sc_didit_host_approval_email` (user, session) | 3.26 | Spanish approval email after a manual review |
| `sc_brevo_sync_job` (user, attempt) | 3.28 on email confirmation and Didit approval; its own retries | Creates/updates the Brevo contact |

### Checking Cron Status
Install WP Crontrol to view scheduled events. `sc_monthly_payout_cron` must show **Once Daily**. The `sc_monthly` (30-day) schedule is legacy and unused.

### Note on WP-Cron
WP-Cron only runs when the site receives a request. Section 8's once-per-month claim and catch-up make a missed batch day safe for cash. Section 6 is not safe (see section 10). The verification and Brevo jobs normally run within seconds because the host's own next page load triggers them, but they also depend on traffic. Trigger `wp-cron.php` from the server's crontab for reliable timing.

**Server crontab — do not use WP-CLI.** `wp cron event run` crashes while Next3Cloud is active (section 4.7), so the jobs would silently never run. Use `curl`, one line **per site in the network**, sent straight to the server rather than through Cloudflare:

    */5 * * * * curl -sk --resolve staycolibri.com:443:127.0.0.1 -o /dev/null https://staycolibri.com/wp-cron.php
    */5 * * * * curl -sk --resolve 2uw.uk:443:127.0.0.1 -o /dev/null https://2uw.uk/wp-cron.php

- **One line per site:** WP-Cron is per site on multisite, and `DISABLE_WP_CRON` in the shared wp-config.php stops page-load cron for **every** site. Any site without its own line stops running cron. List the sites with `SELECT blog_id, domain, path FROM wp_blogs;` (sites with a path, e.g. `2uw.uk/site/`, need that path in the URL).
- **`--resolve … 127.0.0.1`** sends the request to the local web server, so Cloudflare cannot challenge, rate-limit or cache it. **`-k`** is needed because the origin certificate may not be publicly trusted (e.g. a Cloudflare origin certificate); the request never leaves the server.
- Add the lines with `crontab -e` as the `bitnami` user. Test each line by running it by hand with `-w "%{http_code}\n"` instead of `-o /dev/null` (expect `200`).
- Watch WP Crontrol for 10–15 minutes: no event should stay overdue. **Only then** add `define('DISABLE_WP_CRON', true);` to wp-config.php, and check WP Crontrol again.

---

## 7. Zapier Flows (retired)

**Retired October 2026.** All three Zaps were replaced by snippets that call Didit and Brevo directly:
| Former Zap | Replaced by |
|---|---|
| Zap 1 — New Host → ID Check + CRM (Didit session, verification email) | 3.27 (and 3.28 for Brevo) |
| Zap 2 — New Guest → CRM (Brevo, list 3) | 3.28 |
| Zap 3 — Didit ID → WordPress User Verification (relay to `/verify-user`) | 3.26 |

**Why:** fewer moving parts; the Didit webhook is now signed and checked (the Zapier catch hook was unauthenticated); hosts are matched by user ID and session, not email; Cuban ID numbers and full user meta no longer pass through a US processor; retries and failure alerts are under our control.

**Clean-up status:**
- [x] Zaps disabled
- [x] 3.1 `/verify-user` trashed
- [x] WordPress application password used by Zap 3 deleted
- [x] Delete the three Zaps in Zapier
- [x] Delete the Didit webhook destination that points at the Zapier catch hook
- [x] Deactivate the WordPress → Zapier registration webhook snippet (it sent email, account type and all user meta to Zapier after every registration)
- [ ] Clear Zapier task history, then consider cancelling the Zapier subscription

---

## 8. Go-Live Checklist

### Done
- [x] Tropipay API — 2FA resolved via server-side TOTP
- [x] Live Tropipay credentials in wp-config.php (rotated October 2026)
- [x] Booking cancellations — host cancellation flow (3.22), refunds and cancellations recorded in the Host Ledger
- [x] Append-only Host Ledger with backfill (3.15)
- [x] Zapier replaced by direct Didit and Brevo integrations (3.26–3.28), tested end to end on the Didit sandbox (automatic and manual-review approvals)
- [x] Verification email redesigned (3.27 V1.7 on the 3.31 Email Layout, Zunzún palette); design approved October 2026
- [x] Listing images: portrait photos rendering taller fixed by 3.33 (Next3Cloud size-URL bug); all images now load their proper sizes (October 2026)
- [x] Cookie audit and banner (October 2026): inventory taken; Pressidium configured opt-in with equal Accept/Reject; Order Attribution gated as Analytics; Mapbox storage blocked (3.35); footer settings link (3.36); privacy and cookie policy drafted

### Must do before launch
- [x] **Didit switched to live** (October 2026): live destination (`/wp-json/custom/v1/didit-webhook`, v3, `status.updated`); live key (Sessions write, IP-restricted), workflow and secret in wp-config.php; `SC_DIDIT_ENVIRONMENT = 'live'`; sandbox destination disabled
- [x] Live switch confirmed (October 2026): Try Webhook → `test_webhook_ok` (env live, v3 payload); a test host's link email created a session on the live workflow through the IP-restricted key
- [ ] Delete the live-test host user, its Brevo contact and its Didit session
- [ ] **Check the new verification email** (3.27 V1.7 + 3.31 V2.3): deploy 3.31 then 3.27, use **Send verification link** on a test host, and open it on a real phone, in the Gmail app and in Outlook. Confirm the real logo shows (CID attachment) and the colours and button look right
- [ ] **Didit workflow accepts the carné only:** the email says passports are not accepted. Confirm the live workflow only allows the Cuban carné de identidad as a document, otherwise a host can still submit a passport
- [ ] **First real Cuban host:** check (1) the `approved_verified` log line — which field the carné number came from (`personal_number` / `document_number`) — and that `hp_carne` holds the carné, not a passport number; (2) first name / both surnames split as expected, with accents, and title-cased correctly; (3) display name after approval; (4) Tropipay beneficiary created with those names and the first payout succeeds (Tropipay may compare with the account holder's name). Adjust 3.26 if needed
- [ ] Decide the public host name: HivePress Display Name is *Full name* (full legal name, both surnames, on listings and the host profile). *First name + initial* is more private
- [ ] **Review the Didit callback caveats** (noted October 2026): a just-approved host can briefly see `/verification-pending/` if Didit's redirect beats its webhook (consider polling `hp_verified` on that page); after 3.25 is deactivated there is no notice for unverified hosts outside `/submit-listing` (consider an `/account/` notice like the payment-setup one); hosts may return to the site logged out on the phone they verified with
- [ ] **Guest marketing consent:** add an opt-in to guest registration; only opted-in guests should go on Brevo list 3 (3.28 currently adds all guests, as Zap 2 did). The privacy policy already says marketing is opt-in
- [ ] **Fix Section 6 checkout window** (section 10) — first check the ledger for bookings showing "Eligible — awaiting payout · Tropipay payout record not yet created"
- [ ] **Clean start for the ledger:** remove test orders, payouts, cash balance entries and adjustments → `/?sc_ledger_admin=1` → **Reset ledger** → **Run one-time backfill** → confirm every host reconciles → **Mark ledger live**
- [ ] Enable **Versioning** on S3 bucket `staycolibri-ledgers`
- [ ] Check any order partially refunded before V3.5 (its refund split may be wrong — section 10)
- [ ] Deactivate coming soon snippet — removes redirect to /coming-soon/
- [ ] Deactivate unverified host lockdown snippet (3.25) — keep 3.2 Submit Listing Access Control active
- [ ] Deactivate contact page nav restriction snippet — restores full nav on /contact/
- [ ] Deactivate 3.11 Register Modal Warning (hosts-only message is wrong once guests can register)
- [ ] Payment setup page — add Tropipay confirmation checkbox
- [ ] **Mollie: turn test mode off** (WooCommerce → Settings → Payments → Mollie) and confirm the live API key is set — cash-host bookings are paid through Mollie (3.5)
- [ ] **Remove Jetpack and WooCommerce Tax** (decided October 2026). Jetpack 16.3.1 was active and connected to WordPress.com with 13 modules (incl. sync, stats, json-api, woocommerce-analytics, blaze, ai, backup) — sending site, user and order data to WordPress.com (US) — and needs `/xmlrpc.php`, which Cloudflare now blocks. Nothing depends on it: backups are done at server level, no site-verification codes are stored in Jetpack, the Contact Us page uses Contact Form 7 (0 Jetpack form submissions). Jetpack's "powering other plugins" warning lists WooCommerce (optional connection features only — WooPayments, WordPress.com sign-in for the Woo mobile app; not used) and WooCommerce Tax (`woocommerce-services`, removed with it). Steps:
  1. WooCommerce → Settings → Tax: confirm **Automated taxes** is off
  2. Jetpack → Dashboard → Manage your connection → **Disconnect** (before deleting, so the sync stops and the site leaves WordPress.com)
  3. Deactivate and delete **Jetpack**, then **WooCommerce Tax** (`woocommerce-services`)
  4. WordPress.com account: remove staycolibri.com from the sites list if still shown
  5. WP Crontrol: delete leftover `jetpack_*`, `jp_*`, `wc_connect_*` and `bmi_*` events
  6. Test the contact form and a test checkout
- [ ] **Google Search Console:** verify the site directly in Google's console (Domain property, DNS TXT record in Cloudflare) — not through a plugin

### Privacy and cookies (pre-launch)
- [ ] **Register with the ICO** (data protection fee) and add the registration reference to section 1 of the privacy policy
- [ ] **Publish the privacy and cookie policy at `/privacy/`**; give the "8. Cookies" heading the HTML anchor `cookies`; link it in the site footer
- [ ] Let logged-out visitors and unverified hosts reach `/privacy/` while 3.8 and 3.25 are active (both allowlists)
- [ ] **Didit model-training opt-out:** ask Didit support (not available in the console); once confirmed, delete the training sentence from section 4 of the policy
- [ ] **Legal review of the policy** (Fox Williams)
- [ ] **Import the final Pressidium settings** (`pressidium-settings-staycolibri-analytics.json`, revision 4), then re-add/check the two Blocked Scripts rows
- [ ] **Footer "Cookie settings" link:** add the menu item (`#cookie-settings`) and test that it opens the settings (3.36)
- [ ] **Run the cookie consent tests** in section 4.8 (no `sbjs_*` before consent or after Reject; cleared on withdrawal; Origin recorded after Accept)
- [ ] **Deploy and test 3.35** on the live site (no `mapbox.*` keys; `events.mapbox.com` still called; Submit Listing pin works)
- [ ] Log out the `host01` test session used during the cookie audit (its login cookies were copied into a chat)
- [ ] Delete the TEMP `customer-post-data` trap snippet if it was created

### Security (pre-launch)
- [ ] **WordPress admin 2FA** on every admin account; keep admin accounts to the minimum, strong unique passwords
- [ ] **IP restriction on wp-admin** (Cloudflare firewall rule)
- [x] **Payout detail change logging** — 3.4 V3.1: logged (time, who, where, IP) and emailed to admin and host on every change
- [ ] **Verified process for changing payout details** — define how a host's change request is verified before an admin edits them (e.g. re-run Didit or confirm from the registered email)
- [x] **Security review of all WPCode snippets** (October 2026): no hard-coded secrets; host name/carné/account-type lock moved server-side (3.10 V1.1); cancellation reasons restricted to the booking's host (3.22 V1.1); undocumented snippets documented (3.29, 3.30) and a Zapier-era endpoint deleted
- [ ] **Payout amount sanity check** — before the Tropipay API call (Section 7), cross-check `hp_amount` against the original order (`_sc_original_accommodation_amount` ± adjustments); abort and email admin on mismatch. Same cross-check for the cash batch (sum of entries vs orders)
- [ ] **Server migration:** add the new server's IP (IPv4 and IPv6 if used) to the live Didit API key allowlist **before** switching DNS
- [ ] **Secrets out of wp-config.php** — protects Tropipay/AWS/Didit/Brevo credentials against file-read-only attacks (vulnerable plugin). Was planned as AWS Secrets Manager; **re-decide if moving to OVH** (UK/EU-only providers)
- [ ] **Server access hardening** — SSH key-only, SSH restricted by IP, MFA on the hosting console, least-privilege cloud credentials
- [ ] **Restrict WPCode** to one trusted admin; review active snippets periodically
- [x] **Secrets exposed in the Zapier export:** WordPress application password deleted; the `X-Secret-Key` header is inert now 3.1 is trashed
- [x] **Brevo API key moved out of the 3.8 Coming Soon snippet** into `SC_BREVO_API_KEY` (V1.1); the old hard-coded key has expired
- [ ] Delete local copies of the Zapier export (contains dead credentials) and clear Zapier task history before cancelling (it holds host names/ID numbers from Zap 3 runs and user meta from the registration webhook)
- [x] **Cloudflare Turnstile on the HivePress login, register and password forms** (3.34 V1.1, October 2026): server-side check on the REST endpoints, widget in the modals; tested (curl rejected; modal login, wrong-then-right password, register, forgot password)
- [x] **Turnstile keys in wp-config.php** (`SC_TURNSTILE_SITE_KEY` / `SC_TURNSTILE_SECRET_KEY`, October 2026)
- [x] **XML-RPC blocked** in Cloudflare (WAF custom rule, `staycolibri.com` zone, October 2026)
- [x] **XML-RPC blocked on the `2uw.uk` zone** too (separate zone; same rule, October 2026)
- [x] **Cloudflare rate-limiting rule** on the HivePress login endpoint and `/wp-login.php` (5 per 10 s per IP, block 10 s; section 4.6)
- [x] **Next3Cloud bugs reported** (image sizes; WP-CLI fatal) — October 2026. Follow up; 3.33 stays active meanwhile
- [ ] **Mapbox token:** confirm the public token is URL-restricted to staycolibri.com, has public scopes only, and that billing alerts are set
- [ ] Keep `WP_DEBUG_LOG` off in production; never log tokens or the TOTP `securityCode`
- [ ] Secret scanning on the docs repository (gitleaks pre-commit and/or GitHub push protection)

### Recommended
- [x] Host and guest emails rebuilt on the 3.31 design (October 2026): 3.4 V3.2, 3.6 Section 6, 3.22 V1.2, 3.24 V2.4, 3.26 V1.0.5, with 3.31 V2.4 and 3.27 V1.8. Admin emails stay plain text
- [ ] **Deploy and test the rolled-out emails**: deploy 3.31 V2.4 first, then the others; trigger one of each (change a test host's Tropipay email, add and delete an adjustment, approve after review, cancel a test booking, run the payout cron on a test order) and check them on a phone and in Gmail/Outlook. 3.6 Section 6 is a section of a larger snippet: replace only that section and bump 3.6's own version note
- [x] HivePress's own emails wrapped in the 3.31 design by 3.32 V1.0 (English), with the host-cancellation email suppressed
- [ ] **Test the cash-host emails (3.6 V3.6)**: on a test cash order with checkout today, run the daily cron (WP Crontrol) and check the email's amount and delivery date; mark a test cash payout Completed and check the final calculation against the admin batch email
- [ ] **Test 3.32**: deploy it after 3.31, then trigger an email confirmation, a password reset and a test booking, check each looks right, and cancel a test booking as the host to confirm the guest gets only the 3.22 email, and that a host gets "Booking Confirmed" but no "Order Received" when a booking is paid (and check HivePress's own cancellation email is truly the `booking_cancel_vendor` event)
- [ ] **Server crontab for `wp-cron.php`** — optional. WP-Cron checked healthy in October 2026 (no overdue events; `sc_ledger_nightly` and both payout crons ran on schedule with all their handlers attached). Revisit if WP Crontrol shows overdue events. If done: one `curl` line per network site to 127.0.0.1, **not** `wp cron` (section 6); confirm in WP Crontrol, then `DISABLE_WP_CRON`
- [ ] **Harden `customer-post-data`** (section 10): find the code that sets it, then remove it or make it Secure + HttpOnly with only the fields needed
- [ ] Change the Mapbox account username (the public token reveals it) — nice to do
- [ ] Add Max Guests attribute to listings

### wp-config.php for live

    define('SC_TROPIPAY_CLIENT_ID',     '[IN PASSWORD MANAGER]');
    define('SC_TROPIPAY_CLIENT_SECRET', '[IN PASSWORD MANAGER]');
    define('SC_TROPIPAY_SANDBOX',       false);
    define('SC_TROPIPAY_ACCOUNT_ID',    80001);
    define('SC_TROPIPAY_TOTP_SECRET',   '[IN PASSWORD MANAGER]');

    define('SC_DIDIT_ENVIRONMENT',      'live');
    define('SC_DIDIT_API_KEY',          '[IN PASSWORD MANAGER]');   // live application — Sessions write, IP-restricted
    define('SC_DIDIT_WORKFLOW_ID',      '[IN PASSWORD MANAGER]');   // live application
    define('SC_DIDIT_WEBHOOK_SECRET',   '[IN PASSWORD MANAGER]');   // live destination

    define('SC_BREVO_API_KEY',          '[IN PASSWORD MANAGER]');
    define('SC_ADMIN_EMAIL',            'jonny@auno.uk');

    define('SC_TURNSTILE_SITE_KEY',     '[IN PASSWORD MANAGER]');   // same widget as the Turnstile plugin
    define('SC_TURNSTILE_SECRET_KEY',   '[IN PASSWORD MANAGER]');

    define('DISABLE_WP_CRON',           true);                      // only once the server crontab is in place

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

### Host Did Not Receive the Didit Link
1. Did they confirm their email? The link is sent only after confirmation (Users list "verified" column / `hp_email_verify_key` absent).
2. Edit User → "Stay Colibri — Didit verification" box: link last emailed, last error.
3. `/?sc_didit_log=1`: `session_link_sent` / `session_job_failed` for that user.
4. Ask them to check spam; then use **Send verification link**.

### Email Looks Wrong or the Logo Is Missing
See 3.31 troubleshooting: plain message = 3.31 inactive; text instead of logo = mailer not passing the embedded image or 3.27 inactive; "Undefined array key" notices = 3.31 and 3.27 versions do not match.

### Host Verified in Didit but Not on the Site
1. `/?sc_didit_log=1` — look for their user ID and the outcome (`ignored_session_mismatch`, `rejected_bad_signature`, `no_matching_host`…). See 3.26 troubleshooting.
2. No line at all: check the destination's Deliveries tab in Didit (Cloudflare 403, timeouts) and replay the event.
3. Check `SC_DIDIT_ENVIRONMENT` matches the Didit application in use.

### Brevo Contact Missing or Incomplete
1. Edit User → "Stay Colibri — Brevo" box: last synced, last error; **Sync to Brevo now**.
2. `/?sc_didit_log=1`: the `brevo_synced` line lists exactly which attributes were sent. See 3.28 troubleshooting.

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
- Check 3.0 Config and 3.18 Guest Pricing Display are active, both **Run Everywhere**
- Prices doubled: an old JS pricing snippet is still active alongside 3.18 — deactivate it
- See 3.18 Troubleshooting

### Listing Photos Different Heights / Images Loading Full Size
1. In the browser console on a listing, list the slide files and sizes (command in 3.33). Every slide should be an `-800x600` file at 800 × 600.
2. Originals loading: 3.33 inactive, or Next3Cloud's filter now runs after it — see 3.33 troubleshooting.
3. A broken image: that size was not offloaded — check Next3Cloud offloads all sizes.
4. To inspect one attachment's metadata and how each size resolves, recreate the temporary admin diagnostic described in 3.33 (delete it afterwards).

### HivePress Login / Register Says "Please complete the security check"
1. Is the Turnstile widget showing above the button? If not, see 3.34 troubleshooting (console check of the form markup).
2. Widget shows but every attempt fails: keys mismatched or hostname not allowed on the Turnstile widget in Cloudflare.
3. Cloudflare siteverify unreachable from the server: look for "Stay Colibri Turnstile: siteverify unreachable" in the PHP error log. 3.34 fails closed; admins can still use `/wp-login.php`.

### Cookies Set Before Consent / Banner Problems
1. Fresh incognito window, DevTools → Application → Cookies and Local Storage, before clicking the banner.
2. `sbjs_*` present: Blocked Scripts rows missing (section 4.8).
3. `mapbox.*` in Local Storage: 3.35 inactive or printing too late (3.35 troubleshooting).
4. A new cookie name appears: identify its source (Network tab search; for server-set cookies, the Set-Cookie column), classify it, add it to the Cookies tab and bump the revision.
5. Footer "Cookie settings" does nothing: 3.36 troubleshooting.

### WP-CLI Commands Crash
`Fatal error: … array_key_exists() … next3-offload/functions.php:1073` — a Next3Cloud bug (section 4.7). Nothing is wrong with the site. Use a temporary admin-only snippet instead of `wp eval`, WP Crontrol instead of `wp cron`, and `curl`/`php wp-cron.php` in the crontab.

### Coming Soon Redirect Not Working
- Check the coming soon snippet is active in WPCode
- Check the allowed slugs array includes any new pages that should be publicly accessible

### WP_DEBUG Already Defined Warning
wp-config.php has WP_DEBUG defined twice. Remove the duplicate definition. The warning is harmless but clutters the error log.

### Brute Force Login Attempts
The server has been receiving brute force attempts targeting WordPress login. In place (October 2026):
- Turnstile on `wp-login.php` and WordPress password reset (Turnstile plugin) and on HivePress's login/register/password REST endpoints (3.34, checked server-side)
- `/xmlrpc.php` blocked in Cloudflare
- Cloudflare rate limit on `/wp-login.php` and the HivePress login endpoint (section 4.6)

Still recommended:
- Two-factor authentication on admin accounts (checklist)
- Optionally Limit Login Attempts Reloaded — note that it only sees `wp-login.php`, not HivePress's REST login

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
| **Carné source field unconfirmed** | Sandbox data is fabricated. On a Cuban passport, `document_number` is the passport number; 3.26 prefers `personal_number`. Passports are no longer accepted (October 2026), so confirm the live Didit workflow only allows the carné; otherwise a passport number could land in `hp_carne` | Check the Didit workflow and the first real host (section 8) |
| **Didit webhook retries are short** (2 retries, ~5 min total) | A delivery dropped during an outage leaves an approved host unverified | Replay from Didit Deliveries. Possible improvement: nightly sweep that fetches the decision for unverified hosts with an open session |
| **Brevo contacts are keyed by email** | If a user changes their email, the next sync creates a second Brevo contact | Merge manually in Brevo if it happens |
| **Sign-up form does not validate phone numbers** | A mistyped number reaches `PHONE` but not `SMS` | Backlog: validate format/length per country on the form |
| **`?sc_rebuild_ledger` runs on GET without a nonce** (3.15) | A forged link opened by an admin could trigger a ledger re-render (no data change) | Accepted — a nonce would break the documented manual URL |
| **3.8 writes its template to a fixed temp file on every request** | Possible clash under concurrent load | Accepted — 3.8 is removed at launch |
| **Name title-casing is rule-based** (3.26) | Rare names may need correcting | Admin edits in wp-admin; originals kept in `sc_didit_*_name_raw` |
| **Tropipay beneficiary name** | 3.6 creates the beneficiary from `first_name` / `last_name`; Tropipay may compare with the account holder's name (accents, one or two surnames) | Check on the first real Tropipay host |
| **Email layout not yet tested in Outlook desktop or the Gmail app** (3.31) | Rounded corners and the button's hard edge may render flat in Outlook; layout should survive | Check with a test email (section 8) |
| **No cash-host email in months without a delivery** | Under $50 or a negative balance creates no payout record, so Section 10 sends nothing; the host sees the carried balance in the dashboard | Add a "no delivery this month" email to Section 8 if wanted |
| **HivePress emails are English only** | The site is English only, so hosts receive HivePress emails in English while snippet emails to hosts are in Spanish | Add per-recipient language to 3.32 later if wanted |
| **Next3Cloud returns the original file for every image size** | Listing crops, card images and thumbnails were all served at full size: portrait photos rendered taller in the listing gallery, and each listing loaded several MB of images | Fixed by 3.33 (October 2026); reported to Next3. Keep 3.33 active while Next3 is in use |
| **WP-CLI fatals while Next3Cloud is active** (`next3_offload_row()`, `functions.php:1073`) | Any `wp` command that loads WordPress with WooCommerce active crashes: no `wp eval`, `wp media regenerate`, `wp cron` | Reported to Next3. Use admin-only snippets, WP Crontrol, and `curl`/`php wp-cron.php` in the crontab |
| **Tropipay plugin warning** (`Undefined variable $hidden`, `wc-tropipay.php` line 322) | Harmless; clutters `debug.log` | Report to the plugin author or ignore |
| **3.34 fails closed** | If Cloudflare siteverify is unreachable or the Turnstile keys are missing, front-end login, registration and password reset are refused | Accepted (security over availability). Admins can use `/wp-login.php` |
| **Cloudflare free-plan rate limit matches on path only** | `/?rest_route=/hivepress/v1/users/login` is not rate-limited | Accepted — 3.34 still requires Turnstile on that URL form |
| **Shared IPs in Cuba (carrier-grade NAT)** | Per-IP limits can hit several real users at once | Rate limit kept at 5 per 10 s; do not lower it |
| **`customer-post-data` cookie** | Set by the server (Set-Cookie on the return from Mollie to `/checkout/`, host-only `staycolibri.com`, session). Holds the whole checkout form: guest name, email, phone, address, payment method, checkout nonce and (with consent) attribution fields. No Secure or HttpOnly flag, and sent with every request until the browser closes. Source not found: `grep -rln` over plugins/themes found nothing (try `grep -RIl` over the whole install, which follows Bitnami's symlinks, and `wp_14_posts` for WPCode). Listed as Necessary ("keeps your checkout details if a payment is cancelled or fails") | Find the source; then remove it if the WooCommerce session already refills the form, or make it Secure + HttpOnly and keep only needed fields |
| **Mapbox storage block relies on patching `Storage.prototype`** (3.35) | A future Mapbox version could store elsewhere; Mapbox's licence bars interfering with its billing/telemetry code (3.35 changes none, but Mapbox could object) | Re-test after HivePress or Mapbox updates |
| **Mapbox public token reveals the account username** (`tokenU` / token payload) | Personal name visible in page source | Nice to do: change the username or move to a company account; restrict the token by URL |
| **Order Attribution only records consenting guests** | Orders from guests who reject Analytics show Origin "Unknown" | Accepted — the cost of collecting it lawfully |

### Backlog — features to build
| Item | Notes |
|---|---|
| Quantity on extras | Let guests order an extra more than once (e.g. breakfast for several people on the same "Breakfast" extra). HivePress Bookings' quantity option (`hp_booking_enable_quantity`) applies to places, not extras — likely needs a custom quantity selector per extra, with the cart line priced accordingly and shown inclusive of the service fee |
| Centralise remittance fees | Move cash (11% + €3.00) and Tropipay (3.5% + €0.50) remittance fees into 3.0 Config as constants/functions, replacing the hardcoded values in 3.6 Payout System (5b, 6, 7, `sc_cash_batch_calc()`), 3.7 Dashboard, 3.24 Financial Tools and 3.15 Host Ledger. Touches the payout cron — build and test carefully; existing payout records keep their stored fees |
| Nightly prices on the guest calendar | Show each day's (inclusive) price in the date picker, as Airbnb does — daily prices (`hp_booking_enable_price`) are enabled, but the guest calendar shows no prices today |
| Status-aware `/verification-pending/` | Show "we're reviewing your verification" / "you're verified — publish your listing" from `sc_didit_status` and `hp_verified`, polling while open |
| Phone validation on sign-up | Validate number length and mobile range for the chosen country |
| Re-verify long-standing hosts | Didit keeps ID and face images for 10 years from the check; hosts active for more than about 4 years should re-verify so identity evidence covers 6 years after their last payout |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | September 2026 | Initial documentation |
| 1.1 | October 2026 | Added host cancellation flow (snippet 3.22), updated go-live checklist, added security risks section |
| 1.2 | October 2026 | V3.3 Payout Cron: refund hook (5b), payout lock (5c), adjustment handling in sections 7 and 8, TOTP live. V3.1 Dashboard Balance Block: payout lock notice, adjustment total display. Added snippets 3.23 (V1.1 Cash Balance Admin Tool) and 3.24 (V1.3 Admin Balance Adjustments). Updated Tropipay section: live credentials, TOTP, credential rotation note |
| 1.3 | October 2026 | **Payout System V3.5:** `sc_cash_batch_calc()` (fee on delivered amount, corrected delivery cap, cent-rounding guard); monthly hook now daily with once-per-month claim and catch-up; `sc_first_uk_working_day_of_month()` honours its argument; Section 9 re-syncs only the linked rollover; `batch_ref`/`payout_ref`; base amount for Tropipay adjustments; rate-fallback flag; 5b original-amount and pre-payout Tropipay refund fixes; `adj_id` on system adjustments. **Dashboard V3.9:** cash shows estimated USD delivery with breakdown modal; Tropipay shows total upcoming revenue; order-lookup fix. **Host Ledger V2.0.4:** append-only SQL ledger, status tracking with history, hash chain, triggers, nightly reconciliation and render, backfill, admin page, hyperlinks. **Admin Financial Tools V2.3:** merges V1.4 Financial Tools, V1.0 Reconcile and V1.1 Cash Balance; overview with reconciliation; ledger viewer and XLSX download; adjustments add/delete only on Edit Host with required deletion reason and host withdrawal email; Pending Cash Balance with protected deletes and no bulk clear. Credentials removed from this file. Added section 10 |
| 1.4 | October 2026 | Added 3.25 V1.0.2 Unverified Host Lockdown (pre-launch): unverified hosts redirected to /verification-pending/ site-wide; go-live checklist item to deactivate it |
| 1.5 | October 2026 | Added 3.0 V1.0 Config (service fee read from HivePress buyer commission; `SC_CONFIG`; `[sc_service_fee]` shortcode). Former 3.18–3.21 replaced by 3.18 V3.2 Guest Pricing Display: listing and daily prices made inclusive server-side via the HivePress field display filter, extras made inclusive in the booking form, JS limited to the fee note, modal and booking breakdown. Added Backlog (extras quantity, Cloudflare Turnstile on login, guest calendar prices) |
| 1.6 | October 2026 | Go-live checklist: full pre-launch security list; Backlog: centralise remittance fees in Config |
| 1.7 | October 2026 | **Zapier retired.** Added 3.26 V1.0.2 Didit Verification Webhook (signed, session-matched, all statuses, admin alerts, approval email after manual review, event log), 3.27 V1.2 Didit Verification Session + Email (sent after email confirmation, retries, resend tool, HivePress welcome email suppressed for hosts), 3.28 V1.3 Brevo Contact Sync (guests and hosts, ID_VERIFIED Boolean, SMS + PHONE with fallback). 3.1 retired. New user meta, option, transient, single events and admin tools documented. Didit sandbox/live setup, Brevo attributes, onboarding flow, go-live and security items (live Didit switch, carné field check, guest consent, Zapier clean-up incl. the WordPress → Zapier webhook snippet) |
| 1.8 | October 2026 | 3.8 Coming Soon V1.1: email confirmation, password reset and login links work for logged-out visitors (HivePress routes allowed by URL path); Brevo key from `SC_BREVO_API_KEY`; `ACCOUNT_TYPE` sent as 3. Zapier clean-up completed. Security: Brevo key out of 3.8 done; Zapier export/task-history clean-up added |
| 1.9 | October 2026 | **Snippet security review.** 3.10 V1.1 server-side lock on host name/carné and all users' account type; 3.22 V1.1 cancellation reason restricted to the booking's host (guest cancellations no longer trigger host-cancellation emails); 3.4 V3.1 admin save fix + payout-detail change log with admin and host emails; 3.25 V1.0.3 (priority 1; V1.0.2 had not been deployed); 3.26 V1.0.4 Spanish-aware title-case names, Didit originals kept, display name set by HivePress's own setting; 3.29 Listing Photo Limit (30, was undocumented at 100); 3.30 Show User ID shortcode documented; full snippet inventory with WPCode title format; 3.11 deactivate at launch; first-real-host checks, public display name decision and accepted risks added |
| 1.10 | October 2026 | **Didit live.** Live application keys, workflow and webhook destination in use; API key scoped to Sessions write and IP-restricted (static IP, IPv6 and server-migration notes); sandbox destination disabled; checklist updated |
| 1.11 | October 2026 | **Email redesign.** New snippet 3.31 V2.3 Email Layout: shared HTML design for host and guest emails (full-width colour bands, Zunzún palette with contrast values, Trebuchet MS, hard-edged button, coloured step tiles, role-named tokens in `sc_email_tokens()`, `sc_email_send()` with the logo embedded as `cid:sc-logo`, design rules incl. the four banned AI-template patterns); admin emails stay plain text. 3.27 V1.7 builds its email with 3.31 (V1.3 first redesign, V1.4 on 3.31 with a dash-free subject, V1.5 palette-driven link colour, V1.6 carné only, V1.7 700 px logo trimmed to 606 × 200 and shown at 200 px), with a plain fallback if 3.31 is inactive. Checklist: test the email in real clients, confirm the Didit workflow accepts the carné only (confirmed October 2026: passports removed), roll the design out to the other host and guest emails (3.4, 3.6, 3.22, 3.24, 3.26). Known issues: email layout untested in Outlook/Gmail, logo data lives in 3.27, HivePress's own emails not styled |
| 1.12 | October 2026 | **Email rollout.** 3.31 V2.4 holds the logo and adds amount and facts blocks, Spanish dates, and an English option for footer and sign-off. Hosts get Spanish emails, guests English. Host and guest emails rebuilt on it: 3.4 V3.2 (host change notice), 3.6 Section 6 (payout on its way, Spanish), 3.22 V1.2 (guest cancellation, English), 3.24 V2.4 (adjustment and withdrawn, Spanish), 3.26 V1.0.5 (host approval). 3.27 V1.8 no longer holds the logo. Each snippet falls back to its old plain email if 3.31 is inactive. Admin emails unchanged. |
| 1.13 | October 2026 | **HivePress emails.** New snippet 3.32 V1.0 HivePress Email Wrapper wraps all HivePress emails in the 3.31 design (English), turns the first bare link into a button, and suppresses HivePress's host-cancellation email (3.22 sends the detailed one). 3.31 accepts an empty hero text. |
| 1.14 | October 2026 | 3.32 V1.1: HivePress's "Order Received" email to hosts is suppressed (hosts already get "Booking Confirmed"). |
| 1.15 | October 2026 | 3.32 V1.2: HivePress "Order Completed" not sent to Tropipay hosts (cash hosts still get it). Noted that cash hosts get no snippet email at checkout or at the monthly batch. |
| 1.16 | October 2026 | **Cash-host emails.** 3.6 V3.6: Section 6b checkout-day email to cash hosts (EUR amount and delivery date) and Section 10 cash delivery confirmation with the final calculation when a cash payout is marked Completed (both Spanish, 3.31 design). 3.31 V2.5 adds `sc_email_ledger()` and `sc_email_month_es()`. 3.32 V1.3: "Order Completed" not sent to any host, "Payout Completed" not sent to cash hosts. |
| 1.17 | October 2026 | 3.31 V2.6: email font changed from Trebuchet MS to Verdana (all emails built with 3.31, including wrapped HivePress emails). |
| 1.18 | October 2026 | **Images and login security.** New 3.33 V1.0 Image Size URL Fix: Next3Cloud's `wp_get_attachment_image_src` filter returned the original file (at 150 × 150) for every size, so listing galleries showed portrait photos taller and every image loaded full size; 3.33 rebuilds size URLs from the metadata. New 3.34 V1.1 Turnstile on HivePress Account Forms: server-side Turnstile check on HivePress's login, register, forgot-password and reset-password REST endpoints (fails closed) plus the widget in the modals (V1.0 matched `action` instead of `data-action`; V1.1 also renders on modal open). New sections 4.6 Cloudflare (Turnstile, XML-RPC block, free-plan rate limit kept at 5 per 10 s because of Cuban carrier-grade NAT) and 4.7 Next3Cloud (two bugs reported: image sizes; WP-CLI fatal). Image sizes table added to section 2. Crontab must use `curl`/`php wp-cron.php`, not WP-CLI. Troubleshooting, checklist and known issues updated; Turnstile removed from the Backlog. |
| 1.19 | October 2026 | **Cookie consent and privacy.** New section 4.8 Pressidium Cookie Consent: opt-in banner with equal Accept/Reject, cookie inventory, Order Attribution (`sbjs_*`) gated as Analytics via Blocked Scripts, settings file, revision rule, tests; privacy and cookie policy at `/privacy/` (controller details, named vs described suppliers, retention). New 3.35 V1.0.1 Mapbox Storage Block (stops `mapbox.*` localStorage identifiers; V1.0 failed WPCode's parser) and 3.36 V1.0 Cookie Settings Menu Link (footer `#cookie-settings` opens the settings; floating button off). Didit (4.2): Spanish contracting entity, retention set to 10 years with rationale, consent screen confirmed, model-training opt-out pending. Transpaygo named as cash delivery partner (internal docs only). Lightsail region London; SES `eu-west-1`. 3.8 and 3.25 notes: allow `/privacy/`. New checklist section "Privacy and cookies"; known issues: `customer-post-data`, Mapbox block and token, attribution coverage; backlog: re-verify long-standing hosts. |

Documentation last updated October 2026. Update this file whenever snippets are added, modified or removed.

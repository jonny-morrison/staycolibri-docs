# Stay Colibri — Site Documentation
**Last Updated:** September 2026  
**Platform:** WordPress Multisite (HivePress/RentalHive)  
**URL:** https://staycolibri.com  
**Subsite prefix:** `wp_14_`  
**Server:** Bitnami AWS Lightsail (Ubuntu)

---

## Table of Contents
1. [System Architecture](#1-system-architecture)
2. [WordPress Data Structure](#2-wordpress-data-structure)
3. [WPCode Snippets](#3-wpcode-snippets)
4. [Third Party Integrations](#4-third-party-integrations)
5. [AWS S3 Ledger System](#5-aws-s3-ledger-system)
6. [Cron Jobs](#6-cron-jobs)
7. [Zapier Flows](#7-zapier-flows)
8. [Go-Live Checklist](#8-go-live-checklist)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. System Architecture

### Stack
- **WordPress Multisite** — main network at `2uw.uk`, subsite at `staycolibri.com` (blog ID 14)
- **HivePress** — listing/booking framework
- **RentalHive** — HivePress extension for rental bookings
- **WooCommerce** — payment processing
- **WPCode** — all custom PHP/JS snippets
- **Cloudflare** — DNS and proxy
- **WP Mail SMTP** — email via Zoho SMTP
- **Next3Cloud** — media files offloaded to Peasoup Cloud S3
- **PhpSpreadsheet** — installed at `/opt/bitnami/wordpress/vendor/autoload.php`

### Key URLs
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

### Admin Access
- WordPress Admin: https://staycolibri.com/wp-admin
- phpMyAdmin: via Bitnami server local access
- WPCode Snippets: WordPress Admin → Code Snippets

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

### Post Types
| Post Type | Description |
|---|---|
| `hp_listing` | Property listings |
| `hp_booking` | Booking records |
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
| `sc_cash_balance_entries` | serialised array | Per-booking cash balance entries |
| `sc_tropipay_deposit_account_id` | integer | Tropipay beneficiary ID for this host |

### Order Meta Keys
| Meta Key | Description |
|---|---|
| `_sc_host_user_id` | WordPress user ID of the host |
| `_sc_host_payment_model` | Payment model at time of booking |
| `_sc_accommodation_amount` | Accommodation amount (excl. service fee) |
| `_sc_checkout_date` | Guest checkout timestamp |
| `_sc_payout_processed` | `0` = not yet paid, `1` = payout created |
| `_sc_vendor_id` | Vendor post ID |
| `_sc_payout_post_id` | ID of associated hp_payout record |

### Payout Post Meta Keys
| Meta Key | Description |
|---|---|
| `hp_amount` | Net payout amount (after Tropipay fee) |
| `sc_accommodation_amt` | Gross accommodation amount |
| `sc_payout_method` | `Tropipay Transfer` or `USD Cash Delivery` |
| `sc_payment_date` | Unix timestamp of scheduled payment date |
| `sc_order_id` | Associated WooCommerce order ID |
| `sc_checkout_date` | Guest checkout timestamp |
| `sc_tropipay_fee` | Tropipay fee deducted |
| `sc_tropipay_transfer_id` | Tropipay transfer ID (after successful payout) |
| `sc_tropipay_transfer_reference` | Tropipay transfer reference |
| `sc_total_eur` | Total EUR for cash batch |
| `sc_payable_usd` | USD amount payable (multiple of $50) |
| `sc_remainder_usd` | USD rollover to next month |
| `sc_eur_usd_rate` | EUR/USD rate at time of batch |

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

### 3.6 V3 Payout Cron (MAIN CRON SNIPPET)
**Type:** PHP  
**This is the most critical snippet on the site.**

Contains 9 sections:

**Section 1 — woocommerce_payment_complete**
Fires when a WooCommerce order is paid. Stores order meta:
- `_sc_host_user_id`, `_sc_accommodation_amount`, `_sc_checkout_date`
- `_sc_host_payment_model`, `_sc_payout_processed = 0`
- For cash hosts: adds entry to `sc_cash_balance_entries`

**Section 2 — Cron schedule registration**
Registers `sc_daily` (daily at 6am UTC) and `sc_monthly` (monthly) cron schedules.

**Section 3 — sc_get_uk_holidays()**
Fetches UK bank holidays from https://www.gov.uk/bank-holidays.json
Cached for 24 hours via WordPress transient `sc_uk_bank_holidays`.

**Section 4 — sc_is_uk_working_day($timestamp)**
Returns true if the given timestamp is a UK working day (not weekend, not bank holiday).

**Section 5 — sc_add_uk_working_days($timestamp, $days)**
Adds N UK working days to a timestamp.

**Section 6 — sc_first_uk_working_day_of_month($timestamp)**
Returns the timestamp of the first UK working day of the month containing the given timestamp.

**Section 7 — Daily cron: Tropipay payout creation**
Fires daily. Finds orders where checkout date = today or tomorrow. For each:
- Creates `hp_payout` post (status: pending)
- Calculates net amount (accommodation minus 3.5% minus 0.50 EUR)
- Sets `sc_payment_date` = 5 UK working days after checkout
- Sends host email in Spanish with expected payment date

**Section 7 continued — Tropipay API transfer**
Finds pending Tropipay payouts where `sc_payment_date` <= today. For each:
- Gets Tropipay access token
- Finds or creates beneficiary by email
- Executes transfer via POST /operations/payout
- On success: marks payout as publish, stores transfer ID
- On failure: emails jonny@auno.uk with details

IMPORTANT — Sandbox/Live switch: Controlled by `SC_TROPIPAY_SANDBOX` in wp-config.php

**Section 8 — Monthly cron: Cash batch processing**
Fires on first UK working day of each month. For each cash host:
- Sums eligible balance entries (checkout date < today, status = pending)
- Calculates payable USD (rounds down to $50 multiple)
- Deducts fee: 11% plus 3.00 EUR
- Creates `hp_payout` post
- Emails jonny@auno.uk with batch details
- Updates `sc_cash_balance_entries` — marks paid entries, creates rollover entry

**Section 9 — save_post hook**
When admin publishes a cash payout record, recalculates remainder and updates `sc_cash_balance_entries`.

**Troubleshooting:**
- If Tropipay payouts are not firing, check `SC_TROPIPAY_SANDBOX` is correct in wp-config.php
- Check payout records exist with post_status = pending and sc_payout_method = Tropipay Transfer
- Check `sc_payment_date` is <= today's timestamp
- UK bank holidays are auto-fetched — if the API is down, the transient `sc_uk_bank_holidays` may be empty
- To manually trigger: add a temporary snippet with `do_action('sc_daily_payout_cron')`

---

### 3.7 V3 Dashboard Balance Block
**Type:** PHP  
**Hook:** `wp_footer` on `/account/vendor/dashboard/`  
**What it does:**
- Tropipay hosts: Shows Fondos Pendientes = sum of pending payout records (net of fees) plus unpaid orders (net of fees)
- Cash hosts: Shows Saldo del Proximo Pago = eligible balance entries plus next payout date
- Hides Request a Payout button for all hosts
- Replaces it with an explanatory note in Spanish

**Troubleshooting:**
- If balance shows wrong amount, check `sc_cash_balance_entries` user meta
- If dashboard crashes with Call to undefined function sc_is_uk_working_day(), the payout cron snippet failed to load before the dashboard snippet — check WPCode snippet priorities

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

### 3.15 Host Ledger System (AWS S3 XLSX)
**Type:** PHP  
**What it does:** Generates and maintains one XLSX ledger file per host in AWS S3 bucket `staycolibri-ledgers`.

**S3 key format:** `ledgers/{user_id}_{slug}.xlsx`

**Triggers:**
- woocommerce_payment_complete → rebuilds host ledger
- sc_daily_payout_cron → rebuilds Tropipay host ledgers
- sc_monthly_payout_cron → rebuilds cash host ledgers
- save_post on published cash payout → rebuilds ledger
- Manual: ?sc_rebuild_ledger=1&host_id=13 or host_id=all

**Ledger structure:**
- Rows 1-10: Host info header (dark background)
- Row 11: Column headers (amber background)
- Row 12+: Booking rows sorted by checkout date

**Columns:**
| Column | Description |
|---|---|
| A | Date |
| B | Order ID |
| C | Property |
| D | Check In |
| E | Check Out |
| F | Nights |
| G | Payout Method |
| H | Guest Total Paid (EUR) |
| I | SC Commission (EUR) |
| J | Gross Booking (EUR) |
| K | Host Balance (EUR) — Excel formula |
| L | Payout Fee (EUR) |
| M | Batch Fixed Fee (EUR) |
| N | Host Payout Balance (EUR) — Excel formula |
| O | Notes |

**Row styling:**
- Future bookings (checkout > today): grey text
- Past bookings: alternating white/light grey
- Paid Tropipay: green tint
- Batch rows: amber, bold

**Troubleshooting:**
- If ledger rebuild fails, check AWS credentials in wp-config.php
- Check PhpSpreadsheet is installed: `ls /opt/bitnami/wordpress/vendor/phpoffice/`
- Check S3 bucket `staycolibri-ledgers` exists and IAM user has write access
- Run manual rebuild: ?sc_rebuild_ledger=1&host_id=all

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

## 4. Third Party Integrations

### 4.1 Tropipay
**Purpose:** Automated payouts to hosts with Tropipay accounts  
**API Version:** v3  
**Sandbox:** https://sandbox.tropipay.me/api/v3  
**Live:** https://www.tropipay.com/api/v3  

**wp-config.php constants:**

    define('SC_TROPIPAY_CLIENT_ID',     '...');
    define('SC_TROPIPAY_CLIENT_SECRET', '...');
    define('SC_TROPIPAY_SANDBOX',       true);   // false for live
    define('SC_TROPIPAY_ACCOUNT_ID',    793);    // 80001 for live

**Payout flow:**
1. Daily cron gets access token via POST /access/token
2. Finds or creates beneficiary via GET /deposit_accounts + POST /deposit_accounts
3. Executes payout via POST /operations/payout with securityCode
4. Stores `sc_tropipay_deposit_account_id` in user meta to avoid recreating beneficiary

**PENDING:** Tropipay support request to disable 2FA requirement for automated payouts.
Currently using securityCode: 123456 in sandbox. Live behaviour TBC.

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
**Purpose:** Live EUR/USD exchange rate for cash batch processing  
**URL:** https://api.frankfurter.app/latest?from=EUR&to=USD  
**Used in:** Monthly cron (section 8) and ledger batch rows  
**Fallback:** Rate defaults to 1.08 if API is unavailable

---

### 4.5 UK Bank Holidays API
**Purpose:** Determines UK working days for payout scheduling  
**URL:** https://www.gov.uk/bank-holidays.json  
**Cached:** 24 hours via WordPress transient `sc_uk_bank_holidays`  
**Used in:** sc_is_uk_working_day(), sc_add_uk_working_days(), sc_first_uk_working_day_of_month()

---

## 5. AWS S3 Ledger System

### Credentials (wp-config.php)

    define('SC_AWS_ACCESS_KEY_ID',     '...');
    define('SC_AWS_SECRET_ACCESS_KEY', '...');
    define('SC_AWS_BUCKET',            'staycolibri-ledgers');
    define('SC_AWS_REGION',            'eu-west-2');

### IAM User
- Username: staycolibri-ledger-writer
- Policy: AmazonS3FullAccess
- Bucket: staycolibri-ledgers (private, all public access blocked)

### File Format
- One XLSX file per host
- Key: ledgers/{user_id}_{slug}.xlsx
- Generated using PhpSpreadsheet

### Manual Rebuild

    https://staycolibri.com/?sc_rebuild_ledger=1&host_id=13
    https://staycolibri.com/?sc_rebuild_ledger=1&host_id=all

Must be logged in as admin.

---

## 6. Cron Jobs

### Registered Schedules
| Schedule | Interval | Hook |
|---|---|---|
| sc_daily | Daily at 6am UTC | sc_daily_payout_cron |
| sc_monthly | Monthly (first of month) | sc_monthly_payout_cron |

### Daily Cron (sc_daily_payout_cron)
Priority order:
1. Priority 10 — Creates Tropipay payout records for checkouts today/tomorrow
2. Priority 40 — Processes pending Tropipay transfers via API + rebuilds ledgers

### Monthly Cron (sc_monthly_payout_cron)
Only fires on first UK working day of the month. Priority order:
1. Priority 10 — Processes cash batch payments
2. Priority 30 — Rebuilds cash host ledgers

### Checking Cron Status
Install WP Crontrol plugin temporarily to view scheduled events.

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

### Must Do Before Launch
- [ ] Tropipay API — resolve 2FA for automated payouts with Tropipay support
- [ ] Switch to live Tropipay credentials in wp-config.php
- [ ] Didit verification — live testing with real Didit account
- [ ] Booking cancellations — implement ledger cancellation rows and refund flow
- [ ] Deactivate coming soon snippet — removes redirect to /coming-soon/
- [ ] Deactivate contact page nav restriction snippet — restores full nav on /contact/
- [ ] Payment setup page — add Tropipay confirmation checkbox

### wp-config.php Switch for Live

    define('SC_TROPIPAY_CLIENT_ID',     'LIVE_CLIENT_ID_IN_PASSWORD_MANAGER');
    define('SC_TROPIPAY_CLIENT_SECRET', 'LIVE_CLIENT_SECRET_IN_PASSWORD_MANAGER');
    define('SC_TROPIPAY_SANDBOX',       false);
    define('SC_TROPIPAY_ACCOUNT_ID',    80001);

---

## 9. Troubleshooting

### Site Gives Critical Error
1. Enable debug logging in wp-config.php:

    define('WP_DEBUG',         true);
    define('WP_DEBUG_LOG',     true);
    define('WP_DEBUG_DISPLAY', false);

2. Check log: tail -100 /opt/bitnami/wordpress/wp-content/debug.log
3. Disable snippets one by one in WPCode until error stops

### Host Dashboard Crashes
Most likely cause: a WPCode snippet calling a function defined in another snippet that has not loaded yet.
- Check for Call to undefined function in debug log
- Wrap the call in function_exists() check
- Adjust snippet load priority in WPCode

### Tropipay Payouts Not Processing
1. Check SC_TROPIPAY_SANDBOX value in wp-config.php
2. Check payout records exist: WordPress Admin → HivePress → Payouts
3. Check sc_payment_date <= today's timestamp in postmeta
4. Check host has sc_tropipay_email set in user meta
5. Check sc_tropipay_deposit_account_id — if empty, beneficiary creation may be failing
6. Add temporary debug snippet and visit ?sc_test_tropipay=1

### Ledger Not Updating
1. Check AWS credentials in wp-config.php
2. Run manual rebuild: ?sc_rebuild_ledger=1&host_id={user_id}
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

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | September 2026 | Initial documentation |
| 1.1 | October 2026 | Added host cancellation flow (snippet 3.22), updated go-live checklist, added security risks section |

---

## 3.22 Host Cancellation Flow
**Type:** PHP + JS
**Hook:** `trashed_post` on `hp_booking`, `wp_footer` on `/account/bookings/`, `wp_ajax_sc_store_cancel_reason`

**What it does:**

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
Fires when HivePress trashes the booking post. Reads the stored reason from the transient. If no reason is found (e.g. admin-initiated cancellation) the hook exits silently.

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
- HivePress may also send its own cancellation notification — this is in addition to it

**Troubleshooting:**
- If admin email is not received, check the transient was stored — add `error_log` to the AJAX handler to confirm
- If check-in/check-out show as Unknown, check `hp_start_time` and `hp_end_time` exist on the booking post in phpMyAdmin
- If the reason dropdown does not appear, check the snippet is firing on `/account/bookings/` URLs
- If the form submits without a reason, check the JS event listener is not being blocked by another script

Documentation last updated October 2026. Update this file whenever snippets are added, modified or removed.

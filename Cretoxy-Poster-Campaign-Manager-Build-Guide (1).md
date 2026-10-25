# Cretoxy Poster Campaign Manager — Complete Build Guide

**How to use this file:** start at Section 0 and go to the end in order. Every step is numbered. Nothing later asks you to return to an earlier step. Every command says where to run it.

**Legend**
- 📋 = a prompt to paste into your AI coding agent (opencode or Antigravity).
- ⚠️ NEEDS YOUR DECISION = something not settled yet. The guide uses the marked recommended default so you can keep moving. All such items are collected in Section 9.
- **(PROPOSED)** = a name or value that was never fixed. It is suggested so the guide is runnable. Change it if you want, but then change it everywhere.
- Prices, package names, durations and inclusions are copied exactly from your **Local Poster Advertising Price Chart**.

**This guide is for the poster service only.**

---

## 0. Project Summary

### 0.1 What the app is
**Cretoxy Poster Campaign Manager** is a web app for Cretoxy's local poster-advertising service. A business (the client) buys a poster package: printed A4 colour posters, a two-person installation team, approved placement locations, photo proof and a location list. The app manages clients, vendors (printers, installers, designers, venue partners), the four poster packages, campaigns, **placements** (every approved location where posters go, with permission status), tracked QR codes and links per zone, creative approvals, proof photos, placement checks, reports, GST invoices, client payments (tracked, not processed), vendor costs and profit. It has **two roles only: Admin and Client**. There is **no public signup**; Admin creates every client account. The Admin uploads **all** proof photos (there is no installer app and no installer login).

**It lives at https://cx.cretoxy.cloud** (confirmed: the app and the QR redirect links share this one address; see Section 9, item 1).

**Final stack:**
- Frontend hosting: **Cloudflare Pages**
- Backend / database / auth / storage: **Supabase**
- Code hosting: **GitHub**
- AI coding tool: **opencode or Antigravity**
- QR scan redirect + logging: a **Cloudflare Worker** on the path `/c/*` of `cx.cretoxy.cloud` (`/c/CLIENTCODE/CAMPAIGNCODE/ZONECODE`)
- Email notifications: email only, **no WhatsApp automation** (a manual "copy message" button is used for WhatsApp)
- Payments: accepted over WhatsApp by wire transfer; the app only records them

**Stack choices made for this guide because nothing fixed them (⚠️ see Section 9):** React + Vite + TypeScript for the frontend, npm as package manager, Supabase Storage for files, Resend for email, PDFs generated in the browser.

### 0.2 External services: used and not used
| Service | Status |
|---|---|
| Cloudflare Pages, Pages Functions, Workers | USED |
| Supabase (Postgres, Auth, Storage, RLS) | USED |
| GitHub | USED |
| opencode or Antigravity | USED to write the code |
| Resend (email) | USED, recommended, ⚠️ not confirmed |
| Cloudflare Cron Trigger (daily alert email Worker) | OPTIONAL (Prompt #14) |
| Cloudflare R2 | NOT used in the main path (⚠️ option in Section 9) |
| WhatsApp automation / WhatsApp Business API | NOT used |
| Public signup | NOT used |
| Installer app / installer login | NOT used |
| Lead scraping | NOT used |
| Payment gateway | NOT used (payments are tracked only) |

### 0.3 Final decisions
| Topic | FINAL decision | Source |
|---|---|---|
| Packages | **Local Reach** (100 posters, ₹5,999, 7 days), **Area Campaign** (250 posters, ₹13,999, 14 days), **High-Visibility** (500 posters, ₹24,999, 30 days), **Mass Local** (1,000 posters, ₹44,999, 30 days). The chart marks "30-day options by quote" on all four packages. Any other size, length or 30-day option is a **Custom quote**. | Price chart |
| Package inclusions | Local Reach: A4 colour posters, two-person installation team, approved placement locations, photo proof + location list. Area Campaign: printing + installation, multiple approved locations, photo verification, campaign tracking code. High-Visibility: two-person installation team, selected zones / local areas, scheduled placement checks, photo evidence + report. Mass Local: large-scale distribution, multiple approved zones, scheduled placement checks, final campaign report. Every package: 30-day options by quote. | Price chart |
| Price shown in the app | Package price is **before GST** (confirmed: the chart prices exclude GST). GST 18% is added as a separate line, editable per campaign. | Chart + earlier GST rule |
| Placement fees | Approved placement / venue fees may change the final quote. They are a **separate line item** on the quote and invoice (`placement_fee_charged`, default ₹0). | Price chart |
| Where posters may go | Only authorised noticeboards, approved ad spaces, partner-shop windows, or private walls with permission. **Never** public walls or utility boxes (can trigger removal or penalties). The app enforces this: a placement cannot be marked Installed until its permission is Approved. | Price chart |
| Stages | Pending, Design, Printing, Installation, Live, Completed, Cancelled | Kept from the earlier spec |
| Roles | Admin and Client only; no public signup | Kept |
| Proof photos | Admin uploads only | Kept |
| Payments / invoices | Inside the app as a tracker; money arrives over WhatsApp by wire transfer | Kept |
| WhatsApp | No automation; manual copy-message button | Kept |
| Themes | White, Black (green), Ocean (blue) using the colors in 0.6 | Kept |
| Leads | Client types in lead counts (per reporting period, see 0.7) | Kept, adapted for short campaigns |
| Build tool | opencode or Antigravity | Kept |
| Build scope | Everything in the feature list (Section 0.7) | Kept |
| Minimum order | Warning below 100 posters (the smallest package); (PROPOSED, not in the chart) | New |
| Poster prices | Only the chart prices above apply. Any other poster price list with different numbers is discarded and must not be used. | Price chart |

### 0.4 Money rules and formulas
- `package unit rate = package price ÷ poster count` (for display only: Local Reach about ₹60, Area Campaign about ₹56, High-Visibility about ₹50, Mass Local about ₹45 per poster).
- `revenue = price + placement_fee_charged` (both before GST).
- `gst_amount = revenue × gst_percent / 100`; `invoice_total = revenue + gst_amount`.
- `planned cost = print_cost_per_poster × poster_count + planned_labour_cost + planned_transport_cost + planned_placement_fees + planned_design_cost`.
- `actual cost = sum of all rows in costs for the campaign`.
- `profit (planned) = revenue − planned cost`; `profit (actual) = revenue − actual cost`; `margin % = profit ÷ revenue × 100`. Profit is before GST and is called **contribution**: it is before sales expenses, general overhead, tax, reprints, failed placements, damage and extra permission costs.
- `outstanding = invoice_total − payments received`.
- **Printing benchmark to verify:** working estimate ₹8 per poster at low volume, ₹6 per poster in bulk. Confirm paper, size, colour, gumming, GST and delivery with the printer. The app defaults `print_cost_per_poster` to ₹6 for 500 posters or more and ₹8 below 500 (the 500 threshold is PROPOSED); it is editable.
- **Cost check example from the chart (1,000-poster campaign):**

| Item | Calculation | Amount |
|---|---|---|
| Printing | 1,000 × ₹6 | ₹6,000 |
| Installation labour | 2 workers × 3 days × ₹1,000/day | ₹6,000 |
| Transport + supplies | allowance | ₹2,000 |
| Approved placement fees | allowance | ₹5,000 |
| Design, coordination + inspections | allowance | ₹2,000 |
| **Total estimated cost** | | **₹21,000** |

  Customer price ₹44,999 − ₹21,000 = **₹23,999** contribution, about 53% of the selling price. These are example assumptions; replace them with real supplier and worker quotes. Worker pay, productivity, transport, print quality, venue fees and permissions can all change the real cost. Keep permission fees as a separate line item if uncertain.
- Rules of thumb: collect the client advance first; pay installers only after proof is verified; vendors pay nothing upfront from the client's side.
- GST registration: mandatory above ₹20 lakh yearly turnover (₹10 lakh in special-category states); inter-state supply can trigger it earlier. Check with an accountant.

### 0.5 Roles and access rules
**Admin:** full access to everything. Creates clients, vendors, campaigns, placements, invoices, costs, reports. Uploads proof photos, creatives, documents. Generates QR codes, invoices, reports and client accounts. Sees costs and profit.

**Client:** logs in only with credentials created by Admin and can change the password after first login. Sees only their own campaigns, location list (without contact phones or venue fees), verified proofs, QR analytics, invoices, payments, reports, documents and notifications. Can approve or reject creative mockups, download invoices and reports, raise support queries, and type lead counts. Cannot see other clients, other campaigns, vendors, costs, venue fees, permission papers or profit.

### 0.6 Themes (exact values)
Three themes, toggleable, saved per user (`users.theme`).

| Token (CSS variable) | White | Black (green) | Ocean (blue) |
|---|---|---|---|
| `--bg` page background | #FFFFFF | #0A0A0A | #0A0A0A |
| `--surface` input / raised surface | #FFFFFF | #141414 | #141414 |
| `--border` | #E5E5E5 | #2A2A2A | #2A2A2A |
| `--text` | #0A0A0A | #EDEDED | #EDEDED |
| `--muted` | #6B6B6B | #888888 | #888888 |
| `--accent` | #0A0A0A | #4ADE80 | #22D3EE |
| `--accent-dim` hover/pressed | #333333 | #3AA962 | #1CA1B5 |
| `--on-accent` text on accent | #FFFFFF | #0A0A0A | #0A0A0A |

Shared status colors: `--success` #22C55E, `--warning` #F59E0B, `--danger` #EF4444. White is only white (white background, black text, black accent, no tint). Black and Ocean share one near-black base; only the accent differs. The accent fills primary buttons and the active nav item. Default theme: Black if the device is in dark mode, otherwise White.

### 0.7 Full feature list (all of it is in scope)
**Access:** 1) Admin and Client login, hashed passwords, role checks on frontend and backend. 2) Admin creates clients (business name, owner, email or username, temporary password, phone, GST number, city, niche, status Active/Paused/Completed/Churned). 3) Theme toggle, mobile-responsive layout, search, filters, sorting, pagination on every list.

**Records:** 4) Clients (profile, assigned campaigns, notes). 5) Vendors (type Printer/Installer/Designer/Venue partner/Other, contact, area, rate notes, payment details, status Active/Inactive). 6) Packages (the four chart packages with poster count, days, price and inclusions; editable by Admin; Custom quote also possible). 7) Campaigns (client, name, package or Custom quote, poster count, duration in days, start and end date, target areas, price, separate placement fee, GST, QR destination WhatsApp/landing page/form/booking page/offer page, notes; total value calculated automatically). 8) Placements (the location list: location name, type Notice board/Approved ad space/Partner-shop window/Private wall with permission, area, address, GPS, contact, permission status Requested/Approved/Rejected, venue fee, poster count, installer, installed date, removed date, status Planned/Installed/Removed/Problem; CSV import of a whole location list; cannot be Installed until permission is Approved). 9) Stage tracker: Pending, Design, Printing, Installation, Live, Completed, Cancelled, with a progress bar; clients see the stage but cannot change it.

**Creative and tracking:** 10) Creative approval (admin uploads mockup and print-ready file; client approves, rejects or comments; all versions and timestamps kept; only approved creatives can be marked Ready for Printing). 11) QR generator (one QR and tracking link per **zone**: either one code for the whole campaign or one per target area; PNG and SVG download; regenerate keeps the same link). 12) Link builder with UTM tags added automatically: `utm_source=poster`, `utm_medium=offline`, `utm_campaign=campaign_name`, `utm_content=zone_code`, `utm_term=area`, with a copy button. 13) QR analytics (scans and unique scans per zone, area, date and campaign; WhatsApp clicks; form submits; top zones and areas; cost per scan; leads come from the client's typed lead counts and cost per lead is calculated from them; a scan alone does not prove a sale). 14) Proof photos (admin uploads with placement, campaign, date and time, GPS where the file has it, area, type Wide shot/Close-up/Before/After; status Verified/Pending/Rejected; clients see only verified photos and can download a proof pack). 15) Location map and location list (pins for every placement, planned versus installed per area, installation progress bar). 16) Placement checks (scheduled inspections per placement: OK, Poster missing, Damaged or peeling, Removed by owner or authority, Reposted; photo, action, repost task list, client notified).

**Reports and money (admin only unless stated):** 17) Client reports (weekly, monthly and final PDF: package, duration, posters ordered and installed, location list, areas, proof summary, placement checks, scans, WhatsApp clicks, leads, status; client downloads only their own). 18) Invoices (quotes and GST invoices: Advance Invoice, Final Invoice, Credit Note; statuses Draft/Sent/Partially Paid/Paid/Overdue/Cancelled; PDF; clients download their own). 19) Client payment tracker (Advance Received/Balance Due/Partially Paid/Fully Paid/Overdue; date, amount, method, UTR or reference, proof screenshot). 20) Cost and vendor payment tracker (Printing, Installation labour, Transport + supplies, Placement fees, Design + coordination, Reprint / damage, QR / tracking, Admin / misc, Sales commission; each cost has a vendor, a pay status Unpaid/Paid/Partially Paid/On Hold, reference, screenshot and bill; installer pay only after proof is verified). 21) Profit calculator (revenue, planned cost, actual cost, contribution and margin per campaign, client and month; never visible to clients).

**Operations:** 22) Admin dashboard (clients, active campaigns, posters installed, revenue, pending payments, unpaid vendor costs, scans, campaigns by stage, alerts for missing proof, permissions pending, ending soon, overdue invoices, unpaid vendor costs, placement problems, repost tasks, low QR activity). 23) Client dashboard (only their own data). 24) Notifications by email (events: campaign started, creative approval needed, report ready, invoice generated, payment due, campaign completed, posters reposted; admin alerts: missing proof, permission pending, campaign ending soon, invoice overdue, vendor payment pending, placement problem, new client query, low QR activity); manual copy-message for WhatsApp; clients have a notifications page. 25) Support queries (client raises, admin replies, sets priority, closes). 26) Document storage (client agreements, venue permission letters, installer agreements, invoices, receipts, permits or NOCs, approved creatives, campaign reports; organised by client, campaign and placement; clients have a documents page for their allowed types). 27) Audit log (client created, campaign created or edited, placement approved or installed, creative approved or rejected, proof uploaded, invoice created or paid, cost marked paid, report generated). 28) Export CSV and PDF for every list and report, with date filters. 29) Optional daily alert email and automatic Overdue marking (Prompt #14).

**Table names kept from the earlier spec:** users, clients, campaigns, creatives, qr_codes, qr_events, proofs, invoices, payments, costs, documents, support_queries, notifications, audit_logs. **Tables added for the poster version (PROPOSED):** `vendors`, `packages`, `campaign_finance`, `placements`, `placement_checks`, `lead_counts`. Total: **20 tables**.

**QR link format (PROPOSED):** `https://cx.cretoxy.cloud/c/CLIENTCODE/CAMPAIGNCODE/ZONECODE`

### 0.8 Why proof photo GPS is optional
Photos forwarded over WhatsApp usually lose GPS data, and admin-uploaded photos cannot be proven genuine. The app stores GPS when the file has it and works without it. The location list and its map pins come from the placement record, which Admin can fill in by hand.

---

## 1. Accounts & Prerequisites

Do these in order. Keep a text note (or password manager) for every key you collect; Section 2 tells you which ones.

**Step 1.1 — Check your computer.** Windows 10/11, macOS or Linux, with a modern browser (Chrome, Edge or Firefox) and about 2 GB free disk space.

**Step 1.2 — Install Node.js.**
1. Go to https://nodejs.org and download the **LTS** version (use Node 22 LTS or newer; minimum 20).
2. Run the installer with all defaults. It adds Node to your PATH automatically.
3. Close and reopen your terminal (Windows: search "PowerShell" → open it. macOS: Terminal app).
4. Verify. Run this anywhere:
```bash
node --version
npm --version
```
Pass: `node --version` shows `v20` or higher and `npm --version` shows a number. If you see "command not found", restart the computer and repeat step 4.

**Step 1.3 — Install Git.**
1. Download from https://git-scm.com/downloads and install with defaults.
2. Reopen the terminal and verify:
```bash
git --version
```
3. Tell Git who you are (run once, anywhere; use your own name and email):
```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

**Step 1.4 — GitHub account.**
1. Go to https://github.com/signup and create an account (plan: **Free**).
2. After signing in click the **+** (top right) → **New repository**.
3. Repository name: `cretoxy-poster-manager`. Choose **Private**. Do **not** tick "Add a README". Click **Create repository**. Leave the page open; Section 4 uses it.

**Step 1.5 — Supabase account.**
1. Go to https://supabase.com → **Start your project** → sign up (GitHub login is fine). Plan: start on **Free** while building.
2. ⚠️ Free projects pause after 7 days of inactivity and have no backups. **Before the first poster with a QR code is printed, upgrade to Pro ($25 per month)**. Upgrade path: Dashboard → your organization → **Billing** → change plan to **Pro**. Pro removes the pause and adds daily backups kept for 7 days.
3. Region choice happens in Section 2 (choose the closest to India).

**Step 1.6 — Cloudflare account and your domain.**
1. Go to https://dash.cloudflare.com/sign-up and create an account. Plan: **Free**. (Skip if you already have one.)
2. Your domain `cretoxy.cloud` must already be added to Cloudflare and show **Active**. Check: Cloudflare dashboard → **Websites** (left menu) → `cretoxy.cloud` → status **Active**. If it is not there: click **Add a domain**, enter `cretoxy.cloud`, choose the **Free** plan, and follow Cloudflare's instructions to change the nameservers at your domain registrar. Wait until the status says **Active**.
3. Free plan limits that matter: Pages has 500 builds per month and unlimited static requests. Workers and Pages Functions share 100,000 requests per day and 10 ms CPU per request. The paid Workers plan ($5 per month) is the upgrade if you outgrow that.
4. The app address is **cx.cretoxy.cloud** (the subdomain `cx` on `cretoxy.cloud`). QR scans use the same address with the path `/c/...`. Nothing to replace: this guide already uses the real names.

**Step 1.7 — Resend account (email).** ⚠️ recommended, not confirmed.
1. Go to https://resend.com/signup and sign up. Plan: **Free** (3,000 emails per month, max 100 per day, 1 domain; Pro is $20 per month).
2. You verify your domain in Section 3, step 3.9. Nothing to do yet.

**Step 1.8 — Install your AI coding agent.**
- opencode: follow the official instructions at https://opencode.ai (the install command is there). ⚠️ I did not look up exact install commands for opencode or Antigravity; use the official pages.
- Antigravity: download from the official Antigravity site and sign in.
- Verify: open the tool in an empty folder and ask "say hello". If it replies, it works.

**Step 1.9 — Create your secrets note.** Create a file on your computer named `poster-manager-secrets.txt` **outside** any project folder with these empty lines. Never upload it anywhere.
```text
SUPABASE_URL=
SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
SUPABASE_DB_PASSWORD=
SCAN_HASH_SALT=
RESEND_API_KEY=
EMAIL_FROM=
ADMIN_ALERT_EMAIL=
```
(`ADMIN_ALERT_EMAIL` is only needed for the optional Prompt #14. Fill it with the email address that should receive your daily alert digest.)

**Prerequisite check:** `node --version` works, `git --version` works, you have GitHub, Supabase, Cloudflare (domain Active) and Resend logins, and an empty private GitHub repo named `cretoxy-poster-manager`.

## 2. Supabase Setup (full detail)

Everything in this section is done in the Supabase dashboard in your browser. Run the SQL blocks **in the order shown** (A, B, C, D, E, F, G, H). Column names below are built from the feature list and your price chart; they are **(PROPOSED)**. Table names follow Section 0.7.

**Step 2.1 — Create the project.**
1. Go to https://supabase.com/dashboard and sign in.
2. Click **New project**. If asked, pick your organization (the default one is fine).
3. Fill in the fields:
   - **Name:** `cretoxy-poster-manager`
   - **Database Password:** click **Generate a password**, then copy it into `poster-manager-secrets.txt` as `SUPABASE_DB_PASSWORD`.
   - **Region:** choose the one closest to India (Mumbai / `ap-south-1` if listed, otherwise the nearest Asia option).
   - **Pricing plan:** Free for now (upgrade rule in Step 1.5).
4. Click **Create new project**. Wait 1–3 minutes until the dashboard loads.

**Step 2.2 — Copy the project URL and keys.**
1. Left menu → **Project Settings** (gear icon) → **API Keys** (older dashboards: **API**).
2. Copy **Project URL** → save as `SUPABASE_URL`.
3. Copy the public key labelled **anon** (newer dashboards call it **publishable**) → save as `SUPABASE_ANON_KEY`. It is safe to use in the browser.
4. Copy the key labelled **service_role** (newer dashboards call it **secret**) → save as `SUPABASE_SERVICE_ROLE_KEY`. **This key bypasses all security. Never put it in frontend code, in GitHub, or in a chat.**
5. Make up a long random text (at least 32 characters, letters and numbers) and save it as `SCAN_HASH_SALT`. It is used to anonymise QR scan visitors.

**Step 2.3 — Turn off public signup (required: no public registration).**
1. Left menu → **Authentication** → **Sign In / Providers** (older dashboards: **Providers**).
2. Find the section **User Signups** and switch **Allow new users to sign up** to **OFF**.
3. Click **Email** in the provider list. Make sure **Enable Email provider** is ON. Leave **Confirm email** ON (admin-created accounts are auto-confirmed by the server code, so clients never get a confirmation step). Click **Save**.
4. Leave every other provider (Google, GitHub, Phone, etc.) **OFF**.

**Step 2.4 — Set the redirect URLs.**
1. Left menu → **Authentication** → **URL Configuration**.
2. **Site URL:** `http://localhost:5173`
3. Under **Redirect URLs** click **Add URL** and add `http://localhost:5173/**`. Click **Save**.
4. (The production address is added in Section 6, after the site exists.)

**Step 2.5 — Run the SQL.** For every block: left menu → **SQL Editor** → **New query** → paste the whole block → click **Run**. Pass = a green "Success. No rows returned". If you get an error, stop and see Section 8 before running the next block.


### SQL Block A — Create all tables and add the four packages
```sql
-- Create clients table
create table public.clients (
  id uuid primary key default gen_random_uuid(),
  client_code text not null unique,
  business_name text not null,
  owner_name text,
  email text,
  phone text,
  address text,
  city text,
  area text,
  niche text,
  gst_number text,
  notes text,
  status text not null default 'Active' check (status in ('Active','Paused','Completed','Churned')),
  created_at timestamptz not null default now()
);

-- Create users table (app profile linked to Supabase Auth)
create table public.users (
  id uuid primary key references auth.users(id) on delete cascade,
  role text not null check (role in ('admin','client')),
  client_id uuid references public.clients(id) on delete set null,
  full_name text,
  theme text not null default 'white' check (theme in ('white','black','ocean')),
  must_change_password boolean not null default true,
  created_at timestamptz not null default now(),
  check (role = 'admin' or client_id is not null)
);

-- Create vendors table (ADMIN ONLY: printers, installers, designers, venue partners)
create table public.vendors (
  id uuid primary key default gen_random_uuid(),
  vendor_type text not null check (vendor_type in ('Printer','Installer','Designer','Venue partner','Other')),
  name text not null,
  contact_person text,
  phone text,
  email text,
  area text,
  rate_notes text,
  payment_details text,
  status text not null default 'Active' check (status in ('Active','Inactive')),
  notes text,
  created_at timestamptz not null default now()
);

-- Create packages table (ADMIN ONLY: the price-chart packages)
create table public.packages (
  id uuid primary key default gen_random_uuid(),
  name text not null unique,
  poster_count integer not null check (poster_count > 0),
  duration_days integer not null check (duration_days > 0),
  price numeric(12,2) not null check (price >= 0),
  inclusions text[] not null default '{}',
  active boolean not null default true,
  sort_order integer not null default 0,
  created_at timestamptz not null default now()
);

-- Create campaigns table (client-visible fields only)
create table public.campaigns (
  id uuid primary key default gen_random_uuid(),
  campaign_code text not null unique,
  client_id uuid not null references public.clients(id) on delete restrict,
  name text not null,
  package_name text not null default 'Custom quote',
  poster_count integer not null check (poster_count > 0),
  duration_days integer not null default 7 check (duration_days > 0),
  start_date date,
  end_date date,
  target_areas text[] not null default '{}',
  price numeric(12,2) not null check (price >= 0),
  placement_fee_charged numeric(12,2) not null default 0 check (placement_fee_charged >= 0),
  gst_percent numeric(5,2) not null default 18,
  qr_destination_type text check (qr_destination_type in ('WhatsApp','Landing page','Form','Booking page','Offer page')),
  qr_destination_url text,
  stage text not null default 'Pending' check (stage in ('Pending','Design','Printing','Installation','Live','Completed','Cancelled')),
  notes text,
  created_at timestamptz not null default now(),
  check (start_date is null or end_date is null or end_date >= start_date)
);

-- Create campaign_finance table (ADMIN ONLY: planned internal costs, hidden from clients)
create table public.campaign_finance (
  campaign_id uuid primary key references public.campaigns(id) on delete cascade,
  print_cost_per_poster numeric(10,2) not null default 8,
  planned_labour_cost numeric(12,2) not null default 0,
  planned_transport_cost numeric(12,2) not null default 0,
  planned_placement_fees numeric(12,2) not null default 0,
  planned_design_cost numeric(12,2) not null default 0
);

-- Create placements table (ADMIN ONLY: every approved location; clients see a safe view)
create table public.placements (
  id uuid primary key default gen_random_uuid(),
  campaign_id uuid not null references public.campaigns(id) on delete cascade,
  placement_code text not null,
  location_name text not null,
  placement_type text not null check (placement_type in ('Notice board','Approved ad space','Partner-shop window','Private wall with permission')),
  area text,
  address text,
  gps_lat numeric(9,6),
  gps_lng numeric(9,6),
  contact_name text,
  contact_phone text,
  permission_status text not null default 'Requested' check (permission_status in ('Requested','Approved','Rejected')),
  permission_note text,
  venue_fee numeric(12,2) not null default 0 check (venue_fee >= 0),
  poster_count integer not null default 0 check (poster_count >= 0),
  installer_vendor_id uuid references public.vendors(id) on delete set null,
  installed_on date,
  removed_on date,
  status text not null default 'Planned' check (status in ('Planned','Installed','Removed','Problem')),
  notes text,
  created_at timestamptz not null default now(),
  unique (campaign_id, placement_code),
  -- Safety rule: posters go only on authorised places, so Installed needs Approved permission
  check (status <> 'Installed' or permission_status = 'Approved')
);

-- Create creatives table (all versions kept)
create table public.creatives (
  id uuid primary key default gen_random_uuid(),
  campaign_id uuid not null references public.campaigns(id) on delete cascade,
  version integer not null default 1,
  kind text not null check (kind in ('Mockup','Print-ready')),
  file_path text not null,
  status text not null default 'Pending' check (status in ('Pending','Approved','Rejected')),
  client_comment text,
  decided_at timestamptz,
  ready_for_printing boolean not null default false,
  created_at timestamptz not null default now(),
  check (not ready_for_printing or status = 'Approved')
);

-- Create qr_codes table (one per zone per campaign; zone ALL = whole campaign)
create table public.qr_codes (
  id uuid primary key default gen_random_uuid(),
  campaign_id uuid not null references public.campaigns(id) on delete cascade,
  client_code text not null,
  campaign_code text not null,
  zone_code text not null,
  area text,
  destination_url text not null,
  tracking_path text not null unique,
  regenerated_at timestamptz,
  created_at timestamptz not null default now(),
  unique (campaign_id, zone_code)
);

-- Create qr_events table (every scan or click)
create table public.qr_events (
  id bigint generated always as identity primary key,
  qr_code_id uuid not null references public.qr_codes(id) on delete cascade,
  event_type text not null default 'scan' check (event_type in ('scan','click','whatsapp_click','form_submit')),
  visitor_hash text,
  device_type text,
  occurred_at timestamptz not null default now()
);

-- Create proofs table (admin-uploaded proof photos)
create table public.proofs (
  id uuid primary key default gen_random_uuid(),
  campaign_id uuid not null references public.campaigns(id) on delete cascade,
  placement_id uuid not null references public.placements(id) on delete cascade,
  file_path text not null,
  photo_type text not null check (photo_type in ('Wide shot','Close-up','Before','After')),
  taken_at timestamptz,
  uploaded_at timestamptz not null default now(),
  gps_lat numeric(9,6),
  gps_lng numeric(9,6),
  area text,
  status text not null default 'Pending' check (status in ('Verified','Pending','Rejected')),
  uploaded_by uuid references public.users(id)
);

-- Create placement_checks table (scheduled inspections and problems; ADMIN ONLY, clients see a safe view)
create table public.placement_checks (
  id uuid primary key default gen_random_uuid(),
  campaign_id uuid not null references public.campaigns(id) on delete cascade,
  placement_id uuid not null references public.placements(id) on delete cascade,
  check_date date not null default current_date,
  result text not null check (result in ('OK','Poster missing','Damaged or peeling','Removed by owner or authority','Reposted')),
  posters_affected integer not null default 0 check (posters_affected >= 0),
  action_taken text,
  repost_needed boolean not null default false,
  photo_path text,
  client_notified boolean not null default false,
  notes text,
  created_at timestamptz not null default now()
);

-- Create invoices table
create table public.invoices (
  id uuid primary key default gen_random_uuid(),
  campaign_id uuid references public.campaigns(id) on delete set null,
  client_id uuid not null references public.clients(id) on delete restrict,
  invoice_number text not null unique,
  invoice_type text not null check (invoice_type in ('Quote','Advance Invoice','Final Invoice','Credit Note')),
  status text not null default 'Draft' check (status in ('Draft','Sent','Partially Paid','Paid','Overdue','Cancelled')),
  issue_date date not null default current_date,
  due_date date,
  client_gst_number text,
  line_items jsonb not null default '[]',
  gst_percent numeric(5,2) not null default 18,
  subtotal numeric(12,2) not null default 0,
  gst_amount numeric(12,2) not null default 0,
  total numeric(12,2) not null default 0,
  created_at timestamptz not null default now()
);

-- Create payments table (client payments received by wire transfer; clients read a safe view)
create table public.payments (
  id uuid primary key default gen_random_uuid(),
  invoice_id uuid references public.invoices(id) on delete set null,
  campaign_id uuid not null references public.campaigns(id) on delete restrict,
  client_id uuid not null references public.clients(id) on delete restrict,
  amount numeric(12,2) not null check (amount > 0),
  paid_on date not null default current_date,
  method text not null default 'Wire transfer',
  utr_reference text,
  proof_path text,
  notes text,
  created_at timestamptz not null default now()
);

-- Create costs table (ADMIN ONLY: every cost, with vendor payment tracking)
create table public.costs (
  id uuid primary key default gen_random_uuid(),
  campaign_id uuid not null references public.campaigns(id) on delete cascade,
  vendor_id uuid references public.vendors(id) on delete set null,
  category text not null check (category in ('Printing','Installation labour','Transport + supplies','Placement fees','Design + coordination','Reprint / damage','QR / tracking','Admin / misc','Sales commission')),
  amount numeric(12,2) not null check (amount >= 0),
  incurred_on date not null default current_date,
  pay_status text not null default 'Unpaid' check (pay_status in ('Unpaid','Paid','Partially Paid','On Hold')),
  paid_amount numeric(12,2) not null default 0 check (paid_amount >= 0),
  paid_on date,
  payment_ref text,
  payment_proof_path text,
  bill_path text,
  notes text,
  created_at timestamptz not null default now()
);

-- Create documents table
create table public.documents (
  id uuid primary key default gen_random_uuid(),
  doc_type text not null check (doc_type in ('Client agreement','Venue permission letter','Installer agreement','Invoice','Receipt','Permit or NOC','Approved creative','Campaign report')),
  title text,
  client_id uuid references public.clients(id) on delete set null,
  campaign_id uuid references public.campaigns(id) on delete set null,
  placement_id uuid references public.placements(id) on delete set null,
  vendor_id uuid references public.vendors(id) on delete set null,
  file_path text not null,
  created_at timestamptz not null default now()
);

-- Create support_queries table
create table public.support_queries (
  id uuid primary key default gen_random_uuid(),
  client_id uuid not null references public.clients(id) on delete cascade,
  campaign_id uuid references public.campaigns(id) on delete set null,
  subject text not null,
  message text not null,
  priority text not null default 'Normal' check (priority in ('Low','Normal','High')),
  status text not null default 'Open' check (status in ('Open','Resolved','Closed')),
  admin_reply text,
  replied_at timestamptz,
  created_at timestamptz not null default now()
);

-- Create notifications table
create table public.notifications (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references public.users(id) on delete cascade,
  channel text not null default 'email' check (channel in ('email','whatsapp-copy')),
  event text not null,
  payload jsonb not null default '{}',
  status text not null default 'Pending' check (status in ('Pending','Sent','Failed')),
  created_at timestamptz not null default now(),
  sent_at timestamptz
);

-- Create audit_logs table
create table public.audit_logs (
  id bigint generated always as identity primary key,
  user_id uuid references public.users(id) on delete set null,
  action text not null,
  record_table text,
  record_id text,
  details jsonb not null default '{}',
  created_at timestamptz not null default now()
);

-- Create lead_counts table (client types lead counts for a reporting period)
create table public.lead_counts (
  id uuid primary key default gen_random_uuid(),
  campaign_id uuid not null references public.campaigns(id) on delete cascade,
  period_start date not null,
  period_end date not null,
  leads integer not null check (leads >= 0),
  entered_by uuid references public.users(id),
  created_at timestamptz not null default now(),
  check (period_end >= period_start),
  unique (campaign_id, period_start, period_end)
);

-- Add the four packages from the price chart
insert into public.packages (name, poster_count, duration_days, price, inclusions, sort_order) values
  ('Local Reach', 100, 7, 5999, array['A4 colour posters','Two-person installation team','Approved placement locations','Photo proof + location list','30-day options by quote'], 1),
  ('Area Campaign', 250, 14, 13999, array['Printing + installation','Multiple approved locations','Photo verification','Campaign tracking code','30-day options by quote'], 2),
  ('High-Visibility', 500, 30, 24999, array['Two-person installation team','Selected zones / local areas','Scheduled placement checks','Photo evidence + report','30-day options by quote'], 3),
  ('Mass Local', 1000, 30, 44999, array['Large-scale distribution','Multiple approved zones','Scheduled placement checks','Final campaign report','30-day options by quote'], 4);
```

### SQL Block B — Indexes (speed)
```sql
-- Add indexes on the columns used for lookups
create index on public.campaigns (client_id);
create index on public.placements (campaign_id);
create index on public.placements (campaign_id, area);
create index on public.creatives (campaign_id);
create index on public.qr_codes (campaign_id);
create index on public.qr_events (qr_code_id, occurred_at);
create index on public.proofs (campaign_id, placement_id);
create index on public.placement_checks (campaign_id, placement_id);
create index on public.invoices (client_id);
create index on public.payments (campaign_id);
create index on public.costs (campaign_id);
create index on public.costs (vendor_id);
create index on public.documents (client_id);
create index on public.support_queries (client_id);
create index on public.lead_counts (campaign_id);
create index on public.audit_logs (created_at);
```

### SQL Block C — Helper functions
```sql
-- Create helper: is the logged-in user an admin?
create or replace function public.is_admin()
returns boolean language sql stable security definer set search_path = public as $$
  select exists (select 1 from public.users u where u.id = auth.uid() and u.role = 'admin');
$$;

-- Create helper: which client does the logged-in user belong to?
create or replace function public.my_client_id()
returns uuid language sql stable security definer set search_path = public as $$
  select client_id from public.users where id = auth.uid();
$$;

-- Create helper: does the logged-in client own this campaign?
create or replace function public.owns_campaign(cid uuid)
returns boolean language sql stable security definer set search_path = public as $$
  select exists (select 1 from public.campaigns c where c.id = cid and c.client_id = public.my_client_id());
$$;
```

### SQL Block D — Turn on row-level security, admin access, client access
```sql
-- Turn on row-level security and give admins full access to every table
do $$
declare t text;
begin
  foreach t in array array['clients','users','vendors','packages','campaigns','campaign_finance','placements','creatives','qr_codes','qr_events','proofs','placement_checks','invoices','payments','costs','documents','support_queries','notifications','audit_logs','lead_counts'] loop
    execute format('alter table public.%I enable row level security', t);
    execute format('create policy "admin_all" on public.%I for all to authenticated using (public.is_admin()) with check (public.is_admin())', t);
  end loop;
end $$;

-- Client policies: each client can read only their own data
create policy "client_read_own" on public.clients for select to authenticated using (id = public.my_client_id());
create policy "user_read_self" on public.users for select to authenticated using (id = auth.uid());
create policy "user_update_self" on public.users for update to authenticated using (id = auth.uid()) with check (id = auth.uid());
revoke update on public.users from authenticated;
grant update (theme, full_name, must_change_password) on public.users to authenticated;

create policy "client_read_own" on public.campaigns for select to authenticated using (client_id = public.my_client_id());
create policy "client_read_own" on public.creatives for select to authenticated using (public.owns_campaign(campaign_id));
create policy "client_read_own" on public.qr_codes for select to authenticated using (public.owns_campaign(campaign_id));
create policy "client_read_own" on public.qr_events for select to authenticated using (
  exists (select 1 from public.qr_codes q where q.id = qr_events.qr_code_id and public.owns_campaign(q.campaign_id)));
create policy "client_read_verified" on public.proofs for select to authenticated using (status = 'Verified' and public.owns_campaign(campaign_id));
create policy "client_read_sent" on public.invoices for select to authenticated using (client_id = public.my_client_id() and status <> 'Draft');
create policy "client_read_docs" on public.documents for select to authenticated using (
  client_id = public.my_client_id() and doc_type in ('Client agreement','Invoice','Receipt','Approved creative','Campaign report'));
create policy "client_read_own" on public.notifications for select to authenticated using (user_id = auth.uid());

create policy "client_read_own" on public.support_queries for select to authenticated using (client_id = public.my_client_id());
create policy "client_insert_own" on public.support_queries for insert to authenticated with check (client_id = public.my_client_id());

create policy "client_read_own" on public.lead_counts for select to authenticated using (public.owns_campaign(campaign_id));
create policy "client_insert_own" on public.lead_counts for insert to authenticated with check (public.owns_campaign(campaign_id));
create policy "client_update_own" on public.lead_counts for update to authenticated using (public.owns_campaign(campaign_id)) with check (public.owns_campaign(campaign_id));
-- No client policy exists for: vendors, packages, campaign_finance, placements, placement_checks, payments, costs, audit_logs.
-- Clients read placements, checks and payments only through the safe views in Block E.
```

### SQL Block E — Creative approval function and the safe views for clients
```sql
-- Create function: client approves or rejects a creative (the only way a client can change a creative)
create or replace function public.decide_creative(p_creative_id uuid, p_decision text, p_comment text)
returns void language plpgsql security definer set search_path = public as $$
begin
  if p_decision not in ('Approved','Rejected') then
    raise exception 'Decision must be Approved or Rejected';
  end if;
  update public.creatives
     set status = p_decision, client_comment = p_comment, decided_at = now()
   where id = p_creative_id and public.owns_campaign(campaign_id);
  if not found then
    raise exception 'Creative not found';
  end if;
end $$;
grant execute on function public.decide_creative(uuid, text, text) to authenticated;

-- Create view: clients see the location list without contact phones, venue fees or permission notes
create or replace view public.client_placements_view as
select p.campaign_id, p.id as placement_id, p.placement_code, p.location_name, p.placement_type,
       p.area, p.address, p.gps_lat, p.gps_lng, p.poster_count, p.installed_on, p.removed_on, p.status
from public.placements p
where public.owns_campaign(p.campaign_id);

-- Create view: clients see check results without internal notes or photos
create or replace view public.client_checks_view as
select c.campaign_id, c.placement_id, c.check_date, c.result, c.posters_affected, c.repost_needed
from public.placement_checks c
where public.owns_campaign(c.campaign_id);

-- Create view: clients see their payments without internal notes or screenshots
create or replace view public.client_payments_view as
select pay.id, pay.invoice_id, pay.campaign_id, pay.amount, pay.paid_on, pay.method, pay.utr_reference
from public.payments pay
where pay.client_id = public.my_client_id();

grant select on public.client_placements_view, public.client_checks_view, public.client_payments_view to authenticated;
revoke all on public.client_placements_view, public.client_checks_view, public.client_payments_view from anon;
```

### SQL Block F — Storage buckets and file access rules
File paths inside buckets follow these patterns **(PROPOSED)**: proofs `CLIENT_ID/CAMPAIGN_ID/PLACEMENT_ID/filename`; creatives `CLIENT_ID/CAMPAIGN_ID/filename`; documents and internal-files `CLIENT_ID/CAMPAIGN_ID/filename`.
```sql
-- Create four private storage buckets
insert into storage.buckets (id, name, public) values
  ('proofs','proofs',false),
  ('creatives','creatives',false),
  ('documents','documents',false),
  ('internal-files','internal-files',false)
on conflict (id) do nothing;

-- Admin can do everything in all four buckets
create policy "storage_admin_all" on storage.objects for all to authenticated
using (public.is_admin() and bucket_id in ('proofs','creatives','documents','internal-files'))
with check (public.is_admin() and bucket_id in ('proofs','creatives','documents','internal-files'));

-- Clients can read only verified proof photos of their own campaigns
create policy "storage_client_read_proofs" on storage.objects for select to authenticated
using (bucket_id = 'proofs' and exists (
  select 1 from public.proofs p
  where p.file_path = storage.objects.name and p.status = 'Verified' and public.owns_campaign(p.campaign_id)));

-- Clients can read creatives of their own campaigns
create policy "storage_client_read_creatives" on storage.objects for select to authenticated
using (bucket_id = 'creatives' and exists (
  select 1 from public.creatives cr
  where cr.file_path = storage.objects.name and public.owns_campaign(cr.campaign_id)));

-- Clients can read their own client-visible documents
create policy "storage_client_read_documents" on storage.objects for select to authenticated
using (bucket_id = 'documents' and exists (
  select 1 from public.documents d
  where d.file_path = storage.objects.name and d.client_id = public.my_client_id()
    and d.doc_type in ('Client agreement','Invoice','Receipt','Approved creative','Campaign report')));
-- 'internal-files' (payment screenshots, vendor payment screenshots, bills, permission letters) is admin-only: no client policy.
```

**Step 2.6 — Create the first Admin user.**
1. Left menu → **Authentication** → **Users** → **Add user** → **Create new user**.
2. Enter your admin **Email** and a strong **Password**. Tick **Auto Confirm User**. Click **Create user**.
3. Go to **SQL Editor** → **New query** and run this (replace `ADMIN-EMAIL` with the email you just used):

### SQL Block G — Make that user an Admin
```sql
-- Add the first admin profile
insert into public.users (id, role, full_name, must_change_password)
select id, 'admin', 'Admin', false from auth.users where email = 'ADMIN-EMAIL';
```
Pass: "Success. 1 row" (look for "1 row affected" or similar). If it says 0 rows, the email does not match exactly; fix and rerun.

**Step 2.7 — Run the verification query.**

### SQL Block H — Check everything exists
```sql
-- List the tables and confirm RLS is on
select tablename, rowsecurity from pg_tables where schemaname = 'public' order by tablename;
```
Pass: 20 rows, and the `rowsecurity` column is `true` for every row.

**Step 2.8 — Environment variable names (PROPOSED; none were fixed yet).**
| Where it is used | Name | Value |
|---|---|---|
| Frontend (public, build-time) | `VITE_SUPABASE_URL` | the Project URL |
| Frontend (public, build-time) | `VITE_SUPABASE_ANON_KEY` | the anon/publishable key |
| Server only (Pages Functions and the Worker) | `SUPABASE_URL` | the Project URL |
| Server only | `SUPABASE_SERVICE_ROLE_KEY` | the service_role/secret key |
| Server only (Worker) | `SCAN_HASH_SALT` | your random text |
| Server only (Pages Functions) | `RESEND_API_KEY`, `EMAIL_FROM` | from Resend, set in Section 3 |
| Server only (optional daily alert Worker, Prompt #14) | `ADMIN_ALERT_EMAIL` | your own email address |

**Section 2 done when:** Block A–H all ran, Step 2.3 signups are OFF, the admin user exists, and `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `SCAN_HASH_SALT` are in your secrets note.

---

## 3. Cloudflare Account, pages

**Which part of Cloudflare does what (so you always know where to click):**
- **Cloudflare Pages** = hosts the website (frontend) and the small server functions in `functions/api`. Dashboard: **Workers & Pages**.
- **Cloudflare Worker `cretoxy-qr-redirect`** = answers every QR scan at `cx.cretoxy.cloud/c/*`, logs it, and redirects. Dashboard: **Workers & Pages** too, but it is a separate item from the Pages project.
- **DNS / SSL** = **Websites** → your domain.
- **R2 / KV** = not used in the main path (see Section 9).

The app lives at **cx.cretoxy.cloud**. QR scans live at **cx.cretoxy.cloud/c/...** on the same address. The Worker route only catches paths that start with `/c/`; everything else on the address goes to the app. **Never use `/c/` for an app page.**

**Step 3.1 — Make a work folder for Cloudflare setup.**
1. Open your terminal and go to the folder where you want all Cretoxy work to live (called the **parent folder** below). Example:
   - macOS / Linux: `cd ~/Documents`
   - Windows PowerShell: `cd $HOME\Documents`
2. Create the setup folder and a package file inside it:
```bash
mkdir cretoxy-cf-setup
cd cretoxy-cf-setup
npm init -y
npm install -D wrangler
```
What it does: makes a throwaway folder and installs Cloudflare's command-line tool (`wrangler`) in it. Wait until the install finishes.
3. Verify:
```bash
npx wrangler --version
```
Pass: prints a version number (minimum 3.x; any current number is fine).

**Step 3.2 — Log in to Cloudflare from the terminal.** Run inside `cretoxy-cf-setup`:
```bash
npx wrangler login
```
A browser opens. Click **Allow**. Pass: the terminal says "Successfully logged in". Check:
```bash
npx wrangler whoami
```
Pass: shows your email and account.

**Step 3.3 — Check the SSL/TLS mode.**
1. Cloudflare dashboard → **Websites** → click your domain → left menu **SSL/TLS** → **Overview**.
2. Note the current mode. **If your existing main site already works, do not change it.** Pages custom domains and Worker routes work in every mode except "Off". If the mode is **Off**, click **Configure** and choose **Full (strict)** (recommended for new setups).

**Step 3.4 — Check that nothing else already uses `cx.cretoxy.cloud`.**
1. Cloudflare dashboard → **Websites** → `cretoxy.cloud` → **DNS** → **Records**.
2. Look for a record whose **Name** is `cx`. If one exists and you do not need it, click **Edit** → **Delete** (Pages creates the right record for you in Step 3.7). If you are not sure what it is, stop and check before deleting.
3. If no `cx` record exists, do nothing. (You do **not** need an extra record for the main domain in this guide.)

**Step 3.5 — Create the Pages project (the website).** Run inside `cretoxy-cf-setup`:
```bash
npx wrangler pages project create cretoxy-poster-manager --production-branch main
```
What it does: creates an empty Pages project named `cretoxy-poster-manager`. If it asks questions, accept the defaults. Pass: it prints a `https://cretoxy-poster-manager.pages.dev` address.

**Step 3.6 — Deploy a placeholder page (so you can attach the domain now).**
1. Create the placeholder:
   - macOS / Linux: `mkdir placeholder-site && echo "<h1>Cretoxy Poster Manager coming soon</h1>" > placeholder-site/index.html`
   - Windows PowerShell: `mkdir placeholder-site; Set-Content placeholder-site\index.html "<h1>Cretoxy Poster Manager coming soon</h1>"`
2. Deploy it:
```bash
npx wrangler pages deploy placeholder-site --project-name cretoxy-poster-manager --branch main
```
Pass: prints "Deployment complete" and a URL. Open the `*.pages.dev` URL in a browser; you should see "Cretoxy Poster Manager coming soon".

**Step 3.7 — Attach the custom domain to the Pages project.**
1. Cloudflare dashboard → **Workers & Pages** → click **cretoxy-poster-manager** → tab **Custom domains** → **Set up a custom domain**.
2. Type `cx.cretoxy.cloud` → **Continue** → **Activate domain** (Cloudflare adds the DNS record for you because the domain is on Cloudflare).
3. Wait 1–5 minutes until the status shows **Active**. Open `https://cx.cretoxy.cloud`. Pass: you see the placeholder page with a padlock.

**Step 3.8 — Create the QR redirect Worker (placeholder version).** Run inside `cretoxy-cf-setup`:
1. Make the folders:
   - macOS / Linux: `mkdir -p qr-redirect/src`
   - Windows PowerShell: `mkdir qr-redirect\src`
2. Create `qr-redirect/wrangler.toml` with exactly this content (replace `YOUR-PROJECT-REF` with the first part of your Supabase Project URL, for example the `abcd1234` in `https://abcd1234.supabase.co`):
```toml
name = "cretoxy-qr-redirect"
main = "src/index.ts"
compatibility_date = "2026-10-01"
workers_dev = false

routes = [
  { pattern = "cx.cretoxy.cloud/c/*", zone_name = "cretoxy.cloud" }
]

[vars]
SUPABASE_URL = "https://YOUR-PROJECT-REF.supabase.co"

[observability]
enabled = true
```
3. Create `qr-redirect/src/index.ts` with exactly this content:
```ts
export interface Env {
  SUPABASE_URL: string;
  SUPABASE_SERVICE_ROLE_KEY: string;
  SCAN_HASH_SALT: string;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    return new Response("QR redirect Worker is live. Real logic arrives in Prompt #5.", { status: 200 });
  },
};
```
4. Add the two secrets. Run inside `cretoxy-cf-setup/qr-redirect` (it will ask you to paste each value, then press Enter):
```bash
cd qr-redirect
npx wrangler secret put SUPABASE_SERVICE_ROLE_KEY
npx wrangler secret put SCAN_HASH_SALT
```
If it asks "There doesn't seem to be a Worker called cretoxy-qr-redirect. Do you want to create a new Worker?" answer **y**.
5. Deploy:
```bash
npx wrangler deploy
```
Pass: prints "Deployed cretoxy-qr-redirect" and the route `cx.cretoxy.cloud/c/*`.
6. Test: open `https://cx.cretoxy.cloud/c/test` in a browser. Pass: you see "QR redirect Worker is live...". If you see a Cloudflare error, see Section 8.

**Step 3.9 — Set up email sending (Resend) and the Pages secrets.**
1. Resend dashboard → **Domains** → **Add Domain** → enter `mail.cretoxy.cloud` (a sending subdomain; **PROPOSED**) → **Add**.
2. Resend shows DNS records (MX, TXT for SPF, TXT/CNAME for DKIM). In Cloudflare → **Websites** → your domain → **DNS** → **Records** → **Add record** for each one exactly as shown. For every record set **Proxy status** to **DNS only** (grey cloud).
3. Back in Resend click **Verify DNS Records**. Wait until the status says **Verified** (can take up to an hour).
4. Resend → **API Keys** → **Create API Key** → name `cretoxy-poster-manager`, permission **Sending access** → **Add** → copy the key (shown once) → save as `RESEND_API_KEY`. Save `EMAIL_FROM` as `Cretoxy Poster Manager <noreply@mail.cretoxy.cloud>`.
5. Set the Pages secrets. Run inside `cretoxy-cf-setup` (each command asks you to paste the value):
```bash
npx wrangler pages secret put SUPABASE_URL --project-name cretoxy-poster-manager
npx wrangler pages secret put SUPABASE_SERVICE_ROLE_KEY --project-name cretoxy-poster-manager
npx wrangler pages secret put RESEND_API_KEY --project-name cretoxy-poster-manager
npx wrangler pages secret put EMAIL_FROM --project-name cretoxy-poster-manager
```
Pass: each prints "Success! Uploaded secret ...".

**Section 3 done when:** `https://cx.cretoxy.cloud` shows the placeholder with a padlock, `https://cx.cretoxy.cloud/c/test` shows the Worker message, Resend shows **Verified**, and all six secrets are set.

---

## 4. Local Project Setup

**Step 4.1 — Create the project folder and connect Git.**
1. Go to the **parent folder** (the folder that contains `cretoxy-cf-setup`). From any place run the same command you used in Step 3.1: macOS / Linux `cd ~/Documents`, Windows PowerShell `cd $HOME\Documents`.
2. Run:
```bash
mkdir cretoxy-poster-manager
cd cretoxy-poster-manager
git init -b main
git remote add origin https://github.com/YOUR-GITHUB-USERNAME/cretoxy-poster-manager.git
```
(Replace `YOUR-GITHUB-USERNAME` with your GitHub username.) Pass: no error messages.

**Step 4.2 — Create the React app (Vite + React + TypeScript).** Run inside `cretoxy-poster-manager` (it is empty now, which is required):
```bash
npm create vite@latest . -- --template react-ts
npm install
```
If it asks "Current directory is not empty" you ran it in the wrong folder. If it asks about "rolldown" or "experimental", choose **No**. Pass: `npm install` ends without red errors.

**Step 4.3 — Install the packages.**
```bash
npm install @supabase/supabase-js react-router-dom leaflet react-leaflet qrcode jspdf jspdf-autotable papaparse jszip
npm install -D wrangler @cloudflare/workers-types @types/leaflet @types/qrcode @types/papaparse
```
What they are for: `@supabase/supabase-js` talks to Supabase; `react-router-dom` = page routing; `leaflet` + `react-leaflet` = location map (⚠️ tile provider undecided); `qrcode` = QR PNG/SVG; `jspdf` + `jspdf-autotable` = PDFs in the browser; `papaparse` = CSV import/export (including the location list import); `jszip` = proof pack zip download; `wrangler` = Cloudflare CLI.

**Step 4.4 — Create the folder structure.** Final structure:
```text
cretoxy-poster-manager/
  docs/BUILD.md                      (this guide)
  scripts/check-supabase.mjs
  supabase/schema.sql                (copy of the SQL from Section 2)
  functions/api/admin/               (Cloudflare Pages Functions, server only)
  workers/qr-redirect/               (the QR Worker from Step 3.8)
  public/_redirects
  src/theme/                         (the three themes)
  src/lib/                           (supabase client, helpers)
  src/components/
  src/pages/admin/
  src/pages/client/
  src/main.tsx  src/App.tsx
  wrangler.toml  package.json  .env  .dev.vars  .gitignore
```
Create the folders:
- macOS / Linux:
```bash
mkdir -p docs scripts supabase functions/api/admin workers src/theme src/lib src/components src/pages/admin src/pages/client
```
- Windows PowerShell:
```powershell
New-Item -ItemType Directory -Force docs, scripts, supabase, functions\api\admin, workers, src\theme, src\lib, src\components, src\pages\admin, src\pages\client
```

**Step 4.5 — Move the Worker into the project.** (Assumes `cretoxy-cf-setup` is next to this folder.)
- macOS / Linux: `cp -r ../cretoxy-cf-setup/qr-redirect workers/` then `rm -rf workers/qr-redirect/node_modules workers/qr-redirect/.wrangler`
- Windows PowerShell: `Copy-Item -Recurse ..\cretoxy-cf-setup\qr-redirect workers\`
Pass: `workers/qr-redirect/wrangler.toml` and `workers/qr-redirect/src/index.ts` exist.

**Step 4.6 — Save this guide and the SQL in the project.** Copy this guide file to `docs/BUILD.md`. Create `supabase/schema.sql` and paste SQL Blocks A to F from Section 2 (in order) for the record.

**Step 4.7 — Create `wrangler.toml` in the project root** (this is the Pages configuration) with exactly this content:
```toml
name = "cretoxy-poster-manager"
pages_build_output_dir = "./dist"
compatibility_date = "2026-10-01"
```

**Step 4.8 — Create `public/_redirects`** with exactly this one line (makes page refreshes work on a single-page app):
```text
/* /index.html 200
```

**Step 4.9 — Replace `.gitignore`** with exactly this:
```text
node_modules
dist
.wrangler
.env
.env.*
.dev.vars
*.log
.DS_Store
poster-manager-secrets.txt
```

**Step 4.10 — Create `.env`** (frontend values; the anon key is public by design). Fill the two values from your secrets note:
```text
VITE_SUPABASE_URL=https://YOUR-PROJECT-REF.supabase.co
VITE_SUPABASE_ANON_KEY=PASTE-ANON-KEY-HERE
```

**Step 4.11 — Create `.dev.vars`** (server-only values used when you test the Pages Functions on your computer; never commit this file):
```text
SUPABASE_URL=https://YOUR-PROJECT-REF.supabase.co
SUPABASE_SERVICE_ROLE_KEY=PASTE-SERVICE-ROLE-KEY-HERE
RESEND_API_KEY=PASTE-RESEND-KEY-HERE
EMAIL_FROM=Cretoxy Poster Manager <noreply@mail.cretoxy.cloud>
```
Also create `workers/qr-redirect/.dev.vars` containing `SUPABASE_SERVICE_ROLE_KEY=...` and `SCAN_HASH_SALT=...` (the same values as in your secrets note).

**Step 4.12 — Set the package scripts.** Open `package.json` and make the `"scripts"` section exactly this (keep everything else):
```json
"scripts": {
  "dev": "vite",
  "build": "tsc -b && vite build",
  "lint": "eslint .",
  "preview": "vite preview",
  "pages:dev": "npm run build && wrangler pages dev dist",
  "check:supabase": "node --env-file=.env scripts/check-supabase.mjs"
}
```
What each does: `dev` = live preview at http://localhost:5173 while you work; `build` = checks types and makes the final site in the `dist` folder (run before every deploy); `lint` = finds code mistakes; `preview` = serves the built site; `pages:dev` = builds, then runs the site together with the `functions/api` server code at http://localhost:8788 (use this to test server features); `check:supabase` = tests your Supabase connection and admin login.

**Step 4.13 — Create `scripts/check-supabase.mjs`** with exactly this content:
```js
import { createClient } from '@supabase/supabase-js';

const url = process.env.VITE_SUPABASE_URL;
const key = process.env.VITE_SUPABASE_ANON_KEY;
const [email, password] = process.argv.slice(2);

if (!url || !key) {
  console.error('Missing VITE_SUPABASE_URL or VITE_SUPABASE_ANON_KEY in .env');
  process.exit(1);
}
if (!email || !password) {
  console.error('Usage: npm run check:supabase -- ADMIN-EMAIL ADMIN-PASSWORD');
  process.exit(1);
}

const supabase = createClient(url, key);
const { data: auth, error: signInError } = await supabase.auth.signInWithPassword({ email, password });
if (signInError) {
  console.error('Sign in failed:', signInError.message);
  process.exit(1);
}
const { data: me, error: readError } = await supabase.from('users').select('role, full_name').eq('id', auth.user.id).single();
if (readError) {
  console.error('Could not read your users row:', readError.message);
  process.exit(1);
}
console.log('OK. Signed in. Your role is:', me.role);
```

**Step 4.14 — Run the checks.**
```bash
npm run check:supabase -- ADMIN-EMAIL ADMIN-PASSWORD
npm run dev
```
Pass: the first prints `OK. Signed in. Your role is: admin`. The second prints a `Local: http://localhost:5173/` address; open it and you see the default Vite page. Press `Ctrl+C` to stop.
```bash
npm run build
```
Pass: ends with "built in ..." and no red errors.

**Step 4.15 — Local setup checklist (answer each yes/no).**
1. Does `node --version` show v20 or higher?
2. Does `npm run check:supabase` print your role as `admin`? (proves Supabase connected, the admin profile row exists and row-level security works)
3. Does `npm run dev` start with no red errors and show a page at http://localhost:5173?
4. Does `npm run build` finish with no errors?
5. Do the files `.env`, `.dev.vars`, `wrangler.toml`, `public/_redirects`, `workers/qr-redirect/src/index.ts` all exist?
6. Is `poster-manager-secrets.txt` outside the project folder?

**Step 4.16 — First commit and push to GitHub.**
```bash
git add .
git status
```
Check that `.env`, `.dev.vars` and `node_modules` are **not** in the list (they must be ignored). Then:
```bash
git commit -m "Scaffold project"
git push -u origin main
```
A browser window may ask you to sign in to GitHub. Pass: refresh your GitHub repo page and the files appear.

Creating accounts, signing in as a client and creating records in the real app are tested in Section 5 after the prompts build them (the checklist in Section 7 covers them).

---

## 5. AI Coding Agent Build Prompts

**How to use this section (follow exactly):**
1. Open the folder `cretoxy-poster-manager` in opencode or Antigravity (opencode: run `opencode` in a terminal inside the folder; Antigravity: File → Open Folder).
2. Paste **PROMPT #1** (everything inside the code block). Wait until the agent says it is done.
3. Do the **Test** and the **Save progress** step under the prompt.
4. Only then paste the next prompt. Never skip or reorder prompts. **Prompt #14 is optional; do it last.**
5. Every prompt repeats the full project context on purpose, so the agent never has to remember earlier chats.

**Save progress (run after every prompt passes its test, inside `cretoxy-poster-manager`):**
```bash
npm run build
git add .
git commit -m "Prompt N done"
git push
```
(Replace N with the prompt number.)

### 📋 PROMPT #1 — Project scaffolding, themes, layout, routing
```text
CONTEXT (read this first; it is the full project context)
Project: "Cretoxy Poster Campaign Manager", an Admin + Client web app for Cretoxy's local poster-advertising service. A client buys a poster package (printed A4 colour posters, an installation team, approved placement locations, photo proof and a location list). The app manages clients, vendors, packages, campaigns, placements (approved locations), tracked QR codes and links per zone, creative approvals, proof photos, placement checks, reports, GST invoices, client payments, vendor costs and profit.
Stack: React + Vite + TypeScript single-page app on Cloudflare Pages at https://cx.cretoxy.cloud; Supabase (Postgres, Auth, Storage, row-level security); Cloudflare Pages Functions in functions/api for server-only work; Cloudflare Worker in workers/qr-redirect for QR scans at https://cx.cretoxy.cloud/c/* (the Worker only handles paths starting with /c/; never use /c/ for an app page). Package manager: npm only.
Roles: admin and client only. No public signup and no register page; Admin creates client accounts; clients can change their password after first login. Not in this version: installer app, installer login, WhatsApp automation, lead scraping.
Folder structure: docs/BUILD.md, scripts/, supabase/schema.sql, functions/api/admin/, workers/qr-redirect/, workers/daily-alerts/ (optional, Prompt #14), public/_redirects, src/theme/, src/lib/, src/components/, src/pages/admin/, src/pages/client/, src/main.tsx, src/App.tsx, wrangler.toml, package.json.
Environment variables: frontend VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY (in .env); server-only SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, SCAN_HASH_SALT, RESEND_API_KEY, EMAIL_FROM (in .dev.vars locally, Cloudflare secrets in production). Never put the service role key in frontend code.
Database (already created in Supabase, do NOT recreate; exact names are in supabase/schema.sql): tables users, clients, vendors, packages, campaigns, campaign_finance, placements, creatives, qr_codes, qr_events, proofs, placement_checks, invoices, payments, costs, documents, support_queries, notifications, audit_logs, lead_counts; views client_placements_view, client_checks_view, client_payments_view; function decide_creative(p_creative_id, p_decision, p_comment); private storage buckets proofs, creatives, documents, internal-files with file path patterns proofs = CLIENT_ID/CAMPAIGN_ID/PLACEMENT_ID/filename and the others = CLIENT_ID/CAMPAIGN_ID/filename. Row-level security is on: admin sees everything; a client sees only their own data and can never read vendors, packages, campaign_finance, placements, placement_checks, payments, costs or audit_logs directly (clients use the three views for placements, checks and payments). Do not change the database yourself: if a schema change is needed, write it to supabase/extra-N.sql with comments and tell me to run it in Supabase SQL Editor, New query, Run.
Names to keep exactly: campaign stages Pending, Design, Printing, Installation, Live, Completed, Cancelled. Packages Local Reach, Area Campaign, High-Visibility, Mass Local, and Custom quote. Placement types Notice board, Approved ad space, Partner-shop window, Private wall with permission. Permission statuses Requested, Approved, Rejected. Placement statuses Planned, Installed, Removed, Problem. Proof photo types Wide shot, Close-up, Before, After. QR link format https://cx.cretoxy.cloud/c/CLIENTCODE/CAMPAIGNCODE/ZONECODE. UTM keys utm_source=poster, utm_medium=offline, utm_campaign=campaign_name, utm_content=zone_code, utm_term=area.
Pricing rules: packages (table packages, prices before GST): Local Reach 100 posters ₹5,999 7 days; Area Campaign 250 posters ₹13,999 14 days; High-Visibility 500 posters ₹24,999 30 days; Mass Local 1,000 posters ₹44,999 30 days; 30-day options by quote; anything else is Custom quote. GST default 18 percent, editable per campaign, shown as a separate line; placement fees charged to the client are a separate line item (placement_fee_charged); show money as ₹ with Indian digit grouping (en-IN). revenue = price + placement_fee_charged; gst = revenue x gst_percent / 100; invoice total = revenue + gst; planned cost = print_cost_per_poster x poster_count + planned_labour_cost + planned_transport_cost + planned_placement_fees + planned_design_cost; actual cost = sum of the costs rows of the campaign; profit = revenue - cost (planned or actual), before GST, called contribution, admin-only; margin = profit / revenue x 100. Show a warning (not a block) when poster_count is below 100.
Placement rule: posters go only on authorised noticeboards, approved ad spaces, partner-shop windows or private walls with permission, never on public walls or utility boxes. The database refuses to mark a placement Installed unless permission_status is Approved; show that as a friendly message.
Themes: three CSS-variable themes saved in users.theme as white, black, ocean. Tokens in order: --bg, --surface, --border, --text, --muted, --accent, --accent-dim, --on-accent. white = #FFFFFF, #FFFFFF, #E5E5E5, #0A0A0A, #6B6B6B, #0A0A0A, #333333, #FFFFFF. black (green) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #4ADE80, #3AA962, #0A0A0A. ocean (blue) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #22D3EE, #1CA1B5, #0A0A0A. Status colors: success #22C55E, warning #F59E0B, danger #EF4444. The accent fills primary buttons and the active nav item. White is only white. UI is clean, minimal, dashboard style and mobile-responsive; every list has search, filters, sorting and pagination.

STAGE 1 TASK: scaffolding, themes, layout, routing, config.
The Vite React TypeScript project already exists with packages installed. Do this:
1. Create src/theme/themes.css defining the three themes as CSS variables using the exact tokens above, applied with a data-theme attribute on the html element (values white, black, ocean). Create src/theme/ThemeProvider.tsx with a useTheme hook: default is black if the device prefers dark, otherwise white; the choice is stored in localStorage key cx-theme and (once logged in) also saved to users.theme.
2. Create src/theme/ThemeToggle.tsx: three buttons White, Black, Ocean.
3. Create src/lib/supabase.ts that creates the Supabase client from VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY and throws a clear error if either is missing.
4. Create src/lib/config.ts holding in one place: APP_URL = "https://cx.cretoxy.cloud", QR_BASE_URL = "https://cx.cretoxy.cloud/c", DEFAULT_GST = 18, MIN_POSTERS = 100, CHECK_EVERY_DAYS = 7, PRINT_COST_LOW_VOLUME = 8, PRINT_COST_BULK = 6, BULK_FROM_POSTERS = 500, MAP_TILE_URL (default the standard OpenStreetMap tile URL template) and MAP_ATTRIBUTION (with a code comment that heavy use requires a proper tile provider), PLACEMENT_RULE_TEXT = "Posters go only on authorised noticeboards, approved ad spaces, partner-shop windows or private walls with permission. Never on public walls or utility boxes.", and BANK_DETAILS = a clearly marked placeholder object for wire transfer details (bank name, account name, account number, IFSC) that I will fill in later.
5. Create src/lib/format.ts with formatINR(number) using en-IN grouping, formatDate (Asia/Kolkata), unitRate(price, posterCount), defaultPrintCost(posterCount) (8 below 500 posters, 6 from 500), and a calcCampaign(campaign, finance, costs) helper implementing the pricing formulas above and returning revenue, gstAmount, invoiceTotal, plannedCost, actualCost, plannedProfit, actualProfit, plannedMargin, actualMargin (margins in percent, 0 when revenue is 0).
6. Create reusable components in src/components: AppShell (sidebar on desktop, bottom or drawer nav on mobile, header with ThemeToggle and logout), DataTable (search, sort, filter slot, pagination, CSV export button), StatCard, StatusBadge, EmptyState, ConfirmDialog, Toast, PlacementRuleNotice (shows PLACEMENT_RULE_TEXT).
7. Set up react-router-dom in src/App.tsx with these routes: /login, /admin (dashboard), /admin/clients, /admin/vendors, /admin/packages, /admin/campaigns, /admin/campaigns/:id, /admin/placements, /admin/proofs, /admin/finance, /admin/invoices, /admin/reports, /admin/support, /admin/documents, /admin/audit, /client (dashboard), /client/campaigns/:id, /client/invoices, /client/reports, /client/documents, /client/notifications, /client/support. For now each page can be a titled placeholder using AppShell. There must be NO register or sign-up route. Never create a route that starts with /c/ (that path belongs to the QR Worker).
8. Style everything with plain CSS using the variables (no UI framework). Primary buttons and the active nav item use var(--accent) with var(--on-accent) text.

WHEN YOU ARE DONE: (1) list every file you created or modified and summarize the changes in plain language; (2) run npm run build and fix every error; (3) tell me exactly how to test what you built (URL and click steps, and what a pass looks like); (4) do not start the next stage.
```
**Test:** run `npm run dev`, open http://localhost:5173. Pass: you can click the three theme buttons and the colors change; no sign-up link exists.

### 📋 PROMPT #2 — Auth, roles and admin-created client accounts
```text
CONTEXT (read this first; it is the full project context)
Project: "Cretoxy Poster Campaign Manager", an Admin + Client web app for Cretoxy's local poster-advertising service. A client buys a poster package (printed A4 colour posters, an installation team, approved placement locations, photo proof and a location list). The app manages clients, vendors, packages, campaigns, placements (approved locations), tracked QR codes and links per zone, creative approvals, proof photos, placement checks, reports, GST invoices, client payments, vendor costs and profit.
Stack: React + Vite + TypeScript single-page app on Cloudflare Pages at https://cx.cretoxy.cloud; Supabase (Postgres, Auth, Storage, row-level security); Cloudflare Pages Functions in functions/api for server-only work; Cloudflare Worker in workers/qr-redirect for QR scans at https://cx.cretoxy.cloud/c/* (the Worker only handles paths starting with /c/; never use /c/ for an app page). Package manager: npm only.
Roles: admin and client only. No public signup and no register page; Admin creates client accounts; clients can change their password after first login. Not in this version: installer app, installer login, WhatsApp automation, lead scraping.
Folder structure: docs/BUILD.md, scripts/, supabase/schema.sql, functions/api/admin/, workers/qr-redirect/, workers/daily-alerts/ (optional, Prompt #14), public/_redirects, src/theme/, src/lib/, src/components/, src/pages/admin/, src/pages/client/, src/main.tsx, src/App.tsx, wrangler.toml, package.json.
Environment variables: frontend VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY (in .env); server-only SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, SCAN_HASH_SALT, RESEND_API_KEY, EMAIL_FROM (in .dev.vars locally, Cloudflare secrets in production). Never put the service role key in frontend code.
Database (already created in Supabase, do NOT recreate; exact names are in supabase/schema.sql): tables users, clients, vendors, packages, campaigns, campaign_finance, placements, creatives, qr_codes, qr_events, proofs, placement_checks, invoices, payments, costs, documents, support_queries, notifications, audit_logs, lead_counts; views client_placements_view, client_checks_view, client_payments_view; function decide_creative(p_creative_id, p_decision, p_comment); private storage buckets proofs, creatives, documents, internal-files with file path patterns proofs = CLIENT_ID/CAMPAIGN_ID/PLACEMENT_ID/filename and the others = CLIENT_ID/CAMPAIGN_ID/filename. Row-level security is on: admin sees everything; a client sees only their own data and can never read vendors, packages, campaign_finance, placements, placement_checks, payments, costs or audit_logs directly (clients use the three views for placements, checks and payments). Do not change the database yourself: if a schema change is needed, write it to supabase/extra-N.sql with comments and tell me to run it in Supabase SQL Editor, New query, Run.
Names to keep exactly: campaign stages Pending, Design, Printing, Installation, Live, Completed, Cancelled. Packages Local Reach, Area Campaign, High-Visibility, Mass Local, and Custom quote. Placement types Notice board, Approved ad space, Partner-shop window, Private wall with permission. Permission statuses Requested, Approved, Rejected. Placement statuses Planned, Installed, Removed, Problem. Proof photo types Wide shot, Close-up, Before, After. QR link format https://cx.cretoxy.cloud/c/CLIENTCODE/CAMPAIGNCODE/ZONECODE. UTM keys utm_source=poster, utm_medium=offline, utm_campaign=campaign_name, utm_content=zone_code, utm_term=area.
Pricing rules: packages (table packages, prices before GST): Local Reach 100 posters ₹5,999 7 days; Area Campaign 250 posters ₹13,999 14 days; High-Visibility 500 posters ₹24,999 30 days; Mass Local 1,000 posters ₹44,999 30 days; 30-day options by quote; anything else is Custom quote. GST default 18 percent, editable per campaign, shown as a separate line; placement fees charged to the client are a separate line item (placement_fee_charged); show money as ₹ with Indian digit grouping (en-IN). revenue = price + placement_fee_charged; gst = revenue x gst_percent / 100; invoice total = revenue + gst; planned cost = print_cost_per_poster x poster_count + planned_labour_cost + planned_transport_cost + planned_placement_fees + planned_design_cost; actual cost = sum of the costs rows of the campaign; profit = revenue - cost (planned or actual), before GST, called contribution, admin-only; margin = profit / revenue x 100. Show a warning (not a block) when poster_count is below 100.
Placement rule: posters go only on authorised noticeboards, approved ad spaces, partner-shop windows or private walls with permission, never on public walls or utility boxes. The database refuses to mark a placement Installed unless permission_status is Approved; show that as a friendly message.
Themes: three CSS-variable themes saved in users.theme as white, black, ocean. Tokens in order: --bg, --surface, --border, --text, --muted, --accent, --accent-dim, --on-accent. white = #FFFFFF, #FFFFFF, #E5E5E5, #0A0A0A, #6B6B6B, #0A0A0A, #333333, #FFFFFF. black (green) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #4ADE80, #3AA962, #0A0A0A. ocean (blue) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #22D3EE, #1CA1B5, #0A0A0A. Status colors: success #22C55E, warning #F59E0B, danger #EF4444. The accent fills primary buttons and the active nav item. White is only white. UI is clean, minimal, dashboard style and mobile-responsive; every list has search, filters, sorting and pagination.

STAGE 2 TASK: authentication, role-based access, admin creates client accounts.
Existing: theme system, AppShell, router with placeholder pages, src/lib/supabase.ts. Do this:
1. Build /login (email + password, Supabase Auth signInWithPassword, clear error messages, no register link, no forgot-account signup). After login, read the row in the users table for the signed-in user and send role admin to /admin and role client to /client.
2. Create src/lib/auth.tsx with an AuthProvider and a useAuth hook exposing session, profile (role, client_id, full_name, theme, must_change_password) and signOut. Create route guards: AdminRoute (role admin only) and ClientRoute (role client only); wrong role redirects to its own home. These guards are only for UX; real security is row-level security.
3. First-login password change: if users.must_change_password is true for a client, force them to a /change-password page (new password twice, minimum 8 characters), call supabase.auth.updateUser, then update users.must_change_password to false and continue.
4. Server code (Cloudflare Pages Functions, never frontend): create functions/api/admin/create-client.ts handling POST /api/admin/create-client. It must: read the Authorization Bearer token; verify it with the Supabase auth admin API using SUPABASE_URL and SUPABASE_SERVICE_ROLE_KEY from env; confirm that user has role admin in the users table; otherwise return 403. Then create the Supabase Auth user (email, the admin-supplied temporary password, email_confirm true), insert the clients row (client_code is an uppercase letters-and-digits code up to 8 characters, unique, generated from the business name, business_name, owner_name, email, phone, address, city, area, niche, gst_number, notes, status Active) and insert the users row (role client, client_id, full_name, must_change_password true). If any step fails, roll back what was created and return a clear JSON error.
5. Also create functions/api/admin/reset-client-password.ts (POST /api/admin/reset-client-password) with the same admin check: sets a new temporary password for a client user and sets must_change_password true.
6. Add TypeScript types for the Cloudflare environment (env.d.ts) using @cloudflare/workers-types.
7. Build the admin page /admin/clients minimal form calling create-client so the full flow can be tested (full client management comes in the next stage).

WHEN YOU ARE DONE: (1) list every file you created or modified and summarize the changes in plain language; (2) run npm run build and fix every error; (3) tell me exactly how to test what you built (URL and click steps, and what a pass looks like); (4) do not start the next stage.
```
**Test:** run `npm run pages:dev`, open http://localhost:8788/login, sign in with your admin email. Pass: you land on /admin. Open /admin/clients, create a test client with a temporary password, sign out, sign in as that client: you are forced to change the password, then land on /client, and typing /admin in the address bar sends you back to /client.

### 📋 PROMPT #3 — Clients, Vendors and Packages
```text
CONTEXT (read this first; it is the full project context)
Project: "Cretoxy Poster Campaign Manager", an Admin + Client web app for Cretoxy's local poster-advertising service. A client buys a poster package (printed A4 colour posters, an installation team, approved placement locations, photo proof and a location list). The app manages clients, vendors, packages, campaigns, placements (approved locations), tracked QR codes and links per zone, creative approvals, proof photos, placement checks, reports, GST invoices, client payments, vendor costs and profit.
Stack: React + Vite + TypeScript single-page app on Cloudflare Pages at https://cx.cretoxy.cloud; Supabase (Postgres, Auth, Storage, row-level security); Cloudflare Pages Functions in functions/api for server-only work; Cloudflare Worker in workers/qr-redirect for QR scans at https://cx.cretoxy.cloud/c/* (the Worker only handles paths starting with /c/; never use /c/ for an app page). Package manager: npm only.
Roles: admin and client only. No public signup and no register page; Admin creates client accounts; clients can change their password after first login. Not in this version: installer app, installer login, WhatsApp automation, lead scraping.
Folder structure: docs/BUILD.md, scripts/, supabase/schema.sql, functions/api/admin/, workers/qr-redirect/, workers/daily-alerts/ (optional, Prompt #14), public/_redirects, src/theme/, src/lib/, src/components/, src/pages/admin/, src/pages/client/, src/main.tsx, src/App.tsx, wrangler.toml, package.json.
Environment variables: frontend VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY (in .env); server-only SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, SCAN_HASH_SALT, RESEND_API_KEY, EMAIL_FROM (in .dev.vars locally, Cloudflare secrets in production). Never put the service role key in frontend code.
Database (already created in Supabase, do NOT recreate; exact names are in supabase/schema.sql): tables users, clients, vendors, packages, campaigns, campaign_finance, placements, creatives, qr_codes, qr_events, proofs, placement_checks, invoices, payments, costs, documents, support_queries, notifications, audit_logs, lead_counts; views client_placements_view, client_checks_view, client_payments_view; function decide_creative(p_creative_id, p_decision, p_comment); private storage buckets proofs, creatives, documents, internal-files with file path patterns proofs = CLIENT_ID/CAMPAIGN_ID/PLACEMENT_ID/filename and the others = CLIENT_ID/CAMPAIGN_ID/filename. Row-level security is on: admin sees everything; a client sees only their own data and can never read vendors, packages, campaign_finance, placements, placement_checks, payments, costs or audit_logs directly (clients use the three views for placements, checks and payments). Do not change the database yourself: if a schema change is needed, write it to supabase/extra-N.sql with comments and tell me to run it in Supabase SQL Editor, New query, Run.
Names to keep exactly: campaign stages Pending, Design, Printing, Installation, Live, Completed, Cancelled. Packages Local Reach, Area Campaign, High-Visibility, Mass Local, and Custom quote. Placement types Notice board, Approved ad space, Partner-shop window, Private wall with permission. Permission statuses Requested, Approved, Rejected. Placement statuses Planned, Installed, Removed, Problem. Proof photo types Wide shot, Close-up, Before, After. QR link format https://cx.cretoxy.cloud/c/CLIENTCODE/CAMPAIGNCODE/ZONECODE. UTM keys utm_source=poster, utm_medium=offline, utm_campaign=campaign_name, utm_content=zone_code, utm_term=area.
Pricing rules: packages (table packages, prices before GST): Local Reach 100 posters ₹5,999 7 days; Area Campaign 250 posters ₹13,999 14 days; High-Visibility 500 posters ₹24,999 30 days; Mass Local 1,000 posters ₹44,999 30 days; 30-day options by quote; anything else is Custom quote. GST default 18 percent, editable per campaign, shown as a separate line; placement fees charged to the client are a separate line item (placement_fee_charged); show money as ₹ with Indian digit grouping (en-IN). revenue = price + placement_fee_charged; gst = revenue x gst_percent / 100; invoice total = revenue + gst; planned cost = print_cost_per_poster x poster_count + planned_labour_cost + planned_transport_cost + planned_placement_fees + planned_design_cost; actual cost = sum of the costs rows of the campaign; profit = revenue - cost (planned or actual), before GST, called contribution, admin-only; margin = profit / revenue x 100. Show a warning (not a block) when poster_count is below 100.
Placement rule: posters go only on authorised noticeboards, approved ad spaces, partner-shop windows or private walls with permission, never on public walls or utility boxes. The database refuses to mark a placement Installed unless permission_status is Approved; show that as a friendly message.
Themes: three CSS-variable themes saved in users.theme as white, black, ocean. Tokens in order: --bg, --surface, --border, --text, --muted, --accent, --accent-dim, --on-accent. white = #FFFFFF, #FFFFFF, #E5E5E5, #0A0A0A, #6B6B6B, #0A0A0A, #333333, #FFFFFF. black (green) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #4ADE80, #3AA962, #0A0A0A. ocean (blue) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #22D3EE, #1CA1B5, #0A0A0A. Status colors: success #22C55E, warning #F59E0B, danger #EF4444. The accent fills primary buttons and the active nav item. White is only white. UI is clean, minimal, dashboard style and mobile-responsive; every list has search, filters, sorting and pagination.

STAGE 3 TASK: client management, the vendor directory and the package catalog (admin pages).
Existing: theme system, AppShell, DataTable, auth with admin and client guards, functions/api/admin/create-client.ts and reset-client-password.ts. Do this:
1. /admin/clients: DataTable of clients with search, status filter (Active, Paused, Completed, Churned), sorting, pagination, CSV export. Add/edit form with business_name, owner_name, email, phone/WhatsApp, address, city, area, niche, gst_number, notes, status. Creating a client uses the create-client function (temporary password field, shown once with a copy button and a copy-message button that copies a plain text welcome message with login link https://cx.cretoxy.cloud/login and temporary password for sending over WhatsApp by hand). Edit and status changes update the clients table directly. Row action: Reset password (calls reset-client-password). Show each client's assigned campaigns.
2. /admin/vendors: DataTable of vendors with search, filters by vendor_type (Printer, Installer, Designer, Venue partner, Other) and status (Active, Inactive), sort, pagination, CSV export. Add/edit form with vendor_type, name, contact_person, phone, email, area, rate_notes, payment_details, status, notes. A vendor detail view showing the costs linked to that vendor (amount, pay_status, paid_amount) with total paid and total unpaid, the placements where the vendor is the installer, and a place for documents (installer agreement) that will be wired up in a later stage (show a disabled placeholder now).
3. /admin/packages: list of packages with name, poster_count, duration_days, price (before GST), unit rate per poster (price divided by poster_count, shown to 2 decimals), inclusions (editable list of lines), active switch and sort_order. Admin can edit and add packages; there is no delete (set inactive instead). Show a note: "Prices are before GST. Anything not matching a package is a Custom quote." The four seeded packages (Local Reach, Area Campaign, High-Visibility, Mass Local) must appear.
4. Handle errors from Supabase with toast messages. Validate all forms with clear messages.

WHEN YOU ARE DONE: (1) list every file you created or modified and summarize the changes in plain language; (2) run npm run build and fix every error; (3) tell me exactly how to test what you built (URL and click steps, and what a pass looks like); (4) do not start the next stage.
```
**Test:** Pass: you can add, edit, search and export clients and vendors; the four packages show with unit rates; a client you create can sign in; a duplicate package name shows a clear error.

### 📋 PROMPT #4 — Campaigns, placements, stage tracker
```text
CONTEXT (read this first; it is the full project context)
Project: "Cretoxy Poster Campaign Manager", an Admin + Client web app for Cretoxy's local poster-advertising service. A client buys a poster package (printed A4 colour posters, an installation team, approved placement locations, photo proof and a location list). The app manages clients, vendors, packages, campaigns, placements (approved locations), tracked QR codes and links per zone, creative approvals, proof photos, placement checks, reports, GST invoices, client payments, vendor costs and profit.
Stack: React + Vite + TypeScript single-page app on Cloudflare Pages at https://cx.cretoxy.cloud; Supabase (Postgres, Auth, Storage, row-level security); Cloudflare Pages Functions in functions/api for server-only work; Cloudflare Worker in workers/qr-redirect for QR scans at https://cx.cretoxy.cloud/c/* (the Worker only handles paths starting with /c/; never use /c/ for an app page). Package manager: npm only.
Roles: admin and client only. No public signup and no register page; Admin creates client accounts; clients can change their password after first login. Not in this version: installer app, installer login, WhatsApp automation, lead scraping.
Folder structure: docs/BUILD.md, scripts/, supabase/schema.sql, functions/api/admin/, workers/qr-redirect/, workers/daily-alerts/ (optional, Prompt #14), public/_redirects, src/theme/, src/lib/, src/components/, src/pages/admin/, src/pages/client/, src/main.tsx, src/App.tsx, wrangler.toml, package.json.
Environment variables: frontend VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY (in .env); server-only SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, SCAN_HASH_SALT, RESEND_API_KEY, EMAIL_FROM (in .dev.vars locally, Cloudflare secrets in production). Never put the service role key in frontend code.
Database (already created in Supabase, do NOT recreate; exact names are in supabase/schema.sql): tables users, clients, vendors, packages, campaigns, campaign_finance, placements, creatives, qr_codes, qr_events, proofs, placement_checks, invoices, payments, costs, documents, support_queries, notifications, audit_logs, lead_counts; views client_placements_view, client_checks_view, client_payments_view; function decide_creative(p_creative_id, p_decision, p_comment); private storage buckets proofs, creatives, documents, internal-files with file path patterns proofs = CLIENT_ID/CAMPAIGN_ID/PLACEMENT_ID/filename and the others = CLIENT_ID/CAMPAIGN_ID/filename. Row-level security is on: admin sees everything; a client sees only their own data and can never read vendors, packages, campaign_finance, placements, placement_checks, payments, costs or audit_logs directly (clients use the three views for placements, checks and payments). Do not change the database yourself: if a schema change is needed, write it to supabase/extra-N.sql with comments and tell me to run it in Supabase SQL Editor, New query, Run.
Names to keep exactly: campaign stages Pending, Design, Printing, Installation, Live, Completed, Cancelled. Packages Local Reach, Area Campaign, High-Visibility, Mass Local, and Custom quote. Placement types Notice board, Approved ad space, Partner-shop window, Private wall with permission. Permission statuses Requested, Approved, Rejected. Placement statuses Planned, Installed, Removed, Problem. Proof photo types Wide shot, Close-up, Before, After. QR link format https://cx.cretoxy.cloud/c/CLIENTCODE/CAMPAIGNCODE/ZONECODE. UTM keys utm_source=poster, utm_medium=offline, utm_campaign=campaign_name, utm_content=zone_code, utm_term=area.
Pricing rules: packages (table packages, prices before GST): Local Reach 100 posters ₹5,999 7 days; Area Campaign 250 posters ₹13,999 14 days; High-Visibility 500 posters ₹24,999 30 days; Mass Local 1,000 posters ₹44,999 30 days; 30-day options by quote; anything else is Custom quote. GST default 18 percent, editable per campaign, shown as a separate line; placement fees charged to the client are a separate line item (placement_fee_charged); show money as ₹ with Indian digit grouping (en-IN). revenue = price + placement_fee_charged; gst = revenue x gst_percent / 100; invoice total = revenue + gst; planned cost = print_cost_per_poster x poster_count + planned_labour_cost + planned_transport_cost + planned_placement_fees + planned_design_cost; actual cost = sum of the costs rows of the campaign; profit = revenue - cost (planned or actual), before GST, called contribution, admin-only; margin = profit / revenue x 100. Show a warning (not a block) when poster_count is below 100.
Placement rule: posters go only on authorised noticeboards, approved ad spaces, partner-shop windows or private walls with permission, never on public walls or utility boxes. The database refuses to mark a placement Installed unless permission_status is Approved; show that as a friendly message.
Themes: three CSS-variable themes saved in users.theme as white, black, ocean. Tokens in order: --bg, --surface, --border, --text, --muted, --accent, --accent-dim, --on-accent. white = #FFFFFF, #FFFFFF, #E5E5E5, #0A0A0A, #6B6B6B, #0A0A0A, #333333, #FFFFFF. black (green) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #4ADE80, #3AA962, #0A0A0A. ocean (blue) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #22D3EE, #1CA1B5, #0A0A0A. Status colors: success #22C55E, warning #F59E0B, danger #EF4444. The accent fills primary buttons and the active nav item. White is only white. UI is clean, minimal, dashboard style and mobile-responsive; every list has search, filters, sorting and pagination.

STAGE 4 TASK: campaigns, internal finance fields, placements (the location list), stage tracker.
Existing: theme, AppShell, DataTable, auth guards, admin pages for clients, vendors and packages, calcCampaign helper in src/lib/format.ts, src/lib/config.ts. Do this:
1. /admin/campaigns: DataTable with search, filters (stage, client, package_name), sort, pagination, CSV export, a progress bar per campaign based on the stage position (Pending 0 percent up to Completed 100 percent; Cancelled shown as cancelled).
2. Create/edit campaign form: client (select), name, campaign_code (auto-generated unique uppercase letters-and-digits code up to 8 characters from the name, editable before first save), package (select from active packages plus Custom quote; choosing a package fills poster_count, duration_days and price and sets package_name, and every one of these stays editable; Custom quote sets package_name to Custom quote), start_date, end_date (suggested as start_date plus duration_days minus 1, editable, must not be before start_date), target_areas (tag input), price (before GST), placement_fee_charged (separate line, default 0), gst_percent (default 18), qr_destination_type (WhatsApp, Landing page, Form, Booking page, Offer page), qr_destination_url, notes. Internal fields saved to campaign_finance: print_cost_per_poster (default from defaultPrintCost and follows poster_count until the admin edits it), planned_labour_cost, planned_transport_cost, planned_placement_fees, planned_design_cost. Show live as the form changes: unit rate per poster, revenue, GST, invoice total, planned cost, planned contribution and margin. Show a clear warning (not a block) when poster_count is below 100 (MIN_POSTERS).
3. /admin/campaigns/:id detail page with tabs: Overview, Placements, Stage, QR (placeholder for now), Proofs (placeholder), Map and Checks (placeholder), Finance (placeholder).
4. Placements tab (also a cross-campaign list at /admin/placements with the same filters): table with placement_code, location_name, placement_type, area, poster_count, permission_status, status, installer, venue_fee. Add/edit form with location_name, placement_type (Notice board, Approved ad space, Partner-shop window, Private wall with permission), area (pick from the campaign target_areas or type), address, gps_lat, gps_lng, contact_name, contact_phone, permission_status (Requested, Approved, Rejected), permission_note, venue_fee, poster_count, installer (select from vendors of type Installer), notes. placement_code is generated per campaign as P001, P002 and so on. Always show the PlacementRuleNotice at the top of the tab.
5. Placement actions: Approve and Reject permission buttons; Mark Installed (sets status Installed and installed_on today; if the database refuses because permission is not Approved, show the friendly message "Permission must be Approved before posters can be installed here"); Mark Removed (status Removed, removed_on today); a bulk action for selected rows. CSV import of a whole location list with papaparse: columns location_name, placement_type, area, address, poster_count, contact_name, contact_phone, venue_fee, gps_lat, gps_lng; show a preview, list invalid rows (for example a placement_type that is not one of the four) and import only after I confirm. CSV export of the location list.
6. Totals bar on the Placements tab: posters planned (sum of poster_count), posters installed (sum for status Installed), posters ordered (campaign poster_count), and a warning when planned does not equal ordered.
7. Stage tab: buttons to move the campaign through Pending, Design, Printing, Installation, Live, Completed, Cancelled, with the progress bar. Clients will see the stage but cannot change it. Moving to Live shows a warning (not a block) if no placement is Installed yet. Moving to Completed or Cancelled asks me to confirm marking every Installed placement as Removed (sets removed_on today).

WHEN YOU ARE DONE: (1) list every file you created or modified and summarize the changes in plain language; (2) run npm run build and fix every error; (3) tell me exactly how to test what you built (URL and click steps, and what a pass looks like); (4) do not start the next stage.
```
**Test:** Pass: create a client, a campaign from the Local Reach package (see 100 posters, ₹5,999, 7 days and the live totals; change poster_count to 60 and see the minimum warning), add three placements, try Mark Installed while permission is Requested (friendly message), Approve then Mark Installed (works), move the stage to Live and then Completed and confirm the removal.

### 📋 PROMPT #5 — QR generator, Worker redirect, link builder, analytics
```text
CONTEXT (read this first; it is the full project context)
Project: "Cretoxy Poster Campaign Manager", an Admin + Client web app for Cretoxy's local poster-advertising service. A client buys a poster package (printed A4 colour posters, an installation team, approved placement locations, photo proof and a location list). The app manages clients, vendors, packages, campaigns, placements (approved locations), tracked QR codes and links per zone, creative approvals, proof photos, placement checks, reports, GST invoices, client payments, vendor costs and profit.
Stack: React + Vite + TypeScript single-page app on Cloudflare Pages at https://cx.cretoxy.cloud; Supabase (Postgres, Auth, Storage, row-level security); Cloudflare Pages Functions in functions/api for server-only work; Cloudflare Worker in workers/qr-redirect for QR scans at https://cx.cretoxy.cloud/c/* (the Worker only handles paths starting with /c/; never use /c/ for an app page). Package manager: npm only.
Roles: admin and client only. No public signup and no register page; Admin creates client accounts; clients can change their password after first login. Not in this version: installer app, installer login, WhatsApp automation, lead scraping.
Folder structure: docs/BUILD.md, scripts/, supabase/schema.sql, functions/api/admin/, workers/qr-redirect/, workers/daily-alerts/ (optional, Prompt #14), public/_redirects, src/theme/, src/lib/, src/components/, src/pages/admin/, src/pages/client/, src/main.tsx, src/App.tsx, wrangler.toml, package.json.
Environment variables: frontend VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY (in .env); server-only SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, SCAN_HASH_SALT, RESEND_API_KEY, EMAIL_FROM (in .dev.vars locally, Cloudflare secrets in production). Never put the service role key in frontend code.
Database (already created in Supabase, do NOT recreate; exact names are in supabase/schema.sql): tables users, clients, vendors, packages, campaigns, campaign_finance, placements, creatives, qr_codes, qr_events, proofs, placement_checks, invoices, payments, costs, documents, support_queries, notifications, audit_logs, lead_counts; views client_placements_view, client_checks_view, client_payments_view; function decide_creative(p_creative_id, p_decision, p_comment); private storage buckets proofs, creatives, documents, internal-files with file path patterns proofs = CLIENT_ID/CAMPAIGN_ID/PLACEMENT_ID/filename and the others = CLIENT_ID/CAMPAIGN_ID/filename. Row-level security is on: admin sees everything; a client sees only their own data and can never read vendors, packages, campaign_finance, placements, placement_checks, payments, costs or audit_logs directly (clients use the three views for placements, checks and payments). Do not change the database yourself: if a schema change is needed, write it to supabase/extra-N.sql with comments and tell me to run it in Supabase SQL Editor, New query, Run.
Names to keep exactly: campaign stages Pending, Design, Printing, Installation, Live, Completed, Cancelled. Packages Local Reach, Area Campaign, High-Visibility, Mass Local, and Custom quote. Placement types Notice board, Approved ad space, Partner-shop window, Private wall with permission. Permission statuses Requested, Approved, Rejected. Placement statuses Planned, Installed, Removed, Problem. Proof photo types Wide shot, Close-up, Before, After. QR link format https://cx.cretoxy.cloud/c/CLIENTCODE/CAMPAIGNCODE/ZONECODE. UTM keys utm_source=poster, utm_medium=offline, utm_campaign=campaign_name, utm_content=zone_code, utm_term=area.
Pricing rules: packages (table packages, prices before GST): Local Reach 100 posters ₹5,999 7 days; Area Campaign 250 posters ₹13,999 14 days; High-Visibility 500 posters ₹24,999 30 days; Mass Local 1,000 posters ₹44,999 30 days; 30-day options by quote; anything else is Custom quote. GST default 18 percent, editable per campaign, shown as a separate line; placement fees charged to the client are a separate line item (placement_fee_charged); show money as ₹ with Indian digit grouping (en-IN). revenue = price + placement_fee_charged; gst = revenue x gst_percent / 100; invoice total = revenue + gst; planned cost = print_cost_per_poster x poster_count + planned_labour_cost + planned_transport_cost + planned_placement_fees + planned_design_cost; actual cost = sum of the costs rows of the campaign; profit = revenue - cost (planned or actual), before GST, called contribution, admin-only; margin = profit / revenue x 100. Show a warning (not a block) when poster_count is below 100.
Placement rule: posters go only on authorised noticeboards, approved ad spaces, partner-shop windows or private walls with permission, never on public walls or utility boxes. The database refuses to mark a placement Installed unless permission_status is Approved; show that as a friendly message.
Themes: three CSS-variable themes saved in users.theme as white, black, ocean. Tokens in order: --bg, --surface, --border, --text, --muted, --accent, --accent-dim, --on-accent. white = #FFFFFF, #FFFFFF, #E5E5E5, #0A0A0A, #6B6B6B, #0A0A0A, #333333, #FFFFFF. black (green) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #4ADE80, #3AA962, #0A0A0A. ocean (blue) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #22D3EE, #1CA1B5, #0A0A0A. Status colors: success #22C55E, warning #F59E0B, danger #EF4444. The accent fills primary buttons and the active nav item. White is only white. UI is clean, minimal, dashboard style and mobile-responsive; every list has search, filters, sorting and pagination.

STAGE 5 TASK: per-zone QR codes, the Cloudflare Worker that redirects and logs scans, UTM link builder, QR analytics.
Existing: campaigns with placements, campaign detail page with a QR tab placeholder, workers/qr-redirect (placeholder Worker, wrangler.toml with route cx.cretoxy.cloud/c/*, env SUPABASE_URL var, secrets SUPABASE_SERVICE_ROLE_KEY and SCAN_HASH_SALT). Do this:
1. QR tab on /admin/campaigns/:id: I choose one of two modes: (a) one code for the whole campaign (zone_code ALL, area empty) or (b) one code per target area (zone_code = uppercase letters and digits, up to 8 characters, made from the area name, unique inside the campaign, editable before first save). Create (or show) one qr_codes row per zone with client_code, campaign_code, zone_code, area, destination_url (from the campaign qr_destination_url) and tracking_path = CLIENTCODE/CAMPAIGNCODE/ZONECODE. The public link shown and encoded in the QR is QR_BASE_URL (from src/lib/config.ts) + "/" + tracking_path, for example https://cx.cretoxy.cloud/c/CLIENTCODE/CAMPAIGNCODE/ZONECODE. Buttons: generate all QR codes, download PNG, download SVG (use the qrcode package, high error correction, white background, quiet zone), regenerate (updates regenerated_at but keeps the same link so printed codes still work), copy link. Warn that a printed QR can never change its link.
2. Link builder: for each QR show the destination URL with UTM parameters appended exactly utm_source=poster, utm_medium=offline, utm_campaign=<campaign name>, utm_content=<zone code>, utm_term=<area>, with a copy button.
3. Rewrite workers/qr-redirect/src/index.ts (complete file): parse the path /c/CLIENTCODE/CAMPAIGNCODE/ZONECODE (uppercase, ignore a trailing slash); look up qr_codes by tracking_path using the Supabase REST API with the service role key; if not found return a small friendly HTML 404 page; otherwise build the destination URL with the UTM parameters (keep any existing query string), return a 302 redirect, and log the scan WITHOUT delaying the redirect (use ctx.waitUntil) by inserting a qr_events row with event_type scan, device_type (mobile, tablet or desktop guessed from the user agent), and visitor_hash = SHA-256 of (client IP + user agent + SCAN_HASH_SALT) as hex. Store no raw IP and no other personal data. Handle all errors so a logging failure never breaks the redirect. Update wrangler.toml only if needed and explain any change. Add a small README in workers/qr-redirect explaining deploy.
4. QR analytics (admin, on the QR tab and a /admin/reports section): scans per zone, per area, per date and per campaign; unique scans = count of distinct visitor_hash; repeat scans; counts by event_type (scan, click, whatsapp_click, form_submit); top zones and top areas; cost per scan = campaign revenue (before GST) divided by total scans; cost per lead = campaign revenue (before GST) divided by total leads (leads come from the lead_counts table, which a later stage fills). Use simple charts (plain SVG or CSS bars, no heavy chart library). Add a low-QR-activity alert (campaign Live with fewer than 5 scans in the last 7 days).
5. Add a note in the UI that a scan is not a lead.

WHEN YOU ARE DONE: (1) list every file you created or modified and summarize the changes in plain language; (2) run npm run build and fix every error; (3) tell me exactly how to test what you built (URL and click steps, and what a pass looks like); (4) do not start the next stage.
```
**Test (local):** Pass: the QR tab lists a QR per zone and downloads PNG and SVG, and the link looks like the format above.

**Deploy the Worker now (one command set; run inside `cretoxy-poster-manager`):**
```bash
cd workers/qr-redirect
npx wrangler deploy
cd ../..
```
What it does: publishes the real redirect and scan-logging code to `cx.cretoxy.cloud/c/*`. Pass: prints "Deployed cretoxy-qr-redirect". If it says you are not logged in, run `npx wrangler login` first and repeat.

**Test the real redirect:** in the QR tab copy one zone's link (it needs a campaign with a destination URL such as your WhatsApp link) and open it in a browser. Pass: you are sent to the destination, and in Supabase → **Table Editor** → `qr_events` a new row appears within a few seconds.

### 📋 PROMPT #6 — Proof photos, GPS metadata, gallery
```text
CONTEXT (read this first; it is the full project context)
Project: "Cretoxy Poster Campaign Manager", an Admin + Client web app for Cretoxy's local poster-advertising service. A client buys a poster package (printed A4 colour posters, an installation team, approved placement locations, photo proof and a location list). The app manages clients, vendors, packages, campaigns, placements (approved locations), tracked QR codes and links per zone, creative approvals, proof photos, placement checks, reports, GST invoices, client payments, vendor costs and profit.
Stack: React + Vite + TypeScript single-page app on Cloudflare Pages at https://cx.cretoxy.cloud; Supabase (Postgres, Auth, Storage, row-level security); Cloudflare Pages Functions in functions/api for server-only work; Cloudflare Worker in workers/qr-redirect for QR scans at https://cx.cretoxy.cloud/c/* (the Worker only handles paths starting with /c/; never use /c/ for an app page). Package manager: npm only.
Roles: admin and client only. No public signup and no register page; Admin creates client accounts; clients can change their password after first login. Not in this version: installer app, installer login, WhatsApp automation, lead scraping.
Folder structure: docs/BUILD.md, scripts/, supabase/schema.sql, functions/api/admin/, workers/qr-redirect/, workers/daily-alerts/ (optional, Prompt #14), public/_redirects, src/theme/, src/lib/, src/components/, src/pages/admin/, src/pages/client/, src/main.tsx, src/App.tsx, wrangler.toml, package.json.
Environment variables: frontend VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY (in .env); server-only SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, SCAN_HASH_SALT, RESEND_API_KEY, EMAIL_FROM (in .dev.vars locally, Cloudflare secrets in production). Never put the service role key in frontend code.
Database (already created in Supabase, do NOT recreate; exact names are in supabase/schema.sql): tables users, clients, vendors, packages, campaigns, campaign_finance, placements, creatives, qr_codes, qr_events, proofs, placement_checks, invoices, payments, costs, documents, support_queries, notifications, audit_logs, lead_counts; views client_placements_view, client_checks_view, client_payments_view; function decide_creative(p_creative_id, p_decision, p_comment); private storage buckets proofs, creatives, documents, internal-files with file path patterns proofs = CLIENT_ID/CAMPAIGN_ID/PLACEMENT_ID/filename and the others = CLIENT_ID/CAMPAIGN_ID/filename. Row-level security is on: admin sees everything; a client sees only their own data and can never read vendors, packages, campaign_finance, placements, placement_checks, payments, costs or audit_logs directly (clients use the three views for placements, checks and payments). Do not change the database yourself: if a schema change is needed, write it to supabase/extra-N.sql with comments and tell me to run it in Supabase SQL Editor, New query, Run.
Names to keep exactly: campaign stages Pending, Design, Printing, Installation, Live, Completed, Cancelled. Packages Local Reach, Area Campaign, High-Visibility, Mass Local, and Custom quote. Placement types Notice board, Approved ad space, Partner-shop window, Private wall with permission. Permission statuses Requested, Approved, Rejected. Placement statuses Planned, Installed, Removed, Problem. Proof photo types Wide shot, Close-up, Before, After. QR link format https://cx.cretoxy.cloud/c/CLIENTCODE/CAMPAIGNCODE/ZONECODE. UTM keys utm_source=poster, utm_medium=offline, utm_campaign=campaign_name, utm_content=zone_code, utm_term=area.
Pricing rules: packages (table packages, prices before GST): Local Reach 100 posters ₹5,999 7 days; Area Campaign 250 posters ₹13,999 14 days; High-Visibility 500 posters ₹24,999 30 days; Mass Local 1,000 posters ₹44,999 30 days; 30-day options by quote; anything else is Custom quote. GST default 18 percent, editable per campaign, shown as a separate line; placement fees charged to the client are a separate line item (placement_fee_charged); show money as ₹ with Indian digit grouping (en-IN). revenue = price + placement_fee_charged; gst = revenue x gst_percent / 100; invoice total = revenue + gst; planned cost = print_cost_per_poster x poster_count + planned_labour_cost + planned_transport_cost + planned_placement_fees + planned_design_cost; actual cost = sum of the costs rows of the campaign; profit = revenue - cost (planned or actual), before GST, called contribution, admin-only; margin = profit / revenue x 100. Show a warning (not a block) when poster_count is below 100.
Placement rule: posters go only on authorised noticeboards, approved ad spaces, partner-shop windows or private walls with permission, never on public walls or utility boxes. The database refuses to mark a placement Installed unless permission_status is Approved; show that as a friendly message.
Themes: three CSS-variable themes saved in users.theme as white, black, ocean. Tokens in order: --bg, --surface, --border, --text, --muted, --accent, --accent-dim, --on-accent. white = #FFFFFF, #FFFFFF, #E5E5E5, #0A0A0A, #6B6B6B, #0A0A0A, #333333, #FFFFFF. black (green) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #4ADE80, #3AA962, #0A0A0A. ocean (blue) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #22D3EE, #1CA1B5, #0A0A0A. Status colors: success #22C55E, warning #F59E0B, danger #EF4444. The accent fills primary buttons and the active nav item. White is only white. UI is clean, minimal, dashboard style and mobile-responsive; every list has search, filters, sorting and pagination.

STAGE 6 TASK: proof-of-posting uploads and gallery. Only the admin uploads proof photos.
Existing: campaigns with placements, storage bucket proofs (private), proofs table, jszip already installed. Do this:
1. /admin/proofs and a Proofs tab on /admin/campaigns/:id: upload form for multiple photos at once. For each photo choose the placement (from the campaign placements), photo_type (Wide shot, Close-up, Before, After) and area (prefilled from the placement). Allow multiple photos per placement.
2. Upload each file to the proofs bucket at CLIENT_ID/CAMPAIGN_ID/PLACEMENT_ID/timestamp-filename (compress images to about 1600 px on the longest side and JPEG quality 0.8 in the browser before upload). Read EXIF (date taken and GPS) from the original file in the browser when present and save taken_at, gps_lat, gps_lng; if absent leave them empty and show a small note that GPS is optional. Insert the proofs row with status Pending and uploaded_by. If the photo has GPS and its placement has no GPS yet, offer a button "Use this photo's GPS for the placement".
3. Gallery: grid with filters by client, campaign, placement, date, area, photo_type, status; click to open large; buttons Verify and Reject (status Verified or Rejected). Show per-placement coverage: a placement is complete when it has at least one Verified Wide shot and one Verified Close-up (this rule is a proposal; keep it in one constant). Show a Missing proof alert list: every Installed placement in a Live campaign that has no Verified photo at all.
4. Use signed URLs (createSignedUrl, 1 hour) to display images from the private bucket.
5. Client proof pack: a function on the client campaign page to download a zip of that campaign's VERIFIED proofs (jszip, signed URLs). Build the client page /client/campaigns/:id Proofs section now with a gallery of verified proofs only, labelled with the placement code and location name from client_placements_view.

WHEN YOU ARE DONE: (1) list every file you created or modified and summarize the changes in plain language; (2) run npm run build and fix every error; (3) tell me exactly how to test what you built (URL and click steps, and what a pass looks like); (4) do not start the next stage.
```
**Test:** Pass: upload 3 photos for a placement, they show as Pending, verify them, sign in as that client and see only the verified ones and download the zip; a Pending photo is not visible to the client.

### 📋 PROMPT #7 — Creative approval
```text
CONTEXT (read this first; it is the full project context)
Project: "Cretoxy Poster Campaign Manager", an Admin + Client web app for Cretoxy's local poster-advertising service. A client buys a poster package (printed A4 colour posters, an installation team, approved placement locations, photo proof and a location list). The app manages clients, vendors, packages, campaigns, placements (approved locations), tracked QR codes and links per zone, creative approvals, proof photos, placement checks, reports, GST invoices, client payments, vendor costs and profit.
Stack: React + Vite + TypeScript single-page app on Cloudflare Pages at https://cx.cretoxy.cloud; Supabase (Postgres, Auth, Storage, row-level security); Cloudflare Pages Functions in functions/api for server-only work; Cloudflare Worker in workers/qr-redirect for QR scans at https://cx.cretoxy.cloud/c/* (the Worker only handles paths starting with /c/; never use /c/ for an app page). Package manager: npm only.
Roles: admin and client only. No public signup and no register page; Admin creates client accounts; clients can change their password after first login. Not in this version: installer app, installer login, WhatsApp automation, lead scraping.
Folder structure: docs/BUILD.md, scripts/, supabase/schema.sql, functions/api/admin/, workers/qr-redirect/, workers/daily-alerts/ (optional, Prompt #14), public/_redirects, src/theme/, src/lib/, src/components/, src/pages/admin/, src/pages/client/, src/main.tsx, src/App.tsx, wrangler.toml, package.json.
Environment variables: frontend VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY (in .env); server-only SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, SCAN_HASH_SALT, RESEND_API_KEY, EMAIL_FROM (in .dev.vars locally, Cloudflare secrets in production). Never put the service role key in frontend code.
Database (already created in Supabase, do NOT recreate; exact names are in supabase/schema.sql): tables users, clients, vendors, packages, campaigns, campaign_finance, placements, creatives, qr_codes, qr_events, proofs, placement_checks, invoices, payments, costs, documents, support_queries, notifications, audit_logs, lead_counts; views client_placements_view, client_checks_view, client_payments_view; function decide_creative(p_creative_id, p_decision, p_comment); private storage buckets proofs, creatives, documents, internal-files with file path patterns proofs = CLIENT_ID/CAMPAIGN_ID/PLACEMENT_ID/filename and the others = CLIENT_ID/CAMPAIGN_ID/filename. Row-level security is on: admin sees everything; a client sees only their own data and can never read vendors, packages, campaign_finance, placements, placement_checks, payments, costs or audit_logs directly (clients use the three views for placements, checks and payments). Do not change the database yourself: if a schema change is needed, write it to supabase/extra-N.sql with comments and tell me to run it in Supabase SQL Editor, New query, Run.
Names to keep exactly: campaign stages Pending, Design, Printing, Installation, Live, Completed, Cancelled. Packages Local Reach, Area Campaign, High-Visibility, Mass Local, and Custom quote. Placement types Notice board, Approved ad space, Partner-shop window, Private wall with permission. Permission statuses Requested, Approved, Rejected. Placement statuses Planned, Installed, Removed, Problem. Proof photo types Wide shot, Close-up, Before, After. QR link format https://cx.cretoxy.cloud/c/CLIENTCODE/CAMPAIGNCODE/ZONECODE. UTM keys utm_source=poster, utm_medium=offline, utm_campaign=campaign_name, utm_content=zone_code, utm_term=area.
Pricing rules: packages (table packages, prices before GST): Local Reach 100 posters ₹5,999 7 days; Area Campaign 250 posters ₹13,999 14 days; High-Visibility 500 posters ₹24,999 30 days; Mass Local 1,000 posters ₹44,999 30 days; 30-day options by quote; anything else is Custom quote. GST default 18 percent, editable per campaign, shown as a separate line; placement fees charged to the client are a separate line item (placement_fee_charged); show money as ₹ with Indian digit grouping (en-IN). revenue = price + placement_fee_charged; gst = revenue x gst_percent / 100; invoice total = revenue + gst; planned cost = print_cost_per_poster x poster_count + planned_labour_cost + planned_transport_cost + planned_placement_fees + planned_design_cost; actual cost = sum of the costs rows of the campaign; profit = revenue - cost (planned or actual), before GST, called contribution, admin-only; margin = profit / revenue x 100. Show a warning (not a block) when poster_count is below 100.
Placement rule: posters go only on authorised noticeboards, approved ad spaces, partner-shop windows or private walls with permission, never on public walls or utility boxes. The database refuses to mark a placement Installed unless permission_status is Approved; show that as a friendly message.
Themes: three CSS-variable themes saved in users.theme as white, black, ocean. Tokens in order: --bg, --surface, --border, --text, --muted, --accent, --accent-dim, --on-accent. white = #FFFFFF, #FFFFFF, #E5E5E5, #0A0A0A, #6B6B6B, #0A0A0A, #333333, #FFFFFF. black (green) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #4ADE80, #3AA962, #0A0A0A. ocean (blue) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #22D3EE, #1CA1B5, #0A0A0A. Status colors: success #22C55E, warning #F59E0B, danger #EF4444. The accent fills primary buttons and the active nav item. White is only white. UI is clean, minimal, dashboard style and mobile-responsive; every list has search, filters, sorting and pagination.

STAGE 7 TASK: creative mockup approval workflow.
Existing: campaigns, creatives table, creatives bucket (private), function decide_creative(p_creative_id, p_decision, p_comment) which is the ONLY way a client can change a creative. Do this:
1. Admin: Creatives tab on /admin/campaigns/:id. Upload a Mockup or a Print-ready file (kind), each upload becomes a new row with version = previous highest version + 1 for that kind; all versions are kept and listed with status and timestamps. Files go to the creatives bucket at CLIENT_ID/CAMPAIGN_ID/filename.
2. Admin can mark a creative Ready for Printing only when its status is Approved (the database enforces this; show a friendly message if it fails). When a Mockup is uploaded, create a notification row for the client (event creative_approval_needed) for the notifications stage.
3. Client: /client/campaigns/:id Creatives section shows mockups (signed URLs), with Approve, Reject and a comment box; call supabase.rpc decide_creative. Show approval or rejection timestamp and comments, and all versions.
4. Print-ready files are visible to the admin only unless approved; clients see mockups.

WHEN YOU ARE DONE: (1) list every file you created or modified and summarize the changes in plain language; (2) run npm run build and fix every error; (3) tell me exactly how to test what you built (URL and click steps, and what a pass looks like); (4) do not start the next stage.
```
**Test:** Pass: upload a mockup as admin; as the client approve it with a comment; as admin mark it Ready for Printing; before approval the Ready button fails with a friendly message.

### 📋 PROMPT #8 — Client payments, vendor costs, profit (admin only)
```text
CONTEXT (read this first; it is the full project context)
Project: "Cretoxy Poster Campaign Manager", an Admin + Client web app for Cretoxy's local poster-advertising service. A client buys a poster package (printed A4 colour posters, an installation team, approved placement locations, photo proof and a location list). The app manages clients, vendors, packages, campaigns, placements (approved locations), tracked QR codes and links per zone, creative approvals, proof photos, placement checks, reports, GST invoices, client payments, vendor costs and profit.
Stack: React + Vite + TypeScript single-page app on Cloudflare Pages at https://cx.cretoxy.cloud; Supabase (Postgres, Auth, Storage, row-level security); Cloudflare Pages Functions in functions/api for server-only work; Cloudflare Worker in workers/qr-redirect for QR scans at https://cx.cretoxy.cloud/c/* (the Worker only handles paths starting with /c/; never use /c/ for an app page). Package manager: npm only.
Roles: admin and client only. No public signup and no register page; Admin creates client accounts; clients can change their password after first login. Not in this version: installer app, installer login, WhatsApp automation, lead scraping.
Folder structure: docs/BUILD.md, scripts/, supabase/schema.sql, functions/api/admin/, workers/qr-redirect/, workers/daily-alerts/ (optional, Prompt #14), public/_redirects, src/theme/, src/lib/, src/components/, src/pages/admin/, src/pages/client/, src/main.tsx, src/App.tsx, wrangler.toml, package.json.
Environment variables: frontend VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY (in .env); server-only SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, SCAN_HASH_SALT, RESEND_API_KEY, EMAIL_FROM (in .dev.vars locally, Cloudflare secrets in production). Never put the service role key in frontend code.
Database (already created in Supabase, do NOT recreate; exact names are in supabase/schema.sql): tables users, clients, vendors, packages, campaigns, campaign_finance, placements, creatives, qr_codes, qr_events, proofs, placement_checks, invoices, payments, costs, documents, support_queries, notifications, audit_logs, lead_counts; views client_placements_view, client_checks_view, client_payments_view; function decide_creative(p_creative_id, p_decision, p_comment); private storage buckets proofs, creatives, documents, internal-files with file path patterns proofs = CLIENT_ID/CAMPAIGN_ID/PLACEMENT_ID/filename and the others = CLIENT_ID/CAMPAIGN_ID/filename. Row-level security is on: admin sees everything; a client sees only their own data and can never read vendors, packages, campaign_finance, placements, placement_checks, payments, costs or audit_logs directly (clients use the three views for placements, checks and payments). Do not change the database yourself: if a schema change is needed, write it to supabase/extra-N.sql with comments and tell me to run it in Supabase SQL Editor, New query, Run.
Names to keep exactly: campaign stages Pending, Design, Printing, Installation, Live, Completed, Cancelled. Packages Local Reach, Area Campaign, High-Visibility, Mass Local, and Custom quote. Placement types Notice board, Approved ad space, Partner-shop window, Private wall with permission. Permission statuses Requested, Approved, Rejected. Placement statuses Planned, Installed, Removed, Problem. Proof photo types Wide shot, Close-up, Before, After. QR link format https://cx.cretoxy.cloud/c/CLIENTCODE/CAMPAIGNCODE/ZONECODE. UTM keys utm_source=poster, utm_medium=offline, utm_campaign=campaign_name, utm_content=zone_code, utm_term=area.
Pricing rules: packages (table packages, prices before GST): Local Reach 100 posters ₹5,999 7 days; Area Campaign 250 posters ₹13,999 14 days; High-Visibility 500 posters ₹24,999 30 days; Mass Local 1,000 posters ₹44,999 30 days; 30-day options by quote; anything else is Custom quote. GST default 18 percent, editable per campaign, shown as a separate line; placement fees charged to the client are a separate line item (placement_fee_charged); show money as ₹ with Indian digit grouping (en-IN). revenue = price + placement_fee_charged; gst = revenue x gst_percent / 100; invoice total = revenue + gst; planned cost = print_cost_per_poster x poster_count + planned_labour_cost + planned_transport_cost + planned_placement_fees + planned_design_cost; actual cost = sum of the costs rows of the campaign; profit = revenue - cost (planned or actual), before GST, called contribution, admin-only; margin = profit / revenue x 100. Show a warning (not a block) when poster_count is below 100.
Placement rule: posters go only on authorised noticeboards, approved ad spaces, partner-shop windows or private walls with permission, never on public walls or utility boxes. The database refuses to mark a placement Installed unless permission_status is Approved; show that as a friendly message.
Themes: three CSS-variable themes saved in users.theme as white, black, ocean. Tokens in order: --bg, --surface, --border, --text, --muted, --accent, --accent-dim, --on-accent. white = #FFFFFF, #FFFFFF, #E5E5E5, #0A0A0A, #6B6B6B, #0A0A0A, #333333, #FFFFFF. black (green) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #4ADE80, #3AA962, #0A0A0A. ocean (blue) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #22D3EE, #1CA1B5, #0A0A0A. Status colors: success #22C55E, warning #F59E0B, danger #EF4444. The accent fills primary buttons and the active nav item. White is only white. UI is clean, minimal, dashboard style and mobile-responsive; every list has search, filters, sorting and pagination.

STAGE 8 TASK: money tracking and profit. Everything here is admin only and must never be visible to clients, except that a client sees their own payments (through client_payments_view) and their own invoices.
Existing: campaigns, campaign_finance, payments, costs, vendors tables, buckets documents and internal-files, calcCampaign helper. Money arrives over WhatsApp by wire transfer, so this is a tracker only; no payment gateway. Do this:
1. /admin/finance with tabs Payments, Costs, Profit.
2. Payments: record payments per campaign (amount, paid_on, method default Wire transfer, utr_reference, proof screenshot uploaded to internal-files at CLIENT_ID/CAMPAIGN_ID/filename, notes). Show per campaign: Advance Received, Balance Due, Partially Paid, Fully Paid, Overdue (computed against invoice total and due dates) and the outstanding amount per client and per campaign.
3. Costs: add costs per campaign with category (Printing, Installation labour, Transport + supplies, Placement fees, Design + coordination, Reprint / damage, QR / tracking, Admin / misc, Sales commission), amount, vendor (optional select from vendors), incurred_on, pay_status (Unpaid, Paid, Partially Paid, On Hold), paid_amount, paid_on, payment_ref, a payment screenshot and a bill (both uploaded to internal-files), notes. Suggest the pay status from paid_amount (0 means Unpaid, less than amount means Partially Paid, equal means Paid). A button "Add planned costs" creates Unpaid cost rows from campaign_finance (Printing = print_cost_per_poster x poster_count, Installation labour, Transport + supplies, Placement fees, Design + coordination, skipping zero amounts) so I can edit them to the real figures. Show a warning when I mark an Installation labour cost Paid and the campaign has no Verified proof ("pay installers only after proof is verified"). Show totals: total cost, paid and unpaid per campaign and per vendor, and a list of all unpaid vendor costs.
4. Profit: per campaign, per client and per month (grouped by the month of the campaign start_date): revenue (price + placement_fee_charged), GST, planned cost, actual cost, planned and actual contribution and margin percent, using calcCampaign. Label the result "Contribution (before GST, sales expenses, overhead, tax, reprints, damage and extra permission costs)". Highlight negative profit in the danger color.
5. CSV export on every table.

WHEN YOU ARE DONE: (1) list every file you created or modified and summarize the changes in plain language; (2) run npm run build and fix every error; (3) tell me exactly how to test what you built (URL and click steps, and what a pass looks like); (4) do not start the next stage.
```
**Test:** Pass: create a Mass Local campaign (1,000 posters, ₹44,999, print cost ₹6) and use Add planned costs with labour 6000, transport 2000, placement fees 5000, design 2000: planned cost is ₹21,000, planned contribution ₹23,999 and margin about 53.3%. Record an advance payment and see the balance due change; sign in as a client and confirm /admin/finance is not reachable and no cost data appears anywhere.

### 📋 PROMPT #9 — Invoices, PDF, reports, lead counts
```text
CONTEXT (read this first; it is the full project context)
Project: "Cretoxy Poster Campaign Manager", an Admin + Client web app for Cretoxy's local poster-advertising service. A client buys a poster package (printed A4 colour posters, an installation team, approved placement locations, photo proof and a location list). The app manages clients, vendors, packages, campaigns, placements (approved locations), tracked QR codes and links per zone, creative approvals, proof photos, placement checks, reports, GST invoices, client payments, vendor costs and profit.
Stack: React + Vite + TypeScript single-page app on Cloudflare Pages at https://cx.cretoxy.cloud; Supabase (Postgres, Auth, Storage, row-level security); Cloudflare Pages Functions in functions/api for server-only work; Cloudflare Worker in workers/qr-redirect for QR scans at https://cx.cretoxy.cloud/c/* (the Worker only handles paths starting with /c/; never use /c/ for an app page). Package manager: npm only.
Roles: admin and client only. No public signup and no register page; Admin creates client accounts; clients can change their password after first login. Not in this version: installer app, installer login, WhatsApp automation, lead scraping.
Folder structure: docs/BUILD.md, scripts/, supabase/schema.sql, functions/api/admin/, workers/qr-redirect/, workers/daily-alerts/ (optional, Prompt #14), public/_redirects, src/theme/, src/lib/, src/components/, src/pages/admin/, src/pages/client/, src/main.tsx, src/App.tsx, wrangler.toml, package.json.
Environment variables: frontend VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY (in .env); server-only SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, SCAN_HASH_SALT, RESEND_API_KEY, EMAIL_FROM (in .dev.vars locally, Cloudflare secrets in production). Never put the service role key in frontend code.
Database (already created in Supabase, do NOT recreate; exact names are in supabase/schema.sql): tables users, clients, vendors, packages, campaigns, campaign_finance, placements, creatives, qr_codes, qr_events, proofs, placement_checks, invoices, payments, costs, documents, support_queries, notifications, audit_logs, lead_counts; views client_placements_view, client_checks_view, client_payments_view; function decide_creative(p_creative_id, p_decision, p_comment); private storage buckets proofs, creatives, documents, internal-files with file path patterns proofs = CLIENT_ID/CAMPAIGN_ID/PLACEMENT_ID/filename and the others = CLIENT_ID/CAMPAIGN_ID/filename. Row-level security is on: admin sees everything; a client sees only their own data and can never read vendors, packages, campaign_finance, placements, placement_checks, payments, costs or audit_logs directly (clients use the three views for placements, checks and payments). Do not change the database yourself: if a schema change is needed, write it to supabase/extra-N.sql with comments and tell me to run it in Supabase SQL Editor, New query, Run.
Names to keep exactly: campaign stages Pending, Design, Printing, Installation, Live, Completed, Cancelled. Packages Local Reach, Area Campaign, High-Visibility, Mass Local, and Custom quote. Placement types Notice board, Approved ad space, Partner-shop window, Private wall with permission. Permission statuses Requested, Approved, Rejected. Placement statuses Planned, Installed, Removed, Problem. Proof photo types Wide shot, Close-up, Before, After. QR link format https://cx.cretoxy.cloud/c/CLIENTCODE/CAMPAIGNCODE/ZONECODE. UTM keys utm_source=poster, utm_medium=offline, utm_campaign=campaign_name, utm_content=zone_code, utm_term=area.
Pricing rules: packages (table packages, prices before GST): Local Reach 100 posters ₹5,999 7 days; Area Campaign 250 posters ₹13,999 14 days; High-Visibility 500 posters ₹24,999 30 days; Mass Local 1,000 posters ₹44,999 30 days; 30-day options by quote; anything else is Custom quote. GST default 18 percent, editable per campaign, shown as a separate line; placement fees charged to the client are a separate line item (placement_fee_charged); show money as ₹ with Indian digit grouping (en-IN). revenue = price + placement_fee_charged; gst = revenue x gst_percent / 100; invoice total = revenue + gst; planned cost = print_cost_per_poster x poster_count + planned_labour_cost + planned_transport_cost + planned_placement_fees + planned_design_cost; actual cost = sum of the costs rows of the campaign; profit = revenue - cost (planned or actual), before GST, called contribution, admin-only; margin = profit / revenue x 100. Show a warning (not a block) when poster_count is below 100.
Placement rule: posters go only on authorised noticeboards, approved ad spaces, partner-shop windows or private walls with permission, never on public walls or utility boxes. The database refuses to mark a placement Installed unless permission_status is Approved; show that as a friendly message.
Themes: three CSS-variable themes saved in users.theme as white, black, ocean. Tokens in order: --bg, --surface, --border, --text, --muted, --accent, --accent-dim, --on-accent. white = #FFFFFF, #FFFFFF, #E5E5E5, #0A0A0A, #6B6B6B, #0A0A0A, #333333, #FFFFFF. black (green) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #4ADE80, #3AA962, #0A0A0A. ocean (blue) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #22D3EE, #1CA1B5, #0A0A0A. Status colors: success #22C55E, warning #F59E0B, danger #EF4444. The accent fills primary buttons and the active nav item. White is only white. UI is clean, minimal, dashboard style and mobile-responsive; every list has search, filters, sorting and pagination.

STAGE 9 TASK: GST invoices and quotes, PDF generation in the browser, weekly, monthly and final client reports, lead counts.
Existing: campaigns, placements, invoices, payments, qr analytics, proofs, documents bucket, jspdf and jspdf-autotable installed. Do this:
1. /admin/invoices: create quotes and GST invoices with invoice_number (auto, unique, format like INV-YYYY-0001 and QT-YYYY-0001, never reused), invoice_type (Quote, Advance Invoice, Final Invoice, Credit Note), status (Draft, Sent, Partially Paid, Paid, Overdue, Cancelled), issue_date, due_date, client_gst_number, line_items (description, quantity, rate), gst_percent, subtotal, gst_amount, total. Prefill line items from the campaign: line 1 = "<package_name>: <poster_count> A4 colour posters, <duration_days> days, installation, photo proof and location list" at the campaign price; line 2 = "Approved placement fees" at placement_fee_charged, only when it is above zero (always a separate line). For Advance and Final invoices I edit the line amounts myself. Automatically show Overdue when due_date has passed and the invoice is unpaid.
2. PDF download of an invoice or quote (jspdf) with the Cretoxy name, client GST number, line items, GST line, total in ₹ en-IN, and the bank wire transfer details read from BANK_DETAILS in src/lib/config.ts. Save a copy of the PDF to the documents bucket and add a documents row (doc_type Invoice, client_id, campaign_id) so the client can download it. Drafts are not visible to clients.
3. Reports: generate Weekly, Monthly and Final PDF campaign reports (client name, campaign name, package, duration, posters ordered and posters installed, number of locations, areas covered, the location list as a table with placement code, location name, type, area, poster count and status, photo proof summary, placement checks summary, QR scans, unique visitors, WhatsApp clicks, leads, cost per lead, campaign status). Never include contact phones, venue fees or costs. Save to the documents bucket with doc_type Campaign report. Admin can download or copy a plain-text summary to paste into WhatsApp by hand (no automation). Clients download only their own reports on /client/reports.
4. Lead counts: on the client campaign page add a Leads section where the client types the number of leads for a reporting period (period_start, period_end, leads; end not before start; insert or update lead_counts for that campaign and period) and sees all periods they entered. Show cost per lead = campaign revenue before GST divided by total leads. Explain in the UI that scans alone do not prove a sale.
5. Client invoices page /client/invoices lists only non-Draft invoices with PDF download, and shows each campaign's payments (from client_payments_view) and the outstanding amount.

WHEN YOU ARE DONE: (1) list every file you created or modified and summarize the changes in plain language; (2) run npm run build and fix every error; (3) tell me exactly how to test what you built (URL and click steps, and what a pass looks like); (4) do not start the next stage.
```
**Test:** Pass: create an invoice from a campaign (the placement fee line appears only when it is above zero), download the PDF, see it as the client; generate a final report PDF; as the client enter lead counts for a period and see cost per lead.

### 📋 PROMPT #10 — Dashboards and alerts
```text
CONTEXT (read this first; it is the full project context)
Project: "Cretoxy Poster Campaign Manager", an Admin + Client web app for Cretoxy's local poster-advertising service. A client buys a poster package (printed A4 colour posters, an installation team, approved placement locations, photo proof and a location list). The app manages clients, vendors, packages, campaigns, placements (approved locations), tracked QR codes and links per zone, creative approvals, proof photos, placement checks, reports, GST invoices, client payments, vendor costs and profit.
Stack: React + Vite + TypeScript single-page app on Cloudflare Pages at https://cx.cretoxy.cloud; Supabase (Postgres, Auth, Storage, row-level security); Cloudflare Pages Functions in functions/api for server-only work; Cloudflare Worker in workers/qr-redirect for QR scans at https://cx.cretoxy.cloud/c/* (the Worker only handles paths starting with /c/; never use /c/ for an app page). Package manager: npm only.
Roles: admin and client only. No public signup and no register page; Admin creates client accounts; clients can change their password after first login. Not in this version: installer app, installer login, WhatsApp automation, lead scraping.
Folder structure: docs/BUILD.md, scripts/, supabase/schema.sql, functions/api/admin/, workers/qr-redirect/, workers/daily-alerts/ (optional, Prompt #14), public/_redirects, src/theme/, src/lib/, src/components/, src/pages/admin/, src/pages/client/, src/main.tsx, src/App.tsx, wrangler.toml, package.json.
Environment variables: frontend VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY (in .env); server-only SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, SCAN_HASH_SALT, RESEND_API_KEY, EMAIL_FROM (in .dev.vars locally, Cloudflare secrets in production). Never put the service role key in frontend code.
Database (already created in Supabase, do NOT recreate; exact names are in supabase/schema.sql): tables users, clients, vendors, packages, campaigns, campaign_finance, placements, creatives, qr_codes, qr_events, proofs, placement_checks, invoices, payments, costs, documents, support_queries, notifications, audit_logs, lead_counts; views client_placements_view, client_checks_view, client_payments_view; function decide_creative(p_creative_id, p_decision, p_comment); private storage buckets proofs, creatives, documents, internal-files with file path patterns proofs = CLIENT_ID/CAMPAIGN_ID/PLACEMENT_ID/filename and the others = CLIENT_ID/CAMPAIGN_ID/filename. Row-level security is on: admin sees everything; a client sees only their own data and can never read vendors, packages, campaign_finance, placements, placement_checks, payments, costs or audit_logs directly (clients use the three views for placements, checks and payments). Do not change the database yourself: if a schema change is needed, write it to supabase/extra-N.sql with comments and tell me to run it in Supabase SQL Editor, New query, Run.
Names to keep exactly: campaign stages Pending, Design, Printing, Installation, Live, Completed, Cancelled. Packages Local Reach, Area Campaign, High-Visibility, Mass Local, and Custom quote. Placement types Notice board, Approved ad space, Partner-shop window, Private wall with permission. Permission statuses Requested, Approved, Rejected. Placement statuses Planned, Installed, Removed, Problem. Proof photo types Wide shot, Close-up, Before, After. QR link format https://cx.cretoxy.cloud/c/CLIENTCODE/CAMPAIGNCODE/ZONECODE. UTM keys utm_source=poster, utm_medium=offline, utm_campaign=campaign_name, utm_content=zone_code, utm_term=area.
Pricing rules: packages (table packages, prices before GST): Local Reach 100 posters ₹5,999 7 days; Area Campaign 250 posters ₹13,999 14 days; High-Visibility 500 posters ₹24,999 30 days; Mass Local 1,000 posters ₹44,999 30 days; 30-day options by quote; anything else is Custom quote. GST default 18 percent, editable per campaign, shown as a separate line; placement fees charged to the client are a separate line item (placement_fee_charged); show money as ₹ with Indian digit grouping (en-IN). revenue = price + placement_fee_charged; gst = revenue x gst_percent / 100; invoice total = revenue + gst; planned cost = print_cost_per_poster x poster_count + planned_labour_cost + planned_transport_cost + planned_placement_fees + planned_design_cost; actual cost = sum of the costs rows of the campaign; profit = revenue - cost (planned or actual), before GST, called contribution, admin-only; margin = profit / revenue x 100. Show a warning (not a block) when poster_count is below 100.
Placement rule: posters go only on authorised noticeboards, approved ad spaces, partner-shop windows or private walls with permission, never on public walls or utility boxes. The database refuses to mark a placement Installed unless permission_status is Approved; show that as a friendly message.
Themes: three CSS-variable themes saved in users.theme as white, black, ocean. Tokens in order: --bg, --surface, --border, --text, --muted, --accent, --accent-dim, --on-accent. white = #FFFFFF, #FFFFFF, #E5E5E5, #0A0A0A, #6B6B6B, #0A0A0A, #333333, #FFFFFF. black (green) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #4ADE80, #3AA962, #0A0A0A. ocean (blue) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #22D3EE, #1CA1B5, #0A0A0A. Status colors: success #22C55E, warning #F59E0B, danger #EF4444. The accent fills primary buttons and the active nav item. White is only white. UI is clean, minimal, dashboard style and mobile-responsive; every list has search, filters, sorting and pagination.

STAGE 10 TASK: admin dashboard, client dashboard, alerts.
Existing: all records, QR analytics, proofs, finance, invoices, reports, lead counts. Do this:
1. /admin dashboard: stat cards for total clients, active campaigns (stage Live), posters installed (sum of poster_count of Installed placements), revenue, pending payments (outstanding), unpaid vendor costs, and QR scans (last 7 days). A campaigns-by-stage view (Pending, Design, Printing, Installation, Live, Completed). An Alerts panel with: missing proof (Installed placements with no Verified photo), permissions pending (placements still Requested in campaigns that are not yet Live), campaign ending soon (end_date within 7 days), unpaid or overdue invoices, unpaid vendor costs, placements with status Problem, open repost tasks, checks due (Installed placements with no check in the last CHECK_EVERY_DAYS days for High-Visibility and Mass Local campaigns), campaigns below minimum order (poster_count under 100), low QR activity. Each alert links to the relevant page.
2. /client dashboard: only the signed-in client's campaigns shown as cards with campaign name, duration in days, poster count, target areas, package, spend before GST, stage with progress bar; posters installed out of ordered, locations, QR scans, leads, photo proofs count; invoices and payment status; reports; open support queries. Data comes only through the client-safe sources (campaigns, client_placements_view, client_checks_view, client_payments_view, qr_events, proofs that are Verified, invoices that are not Draft, lead_counts).
3. /client/campaigns/:id: overview, stage tracker (read only), location list (from client_placements_view, no contact phones or venue fees), verified proofs, QR analytics (their own campaign only), creatives, leads, invoices, reports.
4. Make sure no client page ever queries vendors, packages, campaign_finance, placements, placement_checks, payments, costs or audit_logs directly.

WHEN YOU ARE DONE: (1) list every file you created or modified and summarize the changes in plain language; (2) run npm run build and fix every error; (3) tell me exactly how to test what you built (URL and click steps, and what a pass looks like); (4) do not start the next stage.
```
**Test:** Pass: both dashboards load with correct numbers; the client dashboard shows no costs, venue fees, phone numbers or profit anywhere.

### 📋 PROMPT #11 — Location map and placement checks
```text
CONTEXT (read this first; it is the full project context)
Project: "Cretoxy Poster Campaign Manager", an Admin + Client web app for Cretoxy's local poster-advertising service. A client buys a poster package (printed A4 colour posters, an installation team, approved placement locations, photo proof and a location list). The app manages clients, vendors, packages, campaigns, placements (approved locations), tracked QR codes and links per zone, creative approvals, proof photos, placement checks, reports, GST invoices, client payments, vendor costs and profit.
Stack: React + Vite + TypeScript single-page app on Cloudflare Pages at https://cx.cretoxy.cloud; Supabase (Postgres, Auth, Storage, row-level security); Cloudflare Pages Functions in functions/api for server-only work; Cloudflare Worker in workers/qr-redirect for QR scans at https://cx.cretoxy.cloud/c/* (the Worker only handles paths starting with /c/; never use /c/ for an app page). Package manager: npm only.
Roles: admin and client only. No public signup and no register page; Admin creates client accounts; clients can change their password after first login. Not in this version: installer app, installer login, WhatsApp automation, lead scraping.
Folder structure: docs/BUILD.md, scripts/, supabase/schema.sql, functions/api/admin/, workers/qr-redirect/, workers/daily-alerts/ (optional, Prompt #14), public/_redirects, src/theme/, src/lib/, src/components/, src/pages/admin/, src/pages/client/, src/main.tsx, src/App.tsx, wrangler.toml, package.json.
Environment variables: frontend VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY (in .env); server-only SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, SCAN_HASH_SALT, RESEND_API_KEY, EMAIL_FROM (in .dev.vars locally, Cloudflare secrets in production). Never put the service role key in frontend code.
Database (already created in Supabase, do NOT recreate; exact names are in supabase/schema.sql): tables users, clients, vendors, packages, campaigns, campaign_finance, placements, creatives, qr_codes, qr_events, proofs, placement_checks, invoices, payments, costs, documents, support_queries, notifications, audit_logs, lead_counts; views client_placements_view, client_checks_view, client_payments_view; function decide_creative(p_creative_id, p_decision, p_comment); private storage buckets proofs, creatives, documents, internal-files with file path patterns proofs = CLIENT_ID/CAMPAIGN_ID/PLACEMENT_ID/filename and the others = CLIENT_ID/CAMPAIGN_ID/filename. Row-level security is on: admin sees everything; a client sees only their own data and can never read vendors, packages, campaign_finance, placements, placement_checks, payments, costs or audit_logs directly (clients use the three views for placements, checks and payments). Do not change the database yourself: if a schema change is needed, write it to supabase/extra-N.sql with comments and tell me to run it in Supabase SQL Editor, New query, Run.
Names to keep exactly: campaign stages Pending, Design, Printing, Installation, Live, Completed, Cancelled. Packages Local Reach, Area Campaign, High-Visibility, Mass Local, and Custom quote. Placement types Notice board, Approved ad space, Partner-shop window, Private wall with permission. Permission statuses Requested, Approved, Rejected. Placement statuses Planned, Installed, Removed, Problem. Proof photo types Wide shot, Close-up, Before, After. QR link format https://cx.cretoxy.cloud/c/CLIENTCODE/CAMPAIGNCODE/ZONECODE. UTM keys utm_source=poster, utm_medium=offline, utm_campaign=campaign_name, utm_content=zone_code, utm_term=area.
Pricing rules: packages (table packages, prices before GST): Local Reach 100 posters ₹5,999 7 days; Area Campaign 250 posters ₹13,999 14 days; High-Visibility 500 posters ₹24,999 30 days; Mass Local 1,000 posters ₹44,999 30 days; 30-day options by quote; anything else is Custom quote. GST default 18 percent, editable per campaign, shown as a separate line; placement fees charged to the client are a separate line item (placement_fee_charged); show money as ₹ with Indian digit grouping (en-IN). revenue = price + placement_fee_charged; gst = revenue x gst_percent / 100; invoice total = revenue + gst; planned cost = print_cost_per_poster x poster_count + planned_labour_cost + planned_transport_cost + planned_placement_fees + planned_design_cost; actual cost = sum of the costs rows of the campaign; profit = revenue - cost (planned or actual), before GST, called contribution, admin-only; margin = profit / revenue x 100. Show a warning (not a block) when poster_count is below 100.
Placement rule: posters go only on authorised noticeboards, approved ad spaces, partner-shop windows or private walls with permission, never on public walls or utility boxes. The database refuses to mark a placement Installed unless permission_status is Approved; show that as a friendly message.
Themes: three CSS-variable themes saved in users.theme as white, black, ocean. Tokens in order: --bg, --surface, --border, --text, --muted, --accent, --accent-dim, --on-accent. white = #FFFFFF, #FFFFFF, #E5E5E5, #0A0A0A, #6B6B6B, #0A0A0A, #333333, #FFFFFF. black (green) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #4ADE80, #3AA962, #0A0A0A. ocean (blue) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #22D3EE, #1CA1B5, #0A0A0A. Status colors: success #22C55E, warning #F59E0B, danger #EF4444. The accent fills primary buttons and the active nav item. White is only white. UI is clean, minimal, dashboard style and mobile-responsive; every list has search, filters, sorting and pagination.

STAGE 11 TASK: location map, installation progress, placement checks (inspections and problems).
Existing: placements, placement_checks tables, campaigns, proofs, leaflet and react-leaflet installed. Do this:
1. Map and Checks tab on /admin/campaigns/:id: a Leaflet map with a pin for every placement that has gps_lat and gps_lng (color by status: Planned, Installed, Removed, Problem). Map tiles: read MAP_TILE_URL and MAP_ATTRIBUTION from src/lib/config.ts. Admin can set a placement's GPS by clicking the map or typing it. Below the map show the location list and a planned-versus-installed table per area (posters planned, posters installed) and an installation progress bar (posters installed out of campaign poster_count). Clients see the same map and progress for their own campaign using client_placements_view only.
2. Placement checks (admin): record a check for a placement with check_date, result (OK, Poster missing, Damaged or peeling, Removed by owner or authority, Reposted), posters_affected, action_taken, repost_needed, photo (upload to the proofs bucket at CLIENT_ID/CAMPAIGN_ID/PLACEMENT_ID/checks/timestamp-filename), client_notified, notes. Also a quick Check list mode that lists every Installed placement and lets me tap OK or a problem result for many placements in one go.
3. Result rules: Poster missing, Damaged or peeling and Removed by owner or authority set the placement status to Problem (ask me to confirm). Reposted sets the placement status back to Installed (only if permission is Approved) and closes earlier repost tasks for that placement. OK changes nothing.
4. Repost task list: every check with repost_needed true whose placement has no later Reposted check; show it on this tab and feed it to the dashboard alert.
5. Checks due: for High-Visibility and Mass Local campaigns show which Installed placements have had no check in the last CHECK_EVERY_DAYS days.
6. Client view: a summary from client_checks_view (counts by result and a list of placement code, result and date). Clients never see notes, photos or contact details.
7. When a Reposted check is saved, create a notification row for the client (event posters_reposted).

WHEN YOU ARE DONE: (1) list every file you created or modified and summarize the changes in plain language; (2) run npm run build and fix every error; (3) tell me exactly how to test what you built (URL and click steps, and what a pass looks like); (4) do not start the next stage.
```
**Test:** Pass: set GPS on two placements and see the pins as admin and as the client; record a "Poster missing" check (the placement turns Problem and appears in the repost list), then a "Reposted" check (the placement turns Installed and the task closes).

### 📋 PROMPT #12 — Notifications, support, documents, audit log
```text
CONTEXT (read this first; it is the full project context)
Project: "Cretoxy Poster Campaign Manager", an Admin + Client web app for Cretoxy's local poster-advertising service. A client buys a poster package (printed A4 colour posters, an installation team, approved placement locations, photo proof and a location list). The app manages clients, vendors, packages, campaigns, placements (approved locations), tracked QR codes and links per zone, creative approvals, proof photos, placement checks, reports, GST invoices, client payments, vendor costs and profit.
Stack: React + Vite + TypeScript single-page app on Cloudflare Pages at https://cx.cretoxy.cloud; Supabase (Postgres, Auth, Storage, row-level security); Cloudflare Pages Functions in functions/api for server-only work; Cloudflare Worker in workers/qr-redirect for QR scans at https://cx.cretoxy.cloud/c/* (the Worker only handles paths starting with /c/; never use /c/ for an app page). Package manager: npm only.
Roles: admin and client only. No public signup and no register page; Admin creates client accounts; clients can change their password after first login. Not in this version: installer app, installer login, WhatsApp automation, lead scraping.
Folder structure: docs/BUILD.md, scripts/, supabase/schema.sql, functions/api/admin/, workers/qr-redirect/, workers/daily-alerts/ (optional, Prompt #14), public/_redirects, src/theme/, src/lib/, src/components/, src/pages/admin/, src/pages/client/, src/main.tsx, src/App.tsx, wrangler.toml, package.json.
Environment variables: frontend VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY (in .env); server-only SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, SCAN_HASH_SALT, RESEND_API_KEY, EMAIL_FROM (in .dev.vars locally, Cloudflare secrets in production). Never put the service role key in frontend code.
Database (already created in Supabase, do NOT recreate; exact names are in supabase/schema.sql): tables users, clients, vendors, packages, campaigns, campaign_finance, placements, creatives, qr_codes, qr_events, proofs, placement_checks, invoices, payments, costs, documents, support_queries, notifications, audit_logs, lead_counts; views client_placements_view, client_checks_view, client_payments_view; function decide_creative(p_creative_id, p_decision, p_comment); private storage buckets proofs, creatives, documents, internal-files with file path patterns proofs = CLIENT_ID/CAMPAIGN_ID/PLACEMENT_ID/filename and the others = CLIENT_ID/CAMPAIGN_ID/filename. Row-level security is on: admin sees everything; a client sees only their own data and can never read vendors, packages, campaign_finance, placements, placement_checks, payments, costs or audit_logs directly (clients use the three views for placements, checks and payments). Do not change the database yourself: if a schema change is needed, write it to supabase/extra-N.sql with comments and tell me to run it in Supabase SQL Editor, New query, Run.
Names to keep exactly: campaign stages Pending, Design, Printing, Installation, Live, Completed, Cancelled. Packages Local Reach, Area Campaign, High-Visibility, Mass Local, and Custom quote. Placement types Notice board, Approved ad space, Partner-shop window, Private wall with permission. Permission statuses Requested, Approved, Rejected. Placement statuses Planned, Installed, Removed, Problem. Proof photo types Wide shot, Close-up, Before, After. QR link format https://cx.cretoxy.cloud/c/CLIENTCODE/CAMPAIGNCODE/ZONECODE. UTM keys utm_source=poster, utm_medium=offline, utm_campaign=campaign_name, utm_content=zone_code, utm_term=area.
Pricing rules: packages (table packages, prices before GST): Local Reach 100 posters ₹5,999 7 days; Area Campaign 250 posters ₹13,999 14 days; High-Visibility 500 posters ₹24,999 30 days; Mass Local 1,000 posters ₹44,999 30 days; 30-day options by quote; anything else is Custom quote. GST default 18 percent, editable per campaign, shown as a separate line; placement fees charged to the client are a separate line item (placement_fee_charged); show money as ₹ with Indian digit grouping (en-IN). revenue = price + placement_fee_charged; gst = revenue x gst_percent / 100; invoice total = revenue + gst; planned cost = print_cost_per_poster x poster_count + planned_labour_cost + planned_transport_cost + planned_placement_fees + planned_design_cost; actual cost = sum of the costs rows of the campaign; profit = revenue - cost (planned or actual), before GST, called contribution, admin-only; margin = profit / revenue x 100. Show a warning (not a block) when poster_count is below 100.
Placement rule: posters go only on authorised noticeboards, approved ad spaces, partner-shop windows or private walls with permission, never on public walls or utility boxes. The database refuses to mark a placement Installed unless permission_status is Approved; show that as a friendly message.
Themes: three CSS-variable themes saved in users.theme as white, black, ocean. Tokens in order: --bg, --surface, --border, --text, --muted, --accent, --accent-dim, --on-accent. white = #FFFFFF, #FFFFFF, #E5E5E5, #0A0A0A, #6B6B6B, #0A0A0A, #333333, #FFFFFF. black (green) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #4ADE80, #3AA962, #0A0A0A. ocean (blue) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #22D3EE, #1CA1B5, #0A0A0A. Status colors: success #22C55E, warning #F59E0B, danger #EF4444. The accent fills primary buttons and the active nav item. White is only white. UI is clean, minimal, dashboard style and mobile-responsive; every list has search, filters, sorting and pagination.

STAGE 12 TASK: email notifications, support queries, document storage, audit log, client notifications and documents pages.
Existing: notifications, support_queries, documents, audit_logs tables, Resend secrets RESEND_API_KEY and EMAIL_FROM available to Pages Functions, documents bucket. Do this:
1. Notifications (email only; NO WhatsApp automation): create functions/api/admin/notify.ts (POST /api/admin/notify; admin check as in create-client.ts) that sends an email through the Resend API using RESEND_API_KEY and EMAIL_FROM and records a notifications row (channel email, event, payload, status Sent or Failed, sent_at). Client events: campaign_started, creative_approval_needed, report_ready, invoice_generated, payment_due, campaign_completed, posters_reposted. Admin alert events shown in the admin dashboard: missing_proof, permission_pending, campaign_ending_soon, invoice_overdue, vendor_payment_pending, placement_problem, new_client_query, low_qr_activity. Add a Send email button where each event happens and an optional automatic send when the admin changes a stage. For WhatsApp, every notification also has a Copy message button that copies plain text for the admin to paste by hand.
2. Client notifications page /client/notifications: list of the signed-in client's notifications (event, date, status) with the message text.
3. Support queries: /client/support (create a query with subject, message, optional campaign; list own queries with status and admin reply) and /admin/support (list all, reply, set priority Low, Normal, High, set status Open, Resolved, Closed). A new query creates an admin alert new_client_query.
4. Documents: /admin/documents upload and list documents organised by client, campaign and placement with doc_type (Client agreement, Venue permission letter, Installer agreement, Invoice, Receipt, Permit or NOC, Approved creative, Campaign report); store files in the documents bucket (venue permission letters, installer agreements and permits are admin-only by row-level security; internal-files is for bills and screenshots). Wire the vendor detail page placeholder for the installer agreement and add a venue permission letter upload on each placement. Clients see only their client-visible document types on /client/documents (Client agreement, Invoice, Receipt, Approved creative, Campaign report) with download.
5. Audit log: write an audit_logs row for: client created, campaign created or edited, placement permission approved or rejected, placement installed, creative approved or rejected, proof uploaded, invoice created or paid, cost marked paid, report generated. Admin actions insert directly from the app; for client actions that cannot insert into audit_logs directly (creative decisions, lead counts, support queries) write a supabase/extra-N.sql migration with database triggers and tell me to run it. /admin/audit lists entries with filters (user, action, date) and pagination.

WHEN YOU ARE DONE: (1) list every file you created or modified and summarize the changes in plain language; (2) run npm run build and fix every error; (3) tell me exactly how to test what you built (URL and click steps, and what a pass looks like); (4) do not start the next stage.
```
**Test:** Pass: send a test email to yourself from the app (it arrives within a few minutes); a client raises a query and the admin replies; a document uploads and shows for the right client on /client/documents; the audit log lists your actions.

### 📋 PROMPT #13 — Export, error handling, edge cases, final polish
```text
CONTEXT (read this first; it is the full project context)
Project: "Cretoxy Poster Campaign Manager", an Admin + Client web app for Cretoxy's local poster-advertising service. A client buys a poster package (printed A4 colour posters, an installation team, approved placement locations, photo proof and a location list). The app manages clients, vendors, packages, campaigns, placements (approved locations), tracked QR codes and links per zone, creative approvals, proof photos, placement checks, reports, GST invoices, client payments, vendor costs and profit.
Stack: React + Vite + TypeScript single-page app on Cloudflare Pages at https://cx.cretoxy.cloud; Supabase (Postgres, Auth, Storage, row-level security); Cloudflare Pages Functions in functions/api for server-only work; Cloudflare Worker in workers/qr-redirect for QR scans at https://cx.cretoxy.cloud/c/* (the Worker only handles paths starting with /c/; never use /c/ for an app page). Package manager: npm only.
Roles: admin and client only. No public signup and no register page; Admin creates client accounts; clients can change their password after first login. Not in this version: installer app, installer login, WhatsApp automation, lead scraping.
Folder structure: docs/BUILD.md, scripts/, supabase/schema.sql, functions/api/admin/, workers/qr-redirect/, workers/daily-alerts/ (optional, Prompt #14), public/_redirects, src/theme/, src/lib/, src/components/, src/pages/admin/, src/pages/client/, src/main.tsx, src/App.tsx, wrangler.toml, package.json.
Environment variables: frontend VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY (in .env); server-only SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, SCAN_HASH_SALT, RESEND_API_KEY, EMAIL_FROM (in .dev.vars locally, Cloudflare secrets in production). Never put the service role key in frontend code.
Database (already created in Supabase, do NOT recreate; exact names are in supabase/schema.sql): tables users, clients, vendors, packages, campaigns, campaign_finance, placements, creatives, qr_codes, qr_events, proofs, placement_checks, invoices, payments, costs, documents, support_queries, notifications, audit_logs, lead_counts; views client_placements_view, client_checks_view, client_payments_view; function decide_creative(p_creative_id, p_decision, p_comment); private storage buckets proofs, creatives, documents, internal-files with file path patterns proofs = CLIENT_ID/CAMPAIGN_ID/PLACEMENT_ID/filename and the others = CLIENT_ID/CAMPAIGN_ID/filename. Row-level security is on: admin sees everything; a client sees only their own data and can never read vendors, packages, campaign_finance, placements, placement_checks, payments, costs or audit_logs directly (clients use the three views for placements, checks and payments). Do not change the database yourself: if a schema change is needed, write it to supabase/extra-N.sql with comments and tell me to run it in Supabase SQL Editor, New query, Run.
Names to keep exactly: campaign stages Pending, Design, Printing, Installation, Live, Completed, Cancelled. Packages Local Reach, Area Campaign, High-Visibility, Mass Local, and Custom quote. Placement types Notice board, Approved ad space, Partner-shop window, Private wall with permission. Permission statuses Requested, Approved, Rejected. Placement statuses Planned, Installed, Removed, Problem. Proof photo types Wide shot, Close-up, Before, After. QR link format https://cx.cretoxy.cloud/c/CLIENTCODE/CAMPAIGNCODE/ZONECODE. UTM keys utm_source=poster, utm_medium=offline, utm_campaign=campaign_name, utm_content=zone_code, utm_term=area.
Pricing rules: packages (table packages, prices before GST): Local Reach 100 posters ₹5,999 7 days; Area Campaign 250 posters ₹13,999 14 days; High-Visibility 500 posters ₹24,999 30 days; Mass Local 1,000 posters ₹44,999 30 days; 30-day options by quote; anything else is Custom quote. GST default 18 percent, editable per campaign, shown as a separate line; placement fees charged to the client are a separate line item (placement_fee_charged); show money as ₹ with Indian digit grouping (en-IN). revenue = price + placement_fee_charged; gst = revenue x gst_percent / 100; invoice total = revenue + gst; planned cost = print_cost_per_poster x poster_count + planned_labour_cost + planned_transport_cost + planned_placement_fees + planned_design_cost; actual cost = sum of the costs rows of the campaign; profit = revenue - cost (planned or actual), before GST, called contribution, admin-only; margin = profit / revenue x 100. Show a warning (not a block) when poster_count is below 100.
Placement rule: posters go only on authorised noticeboards, approved ad spaces, partner-shop windows or private walls with permission, never on public walls or utility boxes. The database refuses to mark a placement Installed unless permission_status is Approved; show that as a friendly message.
Themes: three CSS-variable themes saved in users.theme as white, black, ocean. Tokens in order: --bg, --surface, --border, --text, --muted, --accent, --accent-dim, --on-accent. white = #FFFFFF, #FFFFFF, #E5E5E5, #0A0A0A, #6B6B6B, #0A0A0A, #333333, #FFFFFF. black (green) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #4ADE80, #3AA962, #0A0A0A. ocean (blue) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #22D3EE, #1CA1B5, #0A0A0A. Status colors: success #22C55E, warning #F59E0B, danger #EF4444. The accent fills primary buttons and the active nav item. White is only white. UI is clean, minimal, dashboard style and mobile-responsive; every list has search, filters, sorting and pagination.

STAGE 13 TASK: data export, error handling, edge cases, final cleanup and polish.
Existing: the complete app from stages 1 to 12. Do this:
1. Data export: CSV export of clients, vendors, campaigns, placements, leads (lead_counts), QR analytics, invoices, payments, costs and reports, with a date-range filter before export (admin). PDF export for campaign reports and invoices already exist; make sure every list or report page has the right export buttons. Clients can export only their own data.
2. Error handling: every Supabase and function call shows a friendly toast on failure with a Retry where useful; add an error boundary; handle expired sessions by redirecting to /login; handle offline state; loading skeletons on every list; empty states on every list.
3. Edge cases: duplicate package name or placement_code; deleting a client, vendor, campaign or placement that is in use (block with a clear message, offer Paused, Inactive, Churned or Removed instead); campaign dates (end_date after start_date); poster_count versus placement totals mismatch warning; trying to install a placement without Approved permission (friendly message); negative or zero amounts; very long text; uploading wrong file types or files over 10 MB; QR link for a zone that no longer exists (the Worker should still show a friendly page); timezones (show Asia/Kolkata); invoice numbers never reused; CSV location list with bad rows.
4. Security review: confirm no component or function exposes SUPABASE_SERVICE_ROLE_KEY to the browser; confirm clients cannot reach admin routes or read vendors, packages, campaign_finance, placements, placement_checks, payments, costs or audit_logs directly; list every place you rely on row-level security and every function that checks role admin.
5. Polish: consistent spacing and typography, accessible contrast in all three themes (White, Black, Ocean), keyboard focus styles, mobile layouts checked at 375 px width, favicon and page titles (app name "Cretoxy Posters"), remove unused code, run npm run lint and fix warnings.
6. Update docs/BUILD.md only if something differs from it, and write a short docs/README-ops.md describing how to deploy the Pages site and the Worker and how to back up data (include a weekly CSV export of clients, campaigns, placements, invoices, payments and costs saved outside Supabase).

WHEN YOU ARE DONE: (1) list every file you created or modified and summarize the changes in plain language; (2) run npm run build and fix every error; (3) tell me exactly how to test what you built (URL and click steps, and what a pass looks like); (4) do not start the next stage.
```
**Test:** Pass: `npm run lint` and `npm run build` finish clean; sign in as admin and as a client on a phone-width browser window and switch all three themes; every list exports a CSV.

### 📋 PROMPT #14 (OPTIONAL, do it last) — Daily alert email and automatic Overdue marking
```text
CONTEXT (read this first; it is the full project context)
Project: "Cretoxy Poster Campaign Manager", an Admin + Client web app for Cretoxy's local poster-advertising service. A client buys a poster package (printed A4 colour posters, an installation team, approved placement locations, photo proof and a location list). The app manages clients, vendors, packages, campaigns, placements (approved locations), tracked QR codes and links per zone, creative approvals, proof photos, placement checks, reports, GST invoices, client payments, vendor costs and profit.
Stack: React + Vite + TypeScript single-page app on Cloudflare Pages at https://cx.cretoxy.cloud; Supabase (Postgres, Auth, Storage, row-level security); Cloudflare Pages Functions in functions/api for server-only work; Cloudflare Worker in workers/qr-redirect for QR scans at https://cx.cretoxy.cloud/c/* (the Worker only handles paths starting with /c/; never use /c/ for an app page). Package manager: npm only.
Roles: admin and client only. No public signup and no register page; Admin creates client accounts; clients can change their password after first login. Not in this version: installer app, installer login, WhatsApp automation, lead scraping.
Folder structure: docs/BUILD.md, scripts/, supabase/schema.sql, functions/api/admin/, workers/qr-redirect/, workers/daily-alerts/ (optional, Prompt #14), public/_redirects, src/theme/, src/lib/, src/components/, src/pages/admin/, src/pages/client/, src/main.tsx, src/App.tsx, wrangler.toml, package.json.
Environment variables: frontend VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY (in .env); server-only SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, SCAN_HASH_SALT, RESEND_API_KEY, EMAIL_FROM (in .dev.vars locally, Cloudflare secrets in production). Never put the service role key in frontend code.
Database (already created in Supabase, do NOT recreate; exact names are in supabase/schema.sql): tables users, clients, vendors, packages, campaigns, campaign_finance, placements, creatives, qr_codes, qr_events, proofs, placement_checks, invoices, payments, costs, documents, support_queries, notifications, audit_logs, lead_counts; views client_placements_view, client_checks_view, client_payments_view; function decide_creative(p_creative_id, p_decision, p_comment); private storage buckets proofs, creatives, documents, internal-files with file path patterns proofs = CLIENT_ID/CAMPAIGN_ID/PLACEMENT_ID/filename and the others = CLIENT_ID/CAMPAIGN_ID/filename. Row-level security is on: admin sees everything; a client sees only their own data and can never read vendors, packages, campaign_finance, placements, placement_checks, payments, costs or audit_logs directly (clients use the three views for placements, checks and payments). Do not change the database yourself: if a schema change is needed, write it to supabase/extra-N.sql with comments and tell me to run it in Supabase SQL Editor, New query, Run.
Names to keep exactly: campaign stages Pending, Design, Printing, Installation, Live, Completed, Cancelled. Packages Local Reach, Area Campaign, High-Visibility, Mass Local, and Custom quote. Placement types Notice board, Approved ad space, Partner-shop window, Private wall with permission. Permission statuses Requested, Approved, Rejected. Placement statuses Planned, Installed, Removed, Problem. Proof photo types Wide shot, Close-up, Before, After. QR link format https://cx.cretoxy.cloud/c/CLIENTCODE/CAMPAIGNCODE/ZONECODE. UTM keys utm_source=poster, utm_medium=offline, utm_campaign=campaign_name, utm_content=zone_code, utm_term=area.
Pricing rules: packages (table packages, prices before GST): Local Reach 100 posters ₹5,999 7 days; Area Campaign 250 posters ₹13,999 14 days; High-Visibility 500 posters ₹24,999 30 days; Mass Local 1,000 posters ₹44,999 30 days; 30-day options by quote; anything else is Custom quote. GST default 18 percent, editable per campaign, shown as a separate line; placement fees charged to the client are a separate line item (placement_fee_charged); show money as ₹ with Indian digit grouping (en-IN). revenue = price + placement_fee_charged; gst = revenue x gst_percent / 100; invoice total = revenue + gst; planned cost = print_cost_per_poster x poster_count + planned_labour_cost + planned_transport_cost + planned_placement_fees + planned_design_cost; actual cost = sum of the costs rows of the campaign; profit = revenue - cost (planned or actual), before GST, called contribution, admin-only; margin = profit / revenue x 100. Show a warning (not a block) when poster_count is below 100.
Placement rule: posters go only on authorised noticeboards, approved ad spaces, partner-shop windows or private walls with permission, never on public walls or utility boxes. The database refuses to mark a placement Installed unless permission_status is Approved; show that as a friendly message.
Themes: three CSS-variable themes saved in users.theme as white, black, ocean. Tokens in order: --bg, --surface, --border, --text, --muted, --accent, --accent-dim, --on-accent. white = #FFFFFF, #FFFFFF, #E5E5E5, #0A0A0A, #6B6B6B, #0A0A0A, #333333, #FFFFFF. black (green) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #4ADE80, #3AA962, #0A0A0A. ocean (blue) = #0A0A0A, #141414, #2A2A2A, #EDEDED, #888888, #22D3EE, #1CA1B5, #0A0A0A. Status colors: success #22C55E, warning #F59E0B, danger #EF4444. The accent fills primary buttons and the active nav item. White is only white. UI is clean, minimal, dashboard style and mobile-responsive; every list has search, filters, sorting and pagination.

STAGE 14 TASK (optional): a small scheduled Worker that runs once a day, marks overdue invoices and emails me one alert digest.
Existing: the complete app from stages 1 to 13. Do this:
1. Create workers/daily-alerts/ with wrangler.toml (name = "cretoxy-daily-alerts", main = "src/index.ts", compatibility_date = "2026-10-01", workers_dev = false, [vars] SUPABASE_URL = "https://YOUR-PROJECT-REF.supabase.co" (leave the placeholder and tell me to replace it), [triggers] crons = ["30 3 * * *"] which is 09:00 in Asia/Kolkata, [observability] enabled = true) and src/index.ts with a scheduled handler. Secrets (never in code): SUPABASE_SERVICE_ROLE_KEY, RESEND_API_KEY, ADMIN_ALERT_EMAIL, EMAIL_FROM.
2. In the handler: (a) set invoices to Overdue where status is Sent or Partially Paid and due_date is before today (Asia/Kolkata); (b) count these alerts using the Supabase REST API with the service role key: Installed placements with no Verified proof in Live campaigns, placements still Requested in campaigns that are not Live, campaigns ending within 7 days, Overdue invoices, unpaid or on-hold vendor costs, placements with status Problem, open repost tasks, Live campaigns with fewer than 5 scans in 7 days, open support queries; (c) if at least one count is above zero, send ONE digest email to ADMIN_ALERT_EMAIL through the Resend API and insert a notifications row (channel email, event daily_digest, payload with the counts, status Sent or Failed).
3. Handle every error so one failing step never stops the others. Add workers/daily-alerts/.dev.vars to .gitignore if it is not covered, and a short README with the deploy and test commands.

WHEN YOU ARE DONE: (1) list every file you created or modified and summarize the changes in plain language; (2) run npm run build and fix every error; (3) tell me exactly how to test what you built (URL and click steps, and what a pass looks like); (4) do not start the next stage.
```
**Deploy and test:** do this later, at Step 6.12. Just continue to Section 6 now; Steps 6.1 to 6.11 come first and Step 6.12 is the last step there.

---

## 6. Deployment Steps

You have three environments. Know which one you are touching:

| Environment | Where it runs | How you start it | How to confirm it |
|---|---|---|---|
| **Local** | Your computer | `npm run dev` (port 5173) or `npm run pages:dev` (port 8788, includes `functions/api`) | The address starts with `http://localhost` |
| **Preview** (optional) | `https://preview.cretoxy-poster-manager.pages.dev` | deploy with `--branch preview` (Step 6.6) | Cloudflare → **Workers & Pages** → `cretoxy-poster-manager` → **Deployments** shows the row marked **Preview** |
| **Production** (the real one) | `https://cx.cretoxy.cloud` and QR scans at `https://cx.cretoxy.cloud/c/...` | deploy with `--branch main` (Step 6.7) | The same **Deployments** page shows the newest row marked **Production** |

**Step 6.1 — Tell Supabase about the production address.**
1. Supabase dashboard → **Authentication** → **URL Configuration**.
2. Change **Site URL** to `https://cx.cretoxy.cloud`.
3. Under **Redirect URLs** keep `http://localhost:5173/**` and click **Add URL** to add `https://cx.cretoxy.cloud/**`. Click **Save**.

**Step 6.2 — Make sure all production secrets are in place.** Run inside `cretoxy-poster-manager`:
```bash
npx wrangler pages secret list --project-name cretoxy-poster-manager
```
Pass: the list shows `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `RESEND_API_KEY`, `EMAIL_FROM`. If one is missing, set it with the command from Step 3.9.
```bash
cd workers/qr-redirect
npx wrangler secret list
cd ../..
```
Pass: shows `SUPABASE_SERVICE_ROLE_KEY` and `SCAN_HASH_SALT`.

**Step 6.3 — Make sure the production `.env` is right.** The two `VITE_` values are baked into the site at build time, so open `.env` and confirm both point to your real Supabase project.

**Step 6.4 — Run the quality checks.**
```bash
npm run lint
npm run build
```
Pass: both finish without errors. (Fix errors first by asking the agent: "npm run build fails with this error: ...".)

**Step 6.5 — Deploy the Worker (QR redirect) to production.**
```bash
cd workers/qr-redirect
npx wrangler deploy
cd ../..
```
Pass: "Deployed cretoxy-qr-redirect" and the route `cx.cretoxy.cloud/c/*`.

**Step 6.6 — (Optional) Deploy a preview copy.** Run from the project root (**not** inside `dist` or `functions`; wrangler must see the `functions` folder next to it):
```bash
npx wrangler pages deploy dist --project-name cretoxy-poster-manager --branch preview
```
It prints a preview URL. ⚠️ Preview deployments do **not** receive the production secrets. To test server features there, open Cloudflare → **Workers & Pages** → `cretoxy-poster-manager` → **Settings** → **Variables and Secrets** → switch to **Preview** → add `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `RESEND_API_KEY`, `EMAIL_FROM` as **Secret**, then deploy again. If you skip preview, go straight to Step 6.7.

**Step 6.7 — Deploy the website to production.** From the project root:
```bash
npm run build
npx wrangler pages deploy dist --project-name cretoxy-poster-manager --branch main
```
What it does: uploads the `dist` folder (the site) plus the `functions` folder (the server code) to the production environment. Pass: "Deployment complete" with a URL.

**Step 6.8 — Verify that production matches the deployment rules (routes, custom domain, bindings all live).**
1. Open `https://cx.cretoxy.cloud` → login page with a padlock.
2. Open `https://cx.cretoxy.cloud/admin/clients` and press refresh. Pass: the page loads (not "404").
3. Terminal checks (Windows: use `curl.exe`, macOS/Linux: `curl`). Replace the domain:
```bash
curl -I https://cx.cretoxy.cloud/
curl -i -X POST https://cx.cretoxy.cloud/api/admin/create-client
curl -I https://cx.cretoxy.cloud/c/NOSUCH/NOSUCH/NOSUCH
```
Pass: the first shows `HTTP/2 200`; the second shows `401` or `403` (the server code rejects calls without an admin login, which proves the Function is live); the third shows the Worker's friendly `404` page (proves the route reaches the Worker, not your other site).
4. Cloudflare dashboard → **Workers & Pages** → `cretoxy-poster-manager` → **Custom domains** shows `cx.cretoxy.cloud` as **Active**; **Workers & Pages** → `cretoxy-qr-redirect` → **Settings** → **Domains & Routes** shows `cx.cretoxy.cloud/c/*`.

**Step 6.9 — Upgrade Supabase to Pro before printing any QR code.** Supabase dashboard → your organization → **Billing** → change plan to **Pro** ($25 per month). This stops the 7-day inactivity pause and adds daily backups kept for 7 days.

**Step 6.10 — Redeploy routine (use this every time you change the app later).**
```bash
npm run lint
npm run build
npx wrangler pages deploy dist --project-name cretoxy-poster-manager --branch main
git add .
git commit -m "Describe your change"
git push
```
If you changed the Worker, also run `cd workers/qr-redirect && npx wrangler deploy` (Step 6.5). If you changed the database, run the new SQL file in Supabase first.

**Step 6.11 — How to roll back if something goes wrong.**
- **Website (Pages):** Cloudflare → **Workers & Pages** → `cretoxy-poster-manager` → **Deployments** → find the last good **Production** deployment → click the three dots (**...**) → **Rollback to this deployment** → confirm. Check: that row is now marked **Production** and `https://cx.cretoxy.cloud` behaves like the old version.
- **QR Worker:** inside `workers/qr-redirect` run `npx wrangler deployments list` to see versions, then `npx wrangler rollback` (choose the previous version). Check: open a real QR link and confirm it redirects and a new row appears in `qr_events`.
- **Code:** `git log --oneline` to find the good commit, `git revert BAD-COMMIT-ID`, then redeploy with Step 6.10.
- **Database:** with Pro, Supabase dashboard → **Database** → **Backups** → restore a daily backup (this overwrites current data, so export CSVs first).

> **Note (alternative, not the main path):** you can instead connect the GitHub repo to Pages for automatic deploys (Workers & Pages → `cretoxy-poster-manager` → **Settings** → **Build & deployments**). The guide uses `wrangler pages deploy` because the `VITE_` values come from your local `.env`.

---

**Step 6.12 — (Optional) Deploy the daily alert Worker (Prompt #14).** Only if you did Prompt #14. First open `workers/daily-alerts/wrangler.toml` and replace `YOUR-PROJECT-REF` with the first part of your Supabase Project URL. Then run:
```bash
cd workers/daily-alerts
npx wrangler secret put SUPABASE_SERVICE_ROLE_KEY
npx wrangler secret put RESEND_API_KEY
npx wrangler secret put ADMIN_ALERT_EMAIL
npx wrangler secret put EMAIL_FROM
npx wrangler deploy
cd ../..
```
Each `secret put` asks you to paste the value. If it asks "There doesn't seem to be a Worker called cretoxy-daily-alerts. Do you want to create a new Worker?" answer **y**. Pass: "Deployed cretoxy-daily-alerts" and a schedule line showing `30 3 * * *` (09:00 India time).

**Test it now without waiting for tomorrow.** Create `workers/daily-alerts/.dev.vars` with the same four secrets plus `SUPABASE_URL`, then run:
```bash
cd workers/daily-alerts
npx wrangler dev --test-scheduled
```
Leave it running, open a second terminal and run:
```bash
curl "http://localhost:8787/__scheduled?cron=30+3+*+*+*"
```
Pass: the first terminal shows the handler ran, and if any alert count is above zero an email arrives. Press `Ctrl+C` to stop.

---

## 7. End-to-End Verification Checklist

Tick each box. "Pass" says what you should see. First create the helper script for the security check.

**Step 7.0 — Create `scripts/check-client-access.mjs`** with exactly this content, and add `"check:client": "node --env-file=.env scripts/check-client-access.mjs"` to the `"scripts"` in `package.json`:
```js
import { createClient } from '@supabase/supabase-js';

const url = process.env.VITE_SUPABASE_URL;
const key = process.env.VITE_SUPABASE_ANON_KEY;
const [email, password] = process.argv.slice(2);
if (!url || !key || !email || !password) {
  console.error('Usage: npm run check:client -- CLIENT-EMAIL CLIENT-PASSWORD');
  process.exit(1);
}

const supabase = createClient(url, key);
const { error: signInError } = await supabase.auth.signInWithPassword({ email, password });
if (signInError) { console.error('Sign in failed:', signInError.message); process.exit(1); }

for (const table of ['vendors', 'packages', 'placements', 'placement_checks', 'payments', 'campaign_finance', 'costs', 'audit_logs', 'users', 'clients', 'campaigns', 'proofs', 'invoices', 'client_placements_view', 'client_checks_view', 'client_payments_view']) {
  const { data, error } = await supabase.from(table).select('*');
  console.log(table.padEnd(24), error ? 'ERROR: ' + error.message : data.length + ' rows visible');
}
console.log("EXPECTED: vendors, packages, placements, placement_checks, payments, campaign_finance, costs, audit_logs = 0 rows (or a permission error). users = 1. clients = 1. The three *_view lines and the others = only this client's own data.");
```

### A. Supabase & DB
- [ ] `npm run check:supabase -- ADMIN-EMAIL ADMIN-PASSWORD` prints `Your role is: admin`. *Pass: connected, admin row exists.*
- [ ] SQL Block H shows 20 tables, all `rowsecurity = true`. *Pass: no `false` rows.*
- [ ] The `packages` table has 4 rows: Local Reach, Area Campaign, High-Visibility, Mass Local with the prices from your chart. *Pass: Supabase → Table Editor → `packages`.*
- [ ] Public signup is off. In Git Bash or a macOS/Linux terminal run (replace the two values):
```bash
curl -s -X POST "https://YOUR-PROJECT-REF.supabase.co/auth/v1/signup" -H "apikey: YOUR-ANON-KEY" -H "Content-Type: application/json" -d '{"email":"nobody@example.com","password":"Test12345!"}'
```
*Pass: the answer says signups are not allowed (a message like "Signups not allowed for this instance").*
- [ ] Table Editor shows rows appearing in `clients`, `campaigns`, `placements`, `qr_events`, `proofs` while you test the features below.

### B. Auth & Security
- [ ] Admin signs in at `https://cx.cretoxy.cloud/login` and lands on `/admin`. *Pass: dashboard with admin menu.*
- [ ] Admin creates a client with a temporary password; the client signs in and is forced to change the password, then lands on `/client`. *Pass: the second login uses the new password.*
- [ ] The login page has no Register or Sign up link.
- [ ] While signed in as the client, typing `/admin` in the address bar returns you to `/client`. *Pass: no admin page ever shows.*
- [ ] `npm run check:client -- CLIENT-EMAIL CLIENT-PASSWORD` matches the EXPECTED line. *Pass: 0 rows for the admin-only tables.*
- [ ] The client pages never show contact phones, venue fees, vendor details, costs or profit.
- [ ] Search the code for the service role key: `git grep -n "SERVICE_ROLE"` shows it only in `functions/`, `workers/`, `.dev.vars` mentions in docs, never under `src/`.

### C. Cloudflare Pages & Domain
- [ ] `https://cx.cretoxy.cloud` loads with a padlock. *Pass: login page.*
- [ ] Refreshing `https://cx.cretoxy.cloud/admin/campaigns` works (no 404).
- [ ] `npx wrangler pages secret list --project-name cretoxy-poster-manager` lists all four secrets.
- [ ] The Function is live: `curl -i -X POST https://cx.cretoxy.cloud/api/admin/create-client` returns 401 or 403.
- [ ] The Worker route is live: `https://cx.cretoxy.cloud/c/NOSUCH/NOSUCH/NOSUCH` shows the friendly 404 page from the Worker, while `https://cx.cretoxy.cloud/login` still shows the app.
- [ ] `npx wrangler secret list` (inside `workers/qr-redirect`) shows `SUPABASE_SERVICE_ROLE_KEY` and `SCAN_HASH_SALT`.

### D. Storage (Supabase Storage; R2 is not used)
- [ ] Upload a proof photo as admin. *Pass: the file appears in Supabase → **Storage** → bucket `proofs` under CLIENT_ID/CAMPAIGN_ID/PLACEMENT_ID.*
- [ ] The photo shows as Pending to admin and is **not** visible to the client until you click Verify.
- [ ] After Verify, the client sees it and can download the proof pack zip.
- [ ] A creative mockup uploads to bucket `creatives`; bills and payment screenshots go to `internal-files` and are never visible to the client.
- [ ] The buckets are all private (Storage → each bucket shows **Private**).

### E. Core Features / Flows
- [ ] **Packages and campaign rules:** a Local Reach campaign fills 100 posters, ₹5,999 and 7 days; changing poster_count to 60 shows the minimum-100 warning; the total, GST (default 18%) and invoice total update live; the placement fee shows as a separate line.
- [ ] **Placement permission rule:** marking a placement Installed while permission is Requested is refused with a friendly message; after Approved it works.
- [ ] **Location list import:** a CSV with a bad `placement_type` row lists the bad row and imports only the good rows after you confirm.
- [ ] **Stage tracker:** moving a campaign Pending → Design → Printing → Installation → Live updates the progress bar; the client sees the stage but has no buttons; Completed offers to mark placements Removed.
- [ ] **QR:** the QR tab creates one code for the whole campaign or one per area, with PNG and SVG download, link format `https://cx.cretoxy.cloud/c/CLIENTCODE/CAMPAIGNCODE/ZONECODE`; opening a link redirects with `utm_source=poster`, `utm_medium=offline`, `utm_campaign`, `utm_content`, `utm_term`; a new `qr_events` row appears each time; unique scans and top zones update.
- [ ] **Creative approval:** the client approves or rejects with a comment; Ready for Printing works only after Approved.
- [ ] **Payments and costs:** recording an advance changes Balance Due; Add planned costs creates cost rows; a vendor cost can be marked Paid; the Mass Local example gives ₹21,000 cost, ₹23,999 contribution and about 53.3% margin; negative profit shows in red.
- [ ] **Invoices and reports:** a GST invoice PDF downloads and shows to the client (not while Draft); a weekly or final report PDF with the location list generates and the client can download it.
- [ ] **Leads:** the client types lead counts for a period and cost per lead shows.
- [ ] **Map and checks:** placement pins show for admin and client; a "Poster missing" check turns the placement to Problem and adds a repost task; a "Reposted" check closes it.
- [ ] **Email:** a test notification arrives in your inbox within about 5 minutes (check spam; Resend shows the send in its dashboard).
- [ ] **Support:** a client query appears for the admin; the reply shows for the client.
- [ ] **Documents and audit:** an uploaded agreement appears under the right client on `/client/documents`; the audit log lists created, uploaded and approved actions.
- [ ] **Client notifications:** `/client/notifications` lists the emails sent to that client.
- [ ] **Exports:** every list downloads a CSV; invoices and reports download PDFs.
- [ ] **Themes:** White is only white; Black is green on black; Ocean is blue on black; the choice is remembered after logout and login.
- [ ] **Mobile:** on a phone-width window (375 px) every page is readable without sideways scrolling.
- [ ] **(Optional) Daily alerts:** the Step 6.12 test run sends a digest email when at least one alert exists.

---

## 8. Troubleshooting

Look for the error text, then follow the numbered causes from most likely to least likely.

**8.1 `Invalid login credentials`**
1. Wrong email or password: retype carefully. 2. The user was created in Authentication but has no row in `users`: run SQL Block G again (admin) or recreate the client in the app. 3. The password was changed: use Reset password in the admin Clients page.

**8.2 `new row violates row-level security policy for table "..."`**
1. You are signed in as a client trying an admin action: sign in as admin. 2. The `users` row for your admin has `role` not `admin`: Supabase → **Table Editor** → `users` → check the `role` column. 3. You ran SQL Block D partly: run it again in a new query (if it says a policy already exists, that table is fine).

**8.3 Lists are empty or `permission denied for table ...`**
1. As a client this is correct for `vendors`, `packages`, `placements`, `placement_checks`, `payments`, `campaign_finance`, `costs`, `audit_logs` (clients read placements, checks and payments only through the three views). 2. As admin: check the `users` row role. 3. SQL Block C functions are missing: rerun it.

**8.4 `Signups not allowed for this instance`**
This is expected and good: public signup is off. Create users only through the admin page.

**8.5 `Missing VITE_SUPABASE_URL` or a blank page after deploying**
1. `.env` was missing or wrong when you ran `npm run build`: fix `.env`, run `npm run build` again and redeploy (Step 6.7). 2. The variable names have a typo: they must be exactly `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY`.

**8.6 404 when refreshing a page like `/admin/clients`**
`public/_redirects` is missing or wrong. It must contain exactly `/* /index.html 200`. Fix, rebuild, redeploy.

**8.7 `You are not authenticated` or `Authentication error [code: 10000]` from wrangler**
1. Run `npx wrangler login` again. 2. Run `npx wrangler whoami` and confirm it is the right Cloudflare account.

**8.8 `Project not found` / wrong project name**
The name must be exactly `cretoxy-poster-manager`. List your projects with `npx wrangler pages project list`.

**8.9 `https://cx.cretoxy.cloud/c/...` shows the app's login page (or a 404) instead of the Worker**
1. The Worker route is missing: Cloudflare → **Workers & Pages** → `cretoxy-qr-redirect` → **Settings** → **Domains & Routes** must show `cx.cretoxy.cloud/c/*`. If not, run `npx wrangler deploy` inside `workers/qr-redirect`. 2. The route in `workers/qr-redirect/wrangler.toml` has a typo: `pattern` must be exactly `cx.cretoxy.cloud/c/*` and `zone_name` exactly `cretoxy.cloud`; fix and redeploy. 3. Another Worker or rule already uses `/c/*`: Cloudflare → **Websites** → `cretoxy.cloud` → **Workers Routes** and remove the conflict. 4. The `cx` DNS record is not **Proxied** (grey cloud): Cloudflare → **DNS** → **Records** → edit `cx` → **Proxied** → Save.

**8.10 `Invalid route` / `Could not resolve zone`**
`zone_name` is not exactly `cretoxy.cloud`, or `cretoxy.cloud` is not a domain in your Cloudflare account, or you are logged in to a different account (`npx wrangler whoami`).

**8.11 A Function returns 500 with `SUPABASE_SERVICE_ROLE_KEY` undefined**
1. The secret is not set: Step 3.9. 2. You set it after the last deploy: redeploy (Step 6.7). 3. You are on a Preview deployment: add Preview secrets (Step 6.6).

**8.12 `/api/admin/create-client` returns 404**
1. You deployed from the wrong folder: run `npx wrangler pages deploy dist ...` from the **project root** so the `functions` folder next to it is uploaded. 2. The file is not at `functions/api/admin/create-client.ts`. 3. The agent used a different path: ask it to list the functions folder.

**8.13 `Failed to fetch`, timeouts, or "project is paused" from Supabase**
1. The Free project paused after 7 days idle: Supabase dashboard → open the project → **Restore project**; then upgrade to Pro (Step 6.9). 2. Wrong `VITE_SUPABASE_URL`. 3. Your internet is down.

**8.14 Email not arriving, Resend error `403` or "domain is not verified"**
1. DNS records not verified: Resend → **Domains** → **Verify DNS Records**; in Cloudflare the records must be **DNS only** (grey cloud). 2. `EMAIL_FROM` uses a domain other than `mail.cretoxy.cloud`. 3. Check the Spam folder. 4. Daily limit of 100 emails reached on the Free plan.

**8.15 SQL error `relation "..." already exists` or `policy "..." already exists`**
You ran a block twice. That item already exists; continue with the next block. If a block fails halfway, copy only the statements that did not run into a new query.

**8.16 QR scans do not appear in `qr_events`**
1. The link's `tracking_path` has no `qr_codes` row: regenerate the QR in the app. 2. Worker secrets are missing: `npx wrangler secret list` inside `workers/qr-redirect`, then set and redeploy. 3. See live errors: inside `workers/qr-redirect` run `npx wrangler tail`, then open a QR link and read the log. 4. The Free plan daily request limit is reached (100,000 per day).

**8.17 Photo upload says `new row violates row-level security policy` (Storage)**
1. You are not signed in as admin. 2. The bucket name is wrong: it must be `proofs`, `creatives`, `documents` or `internal-files`. 3. SQL Block F was not run.

**8.18 `npm run build` fails with TypeScript errors**
Copy the full error and paste it to the agent: "npm run build fails with this error. Fix it without changing behaviour." Then rerun the build.

**8.19 `compatibility_date` error in wrangler**
The date in `wrangler.toml` is in the future. Set it to today's date or earlier (for example `2026-10-01`).

---

**8.20 Marking a placement Installed fails with a "check constraint" error**
This is the safety rule working. Posters can be installed only where permission is Approved. Open the placement, set **Permission** to Approved (only after you really have the permission), then mark it Installed.

**8.21 The daily alert email (Prompt #14) never arrives**
1. The secrets are missing: inside `workers/daily-alerts` run `npx wrangler secret list` and set any missing one (`SUPABASE_SERVICE_ROLE_KEY`, `RESEND_API_KEY`, `ADMIN_ALERT_EMAIL`, `EMAIL_FROM`), then redeploy. 2. There was nothing to report: the email is sent only when at least one alert count is above zero. 3. Read the log: inside `workers/daily-alerts` run `npx wrangler tail`, then trigger a test run (Step 6.12). 4. The `SUPABASE_URL` placeholder in `wrangler.toml` was not replaced.

---

## 9. ⚠️ Open Items / Needs My Decision

Each item has: the decision, the options with pros and cons, and the **default the guide uses** so you can keep building. **BLOCKING** = settle before the first real poster with a QR code is printed. **BEFORE BILLING** = settle before you send the first invoice. **CAN WAIT** = nice to have, decide later. Items 1 and 5 are already decided.

### BLOCKING
1. **Where QR links live: DECIDED.** The app and the QR links share `https://cx.cretoxy.cloud`; QR links use the path `/c/...`. Option not chosen: a separate short domain for QR links (pro: shorter QR, can move the app freely; con: second domain to buy and manage). Printed QR codes can never change their link, so `/c/` on this address is reserved forever. Never use `/c/` for an app page.
2. **Supabase plan.** Options: **Free** (pro: $0; con: pauses after 7 days idle, no backups) or **Pro, $25 per month** (pro: no pausing, 7 days of backups; con: monthly cost). **Default:** Free while building, **upgrade to Pro before the first poster is printed** (Step 6.9).
3. **Written permissions.** Options: (a) written permission from every venue, stored as a Venue permission letter (pro: protects you from removal or penalties; con: slower to set up each placement) or (b) verbal permission only (pro: fast; con: no proof if challenged). **Default:** the app blocks "Installed" until permission is Approved; only you can confirm the permission is real. Also check local municipal, housing-society and property-owner rules.
4. **Agreements.** Decide the wording of the Client agreement, Venue permission letter and Installer agreement. Options: draft them yourself (pro: free; con: legal gaps) or have a lawyer draft templates once (pro: safer; con: one-time cost). **Default:** the app only stores the files.

### BEFORE BILLING
5. **GST on package prices: DECIDED.** The chart prices (₹5,999, ₹13,999, ₹24,999, ₹44,999) are **before GST**; the app adds 18% on top (editable per campaign). Option not chosen: prices already include GST (pro: simpler for clients; con: lowers your take-home by about 15%).
6. **GST registration.** Mandatory above ₹20 lakh yearly turnover (₹10 lakh in special-category states); inter-state supply can trigger it earlier. Options: register now (pro: can charge GST and claim input credit; con: filing work) or wait until the threshold (pro: less admin; con: cannot charge GST before registering). **Default:** invoices show a GST line; confirm with an accountant before sending the first one.
7. **Bank wire details.** Fill `BANK_DETAILS` in `src/lib/config.ts` (account name, number, IFSC); they print on invoice PDFs. No alternative; nothing is paid through the app.
8. **Placement fees.** Options: (a) include them in the package price (pro: simple quote; con: you absorb surprises) or (b) charge them as a separate line (`placement_fee_charged`) (pro: matches the chart's advice, protects margin; con: client sees an extra line). **Default:** (b), default ₹0.
9. **30-day options by quote.** Options: use a Custom quote and type the price (pro: flexible; con: manual) or add extra packages in `/admin/packages` (pro: repeatable; con: more to maintain). **Default:** Custom quote.

### CAN WAIT
10. **Printing cost.** Options: keep the defaults of ₹8 per poster under 500 posters and ₹6 from 500 (pro: matches your working estimate; con: unverified) or enter your printer's real quote (pro: accurate; con: need a quote). Confirm paper, size, colour, gumming, GST and delivery. **Default:** the 8/6 split (the 500 threshold is PROPOSED).
11. **Check frequency.** The chart says "scheduled placement checks" without a number. Options: every 3 days (pro: catches missing posters fast; con: more labour), every 7 days (balanced) or only at the end (pro: cheapest; con: problems found late). **Default:** 7 days (`CHECK_EVERY_DAYS` in `src/lib/config.ts`).
12. **Proof completeness rule.** Options: one verified Wide shot and one verified Close-up per placement (pro: shows context and detail; con: more photos) or any single verified photo (pro: faster; con: weaker proof). **Default:** the first (PROPOSED).
13. **Minimum order.** Options: warn below 100 posters (pro: protects margin; con: may turn away tiny tests) or no minimum (pro: easy sales; con: unprofitable jobs). **Default:** a warning, not a block (not in your chart).
14. **Lead counts.** Options: client types leads per reporting period (pro: fits 7 to 30 day campaigns) or per month (con: does not fit short campaigns). **Default:** per period, with start and end dates.
15. **QR zones.** Options: one code for the whole campaign (pro: one print batch; con: no area breakdown) or one per target area (pro: scans by area; con: separate print batches). **Default:** you choose per campaign.
16. **Names (PROPOSED).** App "Cretoxy Poster Campaign Manager" (page title "Cretoxy Posters"), project and Pages name `cretoxy-poster-manager`, Worker `cretoxy-qr-redirect`, daily Worker `cretoxy-daily-alerts`, theme key `cx-theme`, `utm_source=poster`. Changing any means changing it everywhere.
17. **Email provider.** Options: Resend (pro: simple, free tier of 3,000 emails per month; con: needs DNS records) or another provider (con: different setup). **Default:** Resend, sending domain `mail.cretoxy.cloud`; not yet confirmed by you.
18. **Map tiles.** Options: default OpenStreetMap tiles (pro: free; con: light use only) or a paid tile provider (pro: reliable at scale; con: cost, API key). **Default:** OpenStreetMap.
19. **PDF quality.** Options: browser-generated PDFs (pro: free, no server; con: plain look) or designed or server-side PDFs later (pro: branded; con: more work). **Default:** browser-generated.
20. **WhatsApp message wording** for the copy-message buttons is not written; ask the agent to draft it. No automation is used.
21. **Backups.** Options: Supabase Pro backups only (pro: automatic; con: stays inside Supabase) or Pro plus a weekly CSV export saved elsewhere (pro: safest; con: a weekly chore). **Default:** both; Prompt #13 writes the steps.
22. **Storage.** Options: Supabase Storage (pro: one place, security rules built in; con: size limits on the plan) or Cloudflare R2 (pro: cheap large files; con: extra setup). **Default:** Supabase Storage; R2 is not used.
23. **Installer and printer onboarding.** Options: contacts and payment details only (pro: simple; con: you relay work by phone) or vendor logins later (pro: self-serve proof upload; con: a new role to build). **Default:** contacts only; no vendor login.

---

# Fawateer (فواتير) — Graduation Project Prototype

## 1. Context

- **What:** A university graduation project (GP2), Information Systems Department, Imam Mohammad Ibn Saud Islamic University (IMSIU), Riyadh. Course IS 1480, Section 076, Group 07.
- **Team:** Omar Alsalamah (Backend / AI lead), Mohammed Aljaffan (Frontend / Mobile lead). Supervisor: Dr. Mutlaq Aldajani.
- **Goal of this repo:** build a **working prototype** that is demonstrated live in front of the graduation committee. It is not a production system.
- **Important:** the students must be able to explain every part of the code at the defense. Clarity is more important than cleverness.

## 2. The Problem and the Idea

In Saudi Arabia, receipts are mostly paper. They get lost, expenses are tracked by hand, and warranty and return deadlines are missed. Existing digital options are fragmented: Jarir sends SMS, IKEA keeps receipts in its own app, and Expensify and Wave require scanning every receipt by hand.

**Fawateer** is a **Zero-Touch** digital receipt system. The customer pays by card as usual, and a detailed receipt appears in the Fawateer mobile application automatically. AI then categorizes the expense, extracts warranty and return dates, and reminds the customer before deadlines pass. The customer does nothing.

## 3. Design Update After the GP1 Committee Feedback

The committee raised two concerns:
1. Why would shops share invoice data with a private app?
2. **Privacy:** customer data would be shared with a third party (Fawateer) without control.

**Solution adopted:** a **neutral, trusted third party called the Hub** sits between the shops and Fawateer. In real life, this could be ZATCA, which already receives e-invoices from shops, or a SAMA-licensed body. In the prototype, the Hub is a small simulated service.

- Shops never send data to Fawateer directly.
- The Hub forwards only invoices that belong to registered Fawateer customers who gave consent.
- The Hub removes personal data Fawateer does not need (data minimization).
- The Hub **encrypts sensitive items** (for example, underwear or personal-care products) so that Fawateer's server cannot read them. Only the customer's phone can.

## 4. What the Prototype Simulates vs. Assumes

**Assumed (NOT built):**
- **Card linking with the bank.** The customer's card is already linked. The bank issued a Card Token (PAR, Payment Account Reference, EMVCo standard), and the PAR is already stored in the Hub. The bank is an assumed external service.
- **The payment itself.** There is no real card payment. The Mock POS page creates the invoice directly, and the customer's PAR is chosen from a short list of demo customers.
- **The integrated POS.** The shop's cash register already has both the full invoice and the PAR returned by the payment terminal.
- **Servers are always up.** No retry or failure-recovery logic is needed beyond basic error handling.

**Simulated and built (the demo):**
```
Mock POS web page (select shop + customer + products → "Create Invoice")
   ──► POS database (invoice saved with status NEW)
   ──(Hub reads NEW invoices every few seconds)──► Hub database
   ──(Hub encrypts sensitive items, strips personal data, sends HTTP POST)──► Fawateer backend
   ──(Spring AI: category + warranty date)──► Fawateer database
   ──(Firebase Cloud Messaging push)──► Flutter mobile application (decrypts sensitive items)
```

**Live demo at the defense:**
1. Open the Mock POS page. Choose a shop and a demo customer, add a few products (including one sensitive item), and click "Create Invoice".
2. Show the new invoice in `pos_db` with status `NEW`.
3. Within seconds, the Hub picks it up: it appears in `hub_db` and its status in `pos_db` becomes `SENT`.
4. The phone receives a push notification; the receipt appears categorized, with a warranty countdown if applicable.
5. Show `fawateer_db`: the sensitive item name is encrypted text. Show the phone: the same item appears in clear text.
6. Repeat with a second invoice from another shop.

## 5. Architecture

### 5.1 Components

| Folder | Component | Description |
|---|---|---|
| `database/` | PostgreSQL via Docker Compose | One PostgreSQL instance with three databases: `pos_db`, `hub_db`, `fawateer_db`. Includes seed SQL: shops, products, and demo customers for `pos_db`; registered cards for `hub_db`. |
| `pos/` | Mock POS page (Spring Boot + Thymeleaf) | A very simple single page: select shop, demo customer, and products (with quantities), then "Create Invoice". It calculates subtotal, VAT 15%, and total, and saves the invoice in `pos_db` with status `NEW`. No login, no payment, no inventory. |
| `hub/` | Mock Hub (Spring Boot) | Scheduled job that polls `pos_db`, stores in `hub_db`, encrypts sensitive items, and pushes to Fawateer. A simple read-only status page or clear logs is optional. |
| `backend/` | Fawateer backend (Spring Boot) | REST API for the mobile application, ingest endpoint for the Hub, Spring AI processing, FCM notifications, and the warranty reminder scheduler. |
| `mobile/` | Fawateer mobile application (Flutter) | The customer-facing application. |

### 5.2 Data Flow Decisions

- **POS → Hub: pull.** The Hub polls `pos_db` on a fixed interval (about 2 seconds, so the full flow stays close to the ~5-second target) for invoices with `status = 'NEW'`. This reflects the supervisor's "update cycle" idea, since the POS is just a database.
- **Hub → Fawateer: push.** HTTP POST to `POST /api/ingest/receipts`, authenticated with a shared secret header (API key or HMAC signature). It is instant, which keeps the Zero-Touch promise.
- **Hub filtering:** the Hub forwards only invoices whose PAR exists in `hub_db.registered_cards`. Other invoices are logged as `REJECTED` and never leave the Hub.
- **Fawateer → phone:** Firebase Cloud Messaging push notification.

### 5.3 Databases (Starting Point; Refine When Designing)

- **`pos_db`:** `shops` (name, VAT number, branch), `products` (shop_id, name_ar, name_en, unit_price, sensitive, general_category, warranty_text optional), `demo_customers` (display name, par), `invoices` (id, invoice_number, merchant_name, merchant_vat_number, branch, issue_datetime, subtotal, vat_amount, total, currency, payment_method, par, status NEW/SENT) and `invoice_items` (id, invoice_id, name_ar, name_en, quantity, unit_price, vat_rate 15%, line_total, sensitive boolean, general_category for sensitive items, warranty_text optional).
- **`hub_db`:** `registered_cards` (par, fawateer_user_id, customer_encryption_key, consent_given_at, active), `partner_shops`, and `hub_invoices` / `delivery_log` (pos_invoice_id, par, status FORWARDED/REJECTED, timestamps).
- **`fawateer_db`:** `users`, `receipts` (including `source` = ZERO_TOUCH / SCANNED, and the image path for scanned receipts), `receipt_items` (encrypted name fields for sensitive items), `warranties` (end dates for warranty and return), `notifications` / device FCM tokens, `consents`.

Invoice fields must follow ZATCA Fatoora mandatory fields: merchant name, merchant VAT number, items, quantities, unit prices, VAT amount and rate, total, date, and time. Seed data should look realistic for Saudi retail: Arabic and English item names, SAR, VAT 15%, shops such as electronics, grocery, pharmacy, clothing, and furniture.

## 6. Privacy and Encryption

- **Who sees what:**
  - Normal items: visible to the Hub and Fawateer, because Fawateer needs them for AI and analytics.
  - Sensitive items: visible to the Hub; encrypted in Fawateer; decrypted only in the app.
  - Unneeded personal data (customer name, phone, card details): removed by the Hub.
- **Encryption method (keep it simple and explainable):** AES-256-GCM with a **per-customer key**. For the prototype, the key is pre-generated and stored in only two places: `hub_db.registered_cards` and the customer's mobile application (secure storage). **The Fawateer backend never has the key.** The Hub encrypts the names of sensitive items; the app decrypts them.
- **Sensitive items skip the AI.** They use the `general_category` provided by the shop (e.g., "Personal Care"). The app shows a 🔒 icon on them.
- **Consent:** a one-time consent screen in the app explains what is shared (store, items, prices, date), who sees it, and that sensitive items are encrypted. The user must accept to continue. This aligns with the Saudi Personal Data Protection Law (PDPL).

## 7. AI (Spring AI)

- **Tasks:**
  1. **Expense categorization.** Categories: Food, Groceries, Electronics, Healthcare, Clothing, Home, Transport, Entertainment, Personal Care, Other. If not confident, use `Uncategorized`.
  2. **Warranty and return date extraction.** Returns the period and the end date, or null.
  3. **Receipt image reading (UC5, manual upload).** The model reads a photo of a paper receipt and returns structured JSON (merchant, VAT number if present, date, items, prices, VAT, total, warranty text). This replaces classic OCR: the image goes directly to a multimodal model through Spring AI.
  - All tasks are bilingual (Arabic and English).
- Prompts must ask for **strict JSON output**. Parse safely and fall back gracefully on bad output.
- **Model provider:** must be free. To be decided early, for example a free API tier with good Arabic support, or a local model via Ollama. The chosen model **must accept images** (multimodal) for UC5. Keep the provider behind Spring AI configuration so it can be swapped.
- Prompt engineering is iterative. Keep prompts in separate files or templates so they are easy to show and refine.

## 8. Use Cases (from GP1)

Actors: Customer (primary), Bank, Mock POS (now represented through the Hub), Spring AI Engine.

| UC | Name | Prototype status |
|---|---|---|
| UC1 | Link Card via Bank | **Assumed.** PAR is pre-registered. The app may show the linked card as already linked. |
| UC2 | Receive Real-Time Receipt | **Core.** The Hub → backend flow. «include» UC3, «extend» UC4, extended by UC8. |
| UC3 | Categorize Expense | **Core.** Spring AI. |
| UC4 | Track Warranty & Return Period | **Core.** Spring AI extraction plus a scheduler. Extended by UC9. |
| UC5 | Scan Paper Receipt (Hybrid OCR) | **Core, built after the main Zero-Touch flow.** Photo or gallery image → multimodal AI → structured receipt → user reviews → saved. «include» UC3, «extend» UC4. See Section 8.1. |
| UC6 | View Receipts & Search | **Core.** List, filter by date, merchant, or category, and detail view. |
| UC7 | View Spending Analytics | **Core.** Charts by category and period. |
| UC8 | Receive Receipt Notification | **Core.** FCM push after a new receipt. |
| UC9 | Receive Warranty Alert | **Core.** Scheduled push before a deadline (e.g., 3 days before). |

### 8.1 The Fawateer Mobile Application (the Main Product)

The mobile application is what the customer sees and what the committee judges most. It must feel like a real, polished product: clean, modern, organized, and easy to use in Arabic and English.

**Core idea:** the customer opens the app and their receipts are **already there**: organized, categorized, and searchable, without any effort. Manual upload exists only for paper receipts from shops outside the Zero-Touch flow.

**Screens and features:**

1. **Login / registration + consent.** Simple email and password login (JWT). A one-time consent screen explains data sharing and encryption (see Section 6).
2. **Home (dashboard).**
   - This month's total spending, with a comparison to last month (up or down %).
   - A small breakdown by category.
   - Recent receipts (latest first).
   - **Upcoming deadlines:** warranties and return periods ending soon, with countdowns.
   - A clear "+ Add receipt" button for manual upload.
3. **Receipts list.**
   - Grouped by month (newest first).
   - Each card shows merchant, date, total, and a category icon and color.
   - Search by merchant or item name. Filter by category, date range, and source (automatic / scanned).
4. **Receipt detail.**
   - Merchant, VAT number, date and time, items with quantities and prices, VAT, and total.
   - The AI category, with an option for the user to change it.
   - 🔒 for encrypted sensitive items, decrypted on the device.
   - A warranty and return countdown, if available.
   - A source badge: "Zero-Touch" or "Scanned".
5. **Manual upload (UC5): photo → AI → receipt.**
   - The user takes a photo with the camera or picks one from the gallery.
   - The app uploads it to the backend at `POST /api/receipts/scan` (JWT).
   - The backend sends the image to Spring AI (multimodal model), which returns structured JSON.
   - The backend runs the same categorization and warranty extraction used for Zero-Touch receipts.
   - The app shows a **review screen** with editable fields so the user can fix any AI mistakes. The user then confirms.
   - The receipt is saved like any other receipt, with source = `SCANNED`. The original image is stored and viewable.
   - Scanned receipts do **not** go through the Hub; the customer uploads their own data directly.
6. **Spending analytics (UC7).** Monthly view is the default; weekly is a secondary toggle.
   - Total for the selected month, compared to the previous month.
   - Spending by category (pie or donut chart, with amounts and %).
   - Trend of monthly totals over the last 6 months (bar chart).
   - Top merchants for the selected month.
   - Weekly view: spending per day for the selected week.
   - Tapping a category opens the receipts in that category.
7. **Notifications.**
   - New receipt arrived (UC8). Tapping it opens the receipt.
   - Warranty or return deadline approaching (UC9, e.g., 3 days before).
   - A simple in-app notification list.
8. **Profile / settings.**
   - Linked card shown as "Linked" (PAR assumed).
   - Language toggle (Arabic / English with full RTL).
   - Notification preferences.
   - Logout.

**Design direction:**
- A modern, calm look with consistent category colors and icons.
- Clear empty states and friendly loading states.
- Plain-language error messages.
- Core actions within 3 taps of Home.

## 9. Non-Functional Targets That Affect the Code

- **Security:**
  - JWT authentication for all mobile application APIs.
  - Users can access only their own receipts.
  - Lock the account after 5 failed login attempts.
  - Never store card numbers; only the PAR is stored, and only in the Hub.
  - Shared-secret authentication on the ingest endpoint.
  - No secrets in code; use environment variables.
- **Performance (goals):**
  - Backend processing of an ingested receipt in about 2 seconds.
  - AI processing in about 3 seconds.
  - Notification within seconds of the Hub forwarding.
  - Receipt list loads in about 2 seconds.
- **Usability:**
  - Arabic and English with full RTL support.
  - Core actions within 3 taps of the home screen.
  - Plain-language error messages.
  - Dates in Saudi time (UTC+3).
- **Compatibility:** Android 9+ (main demo target on emulator or device); iOS 14+ if possible.
- **Reliability:** handle malformed payloads without crashing; log errors clearly.
- **Green IT:** use FCM push, not polling, from the app.

## 10. Tech Stack

- **Mock POS, Backend, and Hub:** Java 21, Spring Boot 3, Spring Web, Spring Data JPA, PostgreSQL, Maven. The Mock POS page uses Thymeleaf (server-rendered HTML, no JavaScript framework). The backend adds Spring Security + JWT, Spring AI, and Spring scheduling; the Hub uses Spring scheduling.
- **Mobile:** Flutter (Dart), Firebase Messaging, secure storage for the encryption key and JWT, a charts package for analytics, and localization (AR/EN).
- **Infrastructure:** Docker Compose for PostgreSQL. The demo runs locally; optional free-tier cloud deployment for the backend.
- **Tools:** Git and GitHub. **Free and open-source tools only** (no paid services).

## 11. What the Final Report Needs from the Code

The code must make it easy to produce Chapter 4 and Chapter 5 of the report:
- **Chapter 4:**
  - Activity diagrams for each use case
  - Class diagram
  - Sequence diagrams (especially the full flow from POS DB to phone)
  - ERD after normalization and a data dictionary for `fawateer_db`
  - Component diagram
  - Interface screenshots
- **Chapter 5:**
  - Hardware and software specifications
  - Test cases (purpose, inputs, expected output, pass/fail, result)
  - Deployment diagram
  - Installation steps
- **Testing types:** integration testing (full flow), AI accuracy validation (bilingual sample receipts with expected categories and dates), security testing (JWT, access control, ingest authentication), and UAT.

Keep entities, endpoints, and flows clean and consistent so these diagrams can be derived from the code.

## 12. Out of Scope

- Real banks, real card linking, and real payment networks
- Real ZATCA integration
- Real shops and real POS hardware
- A full cashier system (the Mock POS page is a minimal demo page only: no payments, inventory, or login)
- A merchant dashboard
- Retry and recovery infrastructure
- Production hosting at scale

## 13. Code Rules

- Simple layered structure in each Spring app: `controller` → `service` → `repository`, plus `entity`, `dto`, `config`, `exception`.
- Clear names, small methods, short comments that explain **why**. No over-engineering, no unnecessary patterns or abstractions.
- Validate inputs. Use a global exception handler. Return consistent JSON errors.
- Configuration in `application.yml` and environment variables. Provide an `.env.example` file.
- Each component has a short `README.md` with how to run it.
- Flutter: a clear folder structure (screens, widgets, services, models), with simple state management that is easy to explain.
- **Student-level simplicity.** This is a graduation project built by students. Prefer the most straightforward solution, even if it is less elegant. Use few classes, plain Spring features, and no advanced patterns, custom annotations, or clever generics. Every class must be easy for a student to explain in one or two sentences.
- **No extra features.** Build only what maps to a use case or requirement in CLAUDE.md.
- **Tests:** only the key tests for each feature (2–3), enough for the report's test cases.
## 14. Workflow Rules for Claude Code

1. **Plan before coding.** For any non-trivial task, propose a plan and wait for approval.
2. Work on **one feature at a time**. Keep each change small and runnable.
3. After each feature, write `docs/explain/<feature>.md` in **simple English**: what it does, which files are involved, how the data flows, and likely committee questions with short answers.
4. Update `docs/progress.md` (done / next / open issues) after each task.
5. Ask before adding new dependencies or changing the architecture in this file.
6. **Terminology:** "mobile application", "Card Token (PAR)", "Zero-Touch", "Hub", "Mock POS page", "Mock POS database".

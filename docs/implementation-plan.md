# Fawateer Prototype — Implementation Plan

**Goal:** A working Zero-Touch demo. An invoice created on the Mock POS page reaches the customer's mobile application within about 5 seconds, already categorized by AI, with sensitive items encrypted between the Hub and the phone. Paper receipts can also be added by photo.

**Architecture:** One PostgreSQL server (three databases) and four programs.
- **Zero-Touch path:** the Mock POS page writes invoices to the Mock POS database. The Hub polls that database, filters, removes personal data, encrypts sensitive items, and pushes the receipt to the Fawateer backend. The backend runs AI, stores the receipt, and sends a push notification to the Flutter mobile application.
- **Paper receipts (UC5):** a shorter path. The mobile application uploads a photo straight to the backend, without the Hub. The AI reads it, and the user reviews it before it is saved.

**Tech stack:** Java 21, Spring Boot 3.5, Spring AI 1.1 (Gemini), PostgreSQL 17 (Docker Compose), Thymeleaf, Flutter 3.41, Firebase Cloud Messaging.

**Spec:** [CLAUDE.md](../CLAUDE.md)

**Status:** Version 3.
- All decisions in Part 4 and the points U2–U14 are approved.
- The build order is the Fawateer side first (Phase A), then the data source and live flow (Phase B), then demo readiness (Phase C).

**Simplicity rule for the whole plan (CLAUDE.md §13):**
- few classes
- plain Spring and Flutter features
- 2–3 key tests per step
- nothing that does not map to a use case

**How to read this plan:**
- Part 1: the folders and files.
- Part 2: the rules that all components must share.
- Part 3: the build steps, in order.
- Part 4: the decisions.
- Part 5: risks and unclear points.
- Part 6: things that commonly break.

---

## Part 1 — Repository Structure

```
Fawateer/
├── CLAUDE.md
├── README.md                        project overview + how to run the whole demo
├── .env.example                     every environment variable, with fake values
├── .gitignore                       ignores .env, build output, uploads/, Firebase key files
│
├── database/
│   ├── docker-compose.yml           PostgreSQL 17 + pgAdmin (to show the databases in the demo)
│   ├── README.md
│   ├── init/                        runs automatically the first time PostgreSQL starts
│   │   ├── 00-create-databases.sql  fawateer_db (Phase A); pos_db and hub_db added in Phase B
│   │   ├── 10-pos-schema.sql        shops, products, demo_customers, invoices, invoice_items
│   │   ├── 11-pos-seed.sql          5 shops, ~40 products (Arabic + English), 3 demo customers
│   │   ├── 20-hub-schema.sql        registered_cards, partner_shops, hub_invoices
│   │   ├── 21-hub-seed.sql          2 registered cards (keys generated at start-up), partner shops
│   │   ├── 30-fawateer-schema.sql   users, receipts, receipt_items, warranties, notifications
│   │   └── 31-fawateer-seed.sql     demo users + 6 months of past receipts for the charts
│   └── demo/
│       └── warranty-reminder-demo.sql   moves one deadline to "in 2 days" to demo UC9 live
│
├── pos/                             Mock POS page (Spring Boot + Thymeleaf), port 8081
│   └── src/main/java/sa/edu/imsiu/fawateer/pos/
│       ├── PosApplication.java
│       ├── controller/  PosController
│       ├── service/     InvoiceService          totals, VAT, invoice number
│       ├── repository/  ShopRepository, ProductRepository, DemoCustomerRepository, InvoiceRepository
│       ├── entity/      Shop, Product, DemoCustomer, Invoice, InvoiceItem
│       ├── dto/         CreateInvoiceForm
│       └── exception/   GlobalExceptionHandler
│       (resources: application.yml, templates/index.html, static/pos.css)
│
├── hub/                             Mock Hub (Spring Boot), port 8082
│   └── src/main/java/sa/edu/imsiu/fawateer/hub/
│       ├── HubApplication.java
│       ├── service/     HubService              every 2 s: read → check → minimize + encrypt → send
│       │                EncryptionService       AES-256-GCM
│       │                FawateerClient          signs (HMAC) and sends the receipt to the backend
│       ├── repository/  PosInvoiceRepository (plain SQL on pos_db), RegisteredCardRepository,
│       │                PartnerShopRepository, HubInvoiceRepository
│       ├── entity/      RegisteredCard, PartnerShop, HubInvoice
│       ├── dto/         PosInvoice, PosInvoiceItem, ReceiptPayload, ReceiptPayloadItem
│       ├── controller/  StatusController        read-only status page + customer keys
│       └── config/      PosDataSourceConfig     the second database connection (pos_db)
│       (resources: application.yml, templates/status.html)
│
├── backend/                         Fawateer backend (Spring Boot), port 8080
│   └── src/main/java/sa/edu/imsiu/fawateer/backend/
│       ├── controller/  IngestController        receipts from the Hub
│       │                AuthController          register, login
│       │                ProfileController       me, consent, FCM token, notification settings
│       │                ReceiptController       list, detail, change category, image, scan, save scanned
│       │                AnalyticsController     Home dashboard + analytics screen
│       │                NotificationController  in-app notification list
│       ├── service/     IngestService           checks the Hub signature, saves the receipt
│       │                AuthService             passwords, lockout, JWT
│       │                ReceiptService          AI processing, warranties, search, category change
│       │                ScanService             photo → draft → saved SCANNED receipt
│       │                AiService               the 3 AI tasks + safe reading of the JSON answer
│       │                AnalyticsService        totals, categories, trend, top merchants
│       │                NotificationService     checks the user's settings, sends FCM, saves the list row
│       │                ReminderService         daily warranty/return reminders
│       ├── repository/  UserRepository, ReceiptRepository, WarrantyRepository, NotificationRepository
│       ├── entity/      User, Receipt, ReceiptItem, Warranty, Notification,
│       │                Category (enum), ReceiptSource (enum)
│       ├── dto/         request and response records
│       ├── config/      SecurityConfig, FirebaseConfig
│       └── exception/   GlobalExceptionHandler  one JSON error format for everything
│       (resources: application.yml, prompts/categorize-receipt.st, prompts/extract-warranty.st,
│        prompts/read-receipt-image.st; test resources: ai-eval/)
│
├── mobile/                          Fawateer mobile application (Flutter)
│   ├── pubspec.yaml, README.md
│   ├── assets/fonts/                one Arabic-friendly font (optional)
│   ├── lib/
│   │   ├── main.dart, app.dart      start-up, theme, language and RTL
│   │   ├── config/      api_config.dart, theme.dart,
│   │   │                categories.dart  (one color + one icon per category, used everywhere)
│   │   ├── l10n/        app_en.arb, app_ar.arb
│   │   ├── models/      user, receipt (with items), dashboard, analytics, scan_draft, app_notification
│   │   ├── services/    api_service (every backend call), crypto_service, push_service
│   │   ├── state/       app_state.dart   the only shared state: login, language, key
│   │   ├── screens/     login, register, consent, key_setup, main_shell (bottom bar), home,
│   │   │                receipt_list, receipt_detail, scan, scan_review, analytics,
│   │   │                notifications, profile
│   │   └── widgets/     receipt_card, category_icon, deadline_tile, empty_state
│   └── test/
│
└── docs/
    ├── implementation-plan.md       this file
    ├── progress.md                  done / next / open issues
    ├── decisions.md                 every decision and why (defense material)
    ├── demo-script.md               click-by-click defense demo + backup plan
    ├── installation.md              Chapter 5 installation steps
    ├── crypto-test-vector.json      made by the Java tests, checked by the Dart tests
    ├── api/                         ingest-contract.md, mobile-api.md, requests.http
    ├── explain/                     one simple-English file per feature (CLAUDE.md §14)
    ├── testing/                     test-cases.md, ai-accuracy.md
    └── diagrams/                    Mermaid sources: activity, sequence, component, deployment, class, ERD
```

**Notes:**
- **Timed jobs** are plain `@Scheduled` methods inside a service (`HubService`, `ReminderService`). There are no extra scheduler classes.
- **Errors** use Spring's built-in `ResponseStatusException`. There are no custom exception classes; `GlobalExceptionHandler` turns every error into the same JSON shape.
- **Receipt items** are saved together with their receipt, so there is no separate item repository.
- **Mobile screens** call `ApiService` directly. Every screen follows the same pattern: loading → data (or a friendly empty state) → plain-language error. Only login, language, and the key are shared state.
- **The receipt DTO:** the Hub and the backend each have **their own copy**. In real life they belong to different organizations, so they share only the JSON contract in `docs/api/ingest-contract.md`, not code.

---

## Part 2 — Shared Rules (Decide Once, Use Everywhere)

### 2.1 Ports

| Component | Address |
|---|---|
| PostgreSQL | `localhost:5432` |
| pgAdmin (database viewer) | http://localhost:5050 |
| Fawateer backend | http://localhost:8080 (emulator: http://10.0.2.2:8080; real phone: the laptop's IP on the hotspot) |
| Mock POS page | http://localhost:8081 |
| Hub status page | http://localhost:8082 |

### 2.2 Status Values

- **`pos_db.invoices.status`:** `NEW` → `SENT`. `SENT` means "the Hub has picked it up", for both forwarded and rejected invoices. This way the shop cannot learn which customers use Fawateer.
- **`hub_db.hub_invoices.status`:**
  - `FORWARDED`: delivered to Fawateer.
  - `REJECTED`: PAR not registered, no consent, or the shop is not a partner.
- **If sending fails:** the Hub writes a clear error to the log, with the invoice number, and moves on. There is no `FAILED` status and no retry, because servers are assumed to be up (CLAUDE.md §4, §12).

### 2.3 What the Hub Sends: `POST /api/ingest/receipts`

- **Header `X-Hub-Signature`:** HMAC-SHA256 of the request body, made with the shared secret `HUB_INGEST_SECRET`.
- **Kept:**
  - Header fields:
    - `hubReference` (links the receipt to the Hub's record, handy in the demo)
    - `fawateerUserId`
    - merchant name, merchant VAT number, branch
    - invoice number, issue date-time (ISO-8601 with `+03:00`)
    - subtotal, VAT amount, total, currency, payment method
    - return policy text
  - Per item:
    - quantity, unit price, VAT rate, line total, `sensitive` flag
    - normal items: `nameAr`, `nameEn`, and the warranty text
    - sensitive items: `encryptedNameAr`, `encryptedNameEn`, and `generalCategory`
- **Removed by the Hub:** PAR, customer name, customer phone, masked card number, and POS internal IDs.

### 2.4 Encryption Format (Java and Dart Must Match Exactly)

- **Algorithm:** AES-256-GCM.
- **Key:** 32 random bytes, stored as Base64. There is one key per customer.
- **IV:** a new random 12-byte IV for every encrypted field.
- **Stored value:** `Base64( IV ‖ ciphertext ‖ 16-byte tag )`.
- **Text:** converted to UTF-8 bytes before encryption.
- **Check:** a Java test writes `docs/crypto-test-vector.json`, and a Dart test must decrypt it to the same Arabic text.

### 2.5 Mobile API

All endpoints need a JWT except `register` and `login`.

| Method | Path | Used by |
|---|---|---|
| POST | `/api/auth/register`, `/api/auth/login` | Login / registration |
| GET | `/api/me` | Profile: card linked, consent, notification settings |
| POST | `/api/me/consent` | Consent screen |
| PUT | `/api/me/fcm-token` | Push setup |
| PUT | `/api/me/notification-settings` | Profile: two switches (new receipts, deadline alerts) |
| GET | `/api/dashboard` | Home |
| GET | `/api/receipts?q&category&source&from&to` | Receipt list (UC6) |
| GET | `/api/receipts/{id}` | Receipt detail (UC6) |
| PUT | `/api/receipts/{id}/category` | Change category (UC3 alternative flow) |
| GET | `/api/receipts/{id}/image` | Scanned receipt photo |
| POST | `/api/receipts/scan` | Photo → draft for review (UC5) |
| POST | `/api/receipts` | Save the reviewed draft as `SCANNED` (UC5) |
| GET | `/api/analytics/monthly?month=2026-10` | Analytics, monthly view (UC7) |
| GET | `/api/analytics/weekly?weekStart=2026-10-04` | Analytics, weekly view (UC7) |
| GET | `/api/notifications` | Notifications list (UC8, UC9) |

### 2.6 Categories

Food, Groceries, Electronics, Healthcare, Clothing, Home, Transport, Entertainment, Personal Care, Other, Uncategorized.

- The same list is used in the AI prompts, the Java `Category` enum, the shops' `general_category` values, and the app's translations.
- Each category has one fixed color and one icon in the app (`config/categories.dart`).

### 2.7 Receipt Sources

- **`ZERO_TOUCH`:** came through the Hub.
- **`SCANNED`:** photo uploaded by the user.

The app shows these as a "Zero-Touch" or "Scanned" badge, and the receipt list can be filtered by source.

---

## Part 3 — Build Order

### Why This Order

1. **Fawateer side first.**
   - The mobile application is the main product the committee judges (CLAUDE.md §8.1), so Phase A builds it first.
   - At the end of Phase A, the whole mobile application works with realistic seeded receipts and real scanned photos. That gives early screenshots for Chapter 4 and early UAT.
2. **Then the data source.** Phase B adds the Mock POS page and the Hub, and replaces the seeded data with the live Zero-Touch flow. After Step 17, the full live demo from CLAUDE.md §4 works.
3. **Two small adjustments** keep every step checkable:
   - The scan **app** (Step 9) comes after the first screens, because it needs login, the API service, and the receipt detail screen.
   - The **ingest endpoint** (Step 13) comes before the Hub, so the Hub has somewhere to send receipts when we test it.
4. **Early risk checks:**
   - The AI key and image reading are tested in Step 0.
   - The Java ↔ Dart encryption format is built in Phase B. The shared test vector catches any mismatch right away (Steps 15–16).

**Owner:** I build every step in this session. The owner reviews it and must be able to explain it at the defense: Omar for the backend and AI, Mohammed for the mobile application (CLAUDE.md §1).

**Database resets in Phase B:** Steps 12 and 14 add new databases. PostgreSQL only runs the `init/` scripts on a fresh start, so these steps need `docker compose down -v`.
- The seeded data is recreated automatically.
- Receipts scanned during Phase A are deleted.

### Overview

| # | Step | Owner | You can check it by… |
|---|---|---|---|
| | **Phase A — Fawateer side** | | |
| 0 | Tools and empty repository | Both | every tool runs; first commit |
| 1 | `fawateer_db` + seeded demo receipts | Omar | pgAdmin shows 6 months of Arabic/English receipts |
| 2 | Backend: accounts and security | Omar | login, lockout, and 401 checks |
| 3 | Backend: receipts, search, Home, analytics, change category, notifications | Omar | a user sees only their own receipts; sums match SQL |
| 4 | AI: category + warranty/return dates | Omar | accuracy run on bilingual sample receipts |
| 5 | Scan paper receipt: backend (UC5) | Omar | upload a photo → draft → saved as `SCANNED` |
| 6 | App foundation | Mohammed | login → consent → bottom bar, AR/EN; profile switches work |
| 7 | Receipt list + detail | Mohammed | grouped by month; filters; category can be changed |
| 8 | Home dashboard | Mohammed | this month vs last month, categories, recent receipts, deadlines |
| 9 | Scan paper receipt: app (UC5) | Mohammed | photo → review/edit → saved, "Scanned" badge, photo viewable |
| 10 | Analytics screen (UC7) | Mohammed | monthly/weekly charts match SQL totals |
| 11 | Notifications screen | Mohammed | seeded notifications listed; tap opens the receipt |
| | **Milestone A: the whole mobile application works with demo data and scanned receipts** | | |
| | **Phase B — Data source and live flow** | | |
| 12 | `pos_db` + Mock POS page | Omar | create an invoice in the browser → `NEW` row in `pos_db` |
| 13 | Ingest endpoint | Omar | tests: a signed receipt is saved and categorized; a bad signature is refused |
| 14 | `hub_db` + Hub: poll every 2 s, filter, forward | Omar | POS invoice appears in the app's data within ~2 s; unregistered customer is `REJECTED` |
| 15 | Hub: data minimization + encryption | Omar | sensitive item is unreadable in `fawateer_db` |
| 16 | Decryption in the app | Mohammed | the same item is readable with 🔒 on the phone |
| 17 | Push notifications (UC8) | Both | **POS click → phone notification in ~5 s (full live demo works)** |
| 18 | Warranty/return reminders (UC9) | Omar | reminder push arrives after running the demo SQL |
| | **Phase C — Demo readiness** | | |
| 19 | Demo readiness + report material | Both | two clean rehearsals on the laptop + real phone |

**After every step:**
1. I stop and tell you exactly what to run to check it.
2. I write `docs/explain/<feature>.md` in simple English (what it does, files, data flow, likely committee questions).
3. I update `docs/progress.md`.
4. One git commit.

---

## Phase A — Fawateer Side

### Step 0 — Tools and Empty Repository

**Goal:** every tool works before any code is written.

**I do:**
- `git init`, `.gitignore`, `README.md`, `.env.example`, `docs/progress.md`, `docs/decisions.md`, `docs/explain/tools-and-setup.md`.
- Update CLAUDE.md §5.2 to "about 2 seconds" (N3).
- The first commit.

**You do:**
- **Docker:** install Docker Desktop.
- **Java:** install JDK 21 (Eclipse Temurin) and point `JAVA_HOME` to it.
- **Android:**
  - Open Android Studio and install the SDK command-line tools.
  - Accept the licences.
  - Create a Pixel, Android 14, **Google Play** emulator.
- **AI:**
  - Create a Gemini API key in AI Studio.
  - Try 3 Arabic product names and one real Arabic paper receipt photo.
- **GitHub:** push to a private GitHub repository.

**Verify:**
- `docker run hello-world` prints its message.
- `java -version` shows 21.
- `flutter doctor` shows no Android errors.
- The emulator shows the Play Store.
- AI Studio reads the receipt photo.
- `git log` shows the first commit.

---

### Step 1 — `fawateer_db` and Seeded Demo Receipts

**Goal:** one command starts PostgreSQL with `fawateer_db` full of realistic demo data.

**Files:** `database/docker-compose.yml` (PostgreSQL 17 + pgAdmin), `init/00-create-databases.sql`, `init/30-fawateer-schema.sql`, `init/31-fawateer-seed.sql`, `database/README.md`.

**Schema (5 tables):**
- `users`: login, lockout counters, card-linked flag, consent time, FCM token, two notification switches
- `receipts`: including `source`, `category`, `image_path`, `hub_reference`
- `receipt_items`
- `warranties`
- `notifications`

**Seed:**
- **Sara and Khalid:**
  - BCrypt passwords, "card linked", consent not yet given (so the consent screen appears on first login).
  - Fixed user IDs, so the Hub can point to them in Phase B.
- **Receipts:**
  - About 40 for Sara across the last 6 months, and a few for Khalid.
  - **Dates are calculated from the current date**, so "this month", "last month", and the 6-month trend always have data.
  - They come from the same 5 fictional shops the Mock POS page will use (electronics, grocery, pharmacy, clothing, furniture).
  - Arabic and English item names, VAT 15%.
  - Categories are already filled, as if the AI ran. Two are left `Uncategorized`, to demo changing the category.
  - All have `source = ZERO_TOUCH`.
  - Normal items only. Encrypted sensitive items need the Hub's key, so they come in Phase B.
- **Warranties:** a few warranty and return periods, two of them ending within the next 2 weeks (for Home's upcoming deadlines).
- **Notifications:** 3 for Sara (2 new receipts, 1 deadline).

**Verify:**
- `docker compose up -d` starts the database.
- pgAdmin shows the 5 tables.
- A SQL count shows Sara's receipts spread over 6 months.
- Arabic item names display correctly.

---

### Step 2 — Backend: Accounts and Security

**Goal:** the backend project, secure login, and the user's own settings.

**Files:**
- **Project:** `backend/pom.xml`, `mvnw`, `application.yml` (reads `../.env`), `BackendApplication`, `backend/README.md`.
- **Code:** `User` entity, `UserRepository`, `SecurityConfig`, `AuthService`, `AuthController`, `ProfileController`, `GlobalExceptionHandler`.
- **Docs:** `docs/api/requests.http`, which lets you try every endpoint from VS Code.

**Endpoints:** `register`, `login`, `GET /api/me`, `POST /api/me/consent`, `PUT /api/me/fcm-token`, `PUT /api/me/notification-settings`.

**Rules:**
- **Passwords:** stored with BCrypt.
- **Lockout:** after 5 wrong passwords, the account is locked for 15 minutes.
- **Error messages:** the same message for a wrong email and a wrong password.
- **JWT:** valid for 24 h, signed with `JWT_SECRET`.
- **Access:** every endpoint needs a JWT except register and login. Ingest is added in Step 13.
- **Notification settings:** two switches, **new receipts** (UC8) and **deadline alerts** (UC9), both on by default.

**Key tests:**
1. The 5th wrong password locks the account, and the correct password is refused while it is locked.
2. A missing or forged token returns 401.

**Verify:**
- The backend starts. JPA `validate` passes, which proves the entities match the SQL.
- In `requests.http`, log in as Sara. `/api/me` shows the card as linked, her consent status, and both switches on.

---

### Step 3 — Backend: Receipts, Search, Home, Analytics, Change Category, Notifications

**Goal:** all the data the screens need.

**Endpoints:**
- **`GET /api/dashboard` (Home):**
  - this month's total, last month's total, and the % change
  - this month's breakdown by category
  - the 5 latest receipts
  - upcoming deadlines in the next 30 days, with days left
- **`GET /api/receipts`:**
  - newest first; the app groups them by month
  - `q` matches the merchant or a normal item name
  - filters: category, source, date range
  - no paging, because the demo data is small
- **`GET /api/receipts/{id}`**
- **`PUT /api/receipts/{id}/category`:** the user picks a category from the list (UC3 alternative flow). Analytics then use the new category.
- **`GET /api/analytics/monthly`:**
  - total and the previous month's total
  - by category, with amounts and %
  - the last 6 monthly totals
  - the top 5 merchants
- **`GET /api/analytics/weekly`:** the total for each of the 7 days. Weeks start on Sunday.
- **`GET /api/notifications`:** newest first.

**Rules:**
- The user ID always comes **from the JWT**.
- Someone else's receipt returns **404**.
- Amounts include VAT.
- Days and months are counted in Riyadh time.

**Files:**
- **Entities:** `Receipt`, `ReceiptItem`, `Warranty`, `Notification`, `Category`, `ReceiptSource`, with their repositories.
- **Code:** `ReceiptController`, `ReceiptService`, `AnalyticsController`, `AnalyticsService`, `NotificationController`.

**Key tests:**
1. Sara cannot read Khalid's receipt (404).
2. Changing the category to an invalid value returns 400. A valid change is saved and shows up in the monthly analytics.
3. The monthly sums match a hand-calculated example, including a receipt at 23:30 Riyadh time on the last day of the month.

**Verify:** in `requests.http`, Sara sees only her receipts. The dashboard and analytics numbers equal SQL sums in pgAdmin.

---

### Step 4 — AI: Category and Warranty/Return Dates (UC3, UC4)

**Goal:** the AI part that both scanned receipts (Step 5) and Zero-Touch receipts (Step 13) use.

**How it works:** `ReceiptService` calls `AiService` for each new receipt.
1. **Sensitive items** are **never sent to the AI**. If a receipt has only sensitive items, its category is their shop-provided `general_category`.
2. **Category:** one AI call with `prompts/categorize-receipt.st`. It receives the merchant name and the normal item names and returns `{"category": "Electronics"}`. There is one category per receipt.
3. **Warranty and return:** only if there is warranty or return text, a second call with `prompts/extract-warranty.st`. It returns **periods**, and **Java calculates the end dates** from the receipt date.
4. **Safe reading:** `AiService` reads the JSON answer safely.
   - It removes Markdown code fences around the JSON.
   - An unknown category becomes `Uncategorized`.
   - Any error, or a timeout after 10 s, gives `Uncategorized` and no dates.
   - A receipt is never lost because of the AI.
5. **Timing:** the time of each AI call is logged, to stay within the ~5 s end-to-end target.

**Provider:** Gemini through Spring AI. The model name is one setting in `.env`. Ollama is added only if Gemini causes problems.

**AI accuracy set (CLAUDE.md §11):**
- `ai-eval/receipts.json`: about 20 bilingual sample receipts, each with its expected category and warranty/return periods.
- An on-demand test calls Gemini and prints the accuracy, which goes into `docs/testing/ai-accuracy.md`.

**Key tests (no internet):**
1. A broken AI answer or an unknown category gives `Uncategorized`.
2. 2 years from 2026-10-04 is 2028-10-04.
3. A receipt with only sensitive items gets its `general_category`, and the AI is not called.

**Verify:** run the accuracy test with your Gemini key. **Targets:** at least 85% correct categories and at least 90% correct periods. The log shows how long each call took.

---

### Step 5 — Scan Paper Receipt: Backend (UC5)

**How it works:**
1. **`POST /api/receipts/scan`** (photo + JWT):
   - `ScanService` saves the photo as `uploads/<userId>/<random name>.jpg`.
   - `AiService` reads the photo with `prompts/read-receipt-image.st`. The answer is JSON: merchant, VAT number (if printed), date, items (name, quantity, unit price), VAT, total, warranty text, return policy text.
   - The **same** category and warranty steps from Step 4 then run.
   - The response is a **draft**. Nothing is saved in the database yet.
2. **`POST /api/receipts`** (the reviewed draft):
   - It checks that merchant, date, total, and at least one item are present.
   - It calculates the warranty and return end dates from the confirmed date.
   - It saves the receipt with `source = SCANNED` and the image path.
   - The photo is only looked up in the user's own folder, so nobody can attach another user's photo.
3. **`GET /api/receipts/{id}/image`:** works for the owner only.

**Rules:**
- **Unreadable photo:** a photo that is not a receipt, or cannot be read, returns **422** "We couldn't read a receipt in this photo. Try again in better light."
- **No Hub:** scanned receipts do not go through the Hub, and nothing is encrypted (U12).
- **Size:** uploads are limited to 10 MB.
- **Accuracy:** add 5 real paper-receipt photos to the AI accuracy set.

**Key tests (the AI is faked):**
1. A good JSON answer gives a draft with a category.
2. A garbage answer gives 422 with the plain message.
3. Saving a draft creates a `SCANNED` receipt, and another user gets 404 for its image.

**Verify:**
- In `requests.http`, upload a real Arabic receipt photo. A draft comes back.
- Save it. The database shows `SCANNED`, and the photo is in `uploads/`.

---

### Step 6 — Mobile App Foundation

**Goal:** the app runs on the emulator in Arabic and English, with login, consent, bottom navigation, and profile.

**Look and feel first:**
- **`theme.dart`:** calm colors and rounded cards.
- **`categories.dart`:** one color and one icon per category, used on every screen.
- **Font:** optionally, one bundled Arabic-friendly font.

**Screens:**
- **Login and register.**
- **Consent:** the user must accept to continue (PDPL).
- **`main_shell`:** a bottom bar with Home, Receipts, Analytics, Notifications, and Profile.
- **Profile:**
  - the card shown as "Linked"
  - the language toggle
  - **two notification switches**
  - logout

**Services and state:**
- **`api_service`:** every backend call. It adds the JWT and turns errors into plain messages.
- **`app_state`:** one provider.
- **JWT:** kept in secure storage.

**Other details:**
- **Backend address:** set at build time with `--dart-define`.
- **Android:** minimum SDK 28, which means Android 9+.
- **Not yet:** the key setup screen comes in Step 16.

**Key tests:**
1. On the consent screen, Continue stays disabled until the user accepts.
2. A failed login shows a plain-language message.

**Verify:**
- Log in as Sara and accept consent. The bottom bar appears.
- Switch to Arabic. The whole app flips to right-to-left.
- Turn a notification switch off and reopen Profile. It is still off.
- Enter a wrong password 5 times. A plain lock message appears.

---

### Step 7 — Receipt List and Detail (UC6, UC3 Change Category)

**Receipt list:**
- **Grouped by month**, newest first, with month headers.
- Each card shows the merchant, date, total, and the category icon and color.
- A search box, plus filters for **category, date range, and source** (Zero-Touch / Scanned).
- A friendly empty state.

**Receipt detail:**
- merchant, VAT number, date and time (Riyadh time)
- items with quantities and prices, VAT, and total
- **the category with a "Change" button**
- the warranty and return countdown
- the **source badge**
- Decrypting 🔒 items comes in Step 16.

**Key tests:**
1. Receipts are placed under the correct month header.
2. The countdown shows the correct number of days for a fixed date.

**Verify:**
- Change an "Uncategorized" seeded receipt to Groceries. The list icon updates.
- The filters work.
- Search works in Arabic and English.
- A receipt detail is at most 3 taps from Home.

---

### Step 8 — Home Dashboard

**Screen:**
- This month's total, with ↑/↓ % compared to last month.
- A small breakdown by category.
- Recent receipts.
- **Upcoming deadlines** with countdowns. Tapping one opens the receipt.
- Pull to refresh.
- An empty state for new users.
- The "+ Add receipt" button is added in Step 9.

**Key tests:**
1. The % label shows the correct direction and value for given totals.
2. The empty state appears when there are no receipts.

**Verify:** the numbers equal `/api/dashboard` and the SQL sums. Deadlines are sorted with the soonest first.

---

### Step 9 — Scan Paper Receipt: App (UC5)

**Screens:**
- **"+ Add receipt" on Home** opens the scan screen.
  - Camera or Gallery (`image_picker`). The photo is resized to about 1600 px before upload.
  - A one-line notice says that the photo is read by an AI service (U13).
  - A "Reading your receipt…" loading state.
- **Review screen:**
  - editable merchant, date, items (name, quantity, price), total, and category
  - warranty information shown read-only
  - "Save" stores the receipt and opens its detail
- **Receipt detail:**
  - the "Scanned" badge
  - a photo thumbnail that opens full-screen with zoom

**Key tests:**
1. The review screen blocks saving when the total is empty or not a number.
2. A scanned receipt shows the "Scanned" badge.

**Verify:**
- Load a real Arabic paper receipt from the emulator gallery. Fix one field in the review screen and save.
- The receipt appears with the Scanned badge, the source filter finds it, and the photo can be viewed.
- The camera is at most 3 taps from Home.

---

### Step 10 — Analytics Screen (UC7)

**Monthly view (default):**
- A month picker (◀ October 2026 ▶).
- The total compared to the previous month.
- A **donut chart by category**, with amounts and %.
- A **6-month trend** bar chart.
- The **top merchants**.
- Tapping a category opens the receipt list filtered by that category and month.

**Weekly view:** a toggle that shows a week picker and one bar per day.

**Tests:** the numbers are already tested in Step 3. This step is checked by hand.

**Verify:** the chart totals equal SQL sums. The days of a week add up correctly. The screen works in right-to-left mode.

---

### Step 11 — Notifications Screen

**Screen:**
- The in-app list, newest first.
- An icon for "new receipt" and another for "deadline".
- Tapping an entry opens the receipt.
- An empty state.
- The data comes from the seeded notifications. Real pushes are added in Step 17.

**Key test:** each notification type shows the correct icon and text.

**Verify:** Sara's 3 seeded notifications are listed, and tapping one opens the right receipt.

**Milestone A:** the whole mobile application works with demo data and scanned receipts. Take the first Chapter 4 screenshots here.

---

## Phase B — Data Source and Live Flow

### Step 12 — `pos_db` and the Mock POS Page

**Goal:** create an invoice in the browser; it is saved in `pos_db` with status `NEW`.

**Database:**
- Add `pos_db` to `00-create-databases.sql`, plus `10-pos-schema.sql` and `11-pos-seed.sql`. This needs a reset with `docker compose down -v`.
- **Seed:**
  - the same 5 fictional shops, with ZATCA-style VAT numbers, Riyadh branches, and return policies
  - about 8 products per shop in Arabic and English, with warranty text on electronics and furniture
  - 1–2 sensitive products in the pharmacy and clothing shops
  - 3 demo customers (Sara, Khalid, and Fahad, who is not registered), each with a fake PAR, a phone number, and a masked card number

**App:** `PosController`, `InvoiceService`, 4 repositories, 5 entities, `CreateInvoiceForm`, `index.html`, `pos.css`.
1. Choose a shop. The page reloads with that shop's products.
2. Choose a customer and type quantities.
3. Click "Create Invoice". The invoice is saved with status `NEW`; the merchant, customer, and product details are copied onto it.
4. A "Recent invoices" table refreshes every 2 seconds.

**Key tests:**
1. 2 × 49.99 + 1 × 10.00 gives subtotal 109.98, VAT 16.50, and total 126.48.
2. An invoice with no items or a negative quantity is rejected.

**Verify:** at http://localhost:8081, create an invoice with an Arabic-named product and one sensitive item. pgAdmin shows status `NEW` with the correct totals.

---

### Step 13 — Ingest Endpoint

**Goal:** the backend can safely receive a receipt from the Hub, save it, and run the AI from Step 4 on it.

**Files:** `IngestController`, `IngestService`, the ingest DTOs, `SecurityConfig` (ingest uses the Hub signature, not a JWT), and `docs/api/ingest-contract.md`.

**Rules:**

| Request | Response |
|---|---|
| Wrong or missing signature | **401**, even with a valid JWT |
| Invalid payload | **400**, listing the bad fields |
| Unknown user | **400** |

**Key tests (the AI is faked):**
1. A valid signed receipt returns 201. It is saved with readable Arabic names and gets a category.
2. A wrong signature returns 401.
3. An invalid payload returns 400 in the standard JSON error format.

**Verify:** the tests are green. The real flow is checked in Step 14.

---

### Step 14 — `hub_db` and the Hub: Poll, Filter, Forward

**Goal:** the first end-to-end flow, from the Mock POS database to the mobile application.

**Database:**
- Add `hub_db`, `20-hub-schema.sql`, and `21-hub-seed.sql`, then reset.
- Registered cards exist for Sara and Khalid, with their keys generated by PostgreSQL. All 5 shops are partners.

**How it works:** `HubService` runs every **2 seconds**. For each `NEW` invoice, it:
1. Reads the invoice and its items.
2. Checks that the shop is a partner and the PAR is registered, active, and has consent. If not, it saves `REJECTED`.
3. Replaces the PAR with `fawateerUserId`.
4. Signs the payload and sends it, then saves `FORWARDED`. If sending fails, it logs the error.
5. Marks the POS invoice `SENT`.

**Key tests:**
1. Unregistered PAR: `REJECTED`, and nothing is sent.
2. Registered PAR: the payload has `fawateerUserId` and **no PAR**.

**Verify:**
- **Sara's invoice:** within ~2 s the POS shows `SENT`, the Hub shows `FORWARDED`, and the receipt appears **in the app** with an AI category.
- **Fahad's invoice:** the Hub shows `REJECTED`, and nothing reaches Fawateer.
- **A pharmacy invoice with only sensitive items:** the receipt gets Personal Care, and the log shows no AI call.

---

### Step 15 — Hub Privacy: Data Minimization and Encryption

**Main files:**
- **`HubService.buildPayload`:** the one method that decides what leaves the Hub.
- **`EncryptionService`:** AES-256-GCM, as described in Part 2.4.
- **`StatusController` and `status.html`:** the delivery log, plus the "customer keys" panel (U2).
- **`docs/crypto-test-vector.json`.**

**Key tests:**
1. Encrypt then decrypt gives back the same Arabic text. This test also writes the test vector.
2. The payload contains no PAR, customer name, phone, or card, and sensitive items have encrypted names only.

**Verify:**
- In `fawateer_db.receipt_items`, only the sensitive item's names are Base64 gibberish.
- `pos_db` still shows the customer name, phone, and card. `fawateer_db` does not.
- The Hub status page lists `FORWARDED` and `REJECTED` invoices.

---

### Step 16 — Decryption in the App

**Screens and services:**
- **Key setup:** paste the key from the Hub status page. It is saved in secure storage. This screen is reached once, after consent, and from Profile.
- **`crypto_service`:** AES-256-GCM decryption.
- **Receipt detail:** shows 🔒 items in clear text, or "🔒 Encrypted item" when the phone has no key.

**Key tests:**
1. Dart decrypts the Java test vector to the same Arabic text.
2. A wrong key gives "cannot decrypt" instead of a crash.

**Verify (demo moment #5):** the sensitive item is readable with 🔒 on the phone, while the same row in pgAdmin is Base64.

---

### Step 17 — Push Notifications (UC8)

**Setup:**
- Create a free Firebase project.
- Put `google-services.json` in the app.
- Put the service-account file path in `.env`.
- Keep both files out of git.

**Backend:**
- After a Zero-Touch receipt is processed, `NotificationService` **first checks the user's "new receipts" switch**.
  - If it is off, nothing is sent; the receipt still appears in the app.
  - If it is on, it sends the FCM message and saves a row for the notifications list.
- The message never includes item names.

**App:**
- `push_service`:
  - asks for notification permission (required on Android 13+).
  - sends the FCM token.
  - shows a small in-app banner while the app is open.
  - opens the receipt when the notification is tapped.

**Key tests:**
1. With the switch off, there is no push and no list row.
2. A user with no FCM token causes no crash.

**Verify:**
- Run CLAUDE.md §4 demo steps 1–5, with the app in the background and then in the foreground.
- The time from the "Create Invoice" click to the notification is **about 5 s**.
- Turn the switch off and create another invoice. The receipt appears, but no push arrives.

---

### Step 18 — Warranty and Return Reminders (UC9)

**Backend (`ReminderService`):**
- Runs daily at 09:00 Riyadh time. The `demo` profile runs it every 30 s.
- Finds deadlines within the next 3 days that have no reminder yet, **for users whose "deadline alerts" switch is on**.
- Sends a push and saves a list row.
- Saves `reminder_sent_at`, so each reminder is sent only once.

**Demo:** `database/demo/warranty-reminder-demo.sql` moves one deadline to 2 days from today.

**Key tests:**
1. A deadline in 3 days gives exactly one reminder, even when the job runs twice.
2. With "deadline alerts" off, nothing is sent.

**Verify:** run the demo SQL. The push arrives within 30 s and appears in the Notifications screen.

---

## Phase C — Demo Readiness

### Step 19 — Demo Readiness and Report Material

- **Bad input:** a sweep of broken JSON, huge quantities, empty strings, and wrong types. Every case gets the same JSON error format and nothing crashes.
- **Logs:** one clear log line per stage, all carrying the same invoice number:
  1. POS saved
  2. Hub picked it up
  3. forwarded
  4. AI done
  5. push sent
- **`docs/demo-script.md`:** the 6 demo steps from CLAUDE.md §4, plus a short UC5 scan segment, with exact clicks and what to say. **Backup plan:** a second hotspot (or university Wi-Fi) and a recorded video.
- **`docs/testing/test-cases.md`:** the Chapter 5 table (purpose, inputs, expected, actual, pass/fail), built from the key tests, the manual checks, and a UAT form.
- **`docs/diagrams/`:** the Chapter 4 diagrams:
  - an activity diagram for each use case
  - sequence diagram (Mock POS database → phone)
  - component diagram
  - deployment diagram
  - class diagram
  - ERD and data dictionary for `fawateer_db`
  - interface screenshots
- **`docs/installation.md`:** the installation steps, plus the hardware and software specifications.

**Verify:** two complete rehearsals in a row **on the defense laptop with the real phone over the hotspot**, including the second invoice from another shop and one scanned receipt. Then record the backup video.

---

## Part 4 — Decisions (All Approved)

### 4.1 Main Decisions

- **D1 — AI provider:**
  - Gemini (free-tier Flash model) is the main provider, through Spring AI. The model name is one setting.
  - Ollama + Gemma 3 4B and the provider comparison are optional, added only if Gemini causes problems.
  - The AI only sees normal product names, the shop name, warranty text, and (for UC5) the user's own receipt photo. It never sees the customer's identity or sensitive Zero-Touch items.
- **D2 — Versions:** Spring Boot 3.5.x + Spring AI 1.1.x + Java 21.
- **D3 — Machines:**
  - Build and test on this PC with the emulator.
  - The defense will probably be on a laptop with a real phone over a phone hotspot.
  - The laptop runs Docker and the three Spring apps, so it needs about 8 GB of free RAM.
- **N1 — Category level:** one category per receipt. The user's manual change shows everywhere, including analytics.
- **N2 — Users table:** the consent time, FCM token, and two notification switches are columns in `users`, so there are no separate consent or device-token tables.
- **N3 — Polling:** CLAUDE.md §5.2 says "about 2 seconds".
- **N4 — Scanned photos:** they are stored in a local `uploads/` folder, and the database keeps only the path. A draft is not saved until the user confirms.
- **N5 — Weeks:** weeks start on Sunday in the weekly view.

### 4.2 Defaults

| Topic | Default | Why |
|---|---|---|
| Ingest authentication | HMAC-SHA256 signature of the body in `X-Hub-Signature` | Proves the request came from the Hub **and** was not changed |
| Mobile login tokens | Spring Security's built-in JWT support (HS256, 24 h) | No third-party JWT library |
| Database schema | Plain SQL files in `database/init`; JPA only validates | The SQL files *are* the ERD and data dictionary |
| Hub reading `pos_db` | Plain SQL (`JdbcClient`); JPA for its own `hub_db` | The Hub reads the POS tables like an outside system |
| Configuration | One `.env` at the repo root, imported by all three Spring apps | One place for secrets |
| Money | `NUMERIC(12,2)` / `BigDecimal`; prices excluding VAT; VAT = 15% of subtotal, rounded half-up | Exact halalas; matches the ZATCA fields |
| Time | `timestamptz` in the database; shown in Asia/Riyadh | Correct dates around midnight |
| AI dates | AI returns the period; Java calculates the end date | AI models make arithmetic mistakes |
| AI calls | Separate prompt files; the warranty call only runs when there is warranty or return text | Easy to show and test; faster |
| Processing | Synchronous inside the request, 10 s AI timeout with fallback | The simplest flow to explain |
| Failed forwarding | Log the error; no `FAILED` status, no retry | Servers are assumed up (CLAUDE.md §4) |
| Hub polling | Every 2 seconds | Keeps end-to-end close to ~5 s |
| Flutter state | One provider (`app_state`) for login, language, key; screens load their own data | Fewer classes; one pattern everywhere |
| Flutter translations | Flutter's built-in `gen-l10n` with ARB files | Right-to-left works automatically |
| Database viewer for the demo | pgAdmin in docker-compose | Nothing extra to install |
| Shop names | Fictional | No real brands without permission |
| Java packages | `sa.edu.imsiu.fawateer.pos` / `.hub` / `.backend` | A clean class diagram |

### 4.3 Dependencies

| Where | Dependency | Why |
|---|---|---|
| Backend | `spring-ai-starter-model-google-genai` | Gemini (text + images) |
| Backend | `spring-ai-starter-model-ollama` (optional) | Only if Gemini causes problems |
| Backend | `spring-boot-starter-oauth2-resource-server` | Spring's built-in JWT checking |
| Backend | `firebase-admin` | Sending FCM push notifications |
| Mobile | `provider` | The one shared state object |
| Mobile | `http` | Backend calls, including the photo upload |
| Mobile | `flutter_secure_storage` | JWT + encryption key |
| Mobile | `cryptography` | AES-256-GCM decryption |
| Mobile | `intl` (+ Flutter's `flutter_localizations`) | AR/EN, dates, numbers, month grouping |
| Mobile | `firebase_core`, `firebase_messaging` | Receiving push |
| Mobile | `fl_chart` | Donut, trend, and weekly charts |
| Mobile | `image_picker` | Camera and gallery (UC5) |
| Mobile | One Arabic-friendly font file (optional) | A consistent, polished look in AR and EN |
| Tools | Docker Desktop, pgAdmin (container), VS Code REST Client | Running and checking everything |

---

## Part 5 — Unclear Points and Risks

### 5.1 Unclear Points (All Resolved)

- **U2:** the key is pasted into the app from the Hub status page (Steps 15–16).
- **U3:** shops have a return policy text.
- **U4:** POS invoices carry the customer name, phone, and masked card, which the Hub removes.
- **U5:** Sara and Khalid are pre-linked. New users see "No card linked" and receive no Zero-Touch receipts.
- **U6:** consent at the Hub is pre-seeded. Consent in the app is required before using it.
- **U7:** the POS shows `SENT` for rejected invoices too.
- **U8:** encrypted names are not searchable on the server.
- **U9:** UC9 is demonstrated with the `demo` profile and the demo SQL.
- **U10:** lockout lasts 15 minutes.
- **U11:** the Hub writes `SENT` to the POS database as a demo shortcut.
- **U12:** scanned receipts are not encrypted, because the customer uploads their own data.
- **U13:** the scan screen tells the user that the photo is read by an AI service.
- **U14:** with a notification switch off, there is no push and no list row. The receipt still appears.

### 5.2 Risks

#### R1. Tools missing on this PC

- Docker is not installed.
- The JDK is 23, not 21.
- Android SDK command-line tools are missing, and the licences are not accepted.

**Mitigation:** Step 0.

#### R2. Network at the defense

The phone hotspot provides both the local network (phone → laptop) and the internet (Gemini, FCM).

**Mitigation:**
- Allow port 8080 in the laptop's firewall.
- Check the laptop's hotspot IP on the day. Changing it is one build flag.
- Keep a second hotspot and a recorded video as the backup.

#### R3. Free-tier changes

**Mitigation:**
- The model name is a setting.
- Check AI Studio the week before the defense.
- Add Ollama only if needed.

#### R4. Java and Dart encryption mismatch

**Mitigation:** the fixed format in Part 2.4 and the shared test vector (Steps 15–16).

#### R5. Arabic text getting corrupted

**Mitigation:** UTF-8 everywhere, and every step's check uses Arabic names.

#### R6. Android networking

- Cleartext HTTP is blocked by default.
- Android 13+ asks for notification permission.
- FCM needs a Google Play emulator image.

**Mitigation:** handled in Steps 0, 6, and 17.

#### R7. Spring Boot 3.5 end of support

Accepted in D2.

#### R8. "Free and open-source tools only"

Docker Desktop, Gemini, and Firebase are free but not open source. FCM is required by CLAUDE.md anyway, so I read the rule as "no paid services".

#### R9. AI speed versus the ~5 s target

**Mitigation:**
- Use a fast Flash model with "thinking" at its lowest setting.
- Make the warranty call only when needed.
- Log the timings in Step 4.
- If it is too slow, merge the two prompts into one call.

#### R10. Reading crumpled or faded Arabic thermal receipts

**Mitigation:**
- Test one real photo in Step 0.
- The review screen lets the user fix mistakes.
- Measure 5 real receipts in Step 5.

#### R11. Seeded dates drift

The seed calculates dates from the day the database is created, so the data gets older over time.

**Mitigation:** reset the database (`docker compose down -v`) a day before the defense, so "this month" is full.

---

## Part 6 — Things That Commonly Break (and the Test That Catches Each)

1. **Arabic text in every step of the chain.**
   - Expected: the phone shows exactly the Arabic name typed in the Mock POS page.
   - Tested in Steps 12, 13, 15, and 16.
2. **The AI returns broken JSON, an unknown category, or takes too long.**
   - Expected: the receipt is still saved as `Uncategorized`, and the user can change it.
   - Tested in Steps 3 and 4.
3. **The phone has no key or the wrong key.**
   - Expected: "🔒 Encrypted item", no crash.
   - Tested in Step 16.
4. **An invoice near midnight** (23:30 Riyadh is 20:30 UTC).
   - Expected: it appears on the correct day and month in the list and the charts.
   - Tested in Steps 3 and 7.
5. **A photo that is not a receipt, or is too blurry.**
   - Expected: a plain "couldn't read a receipt" message; nothing is saved.
   - Tested in Step 5.

---

## References

- Gemini API terms (free tier data use): https://ai.google.dev/gemini-api/terms
- Spring AI 1.1 release (Google GenAI support, Boot 3.5): https://spring.io/blog/2025/11/12/spring-ai-1-1-GA-released/
- Spring AI Google GenAI chat docs: https://docs.spring.io/spring-ai/reference/api/chat/google-genai-chat.html

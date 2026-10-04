# Fawateer (فواتير) — Zero-Touch Digital Receipts

Graduation project prototype (GP2), Information Systems Department, Imam Mohammad Ibn Saud Islamic University (IMSIU), Riyadh. Course IS 1480, Section 076, Group 07.

- **Team:**
  - Omar Alsalamah (Backend / AI lead)
  - Mohammed Aljaffan (Frontend / Mobile lead)
- **Supervisor:** Dr. Mutlaq Aldajani

## What It Does

The customer pays by card as usual, and a detailed receipt appears in the Fawateer mobile application automatically. AI categorizes the expense, finds warranty and return deadlines, and reminds the customer before they pass.

A neutral **Hub** sits between the shops and Fawateer. It:
- forwards only the invoices of registered customers who gave consent
- removes personal data that Fawateer does not need
- encrypts sensitive items so that only the customer's phone can read them

Paper receipts from other shops can be added by taking a photo.

## Components

| Folder | Component | Port | Status |
|---|---|---|---|
| `database/` | PostgreSQL + pgAdmin (Docker Compose) | 5432 / 5050 | not started |
| `backend/` | Fawateer backend (Spring Boot) | 8080 | not started |
| `mobile/` | Fawateer mobile application (Flutter) | — | not started |
| `pos/` | Mock POS page (Spring Boot + Thymeleaf) | 8081 | not started |
| `hub/` | Mock Hub (Spring Boot) | 8082 | not started |

## Requirements

- Docker Desktop
- JDK 21 (Eclipse Temurin)
- Maven 3.9+
- Flutter 3.41+ and Android Studio (emulator with a Google Play image)
- A Google AI Studio (Gemini) API key
- Git

## Getting Started

1. Copy `.env.example` to `.env` and fill in the values.
2. Each component has its own `README.md` with how to run it. They are added as each part is built.

## Documentation

- [docs/implementation-plan.md](docs/implementation-plan.md): the build plan, step by step
- [docs/progress.md](docs/progress.md): what is done, what is next, open issues
- [docs/decisions.md](docs/decisions.md): every design decision and why
- [docs/explain/](docs/explain/): a simple explanation of each feature

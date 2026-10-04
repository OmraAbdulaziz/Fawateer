# Step 0 — Tools and Setup

## What This Step Does

It prepares the computer and the Git repository before any code is written. Every later step depends on these tools.

## The Tools and Why We Use Them

| Tool | What it is | What we use it for |
|---|---|---|
| Git + GitHub | Version control | Keeps the history of every step and lets the two team members share the code. |
| Docker Desktop | Runs programs inside containers | Starts PostgreSQL and pgAdmin with one command, the same way on every computer. |
| PostgreSQL 17 | Database | Holds the three databases: `pos_db`, `hub_db`, and `fawateer_db`. |
| pgAdmin | Database viewer in the browser | Shows the databases to the committee during the demo. |
| JDK 21 | Java (long-term support version) | Runs the three Spring Boot programs: the Mock POS page, the Hub, and the backend. |
| Maven | Java build tool | Downloads libraries, runs the programs, and runs the tests. |
| Flutter + Android Studio | Mobile framework + Android tools | Builds the mobile application and runs it on an emulator or a real phone. |
| Google AI Studio (Gemini) | AI service with a free tier | Chooses categories, finds warranty dates, and reads receipt photos. |
| Firebase (from Step 17) | Google's push service | Sends push notifications to the phone. |

## Files in This Step

| File | Purpose |
|---|---|
| `.gitignore` | Lists files Git must never save: the `.env` secrets, build output, uploaded photos, and Firebase keys. |
| `.env.example` | A template of all settings with fake values. Copy it to `.env` and fill in the real values. |
| `README.md` | Project overview. |
| `docs/implementation-plan.md` | The build plan. |
| `docs/decisions.md` | Why we chose each option. |
| `docs/progress.md` | What is done, what is next, and the open issues. |

## How Settings and Secrets Flow

```
.env (only on your computer)
   ──► read by Docker Compose (database passwords)
   ──► read by the Spring Boot programs at start-up (JWT secret, Gemini key, ...)

GitHub only ever sees .env.example, which contains fake values.
```

## Likely Committee Questions

- **Why Docker?** One command starts the same database setup on any computer, including the defense laptop. Nobody has to install PostgreSQL by hand.
- **Why are the keys not in the code?** Anything in the code ends up on GitHub. Secrets live in `.env`, which Git ignores (CLAUDE.md §9: no secrets in code).
- **Why Java 21?** It is a long-term support (LTS) version, and it is what Spring Boot 3.5 is tested with.
- **Why Flutter?** One code base for Android and iOS, with good support for Arabic and right-to-left layouts.
- **Why Gemini?** The project must use free tools. Gemini's free tier reads Arabic text and receipt photos well and needs no special hardware.

# 🙏 Dankbaarheid — Dutch Gratitude Journal

**The first Dutch gratitude app that _forces_ you to think deeper every day.**

Built on Oracle APEX 24.2 and Oracle AI Database 26ai. Runs for free on Oracle Cloud's Always Free tier. No Firebase, no third-party services, no App Store required.


---

## Why This Exists

Gratitude journaling works — [64+ clinical trials](https://positivepsychology.com/gratitude-journal-research/) show it reduces depressive symptoms by up to 35%. But every existing app lets you write "my family" 365 times and call it a practice.

**Dankbaarheid doesn't.** If your entry is 85%+ similar to something you wrote in the past year, it gets blocked. No soft warnings, no "submit anyway" buttons. You _must_ find something new to be grateful for.

This turns a daily habit into a genuine reflection exercise. After a month, you can't be lazy anymore — and that's the point.

### The Market Gap

| App | Dutch | Web | Daily Questions | Duplicate Block | Price |
|-----|:-----:|:---:|:---------------:|:---------------:|------:|
| Gratitude (gratefulness.me) | Partial | ❌ | ✅ | ❌ | Free / €30/yr |
| 365 Gratitude | ❌ | ❌ | ✅ | ❌ | €60/yr |
| Happyfeed | ❌ | ✅ | ✅ | ❌ | €40/yr |
| MindHappy | ✅ | ❌ | ✅ | ❌ | €30/yr |
| **Dankbaarheid** | **✅** | **✅** | **✅** | **✅** | **Free** |

No existing app combines all four. This one does.

---

## Features

🇳🇱 **100+ Dutch Questions** — Six categories (mensen, momenten, objecten, ervaringen, natuur, algemeen) in informal Dutch (je/jij), ranging from simple to thought-provoking.

🚫 **Hard Duplicate Detection** — Uses Oracle's `FUZZY_MATCH(TRIGRAM, ...)` and `FUZZY_MATCH(LEVENSHTEIN, ...)` operators, taking the highest match. Entries scoring 85%+ similarity against the past 365 days are blocked with a clear error showing the previous answer and date.

🔔 **21:00 Push Notifications** — Daily reminders via APEX native push (Web Push API + VAPID). Handles Dutch daylight saving (CET ↔ CEST) automatically through `DBMS_SCHEDULER` with `Europe/Amsterdam` timezone.

📱 **Progressive Web App** — Installable on iOS 16.4+ and Android. Standalone mode (no browser chrome). Works offline for viewing history.

🔥 **Streak Tracking** — Current and longest streak displayed in the navigation bar. Consecutive days are calculated by walking backwards from today.

🌙 **Dark Mode** — Follows OS preference by default, with manual override. Two Theme Roller styles (Licht/Donker) switched via `apex.theme.setStyle()`.

🔐 **Custom Authentication** — Self-registration with SHA-256 hashed passwords. No dependency on Oracle APEX accounts or Social Sign-In (though those can be added).

💰 **Zero Hosting Cost** — Runs entirely on OCI Always Free tier (20 GB storage, 30 concurrent sessions). The daily scheduler job prevents auto-shutdown.

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| **Frontend** | Oracle APEX 24.2 (Universal Theme + Custom CSS) | Declarative PWA, native push, rapid development |
| **Backend** | PL/SQL — `PKG_JOURNAL` package | All business logic in one package, close to the data |
| **Database** | Oracle AI Database 26ai (Always Free) | `FUZZY_MATCH` operator with proper Dutch character support |
| **Duplicate Detection** | `FUZZY_MATCH(TRIGRAM)` + `FUZZY_MATCH(LEVENSHTEIN)` | TRIGRAM handles word reordering; Levenshtein catches close edits |
| **Push Notifications** | APEX native PWA push | One PL/SQL call → Apple, Google, Microsoft, Mozilla push services |
| **Scheduling** | `DBMS_SCHEDULER` | Timezone-aware (`Europe/Amsterdam`), auto-handles DST |
| **Styling** | Theme Roller + Custom CSS (Georgia serif, cream/brown palette) | Warm, journal-like aesthetic |

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│                  User's Browser                  │
│        (PWA — installed on home screen)          │
├─────────────────────────────────────────────────┤
│                   Oracle APEX 24.2               │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │  Login   │ │ Vandaag  │ │ Mijn antwoorden  │ │
│  │ Register │ │  (daily  │ │    (history)     │ │
│  │  Pages   │ │ question)│ │    Cards view    │ │
│  └──────────┘ └────┬─────┘ └──────────────────┘ │
│                     │                             │
│              Dynamic Action                       │
│              (AJAX submit)                        │
├─────────────────────┼───────────────────────────┤
│                     ▼                             │
│              PKG_JOURNAL (PL/SQL)                │
│  ┌──────────────────────────────────────────┐    │
│  │ authenticate_user()  register_user()     │    │
│  │ get_daily_question() submit_entry()      │    │
│  │ get_current_streak() get_longest_streak()│    │
│  └──────────────────┬───────────────────────┘    │
│                     │                             │
│          ┌──────────▼──────────┐                  │
│          │   FUZZY_MATCH()    │                  │
│          │ TRIGRAM + LEVENSHTEIN│                  │
│          │  threshold >= 85%   │                  │
│          └──────────┬──────────┘                  │
│                     │                             │
├─────────────────────┼───────────────────────────┤
│                     ▼                             │
│         Oracle AI Database 26ai                  │
│  ┌───────────┐ ┌───────────┐ ┌────────────────┐ │
│  │ APP_USERS │ │ QUESTIONS │ │JOURNAL_ENTRIES │ │
│  │           │ │ (100+ NL) │ │ (1 per day)    │ │
│  └───────────┘ └───────────┘ └────────────────┘ │
│                                                   │
│  ┌───────────────────┐  ┌──────────────────────┐ │
│  │ QUESTION_HISTORY  │  │ DBMS_SCHEDULER       │ │
│  │ (365-day rotation)│  │ 21:00 Europe/Amsterdam│ │
│  └───────────────────┘  └──────────┬───────────┘ │
│                                     │             │
│                          apex_pwa.send_push_      │
│                          notification()           │
└─────────────────────────────────────────────────┘
```

---

## Database Schema

```sql
APP_USERS              QUESTIONS              JOURNAL_ENTRIES
├── user_id (PK)       ├── question_id (PK)   ├── entry_id (PK)
├── username (UQ)      ├── text_nl             ├── user_id (FK → APP_USERS)
├── email (UQ)         ├── category            ├── question_id (FK → QUESTIONS)
├── password_hash      └── created_at          ├── entry_text (min 10 chars)
├── display_name                               ├── entry_date (UQ with user_id)
├── dark_mode                                  └── created_at
└── timezone

QUESTION_HISTORY
├── history_id (PK)
├── user_id (FK → APP_USERS)
├── question_id (FK → QUESTIONS)
└── shown_at
```

**No `push_subscriptions` table needed** — APEX manages this natively via the `APEX_APPL_PUSH_SUBSCRIPTIONS` view.

---

## Quick Start

### Prerequisites

- Oracle Cloud account ([sign up free](https://signup.cloud.oracle.com))
- Always Free Autonomous Database with Oracle AI Database 26ai
- APEX 24.2+ workspace

### 1. Install Database Objects

Download or clone this repo, then connect with SQLcl:

```bash
sql JOURNAL/<password>@<your_connection>
@scripts/install.sql
```

This creates all tables, the `PKG_JOURNAL` package, indexes, and seeds 100 Dutch questions.

### 2. Import the APEX Application

1. Open APEX → **App Builder** → **Import**
2. Select `apex/f100/install.sql`
3. Follow the import wizard
4. Set the Authentication Scheme to **Custom Journal Auth** (function: `pkg_journal.authenticate_user`)

### 3. Configure PWA & Push Notifications

1. **Shared Components** → **Progressive Web App** → Enable, set Installable = Yes
2. **Push Notifications** → toggle On → **Generate Credentials**
3. Upload an app icon (512×512 PNG)
4. Click **Add Settings Page** for user opt-in

### 4. Create the Daily Reminder Job

```sql
-- Run in SQL Workshop → SQL Commands
@scripts/scheduler.sql
```

This creates a `DBMS_SCHEDULER` job that fires at 21:00 `Europe/Amsterdam` every day, sending push notifications to all subscribed users.

### 5. Test

1. Register a new account via the app
2. Answer today's question
3. Install the PWA on your phone (Safari → Share → Add to Home Screen)
4. Enable notifications in the app settings
5. Wait for 21:00 — or run `DBMS_SCHEDULER.RUN_JOB('DAILY_GRATITUDE_REMINDER')` manually

---

## Project Structure

```
dankbaarheid-journal/
├── README.md
├── .gitignore
├── apex/                          # APEX application (split export)
│   └── f100/
│       ├── install.sql            # Single-file import for APEX
│       └── application/
│           ├── pages/             # Individual page exports
│           ├── shared_components/ # Auth, templates, LOVs, etc.
│           └── create_application.sql
├── database/
│   ├── tables/
│   │   ├── app_users.sql
│   │   ├── questions.sql
│   │   ├── journal_entries.sql
│   │   └── question_history.sql
│   ├── packages/
│   │   ├── pkg_journal.pks       # Package specification
│   │   └── pkg_journal.pkb       # Package body
│   ├── data/
│   │   └── seed_questions.sql    # 100 Dutch questions
│   └── indexes/
│       └── all_indexes.sql
├── static/
│   ├── css/
│   │   └── app.css               # Custom warm styling + dark mode
│   └── js/
│       └── app.js                # Dark mode controller
├── docs/
│   ├── images/                   # README screenshots
│   └── livelabs/                 # Complete step-by-step build guide
│       ├── introduction/
│       ├── lab1-provision-oci/
│       ├── lab2-create-schema/
│       ├── lab3-build-apex-app/
│       ├── lab4-duplicate-detection/
│       ├── lab5-pwa-notifications/
│       ├── lab6-styling-dark-mode/
│       ├── lab7-git-export/
│       └── workshops/tenancy/manifest.json
└── scripts/
    ├── install.sql               # Master database install script
    ├── scheduler.sql             # 21:00 reminder job setup
    └── export.sh                 # Re-export script for development
```

---

## The LiveLabs Workshop

Want to build this from scratch? The `docs/livelabs/` directory contains a complete 7-lab tutorial in Oracle LiveLabs format:

| Lab | What You Build | Time |
|-----|---------------|------|
| **1** | Provision Always Free ADB + APEX workspace | 20 min |
| **2** | Database schema, `PKG_JOURNAL` package, 100 Dutch questions | 25 min |
| **3** | APEX app: auth, registration, daily question, history, settings | 60 min |
| **4** | Fuzzy duplicate detection with `FUZZY_MATCH` deep dive | 25 min |
| **5** | PWA + native push notifications + 21:00 scheduler | 30 min |
| **6** | Custom CSS, Theme Roller, dark mode toggle | 35 min |
| **7** | SQLcl split export + Git setup | 25 min |
| | **Total** | **~4 hours** |

---

## Design Decisions

### Why hard blocking instead of soft warnings?

Soft warnings create click-through fatigue — users ignore them and submit anyway. Hard blocking removes the decision entirely: if it's too similar, you _have_ to think of something new. This is the entire point of a gratitude practice. It's a **feature as constraint**, like Twitter's character limit forcing conciseness.

### Why `FUZZY_MATCH(TRIGRAM)` over `UTL_MATCH`?

`FUZZY_MATCH` (introduced in Oracle 23ai) operates on **characters**, not bytes. This matters for Dutch text with characters like ë, ï, and é — `UTL_MATCH.EDIT_DISTANCE_SIMILARITY` works byte-by-byte and can miscalculate similarity for multi-byte characters. We use `GREATEST(TRIGRAM, LEVENSHTEIN)` because TRIGRAM tolerates word reordering while Levenshtein catches single-word edits.

### Why not vector/semantic similarity (RAG)?

Semantic similarity would block "mijn gezondheid" and "dat ik gezond ben" as duplicates — same meaning, different words. But rewriting "my health" as "that I can wake up every morning without pain" is _exactly the deeper reflection_ we want to encourage. FUZZY_MATCH catches lazy repetitions; semantic similarity would punish creative reframing. We want the latter.

### Why APEX native push instead of Firebase?

APEX 23.1+ handles push notifications natively. One PL/SQL call (`apex_pwa.send_push_notification`) delivers to Apple, Google, Microsoft, and Mozilla push services. Zero external dependencies, zero API keys, zero cost, zero maintenance. Adding Firebase would mean managing a second service for no benefit.

### Why `DBMS_SCHEDULER` with timezone instead of UTC offset?

Using `Europe/Amsterdam` as the timezone region name (not `+01:00` or `+02:00`) means the scheduler **automatically handles daylight saving time**. The Netherlands switches between CET (UTC+1) and CEST (UTC+2), and the scheduler adjusts the job's run time accordingly. A fixed UTC offset would send notifications at the wrong time for half the year.

---

## Always Free Tier Limits

This app runs comfortably within Oracle Cloud's free limits:

| Resource | Limit | App Usage |
|----------|-------|-----------|
| Storage | 20 GB | ~200 MB for 500 users × 365 entries |
| Concurrent DB sessions | 30 | ~3–6 HTTP users simultaneously |
| OCPUs | 1 | Sufficient for journal workload |
| Outbound data | 10 TB/month | Negligible for a text-based app |

The daily scheduler job keeps the database active, preventing the 7-day auto-shutdown. Suitable for personal use or a small community. For scaling beyond ~6 concurrent users, upgrade to a paid ECPU model.

---

## Development

### Re-exporting after changes

After modifying the app in APEX, re-export with:

```bash
./scripts/export.sh
```

Or manually:

```bash
sql JOURNAL/<password>@<connection>
apex export -applicationid 100 -split -skipExportDate -expOriginalIds -expSupportingObjects Y -dir apex/
exit
```

### Key flags

| Flag | Purpose |
|------|---------|
| `-split` | Individual files per component (diff-friendly) |
| `-skipExportDate` | No noisy timestamp changes in diffs |
| `-expOriginalIds` | Stable component IDs across environments |
| `-expSupportingObjects Y` | Includes database objects referenced by the app |

---

## Contributing

Contributions are welcome, especially:

- Additional Dutch gratitude questions (maintain informal je/jij tone)
- UI improvements and accessibility enhancements
- Translations of the LiveLabs guide
- Performance optimizations for larger user bases
- Bug reports and feature suggestions

Please open an issue first to discuss significant changes.

---

## License

MIT — use it, fork it, learn from it.

---

## Author

Built by **Raoul** — Oracle APEX Developer, Amsterdam 🇳🇱

*This project was motivated by a personal positive experience with daily gratitude practice and the observation that no Dutch gratitude app combines web access, unique daily questions, and duplicate prevention.*

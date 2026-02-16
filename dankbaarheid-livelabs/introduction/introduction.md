# Introduction

## About this Workshop

Research shows that practicing daily gratitude can reduce depressive symptoms by up to 35%. But none of the existing gratitude apps combine **Dutch language content**, **web accessibility**, **daily unique questions**, and **intelligent duplicate detection**. In this workshop, you will build exactly that — a production-quality Dutch gratitude journaling app on Oracle APEX.

What makes this app special is the **hard duplicate blocking** feature: users cannot submit an answer they've already given within the past year. Using Oracle's `FUZZY_MATCH` operator with the TRIGRAM algorithm (the equivalent of PostgreSQL's `pg_trgm`), we detect 85%+ text similarity and force users to think deeper — turning a simple journaling app into a genuine personal growth tool.

The app runs entirely on Oracle Cloud Infrastructure's **Always Free tier** at zero cost, using Oracle AI Database 26ai and APEX 24.2. It is a Progressive Web App with native push notifications — no Firebase, no third-party services, no App Store required.

![Placeholder: screenshot of the finished app on mobile showing the daily question page with warm cream/brown styling](images/finished-app-mobile.png " ")

Estimated Workshop Time: 4–5 hours

### Objectives

In this workshop, you will learn how to:

- Provision an Always Free Autonomous Database on OCI with Oracle AI Database 26ai
- Build a complete APEX application with custom authentication and self-registration
- Implement fuzzy duplicate detection using the `FUZZY_MATCH` operator with TRIGRAM algorithm
- Configure APEX as a Progressive Web App with native push notifications
- Schedule daily reminders at 21:00 Amsterdam time using `DBMS_SCHEDULER`
- Create a warm, beautiful journaling interface with custom CSS and dark mode
- Export your APEX application for Git version control using SQLcl split export

### Prerequisites

This workshop assumes you have:

- An Oracle Cloud account (sign up free at [signup.cloud.oracle.com](https://signup.cloud.oracle.com))
- Basic familiarity with SQL and PL/SQL
- Basic understanding of Oracle APEX (no expert knowledge needed)
- A modern web browser (Chrome, Edge, Safari 16.4+, or Firefox)
- A smartphone for testing PWA installation and push notifications

### Architecture Overview

The application uses a simple but powerful architecture:

| Layer | Technology |
|---|---|
| **Frontend** | Oracle APEX 24.2 (Universal Theme, Custom CSS) |
| **Backend** | PL/SQL stored procedures and APEX processes |
| **Database** | Oracle AI Database 26ai (Always Free tier) |
| **Duplicate Detection** | `FUZZY_MATCH(TRIGRAM, ...)` operator |
| **Push Notifications** | APEX native PWA push (Web Push API + VAPID) |
| **Scheduling** | `DBMS_SCHEDULER` with Europe/Amsterdam timezone |
| **Hosting** | OCI Autonomous Database (HTTPS via ORDS) |

### Application Features

**Must Have (this workshop):**
- Daily unique Dutch gratitude questions from a pool of 365+
- Hard duplicate blocking (85%+ similarity within 365 days)
- Push notifications at 21:00 as daily reminders
- PWA installable on iOS and Android
- Answer history with search
- Streak tracking (current and longest)
- Custom authentication with self-registration
- Warm journaling aesthetic with dark mode

**Won't Have (v1):**
- Social sharing
- AI-generated questions
- Multi-language support
- Export to PDF

### Database Schema Overview

We will create these database objects:

| Object | Purpose |
|---|---|
| `APP_USERS` | Custom authentication with hashed passwords |
| `QUESTIONS` | Pool of 365+ Dutch gratitude questions |
| `JOURNAL_ENTRIES` | User answers with one-per-day constraint |
| `QUESTION_HISTORY` | Tracks which questions each user has seen |
| `PKG_JOURNAL` | PL/SQL package for all business logic |
| `FIND_SIMILAR_JOURNAL_ENTRY` | Procedure using `FUZZY_MATCH` for duplicate detection |
| `DAILY_GRATITUDE_REMINDER` | Scheduler job for 21:00 push notifications |

> **Note:** Push notification subscriptions are managed natively by APEX via the `APEX_APPL_PUSH_SUBSCRIPTIONS` view — no custom table needed.

You may now **proceed to the next lab**.

## Acknowledgements

* **Author** - Raoul, Oracle APEX Developer
* **Last Updated By/Date** - Raoul, February 2026

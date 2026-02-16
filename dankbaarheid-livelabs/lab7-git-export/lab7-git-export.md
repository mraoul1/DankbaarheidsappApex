# Lab 7: Git Export and Deployment

## Introduction

In this final lab, you will export your APEX application for version control, set up a Git repository structure suitable for public sharing, and create a master install script so others can deploy the app in their own APEX environments. We'll use SQLcl's split export for granular, diff-friendly version control.

Estimated Time: 25 minutes

### Objectives

In this lab, you will:
- Export the APEX application using SQLcl split export
- Set up a Git repository structure for public sharing
- Create database install scripts
- Write a README.md for the repository
- Configure `.gitignore` for APEX projects

### Prerequisites

This lab assumes you have:
- Completed Labs 1–6 (complete working application)
- Git installed on your machine
- A GitHub/GitLab account (for public sharing)
- SQLcl installed (download from oracle.com/sqlcl or `brew install sqlcl` on Mac)

## Task 1: Export the APEX Application Using SQLcl

SQLcl's split export creates individual SQL files per page and shared component — ideal for meaningful `git diff` output.

1. Open a terminal on your Mac.

2. Connect to your Autonomous Database using SQLcl. You'll need the database wallet:
    - Download the wallet from OCI Console → Autonomous Database → DB Connection → Download Instance Wallet
    - Unzip the wallet to a folder (e.g., `~/wallets/dankbaarheid`)

3. Connect:

    ```bash
    sql JOURNAL/<your_password>@dankbaarheiddb_low?wallet_location=/Users/you/wallets/dankbaarheid
    ```

    ![Placeholder: Terminal showing successful SQLcl connection](images/sqlcl-connect.png " ")

4. Run the split export with optimized flags:

    ```sql
    apex export -applicationid 100 -split -skipExportDate -expOriginalIds -expSupportingObjects Y -dir apex_export
    ```

    | Flag | Purpose |
    |---|---|
    | `-split` | Creates individual files per component |
    | `-skipExportDate` | Eliminates noisy timestamp changes in diffs |
    | `-expOriginalIds` | Preserves original component IDs |
    | `-expSupportingObjects Y` | Includes database objects |
    | `-dir apex_export` | Output directory |

    ![Placeholder: Terminal showing export progress and completion](images/sqlcl-export.png " ")

5. Exit SQLcl:

    ```sql
    exit
    ```

6. Verify the export structure:

    ```bash
    ls -la apex_export/f100/
    ```

    You should see directories like `application/`, with subdirectories for `pages/`, `shared_components/`, etc.

    ![Placeholder: Terminal showing export directory listing](images/sqlcl-export-structure.png " ")

## Task 2: Set Up the Git Repository

1. Create the repository structure:

    ```bash
    mkdir dankbaarheid-journal
    cd dankbaarheid-journal

    # Move the APEX export into the repo
    mkdir -p apex
    mv ~/apex_export/f100 apex/

    # Create remaining directories
    mkdir -p database/{tables,packages,procedures,functions,data,indexes}
    mkdir -p static/{css,js,images}
    mkdir -p docs/livelabs
    mkdir -p scripts
    ```

2. Initialize Git:

    ```bash
    git init
    ```

## Task 3: Create Database Install Scripts

1. Create the master install script that runs everything in order:

    ```bash
    cat > scripts/install.sql << 'EOF'
    -- ============================================
    -- Dankbaarheid Journal — Master Install Script
    -- ============================================
    -- Run this in your APEX workspace schema via SQLcl:
    --   sql JOURNAL/password@your_connection
    --   @scripts/install.sql
    -- ============================================

    PROMPT ========================================
    PROMPT Installing Dankbaarheid Journal Database Objects
    PROMPT ========================================

    -- Tables
    PROMPT Creating tables...
    @database/tables/app_users.sql
    @database/tables/questions.sql
    @database/tables/journal_entries.sql
    @database/tables/question_history.sql

    -- Indexes
    PROMPT Creating indexes...
    @database/indexes/all_indexes.sql

    -- Packages
    PROMPT Creating PL/SQL packages...
    @database/packages/pkg_journal.pks
    @database/packages/pkg_journal.pkb

    -- Seed data
    PROMPT Seeding 100 Dutch gratitude questions...
    @database/data/seed_questions.sql

    PROMPT ========================================
    PROMPT Database installation complete!
    PROMPT
    PROMPT Next steps:
    PROMPT 1. Import the APEX application from apex/f100/install.sql
    PROMPT 2. Configure PWA and push notifications (see README)
    PROMPT ========================================
    EOF
    ```

2. Split the database objects into individual files. Copy the table DDL from Lab 2 into separate files:

    ```bash
    # Create table files (copy the CREATE TABLE statements from Lab 2)
    # database/tables/app_users.sql
    # database/tables/questions.sql
    # database/tables/journal_entries.sql
    # database/tables/question_history.sql

    # Create index file
    # database/indexes/all_indexes.sql

    # Create package files
    # database/packages/pkg_journal.pks  (specification)
    # database/packages/pkg_journal.pkb  (body)

    # Create seed data file
    # database/data/seed_questions.sql
    ```

    > **Tip:** You can extract these from the SQL you ran in Lab 2. Each file should contain one `CREATE TABLE`, `CREATE INDEX`, or `CREATE OR REPLACE PACKAGE` statement.

    ![Placeholder: File explorer showing organized database directory](images/git-database-files.png " ")

3. Copy your static files:

    ```bash
    # Copy CSS and JS from APEX Static Files
    # You can download them from Shared Components → Static Application Files
    cp /path/to/app.css static/css/
    cp /path/to/app.js  static/js/
    ```

## Task 4: Create the README

1. Create `README.md` in the repository root:

    ```bash
    cat > README.md << 'READMEEOF'
    # Dankbaarheid — Dutch Gratitude Journal

    A production-quality Dutch gratitude journaling Progressive Web App built on Oracle APEX and Oracle AI Database 26ai. Features daily unique questions, hard duplicate detection using `FUZZY_MATCH`, push notifications at 21:00, and a warm journaling interface with dark mode.

    ## Features

    - 🇳🇱 **100+ Dutch gratitude questions** across 6 categories
    - 🚫 **Hard duplicate detection** — 85%+ similar entries within 365 days are blocked using `FUZZY_MATCH(TRIGRAM, ...)`
    - 🔔 **Daily push notifications** at 21:00 Amsterdam time (auto-handles DST)
    - 📱 **Progressive Web App** — installable on iOS and Android
    - 🔥 **Streak tracking** — current and longest streaks
    - 🌙 **Dark mode** — follows OS preference with manual override
    - 🔐 **Custom authentication** with self-registration
    - 💰 **Zero cost** — runs entirely on OCI Always Free tier

    ## Tech Stack

    | Layer | Technology |
    |---|---|
    | Frontend | Oracle APEX 24.2 (Universal Theme + Custom CSS) |
    | Backend | PL/SQL (PKG_JOURNAL package) |
    | Database | Oracle AI Database 26ai |
    | Duplicate Detection | `FUZZY_MATCH(TRIGRAM, ...)` + `FUZZY_MATCH(LEVENSHTEIN, ...)` |
    | Push Notifications | APEX native PWA push (VAPID/Web Push API) |
    | Scheduling | DBMS_SCHEDULER with Europe/Amsterdam timezone |
    | Hosting | OCI Autonomous Database Always Free |

    ## Installation

    ### Prerequisites

    - Oracle Cloud account with Always Free Autonomous Database
    - APEX 24.2 or later
    - Oracle Database 23ai or 26ai (for `FUZZY_MATCH`)
    - SQLcl for database object installation

    ### Step 1: Install Database Objects

    ```bash
    sql JOURNAL/<password>@<your_connection>
    @scripts/install.sql
    ```

    ### Step 2: Import APEX Application

    1. Open APEX → App Builder → Import
    2. Select `apex/f100/install.sql`
    3. Follow the import wizard

    ### Step 3: Configure PWA & Push Notifications

    1. Shared Components → Progressive Web App → Enable
    2. Push Notifications → Generate Credentials
    3. Upload an app icon (512x512 PNG)

    ### Step 4: Create Scheduler Job

    Run the scheduler SQL from `scripts/scheduler.sql` to set up the 21:00 daily reminder.

    ## Project Structure

    ```
    dankbaarheid-journal/
    ├── README.md
    ├── .gitignore
    ├── apex/                    # APEX application (split export)
    │   └── f100/
    ├── database/                # Database objects
    │   ├── tables/
    │   ├── packages/
    │   ├── data/                # Seed questions
    │   └── indexes/
    ├── static/                  # CSS and JS files
    │   ├── css/
    │   └── js/
    ├── docs/
    │   └── livelabs/            # LiveLabs workshop (implementation guide)
    └── scripts/
        ├── install.sql          # Master install script
        └── scheduler.sql        # Scheduler job setup
    ```

    ## LiveLabs Implementation Guide

    A complete step-by-step guide to build this app from scratch is available in the `docs/livelabs/` directory. It follows Oracle LiveLabs format and covers:

    1. Provisioning an Always Free Autonomous Database
    2. Creating the database schema
    3. Building the APEX application
    4. Implementing fuzzy duplicate detection
    5. Configuring PWA and push notifications
    6. Styling with custom CSS and dark mode
    7. Git export and deployment

    ## Key Design Decisions

    **Why hard duplicate blocking (not soft warnings)?**
    Soft warnings create "click-through fatigue" — users ignore them. Hard blocking forces genuine reflection, which is the entire point of a gratitude practice. If your entry is too similar, you *must* think of something different.

    **Why `FUZZY_MATCH(TRIGRAM)` over `UTL_MATCH`?**
    `FUZZY_MATCH` (26ai/23ai) handles multi-byte characters correctly, essential for Dutch text with ë, ï, é. `UTL_MATCH` works byte-by-byte, which is less precise. We use `GREATEST(TRIGRAM, LEVENSHTEIN)` for the best coverage.

    **Why APEX native push instead of Firebase?**
    APEX 23.1+ handles push notifications natively using the Web Push API with VAPID keys. Zero external dependencies, zero API keys, zero cost. One PL/SQL call sends to Apple, Google, Microsoft, and Mozilla push services.

    ## License

    MIT

    ## Author

    Built by Raoul — Oracle APEX Developer
    READMEEOF
    ```

    ![Placeholder: README rendered on GitHub](images/git-readme.png " ")

## Task 5: Configure .gitignore

1. Create `.gitignore`:

    ```bash
    cat > .gitignore << 'EOF'
    # Oracle wallet files (NEVER commit these)
    wallet/
    *.wallet.zip
    cwallet.sso
    ewallet.p12
    tnsnames.ora
    sqlnet.ora

    # Credentials
    .env
    .env.local
    *.credentials

    # macOS
    .DS_Store
    .AppleDouble
    .LSOverride

    # IDE files
    .idea/
    .vscode/
    *.swp
    *.swo
    *~

    # Node (if using any build tools)
    node_modules/

    # Compiled Python
    __pycache__/
    *.pyc

    # Logs
    *.log
    EOF
    ```

## Task 6: Copy LiveLabs Documentation

1. Copy the LiveLabs files into the repository:

    ```bash
    # Copy the LiveLabs workshop files you created
    cp -r /path/to/livelabs/* docs/livelabs/
    ```

    The `docs/livelabs/` directory should contain:
    ```
    docs/livelabs/
    ├── introduction/
    │   └── introduction.md
    ├── lab1-provision-oci/
    │   └── lab1-provision-oci.md
    ├── lab2-create-schema/
    │   └── lab2-create-schema.md
    ├── lab3-build-apex-app/
    │   └── lab3-build-apex-app.md
    ├── lab4-duplicate-detection/
    │   └── lab4-duplicate-detection.md
    ├── lab5-pwa-notifications/
    │   └── lab5-pwa-notifications.md
    ├── lab6-styling-dark-mode/
    │   └── lab6-styling-dark-mode.md
    ├── lab7-git-export/
    │   └── lab7-git-export.md
    └── workshops/
        └── tenancy/
            └── manifest.json
    ```

## Task 7: Commit and Push

1. Stage and commit all files:

    ```bash
    git add .
    git commit -m "feat: initial release — Dutch gratitude journal on Oracle APEX

    - Custom authentication with self-registration
    - 100 Dutch gratitude questions across 6 categories
    - Hard duplicate detection using FUZZY_MATCH (85% threshold)
    - PWA with native push notifications
    - Daily 21:00 reminder via DBMS_SCHEDULER
    - Warm cream/brown styling with dark mode
    - Complete LiveLabs implementation guide"
    ```

2. Create a GitHub repository and push:

    ```bash
    # Create repo on GitHub first, then:
    git remote add origin https://github.com/yourusername/dankbaarheid-journal.git
    git branch -M main
    git push -u origin main
    ```

    ![Placeholder: GitHub repository page showing the committed files](images/git-github-repo.png " ")

## Task 8: Verify the Repository

Check that your public repository:

- [ ] Has a clear README with installation instructions
- [ ] Does NOT contain any wallet files or credentials
- [ ] Contains the APEX split export in `apex/f100/`
- [ ] Contains database scripts in `database/`
- [ ] Contains the LiveLabs documentation in `docs/livelabs/`
- [ ] Has a proper `.gitignore`
- [ ] Has the master install script in `scripts/`

> **Important:** Double-check that no wallet files, passwords, or API keys are committed. Run `git log --all --diff-filter=A -- "*.wallet*" "*.p12" "*.sso"` to verify.

## Congratulations!

You have successfully built and published a complete Dutch gratitude journaling application on Oracle APEX! 🎉

Here's what you've accomplished across all seven labs:

| Lab | What You Built |
|---|---|
| **Lab 1** | Provisioned an Always Free Autonomous Database with APEX |
| **Lab 2** | Created the database schema with 100 Dutch questions |
| **Lab 3** | Built the APEX app with authentication, daily questions, and history |
| **Lab 4** | Implemented fuzzy duplicate detection with FUZZY_MATCH |
| **Lab 5** | Configured PWA with native push notifications at 21:00 |
| **Lab 6** | Styled the app with warm colors and dark mode |
| **Lab 7** | Exported to Git and prepared for public sharing |

### Next Steps

- **Use the app daily** — dogfood it for 30 days before sharing
- **Add more questions** — expand beyond 100 to give users more variety
- **Blog about it** — share the LiveLabs guide on your blog
- **Iterate** — add features like search, streak history graphs, or data export

### Resources

- [Oracle APEX Documentation](https://docs.oracle.com/en/database/oracle/apex/)
- [APEX PWA Documentation](https://docs.oracle.com/en/database/oracle/apex/24.2/htmdb/creating-a-progressive-web-app.html)
- [FUZZY_MATCH Documentation](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/data-quality-operators.html)
- [OCI Always Free Resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)

## Acknowledgements

* **Author** - Raoul, Oracle APEX Developer
* **Last Updated By/Date** - Raoul, February 2026

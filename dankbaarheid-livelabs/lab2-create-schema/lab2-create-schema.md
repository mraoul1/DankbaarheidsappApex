# Lab 2: Create the Database Schema

## Introduction

In this lab, you will create the database schema for the gratitude journal app. This includes user management tables, the questions pool, journal entries with a one-per-day constraint, question rotation history, and a comprehensive PL/SQL package for all business logic. You'll also seed 100 Dutch gratitude questions across six categories.

Estimated Time: 25 minutes

### Objectives

In this lab, you will:
- Create all database tables with proper constraints and indexes
- Build a PL/SQL package with authentication, journal entry, streak, and question rotation logic
- Seed 100 Dutch gratitude questions across six categories
- Verify the schema works correctly

### Prerequisites

This lab assumes you have:
- Completed Lab 1 (Autonomous Database with APEX workspace ready)
- Access to SQL Workshop → SQL Commands in your APEX workspace

## Task 1: Create the Core Tables

1. Navigate to **SQL Workshop** → **SQL Commands** in your APEX workspace.

    ![Placeholder: SQL Workshop > SQL Commands navigation](images/apex-sql-commands-nav.png " ")

2. Run the following SQL to create the `APP_USERS` table for custom authentication:

    ```sql
    CREATE TABLE app_users (
        user_id        NUMBER GENERATED ALWAYS AS IDENTITY
                       CONSTRAINT app_users_pk PRIMARY KEY,
        username       VARCHAR2(255)  NOT NULL,
        email          VARCHAR2(255)  NOT NULL,
        password_hash  VARCHAR2(255)  NOT NULL,
        display_name   VARCHAR2(255),
        timezone       VARCHAR2(50)   DEFAULT 'Europe/Amsterdam',
        dark_mode      CHAR(1)        DEFAULT 'N'
                       CONSTRAINT app_users_dm_ck CHECK (dark_mode IN ('Y','N')),
        created_at     TIMESTAMP      DEFAULT SYSTIMESTAMP,
        last_login     TIMESTAMP,
        is_active      CHAR(1)        DEFAULT 'Y'
                       CONSTRAINT app_users_active_ck CHECK (is_active IN ('Y','N')),
        CONSTRAINT app_users_username_uq UNIQUE (username),
        CONSTRAINT app_users_email_uq   UNIQUE (email)
    );
    ```

    ![Placeholder: SQL Commands showing successful table creation](images/apex-create-app-users.png " ")

3. Run the following SQL to create the `QUESTIONS` table:

    ```sql
    CREATE TABLE questions (
        question_id    NUMBER GENERATED ALWAYS AS IDENTITY
                       CONSTRAINT questions_pk PRIMARY KEY,
        text_nl        VARCHAR2(500)  NOT NULL,
        category       VARCHAR2(50)   NOT NULL
                       CONSTRAINT questions_cat_ck CHECK (
                           category IN ('mensen','momenten','objecten','ervaringen','natuur','algemeen')
                       ),
        created_at     TIMESTAMP      DEFAULT SYSTIMESTAMP
    );
    ```

4. Run the following SQL to create the `JOURNAL_ENTRIES` table. This is the heart of the app — note the one-entry-per-user-per-day constraint and minimum length check:

    ```sql
    CREATE TABLE journal_entries (
        entry_id       NUMBER GENERATED ALWAYS AS IDENTITY
                       CONSTRAINT journal_entries_pk PRIMARY KEY,
        user_id        NUMBER         NOT NULL
                       CONSTRAINT je_user_fk REFERENCES app_users(user_id) ON DELETE CASCADE,
        question_id    NUMBER         NOT NULL
                       CONSTRAINT je_question_fk REFERENCES questions(question_id),
        entry_text     VARCHAR2(4000) NOT NULL,
        entry_date     DATE           DEFAULT TRUNC(SYSDATE),
        created_at     TIMESTAMP      DEFAULT SYSTIMESTAMP,
        CONSTRAINT je_min_length_ck CHECK (LENGTH(TRIM(entry_text)) >= 10),
        CONSTRAINT je_one_per_day_uq UNIQUE (user_id, entry_date)
    );
    ```

5. Run the following SQL to create the `QUESTION_HISTORY` table that tracks which questions each user has seen:

    ```sql
    CREATE TABLE question_history (
        history_id     NUMBER GENERATED ALWAYS AS IDENTITY
                       CONSTRAINT question_history_pk PRIMARY KEY,
        user_id        NUMBER         NOT NULL
                       CONSTRAINT qh_user_fk REFERENCES app_users(user_id) ON DELETE CASCADE,
        question_id    NUMBER         NOT NULL
                       CONSTRAINT qh_question_fk REFERENCES questions(question_id),
        shown_at       TIMESTAMP      DEFAULT SYSTIMESTAMP
    );
    ```

## Task 2: Create Indexes for Performance

1. Run the following SQL to create indexes that optimize the most common queries:

    ```sql
    -- Fast lookup of a user's recent journal entries (for duplicate detection and history)
    CREATE INDEX idx_je_user_date ON journal_entries(user_id, entry_date DESC);

    -- Fast lookup of question history per user (for question rotation)
    CREATE INDEX idx_qh_user_shown ON question_history(user_id, shown_at);

    -- Fast user lookup for authentication
    CREATE INDEX idx_users_username ON app_users(UPPER(username));
    ```

    ![Placeholder: SQL Commands showing successful index creation](images/apex-create-indexes.png " ")

> **Note:** Unlike PostgreSQL's `pg_trgm` GIN index, Oracle's `FUZZY_MATCH` operator does not use or require a special text index. Performance is achieved by first filtering on `user_id` and `entry_date` (which reduces the comparison set to ~365 rows per user), making the fuzzy comparison well within sub-second response times.

## Task 3: Create the PL/SQL Package

The `PKG_JOURNAL` package encapsulates all business logic: authentication, password hashing, user registration, daily question rotation, journal entry submission with duplicate detection, and streak calculation.

1. Run the following SQL to create the **package specification**:

    ```sql
    CREATE OR REPLACE PACKAGE pkg_journal AS

        ---------------------------------------------------------------
        -- Authentication
        ---------------------------------------------------------------
        FUNCTION hash_password(
            p_username IN VARCHAR2,
            p_password IN VARCHAR2
        ) RETURN VARCHAR2;

        FUNCTION authenticate_user(
            p_username IN VARCHAR2,
            p_password IN VARCHAR2
        ) RETURN BOOLEAN;

        PROCEDURE register_user(
            p_username     IN  VARCHAR2,
            p_email        IN  VARCHAR2,
            p_password     IN  VARCHAR2,
            p_display_name IN  VARCHAR2 DEFAULT NULL,
            p_user_id      OUT NUMBER
        );

        ---------------------------------------------------------------
        -- Daily Question
        ---------------------------------------------------------------
        PROCEDURE get_daily_question(
            p_user_id      IN  NUMBER,
            p_question_id  OUT NUMBER,
            p_question_text OUT VARCHAR2,
            p_category     OUT VARCHAR2
        );

        ---------------------------------------------------------------
        -- Journal Entry (with duplicate detection)
        ---------------------------------------------------------------
        PROCEDURE submit_entry(
            p_user_id       IN  NUMBER,
            p_question_id   IN  NUMBER,
            p_entry_text    IN  VARCHAR2,
            p_is_duplicate  OUT NUMBER,
            p_dup_text      OUT VARCHAR2,
            p_dup_date      OUT DATE,
            p_dup_pct       OUT NUMBER,
            p_entry_id      OUT NUMBER
        );

        ---------------------------------------------------------------
        -- Streak Calculation
        ---------------------------------------------------------------
        FUNCTION get_current_streak(
            p_user_id IN NUMBER
        ) RETURN NUMBER;

        FUNCTION get_longest_streak(
            p_user_id IN NUMBER
        ) RETURN NUMBER;

        ---------------------------------------------------------------
        -- Check if user answered today
        ---------------------------------------------------------------
        FUNCTION answered_today(
            p_user_id IN NUMBER
        ) RETURN BOOLEAN;

        ---------------------------------------------------------------
        -- Get user_id for the currently logged-in APEX user
        ---------------------------------------------------------------
        FUNCTION get_current_user_id RETURN NUMBER;

    END pkg_journal;
    /
    ```

    ![Placeholder: SQL Commands showing package spec compiled successfully](images/apex-pkg-spec.png " ")

2. Run the following SQL to create the **package body**:

    ```sql
    CREATE OR REPLACE PACKAGE BODY pkg_journal AS

        -- Private constant for password hashing salt
        c_salt CONSTANT RAW(32) := UTL_RAW.CAST_TO_RAW('DankbaarheidApp2026!SecureSalt');

        ---------------------------------------------------------------
        -- HASH_PASSWORD
        ---------------------------------------------------------------
        FUNCTION hash_password(
            p_username IN VARCHAR2,
            p_password IN VARCHAR2
        ) RETURN VARCHAR2 IS
        BEGIN
            RETURN RAWTOHEX(
                DBMS_CRYPTO.HASH(
                    UTL_RAW.CAST_TO_RAW(
                        UPPER(p_username)
                        || UTL_RAW.CAST_TO_VARCHAR2(c_salt)
                        || p_password
                    ),
                    DBMS_CRYPTO.HASH_SH256
                )
            );
        END hash_password;

        ---------------------------------------------------------------
        -- AUTHENTICATE_USER
        ---------------------------------------------------------------
        FUNCTION authenticate_user(
            p_username IN VARCHAR2,
            p_password IN VARCHAR2
        ) RETURN BOOLEAN IS
            l_stored_hash VARCHAR2(255);
        BEGIN
            SELECT password_hash
            INTO   l_stored_hash
            FROM   app_users
            WHERE  UPPER(username) = UPPER(p_username)
               AND is_active = 'Y';

            IF hash_password(p_username, p_password) = l_stored_hash THEN
                UPDATE app_users
                SET    last_login = SYSTIMESTAMP
                WHERE  UPPER(username) = UPPER(p_username);
                RETURN TRUE;
            END IF;
            RETURN FALSE;
        EXCEPTION
            WHEN NO_DATA_FOUND THEN
                RETURN FALSE;
        END authenticate_user;

        ---------------------------------------------------------------
        -- REGISTER_USER
        ---------------------------------------------------------------
        PROCEDURE register_user(
            p_username     IN  VARCHAR2,
            p_email        IN  VARCHAR2,
            p_password     IN  VARCHAR2,
            p_display_name IN  VARCHAR2 DEFAULT NULL,
            p_user_id      OUT NUMBER
        ) IS
        BEGIN
            INSERT INTO app_users (username, email, password_hash, display_name)
            VALUES (
                LOWER(TRIM(p_username)),
                LOWER(TRIM(p_email)),
                hash_password(p_username, p_password),
                NVL(p_display_name, INITCAP(p_username))
            )
            RETURNING user_id INTO p_user_id;
        END register_user;

        ---------------------------------------------------------------
        -- GET_DAILY_QUESTION
        -- Returns a question the user hasn't seen in the past 365 days.
        -- If all questions have been seen, returns the least recently seen.
        ---------------------------------------------------------------
        PROCEDURE get_daily_question(
            p_user_id       IN  NUMBER,
            p_question_id   OUT NUMBER,
            p_question_text OUT VARCHAR2,
            p_category      OUT VARCHAR2
        ) IS
        BEGIN
            -- First try: a question NOT seen in the last 365 days
            BEGIN
                SELECT question_id, text_nl, category
                INTO   p_question_id, p_question_text, p_category
                FROM (
                    SELECT q.question_id, q.text_nl, q.category
                    FROM   questions q
                    WHERE  q.question_id NOT IN (
                        SELECT qh.question_id
                        FROM   question_history qh
                        WHERE  qh.user_id = p_user_id
                           AND qh.shown_at > SYSTIMESTAMP - INTERVAL '365' DAY
                    )
                    ORDER BY DBMS_RANDOM.VALUE
                )
                WHERE ROWNUM = 1;
            EXCEPTION
                WHEN NO_DATA_FOUND THEN
                    -- Fallback: least recently seen question
                    SELECT question_id, text_nl, category
                    INTO   p_question_id, p_question_text, p_category
                    FROM (
                        SELECT q.question_id, q.text_nl, q.category
                        FROM   questions q
                        LEFT JOIN question_history qh
                            ON q.question_id = qh.question_id
                           AND qh.user_id = p_user_id
                        ORDER BY qh.shown_at ASC NULLS FIRST, DBMS_RANDOM.VALUE
                    )
                    WHERE ROWNUM = 1;
            END;

            -- Record this question as shown
            INSERT INTO question_history (user_id, question_id)
            VALUES (p_user_id, p_question_id);
        END get_daily_question;

        ---------------------------------------------------------------
        -- SUBMIT_ENTRY
        -- Checks for duplicates using FUZZY_MATCH, then inserts if unique.
        ---------------------------------------------------------------
        PROCEDURE submit_entry(
            p_user_id       IN  NUMBER,
            p_question_id   IN  NUMBER,
            p_entry_text    IN  VARCHAR2,
            p_is_duplicate  OUT NUMBER,
            p_dup_text      OUT VARCHAR2,
            p_dup_date      OUT DATE,
            p_dup_pct       OUT NUMBER,
            p_entry_id      OUT NUMBER
        ) IS
            v_normalized_new VARCHAR2(4000);
        BEGIN
            p_is_duplicate := 0;
            p_dup_text     := NULL;
            p_dup_date     := NULL;
            p_dup_pct      := 0;
            p_entry_id     := NULL;

            -- Normalize input
            v_normalized_new := TRIM(REGEXP_REPLACE(LOWER(p_entry_text), '\s+', ' '));

            -- Check for duplicates in the past 365 days
            BEGIN
                SELECT entry_text, entry_date, sim_score
                INTO   p_dup_text, p_dup_date, p_dup_pct
                FROM (
                    SELECT
                        je.entry_text,
                        je.entry_date,
                        GREATEST(
                            FUZZY_MATCH(
                                TRIGRAM,
                                v_normalized_new,
                                TRIM(REGEXP_REPLACE(LOWER(je.entry_text), '\s+', ' '))
                            ),
                            FUZZY_MATCH(
                                LEVENSHTEIN,
                                v_normalized_new,
                                TRIM(REGEXP_REPLACE(LOWER(je.entry_text), '\s+', ' '))
                            )
                        ) AS sim_score
                    FROM journal_entries je
                    WHERE je.user_id = p_user_id
                      AND je.entry_date >= TRUNC(SYSDATE) - 365
                    ORDER BY sim_score DESC
                )
                WHERE sim_score >= 85
                  AND ROWNUM = 1;

                -- Duplicate found!
                p_is_duplicate := 1;
                RETURN;
            EXCEPTION
                WHEN NO_DATA_FOUND THEN
                    NULL; -- No duplicate, proceed with insert
            END;

            -- Insert the new entry
            INSERT INTO journal_entries (user_id, question_id, entry_text, entry_date)
            VALUES (p_user_id, p_question_id, TRIM(p_entry_text), TRUNC(SYSDATE))
            RETURNING entry_id INTO p_entry_id;
        END submit_entry;

        ---------------------------------------------------------------
        -- GET_CURRENT_STREAK
        -- Counts consecutive days with entries, going backwards from today.
        ---------------------------------------------------------------
        FUNCTION get_current_streak(
            p_user_id IN NUMBER
        ) RETURN NUMBER IS
            v_streak NUMBER := 0;
            v_check_date DATE := TRUNC(SYSDATE);
            v_found NUMBER;
        BEGIN
            LOOP
                SELECT COUNT(*)
                INTO   v_found
                FROM   journal_entries
                WHERE  user_id = p_user_id
                   AND entry_date = v_check_date;

                EXIT WHEN v_found = 0;

                v_streak := v_streak + 1;
                v_check_date := v_check_date - 1;
            END LOOP;

            RETURN v_streak;
        END get_current_streak;

        ---------------------------------------------------------------
        -- GET_LONGEST_STREAK
        ---------------------------------------------------------------
        FUNCTION get_longest_streak(
            p_user_id IN NUMBER
        ) RETURN NUMBER IS
            v_longest  NUMBER := 0;
            v_current  NUMBER := 0;
            v_prev_date DATE := NULL;
        BEGIN
            FOR rec IN (
                SELECT entry_date
                FROM   journal_entries
                WHERE  user_id = p_user_id
                ORDER BY entry_date ASC
            ) LOOP
                IF v_prev_date IS NULL OR rec.entry_date = v_prev_date + 1 THEN
                    v_current := v_current + 1;
                ELSE
                    v_current := 1;
                END IF;

                IF v_current > v_longest THEN
                    v_longest := v_current;
                END IF;

                v_prev_date := rec.entry_date;
            END LOOP;

            RETURN v_longest;
        END get_longest_streak;

        ---------------------------------------------------------------
        -- ANSWERED_TODAY
        ---------------------------------------------------------------
        FUNCTION answered_today(
            p_user_id IN NUMBER
        ) RETURN BOOLEAN IS
            v_count NUMBER;
        BEGIN
            SELECT COUNT(*)
            INTO   v_count
            FROM   journal_entries
            WHERE  user_id = p_user_id
               AND entry_date = TRUNC(SYSDATE);

            RETURN v_count > 0;
        END answered_today;

        ---------------------------------------------------------------
        -- GET_CURRENT_USER_ID
        -- Returns user_id for the currently logged-in APEX user.
        ---------------------------------------------------------------
        FUNCTION get_current_user_id RETURN NUMBER IS
            v_user_id NUMBER;
        BEGIN
            SELECT user_id
            INTO   v_user_id
            FROM   app_users
            WHERE  UPPER(username) = UPPER(V('APP_USER'));

            RETURN v_user_id;
        EXCEPTION
            WHEN NO_DATA_FOUND THEN
                RETURN NULL;
        END get_current_user_id;

    END pkg_journal;
    /
    ```

    ![Placeholder: SQL Commands showing package body compiled successfully](images/apex-pkg-body.png " ")

3. Verify the package compiled without errors:

    ```sql
    SELECT object_name, object_type, status
    FROM   user_objects
    WHERE  object_name = 'PKG_JOURNAL';
    ```

    Both rows (PACKAGE and PACKAGE BODY) should show status **VALID**.

    ![Placeholder: Query result showing PKG_JOURNAL as VALID](images/apex-pkg-valid.png " ")

## Task 4: Seed 100 Dutch Gratitude Questions

1. Run the following SQL to insert 100 Dutch gratitude questions. These use informal Dutch (je/jij) and span six categories:

    ```sql
    INSERT ALL
        -- ============ MENSEN (People) ============
        INTO questions (text_nl, category) VALUES ('Welke persoon heeft je deze week geholpen?', 'mensen')
        INTO questions (text_nl, category) VALUES ('Wie heeft je vandaag aan het lachen gemaakt?', 'mensen')
        INTO questions (text_nl, category) VALUES ('Voor welke vriend of vriendin ben je dankbaar?', 'mensen')
        INTO questions (text_nl, category) VALUES ('Welk familielid waardeer je vandaag extra?', 'mensen')
        INTO questions (text_nl, category) VALUES ('Wie heeft je iets waardevols geleerd?', 'mensen')
        INTO questions (text_nl, category) VALUES ('Welke collega maakt je werkdag beter?', 'mensen')
        INTO questions (text_nl, category) VALUES ('Voor welke buurman of buurvrouw ben je dankbaar?', 'mensen')
        INTO questions (text_nl, category) VALUES ('Wie heeft je ooit geïnspireerd om iets te proberen?', 'mensen')
        INTO questions (text_nl, category) VALUES ('Welke leraar of mentor heeft je gevormd?', 'mensen')
        INTO questions (text_nl, category) VALUES ('Aan wie denk je met een warm gevoel?', 'mensen')
        INTO questions (text_nl, category) VALUES ('Wie accepteert je precies zoals je bent?', 'mensen')
        INTO questions (text_nl, category) VALUES ('Welke onbekende was vandaag aardig tegen je?', 'mensen')
        INTO questions (text_nl, category) VALUES ('Voor wie zou je graag iets terug willen doen?', 'mensen')
        INTO questions (text_nl, category) VALUES ('Wie vertrouw je met je diepste gedachten?', 'mensen')
        INTO questions (text_nl, category) VALUES ('Welk persoon maakt moeilijke tijden draaglijker?', 'mensen')
        INTO questions (text_nl, category) VALUES ('Wie heeft je de afgelopen maand verrast?', 'mensen')
        INTO questions (text_nl, category) VALUES ('Welke relatie in je leven koester je het meest?', 'mensen')
        -- ============ MOMENTEN (Moments) ============
        INTO questions (text_nl, category) VALUES ('Welk moment deed je vandaag glimlachen?', 'momenten')
        INTO questions (text_nl, category) VALUES ('Wat was het fijnste moment van deze week?', 'momenten')
        INTO questions (text_nl, category) VALUES ('Welk onverwacht moment maakte je dag beter?', 'momenten')
        INTO questions (text_nl, category) VALUES ('Wanneer voelde je je vandaag het meest ontspannen?', 'momenten')
        INTO questions (text_nl, category) VALUES ('Welk klein geluksmomentje heb je vandaag ervaren?', 'momenten')
        INTO questions (text_nl, category) VALUES ('Wat is een herinnering die je altijd blij maakt?', 'momenten')
        INTO questions (text_nl, category) VALUES ('Welk moment van stilte waardeerde je recent?', 'momenten')
        INTO questions (text_nl, category) VALUES ('Wanneer voelde je je deze week trots?', 'momenten')
        INTO questions (text_nl, category) VALUES ('Welk gesprek gaf je vandaag energie?', 'momenten')
        INTO questions (text_nl, category) VALUES ('Wat was het laatste moment waarop je hardop lachte?', 'momenten')
        INTO questions (text_nl, category) VALUES ('Welk moment van verbinding heb je vandaag gehad?', 'momenten')
        INTO questions (text_nl, category) VALUES ('Wanneer voelde je je vandaag écht aanwezig?', 'momenten')
        INTO questions (text_nl, category) VALUES ('Welk moment van vandaag zou je willen herbelevenl?', 'momenten')
        INTO questions (text_nl, category) VALUES ('Wat was een verrassend positief moment deze week?', 'momenten')
        INTO questions (text_nl, category) VALUES ('Welk alledaags moment gaf je een goed gevoel?', 'momenten')
        INTO questions (text_nl, category) VALUES ('Wanneer voelde je je recent echt gelukkig?', 'momenten')
        INTO questions (text_nl, category) VALUES ('Welk ochtendmoment zette de toon voor je dag?', 'momenten')
        -- ============ OBJECTEN (Possessions) ============
        INTO questions (text_nl, category) VALUES ('Welk bezit maakt je dagelijks leven makkelijker?', 'objecten')
        INTO questions (text_nl, category) VALUES ('Welk voorwerp in je huis geeft je comfort?', 'objecten')
        INTO questions (text_nl, category) VALUES ('Welk kledingstuk draag je het liefst?', 'objecten')
        INTO questions (text_nl, category) VALUES ('Welk boek heeft je leven beïnvloed?', 'objecten')
        INTO questions (text_nl, category) VALUES ('Welke technologie maakt je werk of leven beter?', 'objecten')
        INTO questions (text_nl, category) VALUES ('Welk cadeau dat je ooit kreeg koester je nog steeds?', 'objecten')
        INTO questions (text_nl, category) VALUES ('Welk meubel in je huis vind je het prettigst?', 'objecten')
        INTO questions (text_nl, category) VALUES ('Welk gereedschap of instrument gebruik je graag?', 'objecten')
        INTO questions (text_nl, category) VALUES ('Voor welk voedsel of drankje ben je dankbaar?', 'objecten')
        INTO questions (text_nl, category) VALUES ('Welke foto geeft je een warm gevoel als je ernaar kijkt?', 'objecten')
        INTO questions (text_nl, category) VALUES ('Welk klein luxeproduct maakt je dag aangenamer?', 'objecten')
        INTO questions (text_nl, category) VALUES ('Wat is iets dat je bezit waarvan je hoopt het nooit kwijt te raken?', 'objecten')
        INTO questions (text_nl, category) VALUES ('Welk muziekinstrument of apparaat brengt je vreugde?', 'objecten')
        INTO questions (text_nl, category) VALUES ('Welk keukengereedschap zou je niet willen missen?', 'objecten')
        INTO questions (text_nl, category) VALUES ('Welk vervoermiddel maakt je leven mobieler?', 'objecten')
        INTO questions (text_nl, category) VALUES ('Welk object herinnert je aan een bijzonder iemand?', 'objecten')
        -- ============ ERVARINGEN (Experiences) ============
        INTO questions (text_nl, category) VALUES ('Welke ervaring heeft je als persoon doen groeien?', 'ervaringen')
        INTO questions (text_nl, category) VALUES ('Welke reis of vakantie herinner je je met plezier?', 'ervaringen')
        INTO questions (text_nl, category) VALUES ('Welke moeilijke ervaring heeft je sterker gemaakt?', 'ervaringen')
        INTO questions (text_nl, category) VALUES ('Welk nieuw ding heb je recent geleerd?', 'ervaringen')
        INTO questions (text_nl, category) VALUES ('Welke culturele ervaring heeft indruk op je gemaakt?', 'ervaringen')
        INTO questions (text_nl, category) VALUES ('Welke sportieve of fysieke activiteit geniet je van?', 'ervaringen')
        INTO questions (text_nl, category) VALUES ('Welk avontuur zou je graag opnieuw beleven?', 'ervaringen')
        INTO questions (text_nl, category) VALUES ('Welke cursus of opleiding was de moeite waard?', 'ervaringen')
        INTO questions (text_nl, category) VALUES ('Welke vrijwilligerservaring heeft je geraakt?', 'ervaringen')
        INTO questions (text_nl, category) VALUES ('Welk concert, film of voorstelling raakte je?', 'ervaringen')
        INTO questions (text_nl, category) VALUES ('Welke werkervaring was verrassend leerzaam?', 'ervaringen')
        INTO questions (text_nl, category) VALUES ('Welke uitdaging heb je overwonnen waar je trots op bent?', 'ervaringen')
        INTO questions (text_nl, category) VALUES ('Welke maaltijd of culinaire ervaring was bijzonder?', 'ervaringen')
        INTO questions (text_nl, category) VALUES ('Welk feest of evenement herinner je je met een glimlach?', 'ervaringen')
        INTO questions (text_nl, category) VALUES ('Welke creatieve bezigheid geeft je voldoening?', 'ervaringen')
        INTO questions (text_nl, category) VALUES ('Welke samenwerking bracht onverwacht mooi resultaat?', 'ervaringen')
        INTO questions (text_nl, category) VALUES ('Welke fout werd uiteindelijk een waardevolle les?', 'ervaringen')
        -- ============ NATUUR (Nature) ============
        INTO questions (text_nl, category) VALUES ('Welk stukje natuur viel je vandaag op?', 'natuur')
        INTO questions (text_nl, category) VALUES ('Welke plek in de natuur geeft je rust?', 'natuur')
        INTO questions (text_nl, category) VALUES ('Welk seizoen waardeer je het meest en waarom?', 'natuur')
        INTO questions (text_nl, category) VALUES ('Welk dier maakt je blij als je het ziet?', 'natuur')
        INTO questions (text_nl, category) VALUES ('Welke zonsopgang of zonsondergang herinner je je?', 'natuur')
        INTO questions (text_nl, category) VALUES ('Welk geluid uit de natuur vind je rustgevend?', 'natuur')
        INTO questions (text_nl, category) VALUES ('Welke plant of bloem fleurt je omgeving op?', 'natuur')
        INTO questions (text_nl, category) VALUES ('Wat waardeer je aan het weer van vandaag?', 'natuur')
        INTO questions (text_nl, category) VALUES ('Welke wandeling of fietstocht in de natuur herinner je je?', 'natuur')
        INTO questions (text_nl, category) VALUES ('Welk uitzicht vanuit je raam of werkplek waardeer je?', 'natuur')
        INTO questions (text_nl, category) VALUES ('Welk watergebied (zee, meer, rivier) geeft je energie?', 'natuur')
        INTO questions (text_nl, category) VALUES ('Welke geur uit de natuur vind je heerlijk?', 'natuur')
        INTO questions (text_nl, category) VALUES ('Welk park of bos in de buurt bezoek je graag?', 'natuur')
        INTO questions (text_nl, category) VALUES ('Welk natuurfenomeen verbaast je nog steeds?', 'natuur')
        INTO questions (text_nl, category) VALUES ('Welk moment buiten gaf je vandaag frisse energie?', 'natuur')
        INTO questions (text_nl, category) VALUES ('Wat vind je mooi aan de lucht van vandaag?', 'natuur')
        -- ============ ALGEMEEN (General) ============
        INTO questions (text_nl, category) VALUES ('Waar ben je vandaag dankbaar voor?', 'algemeen')
        INTO questions (text_nl, category) VALUES ('Wat maakt jouw leven de moeite waard?', 'algemeen')
        INTO questions (text_nl, category) VALUES ('Welke gewoonte ben je blij dat je hebt ontwikkeld?', 'algemeen')
        INTO questions (text_nl, category) VALUES ('Wat is iets dat je als vanzelfsprekend beschouwt maar eigenlijk bijzonder is?', 'algemeen')
        INTO questions (text_nl, category) VALUES ('Waar kijk je naar uit deze week?', 'algemeen')
        INTO questions (text_nl, category) VALUES ('Welke kans heb je recent gekregen?', 'algemeen')
        INTO questions (text_nl, category) VALUES ('Wat is iets kleins dat je dag een beetje beter maakte?', 'algemeen')
        INTO questions (text_nl, category) VALUES ('Waar ben je trots op in je leven op dit moment?', 'algemeen')
        INTO questions (text_nl, category) VALUES ('Welke vrijheid waardeer je het meest?', 'algemeen')
        INTO questions (text_nl, category) VALUES ('Wat is een simpel plezier dat je nooit verveelt?', 'algemeen')
        INTO questions (text_nl, category) VALUES ('Welke vaardigheid heb je die je leven verrijkt?', 'algemeen')
        INTO questions (text_nl, category) VALUES ('Waar voel je je thuis en waarom?', 'algemeen')
        INTO questions (text_nl, category) VALUES ('Wat zou je missen als het er morgen niet meer was?', 'algemeen')
        INTO questions (text_nl, category) VALUES ('Welke dagelijkse routine geeft je structuur en rust?', 'algemeen')
        INTO questions (text_nl, category) VALUES ('Waarvoor zou je je toekomstige zelf willen bedanken?', 'algemeen')
        INTO questions (text_nl, category) VALUES ('Welk compliment heb je recent ontvangen dat je raakte?', 'algemeen')
        INTO questions (text_nl, category) VALUES ('Wat is er goed gegaan vandaag, hoe klein ook?', 'algemeen')
    SELECT 1 FROM DUAL;

    COMMIT;
    ```

    ![Placeholder: SQL Commands showing 100 rows inserted](images/apex-seed-questions.png " ")

2. Verify the questions were inserted correctly:

    ```sql
    SELECT category, COUNT(*) AS question_count
    FROM   questions
    GROUP BY category
    ORDER BY category;
    ```

    You should see approximately 16–17 questions per category, totaling 100.

    ![Placeholder: Query result showing question counts per category](images/apex-verify-questions.png " ")

## Task 5: Test the Package

Let's verify the business logic works correctly before building the APEX app.

1. Test user registration:

    ```sql
    DECLARE
        v_user_id NUMBER;
    BEGIN
        pkg_journal.register_user(
            p_username     => 'testuser',
            p_email        => 'test@example.com',
            p_password     => 'Test1234!',
            p_display_name => 'Test User',
            p_user_id      => v_user_id
        );
        DBMS_OUTPUT.PUT_LINE('User created with ID: ' || v_user_id);
    END;
    /
    ```

    > **Note:** To see the DBMS_OUTPUT, make sure to click the **DBMS Output** toggle or check the output panel below the SQL Commands area.

    ![Placeholder: SQL Commands showing user created successfully](images/apex-test-register.png " ")

2. Test authentication:

    ```sql
    BEGIN
        IF pkg_journal.authenticate_user('testuser', 'Test1234!') THEN
            DBMS_OUTPUT.PUT_LINE('Authentication successful!');
        ELSE
            DBMS_OUTPUT.PUT_LINE('Authentication failed!');
        END IF;
    END;
    /
    ```

3. Test daily question:

    ```sql
    DECLARE
        v_question_id   NUMBER;
        v_question_text VARCHAR2(500);
        v_category      VARCHAR2(50);
    BEGIN
        pkg_journal.get_daily_question(
            p_user_id       => 1,
            p_question_id   => v_question_id,
            p_question_text => v_question_text,
            p_category      => v_category
        );
        DBMS_OUTPUT.PUT_LINE('Q' || v_question_id || ' [' || v_category || ']: ' || v_question_text);
    END;
    /
    ```

4. Test journal entry submission:

    ```sql
    DECLARE
        v_is_dup  NUMBER;
        v_dup_txt VARCHAR2(4000);
        v_dup_dt  DATE;
        v_dup_pct NUMBER;
        v_entry   NUMBER;
    BEGIN
        pkg_journal.submit_entry(
            p_user_id      => 1,
            p_question_id  => 1,
            p_entry_text   => 'Ik ben dankbaar voor mijn gezondheid en dat ik elke dag mag opstaan',
            p_is_duplicate => v_is_dup,
            p_dup_text     => v_dup_txt,
            p_dup_date     => v_dup_dt,
            p_dup_pct      => v_dup_pct,
            p_entry_id     => v_entry
        );
        IF v_is_dup = 1 THEN
            DBMS_OUTPUT.PUT_LINE('DUPLICATE detected! ' || v_dup_pct || '% similar to entry from ' || v_dup_dt);
        ELSE
            DBMS_OUTPUT.PUT_LINE('Entry saved with ID: ' || v_entry);
        END IF;
    END;
    /
    ```

    ![Placeholder: SQL Commands showing successful entry submission](images/apex-test-submit.png " ")

5. Test duplicate detection (run this **after** the previous test):

    ```sql
    DECLARE
        v_is_dup  NUMBER;
        v_dup_txt VARCHAR2(4000);
        v_dup_dt  DATE;
        v_dup_pct NUMBER;
        v_entry   NUMBER;
    BEGIN
        -- This should be detected as a duplicate (very similar to previous entry)
        pkg_journal.submit_entry(
            p_user_id      => 1,
            p_question_id  => 2,
            p_entry_text   => 'Ik ben dankbaar voor mijn gezondheid en dat ik gezond mag zijn',
            p_is_duplicate => v_is_dup,
            p_dup_text     => v_dup_txt,
            p_dup_date     => v_dup_dt,
            p_dup_pct      => v_dup_pct,
            p_entry_id     => v_entry
        );
        IF v_is_dup = 1 THEN
            DBMS_OUTPUT.PUT_LINE('DUPLICATE detected! ' || v_dup_pct || '% match');
            DBMS_OUTPUT.PUT_LINE('Previous: ' || v_dup_txt);
        ELSE
            DBMS_OUTPUT.PUT_LINE('Entry saved (not expected!) with ID: ' || v_entry);
        END IF;
    END;
    /
    ```

    You should see "DUPLICATE detected!" with a similarity percentage of 85 or higher.

    ![Placeholder: SQL Commands showing duplicate detected](images/apex-test-duplicate.png " ")

6. Clean up the test user and data (we'll create real users through the APEX app):

    ```sql
    DELETE FROM journal_entries WHERE user_id = 1;
    DELETE FROM question_history WHERE user_id = 1;
    DELETE FROM app_users WHERE user_id = 1;
    COMMIT;
    ```

Your database schema is complete and tested! You may now **proceed to the next lab**.

## Acknowledgements

* **Author** - Raoul, Oracle APEX Developer
* **Last Updated By/Date** - Raoul, February 2026

# Lab 1: Provision an Always Free Autonomous Database

## Introduction

In this lab, you will create an Oracle Cloud Infrastructure (OCI) account and provision an Always Free Autonomous Database running Oracle AI Database 26ai with APEX 24.2. This gives you a fully managed Oracle database with APEX pre-installed — at zero cost, forever.

Estimated Time: 20 minutes

### Objectives

In this lab, you will:
- Create an OCI account (if you don't have one)
- Provision an Always Free Autonomous Database
- Access APEX and create a workspace
- Verify your environment is ready

### Prerequisites

This lab assumes you have:
- A valid email address for OCI registration
- A credit card for identity verification (you will NOT be charged)

## Task 1: Sign Up for Oracle Cloud Free Tier

If you already have an OCI account, skip to Task 2.

1. Navigate to [signup.cloud.oracle.com](https://signup.cloud.oracle.com).

    ![Placeholder: OCI signup page](images/oci-signup-page.png " ")

2. Fill in your details:
    - **Country**: Netherlands (or your country)
    - **Name** and **Email**: Your details
    - Click **Verify my email**

3. Complete email verification by clicking the link in the email you receive.

4. Set your **Account Name** (this becomes your tenancy name) and **Home Region**.

    > **Important:** Choose your home region carefully — **Always Free resources are only available in your home region** and it cannot be changed later. For the Netherlands, choose **EU Frankfurt (eu-frankfurt-1)** or **EU Amsterdam (eu-amsterdam-1)** if available.

5. Enter your **address** and **payment verification** (credit card). Oracle will place a temporary hold of approximately €1 that is immediately released. You will not be charged for Always Free resources.

6. Click **Start my free trial**. Your account will be provisioned within a few minutes.

## Task 2: Provision an Always Free Autonomous Database

1. Sign in to the OCI Console at [cloud.oracle.com](https://cloud.oracle.com).

    ![Placeholder: OCI Console login page](images/oci-console-login.png " ")

2. From the hamburger menu (☰), navigate to **Oracle AI Database** → **Autonomous AI Database**.

    ![Placeholder: Navigation menu showing Oracle Database > Autonomous Database](images/oci-nav-autonomous-db.png " ")

3. Click **Create Autonomous Database**.

4. Configure the database with these settings:

    | Setting | Value |
    |---|---|
    | **Display name** | `DankbaarheidDB` |
    | **Database name** | `dankbaarheiddb` |
    | **Workload type** | Transaction Processing |
    | **Always Free** | Toggle **ON** (this is critical!) |
    | **Database version** | 26ai (or newest available) |

    ![Placeholder: Create ADB form with Always Free toggled on](images/oci-create-adb-settings.png " ")

5. Under **Create administrator credentials**, set a strong **ADMIN password**. Save this password — you'll need it shortly.

    > **Note:** The password must be 12-30 characters with at least one uppercase, one lowercase, and one number.

6. Under **Choose network access**, keep **Secure access from everywhere** selected.

8. Click **Create Autonomous Database**.

9. Wait approximately 2–5 minutes for the status to change from **Provisioning** to **Available** (green).

    ![Placeholder: ADB showing Available status with green icon](images/oci-adb-available.png " ")

> **Important:** Always Free databases **auto-stop after 7 days of inactivity**. The daily scheduler job we create in Lab 5 will keep it active. If it stops, you can manually restart it from this page. A stopped database is **permanently deleted after 90 cumulative days** in stopped state.

## Task 3: Access APEX

1. On the Autonomous Database details page, click **Database Actions** → **View all database actions**.

    ![Placeholder: Database Actions dropdown on ADB details page](images/oci-db-actions-dropdown.png " ")

2. In Database Actions, look for **APEX Instance** and click it the instance name DankbaarheidDB. On the Instance page click Launch Apex

    ![Placeholder: Database Actions launchpad showing APEX tile](images/oci-db-actions-apex.png " ")

3. You will be taken to the APEX Administration Services login. Log in with:
    - **Username**: `ADMIN`
    - **Password**: The ADMIN password you set in Task 2

    ![Placeholder: APEX Admin Services login page](images/apex-admin-login.png " ")

## Task 4: Create an APEX Workspace

1. After logging in as ADMIN, click **Create Workspace**.

    ![Placeholder: APEX Admin showing Create Workspace button](images/apex-admin-create-ws.png " ")

2. Choose **New Schema**.

3. Fill in the workspace details:

    | Setting | Value |
    |---|---|
    | **Workspace Name** | `JOURNAL` |
    | **Workspace Username** | `JOURNAL` |
    | **Workspace Password** | Choose a strong password (save it!) |

    ![Placeholder: Create Workspace form filled in](images/apex-create-workspace.png " ")

4. Click **Create Workspace**.

5. Click the link to **sign out** of ADMIN, then **sign in** to your new workspace:
    - **Workspace**: `JOURNAL`
    - **Username**: `JOURNAL`
    - **Password**: The password you just set

    ![Placeholder: APEX workspace login page](images/apex-workspace-login.png " ")

6. You should now see the APEX App Builder home page.

    ![Placeholder: APEX App Builder home page with no applications](images/apex-app-builder-home.png " ")

## Task 5: Verify Your Environment

Let's confirm everything is working correctly.

1. Click **SQL Workshop** → **SQL Commands**.

    ![Placeholder: SQL Workshop navigation](images/apex-sql-workshop-nav.png " ")

2. Run this query to verify your database version:

    ```sql
    SELECT banner_full FROM v$version;
    ```

    You should see something like: `Oracle Database 26ai Enterprise Edition Release 26.0.0.0.0`

    ![Placeholder: SQL Commands showing database version result](images/apex-verify-db-version.png " ")

3. Verify that `FUZZY_MATCH` is available (this is essential for duplicate detection):

    ```sql
    SELECT FUZZY_MATCH(TRIGRAM, 'hello world', 'hello world today') AS similarity FROM DUAL;
    ```

    You should see a number between 0 and 100 (around 70-80 for this test).

    ![Placeholder: SQL Commands showing FUZZY_MATCH result](images/apex-verify-fuzzy-match.png " ")

4. Verify the APEX version:

    ```sql
    SELECT version_no FROM apex_release;
    ```

    You should see `24.2.x` or newer.

    ![Placeholder: SQL Commands showing APEX version](images/apex-verify-apex-version.png " ")

> **Note:** If `FUZZY_MATCH` is not recognized, your database version may be older than 23ai. Check the Autonomous Database details page and ensure you selected version 26ai (or 23ai as minimum). If using an older version, we'll provide a `UTL_MATCH` fallback in Lab 4.

You have successfully provisioned your Always Free Autonomous Database with APEX! You may now **proceed to the next lab**.

## Acknowledgements

* **Author** - Raoul, Oracle APEX Developer
* **Last Updated By/Date** - Raoul, February 2026

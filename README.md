-----

# Tech Stack:

![Snowflake](https://img.shields.io/badge/Snowflake-2C9DCB?style=for-the-badge&logo=Snowflake&logoColor=white)
![GitHub](https://img.shields.io/badge/github-%23121011.svg?style=for-the-badge&logo=github&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/github%20actions-%232671E5.svg?style=for-the-badge&logo=githubactions&logoColor=white)

-----

# Simple Snowflake CI/CD Pipeline with GitHub Actions

This repository demonstrates a simple, effective, and production-ready CI/CD pipeline for managing and deploying Snowflake database objects using GitHub Actions.

The pipeline is designed to be **declarative and idempotent**. It supports two deployment methods:

1.  **Incremental Deployment**: Automatically deploys only the `.sql` files that have been modified in a push.
2.  **Full Deployment**: Manually triggered to deploy *all* `.sql` files, which is useful for rebuilding an environment.

-----

## How It Works ⚙️

The CI/CD process is built around a standard Git branching strategy (`DEV` -\> `main`) and two powerful GitHub Actions workflows.

### Branching Strategy

  * **`main` branch**: This is the **Production** branch. Any commit to this branch triggers the incremental production deployment workflow.
  * **`DEV` branch**: This is the **Development** branch. All new work happens here.

### The CI/CD Pipelines

  * **Incremental Deployment (`prod_deploy.yml`)**:

      * **Trigger**: Runs automatically on every `push` to the `main` branch.
      * **Logic**: Uses `git diff` to find only the `.sql` files that were changed in the push.
      * **Logging**: Records each deployment in the history table with the type `'NORMAL DEPLOY'`.

  * **Full Deployment (`full_deploy.yml`)**:

      * **Trigger**: Must be run **manually** from the GitHub Actions tab.
      * **Logic**: Uses `find` to get a list of *every single* `.sql` file in the `Snowflake` folder, ignoring commit history.
      * **Logging**: Records each deployment in the history table with the type `'FULL DEPLOY'`.

Both pipelines sort the files alphabetically to ensure a correct deployment order (e.g., schemas before tables) and log every success or failure to a history table for auditing.

-----

## Setup Guide 🛠️

> **Note:** A complete SQL script containing all the setup commands below can be found in the `SetupEnv/setup.sql` file in this repository.

To use this project, follow these setup steps.

### 1\. Snowflake Configuration

Run the following SQL in your Snowflake account using an `ACCOUNTADMIN` role to create the necessary objects.

#### A. Create Core Objects

```sql
-- Create a role, warehouse, and database for the pipeline
CREATE ROLE IF NOT EXISTS CICD_ROLE;
CREATE WAREHOUSE IF NOT EXISTS CICD_WH WAREHOUSE_SIZE = 'XSMALL';
CREATE DATABASE IF NOT EXISTS CICD_PROD_DB; -- Or CICD_DEV_DB for a development environment

-- Grant initial privileges
GRANT USAGE ON WAREHOUSE CICD_WH TO ROLE CICD_ROLE;
GRANT USAGE ON DATABASE CICD_PROD_DB TO ROLE CICD_ROLE;
GRANT CREATE SCHEMA ON DATABASE CICD_PROD_DB TO ROLE CICD_ROLE;
```

#### B. Generate Key-Pair Credentials

On your local machine, run the following `openssl` commands to generate a secure, encrypted key pair.

```bash
# 1. Generate an encrypted private key (it will ask for a passphrase)
openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out snowflake_cicd_key.p8 -v2 aes-256-cbc

# 2. Generate the public key from the private key
openssl rsa -in snowflake_cicd_key.p8 -pubout -out snowflake_cicd_key.pub
```

#### C. Create User and Assign Public Key

Copy the content of the `snowflake_cicd_key.pub` file (the text *between* the `BEGIN` and `END` lines) and paste it into the SQL command below.

```sql
USE ROLE SECURITYADMIN;

-- Create the user and assign the public key
CREATE USER IF NOT EXISTS CICD_USER
  RSA_PUBLIC_KEY = 'MIIBIjANBgkqh...' -- <-- PASTE PUBLIC KEY HERE
  DEFAULT_ROLE = CICD_ROLE
  DEFAULT_WAREHOUSE = CICD_WH
  MUST_CHANGE_PASSWORD = FALSE;

-- Grant the custom role to the new user
GRANT ROLE CICD_ROLE TO USER CICD_USER;
```

#### D. Create the Monitoring Table

```sql
CREATE TABLE IF NOT EXISTS CICD_PROD_DB.TECH.DEPLOYMENT_HISTORY (
    DEPLOYMENT_TIMESTAMP TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR,
    COMMIT_SHA VARCHAR,
    GITHUB_ACTOR VARCHAR,
    STATUS VARCHAR,
    TYPE VARCHAR,
    ERROR_MESSAGE VARCHAR
);
```

### 2\. GitHub Secrets Configuration

In your GitHub repository, go to **Settings \> Secrets and variables \> Actions** and create the following secrets.

| Secret Name                        | Description                                          | Example Value                       |
| :--------------------------------- | :--------------------------------------------------- | :---------------------------------- |
| `SNOWFLAKE_ACCOUNT`                | Your Snowflake account locator                       | `xy12345.us-east-1`                 |
| `SNOWFLAKE_USER`                   | The user you created                                 | `CICD_USER`                         |
| `SNOWFLAKE_ROLE`                   | The role you configured                              | `CICD_ROLE`                         |
| `SNOWFLAKE_DATABASE`               | The target database for this environment             | `CICD_PROD_DB`                      |
| `SNOWFLAKE_WAREHOUSE`              | The warehouse to use                                 | `CICD_WH`                           |
| `SNOWFLAKE_PRIVATE_KEY_RAW`        | The **entire content** of your `.p8` file      | `-----BEGIN ENCRYPTED PRIVATE...` |
| `SNOWFLAKE_PRIVATE_KEY_PASSPHRASE` | The password for your private key                    | `Your-Secret-Passphrase`            |

-----

## How to Use 🚀

### Incremental Deployment (Standard Workflow)

1.  **Develop on the `DEV` branch.** Create or modify your `.sql` files.
2.  **Commit and push** your changes to the `DEV` branch.
3.  **Create a Pull Request** from `DEV` to `main`.
4.  **Merge the Pull Request.** This automatically triggers the incremental deployment.

### Full Deployment (Manual Workflow)

Use this for rebuilding an environment or performing a clean installation.

1.  Go to the **Actions** tab in your GitHub repository.
2.  In the left sidebar, click on the **"Full Snowflake Deployment"** workflow.
3.  Click the **"Run workflow"** dropdown button, and then click the green **"Run workflow"** button to start the deployment.

-----

## Folder Structure

The repository is organized to ensure a logical deployment order. The workflow sorts changed files alphabetically, so the **numbered prefixes are crucial**.

```plaintext
.
├── .github/
│   └── workflows/
│       └── prod_deploy.yml   <-- The main deployment workflow
│
├── SetupEnv/
│   └── setup.sql           <-- All setup commands in one file
│
└── Snowflake/
    ├── 00_Schema/          <-- Schemas are created first
    ├── 01_Table/           <-- Tables are created second
    └── 02_View/            <-- Views are created last
```

### Extending the Structure

This structure is fully extensible. You can add folders for any other Snowflake object type you need. Just be sure to use a numbered prefix that reflects the correct order of dependency.

For example, you could add:

  * `03_Stage/`
  * `04_Pipe/`
  * `05_Function/`
  * `06_Procedure/`

The pipeline will automatically pick up `.sql` files in these new folders and deploy them in the correct, sorted order.

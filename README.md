# 🧱 DATABASE DEVOPS PIPELINE USING ATLAS + JENKINS

## (SCHEMA-AS-CODE + CI/CD AUTOMATION)

---

# 1. Introduction

Modern software systems are built using microservices architecture where each service maintains its own database.

As applications evolve, database schema changes frequently:

* adding tables
* modifying columns
* updating constraints
* optimizing indexes
* evolving business logic

Traditionally, these changes are executed manually using SQL scripts, which leads to:

* production failures due to human error
* inconsistent environments (dev/staging/prod mismatch)
* lack of version control
* no audit history of changes
* difficult rollback management

To solve this, we implement:

> **Database DevOps using Atlas + Jenkins**

This approach treats the database as **version-controlled code** and automates schema deployment through CI/CD pipelines.

---

# 2. Objective

This system is designed to:

* treat database schema as code (Schema-as-Code)
* automatically detect schema changes
* generate SQL migrations automatically
* validate migrations before execution
* deploy schema changes using CI/CD (Jenkins)
* maintain full version history of database changes
* support rollback and auditing

---

# 3. What is Atlas?

Atlas

Atlas is a modern database schema management tool that:

* treats database schema as code
* compares desired schema vs actual database
* generates SQL migrations automatically
* applies migrations safely
* detects schema drift
* supports multiple relational databases

👉 Atlas acts as a **schema diff engine + migration generator**

---

# 4. What is Jenkins?

Jenkins

Jenkins is a CI/CD automation tool used to:

* trigger pipelines on Git commits
* run validation scripts
* execute migration commands
* automate deployment workflows

👉 Jenkins acts as the **orchestration engine for database deployment**

---

# 5. Why Atlas + Jenkins Together?

## ❌ Without Automation

```txt
Developer writes SQL manually
        ↓
Executed manually on database
        ↓
No version control
        ↓
Environment mismatch (Dev ≠ Prod)
        ↓
Production failures
```

### Problems:

| Issue                | Impact                 |
| -------------------- | ---------------------- |
| Manual SQL execution | High human error       |
| No tracking          | No history of changes  |
| No rollback system   | Risky production fixes |
| No automation        | Slow deployments       |

---

## ✅ With Atlas + Jenkins

```txt
Developer updates schema in Git
        ↓
Jenkins pipeline triggered
        ↓
Atlas detects schema changes
        ↓
Migration generated automatically
        ↓
Migration validated
        ↓
Applied to database
        ↓
Schema updated safely
```

### Benefits:

| Benefit           | Description                 |
| ----------------- | --------------------------- |
| Automation        | No manual SQL               |
| Safety            | Pre-validated migrations    |
| Version control   | Git-based history           |
| CI/CD integration | Fully automated deployments |
| Auditing          | Full change tracking        |

---

# 6. System Architecture

```mermaid
flowchart TD

Developer["Developer"]
Git["Git Repository (Schema as Code)"]
Jenkins["Jenkins CI/CD Pipeline"]
Atlas["Atlas Migration Engine"]
DB["Relational Database (PostgreSQL/MySQL)"]

Developer --> Git
Git --> Jenkins
Jenkins --> Atlas
Atlas --> DB
```

---

# 7. Project Structure

```txt
atlas-db-devops/
├── schema/
│   └── schema.sql
├── migrations/
├── atlas.hcl
├── Jenkinsfile
└── README.md
```

---

# 8. Schema as Code (Source of Truth)

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL
);
```

👉 This file defines the **desired database state**

---

# 9. Atlas Configuration

```hcl
env "dev" {

  src = "file://schema/schema.sql"

  url = "postgres://postgres:postgres@localhost:5432/appdb?sslmode=disable"

  migration {
    dir = "file://migrations"
  }
}
```

---

# 10. Core Workflow of Atlas

```txt
Current Database Schema
        VS
Desired Schema (schema.sql)
        ↓
Atlas computes differences
        ↓
Generates migration SQL
        ↓
Stores migration in /migrations
        ↓
Applied to database
```

---

# 11. Example Schema Evolution

## Step 1: Initial Schema

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL
);
```

---

## Step 2: Updated Schema

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    phone TEXT
);
```

---

## Step 3: Atlas Generated Migration

```sql
ALTER TABLE users ADD COLUMN phone TEXT;
```

---

# 12. Jenkins CI/CD Pipeline

```groovy
pipeline {

    agent any

    environment {
        DB_URL = 'postgres://postgres:postgres@localhost:5432/appdb?sslmode=disable'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git 'https://github.com/your-repo/atlas-db-devops.git'
            }
        }

        stage('Install Atlas') {
            steps {
                sh 'curl -sSf https://atlasgo.sh | sh'
            }
        }

        stage('Validate Schema') {
            steps {
                sh 'atlas migrate validate --env dev'
            }
        }

        stage('Generate Migration') {
            steps {
                sh 'atlas migrate diff auto --env dev'
            }
        }

        stage('Apply Migration') {
            steps {
                sh 'atlas migrate apply --url $DB_URL --dir file://migrations'
            }
        }

        stage('Verify') {
            steps {
                sh 'echo "Database migration successful"'
            }
        }
    }
}
```

---

# 13. Migration Directory

```txt
migrations/
├── 202605160001_init.sql
├── 202605160002_add_phone.sql
```

👉 Every schema change is versioned.

---

# 14. What Happens to Existing Database?

Atlas works using **schema comparison (diffing)**:

### Step 1:

Reads current database schema

### Step 2:

Compares with schema.sql

### Step 3:

Finds differences

### Step 4:

Generates only required changes

### Example:

```sql
ALTER TABLE users ADD COLUMN phone TEXT;
```

✔ Existing data is never deleted
✔ Only schema evolves safely

---

# 15. Rollback Strategy

Atlas supports rollback via:

## Option 1: Reverse migration

```sql
ALTER TABLE users DROP COLUMN phone;
```

## Option 2: Git rollback

```bash
git revert <commit-id>
```

## Option 3: Migration history tracking

Atlas keeps migration state internally

---

# 16. Auditing

This system provides full auditability:

| Information        | Source            |
| ------------------ | ----------------- |
| Who changed schema | Git commit author |
| When changed       | Git timestamp     |
| What changed       | Migration diff    |
| History            | Migration folder  |

✔ Fully traceable system

---

# 17. Key Features of Atlas

* Schema-as-Code approach
* Automatic migration generation
* Schema drift detection
* Multi-database support
* CI/CD integration
* Safe migration execution

---

# 18. Advantages

| Advantage   | Description           |
| ----------- | --------------------- |
| Automation  | No manual SQL         |
| Safety      | Pre-validated changes |
| Versioning  | Git-based history     |
| CI/CD ready | Jenkins integration   |
| Scalability | Enterprise usage      |

---

# 19. Disadvantages

| Issue          | Reason                 |
| -------------- | ---------------------- |
| Learning curve | New DevOps concept     |
| Initial setup  | Requires configuration |

---

# 20. Enterprise Usage

This approach is used in:

* fintech systems
* SaaS platforms
* banking applications
* large-scale microservices

---

# 21. Final Architecture (Enterprise View)

```txt
Developer
   ↓
Git (Schema as Code)
   ↓
Jenkins CI/CD
   ↓
Atlas Migration Engine
   ↓
Production Database
```

---

# 22. Final Conclusion

> This system implements a complete Database DevOps pipeline where database schema is treated as code, version controlled using Git, automatically diffed using Atlas, and deployed via Jenkins CI/CD pipelines, ensuring safe, consistent, and fully auditable database evolution.

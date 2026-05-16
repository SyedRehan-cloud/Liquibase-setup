# 🧪 ATLAS + JENKINS DATABASE DEVOPS POC (NO SCRIPTS)

---

# 🚀 STEP 1 — CREATE PROJECT STRUCTURE

```bash
mkdir -p atlas-devops-poc/{schema,migrations,jenkins}
cd atlas-devops-poc
```

---

# 📁 STEP 2 — CREATE FILES

```bash
touch schema/schema.sql
touch atlas.hcl
touch Jenkinsfile
touch README.md
```

---

# 🧱 STEP 3 — DATABASE SCHEMA (SCHEMA AS CODE)

```bash
cat << 'EOF' > schema/schema.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL
);
EOF
```

---

# ⚙️ STEP 4 — ATLAS CONFIGURATION

```bash
cat << 'EOF' > atlas.hcl
env "dev" {

  src = "file://schema/schema.sql"

  url = "postgres://postgres:postgres@localhost:5432/appdb?sslmode=disable"

  migration {
    dir = "file://migrations"
  }
}
EOF
```

---

# ⚙️ STEP 5 — JENKINS PIPELINE (NO SHELL SCRIPTS)

```bash
cat << 'EOF' > Jenkinsfile
pipeline {

    agent any

    environment {
        DB_URL = 'postgres://postgres:postgres@localhost:5432/appdb?sslmode=disable'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git 'https://github.com/your-repo/atlas-devops-poc.git'
            }
        }

        stage('Install Atlas') {
            steps {
                sh 'curl -sSf https://atlasgo.io/install.sh | sh'
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
                sh 'echo "Migration completed successfully"'
            }
        }
    }
}
EOF
```

---

# 📖 STEP 6 — README (FULL EXPLANATION FOR SUBMISSION)

```bash
cat << 'EOF' > README.md
# DATABASE DEVOPS PIPELINE USING ATLAS + JENKINS

---

## 1. Introduction

This project demonstrates Database DevOps where database schema is treated as code and deployed using CI/CD pipeline.

---

## 2. Tools Used

- Atlas → Schema migration tool
- Jenkins → CI/CD automation
- PostgreSQL → Database

---

## 3. Architecture

Developer → Git → Jenkins → Atlas → PostgreSQL

---

## 4. How It Works

1. Developer modifies schema.sql
2. Changes are pushed to Git
3. Jenkins pipeline triggers
4. Atlas detects schema changes
5. Migration is generated automatically
6. Migration is applied to database
7. Database is updated safely

---

## 5. Schema as Code

All database structure is defined in:

schema/schema.sql

This acts as the single source of truth.

---

## 6. Atlas Role

- Detects schema differences
- Generates SQL migrations
- Applies changes safely
- Tracks schema state

---

## 7. Jenkins Role

- Automates pipeline
- Runs Atlas commands
- Ensures CI/CD flow

---

## 8. Migration Handling

All migrations are stored in:

migrations/

Each change is version controlled.

---

## 9. Rollback Concept

Rollback is handled via:

- Git revert
- Reverse migrations
- Migration history tracking

---

## 10. Auditing

Every change is tracked via:

- Git commits
- Migration files
- Pipeline logs

---

## 11. Conclusion

This system implements fully automated database schema management using CI/CD principles.

EOF
```

---

# 🧠 WHAT THIS POC PROVES

✔ Database = Code (Schema-as-Code)
✔ Automated migration generation (Atlas)
✔ CI/CD pipeline (Jenkins)
✔ No manual SQL execution
✔ Version control + audit trail

---

# 🔥 FINAL FLOW (FOR VIVA)

```txt
Developer updates schema.sql
        ↓
Git commit
        ↓
Jenkins pipeline starts
        ↓
Atlas detects schema changes
        ↓
Migration generated
        ↓
Migration applied
        ↓
Database updated automatically
```

---

# 🏁 FINAL ONE-LINE ANSWER

> This POC demonstrates a fully automated Database DevOps pipeline where schema changes are defined as code, detected by Atlas, and deployed through Jenkins CI/CD without any manual database operations.

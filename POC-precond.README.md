# LIQUIBASE + JENKINS CI/CD PIPELINE (ENTERPRISE POC – COMPLETE GUIDE)

# 1. Objective

This project demonstrates a complete **Database DevOps CI/CD pipeline** using:

* **Liquibase** for database version control
* **PostgreSQL** as the target database
* **Jenkins** for CI/CD automation
* **XML changelogs** for schema evolution
* **Rollback + validation + audit history**

The POC shows how database schema changes move safely through environments in a controlled and repeatable way.

---

# 2. Architecture Flow

```text
Developer
   ↓
Git Repository
   ↓
Jenkins Pipeline
   ↓
Liquibase Engine
   ↓
JDBC Driver
   ↓
PostgreSQL Database
   ↓
DATABASECHANGELOG Tables
```

---

# 3. Technologies Used

| Technology     | Purpose                                  |
| -------------- | ---------------------------------------- |
| Java 17        | Required runtime for Liquibase + Jenkins |
| Liquibase      | Database migration/versioning tool       |
| PostgreSQL     | Relational database                      |
| JDBC Driver    | Java DB connectivity layer               |
| Jenkins        | CI/CD automation                         |
| XML Changelogs | Declarative DB migration definitions     |

---

# 4. Prerequisites Installation

---

# 4.1 Install Java

```bash
sudo apt update -y
sudo apt install openjdk-17-jdk -y

java -version
```

---

## Why Java is Required

Liquibase and Jenkins are Java applications.

Java provides:

* JVM runtime
* JDBC support
* execution engine for Liquibase

---

# 4.2 Install PostgreSQL

```bash
sudo apt install postgresql postgresql-contrib -y

sudo systemctl enable postgresql
sudo systemctl start postgresql
```

---

# 4.3 Create Database + User

```bash
sudo -u postgres psql
```

Inside PostgreSQL:

```sql
CREATE DATABASE appdb;

CREATE USER admin WITH PASSWORD 'admin';

GRANT ALL PRIVILEGES ON DATABASE appdb TO admin;

\q
```

---

# What These Objects Mean

| Object     | Meaning                   |
| ---------- | ------------------------- |
| appdb      | Application database      |
| admin      | DB user used by Liquibase |
| privileges | Allows schema changes     |

---

# 5. Install JDBC Driver

```bash
wget https://jdbc.postgresql.org/download/postgresql-42.7.5.jar
```

Liquibase package already contains a `lib/` folder.

Move JDBC driver:

```bash
mv postgresql-42.7.5.jar lib/
```

---

# Why JDBC Driver is Required

Liquibase cannot directly communicate with PostgreSQL.

Architecture:

```text
Liquibase
   ↓
PostgreSQL JDBC Driver
   ↓
PostgreSQL Database
```

JDBC translates Java DB calls into PostgreSQL network communication.

Without JDBC:

* no DB connection
* no migrations
* no schema updates

---

# 6. Install Liquibase

```bash
wget https://github.com/liquibase/liquibase/releases/download/v4.27.0/liquibase-4.27.0.tar.gz

tar -xvzf liquibase-4.27.0.tar.gz

cd liquibase-4.27.0
```

Verify installation:

```bash
./liquibase --version
```

---

# 7. Create Project Structure

```bash
mkdir -p liquibase-enterprise-poc/db/changelog

cd liquibase-enterprise-poc
```

---

# FINAL PROJECT STRUCTURE

```text
liquibase-enterprise-poc/
│
├── liquibase
├── liquibase.properties
│
├── lib/
│   └── postgresql-42.7.5.jar
│
└── db/
    └── changelog/
        ├── db.changelog-master.xml
        ├── 001-create-users.xml
        ├── 002-add-email.xml
        ├── 002-seed.xml
        └── 003-safe-update.xml
```

---

# 8. liquibase.properties (CONFIGURATION FILE)

Create file:

```bash
cat << 'EOF' > liquibase.properties
changeLogFile=db/changelog/db.changelog-master.xml
url=jdbc:postgresql://localhost:5432/appdb
username=admin
password=admin
driver=org.postgresql.Driver
searchPath=db/changelog
EOF
```

---

# Purpose of liquibase.properties

This is the central Liquibase configuration file.

It defines:

| Property      | Purpose                  |
| ------------- | ------------------------ |
| changeLogFile | Entry changelog file     |
| url           | PostgreSQL JDBC URL      |
| username      | DB username              |
| password      | DB password              |
| driver        | JDBC driver class        |
| searchPath    | XML file search location |

---

# File Content Explanation

---

## changeLogFile

```properties
changeLogFile=db/changelog/db.changelog-master.xml
```

Meaning:

```text
Start Liquibase execution from this master changelog file
```

This is the orchestration controller.

---

## url

```properties
url=jdbc:postgresql://localhost:5432/appdb
```

Meaning:

| Part       | Meaning              |
| ---------- | -------------------- |
| jdbc       | Java DB connectivity |
| postgresql | PostgreSQL driver    |
| localhost  | DB server            |
| 5432       | PostgreSQL port      |
| appdb      | Database name        |

---

## username/password

```properties
username=admin
password=admin
```

Credentials used by Liquibase.

---

## driver

```properties
driver=org.postgresql.Driver
```

Java class used to load PostgreSQL JDBC driver.

---

## searchPath

```properties
searchPath=db/changelog
```

Liquibase searches XML files inside this folder.

This solved your earlier error:

```text
file not found in configured search path
```

---

# 9. MASTER CHANGELOG FILE

# File: db.changelog-master.xml

```bash
cat << 'EOF' > db/changelog/db.changelog-master.xml
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
       http://www.liquibase.org/xml/ns/dbchangelog
       http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.3.xsd">

    <include file="001-create-users.xml" relativeToChangelogFile="true"/>
    <include file="002-add-email.xml" relativeToChangelogFile="true"/>
    <include file="002-seed.xml" relativeToChangelogFile="true"/>
    <include file="003-safe-update.xml" relativeToChangelogFile="true"/>

</databaseChangeLog>
EOF
```

---

# Purpose of Master Changelog

This is the:

# DATABASE DEPLOYMENT CONTROLLER

It does NOT execute SQL directly.

Instead it says:

```text
Run migration files in this order
```

---

# Why Execution Order Matters

Correct order:

```text
1. Create users table
2. Add email column
3. Insert seed data
4. Update records
```

Incorrect order would fail.

Example:

If seed file runs BEFORE email column exists:

```text
ERROR:
column "email" does not exist
```

This is exactly the issue you encountered during the POC.

---

# Why relativeToChangelogFile="true" Matters

```xml
relativeToChangelogFile="true"
```

Meaning:

```text
Resolve XML file paths relative to current file location
```

This prevents:

* Jenkins path issues
* Docker path issues
* CI/CD execution failures

---

# 10. FILE: 001-create-users.xml

```bash
cat << 'EOF' > db/changelog/001-create-users.xml
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
       http://www.liquibase.org/xml/ns/dbchangelog
       http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.3.xsd">

    <changeSet id="1" author="dev">

        <createTable tableName="users">

            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true"/>
            </column>

            <column name="name" type="VARCHAR(100)"/>

            <column name="contact" type="VARCHAR(20)"/>

        </createTable>

    </changeSet>

</databaseChangeLog>
EOF
```

---

# Purpose of This File

This creates the initial application table.

---

# What Database Object is Created

## Table: users

| Column  | Type         | Meaning        |
| ------- | ------------ | -------------- |
| id      | BIGINT       | Primary key    |
| name    | VARCHAR(100) | User name      |
| contact | VARCHAR(20)  | Contact number |

---

# Important Concepts

---

## changeSet

```xml
<changeSet id="1" author="dev">
```

A changeset is:

```text
One database migration unit
```

Liquibase tracks execution using:

```text
id + author + filename
```

Stored in:

```text
DATABASECHANGELOG
```

---

## createTable

```xml
<createTable tableName="users">
```

Liquibase converts this into PostgreSQL SQL.

Generated SQL:

```sql
CREATE TABLE users (...);
```

---

## autoIncrement

```xml
autoIncrement="true"
```

Database automatically generates IDs.

---

## constraints

```xml
<constraints primaryKey="true"/>
```

Creates PRIMARY KEY constraint.

---

# Generated SQL

Liquibase internally generates:

```sql
CREATE TABLE public.users (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY NOT NULL,
    name VARCHAR(100),
    contact VARCHAR(20),
    CONSTRAINT users_pkey PRIMARY KEY (id)
);
```

---

# 11. FILE: 002-add-email.xml

```bash
cat << 'EOF' > db/changelog/002-add-email.xml
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
       http://www.liquibase.org/xml/ns/dbchangelog
       http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.3.xsd">

    <changeSet id="2" author="dev">

        <preConditions onFail="MARK_RAN">
            <tableExists tableName="users"/>
            <not>
                <columnExists tableName="users" columnName="email"/>
            </not>
        </preConditions>

        <addColumn tableName="users">
            <column name="email" type="VARCHAR(255)"/>
        </addColumn>

    </changeSet>

</databaseChangeLog>
EOF
```

---

# Purpose of This File

This evolves the database schema.

Adds:

```text
email column to users table
```

---

# Why preConditions Are Important

This is an enterprise-grade safety feature.

```xml
<preConditions onFail="MARK_RAN">
```

Liquibase checks conditions BEFORE executing migration.

---

# Condition 1

```xml
<tableExists tableName="users"/>
```

Verify users table exists.

---

# Condition 2

```xml
<columnExists tableName="users" columnName="email"/>
```

Checks if email already exists.

---

# NOT Block

```xml
<not>
```

Meaning:

```text
Execute only if email column DOES NOT exist
```

---

# onFail="MARK_RAN"

If condition fails:

```text
Do NOT crash deployment
Mark migration as executed
```

Very useful in enterprise environments.

---

# Generated SQL

```sql
ALTER TABLE public.users
ADD email VARCHAR(255);
```

---

# 12. FILE: 002-seed.xml

```bash
cat << 'EOF' > db/changelog/002-seed.xml
<databaseChangeLog
 xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xsi:schemaLocation="
 http://www.liquibase.org/xml/ns/dbchangelog
 http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.3.xsd">

 <changeSet id="2-seed" author="dev">

   <insert tableName="users">
     <column name="name" value="admin"/>
     <column name="email" value="admin@test.com"/>
   </insert>

 </changeSet>

</databaseChangeLog>
EOF
```

---

# Purpose of This File

This inserts initial application data.

This is called:

# Seed Data

---

# What Data is Inserted

| name  | email                                   |
| ----- | --------------------------------------- |
| admin | [admin@test.com](mailto:admin@test.com) |

---

# Important Learning from Your POC

Initially this file executed BEFORE:

```text
002-add-email.xml
```

So Liquibase failed:

```text
ERROR:
column "email" does not exist
```

You fixed it by correcting execution order in:

```text
db.changelog-master.xml
```

Correct order:

```text
001-create-users.xml
002-add-email.xml
002-seed.xml
003-safe-update.xml
```

---

# Generated SQL

```sql
INSERT INTO public.users (name, email)
VALUES ('admin', 'admin@test.com');
```

---

# 13. FILE: 003-safe-update.xml

```bash
cat << 'EOF' > db/changelog/003-safe-update.xml
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
       http://www.liquibase.org/xml/ns/dbchangelog
       http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.3.xsd">

    <changeSet id="3" author="dev">

        <preConditions onFail="HALT">
            <tableExists tableName="users"/>
        </preConditions>

        <update tableName="users">
            <column name="contact" value="9999999999"/>
            <where>id = 1</where>
        </update>

    </changeSet>

</databaseChangeLog>
EOF
```

---

# Purpose of This File

Safely updates existing records.

---

# What It Does

Updates:

```text
contact = 9999999999
```

for:

```text
user id = 1
```

---

# Why preConditions Matter Here

```xml
onFail="HALT"
```

If users table does NOT exist:

```text
STOP deployment immediately
```

This prevents dangerous production failures.

---

# Generated SQL

```sql
UPDATE public.users
SET contact = '9999999999'
WHERE id = 1;
```

---

# 14. Liquibase Validation

Run:

```bash
./liquibase validate
```

---

# What validate Does

Checks:

* XML syntax
* changeset structure
* duplicate IDs
* file paths
* changelog references
* execution safety

---

# Your Real POC Issue

You initially saw:

```text
003-add-email.xml was not found
```

Root cause:

```text
master changelog referenced wrong filename
```

Actual file:

```text
002-add-email.xml
```

You corrected the include reference.

---

# 15. updateSQL Command

Run:

```bash
./liquibase updateSQL
```

---

# Purpose

Shows SQL WITHOUT executing migrations.

Useful for:

* DBA review
* audit approval
* enterprise compliance
* dry runs

---

# What You Observed

Liquibase generated:

```sql
CREATE TABLE
ALTER TABLE
INSERT
UPDATE
```

before actual deployment.

---

# 16. update Command

Run:

```bash
./liquibase update
```

---

# What Happens Internally

```text
1. Read master changelog
2. Read included XML files
3. Validate XML
4. Check DATABASECHANGELOG
5. Execute only new changesets
6. Apply SQL
7. Store audit history
```

---

# Final Successful Execution from Your POC

```text
Running Changeset: 001-create-users.xml
Running Changeset: 002-add-email.xml
Running Changeset: 002-seed.xml
Running Changeset: 003-safe-update.xml
```

All 4 changesets executed successfully.

---

# 17. DATABASECHANGELOG Table

Automatically created by Liquibase.

---

# Purpose

Stores migration history.

---

# Example Contents

| id     | author | filename             | status   |
| ------ | ------ | -------------------- | -------- |
| 1      | dev    | 001-create-users.xml | EXECUTED |
| 2      | dev    | 002-add-email.xml    | EXECUTED |
| 2-seed | dev    | 002-seed.xml         | EXECUTED |
| 3      | dev    | 003-safe-update.xml  | EXECUTED |

---

# Why This is Critical

Prevents rerunning migrations.

Provides:

* audit history
* compliance
* rollback tracking
* deployment consistency

---

# 18. DATABASECHANGELOGLOCK Table

Automatically created.

---

# Purpose

Prevents parallel deployments.

Without lock table:

```text
Two Jenkins jobs could modify DB simultaneously
```

This avoids:

* race conditions
* corruption
* inconsistent schema states

---

# 19. Verify Database

Connect:

```bash
psql -U admin -d appdb -h localhost
```

Check tables:

```sql
\dt
```

Check schema:

```sql
\d users
```

Check data:

```sql
SELECT * FROM users;
```

---

# Expected Final Table Structure

| Column  | Type         |
| ------- | ------------ |
| id      | BIGINT       |
| name    | VARCHAR(100) |
| contact | VARCHAR(20)  |
| email   | VARCHAR(255) |

---

# Expected Data

| id | name  | contact    | email                                   |
| -- | ----- | ---------- | --------------------------------------- |
| 1  | admin | 9999999999 | [admin@test.com](mailto:admin@test.com) |

---

# 20. Rollback

Run:

```bash
./liquibase rollbackCount 1
```

---

# What Rollback Does

Reverts last changeset.

Enterprise use cases:

* failed deployments
* production rollback
* emergency recovery

---

# 21. Jenkins Pipeline

# Jenkinsfile

```groovy
pipeline {

    agent any

    environment {
        LIQUIBASE = "./liquibase"
    }

    stages {

        stage('Validate Changelog') {
            steps {
                sh "${LIQUIBASE} validate"
            }
        }

        stage('Preview SQL') {
            steps {
                sh "${LIQUIBASE} updateSQL"
            }
        }

        stage('Run Migration') {
            steps {
                sh "${LIQUIBASE} update"
            }
        }

        stage('Verify Schema') {
            steps {
                sh "psql -U admin -d appdb -c '\\d users'"
            }
        }

    }

    post {

        success {
            echo "Database deployment successful"
        }

        failure {
            echo "Deployment failed"
        }
    }
}
```

---

# Pipeline Flow

```text
Developer Pushes Code
        ↓
Jenkins Triggered
        ↓
Liquibase validate
        ↓
Liquibase updateSQL
        ↓
Liquibase update
        ↓
PostgreSQL Updated
        ↓
Audit History Stored
```

---

# 22. Enterprise Concepts Demonstrated

| Feature               | Demonstrated |
| --------------------- | ------------ |
| Version-controlled DB | Yes          |
| Ordered migrations    | Yes          |
| Schema evolution      | Yes          |
| Seed data             | Yes          |
| Preconditions         | Yes          |
| Safe updates          | Yes          |
| Audit history         | Yes          |
| CI/CD automation      | Yes          |
| Rollback capability   | Yes          |
| Dry-run SQL           | Yes          |

---

# 23. Most Important Learning From This POC

Your POC demonstrated a REAL enterprise migration problem:

```text
Seed migration executed before schema evolution
```

which caused:

```text
column "email" does not exist
```

You fixed it by correcting migration order in:

```text
db.changelog-master.xml
```

This is one of the most important database DevOps concepts:

# DATABASE CHANGE ORDER MATTERS

---

# 24. Final One-Line Understanding

Liquibase is a database version-control and migration framework that safely manages schema evolution through ordered XML changesets, JDBC-based PostgreSQL connectivity, audit tracking, rollback support, and CI/CD automation via Jenkins.

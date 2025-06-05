# Log Analyzer – Architecture 

## Table of Contents
1. [Introduction and Purpose](#1-introduction-and-purpose)
2. [System Overview & High-Level Flow](#2-system-overview--high-level-flow)
3. [Key Technologies & Dependencies](#3-key-technologies--dependencies)
4. [Project Structure](#4-project-structure)
5. [Configuration Management](#5-configuration-management)
6. [Database Design](#6-database-design)
7. [Log Ingestion and Bootstrapping](#7-log-ingestion-and-bootstrapping)
8. [User Authentication & Authorization](#8-user-authentication--authorization)
9. [Web Application Flow](#9-web-application-flow)
10. [Frontend User Interface](#10-frontend-user-interface)
11. [Logging, Security, and Error Handling](#11-logging-security-and-error-handling)
12. [Extensibility & Recommendations](#12-extensibility--recommendations)
13. [System Diagram](#13-system-diagram)
14. [Key Functions & Their Roles](#14-key-functions--their-roles)
15. [Sample .env File](#15-sample-env-file)

---

## 1. Introduction

The **Log Analyzer** is a secure, web-based dashboard for reviewing and analyzing logs produced by ACCESS question-answering (QA) tool. It enables authorized users to:
- Import and store system logs
- View, filter, and tag log entries
- Review and classify queries, responses, and their quality
- Export data for further analysis
- Control access via CILogon SSO and managed allowlists

---

## 2. System Overview & High-Level Flow

1. **Startup**: Loads config, sets up DB, OAuth, and logging.
2. **Log Import**: On first launch or as needed, parses log files and stores entries in a SQLite database.
3. **Authentication**: All users log in via CILogon; role-based access applied.
4. **Dashboard**: Users browse, filter, review, and annotate logs. Some roles can edit data.
5. **Metrics & Analysis**: Real-time filtering, metrics summaries, and visualizations.
6. **Export**: Download filtered data as CSV or XLSX.
7. **Audit Logging**: All user/security events are captured for audits.

---

## 3. Key Technologies & Dependencies

- **Flask**: Web framework
- **Authlib**: OAuth2/CILogon SSO
- **SQLite3**: Lightweight database
- **dotenv**: Configuration/secret loading
- **Plotly**: Interactive charts
- **pandas, csv, xlsxwriter**: For exporting and data processing
- **select2.js, jQuery**: UI enhancements
- **logging**: Application and security/event logging

---

## 4. Project Structure

```text
log-analyzer/
│
├── .env                         # Secrets and config (not in VCS)
|── requirements.txt             # Python package dependencies
├── app.py                       # All backend logic and Flask app
├── qa_log_entries.db            # SQLite database
├── logs/                        # App and access logs
│   ├── app.log
│   ├── unauthorized_access.log
│   └── user_logins.log
├── authorized_users.txt         # Write-enabled users (eppn per line)
├── authorized_users_read_only.txt # Read-only users
├── tags.txt                     # Available tag list
├── templates/
│   └── index.html               # Main dashboard template

```
---

## 5. Configuration Management

- **Secrets** and environment-specific parameters go in `.env`
    - OAuth credentials, DB paths, Flask secret, log directories
    - Loaded at runtime using `dotenv`
- All sensitive values **kept out of code and version control**

## 6. Database Design

### Table: `logs`

| Field                   | Type     | Purpose                                                        |
|-------------------------|----------|----------------------------------------------------------------|
| id (Primary Key)        | INTEGER  | Unique row identifier                                          |
| timestamp               | TEXT     | Log date/time (`%Y-%m-%d %H:%M:%S,%f`)                         |
| query                   | TEXT     | The user’s question/query                                      |
| response                | TEXT     | AI's answer/response                                           |
| is_independent_question | TEXT     | "Yes"/"No"/"" — QA tagging                                     |
| response_review         | TEXT     | "Correct"/"Partially"/"Incorrect"/"I Don't Know"/""            |
| query_review            | TEXT     | "Good"/"Acceptable"/"Bad"/"I Don't Know"/""                    |
| urls_review             | TEXT     | "Good"/"Acceptable"/"Bad"/"I Don't Know"/""                    |
| tags                    | TEXT     | Comma-separated user-applied tags                              |
| ai_generated_tags       | TEXT     | AI-assigned tags (optional)                                    |
| last_updated_by         | TEXT     | Reviewer username/EPPN                                         |
| last_updated_at         | TEXT     | Last review/update timestamp                                   |

> _The table is created if not present at startup. New logs (unique on timestamp + query) are inserted during ingestion._

---

## 7. Log Ingestion and Bootstrapping

- On launch, `init_db()` ensures the database and `logs` table exist.
- The system scans all files matching `simple_qa.log*` (plain or gzipped) in the configured log directory.
- Each eligible log file is parsed to extract:
    - **Timestamp**
    - **Query**
    - **Response**
- Each entry is checked and de-duplicated before insertion into the database.
- **Deduplication** in this context means that, before a log entry is inserted into the database, the system checks if an identical log (defined as having the exact same timestamp and query) already exists. If it does, the system does not insert it again.

---

## 8. User Authentication & Authorization

### SSO Authentication

- **CILogon OAuth2 SSO**:  
  All logins require institutional authentication.

### Role-based Authorization

- **`authorized_users.txt`** for write/users
- **`authorized_users_read_only.txt`** for read-only
- Role is checked at login, stored in the session, and enforced in both the UI and backend.

### Security/Audit Logging

- Unauthorized access and all logins are logged with timestamps, user identifiers, and IPs.

---

## 9. Web Application Flow

### Main Routing

- `/` — Main dashboard (list, filter, review logs, view analytics)
- `/login`, `/authorize`, `/logout` — SSO handling, session management
- `/update_entry` (POST) — Save review/tag changes
- `/get_metrics` (GET, AJAX) — Serve metrics/analytics summary
- `/download_all` (GET) — Export filtered results (CSV/XLSX)
- `/unauthorized` — Shown to denied users

### Filtering

- **Filters**: Date range, review fields, tag fields, review status, “Has Tags” status, etc.
- **Pagination**: 50 logs per page, with prev/next/first/last controls
- **SQL query**: Dynamically built to match user/filters in every request

### Reviewing & Editing

- Each log shown in editable form fields/radio buttons
- Edits trigger AJAX updates (POST `/update_entry`) that update the DB in realtime
- “No” for independent disables related review fields

### Metrics & Visualization

- Metrics: counts, summaries, percentages by review/result
- Interactive Plotly chart showing volume over time by selected granularity (day/week/month)
- Metrics and charts update via AJAX when filters change

### Export

- Download all filtered results as:
    - CSV (streamed)
    - XLSX (pandas DataFrame, with correct column order)
- Export includes all selected filters/fields

---

## 10. Frontend User Interface

- **Responsive dashboard** (HTML/Jinja)
    - Extensive filtering, tag, and review controls
    - In-table editing (write users only)
    - Pagination at top and bottom
- **select2.js** powers tag/option dropdowns for a user-friendly experience
- **AJAX**: All edits, metrics updates, and downloads happen asynchronously
- **Security**: All output is sanitized (bleach) and controls disabled for read-only roles

---

## 11. Logging, Security, and Error Handling

- **Application/Event logs**: `logs/app.log`
- **Unauthorized/Access logs**: `logs/unauthorized_access.log`, `logs/user_logins.log`
- **Field update logging**: All review changes (fields, old/new values, who/when) are logged
- **Error handling**: Exceptions logged with tracebacks, user-facing errors are “flashed” in the UI
- **Sensitive data**: Managed by environment/config files, never stored in VCS

---

## 12. System Diagram

```text
[Log Files] --> [Flask App (app.py)] <--> [SQLite DB]
                                ^        ^         ^
                                |        |         |
                        [User/Browser]   |   [CILogon SSO]
                                |        |
                  [Exports, AJAX, Metrics] 
                                |
                         [Access Logs]

```
---

## 13. Key Functions & Their Roles

Below are the principal backend functions/pipeline stages and descriptions:

### Initialization & Config

- `load_dotenv()`  
  Loads env vars from `.env` for config/secrets.

- `init_db()`  
  On startup; ensures DB exists, creates tables, ingests logs.

### Log Ingestion & Parsing

- `read_logs_from_files()`  
  Reads and aggregates all raw logs from files/directories.

- `parse_log(file, start_date, end_date)`  
  Uses regex to parse single files; extracts log fields.

- `insert_log(conn, log)`  
  Inserts a deduplicated log into DB.

### Tag Handling

- `read_tags()`  
  Parses `tags.txt` and returns the list of tags.

### Metrics & Visualization

- `calculate_metrics(logs, view_by)`  
  Aggregates metrics per filter: daily/weekly/monthly.

- `generate_graph(metrics)`  
  Generates Plotly HTML for the metrics timeseries chart.

- `calculate_review_counts(logs)`  
  Summarizes review distributions for summary dashboard.

### Filtering & Pagination

- `get_paginated_logs(logs, page, per_page)`  
  Handles slicing logic for paginated results.

### User Authorization

- `is_write_user(eppn)`  
  Returns True if user has write access.

- `is_read_only_user(eppn)`  
  Returns True for read-only users.

### Flask Routes

- `home_route()`  
  Main route: filtering logic, DB queries, metric aggregation, and rendering main page.

- `update_entry()`  
  AJAX: Updates review/tags/status for a single log row.

- `get_metrics_endpoint()`  
  AJAX: Returns JSON metrics for current filters.

- `download_all()`  
  Trigger: Exports current filter results as CSV/XLSX.

- `login()`, `authorize()`, `logout()`  
  CILogon SSO and session handling.

- `unauthorized()`  
  Renders the unauthorized page.

### Audit Logging

- Application loggers record:
    - All user logins
    - Unauthorized attempts
    - DB edits with before/after, reviewer, and timestamp
    - Errors and exceptions

---

## 14. Sample .env File

```ini
# .env (sample configuration for Log Analyzer)
# Fill in your own credentials, secrets, and paths

# CILogon OAuth Details
CILOGON_CLIENT_ID=cilogon:/client_id/your-client-id
CILOGON_CLIENT_SECRET=your-cilogon-client-secret

# Flask App Secret
FLASK_SECRET_KEY=replace_this_with_a_secret_key

# Database configuration
DATABASE_PATH=/path/to/your/qa_log_entries.db
DATABASE_URL=sqlite:////path/to/your/qa_log_entries.db

# OAuth/SSO Redirect & Identity Provider Hint
IDP_HINT='https://your-idp.example.org/idp'
REDIRECT_URI='https://your-app-domain.org:2222/authorize'

# Log directories
READ_ONLY_LOG_DIR='/path/to/log/files/'
APP_LOG_DIR='/path/to/app/logs/'

# User allowlist/role files
AUTHORIZED_USERS_FILE='/path/to/authorized_users.txt'

# (Optional) Other settings:
# READONLY_USERS_FILE='/path/to/authorized_users_read_only.txt'
# TAGS_FILE='/path/to/tags.txt'

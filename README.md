# Pen-Repo: Organizational Penetration Testing Reporting Console

Pen-Repo is an enterprise-grade, self-hosted web console built for security operations centers (SOCs), red teams, and security agencies. It simplifies client management, report editing, and report generation in multiple formats (PDF, DOCX, and HTML).

Designed for multi-tenant internal organizational workflows, Pen-Repo enforces segregation of duties between **Pentesters** (authors) and **Admin/Reviewers** (auditors/approvers). It outputs client-ready documentation compliant with EG-CERT reporting guidelines.

---

## Table of Contents
1. [System Architecture & Data Flow](#1-system-architecture--data-flow)
2. [Prerequisites & Host Requirements](#2-prerequisites--host-requirements)
3. [Deployment & Quick Start](#3-deployment--quick-start)
4. [Environment Configuration Reference](#4-environment-configuration-reference)
5. [Creating the First Admin/Superuser](#5-creating-the-first-adminsuperuser)
6. [Account Registration & Admin Approval Workflow](#6-account-registration--admin-approval-workflow)
7. [User Guide: Professional Report Authoring](#7-user-guide-professional-report-authoring)
    - [7.1 Creating Clients & Workspaces](#71-creating-clients--workspaces)
    - [7.2 Drafting Report Scope & Timeline](#72-drafting-report-scope--timeline)
    - [7.3 Auditing Findings & Vulnerability Catalog](#73-auditing-findings--vulnerability-catalog)
    - [7.4 Sorting & Reordering Severity Groups](#74-sorting--reordering-severity-groups)
    - [7.5 Authoring Interactive Proof-of-Concepts (PoCs)](#75-authoring-interactive-proof-of-concepts-pocs)
    - [7.6 Running Verification Retests](#76-running-verification-retests)
8. [Report Exporter Formats](#8-report-exporter-formats)
    - [8.1 Interactive Word Reports (DOCX)](#81-interactive-word-reports-docx)
    - [8.2 Standalone Offline HTML Reports](#82-standalone-offline-html-reports)
    - [8.3 High-Fidelity PDF Documents](#83-high-fidelity-pdf-documents)
9. [Detailed Docker Operations Manual](#9-detailed-docker-operations-manual)
10. [Database Migrations & Maintenance](#10-database-migrations--maintenance)
11. [Troubleshooting & Build Hacks](#11-troubleshooting--build-hacks)

---

## 1. System Architecture & Data Flow

Pen-Repo uses an isolated 4-tier network stack inside Docker. All external traffic is routed through a single Nginx gateway to enforce a single-origin policy:

```
┌────────────────────────────────────────────────────────────────────────┐
│                          Host Browser / Client                         │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Port 80 (HTTP/HTTPS)
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│               Nginx Reverse Proxy (Frontend Container)                 │
│  - Serves compiled React SPA bundle.                                   │
│  - Reverse-proxies /api/* (API) and /super-secret-portal-2026/*        │
│    (Django admin) to the backend.                                      │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Internal pentest_net
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│              Django REST & Gunicorn (Backend Container)                │
│  - Runs business logic, serialization, and RBAC policy checks.        │
│  - Generates HTML templates, base64 encoders, and document exports.   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                     PostgreSQL 16 (DB Container)                       │
│  - Stores clients, projects, vulnerability templates, and users.       │
└────────────────────────────────────────────────────────────────────────┘
```

The database models use a strict relationship hierarchy:
- **Client**: Represents the organization under review. Stores name, metadata, and logo.
- **Report**: Represents a specific engagement for a client. Stores timeline, scoped targets, and executive summaries.
- **ReportVulnerability (Finding)**: An identified security vulnerability linked to the report. Stores descriptions, CVSS scoring, and custom remediation advice.
- **PoCStep / StepImages**: Sequential reproduction steps containing captions, code blocks, primary proof screenshots, and multiple supplementary figures.

---

## 2. Prerequisites & Host Requirements

Before deploying the console, verify that your host system meets the following requirements:
*   **Docker Engine**: Version `20.10.0+` or Docker Desktop.
*   **Docker Compose**: Version `1.29.0+` (supporting the `docker-compose` utility).
*   **Port Availability**: Port 80 must be free on your host. If you have another web server running, stop it or change the `FRONTEND_PORT` variable in your configuration.

---

## 3. Deployment & Quick Start

Follow these steps to deploy Pen-Repo in your organization:

### Step 1: Clone and Enter the Project Root
Extract the project codebase to your local directory:
```bash
cd "Pentesting report project"
```

### Step 2: Configure Environment Variables
Copy the example configuration to create your active environment file:
```bash
cp .env.example .env
```
Open the `.env` file in a text editor and fill in your keys. Refer to [Environment Configuration](#4-environment-configuration-reference) below.

### Step 3: Run the Stack
Run the docker-compose command to build and launch all containers:
```bash
docker-compose up -d --build
```
*Gunicorn starts listening on port `:8000` internally, while Nginx binds to host port `:80` to proxy the frontend React bundle.*

### Step 4: Verify Container Health
Run the status check command to verify that all containers are healthy:
```bash
docker-compose ps
```
You should see:
```
NAME                                 STATUS
pentestingreportproject-backend-1    Up (healthy)
pentestingreportproject-db-1         Up (healthy)
pentestingreportproject-frontend-1   Up (healthy)
```

---

## 4. Environment Configuration Reference

Edit the `.env` file to customize your configuration:

| Variable | Description | Recommended Value |
| :--- | :--- | :--- |
| `DJANGO_SECRET_KEY` | Secret key for Django cryptographic signing | Choose a long, random alphanumeric string |
| `DJANGO_DEBUG` | Enable Django debug warnings | Set to `false` in production |
| `DJANGO_ALLOWED_HOSTS`| Permitted HTTP request hostnames | `localhost,backend,frontend` or your domain |
| `DJANGO_COOKIE_SECURE`| Force transport cookies to require HTTPS | `true` (HTTPS) / `false` (plain HTTP local testing) |
| `DB_NAME` | Database name | `pentest_reporting` |
| `DB_USER` | PostgreSQL user | `postgres` |
| `DB_PASSWORD` | PostgreSQL password | Choose a strong, unique database password |
| `FRONTEND_PORT` | The host port Nginx binds to | `80` (or `8080` if port 80 is occupied) |

---

## 5. Creating the First Admin/Superuser

Because self-registered signups are locked until verified by an administrator, you must bootstrap the system by manually creating the first administrator account.

Run the following command in your terminal:
```bash
docker-compose exec backend python manage.py createsuperuser
```
Follow the interactive prompt:
1. Enter a **Username** (e.g., `admin_reviewer`).
2. Enter an **Email address** (e.g., `admin@myorg.com`).
3. Enter a secure **Password** (minimum 8 characters).
4. Re-enter the password.

Once created:
- Log in to the console at `http://localhost/` using these credentials.
- Access the Django Admin portal at the custom path `http://localhost/super-secret-portal-2026/`.
- Navigate to the **Team & Roles** workspace inside the web console to approve new signups.

---

## 6. Account Registration & Admin Approval Workflow

To prevent unauthorized registrations on public-facing console hosts, Pen-Repo uses an approval flow:

```
[ Signup Form ] ──> [ Saved (Unapproved State) ] ──> [ Login Blocked (403 Forbidden) ]
                                                             │
                                                             ▼ (Admin logs in)
[ Admin Approves via Team & Roles Page ] ───────────> [ Login Enabled (200 OK) ]
```

### 1. Registration Phase
- A user visits `http://localhost/register` and fills out the form.
- The registration UI will redirect to the login screen and display a green notification: *"Your account has been created successfully, but has not been approved by an administrator yet."*
- If the user attempts to log in immediately, authentication is blocked and returns a `403 Forbidden` alert with the same message.

### 2. Verification Phase
- An Admin/Reviewer logs in and goes to the **Team & Roles** page (`http://localhost/users`).
- The unverified registrant will be shown with a red **Not Approved** badge.
- To grant them access, the admin clicks **Approve User**. The status badge changes to a green **Approved** badge.
- The pentester can now log in immediately.
- The admin can demote, promote, or disable the user's access at any time by clicking **Revoke Access**.
- Teammates created directly by the admin in the modal popup bypass this queue and default to **Approved**.

---

## 7. User Guide: Professional Report Authoring

This section outlines how to use Pen-Repo features:

### 7.1 Creating Clients & Workspaces
Every project is grouped under a **Client** profile:
1. Open the sidebar and click **Clients**.
2. Click **Create Client**.
3. Set the Client Name, optional description, and upload a high-resolution logo.
4. The client's logo will automatically render on the cover page of all exported reports.

### 7.2 Drafting Report Scope & Timeline
1. Go to the dashboard and click **+ New Report**.
2. In the creation wizard:
   - Select the Client.
   - Set the Project Name (e.g. *Internal Network Assessment 2026*).
   - Configure the timeline dates and enter the estimated Man-days.
3. Once created, click on the report card to open the editor.
4. Add **Targets in Scope** (one URL or IP per line), a **Limitations** statement, and an **Executive Summary** bullet list.

### 7.3 Auditing Findings & Vulnerability Catalog
1. Click **+ Add Finding** in the report sidebar.
2. Search and select a vulnerability from the master catalog (e.g., *SQL Injection*).
3. The finding card will load the pre-configured details. You can customize the description, remediation advice, and title for this specific project without altering the master template.
4. Set the **Severity** (Critical, High, Medium, Low, Informative) and enter a valid **CVSS Score** (from `0.0` to `10.0`) and the calculator URL.
5. List the **Affected Assets** identified on the network.

### 7.4 Sorting & Reordering Severity Groups
Findings are dynamically sorted by **Severity** (Critical first, down to Informative), then by **Order**, and then by **CVSS Score** (descending).
- **Rearranging Findings**: To adjust the display order of findings, use the up (▲) and down (▼) buttons on the finding cards.
- **Severity Boundaries**: The up and down buttons are restricted to swap order numbers strictly within the finding's severity group. This preserves the severity structure of the final report.

### 7.5 Authoring Interactive Proof-of-Concepts (PoCs)
To provide steps to reproduce:
1. Scroll to the **PoC Steps** section on a finding card.
2. Click **Add Step**.
3. Type the Step Title and Step Description.
4. **Proofs / Screenshots**:
   - Drag and drop or browse to upload a primary screenshot.
   - Enter a descriptive caption. Caption updates are debounced automatically by the UI for 500ms to save your input without flooding the server with API requests.
   - Upload additional images using the **+ Add Image** button below the step to provide multiple proof screenshots.
   - Delete any image (including the main cover photo or step screenshots) by clicking the overlay **Remove** or **×** buttons.
5. **Code & Payload Blocks**: Write code blocks or raw payload parameters. They will render inside custom monospace blocks.

### 7.6 Running Verification Retests
When a client requests a verification retest:
1. Open the report and click **Start Retest**.
2. This duplicates the report data, cloning the project metadata, scoped targets, and custom vulnerability descriptions, remediations, and CVSS calculators.
3. The vulnerabilities will reset their verification states so you can document fixed issues without data loss.

---

## 8. Report Exporter Formats

Pen-Repo generates three professional export formats:

### 8.1 Interactive Word Reports (DOCX)
The DOCX exporter builds reports designed to match the PDF layout:
- **Margins**: Configures page margins to 2.2cm (top), 2.5cm (bottom), and 1.8cm (sides).
- **Interactive Links**: The Table of Contents is fully hyperlinked. Section labels and page numbers link to their target headings.
- **Overview Table**: ID cells (e.g. *Vuln 01*) and Vulnerability Names inside the Finding Overview table link directly to their technical details sections.
- **Shaded Severity Badges**: Shades cell backgrounds matching their severity rating (Red, Orange, Green, Cyan).
- **Code Fences**: Formats code blocks inside light-gray shaded single-cell tables (`#F5F5F5`) with thin margins and Consolas monospace fonts.
- **Headers & Footers**: Hides headers and footers on the cover page. Sub-pages render a document name header and page number footer.

### 8.2 Standalone Offline HTML Reports
- **Inline Base64 Encoding**: Encodes the client logo, cover image, and step screenshots on-the-fly into base64 data URIs.
- **Offline Compatibility**: Downloaded HTML files are self-contained and run locally on any system with all images rendered correctly.

### 8.3 High-Fidelity PDF Documents
- **Engine**: Rendered server-side using WeasyPrint.
- **Features**: Generates printable PDF documents with EG-CERT headers, page breaks, and embedded threat charts.

---

## 9. Detailed Docker Operations Manual

Use these docker-compose commands to manage the application lifecycle:

### Rebuild and Run Stack
Run this command to build the containers and start them in the background:
```bash
docker-compose up -d --build
```

### Stop Containers (Gracefully)
Stops execution of the services without destroying the volumes or database state:
```bash
docker-compose stop
```

### Start Containers
Resumes execution of stopped services:
```bash
docker-compose start
```

### Shut Down Stack
Stop and delete the container instances and the virtual network:
```bash
docker-compose down
```

### Purge All Volumes
To wipe the database state and delete all uploaded media files to start fresh:
```bash
docker-compose down -v
```

### Check Service Status
```bash
docker-compose ps
```

### Fetch Backend Logs
```bash
docker-compose logs -f backend
```

---

## 10. Database Migrations & Maintenance

To run database maintenance scripts inside the running backend container:

### Generate Schema Migrations
```bash
docker-compose exec backend python manage.py makemigrations
```

### Apply Schema Migrations
```bash
docker-compose exec backend python manage.py migrate
```

### Run Custom Database Seeder
To repopulate or seed the master catalog of vulnerabilities:
```bash
docker-compose exec backend python manage.py seed_vulnerabilities
```

---

## 11. Troubleshooting & Build Hacks

### Offline Build DNS Issues
When building in an offline network environment, BuildKit may fail to check base image metadata and return: `dial tcp: lookup registry-1.docker.io: no such host`.
*   **Fix**: Both frontend and backend Dockerfiles are pinned to local image cache digests. If BuildKit still attempts to lookup remote shas, disable BuildKit for the build command:
    ```powershell
    $env:DOCKER_BUILDKIT=0
    docker-compose build
    ```

# Luciano Vernazza — Portfolio

🇦🇷 [Leer en español](README.es.md)

**Project Manager · Data Migrations & Technical Onboarding** at a SaaS practice-management platform for plastic surgery and aesthetic clinics · Argentina

I work at the point where a clinic leaves its old system and starts running on a new platform. That means moving years of patients, appointments, and clinical documents, and building the forms the clinic will use from day one. My job is to make that process predictable. I design and build tools that replace manual steps, catch problems before they reach the client, and give leadership hard data to make decisions.

> All people and client names have been anonymized. Metrics tied to individuals are shown as percentages only.

---

## Contents

1. [Context: what a migration looks like](#1-context-what-a-migration-looks-like)
2. [Prevalidation App (Flask)](#2-prevalidation-app-flask)
3. [Prevalidation Notebooks on Azure Databricks](#3-prevalidation-notebooks-on-azure-databricks)
4. [TAP Mapper — CLI + Claude Skill](#4-tap-mapper--cli--claude-skill)
5. [Intake Form Editor](#5-intake-form-editor)
6. [EHR Template Converter](#6-ehr-template-converter)
7. [Fillable PDF Pipeline](#7-fillable-pdf-pipeline)
8. [Consent Form Batch Uploader](#8-consent-form-batch-uploader)
9. [n8n Automation](#9-n8n-automation)
10. [Migration Project Tracker](#10-migration-project-tracker)
11. [Project Database Updater — Claude Skill](#11-project-database-updater--claude-skill)
12. [Performance Analytics & KPIs](#12-performance-analytics--kpis)
13. [Orphaned Clinical File Matcher](#13-orphaned-clinical-file-matcher)
14. [Support Scripts](#14-support-scripts)
15. [Documentation & Knowledge Transfer](#15-documentation--knowledge-transfer)
16. [Tech Stack](#16-tech-stack)

---

## 1. Context: what a migration looks like

Every migration goes through four stages:

- **Staging:** a first trial load into the client's account.
- **Gap:** the new data created since staging.
- **Go-Live:** the final load.
- **Files & Media:** clinical documents, images, and attachments.

Data arrives from many different EHRs and is rarely clean: CSVs with misplaced quotes and commas, mixed encodings, columns named differently in every system, inconsistent values, and in some cases tens of thousands of distinct appointment types.

When I started, much of this process depended on manual steps, knowledge held by a few people, and tools only a technical person could use. Each project on this page tackles a specific part of that problem.

| Stage | Tools |
|---|---|
| Data intake & prevalidation | Prevalidation App, Databricks notebooks |
| Appointment mapping | TAP Mapper |
| Forms (intake, consent, clinical templates) | Intake Form Editor, EHR Template Converter, Fillable PDF Pipeline, Batch Uploader |
| Files & Media | Clinical file matcher, validation reports |
| Project tracking | n8n, Migration Project Tracker, Project Database Updater |
| Leadership | Linear exporter, performance dashboards, forms KPIs |

---

## 2. Prevalidation App (Flask)

**Stack:** Python · Flask · pandas · openpyxl · Built with Claude Code

### The problem
Before migrating, the client's files have to be checked for blank fields and values with no equivalent on the platform (gender written five different ways, phone numbers missing digits, impossible birth dates). That check ran in Azure Databricks notebooks, which had several problems:

- You had to know how to configure widgets and pick the right notebook depending on how many files the client sent.
- Column mapping was done by hand on every project.
- Appointment-type mapping was a separate process, in a separate file.
- Product Specialists, who validate that mapping with the client, couldn't take part without asking a technical person to run everything.

### The goal
Let anyone on the team, technical or not, run a full prevalidation without touching code. I also wanted the process to be the same every time, so results wouldn't depend on who ran it, and to let the Product Specialist step in for their part without waiting on anyone.

### How it was solved
I built a local web app that guides the process in five steps, with two roles:

- **Project Manager:** full access, from creating the client to downloading reports.
- **Product Specialist:** goes straight to appointment mapping and sees earlier steps in read-only mode.

**The flow:**

1. **File upload.** The app detects which configuration applies:

   | Files | Mode |
   |---|---|
   | Patients | Patients-Only |
   | Patients + appointments | Standard |
   | Patients + 2 appointment files | Dual Appointments (`[A]` / `[B]` columns) |
   | 2 patient files | Dual Patients |

2. **Column mapping.** Client columns are matched to the 24 standard patient fields and 15 appointment fields. The app suggests the mapping based on the column names each EHR typically uses.
3. **Results.** Each field is classified as ok, blank, or conflicting. Conflicts open in a modal, where mapping rules are created ("M", "Masc", and "male" all go to Male). By design, a conflicting value can't be the target of a rule.
4. **Appointment mapping**, integrated with the TAP Mapper (section 4).
5. **Reports:** a Word report for the client, CSVs of problem records, the appointment mapping file, and an Excel findings summary.

Parsing was the hardest part:

- **Matching production.** The same CSV gave different results depending on which program opened it, and we found that LibreOffice interpreted quotes differently. The app replicates how the production system reads the file.
- **Dirty files.** It detects encoding (UTF-8, UTF-8 with BOM, UTF-16) and delimiter. For corrupted Excel files it tries three engines in sequence: openpyxl, xlrd, and `read_html`.
- **Realistic tests.** I generated 14 test CSVs from real client mapping rules, so the app would face the same problems each EHR sends.

### The impact
Prevalidation no longer depends on knowing Databricks. The flow is the same every time, and the Product Specialist works on their part without intermediaries. Appointment mapping became a step within the same process instead of a separate file. The app was demoed to the CTO, who asked for the repository to build a deployment pipeline and take it from local use to a team service. When the code was accidentally lost, it could be rebuilt thanks to a 16-section context document.

---

## 3. Prevalidation Notebooks on Azure Databricks

**Stack:** PySpark · pandas · SQL · Unity Catalog

### The problem
The notebooks were the team's prevalidation tool, but they failed on exactly the hardest clients:

- **They didn't cover every delivery format.** Some EHRs send appointments in two separate files (type and status in one, everything else in the other).
- **pandas silently dropped rows broken by misplaced quotes.** Validation reported everything was fine because the problem rows simply didn't exist.
- **Some Excel files broke the conversion to Spark** because of mixed data types.
- **Certain queries hung indefinitely** when the client's catalog was misconfigured.
- **The findings summary took forever** on large files, because it was written cell by cell.

### The goal
Make the notebooks reliable with any file and make sure they never returned a false "all clear". I also wanted them to stop hanging and to cover every delivery variant we handled, without changing the validation logic that already worked.

### How it was solved
- **Custom parser for damaged CSVs.** It keeps every row, even broken ones, so shifted values show up as conflicts instead of disappearing.
- **Excel read as text** (`dtype=str`), with fallback between engines for corrupted files.
- **60-second timeout** with `ThreadPoolExecutor` on Spark queries. If the catalog is wrong, the notebook reports it instead of hanging.
- **Row caps on the Excel detail sheets**, with a pointer to the full CSV.
- **Appointment types and purposes read from the client's actual table**, instead of from tables that didn't exist in every catalog.
- **Three variants maintained in parallel:** standard, one adapted to a specific EHR, and one for clients who send two appointment files. Every change is applied consistently across all three.

### The impact
The notebooks stopped producing false positives on malformed files, which were exactly the ones most likely to break a migration. They stopped hanging and started covering clients that previously had to be processed by hand. All the parsing lessons from this work were later carried over to the Prevalidation App.

---

## 4. TAP Mapper — CLI + Claude Skill

**Stack:** Python · SQLite · openpyxl

### The problem
Every clinic names its appointments its own way. Each appointment type and purpose from the old system has to be translated into the platform's standard values ("Botox f/u 2wk" becomes a neurotoxin follow-up, for example). Some clients have hundreds of values; others have more than 10,000.

It was done by hand, row by row, which caused three problems:
- It took a very long time.
- Two people would map the same value differently.
- Corrections made by specialists weren't reused for the next client.

### The goal
Automate most of the mapping and have the tool learn from every project, so each new client starts with more already mapped. It also had to be clear which rows needed human review, and why.

### How it was solved
- **A CLI with a SQLite knowledge base** fed by historical mappings. Product Specialist corrections carry three times the weight of other inputs, because they are the validated reference.
- **Four commands:** `map` (maps a new file and learns from it), `import` (loads historical files), `corrections` (learns from corrections), and `status`.
- **Multi-pass matching logic:** procedure-specific terms first, then broader clinical patterns, then generic follow-up language, and administrative cases last. Combined appointments go to the primary procedure.
- **Color-coded confidence levels:** high, needs review, or no match. Every uncertain row carries a comment explaining the decision, so the specialist reviews only what's needed.
- **A Claude Skill** to run it in natural language, in English or Spanish.

### The impact
Mapping files come out with 90% to 100% automatic coverage, even for clients with more than 10,000 purposes. Human review focuses on the flagged rows. The specialist gets an annotated file instead of a blank spreadsheet, and every correction improves the result for the next client. It has been used on more than 10 clients.

---

## 5. Intake Form Editor

**Stack:** Flask · React

### The problem
Intake forms (the ones patients fill out before a visit) are stored as JSON documents in a NoSQL database. Changing a question, adding a field, or making one required meant editing the JSON by hand in a code editor and uploading it through Postman.

This caused several problems:
- Only a technical person could do it.
- A syntax error broke the form.
- There was no way to see the result before publishing.
- Every client change request turned into a small project.

### The goal
Make editing a form as simple as editing a document. I wanted to see the result while editing, without touching JSON and without any risk of breaking the structure.

### How it was solved
A visual editor with a bidirectional parser: it reads the real JSON, turns it into editable state, and on export generates the complete JSON, including technical fields.

- **Import** from `.json` or `.http` files, automatically stripping HTTP headers.
- **Section sidebar.** Fields are grouped by row, so two fields on the same row appear side by side, just like in the real form.
- **Per-field editing:** type (text, date, phone, yes/no, dropdown, checkbox, signature, grid, subheader), question, required flag, options, position, and which section of the clinical record the answer maps to.
- **Rich text** for consent wording.
- **Preview** on iPhone, iPad, Android, and desktop.
- **Drafts** to pick up work later.
- **REST API** with form CRUD and import/export endpoints.

It was validated with a full round trip (import and re-export) on a real form with 150+ fields and 12 sections, with no data loss.

### The impact
Editing a form no longer requires JSON knowledge and no longer risks breaking it. The preview lets you check the result before delivering it, and client change requests are handled in the editor instead of in code. It also opens the door for non-technical team members to make adjustments they previously had to escalate.

---

## 6. EHR Template Converter

**Stack:** Claude Skill · Python · poppler · JavaScript

### The problem
Clinics send their clinical note templates as PDFs, and they had to be rebuilt in the platform's editor. Before building anything I ran a technical investigation (internal documentation, team conversations, and tickets) and found:

- **The process was entirely manual.** Each PDF was rebuilt click by click in an editor with limitations (for example, tables capped at 10×10).
- **There was no API** for loading templates programmatically.
- **There was no way to copy templates between accounts.** This was a real problem for groups of clinics that share templates.
- **The team built templates for almost every client**, so the bottleneck was ours.

### The goal
Move from rebuilding each template by hand to converting it automatically. I also wanted to process whole batches and copy templates between accounts without redoing them.

### How it was solved
- **Reverse-engineering the editor.** Through systematic testing I found out which HTML survives being pasted into the TinyMCE editor:
  - native checkboxes
  - non-editable fill-in fields
  - colors with `data-mce-style` attributes
  - base64 images
  - cell backgrounds (these get stripped, so the workaround is wrapping the content in a colored `span`)
- **A Claude Skill** that takes a folder of PDFs and returns a folder of HTML files ready to paste. It rasterizes each page, extracts embedded images, and generates HTML that follows those rules.
- **A bookmarklet** that exports the template open in one account as an HTML file, so it can be pasted into another.
- The skill was packaged so other team members can install and use it.

### The impact
Template creation went from rebuilding each PDF by hand to converting whole batches and reviewing the result. Copying templates between accounts, which used to mean rebuilding them, is now export and paste. And the knowledge of what the editor accepts is captured in the skill, instead of depending on the memory of whoever discovered it.

---

## 7. Fillable PDF Pipeline

**Stack:** Python · PyMuPDF

### The problem
Informed consent forms arrive as flat PDFs. For the platform to fill in patient data automatically (name, date of birth, provider, appointment date) and collect digital signatures, every field has to exist as a form field with the exact name of the matching macro.

Doing this by hand in a PDF editor, form by form, took a lot of time, and clients usually send dozens of consent forms at once.

### The goal
Process whole batches of consent forms at once. The aim was to automatically detect where each piece of data and each signature goes, without covering any text in the original document and without changing its design.

### How it was solved
A five-stage pipeline built on an internal field-generation skill:

1. **Extraction.** Detects fill-in lines, whether underscores or drawn lines, and links them to their label, whether it sits above, below, or beside the line.
2. **Cleanup.** Removes false positives from body text and links labels that end up in separate text blocks.
3. **Signatures.** Handles the common pattern of a signature written above a line with the label underneath.
4. **Overlaps.** Adjusts each field so it doesn't cover any text.
5. **Batch apply and verification.** Generates the final PDFs and checks that everything landed in the right place.

### The impact
Batches of 40 consent forms, with 300+ fields across text, signatures, and checkboxes, are processed in a single run. What used to be manual work, form by form, became a review of the output. Each form keeps its original design, and the fields are ready for the platform's auto-fill.

---

## 8. Consent Form Batch Uploader

**Stack:** Python CLI · REST API

### The problem
Once prepared, consent forms had to be uploaded one by one to the client's account through the web interface. With batches of dozens of forms, this took a long time and was error-prone.

### The goal
Upload an entire folder in one go, with the option to simulate the upload before running it.

### How it was solved
I picked up a tool a colleague had started and turned it into an interactive CLI:
- You drag the folder into the terminal.
- You look up the client's account by name or ID.
- You log in and see a summary before confirming.
- Options: `--dry-run` to simulate, delay between uploads, extension filters, and the forms' initial status.
- It includes a set of 10 test PDFs.

### The impact
The batch upload flow is solved on the tool side. Production use is blocked by a bug in the platform's API, which has been identified and documented for the development team.

---

## 9. n8n Automation

**Stack:** n8n · Linear · Notion · Slack · Fellow.ai

### The problem
Information about each migration lived in several places: tasks in Linear, tracking in Notion, communication in Slack, and decisions in meetings. Keeping it all in sync was manual. This caused several problems:
- Statuses went stale.
- Nobody found out in time when a task got blocked.
- Client-facing project managers had to keep asking where each project stood.

### The goal
Make Linear the single source of truth and have everything else update itself. Relevant notifications had to reach Slack without anyone writing them.

### How it was solved
**Workflows in production:**
- Automatic issue labeling in Linear
- Automated Slack messages
- Alerts when a migration is ready for validation
- Project status transitions
- Automatic sub-task creation based on the source EHR

**Workflows designed:**
- A router that syncs Linear to Notion
- Phase-transition handling
- Milestone calendar sync
- Automatic "blocked since" timestamping (every 15 minutes)
- A daily Slack digest
- Upcoming-milestone alerts

**Architecture decisions:**
- The project phase is derived from active tasks, not from project status, because the status was updated late.
- Each issue syncs 1:1 as a task, which removed an entire workflow.
- Native Notion nodes are used, with the Linear ID as the key to avoid duplicates.

For the pending workflows I left a field-mapping spec, a design document, and a brief for Claude Code to build them autonomously.

### The impact
Repetitive labeling, notification, and status-change tasks are no longer done by hand. The team hears about changes in Slack as they happen. And the full architecture is defined for Notion project tracking to update itself from Linear. Along the way I went from using n8n for the first time to designing a system of coordinated workflows.

---

## 10. Migration Project Tracker

**Stack:** Notion

### The problem
The project managers who guide each client don't work in Linear and had no simple view of what stage each migration was in or which milestones were coming up. The information existed, but it was scattered and formatted for developers.

### The goal
A single view of the status of every migration, adapted to whoever is looking: calendars for project managers and task detail for developers.

### How it was solved
I designed a Notion system from scratch:
- **Two related databases:** projects and tasks.
- **A milestones database** and a general calendar.
- **Dedicated views:** a calendar for each project manager and detail views for developers.

It is fed automatically from Linear through the n8n workflows and posts notifications to Slack.

### The impact
Each project manager sees their clients' milestones on their own calendar without asking for updates. Developers and project managers look at the same information, each in the format that works for them, and the "where is that migration?" questions go away.

---

## 11. Project Database Updater — Claude Skill

**Stack:** Claude Skill · MCP (Notion, Linear, Fellow)

### The problem
The entire implementation team uses a shared Notion database with the status of each client. Its data migration section has 24 fields: status, staging and go-live dates, appointment mapping approval, full export receipt, and so on. They were filled in by hand, so in practice they were empty or out of date, and other teams made decisions with stale information.

### The goal
Keep those fields current with no manual work, pulling the information from where it actually lives: task status in Linear and what gets decided in meetings.

### How it was solved
A Claude Skill that cross-references Linear, Fellow meeting summaries, and Notion:
- **`initial` mode:** backfills the history of every active project.
- **`weekly` mode:** updates only what changed during the week.
- **Approval first:** proposes changes in a table and waits for approval before applying them.

Lessons from the first run were built into the skill as rules:
- Always check Linear first, even if the Notion field is empty.
- An empty date doesn't mean the event didn't happen.
- Certain labels are equivalent.
- Meetings are reviewed on every run.

### The impact
The full backfill of every active project is done, and the database has been updated weekly since then. Other teams now consult data that reflects the real status. The same integration was used to reassign dozens of projects in a single operation when the project manager team changed, keeping the rest of their information intact.

---

## 12. Performance Analytics & KPIs

**Stack:** Python · Linear GraphQL API · React · PowerPoint · Notion

### The problem
Leadership needed answers to concrete questions: how long a migration takes, how much rework there is, whether the forms team has capacity for more clients, and where work gets stuck. The data was in Linear, but the standard export wasn't usable:

- **Times didn't reflect actual work.** They were calculated in clock hours, including nights and weekends.
- **Issues were missing.** Archived ones were left out.
- **There was no direct way to count the forms** attached to each task.

### The goal
Have reliable metrics, comparable over time, to evaluate team performance, find bottlenecks, and report to leadership with data rather than impressions.

### How it was solved
- **A Linear exporter via GraphQL:**
  - Filters on the server side to avoid pulling unnecessary data.
  - Walks each issue's full history to calculate how long it spent in each status.
  - Measures in business hours and includes archived issues.
  - Counts attached files by reading the embeds in the description, including files inside ZIPs.
- **Performance dashboards** in React and PowerPoint decks covering project duration, time per task type, and validation rounds.
- **Forms KPIs in Notion:**
  - Dataset cleanup: IQR outlier exclusion and removal of tasks that skipped statuses.
  - Charts for task inflow, completed volume, efficiency by form type, and backlog.
- **A React artifact** that queries Linear through MCP and exports to CSV without handling API keys.

### The impact
- **Data-backed performance reviews.** In one of them, the dashboards showed a 60% to 80% reduction in rework over time.
- **The forms bottleneck was queue wait time, not working time.** The capacity analysis showed it, and it shifted the discussion from "work faster" to "how work gets assigned".
- **Ongoing tracking.** The KPI dashboard became a permanent tool.

---

## 13. Orphaned Clinical File Matcher

**Stack:** Python · pandas

### The problem
During the Files & Media stage of one migration, hundreds of clinical documents (RTF notes and EML emails) turned up that couldn't be linked to any patient, so they couldn't be migrated. The first attempt, matching on the file path, found no matches at all.

### The goal
Recover as many of those files as possible by linking them to their patient, and understand precisely why the rest couldn't be linked, so the client could be informed with data.

### How it was solved
I built the matching script and two diagnostic scripts:
- **The first diagnostic** confirmed that paths had the same format in both files, so formatting wasn't the problem.
- **The second** found the real key: the numeric document ID, which appears both in the file name and in the source system's document index.

With that, the script matches in two steps: file → patient in the document index, and patient → ID on the new platform.

### The impact
Identification of files in the source index went from 0% to roughly 95%. The analysis also showed that the real bottleneck was the second step: many patient names didn't match the demographic report. That turned a "can't be migrated" into a concrete, scoped problem to solve with the client.

---

## 14. Support Scripts

Smaller tools that solved specific day-to-day needs:

- **Test CSV generator.** To test the Prevalidation App I needed data that fails the way real data does. I reviewed the documented mapping rules of several clients and generated 14 synthetic files that reproduce each EHR's typical problems. It made it possible to test the app without using patient data.
- **SFTP file counter.** Confirming that a client uploaded everything before starting Files & Media meant counting files on servers with thousands of folders. The script walks the server recursively and prompts for credentials at runtime, with no arguments, so they never end up in the terminal history.
- **Excel merger.** Some systems export data split across many files. The script merges them after checking that they all share the same columns, so a corrupted combined file is never produced unnoticed.
- **Export validation reports.** Files & Media reports follow a Word template that can't be altered. They are filled in by editing the document's XML directly, which keeps the exact formatting and avoids the inconsistencies of doing it by hand.

---

## 15. Documentation & Knowledge Transfer

### The problem
Much of the operational knowledge (how certain migration types are done, what to ask each EHR for, how to explain the process to a client) lived in a few people's heads or was scattered across meeting recordings. When a key person left the team, that risk became urgent.

### The goal
Turn that knowledge into documentation anyone could follow, and have material ready for the internal team as well as for clients and sales.

### How it was solved
- **Operational runbooks** for three migration types, built from the history of recorded meetings.
- **Migration guides for clients and sales**, in two versions: a complete technical one, and a simplified one with custom diagrams for clients who don't need the detail.
- **Project plans and kickoff decks.**
- **Context documents** that make it possible to rebuild or continue an app in a new Claude Code session.

### The impact
The role handover happened without losing operational knowledge. Sales could use the guides in conversations with prospects, and the team has a shared reference instead of depending on asking the one person who "knows".

---

## 16. Tech Stack

| Area | Technologies |
|---|---|
| Languages | Python, JavaScript/React, SQL, PowerShell |
| Libraries | pandas, openpyxl, PyMuPDF, reportlab, paramiko, Flask |
| Data | Azure Databricks (PySpark, Unity Catalog), SQLite |
| Automation | n8n, Claude Skills, Claude Code, MCP |
| Tools | Linear (GraphQL), Notion, Slack, Fellow.ai |

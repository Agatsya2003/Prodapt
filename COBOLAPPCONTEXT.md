# COBOL App Context Pipeline — Developer KT Document

## Table of Contents
1. [Overview](#1-overview)
2. [High-Level Pipeline Flow](#2-high-level-pipeline-flow)
3. [Entry Point & API Trigger](#3-entry-point--api-trigger)
4. [COBOL Parsing Layer](#4-cobol-parsing-layer)
5. [LLM Extraction — Azure OpenAI GPT-4.1-mini](#5-llm-extraction--azure-openai-gpt-41-mini)
6. [Database Storage (Table Mapping)](#6-database-storage-table-mapping)
7. [Design Document Generation](#7-design-document-generation)
8. [App Context Aggregation](#8-app-context-aggregation)
9. [SharePoint Integration](#9-sharepoint-integration)
10. [Cost Tracking](#10-cost-tracking)
11. [Key Dependencies & Frameworks](#11-key-dependencies--frameworks)
12. [Environment Variables](#12-environment-variables)

---

## 1. Overview

This pipeline ingests COBOL mainframe source code (`.cbl` files), extracts structured application context using LLMs, persists it to a MySQL database, generates a Word design document, and uploads artifacts to SharePoint.

**What it produces:**
- Structured context for every COBOL program section stored in DB (reusing the same table structure as SpringBoot services)
- A generated design document (`.docx`) describing the COBOL application
- COBOL source + analysis artifacts uploaded to SharePoint
- Aggregated application-level context in the master table

**Key Note on Table Reuse:**
The pipeline reuses the same DB tables used for Java/SpringBoot app context. The semantic mapping is:
- COBOL **section** → stored as an "API" in `application_api_details`
- COBOL **program orchestration** → stored as a "Camunda workflow" in `application_camunda_details`
- COBOL **copybooks** → stored in `application_sql_table_details`
- COBOL **CICS screens** → stored in `application_ui_details`

---

## 2. High-Level Pipeline Flow

```
User uploads COBOL ZIP (via API)
        │
        ▼
Service type detected as 'cobol'
(detects .cbl / .cpy / .dat extensions)
        │
        ▼
modules/process_cobol_main.py
        │
        ├─── 1. File Discovery
        │         read_all_cobol_files_recursively() → list of .cbl files
        │
        ├─── 2. Per-file COBOL Parsing
        │         parse_cobol_program() → structured metadata JSON
        │
        ├─── 3. LLM Call #1: Document JSON
        │         call_openai() → document.json
        │         (sections, orchestration, copybooks, screens, working storage)
        │
        ├─── 4. LLM Call #2: Enrichment
        │         generate_enrichment_data() → code_quality, complexity, LOC,
        │         business_significance, tables_linked, camunda details
        │
        ├─── 5. DB Inserts
        │         build_and_execute_inserts()
        │         → application_api_details (sections)
        │         → application_camunda_details (orchestration)
        │         → application_sql_table_details (copybooks)
        │         → application_ui_details (screens)
        │         → document_json (raw LLM output)
        │
        ├─── 6. SharePoint Upload
        │         Upload .cbl source + _document.json per program
        │         Store link in application_cobol_sharepoint_links
        │
        └─── 7. App Context + Design Doc
                  generate_app_context() → mst_applications updated
                  Design_Document_COBOL() → .docx generated
```

---

## 3. Entry Point & API Trigger

### API Endpoint
```
POST /bofa-sdlcv3/post/newservice
```
**File:** [app.py](app.py)

Accepts a ZIP file upload. The service type is auto-detected in [modules/service.py](service.py):
```python
if any(f.endswith(('.cpy', '.cbl', '.dat')) for f in all_filenames):
    return 'cobol'
```

### Main Processing Function
**File:** [modules/process_cobol_main.py](modules/process_cobol_main.py)

```python
async def process_cobol_main(path_to_project, application_id, service_id)
```
- `path_to_project` — local path where the uploaded ZIP was extracted
- `application_id` — FK to `mst_applications`
- `service_id` — FK to `application_service_details`

Runs as a **background task** via FastAPI `BackgroundTasks`.

---

## 4. COBOL Parsing Layer

**File:** [modules/process_cobol_main.py](modules/process_cobol_main.py)

### 4.1 File Discovery

```python
read_all_cobol_files_recursively(path_to_project)
```
Recursively finds all `.cbl` files under the project directory. Returns a list of file paths.

### 4.2 Main Parser

```python
parse_cobol_program(filepath, cloc_data=None)
```
Detects format (Fixed vs Free), then extracts:

| What it extracts | COBOL source |
|---|---|
| Program ID, Author, Date-Written | IDENTIFICATION DIVISION |
| File definitions, copy statements | ENVIRONMENT DIVISION |
| Record structures (FD/01 levels) | DATA DIVISION |
| Sections, PERFORM/CALL/IO/SQL operations | PROCEDURE DIVISION |

Returns a structured Python dict that feeds into the LLM prompt.

### 4.3 Supporting Parsing Functions

| Function | Purpose |
|---|---|
| `_clean_cobol_line(line, is_free_format)` | Strips sequence numbers (cols 1-6), comment indicator (col 7), inline `*>` comments. Handles both Fixed Format (cols 1-72) and Free Format. |
| `_parse_record_structure(lines, start_idx)` | Parses FD/01-level record definitions, extracts field names, PIC clauses, nesting levels. |
| `pic_to_length(pic_clause)` | Converts COBOL PIC clauses to numeric byte lengths. Supports `X(n)`, `9(n)`, `S9(n)V9(m)`, etc. |
| `_extract_areas(line)` | Splits a fixed-format line into Area A, Area B, and indicator column. |
| `_diagnose_parsing(lines)` | Diagnostic report: detects FD/01 outside Area A, COPY in Area B — useful for debugging parsing failures. |

### 4.4 Format Detection Logic
```python
# Fixed Format: sequence numbers in columns 1-6, indicator in col 7
# Free Format: detected by IDENTIFICATION DIVISION starting at col 1 without sequence numbers
```

---

## 5. LLM Extraction — Azure OpenAI GPT-4.1-mini

**File:** [modules/process_cobol_main.py](modules/process_cobol_main.py)

### 5.1 LLM Configuration

```python
client = AzureOpenAI(
    api_key=os.getenv("api_key_4_1"),
    api_version=os.getenv("api_version_4_1"),   # e.g. "2024-02-15-preview"
    azure_endpoint=os.getenv("azure_endpoint_4_1")
)
model = "synapt-dev-gpt-4.1-mini"
temperature = 0.1
response_format = {"type": "json_object"}
```

### 5.2 Call #1 — Document JSON Generation

**Function:** `call_openai(cobol_source, parsed_metadata)`

**System Prompt:** `SYSTEM_PROMPT` (200+ lines, defines the exact JSON schema)

**Input to LLM:**
- Raw COBOL source code
- Pre-parsed metadata JSON from `parse_cobol_program()`

**Output — `document.json` structure:**
```json
{
  "addOrUpdatePrograms": [{
    "programId": "CS30003C",
    "fileName": "CS30003C.cbl",
    "purpose": "3-6 sentence description",
    "copybooksUsed": ["COPYBOOK1", "COPYBOOK2"],
    "linkageSummary": {
      "copybook": "...",
      "parameters": [...],
      "fields": [...]
    },
    "orchestration": {
      "entrySection": "MAIN-LOGIC",
      "sequence": [...],
      "description": "..."
    },
    "sections": [{
      "name": "INITIALIZE-SECTION",
      "role": "Short description of what this section does",
      "algorithm": ["Step 1: ...", "Step 2: ...", "...up to 12 steps"]
    }],
    "screenFrames": [{
      "name": "MAIN-SCREEN",
      "purpose": "...",
      "layout": "..."
    }],
    "workingStorageSummary": {
      "constants": [...],
      "workFields": [...]
    }
  }]
}
```

### 5.3 Call #2 — Enrichment Data

**Function:** `generate_enrichment_data(document_json, cobol_source)` (single file)
**Function:** `generate_enrichment_data_batched(document_json, cobol_source)` (large files)

**System Prompt:** `ENRICHMENT_SYSTEM_PROMPT`

**Output — enrichment JSON:**
```json
{
  "api_details": [{
    "section_name": "INITIALIZE-SECTION",
    "long_description": "...",
    "code_quality": 8,
    "code_complexity": "MEDIUM",
    "lines_of_code": 45,
    "business_significance": "...",
    "possible_improvements": "...",
    "tables_linked": ["TABLE1", "TABLE2"]
  }],
  "camunda_details": {
    "workflow_complexity": "MEDIUM",
    "user_task_list": [...],
    "service_task_list": [...],
    "workflow_quality": 7
  },
  "sql_table_details": [{
    "table_name": "COPYBOOK1",
    "description": "...",
    "business_significance": "..."
  }]
}
```

### 5.4 Large File Handling (>3200 lines)

**Function:** `process_large_cobol_file(filepath, ...)`

- Splits file into batches of **3000 lines each**
- Each batch sent to LLM independently via `generate_document_json_batched()`
- Batch results merged into a single `document.json`
- Enrichment generated on the full merged document: `generate_enrichment_data_batched()`

### 5.5 Retry Logic

**Function:** `call_openai_with_retry(prompt, system_prompt, ...)`
- Max **3 retries** on HTTP 429 (rate limit)
- **60-second wait** between retries
- Falls back to returning `None` after exhausting retries

---

## 6. Database Storage (Table Mapping)

**DB Connection:**
```python
import mysql.connector
conn = mysql.connector.connect(
    host=os.getenv("db_host"),
    port=3306,
    user=os.getenv("db_user"),
    password=os.getenv("db_pass"),
    database=os.getenv("db_name")
)
```
Direct `mysql.connector` is used in `process_cobol_main.py` (not SQLAlchemy ORM).

**Main insertion function:** `build_and_execute_inserts(document_json, enrichment_data, application_id, service_id, conn)`

### 6.1 Table: `application_api_details`
**ORM model:** [tables/appapi.py](tables/appapi.py)

**What goes in:** One row **per COBOL section**.

| Column | Source | Notes |
|---|---|---|
| `endpoint` | `{programId}.{section_name}` | Acts as the unique section identifier |
| `http_method` | `"SECTION"` | Static value for COBOL |
| `api_title` | `section_name` | |
| `short_description` | `section.role` | From document.json |
| `long_description` | `enrichment.long_description` | From enrichment call |
| `algorithm` | `section.algorithm` joined with `\n` | Array → newline-separated text |
| `code_quality` | `enrichment.code_quality` | 1–10 integer |
| `code_complexity` | `enrichment.code_complexity` | LOW / MEDIUM / HIGH |
| `lines_of_code` | `enrichment.lines_of_code` | |
| `business_significance` | `enrichment.business_significance` | |
| `possible_improvements` | `enrichment.possible_improvements` | |
| `tables_linked` | `enrichment.tables_linked` | Array → comma-separated string or `"none"` |
| `language` | `"COBOL"` | |
| `service_id` | FK to `application_service_details` | |
| `file_path` | COBOL source file path | |

### 6.2 Table: `application_camunda_details`
**ORM model:** [tables/camunda_details.py](tables/camunda_details.py)

**What goes in:** One row **per COBOL program** (representing the program's orchestration).

| Column | Source | Notes |
|---|---|---|
| `workflow_name` | `{programId}_Orchestration` | |
| `short_description` | Program purpose | From document.json |
| `long_description` | Aggregated section descriptions | |
| `workflow_algorithm` | All section algorithms concatenated | |
| `workflow_complexity` | Derived from section count | 1–10 scale |
| `user_task_list` | `enrichment.camunda_details.user_task_list` | JSON |
| `service_task_list` | `enrichment.camunda_details.service_task_list` | JSON |
| `related_scripts` | Copybooks + linkage info | JSON — used later by Java conversion agents |
| `application_id` | FK | |

### 6.3 Table: `application_sql_table_details`
**ORM model:** [tables/apptable.py](tables/apptable.py)

**What goes in:** One row **per copybook** (de-duplicated).

| Column | Source |
|---|---|
| `table_name` | Copybook name |
| `description` | `enrichment.sql_table_details[].description` |
| `business_significance` | From enrichment |
| `application_id` | FK |

A **duplicate check** is run before insert:
```python
SELECT id FROM application_sql_table_details
WHERE table_name = %s AND application_id = %s
```

### 6.4 Table: `application_ui_details`
**ORM model:** [tables/appui.py](tables/appui.py)

**What goes in:** One row **per CICS screen frame** found in `document.json.screenFrames`.

| Column | Source |
|---|---|
| `screen_name` | `screenFrame.name` |
| `short_description` | `screenFrame.purpose` |
| `api_call` | JSON of API calls from screen |
| `form_field` | JSON of input fields |
| `form_button` | JSON of screen buttons |
| `service_id` | FK |

### 6.5 Table: `document_json`
Stores the raw LLM-generated `document.json` as a JSON blob per program.
```sql
INSERT INTO document_json (app_id, document) VALUES (%s, %s)
```
Later queried by the Java conversion pipeline:
```sql
SELECT document FROM document_json WHERE app_id = %s
```

### 6.6 Table: `application_cobol_sharepoint_links`
Stores the SharePoint folder URL per application after upload completes.

### 6.7 Table: `mst_applications`
**ORM model:** [tables/master.py](tables/master.py)

Updated at the end of processing:
- `application_context` — generated by `generate_app_context()`
- `short_description` — LLM-generated brief summary
- `application_functionality` — aggregated service functionality
- `cost` — total accumulated LLM cost
- `context_percentage` — processing progress (0–100)
- `design_document_link` — SharePoint link to generated .docx

### 6.8 Table: `application_service_details`
**ORM model:** [tables/appservicedet.py](tables/appservicedet.py)

Updated with:
- `service_type = 'cobol'`
- `service_context` — service-level context summary
- `service_functionality` — service functionality description
- `cost` — per-service LLM cost
- `status` — processing status

---

## 7. Design Document Generation

**File:** [DesignDocument_COBOL.py](DesignDocument_COBOL.py)

**Main function:** `Design_Document_COBOL(application_id, db)`

### 7.1 Template
**Template file:** [files/Template_for_Design_Document_Cobol.docx](files/Template_for_Design_Document_Cobol.docx)

The template contains **Word bookmarks**. Each section of the document replaces a named bookmark with LLM-generated content.

### 7.2 Document Sections Generated

| Section Function | Content |
|---|---|
| `cobol_intro_generation()` | Application introduction, purpose, scope |
| `cobol_technical_architecture()` | Tech stack, program structure, integration points |
| `cobol_application_flow()` | Program orchestration flow, section call sequence |
| `cobol_ui_screens()` | CICS/terminal screen descriptions from `application_ui_details` |
| `cobol_section_specifications()` | Detailed specs for each section from `application_api_details` |
| `cobol_data_design()` | Copybook/data structure descriptions from `application_sql_table_details` |
| `cobol_file_structure()` | FD definitions and file organization |

### 7.3 Progress Tracking
Updates `mst_applications` during generation:
```python
# Per section completed:
master.document_percentage = (sections_done / total_sections) * 100
master.document_status = "Generating section X..."
```

---

## 8. App Context Aggregation

**File:** [modules/app_helpers.py](modules/app_helpers.py)

**Function:** `generate_app_context(application_id, application_name, service_details_list)`

### 8.1 What it does
Aggregates all service-level contexts into a single application-level summary.

Uses `AzureChatOpenAI` (LangChain wrapper) with the same Azure OpenAI credentials:
```python
from langchain.chat_models import AzureChatOpenAI
llm = AzureChatOpenAI(
    openai_api_key=os.getenv("api_key_4_1"),
    azure_endpoint=os.getenv("azure_endpoint_4_1"),
    openai_api_version=os.getenv("api_version_4_1"),
    deployment_name="synapt-dev-gpt-4.1-mini"
)
```

### 8.2 LLM Calls Made
1. **Application context** — generates a comprehensive application overview from all service contexts
2. **Short description** — generates a max-4-line summary of the application
3. **Application functionality** — aggregates all service functionality descriptions

### 8.3 Output stored in `mst_applications`
```python
master.application_context = application_context
master.short_description = short_description
master.application_functionality = application_functionality
master.cost += total_llm_cost
```

---

## 9. SharePoint Integration

**File:** [modules/sharepoint_share.py](modules/sharepoint_share.py)

### 9.1 Authentication
```python
# Azure AD OAuth2 via MSAL
get_access_token(client_id, client_secret, tenant_id, scopes)
```

### 9.2 Upload Flow (per COBOL program)

```
get_access_token()
    → get_site_id(hostname, site_path)
        → create_folder_sharepoint(site_id, drive_id, folder_path)
            → upload_file_to_sharepoint_folder(site_id, drive_id, folder_id, file)
                → get_file_link(site_id, drive_id, item_id)
                    → save_sharepoint_link_to_db(application_id, link, conn)
```

### 9.3 Folder Structure
```
{root_folder_name}/
  └── app_{application_id}/
      └── {program_name}/
          ├── {program_name}.cbl           ← COBOL source
          └── {program_name}_document.json ← LLM-generated analysis
```

### 9.4 SharePoint Config
```python
# All from environment variables:
client_id      = os.getenv("client_id")
client_secret  = os.getenv("client_secrete")   # note: typo in env var name
tenant_id      = os.getenv("tenant_id")
hostname       = os.getenv("site_hostname")
site_path      = os.getenv("site_path")
drive_id       = os.getenv("drive_id")
root_folder    = os.getenv("folder_name_main")
```

---

## 10. Cost Tracking

**Functions in** [modules/app_helpers.py](modules/app_helpers.py):
```python
calculate_token_count(text, model)   # uses tiktoken
calculate_cost(input_tokens, output_tokens)
calculate_token_count_4o(text)       # for GPT-4o mini
calculate_cost_4o(input_tokens, output_tokens)
```

Cost rates from environment:
```
input_cost_per_1k       → cost per 1K input tokens
output_cost_per_1k      → cost per 1K output tokens
4o_mini_input_token_1k  → GPT-4o mini input rate
4o_mini_output_token_1k → GPT-4o mini output rate
```

Cost stored at three levels:
| Table | Column | Scope |
|---|---|---|
| `application_api_details` | `cost` | Per section |
| `application_service_details` | `cost` | Per service |
| `mst_applications` | `cost` | Application total |

---

## 11. Key Dependencies & Frameworks

| Library | Version | Used For | Where Used |
|---|---|---|---|
| `openai` (AzureOpenAI) | latest | LLM calls — document.json + enrichment data | `process_cobol_main.py` |
| `langchain` (AzureChatOpenAI) | 0.1.20 | App context aggregation | `app_helpers.py` |
| `mysql-connector-python` | 9.0.0 | Direct DB inserts/queries in processing pipeline | `process_cobol_main.py` |
| `sqlalchemy` | 2.0.31 | ORM for master/service/api table reads | `connect.py`, `app.py` |
| `tiktoken` | latest | Token counting for LLM cost calculation | `app_helpers.py` |
| `python-dotenv` | 1.0.1 | Loading `.env` environment variables | All modules |
| `msal` | 1.29.0 | Azure AD OAuth2 for SharePoint Graph API | `sharepoint_share.py` |
| `requests` | 2.32.3 | HTTP calls to Microsoft Graph API | `sharepoint_share.py` |
| `python-docx` (`docx`) | latest | Reading/writing Word `.docx` design doc template | `DesignDocument_COBOL.py` |
| `fitz` (PyMuPDF) | latest | PDF template/page reading | `document_processor.py` |
| `pdfplumber` | latest | PDF text extraction | `document_processor.py` |
| `Pillow` (PIL) | latest | Image handling in document generation | `document_processor.py` |
| `rich` | latest | Rich terminal formatting/logging | `process_cobol_main.py` |
| `FastAPI` | 0.111.0 | REST API framework | `app.py` |

---

## 12. Environment Variables

```env
# Azure OpenAI (GPT-4.1-mini)
api_key_4_1=<Azure OpenAI API key>
api_version_4_1=2024-02-15-preview
azure_endpoint_4_1=https://<resource>.openai.azure.com/

# Database (MySQL)
db_host=<hostname>
db_user=<username>
db_pass=<password>
db_name=<database name>

# LLM Cost Rates
input_cost_per_1k=<float>
output_cost_per_1k=<float>
4o_mini_input_token_1k=<float>
4o_mini_output_token_1k=<float>

# SharePoint / Azure AD
client_id=<Azure AD app client ID>
client_secrete=<Azure AD client secret>   # note: env var has typo
tenant_id=<Azure tenant ID>
scopes=<comma-separated OAuth scopes>
site_hostname=<SharePoint hostname>
site_path=<SharePoint site path>
drive_id=<SharePoint drive ID>
folder_name_main=<root folder name in SharePoint>
```

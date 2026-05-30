# COBOL to Java Conversion Pipeline — Developer KT Document

## Table of Contents
1. [Overview](#1-overview)
2. [API Endpoints](#2-api-endpoints)
3. [Entry Point & Background Task](#3-entry-point--background-task)
4. [Pre-Conversion: Context Retrieval](#4-pre-conversion-context-retrieval)
5. [4-Agent Architecture](#5-4-agent-architecture)
6. [Agent Execution Order & Data Flow](#6-agent-execution-order--data-flow)
7. [Template Structure (Spring Boot)](#7-template-structure-spring-boot)
8. [Validation](#8-validation)
9. [Output & Finalization](#9-output--finalization)
10. [Database Tables Used](#10-database-tables-used)
11. [LLM Configuration](#11-llm-configuration)
12. [COBOL PIC Clause → Java Type Mapping](#12-cobol-pic-clause--java-type-mapping)
13. [Key Dependencies & Frameworks](#13-key-dependencies--frameworks)
14. [Environment Variables](#14-environment-variables)

---

## 1. Overview

This pipeline takes the **already-stored COBOL app context** (from `COBOLAPPCONTEXT.md` pipeline) and converts it into a production-ready **Spring Boot Java microservice** using a 4-agent AI orchestration system.

**Prerequisite:** The COBOL ingestion pipeline must have already run and stored context in:
- `application_camunda_details` — workflow/orchestration data per COBOL program
- `application_cobol_sharepoint_links` — link to the SharePoint folder containing `document.json`, `analysis.json`, and `.cbl` source files

**What it produces:**
- One Spring Boot microservice ZIP per COBOL workflow (program)
- Entities, DTOs, Repositories, Service interfaces + implementations, REST Controllers
- A `pom.xml` configured for Spring Boot 3.2.0 / Java 17
- A `conversion_context.json` with intermediate agent outputs per workflow
- SharePoint link to the final Java repo ZIP stored in `application_cobol_status.java_repo`

---

## 2. API Endpoints

**File:** [app.py](app.py)

All endpoints require JWT authorization (`user_email: str = Depends(authorize)`).

### 2.1 List Eligible COBOL Applications
```
GET /bofa-sdlcv3/get/cobol_applications
```
Returns applications that have completed COBOL processing.

**DB query:**
```sql
SELECT a.id AS application_id, a.application_name
FROM mst_applications a
JOIN application_service_details s ON a.id = s.application_id
WHERE s.service_type = 'cobol'
```

### 2.2 Get Workflows for an Application
```
GET /bofa-sdlcv3/get/cobol_workflow?application_id=<id>
```
Returns list of workflow names available for conversion.

**DB query:**
```sql
SELECT workflow_name FROM application_camunda_details
WHERE application_id = %s
```

### 2.3 Trigger Conversion
```
POST /bofa-sdlcv3/post/cobol_to_java
Form params: application_id (int), workflow (str)
```
- `workflow`: `"all"` to convert all workflows, or a specific `workflow_name`
- Inserts a record into `application_cobol_status` with `status = 'In Queue'`
- Queues background task: `process_cobol_to_java_conversion(application_id, request_id, workflow)`
- Returns `request_id` for polling

### 2.4 Poll Conversion Status
```
GET /bofa-sdlcv3/get/cobol_status?request_id=<id>
```
**DB query:**
```sql
SELECT status FROM application_cobol_status WHERE id = %s
```
Returns the current human-readable status string (e.g., `"Agent 3: Translating COBOL logic to Java..."`, `"Completed"`).

### 2.5 Download Java Repository
```
GET /bofa-sdlcv3/get/download_java_repo?request_id=<id>
```
**DB query:**
```sql
SELECT java_repo FROM application_cobol_status WHERE id = %s
```
Returns the SharePoint download link for the generated Java ZIP.

---

## 3. Entry Point & Background Task

**File:** [modules/code_conversion_agents/cobol_to_java_agent.py](modules/code_conversion_agents/cobol_to_java_agent.py)

**Main function:**
```python
async def process_cobol_to_java_conversion(application_id: int, request_id: int, workflow: str)
```

**Runs as a FastAPI `BackgroundTask`** — non-blocking, status polled via `GET /cobol_status`.

**Status updates are written to DB throughout:**
```python
def update_cobol_status(request_id, status_message, conn):
    cursor.execute(
        "UPDATE application_cobol_status SET status = %s WHERE id = %s",
        (status_message, request_id)
    )
```

---

## 4. Pre-Conversion: Context Retrieval

Before agents run, the pipeline retrieves COBOL context files from SharePoint and workflow metadata from MySQL.

### 4.1 Fetch SharePoint Link from DB

**Function:** `get_sharepoint_cobol_context_files_link(application_id, conn)`

```sql
SELECT sharepoint_link FROM application_cobol_sharepoint_links
WHERE application_id = %s
```

### 4.2 Download COBOL Context Files from SharePoint

**Function:** `download_cobol_context_from_sharepoint(sharepoint_link, dest_dir)`

Uses Microsoft Graph API via MSAL auth to recursively download the folder:
```
{sharepoint_link}/
  └── {PROGRAM_ID}/
      ├── {PROGRAM_ID}_document.json    ← LLM-generated structure
      ├── {PROGRAM_ID}_analysis.json    ← Parsed metadata
      └── {PROGRAM_ID}.cbl (or .CBL)   ← COBOL source
```

Downloads to a temp directory: `/tmp/temp/cobol_context_{timestamp}/`

### 4.3 Fetch Workflows from DB

**Function:** `CobolToJavaOrchestrator.get_workflows_for_application(application_id)`

```sql
SELECT id, application_id, workflow_name, short_description, long_description,
       workflow_algorithm, workflow_complexity, language,
       user_task_list, service_task_list, workflow_quality,
       workflow_quality_justification, related_scripts
FROM application_camunda_details
WHERE application_id = %s
```

For a specific workflow name:
```sql
WHERE application_id = %s AND workflow_name = %s
```
Note: The `workflow` parameter `"CS30003C"` is normalized → `"CS30003C_Orchestration"` before querying.

---

## 5. 4-Agent Architecture

**Class:** `CobolToJavaOrchestrator` in [modules/code_conversion_agents/cobol_to_java_agent.py](modules/code_conversion_agents/cobol_to_java_agent.py)

All agents use **Azure OpenAI GPT-5.1** directly via `AzureOpenAI` client (not LangChain).

```python
client = AzureOpenAI(
    api_key=os.getenv("AZURE_OPENAI_GPT_51_API_KEY"),
    api_version=os.getenv("AZURE_OPENAI_GPT_51_API_VERSION"),
    azure_endpoint=os.getenv("AZURE_OPENAI_GPT_51_AZURE_ENDPOINT")
)
model = os.getenv("AZURE_OPENAI_GPT_51_MODEL")  # "synapt-dev-gpt-5.1"
```

---

### Agent 1: DataModeler

**Class:** `Agent1_DataModeler`
**Temperature:** `0.1`
**Triggered at:** Stage 2 (after Architect)

**Input:**
- `document_json` — the `document.json` for the COBOL program
- `related_scripts` — copybook names from `application_camunda_details.related_scripts`

**Task:** Analyze `workingStorageSummary` and `linkageSummary` from `document.json` → create JPA Entities and DTOs.

**Output JSON:**
```json
{
  "entities": [{
    "name": "InvoiceHeaderEntity",
    "tableName": "invoice_header",
    "fields": [
      { "name": "documentNumber", "type": "Long", "annotations": ["@Id"] },
      { "name": "totalValue", "type": "BigDecimal", "precision": 12, "scale": 2 }
    ]
  }],
  "dtos": [{
    "name": "WorkflowContextDTO",
    "fields": [...]
  }],
  "copybooks": ["COPYBOOK1", "COPYBOOK2"]
}
```

---

### Agent 2: Architect

**Class:** `Agent2_Architect`
**Temperature:** `0.1`
**Triggered at:** Stage 1 (first agent to run)

**Input:**
- `sql_data` — full workflow row from `application_camunda_details`
- `document_json` — the program's structural analysis

**Task:** Define Spring Boot architecture: Controller endpoints + Service interfaces, mapping COBOL sections to Java service methods.

**Output JSON:**
```json
{
  "controller": {
    "name": "InvoiceOrchestrationController",
    "baseUrl": "/api/v1/invoices",
    "endpoints": [{
      "method": "POST",
      "path": "/initialize",
      "serviceMethod": "initializeOrchestration",
      "requestBody": "WorkflowContextDTO",
      "responseBody": "void"
    }]
  },
  "service": {
    "interfaceName": "InvoiceOrchestrationService",
    "implName": "InvoiceOrchestrationServiceImpl",
    "methods": [...]
  },
  "workflow": {
    "name": "InvoiceOrchestration",
    "basePackage": "com.legacy.nfe"
  }
}
```

---

### Agent 3: LogicImplementer

**Class:** `Agent3_LogicImplementer`
**Temperature:** `0.1`
**Triggered at:** Stage 3

**Input:**
- `data_model` — output from Agent1 (entities + DTOs)
- `architecture` — output from Agent2 (controller/service design)
- `cobol_source` — raw `.cbl` source code

**Task:** Translate COBOL business logic into complete, executable Java method bodies.

**Critical prompt constraint (enforced strictly):**
> - `javaCode` field MUST contain complete, compilable Java
> - NO empty braces `{}`
> - NO `// TODO` comments
> - NO stub methods — every method body must be fully implemented
> - Include all necessary imports

**Output JSON:**
```json
{
  "serviceImplementations": [{
    "className": "InvoiceOrchestrationServiceImpl",
    "javaCode": "package com.legacy.nfe.Service;\n\nimport ...;\n\n@Service\n@RequiredArgsConstructor\npublic class InvoiceOrchestrationServiceImpl implements InvoiceOrchestrationService {\n    private final InvoiceHeaderRepository invoiceHeaderRepository;\n\n    @Override\n    public void initializeOrchestration(WorkflowContextDTO context) {\n        // full implementation here\n    }\n}"
  }]
}
```

---

### Agent 4: QAIntegration

**Class:** `Agent4_QAIntegration`
**Temperature:** `0.05` (lowest — production code generation)
**Triggered at:** Stage 4 (final agent)

**Input:**
- All prior agent outputs (data_model, architecture, service_implementations)
- Template folder structure (scanned from `templates/java-from-cobol/`)

**Task:** Assemble the complete Spring Boot project. Generates all files:
- `pom.xml`
- All Entity classes with JPA annotations
- All DTO classes
- All Repository interfaces extending `JpaRepository`
- Service interface + implementation
- REST Controller
- `application.properties`
- `*Application.java` (main class)

**Template scanning (before Agent4 runs):**
```python
# Extracts folder structure from template directory
_scan_template_structure(template_dir)
# Returns: {
#   "base_package": "com.legacy.nfe",
#   "folders": ["Controller", "Service", "Entity", "Repository", "DTO"],
#   "files": [{"relative_path": "...", "folder_type": "Controller"}, ...]
# }
```

**File writing logic:**
```python
# Agent4 output includes file paths + content
for file in agent4_output["files"]:
    dest_path = output_dir / file["relative_path"]
    dest_path.parent.mkdir(parents=True, exist_ok=True)
    dest_path.write_text(file["content"], encoding="utf-8")
```

---

## 6. Agent Execution Order & Data Flow

```
Per workflow (e.g., CS30003C_Orchestration):
│
├─── Extract program ID from workflow name
│    "CS30003C_Orchestration" → "CS30003C"
│
├─── Load program files from downloaded context:
│    - {program_id}_document.json
│    - {program_id}_analysis.json
│    - {program_id}.cbl
│
├─── STAGE 1: Agent2 (Architect)
│    Input:  workflow row from DB + document.json
│    Output: controller + service interface design
│    Status update: "Agent 2: Designing Spring Boot architecture..."
│
├─── STAGE 2: Agent1 (DataModeler)
│    Input:  document.json + related_scripts (copybooks)
│    Output: JPA entities + DTOs
│    Status update: "Agent 1: Creating data models..."
│
├─── STAGE 3: Agent3 (LogicImplementer)
│    Input:  data_model + architecture + cobol_source
│    Output: complete Java service implementation
│    Status update: "Agent 3: Translating COBOL logic to Java..."
│
├─── STAGE 4: Agent4 (QAIntegration)
│    Input:  all prior outputs + template structure
│    Output: complete Spring Boot project files written to disk
│    Status update: "Agent 4: Assembling and validating microservice..."
│
├─── Validate output
│    _validate_java_code_completeness()
│    _validate_microservice()
│
├─── Create ZIP archive
│    {workflow_name}_microservice_{timestamp}.zip
│
└─── Save conversion_context.json
     (contains all intermediate outputs for debugging/audit)
```

---

## 7. Template Structure (Spring Boot)

**Location:** [modules/code_conversion_agents/templates/java-from-cobol/](modules/code_conversion_agents/templates/java-from-cobol/)

This template represents a reference Spring Boot project. Agent4 uses its folder/package structure as a scaffold.

```
java-from-cobol/
├── pom.xml                                      ← Spring Boot 3.2.0, Java 17
├── InvoiceOrchestrationApplication.java         ← Main app class (root)
└── src/main/java/com/legacy/nfe/
    ├── Controller/
    │   └── InvoiceOrchestrationController.java
    ├── Service/
    │   ├── InvoiceOrchestrationService.java      ← interface
    │   └── InvoiceOrchestrationServiceImpl.java  ← implementation
    ├── Entity/
    │   ├── InvoiceHeaderEntity.java
    │   ├── ClientEntity.java
    │   └── ...
    ├── Repository/
    │   ├── InvoiceHeaderRepository.java
    │   └── ...
    └── DTO/
        └── WorkflowContextDTO.java
```

### pom.xml Dependencies (from template)
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>

<!-- Key dependencies: -->
spring-boot-starter-web
spring-boot-starter-data-jpa
spring-boot-starter-actuator
springdoc-openapi-starter-webmvc-ui   <!-- Swagger UI -->
com.h2database:h2 (runtime)            <!-- In-memory DB for dev -->
org.projectlombok:lombok (optional)
```

**How template is used by Agent4:**
1. Template is copied to output directory: `output/{workflow_name}/`
2. All `.java` files in the copy are **cleared** (emptied)
3. Agent4 writes workflow-specific `.java` content into the cleared files
4. New files not in template are created as needed (Agent4 can add files)

---

## 8. Validation

Both functions are called after Agent4 completes, before ZIP creation.

### 8.1 Code Completeness Validation

**Function:** `_validate_java_code_completeness(output_dir)`

Checks:
- `ServiceImpl` classes have real method bodies (not empty `{}`)
- Controllers have `@RequestMapping` methods with bodies
- No `// TODO` comments remain
- Repositories have proper method signatures

Returns a validation report with any issues found.

### 8.2 Microservice Architecture Validation

**Function:** `_validate_microservice(output_dir)`

Verifies presence of:
- `@RestController` annotation
- `@Service` annotation
- `@Repository` annotation
- `@Entity` annotation
- `@SpringBootApplication` (main class)
- `pom.xml`
- `application.properties`

Returns `{"valid": bool, "principles_found": [...], "missing": [...]}`.

---

## 9. Output & Finalization

**Function:** `finalize_java_repo(output_dir, application_id, request_id, conn)`

### 9.1 Per-workflow output
```
output/{workflow_name}/
├── {workflow_name}_microservice_{YYYYMMDD_HHMMSS}.zip   ← compiled microservice
└── conversion_context.json                              ← agent intermediate outputs
```

`conversion_context.json` contains:
```json
{
  "workflow_name": "CS30003C_Orchestration",
  "program_id": "CS30003C",
  "agent1_data_model": {...},
  "agent2_architecture": {...},
  "agent3_implementations": {...},
  "agent4_file_summary": {...},
  "validation_results": {...},
  "timestamp": "2024-..."
}
```

### 9.2 Final ZIP + SharePoint Upload
After all workflows are processed:
1. Final ZIP created wrapping all workflow outputs
2. Uploaded to SharePoint via Graph API
3. SharePoint download link stored in DB:

```sql
UPDATE application_cobol_status
SET status = 'Completed', java_repo = %s
WHERE id = %s
```

### 9.3 Cleanup
Temp directories removed:
```python
shutil.rmtree("/tmp/temp/cobol_context_{timestamp}/")
```

---

## 10. Database Tables Used

| Table | Operation | Purpose |
|---|---|---|
| `application_cobol_status` | INSERT (on trigger) | Track conversion request; holds request_id |
| `application_cobol_status` | UPDATE throughout | Status messages at each agent stage |
| `application_cobol_status` | UPDATE (on complete) | Store `java_repo` SharePoint link |
| `application_camunda_details` | SELECT | Source of workflow data (algorithm, related_scripts, etc.) |
| `application_cobol_sharepoint_links` | SELECT | Locate the COBOL context files folder on SharePoint |
| `mst_applications` | SELECT (via API) | Filter COBOL-type applications for listing |
| `application_service_details` | SELECT (via API) | Join for cobol app listing (`service_type = 'cobol'`) |

**`application_cobol_status` schema:**
```sql
id           INT PRIMARY KEY AUTO_INCREMENT
application_id INT
status       VARCHAR(500)    -- human-readable status string
java_repo    TEXT            -- SharePoint URL of generated ZIP (set on completion)
updated_at   DATETIME
```

---

## 11. LLM Configuration

```python
# cobol_to_java_agent.py
client = AzureOpenAI(
    api_key=os.getenv("AZURE_OPENAI_GPT_51_API_KEY"),
    api_version=os.getenv("AZURE_OPENAI_GPT_51_API_VERSION"),
    azure_endpoint=os.getenv("AZURE_OPENAI_GPT_51_AZURE_ENDPOINT")
)
model = os.getenv("AZURE_OPENAI_GPT_51_MODEL")   # "synapt-dev-gpt-5.1"
```

| Agent | Temperature | Rationale |
|---|---|---|
| Agent1 (DataModeler) | 0.1 | Deterministic type mapping |
| Agent2 (Architect) | 0.1 | Consistent architecture design |
| Agent3 (LogicImplementer) | 0.1 | Reliable logic translation |
| Agent4 (QAIntegration) | 0.05 | Most deterministic — final production code |

All agents use `response_format={"type": "json_object"}` for structured output.

---

## 12. COBOL PIC Clause → Java Type Mapping

Used by Agent1 (DataModeler) as a reference in its system prompt:

| COBOL PIC Clause | Java Type | Notes |
|---|---|---|
| `PIC 9(n)` | `Integer` or `Long` | Long if n > 9 |
| `PIC S9(n)` | `Integer` or `Long` | Signed integer |
| `PIC X(n)` | `String` | Fixed-length char |
| `PIC S9(n)V99` | `BigDecimal` | Decimal with 2 decimal places |
| `PIC 9(n)V9(m)` | `BigDecimal` | With precision/scale annotation |
| `PIC S9(n)V9(m)` | `BigDecimal` | Signed decimal |
| `PIC A(n)` | `String` | Alphabetic only |

JPA annotation for `BigDecimal`:
```java
@Column(precision = 12, scale = 2)
private BigDecimal totalValue;
```

---

## 13. Key Dependencies & Frameworks

| Library | Version | Used For | Where Used |
|---|---|---|---|
| `openai` (AzureOpenAI) | latest | All 4 agent LLM calls | `cobol_to_java_agent.py` |
| `mysql-connector-python` | 9.0.0 | Direct DB queries — workflow fetch, status updates | `cobol_to_java_agent.py` |
| `sqlalchemy` | 2.0.31 | Minimal use — `text()` for raw SQL in ORM session | `cobol_to_java_agent.py` |
| `msal` | 1.29.0 | Azure AD OAuth2 for SharePoint context download | `sharepoint_share.py` |
| `requests` | 2.32.3 | Microsoft Graph API — download COBOL files from SharePoint | `sharepoint_share.py` |
| `FastAPI` + `BackgroundTasks` | 0.111.0 | Async HTTP trigger + background processing | `app.py` |
| `zipfile` | stdlib | Creating ZIP archives of generated microservices | `cobol_to_java_agent.py` |
| `pathlib` | stdlib | File and directory path management | `cobol_to_java_agent.py` |
| `shutil` | stdlib | Copying template directories, cleanup | `cobol_to_java_agent.py` |
| `python-dotenv` | 1.0.1 | Loading `.env` environment variables | All modules |
| `json` | stdlib | Parsing LLM JSON responses, reading/writing context files | `cobol_to_java_agent.py` |

**Note:** LangChain is listed in `requirements.txt` but is **NOT used** in the Java conversion pipeline. All LLM calls use the `openai` SDK directly.

---

## 14. Environment Variables

```env
# Azure OpenAI (GPT-5.1) — for all 4 agents
AZURE_OPENAI_GPT_51_API_KEY=<Azure OpenAI API key>
AZURE_OPENAI_GPT_51_API_VERSION=2024-02-15-preview
AZURE_OPENAI_GPT_51_AZURE_ENDPOINT=https://<resource>.openai.azure.com/
AZURE_OPENAI_GPT_51_MODEL=synapt-dev-gpt-5.1

# Database (MySQL)
db_host=<hostname>
db_user=<username>
db_pass=<password>
db_name=<database name>

# SharePoint / Azure AD (for COBOL context download)
client_id=<Azure AD app client ID>
client_secrete=<Azure AD client secret>   # note: env var has typo
tenant_id=<Azure tenant ID>
scopes=<comma-separated OAuth scopes>
site_hostname=<SharePoint hostname>
site_path=<SharePoint site path>
drive_id=<SharePoint drive ID>
```

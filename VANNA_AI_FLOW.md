# RapidOne AI Chatbox — Vanna AI Flow (Complete)

> This document explains how the AI query system works end-to-end.
> Starts where AUTH_FLOW.md ends — FastAPI has received the enriched request from .NET with all user context.
> Covers: database architecture, shared Vanna training with XXX_ placeholder, entity detection, permission enforcement, SQL generation, prefix replacement, validation, execution, and response.

---

## Part 1: Where This Flow Begins

AUTH_FLOW.md covers Steps 1-8: login, token, .NET validation, getUserInfo(), X-API-KEY, and how the enriched request reaches FastAPI.

**This document starts at AUTH_FLOW.md Step 8 — FastAPI has the request.**

FastAPI now has:

```
From .NET enriched request body:
    query           = "show me today's appointments"
    user_id         = "user-guid-123"
    user_name       = "Dr. Cohen"
    roles           = ["DoctorWithCalendar"]                          (Layer 1)
    permissions     = ["Schedule - Open the schedule", "..."]         (Layer 2 + 3 combined)
    department_id   = 5
    branch_id       = 2
    default_issuer  = "Issuer1"
    database        = "Keshevrav_RapidOne"                            (main DB name)
    language        = "he-IL"

From FastAPI querying the customer database:
    Layer 4: linked_schedules = ["doctor-guid-1", "doctor-guid-2"]    (from LinkUsersToSchedules)
    Layer 5: allowed_issuers  = ["Issuer1", "Issuer2"]               (from LinkIssuersToUsers)
```

Now the AI processing begins.

---

## Part 2: The Three Database Types Per Customer

Every customer (prefix = `XXX`) has these databases on the **same SQL Server instance**:

| Database | Name Pattern | Contains | Connection |
|---|---|---|---|
| Main App DB | `XXX_RapidOne` | Leads, Appointments, Schedules, Branches, Departments, Users, Permissions | Always connected at startup |
| SAP DB | `XXX_Rapid_Common` | Patients/Customers (OCRD table), Treatments, Treatment Plans | Always connected at startup |
| Issuer DB(s) | `XXX_Issuer1`, `XXX_Issuer2`, ... `XXX_IssuerN` | Invoices, Receipts (financial SAP databases) | Lazy — connected only when a permitted user queries financial data |

Important notes:
- **ALL databases are prefixed with the customer name** — including issuers
- Example for Keshevrav customer: `Keshevrav_RapidOne`, `Keshevrav_Rapid_Common`, `Keshevrav_Issuer1`, `Keshevrav_Issuer2`
- `Issuers.Id` in the database stores the issuer identifier (e.g., `Issuer1`). The full DB name is `{PREFIX}_{IssuerId}`
- A customer can have **multiple Issuer databases** — one per financial entity
- All databases exist on the **same SQL Server instance**, so cross-database JOINs work natively

### Database Discovery on Startup

When the FastAPI server starts for a customer:

```
Read config → DB_PREFIX = "Keshevrav", DB_SERVER = "156.67.105.206"
    → Build known names: Keshevrav_RapidOne, Keshevrav_Rapid_Common
    → Connect to both (ALWAYS connected)
    → Query Issuers table: SELECT Id FROM Keshevrav_RapidOne.dbo.Issuers
    → Discover: Issuer1, Issuer2 (issuer identifiers)
    → Full DB names: Keshevrav_Issuer1, Keshevrav_Issuer2
    → Issuer connections are LAZY (created only when needed)
    → Connection Pool Ready
```

---

## Part 3: Cross-Database Queries — How They Work

SQL Server allows JOINs across databases on the same instance using **fully-qualified table names**:

```
Format: DatabaseName.dbo.TableName
```

Example — Appointments with Patient names (data from 2 different databases):

```sql
SELECT
    a.AppointmentId,
    a.Date,
    a.DoctorId,
    c.CardName AS PatientName,
    c.Phone1
FROM Keshevrav_RapidOne.dbo.Appointments a
JOIN Keshevrav_Rapid_Common.dbo.OCRD c
    ON a.PatientId = c.CardCode
WHERE CAST(a.Date AS DATE) = CAST(GETDATE() AS DATE)
```

No special configuration needed. The SQL Server login used by FastAPI just needs read access to all databases on the instance.

### Common Cross-DB JOIN Scenarios

| Query Type | Databases Involved | Example JOIN |
|---|---|---|
| Appointments + Patient details | `XXX_RapidOne` + `XXX_Rapid_Common` | Appointments JOIN OCRD ON PatientId = CardCode |
| Leads + Patient details | `XXX_RapidOne` + `XXX_Rapid_Common` | Leads JOIN OCRD ON PatientID = CardCode |
| Invoices + Branch details | `XXX_IssuerN` + `XXX_RapidOne` | Invoices JOIN Branches ON BranchId |
| Treatments + Patient details | `XXX_Rapid_Common` (same DB) | Treatments JOIN OCRD ON CardCode |

---

## Part 4: Vanna AI Training Strategy

### 4.1 ONE Shared Vanna For ALL Customers On A Server

All customers on the same server share the **exact same database schema** — just with different data and different prefix. So instead of one Vanna per customer, we use **ONE shared Vanna** trained with `XXX_` placeholder prefix.

```
Server A has 10 customers:
    Keshevrav, ClinicB, ClinicC, ClinicD, ClinicE,
    ClinicF, ClinicG, ClinicH, ClinicI, ClinicJ

Instead of:
    ❌ 10 separate Vanna instances (10x memory, 10x training)

We do:
    ✅ 1 shared Vanna trained with XXX_ prefix
    ✅ Each customer's FastAPI process uses the same trained Vanna
    ✅ After SQL generation, replace XXX_ with actual customer prefix
```

### Why This Works

| Concern | Answer |
|---|---|
| Same schema? | YES — all customers have identical table structures |
| Latency? | BETTER — one vector store in RAM instead of 10. LLM API call is the same regardless |
| Retrieval accuracy? | SAME — vector search matches on question semantics ("appointments", "today"), not on the prefix name |
| Hallucination? | LOW — `XXX_` pattern is heavily reinforced across all training data. SQL validation catches edge cases |
| Memory savings? | ~10x less RAM for embeddings (one vector store instead of 10) |
| Training updates? | SIMPLER — update once, all customers benefit |

### 4.2 Training Data — Three Types (All Using XXX_ Prefix)

Vanna needs three types of training data for accurate SQL generation. **ALL training uses `XXX_` as the database prefix.**

**Type 1: DDL (Table Definitions)**

```python
# Main App DB tables
vn.train(ddl="""
CREATE TABLE XXX_RapidOne.dbo.Appointments (
    AppointmentId INT PRIMARY KEY,
    Date DATETIME,
    DoctorId NVARCHAR(128),         -- FK to AspNetUsers.Id (doctor)
    PatientId NVARCHAR(15),         -- FK to OCRD.CardCode (patient)
    DepartmentId INT,
    StatusId INT,
    Notes NVARCHAR(MAX)
)
""")

vn.train(ddl="""
CREATE TABLE XXX_RapidOne.dbo.Leads (
    LeadId INT PRIMARY KEY,
    FirstName NVARCHAR(100),
    LastName NVARCHAR(100),
    Phone NVARCHAR(20),
    AssignedTo NVARCHAR(128),       -- FK to AspNetUsers.Id (assigned agent)
    Branch INT,                     -- FK to Branches.BranchId (NOT DepartmentID)
    StatusId INT,
    IsDeleted BIT                   -- 0=active, 1=deleted
)
""")

# SAP DB tables
vn.train(ddl="""
CREATE TABLE XXX_Rapid_Common.dbo.OCRD (
    CardCode NVARCHAR(15) PRIMARY KEY,   -- Patient ID
    CardName NVARCHAR(100),              -- Patient full name
    Phone1 NVARCHAR(20),
    Phone2 NVARCHAR(20),
    E_Mail NVARCHAR(100),
    Address NVARCHAR(100),
    U_Gender INT,                        -- 1=Male, 2=Female
    CreateDate DATETIME
)
""")

# Issuer DB tables (same structure for all issuers — also uses XXX_ prefix)
vn.train(ddl="""
CREATE TABLE XXX_Issuer1.dbo.Invoices (
    InvoiceId INT PRIMARY KEY,
    PatientId NVARCHAR(15),
    BranchId INT,
    Amount DECIMAL(18,2),
    InvoiceDate DATETIME,
    Status INT
)
""")

vn.train(ddl="""
CREATE TABLE XXX_Issuer1.dbo.Receipts (
    ReceiptId INT PRIMARY KEY,
    PatientId NVARCHAR(15),
    Amount DECIMAL(18,2),
    ReceiptDate DATETIME,
    Status INT
)
""")
```

**Type 2: Metadata (Column Descriptions + Business Context)**

```python
vn.train(documentation="""
DATABASE STRUCTURE:
- XXX_RapidOne = Main application database (leads, appointments, schedules, branches, departments, users)
- XXX_Rapid_Common = SAP database (patients/customers in OCRD table, treatments, treatment plans)
- XXX_Issuer1, XXX_Issuer2, etc. = Financial SAP databases (invoices, receipts)
- All databases use the XXX_ prefix. XXX is a placeholder for the customer name.
- All databases are on the same SQL Server instance, so cross-database JOINs work.

Table: OCRD (in XXX_Rapid_Common)
- This is the Patients/Customers table (SAP Business One naming convention)
- CardCode = Patient ID (primary key, format: patient number as string)
- CardName = Patient full name
- U_Gender: 1 = Male, 2 = Female
- To get patient details for an appointment, JOIN Appointments.PatientId = OCRD.CardCode

Table: Leads (in XXX_RapidOne)
- Branch column = physical location (int FK to Branches.BranchId), NOT DepartmentID
- AssignedTo = user ID of the assigned call center agent (nvarchar(128))
- IsDeleted: 0 = active lead, 1 = deleted/archived
- Always filter WHERE IsDeleted = 0 unless specifically asked for deleted leads

Table: Appointments (in XXX_RapidOne)
- DoctorId = the doctor who owns this appointment (nvarchar(128), FK to AspNetUsers.Id)
- There is NO 'ScheduleId' column in this table
- To filter by schedule access, use: DoctorId IN (list of allowed doctor IDs)
- DepartmentId = the specialty department where this appointment takes place

Table: LinkUsersToSchedules (in XXX_RapidOne)
- UserId = the user who is viewing (the logged-in user)
- ScheduleId = the doctor whose schedule they can see (this IS a DoctorId from AspNetUsers.Id)
- A "schedule" is not a separate entity — it is a doctor's schedule

Table: Invoices (in XXX_Issuer1, XXX_Issuer2, etc.)
- Each issuer database has the same table structure
- InvoiceId = unique invoice identifier
- PatientId = FK to OCRD.CardCode
- BranchId = FK to Branches.BranchId
""")
```

**Type 3: Sample Question-SQL Pairs (Most Important For Accuracy)**

```python
# Appointments with patient details (cross-DB JOIN)
vn.train(
    question="show me today's appointments",
    sql="""
    SELECT
        a.AppointmentId,
        a.Date,
        u.Name AS DoctorName,
        c.CardName AS PatientName,
        c.Phone1,
        d.DepartmentName
    FROM XXX_RapidOne.dbo.Appointments a
    JOIN XXX_Rapid_Common.dbo.OCRD c ON a.PatientId = c.CardCode
    JOIN XXX_RapidOne.dbo.AspNetUsers u ON a.DoctorId = u.Id
    JOIN XXX_RapidOne.dbo.Departments d ON a.DepartmentId = d.DepartmentId
    WHERE CAST(a.Date AS DATE) = CAST(GETDATE() AS DATE)
    """
)

# Leads for a call center agent (row filter)
vn.train(
    question="show me my leads",
    sql="""
    SELECT
        l.LeadId,
        l.FirstName,
        l.LastName,
        l.Phone,
        l.StatusId,
        b.BranchName
    FROM XXX_RapidOne.dbo.Leads l
    JOIN XXX_RapidOne.dbo.Branches b ON l.Branch = b.BranchId
    WHERE l.AssignedTo = @UserId
    AND l.IsDeleted = 0
    """
)

# Invoices from issuer DB (cross-DB JOIN with main DB)
vn.train(
    question="show invoices with branch names",
    sql="""
    SELECT
        i.InvoiceId,
        i.Amount,
        i.InvoiceDate,
        b.BranchName
    FROM XXX_Issuer1.dbo.Invoices i
    JOIN XXX_RapidOne.dbo.Branches b ON i.BranchId = b.BranchId
    """
)

# Hebrew question example
vn.train(
    question="הראה לי את התורים של היום",
    sql="""
    SELECT
        a.AppointmentId,
        a.Date,
        c.CardName AS PatientName
    FROM XXX_RapidOne.dbo.Appointments a
    JOIN XXX_Rapid_Common.dbo.OCRD c ON a.PatientId = c.CardCode
    WHERE CAST(a.Date AS DATE) = CAST(GETDATE() AS DATE)
    """
)
```

### 4.3 The XXX_ Replacement — How It Works At Runtime

After Vanna generates SQL with `XXX_` prefix, we do a simple string replacement:

```python
DB_PREFIX = "Keshevrav"   # from .env config

# Vanna generates:
sql = "SELECT * FROM XXX_RapidOne.dbo.Leads WHERE IsDeleted = 0"

# Replace XXX_ with actual customer prefix:
sql = sql.replace("XXX_", f"{DB_PREFIX}_")

# Result:
sql = "SELECT * FROM Keshevrav_RapidOne.dbo.Leads WHERE IsDeleted = 0"
```

This ONE replacement handles ALL databases:

```
XXX_RapidOne       → Keshevrav_RapidOne
XXX_Rapid_Common   → Keshevrav_Rapid_Common
XXX_Issuer1        → Keshevrav_Issuer1
XXX_Issuer2        → Keshevrav_Issuer2
```

**Why `XXX_` replacement is safe:**
- `XXX_RapidOne`, `XXX_Rapid_Common`, `XXX_Issuer1` are very specific strings
- They only appear as database names in fully-qualified table references
- They won't appear inside column values, string literals, or WHERE conditions
- After replacement, SQL validation checks all database names against the whitelist

**Why not just replace `XXX` without the underscore?**
- `XXX` could appear inside string values like `'XXXL size'` or column aliases
- `XXX_` is more specific and only matches the database prefix pattern
- Safe and deterministic

### 4.4 Hebrew Entity Mapping

The training and system prompt must include Hebrew-to-English entity mapping so the AI understands Hebrew questions:

```
Hebrew Term     →  English Entity    →  Table
לידים            →  Leads             →  Leads (in XXX_RapidOne)
תורים            →  Appointments      →  Appointments (in XXX_RapidOne)
מטופלים          →  Patients          →  OCRD (in XXX_Rapid_Common)
חשבוניות         →  Invoices          →  Invoices (in XXX_IssuerN)
קבלות            →  Receipts          →  Receipts (in XXX_IssuerN)
רופאים           →  Doctors           →  AspNetUsers (in XXX_RapidOne)
טיפולים          →  Treatments        →  Treatments (in XXX_Rapid_Common)
סניפים           →  Branches          →  Branches (in XXX_RapidOne)
מחלקות           →  Departments       →  Departments (in XXX_RapidOne)
הלידים שלי       →  My Leads          →  Leads WHERE AssignedTo = @UserId
התורים שלי       →  My Appointments   →  Appointments WHERE DoctorId IN (linked schedules)
```

SQL is ALWAYS generated in English regardless of the question language.
The response text should be in the same language as the question.

### 4.5 Training Is Done ONCE — Shared Across All Customers

```
Training flow:
    1. Train Vanna ONCE with XXX_ prefix DDL + metadata + sample pairs
    2. Store trained embeddings in vector database (ChromaDB/Qdrant)
    3. All customer processes on the server point to the SAME vector store
    4. Each process replaces XXX_ with its own DB_PREFIX after SQL generation

What's SHARED (one copy for all customers):
    - Vanna training data (DDL, metadata, sample pairs)
    - Vector store (embeddings)

What's UNIQUE per request:
    - System prompt (user context, permissions, mandatory filters)
    - Available issuer DB names (per user's Layer 5 permissions)
    - XXX_ → actual prefix replacement (per customer config)
    - Conversation memory (per user session)
```

---

## Part 5: Complete Processing Pipeline (Step by Step)

When FastAPI receives a request from .NET, here is exactly what happens:

### Step 1 — Validate X-API-KEY

```python
api_key = request.headers.get("X-API-KEY")
if api_key != AI_API_KEY:
    return 403 Forbidden
```

Already covered in AUTH_FLOW.md. If the key doesn't match, stop immediately.

### Step 2 — Build User Context

Extract user info from the .NET-enriched request body:

```
user_id         = body.user_id                 → "user-guid-123"
roles           = body.roles                   → ["DoctorWithCalendar"]       (Layer 1)
permissions     = body.permissions             → ["Schedule - Open the..."]   (Layer 2 + 3)
department_id   = body.department_id           → 5
branch_id       = body.branch_id              → 2
default_issuer  = body.default_issuer          → "Issuer1"
database        = body.database                → "Keshevrav_RapidOne"
language        = body.language               → "he-IL"
```

Extract the customer prefix from the database name:

```python
# body.database = "Keshevrav_RapidOne"
DB_PREFIX = body.database.replace("_RapidOne", "")   # → "Keshevrav"
```

Load remaining layers from the customer database:

```sql
-- Layer 4: Which doctors' schedules can this user see?
SELECT ScheduleId FROM Keshevrav_RapidOne.dbo.LinkUsersToSchedules
WHERE UserId = 'user-guid-123'
→ Result: ["doctor-guid-1", "doctor-guid-2"]

-- Layer 5: Which issuer databases can this user access?
SELECT IssuerId FROM Keshevrav_RapidOne.dbo.LinkIssuersToUsers
WHERE UserId = 'user-guid-123'
→ Result: ["Issuer1", "Issuer2"]
→ Full DB names: ["Keshevrav_Issuer1", "Keshevrav_Issuer2"]  (prefix + IssuerId)
```

Now we have ALL 5 permission layers:

```
Layer 1 (Roles):              ["DoctorWithCalendar"]              ← from .NET
Layer 2 (UserPermissions):    ["Schedule - Open the..."]          ← from .NET (combined)
Layer 3 (ProfilePermissions): (merged into Layer 2 above)         ← from .NET (combined)
Layer 4 (ScheduleAccess):     ["doctor-guid-1", "doctor-guid-2"] ← from DB
Layer 5 (IssuerAccess):       ["Issuer1", "Issuer2"]             ← from DB
    → Full DB names:          ["Keshevrav_Issuer1", "Keshevrav_Issuer2"]
```

### Step 3 — Detect Entity Type

Before generating SQL, we need to understand what the question is about. This determines which permission checks apply:

```
Question: "show me today's appointments"
    → Entity detected: APPOINTMENTS
    → Permission check: Layer 2 ("Schedule - Open the schedule") + Layer 4 (linked schedules)

Question: "show me my leads"
    → Entity detected: LEADS
    → Permission check: Layer 1 (must have CallCenterAgent or CallCenterManager role)

Question: "show all invoices this month"
    → Entity detected: INVOICES
    → Permission check: Layer 5 (must have linked issuers)

Question: "show patient details for Moshe"
    → Entity detected: PATIENTS
    → Permission check: Layer 2 (must have relevant patient access permission)

Question: "הראה לי את הלידים שלי"
    → Detected language: Hebrew
    → Entity detected: LEADS (הלידים = Leads)
    → Same permission check as English
```

Entity detection can be done by:
1. Keyword matching (fast, for common terms)
2. LLM classification (for complex/ambiguous questions)
3. Combination of both

### Step 4 — Permission Check (Can This User Access This Entity?)

Based on the detected entity, check if the user has permission. If not, return a friendly error — DO NOT generate any SQL.

**LEADS:**
```
Check roles (Layer 1):
    Has "CallCenterAgent"?     → ALLOWED (will filter: WHERE AssignedTo = @UserId AND IsDeleted = 0)
    Has "CallCenterManager"?   → ALLOWED (will filter: WHERE Branch IN (permitted branches) AND IsDeleted = 0)
    Has neither?               → BLOCKED ("You don't have access to lead data")
```

**APPOINTMENTS:**
```
Check permissions (Layer 2+3):
    Has "Schedule - Open the schedule"?  → Check Layer 4
        Has linked schedules?            → ALLOWED (will filter: WHERE DoctorId IN (linked doctor IDs))
        No linked schedules?             → BLOCKED ("You don't have any linked schedules")
    Doesn't have schedule permission?    → BLOCKED ("You don't have permission to view schedules")
```

**INVOICES / RECEIPTS:**
```
Check Layer 5:
    Has linked issuers?    → ALLOWED (will query only permitted issuer databases)
    No linked issuers?     → BLOCKED ("You don't have access to financial data")
```

**PATIENTS / TREATMENTS:**
```
Check permissions (Layer 2+3):
    Has relevant permission?   → ALLOWED (with department filter)
    No permission?             → BLOCKED ("You don't have permission to view patient data")
```

**If blocked: return error response immediately. No SQL generation happens.**

### Step 5 — Build User-Specific Schema

For allowed entities, build a restricted DDL that ONLY contains tables and databases this user can access. This is the PRE-HOC approach — the AI can only see what the user is allowed to see.

The schema uses `XXX_` prefix (same as training), because Vanna was trained with `XXX_`. We replace at Step 7b.

Example for a DoctorWithCalendar user who has schedule permission + 2 linked schedules + 2 issuers:

```
Allowed databases for this user (XXX_ form for Vanna):
    ✅ XXX_RapidOne              (always — main DB)
    ✅ XXX_Rapid_Common          (always — patient data)
    ✅ XXX_Issuer1               (from Layer 5)
    ✅ XXX_Issuer2               (from Layer 5)
    ❌ XXX_Issuer3               (user has no access — NOT included)

Allowed tables:
    ✅ Appointments (with mandatory filter: DoctorId IN ('doctor-guid-1', 'doctor-guid-2'))
    ✅ Departments, Branches (reference tables)
    ✅ OCRD (patient details)
    ✅ Invoices, Receipts (from permitted issuers only)
    ❌ Leads (user has no CC role — table NOT included in schema)
```

The DDL sent to Vanna ONLY includes the allowed tables. If Leads table is not in the DDL, the LLM cannot generate SQL referencing it — security by omission.

### Step 6 — Build System Prompt

The system prompt is the most important part. It tells the LLM everything about:
- Who the user is
- What they can see
- What mandatory filters must be applied
- How to handle Hebrew/English
- Available databases

Example system prompt for this user:

```
You are a SQL query assistant for a clinic management system.

CURRENT USER:
- Name: Dr. Cohen
- User ID: user-guid-123
- Role: DoctorWithCalendar
- Department: ID 5
- Branch: ID 2
- Language: Hebrew (he-IL)

AVAILABLE DATABASES (use XXX_ prefix in all table references):
- XXX_RapidOne (main app — appointments, branches, departments)
- XXX_Rapid_Common (SAP — patients OCRD table, treatments)
- XXX_Issuer1 (financial — invoices, receipts)
- XXX_Issuer2 (financial — invoices, receipts)

MANDATORY RULES — YOU MUST FOLLOW THESE:
1. ALL table references MUST use fully-qualified names: XXX_DatabaseName.dbo.TableName
2. Appointment queries MUST include: WHERE DoctorId IN ('doctor-guid-1', 'doctor-guid-2')
3. Invoice/Receipt queries MUST only reference XXX_Issuer1 or XXX_Issuer2 (no other issuer databases)
4. ONLY generate SELECT statements. Never INSERT, UPDATE, DELETE, DROP, or any DDL.
5. If the user asks about Leads — respond that they don't have access to lead data.
6. Always use the XXX_ prefix for database names.

SCHEMA:
[User-specific DDL from Step 5 — only allowed tables, all with XXX_ prefix]

HEBREW ENTITY MAPPING:
תורים = Appointments, מטופלים = Patients (OCRD table), חשבוניות = Invoices, רופאים = Doctors (AspNetUsers)

RESPONSE LANGUAGE:
- SQL is always in English
- Explanation text should be in Hebrew (user's language is he-IL)
```

### Step 7a — Vanna Generates SQL (With XXX_ Prefix)

With the restricted schema and security rules baked into the system prompt, Vanna generates SQL using `XXX_` prefix:

```
User question: "show me today's appointments"

Vanna generates:
    SELECT
        a.AppointmentId,
        a.Date,
        u.Name AS DoctorName,
        c.CardName AS PatientName,
        c.Phone1,
        d.DepartmentName
    FROM XXX_RapidOne.dbo.Appointments a
    JOIN XXX_Rapid_Common.dbo.OCRD c ON a.PatientId = c.CardCode
    JOIN XXX_RapidOne.dbo.AspNetUsers u ON a.DoctorId = u.Id
    JOIN XXX_RapidOne.dbo.Departments d ON a.DepartmentId = d.DepartmentId
    WHERE CAST(a.Date AS DATE) = CAST(GETDATE() AS DATE)
    AND a.DoctorId IN ('doctor-guid-1', 'doctor-guid-2')
```

The mandatory filter `DoctorId IN (...)` is included because the system prompt instructed the LLM to always include it. This is PRE-HOC — the security is in the prompt, not injected after.

### Step 7b — Replace XXX_ With Actual Customer Prefix

```python
DB_PREFIX = "Keshevrav"   # from config

sql = sql.replace("XXX_", f"{DB_PREFIX}_")
```

Before replacement:
```sql
FROM XXX_RapidOne.dbo.Appointments a
JOIN XXX_Rapid_Common.dbo.OCRD c ON ...
```

After replacement:
```sql
FROM Keshevrav_RapidOne.dbo.Appointments a
JOIN Keshevrav_Rapid_Common.dbo.OCRD c ON ...
```

All databases replaced in one line. Simple, deterministic, safe.

### Step 8 — SQL Validation (5 Security Checks)

After prefix replacement, validate the final SQL. Even though the prompt restricts the LLM, we validate as a safety net:

```
Check 1: Is it SELECT only?
    → Scan for INSERT, UPDATE, DELETE, DROP, TRUNCATE, ALTER, CREATE
    → If found → BLOCKED

Check 2: No dangerous keywords?
    → Scan for EXEC, sp_, xp_, GRANT, REVOKE, OPENROWSET, OPENDATASOURCE
    → If found → BLOCKED

Check 3: All referenced databases in allowed list?
    → Parse SQL → extract database names
    → Check against: ["Keshevrav_RapidOne", "Keshevrav_Rapid_Common", "Keshevrav_Issuer1", "Keshevrav_Issuer2"]
    → If any database not in list → BLOCKED

Check 4: All referenced tables in allowed list?
    → Parse SQL → extract table names
    → Check against user's allowed tables from Step 5
    → If any table not allowed → BLOCKED

Check 5: Mandatory row-level filters present?
    → For Appointments: check DoctorId IN (...) is in WHERE clause
    → For Leads (if somehow generated): check AssignedTo = @UserId
    → For Invoices: check database is in user's allowed issuers
    → If mandatory filter missing → BLOCKED

All 5 pass → proceed to execution
Any fail → return error ("Query blocked: security validation failed")
```

### Step 9 — Execute SQL

```python
# Build allowed database list for this user (with actual prefix)
allowed_dbs = [
    f"{DB_PREFIX}_RapidOne",             # "Keshevrav_RapidOne"
    f"{DB_PREFIX}_Rapid_Common",         # "Keshevrav_Rapid_Common"
]
# Add user's permitted issuers with prefix
for issuer_id in user_allowed_issuers:   # ["Issuer1", "Issuer2"]
    allowed_dbs.append(f"{DB_PREFIX}_{issuer_id}")   # "Keshevrav_Issuer1", "Keshevrav_Issuer2"

# Execute with timeout and row limit
connection = get_connection(database=f"{DB_PREFIX}_RapidOne")  # primary connection
result = connection.execute(validated_sql, timeout=30)          # 30-second timeout
rows = result.fetchmany(1000)                                   # max 1000 rows
columns = [desc[0] for desc in result.description]
```

Cross-database queries work because all databases are on the same SQL Server instance. We use a single connection to the primary database, and the fully-qualified table names handle cross-DB access automatically.

### Step 10 — Format and Return Response

```python
response = {
    "message": "Here are today's appointments for your linked schedules:",
    "sql_query": "SELECT a.AppointmentId, a.Date, ...",
    "columns": ["AppointmentId", "Date", "DoctorName", "PatientName", "Phone1", "DepartmentName"],
    "rows": [
        [101, "2026-02-23 09:00", "Dr. Cohen", "Moshe Levi", "054-1234567", "Dermatology"],
        [102, "2026-02-23 10:30", "Dr. Cohen", "Sara Ben-David", "052-9876543", "Dermatology"]
    ],
    "row_count": 2,
    "language": "he-IL"
}
```

This JSON goes back to .NET, which forwards it to Angular, which displays it in the chatbox.

---

## Part 6: Multi-Issuer Query Handling

A user may have access to multiple issuer databases. All issuer DBs follow the `XXX_IssuerN` naming pattern.

### Scenario 1: User asks about one specific issuer

```
Question: "show invoices from Issuer1"

Vanna generates (with XXX_ prefix):
    SELECT * FROM XXX_Issuer1.dbo.Invoices WHERE ...

After replacement:
    SELECT * FROM Keshevrav_Issuer1.dbo.Invoices WHERE ...

Validate Keshevrav_Issuer1 is in user's allowed list → YES → Execute
```

### Scenario 2: User asks "show all invoices" (multiple issuers)

```
User has access to: ["Issuer1", "Issuer2"]
System prompt says: "Available issuer databases: XXX_Issuer1, XXX_Issuer2"

Vanna generates:
    SELECT InvoiceId, Amount, InvoiceDate, 'Issuer1' AS Source
    FROM XXX_Issuer1.dbo.Invoices
    WHERE InvoiceDate >= '2026-02-01'
    UNION ALL
    SELECT InvoiceId, Amount, InvoiceDate, 'Issuer2' AS Source
    FROM XXX_Issuer2.dbo.Invoices
    WHERE InvoiceDate >= '2026-02-01'

After replacement (XXX_ → Keshevrav_):
    SELECT InvoiceId, Amount, InvoiceDate, 'Issuer1' AS Source
    FROM Keshevrav_Issuer1.dbo.Invoices
    WHERE InvoiceDate >= '2026-02-01'
    UNION ALL
    SELECT InvoiceId, Amount, InvoiceDate, 'Issuer2' AS Source
    FROM Keshevrav_Issuer2.dbo.Invoices
    WHERE InvoiceDate >= '2026-02-01'

Note: 'Issuer1' inside the string literal is NOT affected by replacement
because we replace "XXX_" not "Issuer1"
```

### Scenario 3: User has no issuer access

```
Layer 5 returns empty list → []
→ Issuer tables are NOT included in the schema DDL
→ If user asks about invoices → Step 4 blocks: "You don't have access to financial data"
→ No SQL generated
```

### Scenario 4: Cross-DB JOIN with issuer + main DB

```
Question: "show invoices with branch names"

Vanna generates:
    SELECT i.InvoiceId, i.Amount, b.BranchName
    FROM XXX_Issuer1.dbo.Invoices i
    JOIN XXX_RapidOne.dbo.Branches b ON i.BranchId = b.BranchId

After replacement:
    SELECT i.InvoiceId, i.Amount, b.BranchName
    FROM Keshevrav_Issuer1.dbo.Invoices i
    JOIN Keshevrav_RapidOne.dbo.Branches b ON i.BranchId = b.BranchId

Works because all databases are on the same SQL Server instance.
```

---

## Part 7: Permission Layer Details — Per Entity

### Layer 1: Roles (from .NET → `body.roles`)

| Role | Lead Access | Appointment Access | Patient Access | Invoice Access |
|---|---|---|---|---|
| CallCenterAgent | Own only (AssignedTo = UserId) | If has schedule permission | If has permission | If has linked issuers |
| CallCenterManager | All in permitted departments | If has schedule permission | If has permission | If has linked issuers |
| DoctorWithCalendar | No | Own schedule (linked schedules) | If has permission | If has linked issuers |
| Receptionist | No | If has schedule permission | If has permission | If has linked issuers |
| Admin | All | All | All | All |

### Layer 2 + 3: Permissions (from .NET → `body.permissions`)

These are **combined** by .NET's getUserInfo() into one array. Examples:
- `"Schedule - Open the schedule"` — required for appointment access
- `"Leads - View leads"` — required for lead access (in addition to CC role)
- `"Patients - View patient info"` — required for patient data

### Layer 4: Schedule Access (from DB → `LinkUsersToSchedules`)

```sql
SELECT ScheduleId FROM XXX_RapidOne.dbo.LinkUsersToSchedules WHERE UserId = @user_id
```

- ScheduleId = DoctorId (a doctor's "schedule" IS their user ID)
- Used to filter Appointments: `WHERE DoctorId IN (linked schedule IDs)`
- If empty → no appointment data visible

### Layer 5: Issuer Access (from DB → `LinkIssuersToUsers`)

```sql
SELECT IssuerId FROM XXX_RapidOne.dbo.LinkIssuersToUsers WHERE UserId = @user_id
```

- IssuerId is the issuer identifier (e.g., `Issuer1`)
- Full database name = `{DB_PREFIX}_{IssuerId}` (e.g., `Keshevrav_Issuer1`)
- Used to determine which financial databases the user can query
- If empty → no invoice/receipt data visible

---

## Part 8: Database Whitelist — What Can Be Queried

For every request, build a whitelist of allowed databases (with actual customer prefix):

```python
DB_PREFIX = "Keshevrav"

# Always allowed
allowed_dbs = [
    f"{DB_PREFIX}_RapidOne",        # "Keshevrav_RapidOne"
    f"{DB_PREFIX}_Rapid_Common",    # "Keshevrav_Rapid_Common"
]

# Add user's permitted issuers with prefix
for issuer_id in user_allowed_issuers:
    allowed_dbs.append(f"{DB_PREFIX}_{issuer_id}")
# → adds "Keshevrav_Issuer1", "Keshevrav_Issuer2"

# Always blocked (system databases)
blocked_dbs = ["master", "tempdb", "model", "msdb"]
```

Before executing any SQL, parse all database names referenced in the query and verify every one is in `allowed_dbs` and not in `blocked_dbs`.

---

## Part 9: Error Handling

Every step in the pipeline can fail. Here's what happens at each point:

| Step | Failure | Response to User |
|---|---|---|
| Step 1 (API Key) | X-API-KEY invalid | 403 Forbidden (never reaches user — .NET handles) |
| Step 2 (User Context) | Layer 4/5 DB query fails | 500 "Service temporarily unavailable" |
| Step 3 (Entity Detection) | Cannot determine entity | "I couldn't understand your question. Please try rephrasing." |
| Step 4 (Permission Check) | User has no permission | "You don't have access to [entity] data. Contact your admin." |
| Step 5 (Schema Building) | No allowed tables | "You don't have any data access configured. Contact your admin." |
| Step 7a (SQL Generation) | Vanna/OpenAI fails | "Failed to generate query. Please try again." |
| Step 7b (Prefix Replace) | XXX_ still in SQL after replace | "Internal error. Please try again." (should never happen) |
| Step 8 (SQL Validation) | Security check fails | "Query blocked for security reasons. Please rephrase your question." |
| Step 9 (SQL Execution) | Database timeout | "Query took too long. Try a more specific question." |
| Step 9 (SQL Execution) | SQL error | "Query failed. Please try a different question." |
| Step 9 (SQL Execution) | No results | "No data found matching your query." |

---

## Part 10: Architecture Components

```
FastAPI Server (port 8001 per customer)
│
├── APIKeyValidator
│       Validates X-API-KEY from .NET against .env
│
├── UserContextBuilder
│       Builds complete user context from .NET enriched request
│       Extracts DB_PREFIX from body.database
│       Loads Layer 4 (LinkUsersToSchedules) from DB
│       Loads Layer 5 (LinkIssuersToUsers) from DB
│
├── EntityDetector
│       Classifies question → entity type (leads/appointments/invoices/patients)
│       Handles Hebrew and English
│
├── PermissionChecker
│       Checks if user can access the detected entity
│       Uses all 5 layers
│       Returns: allowed (with filters) or blocked (with reason)
│
├── SchemaBuilder
│       Builds user-specific DDL — only permitted tables and databases
│       Uses XXX_ prefix (matching Vanna training)
│
├── SystemPromptBuilder
│       Constructs the LLM system prompt with:
│       - User context (name, role, department)
│       - Restricted schema DDL (XXX_ prefix)
│       - Mandatory row filters
│       - Security rules
│       - Hebrew entity mapping
│       - Available databases (XXX_ prefix)
│
├── VannaManager
│       Manages the SHARED Vanna instance (one per server, not per customer)
│       Points to shared vector store (trained with XXX_ prefix)
│       Sends question to Vanna with user-specific system prompt
│       Receives generated SQL (with XXX_ prefix)
│
├── PrefixReplacer
│       Replaces XXX_ with actual customer prefix (DB_PREFIX)
│       sql.replace("XXX_", f"{DB_PREFIX}_")
│       Validates no XXX_ remains after replacement
│
├── SQLValidator
│       5 security checks (runs AFTER prefix replacement):
│       1. SELECT-only
│       2. No dangerous keywords
│       3. Allowed databases only (actual names, not XXX_)
│       4. Allowed tables only
│       5. Mandatory row filters present
│
├── DatabaseRouter
│       Routes query to correct database connection
│       Manages connection pool (2 always-on + lazy issuer connections)
│
└── QueryExecutor
        Executes validated SQL with timeout (30s) and row limit (1000)
        Returns columns + rows
```

---

## Part 11: Configuration Per Customer

Each customer deployment has ONE `.env` file:

```env
# Server
PORT=8001                                    # unique per customer on same server

# Security
AI_API_KEY=the-shared-secret-from-web-config  # must match .NET's web.config

# Database
DB_SERVER=156.67.105.206                      # SQL Server IP
DB_PREFIX=Keshevrav                           # customer prefix (used for XXX_ replacement)
DB_USERNAME=sa                                # SQL login
DB_PASSWORD=xxxxx                             # SQL password

# Vanna (shared across all customers on this server)
VANNA_STORE_PATH=/path/to/shared/vanna/store  # shared vector store location

# OpenAI
OPENAI_API_KEY=sk-xxxxx                       # for Vanna's LLM
OPENAI_MODEL=gpt-4o-mini                      # model for SQL generation

# Limits
QUERY_TIMEOUT=30                              # seconds
MAX_ROWS=1000                                 # max rows returned
```

Same code. Same trained Vanna. Only config changes per customer.

### Server-Level Shared Resources

```
Server A (156.67.105.206):
│
├── Shared Vanna Training Store (ONE copy)
│       Trained with XXX_ prefix
│       Used by ALL customer processes on this server
│
├── Customer: Keshevrav (port 8001, DB_PREFIX=Keshevrav)
│       XXX_ → Keshevrav_ replacement
│       Databases: Keshevrav_RapidOne, Keshevrav_Rapid_Common, Keshevrav_Issuer1
│
├── Customer: ClinicB (port 8002, DB_PREFIX=ClinicB)
│       XXX_ → ClinicB_ replacement
│       Databases: ClinicB_RapidOne, ClinicB_Rapid_Common, ClinicB_Issuer1, ClinicB_Issuer2
│
├── Customer: ClinicC (port 8003, DB_PREFIX=ClinicC)
│       XXX_ → ClinicC_ replacement
│       Databases: ClinicC_RapidOne, ClinicC_Rapid_Common, ClinicC_Issuer1
│
└── ... (up to N customers)
```

---

## Part 12: What Each Person Needs To Do

### Us (Backend — FastAPI + Vanna):
1. Train Vanna ONCE with XXX_ prefix DDL + metadata + sample pairs (Part 4)
2. Build FastAPI server with all components from Part 10
3. Implement XXX_ → DB_PREFIX replacement after SQL generation (Part 4.3)
4. Implement entity detection (Part 5, Step 3)
5. Implement 5-layer permission checking (Part 5, Step 4)
6. Implement user-specific schema building with XXX_ prefix (Part 5, Step 5)
7. Implement system prompt builder with Hebrew mapping (Part 5, Step 6)
8. Implement SQL validation — 5 security checks (Part 5, Step 8)
9. Implement database connection management with lazy issuer connections
10. Implement response formatting

### Client .NET Team:
1. Create `POST /api/ai/chat` endpoint (covered in AUTH_FLOW.md)
2. Store X-API-KEY in web.config
3. Forward enriched request to FastAPI (with user info from getUserInfo())
4. Forward FastAPI response back to Angular

### Frontend Developer:
1. Create chatbox component
2. `$http.post('/api/ai/chat', { query: "..." })` — auth is automatic
3. Display response: text message + data table
4. RTL CSS for Hebrew

---

## Part 13: Security Summary

| Security Layer | What It Does | Where |
|---|---|---|
| Bearer Token | Proves user is authenticated | Angular → .NET |
| X-API-KEY | Proves request came from .NET | .NET → FastAPI |
| Entity Detection + Permission Check | Blocks access to unauthorized entities BEFORE SQL generation | FastAPI |
| User-Specific Schema (PRE-HOC) | LLM only sees allowed tables/databases | FastAPI → Vanna |
| Mandatory Row Filters in Prompt (PRE-HOC) | LLM generates SQL with security filters built in | FastAPI → Vanna |
| XXX_ Prefix Training | Generated SQL uses placeholder, not real DB names. Real names only appear after replacement | Vanna → FastAPI |
| SQL Validation (5 checks) | Safety net — catches anything the LLM might generate wrong | FastAPI |
| Database Whitelist | Blocks queries to unauthorized databases | FastAPI |
| Query Timeout + Row Limit | Prevents resource abuse | FastAPI |

The key principle is **PRE-HOC security**: restrict what the LLM sees BEFORE it generates SQL, rather than trying to fix the SQL after generation.

---

## Part 14: Why PRE-HOC Is Better Than POST-HOC

**POST-HOC approach (what the original document proposed):**
```
1. Generate SQL with full schema access
2. After generation, inject WHERE clauses: AND ScheduleId IN (1,2,3) AND DepartmentId = 5
```

Problems with POST-HOC:
- The LLM might generate complex SQL (CTEs, subqueries, UNION) where injection point is ambiguous
- Injecting WHERE clauses into existing SQL can break syntax
- The LLM might reference tables the user can't access — we'd have to detect and block after the fact
- No protection if the LLM hallucinates a different table name

**PRE-HOC approach (what we implement):**
```
1. Build restricted schema — only include tables the user can access
2. Add mandatory filters as RULES in the system prompt
3. LLM generates SQL that is already filtered and restricted
4. Replace XXX_ with actual customer prefix
5. SQL validation is just a safety net, not the primary security
```

Benefits of PRE-HOC:
- The LLM CANNOT reference tables not in the schema — they don't exist in its context
- The LLM generates correct filters because they're in the prompt rules
- No post-processing or SQL manipulation needed
- Works with any SQL complexity (CTEs, subqueries, UNION, etc.)
- SQL validation becomes a safety net, not the only defense

---

## Part 15: Example End-to-End Scenarios

### Scenario A: Doctor asks about today's appointments

```
User: Dr. Cohen (DoctorWithCalendar role)
Customer: Keshevrav (DB_PREFIX = "Keshevrav")
Question: "show me today's appointments"

Step 3 → Entity: APPOINTMENTS
Step 4 → Has "Schedule - Open the schedule"? YES
         Has linked schedules? YES → ["doctor-guid-1", "doctor-guid-2"]
         → ALLOWED

Step 5 → Schema includes: Appointments, OCRD, Departments, Branches, AspNetUsers (all with XXX_ prefix)
         Schema excludes: Leads (no CC role), Invoices (not requested)

Step 6 → Prompt includes: "Appointment queries MUST include: WHERE DoctorId IN ('doctor-guid-1', 'doctor-guid-2')"

Step 7a → Vanna generates (with XXX_ prefix):
    SELECT a.Date, c.CardName AS PatientName, d.DepartmentName
    FROM XXX_RapidOne.dbo.Appointments a
    JOIN XXX_Rapid_Common.dbo.OCRD c ON a.PatientId = c.CardCode
    JOIN XXX_RapidOne.dbo.Departments d ON a.DepartmentId = d.DepartmentId
    WHERE CAST(a.Date AS DATE) = CAST(GETDATE() AS DATE)
    AND a.DoctorId IN ('doctor-guid-1', 'doctor-guid-2')

Step 7b → Replace XXX_ → Keshevrav_:
    SELECT a.Date, c.CardName AS PatientName, d.DepartmentName
    FROM Keshevrav_RapidOne.dbo.Appointments a
    JOIN Keshevrav_Rapid_Common.dbo.OCRD c ON a.PatientId = c.CardCode
    JOIN Keshevrav_RapidOne.dbo.Departments d ON a.DepartmentId = d.DepartmentId
    WHERE CAST(a.Date AS DATE) = CAST(GETDATE() AS DATE)
    AND a.DoctorId IN ('doctor-guid-1', 'doctor-guid-2')

Step 8 → All 5 checks pass
Step 9 → Execute → 5 rows returned
Step 10 → Return JSON with columns + rows
```

### Scenario B: Call center agent asks about leads (Hebrew)

```
User: Sarah (CallCenterAgent role)
Customer: ClinicB (DB_PREFIX = "ClinicB")
Question: "הראה לי את הלידים שלי" (Show me my leads)

Step 3 → Language: Hebrew → Entity: LEADS (הלידים = Leads, שלי = my)
Step 4 → Has CallCenterAgent role? YES
         → ALLOWED (filter: WHERE AssignedTo = 'sarah-guid' AND IsDeleted = 0)

Step 5 → Schema includes: Leads, Branches (with XXX_ prefix)
         Schema excludes: Appointments (no schedule permission), Invoices

Step 6 → Prompt includes: "Lead queries MUST include: WHERE AssignedTo = 'sarah-guid' AND IsDeleted = 0"

Step 7a → Vanna generates:
    SELECT l.LeadId, l.FirstName, l.LastName, l.Phone, b.BranchName
    FROM XXX_RapidOne.dbo.Leads l
    JOIN XXX_RapidOne.dbo.Branches b ON l.Branch = b.BranchId
    WHERE l.AssignedTo = 'sarah-guid'
    AND l.IsDeleted = 0

Step 7b → Replace XXX_ → ClinicB_:
    SELECT l.LeadId, l.FirstName, l.LastName, l.Phone, b.BranchName
    FROM ClinicB_RapidOne.dbo.Leads l
    JOIN ClinicB_RapidOne.dbo.Branches b ON l.Branch = b.BranchId
    WHERE l.AssignedTo = 'sarah-guid'
    AND l.IsDeleted = 0

Step 8 → All 5 checks pass
Step 9 → Execute → 12 rows returned
Step 10 → Return JSON (response text in Hebrew since question was Hebrew)
```

### Scenario C: Receptionist asks about leads (BLOCKED)

```
User: Rachel (Receptionist role, no CC role)
Customer: Keshevrav
Question: "show me all leads"

Step 3 → Entity: LEADS
Step 4 → Has CallCenterAgent role? NO
         Has CallCenterManager role? NO
         → BLOCKED

Response: "You don't have access to lead data. Lead access requires a Call Center role."
No SQL generated. No database query executed. No Vanna call.
```

### Scenario D: User asks about invoices across multiple issuers

```
User: Finance Manager (has issuers: Issuer1, Issuer2, Issuer3)
Customer: Keshevrav (DB_PREFIX = "Keshevrav")
Question: "show total invoices this month by issuer"

Step 3 → Entity: INVOICES
Step 4 → Has linked issuers? YES → ["Issuer1", "Issuer2", "Issuer3"]
         → ALLOWED

Step 5 → Schema includes: Invoices from XXX_Issuer1, XXX_Issuer2, XXX_Issuer3
Step 6 → Prompt includes: "Available issuer databases: XXX_Issuer1, XXX_Issuer2, XXX_Issuer3"

Step 7a → Vanna generates:
    SELECT 'Issuer1' AS IssuerName, COUNT(*) AS InvoiceCount, SUM(Amount) AS Total
    FROM XXX_Issuer1.dbo.Invoices WHERE MONTH(InvoiceDate) = MONTH(GETDATE())
    UNION ALL
    SELECT 'Issuer2', COUNT(*), SUM(Amount)
    FROM XXX_Issuer2.dbo.Invoices WHERE MONTH(InvoiceDate) = MONTH(GETDATE())
    UNION ALL
    SELECT 'Issuer3', COUNT(*), SUM(Amount)
    FROM XXX_Issuer3.dbo.Invoices WHERE MONTH(InvoiceDate) = MONTH(GETDATE())

Step 7b → Replace XXX_ → Keshevrav_:
    SELECT 'Issuer1' AS IssuerName, COUNT(*) AS InvoiceCount, SUM(Amount) AS Total
    FROM Keshevrav_Issuer1.dbo.Invoices WHERE MONTH(InvoiceDate) = MONTH(GETDATE())
    UNION ALL
    SELECT 'Issuer2', COUNT(*), SUM(Amount)
    FROM Keshevrav_Issuer2.dbo.Invoices WHERE MONTH(InvoiceDate) = MONTH(GETDATE())
    UNION ALL
    SELECT 'Issuer3', COUNT(*), SUM(Amount)
    FROM Keshevrav_Issuer3.dbo.Invoices WHERE MONTH(InvoiceDate) = MONTH(GETDATE())

    (Note: 'Issuer1' string literals are NOT affected — only "XXX_" prefix is replaced)

Step 8 → All databases in allowed list? YES → Pass
Step 9 → Execute → 3 rows returned
Step 10 → Return JSON
```

---

## Summary

This document covers the complete Vanna AI flow from the moment FastAPI receives the .NET-enriched request to the moment the response is returned.

**Key principles:**
1. **ONE shared Vanna per server** — trained once with `XXX_` prefix, shared by all customers on the server. Saves memory, simplifies training updates.
2. **XXX_ prefix replacement** — Vanna generates SQL with `XXX_` placeholder. Simple `sql.replace("XXX_", f"{DB_PREFIX}_")` converts to actual customer database names. Works for ALL databases (RapidOne, Rapid_Common, Issuer1, Issuer2, etc.)
3. **PRE-HOC security** — restrict what the LLM sees before SQL generation, don't fix SQL after
4. **5-layer permission enforcement** — 3 layers from .NET, 2 layers from DB
5. **Entity detection first** — determine what the question is about, then check if the user can access it
6. **User-specific schema** — each request gets a DDL with only the tables the user can access
7. **Fully-qualified table names** — enables cross-database JOINs on the same SQL Server instance
8. **SQL validation as safety net** — 5 checks after generation as a fallback, not the primary defense
9. **Hebrew/English bilingual** — entity mapping in training and system prompt, response in question's language

**Related documents:**
- `AUTH_FLOW.md` — covers everything before this flow (login, token, .NET proxy, X-API-KEY)
- `PROJECT_CONTEXT.md` — full project context, database schema, permission system details
- `multi server approach.txt` — how multiple customers run on multiple servers

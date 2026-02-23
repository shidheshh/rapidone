# RapidOne AI Chatbox — Vanna AI Flow (Complete)

> This document explains how the AI query system works end-to-end.
> Starts where AUTH_FLOW.md ends — FastAPI has received the enriched request from .NET with all user context.
> Covers: database architecture, Vanna training, entity detection, permission enforcement, SQL generation, validation, execution, and response.

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
| Issuer DB(s) | `Issuer1`, `Issuer2`, ... `IssuerN` | Invoices, Receipts (financial documents) | Lazy — connected only when a permitted user queries financial data |

Important notes:
- Issuer DB names are the `Issuers.Id` value directly — NOT prefixed with `XXX_`. Example: `Issuer1`, not `Keshevrav_Issuer1`
- A customer can have **multiple Issuer databases** — one per financial entity
- All databases exist on the **same SQL Server instance**, so cross-database JOINs work natively

### Database Discovery on Startup

When the FastAPI server starts for a customer:

```
Read config → DB_PREFIX = "Keshevrav", DB_SERVER = "156.67.105.206"
    → Build known names: Keshevrav_RapidOne, Keshevrav_Rapid_Common
    → Connect to both (ALWAYS connected)
    → Query Issuers table: SELECT Id FROM Keshevrav_RapidOne.dbo.Issuers
    → Discover: Issuer1, Issuer2 (these ARE the database names)
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
| Invoices + Branch details | `IssuerN` + `XXX_RapidOne` | Invoices JOIN Branches ON BranchId |
| Treatments + Patient details | `XXX_Rapid_Common` (same DB) | Treatments JOIN OCRD ON CardCode |

---

## Part 4: Vanna AI Training Strategy

### 4.1 One Vanna Instance Per Customer

We use ONE Vanna instance per customer deployment. This single instance is trained with schemas from ALL databases (Main + SAP + all Issuers). Because all databases are on the same SQL Server, Vanna can generate cross-DB JOINs naturally.

### 4.2 Training Data — Three Types

Vanna needs three types of training data for accurate SQL generation:

**Type 1: DDL (Table Definitions)**

Train Vanna with fully-qualified DDL for every table the AI can access:

```python
# Main App DB tables
vn.train(ddl="""
CREATE TABLE Keshevrav_RapidOne.dbo.Appointments (
    AppointmentId INT PRIMARY KEY,
    Date DATETIME,
    DoctorId NVARCHAR(128),         -- FK to AspNetUsers.Id (doctor)
    PatientId NVARCHAR(15),         -- FK to OCRD.CardCode (patient)
    DepartmentId INT,
    StatusId INT,
    Notes NVARCHAR(MAX)
)
""")

# SAP DB tables
vn.train(ddl="""
CREATE TABLE Keshevrav_Rapid_Common.dbo.OCRD (
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

# Issuer DB tables (same structure for all issuers)
vn.train(ddl="""
CREATE TABLE {ISSUER_DB}.dbo.Invoices (
    InvoiceId INT PRIMARY KEY,
    PatientId NVARCHAR(15),
    BranchId INT,
    Amount DECIMAL(18,2),
    InvoiceDate DATETIME,
    Status INT
)
""")
```

**Type 2: Metadata (Column Descriptions + Business Context)**

DDL alone is not enough. The AI needs to understand what columns mean in the medical CMS context:

```python
vn.train(documentation="""
Table: OCRD (in Keshevrav_Rapid_Common database)
- This is the Patients/Customers table (SAP Business One naming convention)
- CardCode = Patient ID (primary key, format: patient number as string)
- CardName = Patient full name
- U_Gender: 1 = Male, 2 = Female
- This table is in the SAP database (Rapid_Common), NOT in the main RapidOne database
- To get patient details for an appointment, JOIN Appointments.PatientId = OCRD.CardCode

Table: Leads (in Keshevrav_RapidOne database)
- Branch column = physical location (int FK to Branches.BranchId), NOT DepartmentID
- AssignedTo = user ID of the assigned call center agent (nvarchar(128))
- IsDeleted: 0 = active lead, 1 = deleted/archived
- Always filter WHERE IsDeleted = 0 unless specifically asked for deleted leads

Table: Appointments (in Keshevrav_RapidOne database)
- DoctorId = the doctor who owns this appointment (nvarchar(128), FK to AspNetUsers.Id)
- There is NO 'ScheduleId' column in this table
- To filter by schedule access, use: DoctorId IN (list of allowed doctor IDs)
- DepartmentId = the specialty department where this appointment takes place

Table: LinkUsersToSchedules (in Keshevrav_RapidOne database)
- UserId = the user who is viewing (the logged-in user)
- ScheduleId = the doctor whose schedule they can see (this IS a DoctorId from AspNetUsers.Id)
- A "schedule" is not a separate entity — it is a doctor's schedule
""")
```

**Type 3: Sample Question-SQL Pairs**

These are the most important for accuracy. Train with real examples that demonstrate permission-aware queries:

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
    FROM Keshevrav_RapidOne.dbo.Appointments a
    JOIN Keshevrav_Rapid_Common.dbo.OCRD c ON a.PatientId = c.CardCode
    JOIN Keshevrav_RapidOne.dbo.AspNetUsers u ON a.DoctorId = u.Id
    JOIN Keshevrav_RapidOne.dbo.Departments d ON a.DepartmentId = d.DepartmentId
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
    FROM Keshevrav_RapidOne.dbo.Leads l
    JOIN Keshevrav_RapidOne.dbo.Branches b ON l.Branch = b.BranchId
    WHERE l.AssignedTo = @UserId
    AND l.IsDeleted = 0
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
    FROM Keshevrav_RapidOne.dbo.Appointments a
    JOIN Keshevrav_Rapid_Common.dbo.OCRD c ON a.PatientId = c.CardCode
    WHERE CAST(a.Date AS DATE) = CAST(GETDATE() AS DATE)
    """
)
```

### 4.3 Hebrew Entity Mapping

The training and system prompt must include Hebrew-to-English entity mapping so the AI understands Hebrew questions:

```
Hebrew Term     →  English Entity    →  Table
לידים            →  Leads             →  Leads (in XXX_RapidOne)
תורים            →  Appointments      →  Appointments (in XXX_RapidOne)
מטופלים          →  Patients          →  OCRD (in XXX_Rapid_Common)
חשבוניות         →  Invoices          →  Invoices (in IssuerN)
קבלות            →  Receipts          →  Receipts (in IssuerN)
רופאים           →  Doctors           →  AspNetUsers (in XXX_RapidOne)
טיפולים          →  Treatments        →  Treatments (in XXX_Rapid_Common)
סניפים           →  Branches          →  Branches (in XXX_RapidOne)
מחלקות           →  Departments       →  Departments (in XXX_RapidOne)
הלידים שלי       →  My Leads          →  Leads WHERE AssignedTo = @UserId
התורים שלי       →  My Appointments   →  Appointments WHERE DoctorId IN (linked schedules)
```

SQL is ALWAYS generated in English regardless of the question language.
The response text should be in the same language as the question.

### 4.4 Training is Done Once Per Schema Version

Since all customers share the same database schema (just different data and prefix), training is done ONCE. The trained model works for every customer. Only the database prefix changes per deployment (handled by config, not retraining).

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
```

Now we have ALL 5 permission layers:

```
Layer 1 (Roles):              ["DoctorWithCalendar"]           ← from .NET
Layer 2 (UserPermissions):    ["Schedule - Open the..."]       ← from .NET (combined)
Layer 3 (ProfilePermissions): (merged into Layer 2 above)      ← from .NET (combined)
Layer 4 (ScheduleAccess):     ["doctor-guid-1", "doctor-guid-2"] ← from DB
Layer 5 (IssuerAccess):       ["Issuer1", "Issuer2"]            ← from DB
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

Example for a DoctorWithCalendar user who has schedule permission + 2 linked schedules + 2 issuers:

```
Allowed databases for this user:
    ✅ Keshevrav_RapidOne       (always — main DB)
    ✅ Keshevrav_Rapid_Common   (always — patient data)
    ✅ Issuer1                  (from Layer 5)
    ✅ Issuer2                  (from Layer 5)
    ❌ Issuer3                  (user has no access — NOT included)

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

AVAILABLE DATABASES:
- Keshevrav_RapidOne (main app — appointments, branches, departments)
- Keshevrav_Rapid_Common (SAP — patients OCRD table, treatments)
- Issuer1 (financial — invoices, receipts)
- Issuer2 (financial — invoices, receipts)

MANDATORY RULES — YOU MUST FOLLOW THESE:
1. ALL table references MUST use fully-qualified names: DatabaseName.dbo.TableName
2. Appointment queries MUST include: WHERE DoctorId IN ('doctor-guid-1', 'doctor-guid-2')
3. Invoice/Receipt queries MUST only reference Issuer1 or Issuer2 (no other issuer databases)
4. ONLY generate SELECT statements. Never INSERT, UPDATE, DELETE, DROP, or any DDL.
5. If the user asks about Leads — respond that they don't have access to lead data.

SCHEMA:
[User-specific DDL from Step 5 — only allowed tables]

HEBREW ENTITY MAPPING:
תורים = Appointments, מטופלים = Patients (OCRD table), חשבוניות = Invoices, רופאים = Doctors (AspNetUsers)

RESPONSE LANGUAGE:
- SQL is always in English
- Explanation text should be in Hebrew (user's language is he-IL)
```

### Step 7 — Vanna Generates SQL

With the restricted schema and security rules baked into the system prompt, Vanna generates SQL:

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
    FROM Keshevrav_RapidOne.dbo.Appointments a
    JOIN Keshevrav_Rapid_Common.dbo.OCRD c ON a.PatientId = c.CardCode
    JOIN Keshevrav_RapidOne.dbo.AspNetUsers u ON a.DoctorId = u.Id
    JOIN Keshevrav_RapidOne.dbo.Departments d ON a.DepartmentId = d.DepartmentId
    WHERE CAST(a.Date AS DATE) = CAST(GETDATE() AS DATE)
    AND a.DoctorId IN ('doctor-guid-1', 'doctor-guid-2')
```

Notice: The mandatory filter `DoctorId IN (...)` is included because the system prompt instructed the LLM to always include it. This is PRE-HOC — the security is in the prompt, not injected after.

### Step 8 — SQL Validation (5 Security Checks)

Before executing, validate the generated SQL. Even though the prompt restricts the LLM, we validate as a safety net:

```
Check 1: Is it SELECT only?
    → Scan for INSERT, UPDATE, DELETE, DROP, TRUNCATE, ALTER, CREATE
    → If found → BLOCKED

Check 2: No dangerous keywords?
    → Scan for EXEC, sp_, xp_, GRANT, REVOKE, OPENROWSET, OPENDATASOURCE
    → If found → BLOCKED

Check 3: All referenced databases in allowed list?
    → Parse SQL → extract database names
    → Check against: ["Keshevrav_RapidOne", "Keshevrav_Rapid_Common", "Issuer1", "Issuer2"]
    → If any database not in list → BLOCKED

Check 4: All referenced tables in allowed list?
    → Parse SQL → extract table names
    → Check against user's allowed tables from Step 5
    → If any table not allowed → BLOCKED

Check 5: Mandatory row-level filters present?
    → For Appointments: check DoctorId IN (...) is in WHERE clause
    → For Leads (if somehow generated): check AssignedTo = @UserId
    → For Invoices: check database is in allowed issuers
    → If mandatory filter missing → BLOCKED

All 5 pass → proceed to execution
Any fail → return error ("Query blocked: security validation failed")
```

### Step 9 — Execute SQL

```python
# Build allowed database list for this user
allowed_dbs = [
    "Keshevrav_RapidOne",           # always
    "Keshevrav_Rapid_Common",       # always
] + allowed_issuers                  # from Layer 5: ["Issuer1", "Issuer2"]

# Execute with timeout and row limit
connection = get_connection(database="Keshevrav_RapidOne")  # primary connection
result = connection.execute(validated_sql, timeout=30)       # 30-second timeout
rows = result.fetchmany(1000)                                # max 1000 rows
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

A user may have access to multiple issuer databases. Here's how different scenarios are handled:

### Scenario 1: User asks about one specific issuer

```
Question: "show invoices from Issuer1"
→ Generate SQL using only Issuer1.dbo.Invoices
→ Validate Issuer1 is in user's allowed list
→ Execute
```

### Scenario 2: User asks "show all invoices" (multiple issuers)

```
User has access to: ["Issuer1", "Issuer2"]

→ Generate SQL using UNION across all permitted issuers:

SELECT InvoiceId, Amount, InvoiceDate, 'Issuer1' AS Source
FROM Issuer1.dbo.Invoices
WHERE InvoiceDate >= '2026-02-01'
UNION ALL
SELECT InvoiceId, Amount, InvoiceDate, 'Issuer2' AS Source
FROM Issuer2.dbo.Invoices
WHERE InvoiceDate >= '2026-02-01'
```

The system prompt tells the LLM which issuer databases are available and how to UNION them.

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
→
SELECT i.InvoiceId, i.Amount, b.BranchName
FROM Issuer1.dbo.Invoices i
JOIN Keshevrav_RapidOne.dbo.Branches b ON i.BranchId = b.BranchId
```

This works because all databases are on the same SQL Server instance.

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
SELECT ScheduleId FROM LinkUsersToSchedules WHERE UserId = @user_id
```

- ScheduleId = DoctorId (a doctor's "schedule" IS their user ID)
- Used to filter Appointments: `WHERE DoctorId IN (linked schedule IDs)`
- If empty → no appointment data visible

### Layer 5: Issuer Access (from DB → `LinkIssuersToUsers`)

```sql
SELECT IssuerId FROM LinkIssuersToUsers WHERE UserId = @user_id
```

- IssuerId IS the database name directly (e.g., "Issuer1" = database named "Issuer1")
- Used to determine which financial databases the user can query
- If empty → no invoice/receipt data visible

---

## Part 8: Database Whitelist — What Can Be Queried

For every request, build a whitelist of allowed databases:

```python
# Always allowed
allowed_dbs = [
    f"{DB_PREFIX}_RapidOne",        # e.g., "Keshevrav_RapidOne"
    f"{DB_PREFIX}_Rapid_Common",    # e.g., "Keshevrav_Rapid_Common"
]

# Add user's permitted issuers (from Layer 5)
allowed_dbs += user_allowed_issuers   # e.g., ["Issuer1", "Issuer2"]

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
| Step 7 (SQL Generation) | Vanna/OpenAI fails | "Failed to generate query. Please try again." |
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
│       Includes fully-qualified table names
│
├── SystemPromptBuilder
│       Constructs the LLM system prompt with:
│       - User context (name, role, department)
│       - Restricted schema DDL
│       - Mandatory row filters
│       - Security rules
│       - Hebrew entity mapping
│       - Available databases
│
├── VannaManager
│       Manages the Vanna 2.0 Agent instance
│       Sends question to Vanna with system prompt
│       Receives generated SQL
│
├── SQLValidator
│       5 security checks:
│       1. SELECT-only
│       2. No dangerous keywords
│       3. Allowed databases only
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
DB_PREFIX=Keshevrav                           # customer prefix
DB_USERNAME=sa                                # SQL login
DB_PASSWORD=xxxxx                             # SQL password

# OpenAI
OPENAI_API_KEY=sk-xxxxx                       # for Vanna's LLM
OPENAI_MODEL=gpt-4o-mini                      # model for SQL generation

# Limits
QUERY_TIMEOUT=30                              # seconds
MAX_ROWS=1000                                 # max rows returned
```

Same code. Same trained model. Only config changes per customer.

---

## Part 12: What Each Person Needs To Do

### Us (Backend — FastAPI + Vanna):
1. Build FastAPI server with all components from Part 10
2. Train Vanna with DDL + metadata + sample pairs (Part 4)
3. Implement entity detection (Part 5, Step 3)
4. Implement 5-layer permission checking (Part 5, Step 4)
5. Implement user-specific schema building (Part 5, Step 5)
6. Implement system prompt builder with Hebrew mapping (Part 5, Step 6)
7. Implement SQL validation — 5 security checks (Part 5, Step 8)
8. Implement database connection management with lazy issuer connections
9. Implement response formatting

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
4. SQL validation is just a safety net, not the primary security
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
Question: "show me today's appointments"

Step 3 → Entity: APPOINTMENTS
Step 4 → Has "Schedule - Open the schedule"? YES
         Has linked schedules? YES → ["doctor-guid-1", "doctor-guid-2"]
         → ALLOWED

Step 5 → Schema includes: Appointments, OCRD, Departments, Branches, AspNetUsers
         Schema excludes: Leads (no CC role), Invoices (not requested)

Step 6 → Prompt includes: "Appointment queries MUST include: WHERE DoctorId IN ('doctor-guid-1', 'doctor-guid-2')"

Step 7 → Vanna generates:
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

### Scenario B: Call center agent asks about leads

```
User: Sarah (CallCenterAgent role)
Question: "הראה לי את הלידים שלי" (Show me my leads)

Step 3 → Language: Hebrew → Entity: LEADS (הלידים = Leads, שלי = my)
Step 4 → Has CallCenterAgent role? YES
         → ALLOWED (filter: WHERE AssignedTo = 'sarah-guid' AND IsDeleted = 0)

Step 5 → Schema includes: Leads, Branches
         Schema excludes: Appointments (no schedule permission), Invoices

Step 6 → Prompt includes: "Lead queries MUST include: WHERE AssignedTo = 'sarah-guid' AND IsDeleted = 0"

Step 7 → Vanna generates:
    SELECT l.LeadId, l.FirstName, l.LastName, l.Phone, b.BranchName
    FROM Keshevrav_RapidOne.dbo.Leads l
    JOIN Keshevrav_RapidOne.dbo.Branches b ON l.Branch = b.BranchId
    WHERE l.AssignedTo = 'sarah-guid'
    AND l.IsDeleted = 0

Step 8 → All 5 checks pass
Step 9 → Execute → 12 rows returned
Step 10 → Return JSON (response text in Hebrew since question was Hebrew)
```

### Scenario C: Receptionist asks about leads (BLOCKED)

```
User: Rachel (Receptionist role, no CC role)
Question: "show me all leads"

Step 3 → Entity: LEADS
Step 4 → Has CallCenterAgent role? NO
         Has CallCenterManager role? NO
         → BLOCKED

Response: "You don't have access to lead data. Lead access requires a Call Center role."
No SQL generated. No database query executed.
```

### Scenario D: User asks about invoices across multiple issuers

```
User: Finance Manager (has issuers: Issuer1, Issuer2, Issuer3)
Question: "show total invoices this month by issuer"

Step 3 → Entity: INVOICES
Step 4 → Has linked issuers? YES → ["Issuer1", "Issuer2", "Issuer3"]
         → ALLOWED

Step 5 → Schema includes: Invoices from Issuer1, Issuer2, Issuer3
Step 6 → Prompt includes: "Available issuer databases: Issuer1, Issuer2, Issuer3"

Step 7 → Vanna generates:
    SELECT 'Issuer1' AS IssuerName, COUNT(*) AS InvoiceCount, SUM(Amount) AS Total
    FROM Issuer1.dbo.Invoices WHERE MONTH(InvoiceDate) = MONTH(GETDATE())
    UNION ALL
    SELECT 'Issuer2', COUNT(*), SUM(Amount)
    FROM Issuer2.dbo.Invoices WHERE MONTH(InvoiceDate) = MONTH(GETDATE())
    UNION ALL
    SELECT 'Issuer3', COUNT(*), SUM(Amount)
    FROM Issuer3.dbo.Invoices WHERE MONTH(InvoiceDate) = MONTH(GETDATE())

Step 8 → All databases in allowed list? YES → Pass
Step 9 → Execute → 3 rows returned
Step 10 → Return JSON
```

---

## Summary

This document covers the complete Vanna AI flow from the moment FastAPI receives the .NET-enriched request to the moment the response is returned.

**Key principles:**
1. **PRE-HOC security** — restrict what the LLM sees before SQL generation, don't fix SQL after
2. **5-layer permission enforcement** — 3 layers from .NET, 2 layers from DB
3. **Entity detection first** — determine what the question is about, then check if the user can access it
4. **User-specific schema** — each request gets a DDL with only the tables the user can access
5. **Fully-qualified table names** — enables cross-database JOINs on the same SQL Server instance
6. **SQL validation as safety net** — 5 checks after generation as a fallback, not the primary defense
7. **One Vanna instance per customer** — trained once, works for all users with different permission contexts
8. **Hebrew/English bilingual** — entity mapping in training and system prompt, response in question's language

**Related documents:**
- `AUTH_FLOW.md` — covers everything before this flow (login, token, .NET proxy, X-API-KEY)
- `PROJECT_CONTEXT.md` — full project context, database schema, permission system details
- `multi server approach.txt` — how multiple customers run on multiple servers

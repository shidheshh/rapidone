[AUTH_FLOW.md](https://github.com/user-attachments/files/25443533/AUTH_FLOW.md)
# RapidOne AI Chatbox — Authentication Flow (Detailed)

> This document explains the complete authentication flow for the AI chatbox integration.
> Uses a two-layer security model: Bearer token (Angular → .NET) + X-API-KEY (.NET → FastAPI).
> Angular never talks to FastAPI directly. .NET validates the user and proxies the request.

---

## Part 1: What Already Happens When A User Logs In

**File: `RapidOne.UI/src/app/components/auth/auth.service.js`, line 490-517**

When a user enters their username and password on the login screen, the Angular app calls the `getToken()` method:

```javascript
getToken(username, password) {
    const params = "username=" + username + "&password=" + password + "&grant_type=password";
    const request = {
        method: 'POST',
        url: `${self.apiBaseUri}/Token`,       // → POST /api/Token
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
        data: params
    };
    return self.$http(request).then(res => {
        let authData = { token: res.access_token };
        self.localStorage.set('authData', authData);   // saved in localStorage
        return res;
    });
}
```

What happens here:
- Angular sends username + password to the .NET backend at `POST /api/Token`
- .NET validates credentials against `AspNetUsers` table (checks password, lockout, IP, license, etc.)
- .NET returns an **OWIN Bearer token** in the `access_token` field
- Angular saves this token in **localStorage** under the key `authData` as `{ token: "the-token-value" }`
- This token is valid for **14 days** (configured in `Startup.Auth.cs`, line 42)

---

## Part 2: What Happens Right After Login — UserInfo Is Loaded

**File: `RapidOne.UI/src/app/components/auth/auth.service.js`, line 379-430**

Immediately after login, Angular calls `getUserInfo()`:

```javascript
getUserInfo(updateExternalUser) {
    let url = `${self.apiBaseUri}${self.endPoints.user.getUserInfo}`;
    return self.$http.get(url).then(res => {
        let authData = self.localStorage.get('authData');
        authData.userInfo = res;                           // user profile added to authData
        self.localStorage.set('authData', authData);       // saved back to localStorage
    });
}
```

This calls `GET /api/Account/UserInfo` and saves the full user profile into `authData.userInfo` in localStorage. After this, localStorage contains:

```json
{
    "token": "the-owin-bearer-token",
    "userInfo": {
        "id": "user-guid-123",
        "name": "Dr. Cohen",
        "userName": "dr.cohen",
        "roles": ["DoctorWithCalendar"],
        "permissions": ["Schedule - Open the schedule", "..."],
        "currentDepartmentId": 5,
        "currentBranchId": 2,
        "branchName": "Tel Aviv Branch",
        "departmentName": "Dermatology",
        "defaultIssuer": "Issuer1",
        "mainCompanyDb": "Keshevrav_RapidOne",
        "preferredLanguage": "he-IL"
    }
}
```

This data is already sitting in the browser's localStorage for every logged-in user.

---

## Part 3: How The Token Gets Attached To Every Request Automatically

**File: `RapidOne.UI/src/app/authInterceptor.js`, line 47-62**

The `AuthInterceptor` class intercepts EVERY `$http` request before it goes out:

```javascript
self.request = (config) => {
    // Skip external URLs (Microsoft Graph, Cisco, etc.)
    if (config.url.indexOf('https://graph.microsoft.com/') === -1 && ...) {
        config.headers = config.headers || {};
        let authData = localStorageService.get('authData');     // reads from localStorage
        if (authData && authData.token) {
            config.headers.Authorization = 'Bearer ' + authData.token;  // adds Bearer token
        }
    }
    return config;
};
```

This interceptor is registered in the app module:

**File: `RapidOne.UI/src/app/app.js`, line 309, 313**
```javascript
.service('authInterceptor', AuthInterceptor)
// ...
$httpProvider.interceptors.push('authInterceptor');
```

What this means: **Any `$http` call to `/api/*` automatically gets the `Authorization: Bearer <token>` header added.** The developer writing the chatbox doesn't need to manually add any auth header — it happens by itself.

---

## Part 4: How 401 (Unauthorized) Is Handled Automatically

**File: `RapidOne.UI/src/app/authInterceptor.js`, line 280-321**

If any API returns 401, the interceptor handles it:

```javascript
} else if (rejection.status === 401) {
    const authSrv = $injector.get('authSrv');
    // ... checks for two-factor auth scenarios ...
    authSrv.logout();    // clears localStorage, redirects to login page
}
```

This means if the token expires or becomes invalid, the user is automatically logged out and redirected to the login page. Our FastAPI doesn't need to handle this — Angular already does it.

---

## Part 5: What The .NET UserInfo Endpoint Actually Does (Server Side)

**File: `RapidOne.Api/Controllers/AccountController.cs`, line 60-205**

This is the endpoint our FastAPI will call to validate the token. Here's what it does step by step:

**Line 38:** The entire controller is protected with `[RapidOneAuthorize]` — this means .NET's OWIN middleware validates the Bearer token BEFORE the code even runs. If the token is invalid or expired, it returns 401 immediately.

**Line 86:** Gets the user from the database:
```csharp
dbUser = await UserManager.FindByIdAsync(User.Identity.GetUserId());
```

**Line 103:** Loads the user's roles (Layer 1):
```csharp
var roles = await UserManager.GetRolesAsync(dbUser.Id);
```

**Lines 128-155:** Builds the response with all the fields:
```csharp
var result = new UserInfoViewModel {
    Id = dbUser.Id,                                          // user_id
    Name = dbUser.Name,                                      // display name
    Roles = roles,                                           // Layer 1: roles
    CurrentDepartmentId = currentDepartmentId,               // current department
    DefaultIssuer = dbUser.DefaultIssuer,                    // default issuer DB
    MainCompanyDb = ConstantsHelper.DefaultCompanyDB,        // main database name
    PreferredLanguage = dbUser.Culture,                      // language (Hebrew/English)
    // ... many more fields
};
```

**Lines 183-194:** Loads permissions — this is important. It combines **both Layer 2 AND Layer 3** into one `Permissions` array:
```csharp
// Layer 3: Profile permissions
var profiles = await RapidOneUnitOfWork.LinkProfileToUserRepository
    .GetUserProfilesListAsync(dbUser.Id, result.CurrentDepartmentId);

// Layer 2: Direct user permissions
var permissions = await RapidOneUnitOfWork.UserPermissionsRepository
    .GetListOfUserPermissionsNamesByDepartmentAsync(dbUser.Id, result.CurrentDepartmentId);

if (profiles.Count == 0) {
    result.Permissions = permissions;                        // only Layer 2
} else {
    var profilesIds = profiles.Select(d => d.ProfileId).ToList();
    var profilePermissions = await RapidOneUnitOfWork.ProfilesRepository
        .GetPermissionsForProfilesAsync(profilesIds);
    result.Permissions = profilePermissions
        .Concat(permissions).Distinct().ToArray();           // Layer 2 + Layer 3 combined
}
```

**This is a big finding.** The UserInfo endpoint already gives us 3 out of 5 permission layers in ONE call:
- **Layer 1:** `Roles` field
- **Layer 2:** Direct permissions (in `Permissions` field)
- **Layer 3:** Profile permissions (merged into same `Permissions` field)

We only need to additionally query from the database:
- **Layer 4:** `LinkUsersToSchedules` (which schedules/doctors the user can see)
- **Layer 5:** `LinkIssuersToUsers` (which issuer databases the user can access)

---

## Part 6: The Complete Flow For Our AI Chatbox

Now putting it all together. This is exactly what happens when a user uses the AI chatbox.

The flow has **two security layers**:
- **Layer A (Bearer Token):** Angular → .NET. Proves the user is authenticated.
- **Layer B (X-API-KEY):** .NET → FastAPI. Proves the request came from .NET (not a random caller). The X-API-KEY is a shared secret stored in .NET's `web.config` and FastAPI's `.env` — it never reaches the browser.

### Step 1 — User is already logged in.

Token already exists in localStorage. UserInfo already loaded. Nothing new happens here.

### Step 2 — User clicks "AI" button.

Angular opens the chatbox component. No API call yet. No auth action needed.

### Step 3 — User types a question and sends it.

The user only types the query. Nothing else. The chatbox controller makes a normal `$http` call:

```javascript
this.$http.post('/api/ai/chat', { query: userMessage })
```

The `authInterceptor.js` (line 57-58) automatically adds the Bearer token header. So the actual HTTP request that goes out is:

```
POST /api/ai/chat
Headers:
    Authorization: Bearer eyJ0eXAiOiJKV1Q...  (added automatically by authInterceptor.js)
    Content-Type: application/json
Body:
    { "query": "show me today's appointments" }
```

The frontend developer writes ZERO auth code. Just a normal `$http.post()` with only the query in the body.

### Step 4 — .NET receives the request and validates the user.

The .NET endpoint for `/api/ai/chat` is protected with `[RapidOneAuthorize]`. The OWIN middleware validates the Bearer token BEFORE the code runs.

- If the token is **invalid or expired** → .NET returns **401** immediately → Angular's `authInterceptor.js` (line 280) catches the 401 → calls `authSrv.logout()` → user redirected to login page. Fully automatic. FastAPI is never called.
- If the token is **valid** → .NET knows who the user is → continues to Step 5.

### Step 5 — .NET loads full user info using getUserInfo() logic.

.NET calls the same `getUserInfo()` logic internally (same as `AccountController.GetUserInfo`, line 60-205). This loads all user data from the database:

- **Line 86:** Gets the user from AspNetUsers: `UserManager.FindByIdAsync(User.Identity.GetUserId())`
- **Line 103:** Loads roles (Layer 1): `UserManager.GetRolesAsync(dbUser.Id)`
- **Lines 183-194:** Loads permissions — combines Layer 2 (UserPermissions) AND Layer 3 (ProfilePermissions) into one array

After this step, .NET has:
```
Id                  = "user-guid-123"
Name                = "Dr. Cohen"
UserName            = "dr.cohen"
Roles               = ["DoctorWithCalendar"]                           (Layer 1)
Permissions         = ["Schedule - Open the schedule", "..."]          (Layer 2 + 3 combined)
CurrentDepartmentId = 5
CurrentBranchId     = 2
DefaultIssuer       = "Issuer1"
MainCompanyDb       = "Keshevrav_RapidOne"
PreferredLanguage   = "he-IL"
```

### Step 6 — .NET sends the enriched request to FastAPI.

.NET constructs a NEW request to FastAPI. It takes the user's query (from Step 3) and combines it with all the user info it loaded (from Step 5). It also adds the `X-API-KEY` header from `web.config`.

```
POST http://localhost:8001/chat
Headers:
    X-API-KEY: secret-key-from-web.config        ← server-to-server secret, never in browser
Body:
{
    "query": "show me today's appointments",      ← from the user (Step 3)
    "user_id": "user-guid-123",                   ← from getUserInfo() (Step 5)
    "user_name": "Dr. Cohen",                     ← from getUserInfo() (Step 5)
    "roles": ["DoctorWithCalendar"],              ← Layer 1 from getUserInfo()
    "permissions": ["Schedule - Open the schedule", "..."],  ← Layer 2+3 from getUserInfo()
    "department_id": 5,                           ← from getUserInfo()
    "branch_id": 2,                               ← from getUserInfo()
    "default_issuer": "Issuer1",                  ← from getUserInfo()
    "database": "Keshevrav_RapidOne",             ← from getUserInfo()
    "language": "he-IL"                           ← from getUserInfo()
}
```

The user only typed the query. .NET added everything else behind the scenes.

### Step 7 — FastAPI receives the request and validates the API key.

FastAPI checks the `X-API-KEY` header against its `.env` file:

- **X-API-KEY does NOT match** → return 403 Forbidden immediately. Query is not processed. This blocks any direct call to FastAPI that didn't come through .NET.
- **X-API-KEY matches** → request is trusted, continue to Step 8.

### Step 8 — FastAPI now has everything it needs.

From the request body sent by .NET:

```
user_id         = body.user_id          → "user-guid-123"
name            = body.user_name        → "Dr. Cohen"
roles           = body.roles            → ["DoctorWithCalendar"]       (Layer 1)
permissions     = body.permissions      → ["Schedule - Open...", ...]   (Layer 2 + 3 combined)
department_id   = body.department_id    → 5
branch_id       = body.branch_id        → 2
default_issuer  = body.default_issuer   → "Issuer1"
database_name   = body.database         → "Keshevrav_RapidOne"
language        = body.language         → "he-IL"
```

FastAPI then queries the customer's database for the remaining 2 permission layers:
- **Layer 4:** `SELECT ScheduleId FROM LinkUsersToSchedules WHERE UserId = 'user-guid-123'`
- **Layer 5:** `SELECT IssuerId FROM LinkIssuersToUsers WHERE UserId = 'user-guid-123'`

Now FastAPI has ALL 5 permission layers. It proceeds to Vanna AI for SQL generation, executes the SQL, and returns the full JSON response.

### Step 9 — Response flows back to the user.

```
FastAPI returns JSON response to .NET
    → .NET forwards the response to Angular as-is
        → Angular chatbox displays the results (text message + data table)
```

---

## Part 7: Security Summary — Two Layers

| Layer | Where | What It Proves | How |
|---|---|---|---|
| **Bearer Token** | Angular → .NET | The user is authenticated | OWIN middleware validates automatically. Invalid/expired → 401 → Angular handles logout |
| **X-API-KEY** | .NET → FastAPI | The request came from .NET, not a random caller | Shared secret in `web.config` (server) + `.env` (FastAPI). Never visible in browser |

**What this prevents:**
- Someone discovers FastAPI's port (8001) and calls it directly → **blocked** (no X-API-KEY)
- Someone sends a fake X-API-KEY → **blocked** (doesn't match .env)
- Someone sends a request without logging in → **blocked** (.NET returns 401 before reaching FastAPI)
- Someone forges a user_id in the request body → **not possible** (only .NET can call FastAPI, and .NET gets user_id from the validated token, not from user input)

---

## Part 8: What Each Person Needs To Do

### Frontend Developer:
- Create chatbox component (`ai-chat.component.js`, `ai-chat.controller.js`, `ai-chat.html`, `ai-chat.scss`)
- Use normal `$http.post('/api/ai/chat', { query: "..." })` — auth is automatic via authInterceptor.js
- Handle the JSON response (display text message + data table)
- That's it. No auth code, no token management, no API key handling

### Client .NET Team:
- Create one new .NET controller endpoint: `POST /api/ai/chat`
  - Protected with `[RapidOneAuthorize]` (standard, same as other endpoints)
  - Calls `getUserInfo()` logic to load user data
  - Sends enriched request to FastAPI at `http://localhost:8001/chat` with `X-API-KEY` header
  - Forwards FastAPI's response back to Angular
- Store the `X-API-KEY` value in `web.config`

### Us (Backend — FastAPI):
- Build FastAPI server on port 8001
- Validate `X-API-KEY` header against `.env`
- Read user info from request body (sent by .NET)
- Load Layer 4 + Layer 5 from database
- Integrate with Vanna AI for SQL generation
- Execute SQL and return full JSON response

---

## Part 9: Edge Cases

### Token expired (after 14 days without re-login):
.NET returns 401 → `authInterceptor.js` catches it (line 280) → calls `authSrv.logout()` → user redirected to login page. FastAPI is never called. Fully automatic.

### User logs out:
`authSrv.logout()` clears localStorage (`auth.service.js`, line 539+). Token gone. Next API call has no token → .NET returns 401 → same flow as above.

### User switches department mid-session:
They call `POST /api/Account/changedepartment` (`auth.service.js`, line 476). Their `currentDepartmentId` changes in the .NET session. Next AI chat request → .NET loads fresh getUserInfo() with new department → sends updated data to FastAPI.

### User's permissions changed by admin:
Admin changes permissions in the .NET app. Next AI chat request → .NET loads fresh getUserInfo() → sends updated permissions to FastAPI. No caching delay — always fresh because .NET loads from DB every time.

### Multiple concurrent users:
Each user has their own Bearer token → .NET validates each separately → sends each user's own data to FastAPI → completely isolated.

### Someone tries to call FastAPI directly (bypassing .NET):
No X-API-KEY header → FastAPI returns 403 Forbidden immediately. Query is never processed.

---

## Files Reference

### Existing Files (DO NOT MODIFY):

| File | What It Does |
|---|---|
| `RapidOne.UI/src/app/authInterceptor.js` | Auto-attaches Bearer token to all `$http` requests (line 57-58). Handles 401 → logout (line 280) |
| `RapidOne.UI/src/app/components/auth/auth.service.js` | Login (`getToken`, line 490), UserInfo (`getUserInfo`, line 379), token getter (line 168) |
| `RapidOne.UI/src/app/app.constants.js` | API endpoint constants (`getUserInfo` path at line 121) |
| `RapidOne.UI/src/app/app.js` | Interceptor registration (line 309, 313) |
| `RapidOne.Api/Controllers/AccountController.cs` | `GET /api/Account/UserInfo` endpoint (line 60-205) — used by .NET to load user data before calling FastAPI |
| `RapidOne.Api/App_Start/Startup.Auth.cs` | OWIN OAuth config, 14-day token expiry (line 42) |
| `RapidOne.Api/Providers/ApplicationOAuthProvider.cs` | Token generation on login (`GrantResourceOwnerCredentials`) |

### New Files (WE CREATE):

| File | What It Does |
|---|---|
| `RapidOne.UI/src/app/components/ai-chat/ai-chat.component.js` | Chatbox component definition |
| `RapidOne.UI/src/app/components/ai-chat/ai-chat.controller.js` | Chat logic — sends `$http.post('/api/ai/chat', { query: "..." })`, displays response |
| `RapidOne.UI/src/app/components/ai-chat/ai-chat.html` | Chat UI template |
| `RapidOne.UI/src/app/components/ai-chat/ai-chat.scss` | Chat styling (RTL support for Hebrew) |

### New .NET Endpoint (CLIENT .NET TEAM CREATES):

| What | Details |
|---|---|
| `POST /api/ai/chat` controller | Protected with `[RapidOneAuthorize]`. Loads getUserInfo(), sends enriched request to FastAPI with X-API-KEY header |
| `web.config` | Store the `X-API-KEY` value (shared secret for .NET → FastAPI communication) |

### Config:

| What | Details |
|---|---|
| FastAPI `.env` | Store the same `X-API-KEY` value that matches `web.config` |

---

## FastAPI API Key Validation Code (What We Build)

```python
import os
from fastapi import FastAPI, Request, HTTPException
from pydantic import BaseModel
from typing import List, Optional

app = FastAPI()

# X-API-KEY from .env (must match the value in .NET's web.config)
AI_API_KEY = os.getenv("AI_API_KEY")


class ChatRequest(BaseModel):
    query: str
    user_id: str
    user_name: str
    roles: List[str]
    permissions: List[str]
    department_id: int
    branch_id: int
    default_issuer: Optional[str] = None
    database: str
    language: Optional[str] = "en-US"


@app.post("/chat")
async def chat(request: Request, body: ChatRequest):
    # Security Layer: Validate X-API-KEY
    api_key = request.headers.get("X-API-KEY")
    if api_key != AI_API_KEY:
        raise HTTPException(status_code=403, detail="Forbidden")

    # At this point we trust the request — it came from .NET
    # .NET already validated the Bearer token and loaded user info
    #
    # body.user_id        → "user-guid-123"
    # body.roles          → ["DoctorWithCalendar"]          (Layer 1)
    # body.permissions    → ["Schedule - Open the...", ...] (Layer 2 + 3)
    # body.department_id  → 5
    # body.database       → "Keshevrav_RapidOne"
    #
    # Load remaining permission layers from DB:
    # Layer 4: SELECT ScheduleId FROM LinkUsersToSchedules WHERE UserId = ?
    # Layer 5: SELECT IssuerId FROM LinkIssuersToUsers WHERE UserId = ?
    #
    # Then process query with Vanna AI and return response

    return {"message": "...", "result": [...], "sql_query": "..."}
```

---

## Summary

This approach uses a **two-layer security model**:

1. **Bearer Token (Angular → .NET):** The existing OWIN auth system validates the user. Same login, same token, same `authInterceptor.js`, same 401 handling. Zero changes to the existing auth flow.

2. **X-API-KEY (.NET → FastAPI):** A shared secret between .NET's `web.config` and FastAPI's `.env`. .NET adds this header when calling FastAPI. FastAPI rejects any request without a valid key. The key never reaches the browser — it's server-to-server only.

**.NET acts as the gatekeeper:** Angular never talks to FastAPI directly. .NET validates the user, loads all user info via `getUserInfo()`, enriches the request with user context (roles, permissions, department, database, etc.), and forwards the complete request to FastAPI with the X-API-KEY. The user only types the query — .NET adds everything else behind the scenes.

**What's new:**
- One new .NET controller endpoint (`POST /api/ai/chat`) that proxies to FastAPI
- `X-API-KEY` stored in `web.config` (server) and `.env` (FastAPI)
- FastAPI server on port 8001
- Angular chatbox component

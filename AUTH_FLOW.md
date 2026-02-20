# RapidOne AI Chatbox — Authentication Flow (Detailed)

> This document explains the complete authentication flow for the AI chatbox integration.
> No changes to existing .NET backend, Angular frontend, or auth system are needed.

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

Now putting it all together. This is exactly what happens when a user uses the AI chatbox:

### Step 1 — User is already logged in.

Token already exists in localStorage. UserInfo already loaded. Nothing new happens here.

### Step 2 — User clicks "AI" button.

Angular opens the chatbox component. No API call yet. No auth action needed.

### Step 3 — User types a question and sends it.

The chatbox controller makes a normal `$http` call:

```javascript
this.$http.post('/api/ai/chat', { message: userMessage })
```

The `authInterceptor.js` (line 57-58) automatically adds the Bearer token header. So the actual HTTP request that goes out is:

```
POST /api/ai/chat
Headers:
    Authorization: Bearer eyJ0eXAiOiJKV1Q...  (added automatically)
    Content-Type: application/json
Body:
    { "message": "show me today's appointments" }
```

The frontend developer writes ZERO auth code. Just a normal `$http.post()`.

### Step 4 — IIS receives the request.

IIS has a reverse proxy rule: `/api/ai/*` routes to `http://localhost:8001` (our FastAPI).
The Bearer token passes through in the headers untouched.

### Step 5 — FastAPI receives the request.

FastAPI extracts the `Authorization` header. FastAPI CANNOT decode this token because it's an OWIN encrypted token (not standard JWT). But it doesn't need to decode it.

### Step 6 — FastAPI validates the token by calling .NET.

FastAPI makes a localhost HTTP call:

```
GET http://localhost/api/Account/UserInfo
Headers:
    Authorization: Bearer eyJ0eXAiOiJKV1Q...  (same token, forwarded)
```

This is a localhost call (same server) so it's extremely fast (less than 5ms).

The .NET app's `[RapidOneAuthorize]` middleware validates the token. If valid, `AccountController.GetUserInfo()` (line 60-205) runs and returns the full user profile.

If the token is invalid or expired, .NET returns **401**. FastAPI passes this 401 back to Angular. Angular's `authInterceptor.js` (line 280) catches the 401 and calls `authSrv.logout()` which redirects the user to the login page. Everything handled automatically.

### Step 7 — FastAPI caches the result.

We hash the Bearer token and store the UserInfo response in a 5-minute TTL cache. Why:
- The user might send 10 messages in 5 minutes
- We don't want to call .NET's UserInfo endpoint 10 times
- First message: calls .NET, caches result (~5ms)
- Next 9 messages: cache hit, instant (0ms)
- After 5 minutes: cache expires, next message calls .NET again (picks up any permission changes)

### Step 8 — FastAPI now has everything it needs.

From the UserInfo response:

```
user_id         = response.id                   → "user-guid-123"
name            = response.name                 → "Dr. Cohen"
roles           = response.roles                → ["DoctorWithCalendar"]       (Layer 1)
permissions     = response.permissions          → ["Schedule - Open...", ...]   (Layer 2 + 3 combined)
department_id   = response.currentDepartmentId  → 5
branch_id       = response.currentBranchId      → 2
default_issuer  = response.defaultIssuer        → "Issuer1"
database_name   = response.mainCompanyDb        → "Keshevrav_RapidOne"
language        = response.preferredLanguage    → "he-IL"
```

Then FastAPI queries the customer's database for the remaining 2 layers:
- **Layer 4:** `SELECT ScheduleId FROM LinkUsersToSchedules WHERE UserId = 'user-guid-123'`
- **Layer 5:** `SELECT IssuerId FROM LinkIssuersToUsers WHERE UserId = 'user-guid-123'`

Now FastAPI has ALL 5 permission layers. It proceeds to Vanna AI for SQL generation, execution, and response.

---

## Part 7: What Each Person Needs To Do

### Frontend Developer:
- Create chatbox component (`ai-chat.component.js`, `ai-chat.controller.js`, `ai-chat.html`, `ai-chat.scss`)
- Use normal `$http.post('/api/ai/chat', { message: "..." })` — auth is automatic
- For SSE streaming: read token from `localStorageService.get('authData').token` and add it to `fetch()` headers manually
- Handle the response (display text, render data table)
- That's it. No auth code, no token management, no new services

### Client Infra Team:
- Add one IIS reverse proxy rule: `/api/ai/*` → `http://localhost:8001`
- That's it. No .NET code changes, no new endpoints, no new tokens

### Us (Backend):
- Build FastAPI server on port 8001
- Implement token validation by calling `GET http://localhost/api/Account/UserInfo` with the forwarded Bearer token
- Cache the result (5-minute TTL)
- Load Layer 4 + Layer 5 from database
- Integrate with Vanna AI for SQL generation
- Return response via SSE streaming

---

## Part 8: Edge Cases

### Token expired (after 14 days without re-login):
.NET returns 401 → FastAPI returns 401 → `authInterceptor.js` catches it (line 280) → calls `authSrv.logout()` → user redirected to login page. Fully automatic.

### User logs out:
`authSrv.logout()` clears localStorage (`auth.service.js`, line 539+). Token gone. Next API call has no token → .NET returns 401 → same flow as above.

### User switches department mid-session:
They call `POST /api/Account/changedepartment` (`auth.service.js`, line 476). Their `currentDepartmentId` changes. Our cache expires in 5 minutes max → next UserInfo call picks up the new department + new permissions for that department.

### User's permissions changed by admin:
Admin changes permissions in the .NET app. Our cache expires in 5 minutes → FastAPI calls UserInfo again → gets updated permissions. No manual refresh needed.

### Multiple concurrent users:
Each user has their own Bearer token → each gets their own cache entry → completely isolated.

---

## Files Reference

### Existing Files (DO NOT MODIFY):

| File | What It Does |
|---|---|
| `RapidOne.UI/src/app/authInterceptor.js` | Auto-attaches Bearer token to all `$http` requests (line 57-58) |
| `RapidOne.UI/src/app/components/auth/auth.service.js` | Login (`getToken`, line 490), UserInfo (`getUserInfo`, line 379), token getter (line 168) |
| `RapidOne.UI/src/app/app.constants.js` | API endpoint constants (`getUserInfo` path at line 121) |
| `RapidOne.UI/src/app/app.js` | Interceptor registration (line 309, 313) |
| `RapidOne.Api/Controllers/AccountController.cs` | `GET /api/Account/UserInfo` endpoint (line 60-205) |
| `RapidOne.Api/App_Start/Startup.Auth.cs` | OWIN OAuth config, 14-day token expiry (line 42) |
| `RapidOne.Api/Providers/ApplicationOAuthProvider.cs` | Token generation on login (`GrantResourceOwnerCredentials`) |

### New Files (WE CREATE):

| File | What It Does |
|---|---|
| `RapidOne.UI/src/app/components/ai-chat/ai-chat.component.js` | Chatbox component definition |
| `RapidOne.UI/src/app/components/ai-chat/ai-chat.controller.js` | Chat logic, send message, display response |
| `RapidOne.UI/src/app/components/ai-chat/ai-chat.html` | Chat UI template |
| `RapidOne.UI/src/app/components/ai-chat/ai-chat.scss` | Chat styling (RTL support for Hebrew) |

### Config (CLIENT INFRA TEAM):

| What | Details |
|---|---|
| IIS reverse proxy rule | `/api/ai/*` → `http://localhost:8001` |

---

## FastAPI Token Validation Code (What We Build)

```python
import httpx
from cachetools import TTLCache
from fastapi import HTTPException

# Cache: token_hash → user_info, 5-minute TTL
user_cache = TTLCache(maxsize=1000, ttl=300)

async def validate_and_get_user(authorization_header: str):
    """
    Validates the Bearer token by calling .NET's existing UserInfo endpoint.
    Caches the result for 5 minutes to avoid repeated calls.
    """
    # Check cache first
    cache_key = hash(authorization_header)
    if cache_key in user_cache:
        return user_cache[cache_key]

    # Call .NET's existing endpoint (localhost, same server)
    async with httpx.AsyncClient() as client:
        response = await client.get(
            "http://localhost/api/Account/UserInfo",
            headers={"Authorization": authorization_header}
        )

    if response.status_code == 401:
        raise HTTPException(status_code=401, detail="Invalid or expired token")

    if response.status_code != 200:
        raise HTTPException(status_code=502, detail="Auth service unavailable")

    user_info = response.json()
    user_cache[cache_key] = user_info    # cache it
    return user_info
```

---

## Summary

This approach requires **zero changes** to the existing authentication system. We use:
- The **same OWIN Bearer token** that users already have from login
- The **same `authInterceptor.js`** that already attaches tokens to all API requests
- The **same `GET /api/Account/UserInfo`** endpoint that already returns full user profiles with roles and permissions
- The **same 401 handling** that already redirects expired sessions to the login page

The only new thing is one IIS reverse proxy rule to route `/api/ai/*` requests to our FastAPI server.

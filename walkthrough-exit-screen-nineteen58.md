# Exit Screen Walkthrough — Nineteen58 Meeting

**Date:** Monday, 13 July 2026 | 4:00 PM – 5:00 PM (UTC+8)  
**Presenter:** Nathaniel Alcantara (Senior Full Stack Developer, OEPE Team)  
**Organizer:** Carrie Ann Singh (Head of Partner Enablement)  
**Audience:** Nineteen58 team (third-party AI analytics provider)

### Attendees

| Name | Role |
|------|------|
| Carrie Ann Singh | Head of Partner Enablement (EE) |
| kieran@nineteen58.co.za | Nineteen58 |
| James MacRobert | Nineteen58 |
| Eimee Monica Solis | OEPE Team (EE) |
| Bea Jewel Vines | OEPE Team (EE) |
| Kim Geraldine Fabe (KG) | OEPE Team (EE) — built the original modal & API |
| Dan Kenneth Coloma | OEPE Team (EE) |
| Nathaniel Alcantara | OEPE Team (EE) — exit modal redesign |

---

## Agenda at a Glance

| # | Section | Duration | Key Action |
|---|---------|----------|------------|
| 1 | Opening | 1–2 min | Set the scene |
| 2 | The Problem (why we redesigned) | 2 min | Show old 4 reasons vs new 8 |
| 3 | What Changed (the redesign) | 5–7 min | Walk through reasons, contact toggle, X-close |
| 4 | Data Model (`ExitRegistration` table) | 3 min | Show columns, what's in API vs not |
| 5 | Your Existing API (OEPE-2165) | 3–4 min | **Critical** — acknowledge KG's API, discuss gaps |
| 6 | Technical Architecture | 2–3 min | Write path + read path flow |
| 7 | Test Coverage | 1 min | 30 unit tests |
| 8 | Value for Nineteen58 | 2 min | What's immediate vs coming soon |
| 9 | Action Items & Discussion | 3–5 min | PhoneNumber, UserDeviceId, production sign-off |
| 10 | Closing | 1 min | Summarize & final Qs |

---

## Section 1 — Opening (1–2 min)

> "Hi everyone, thanks for joining. I'm Nathaniel from the OEPE team."
>
> "Today I'll walk you through the changes we've made to the registration exit screen on our Identity Server."
>
> "I know your team is already consuming data via the exit feedback API that KG set up earlier — what we've done is significantly enrich the data that flows into it."
>
> "I'll cover what changed in the modal, what new data fields you'll see, and what updates are still needed on the API side."

---

## Section 2 — The Problem (2 min)

**Context:** When a user is on the registration page and clicks **"Cancel Registration"**, a modal pops up asking why they're leaving. The response is saved to the `ExitRegistration` SQL table and exposed to Nineteen58 via the exit feedback API.

### What Was Wrong

- Only **4 generic reasons** — too broad for actionable insight
- No way for users to request follow-up contact
- No phone number capture
- Email fields were coded but **never enabled** (commented out in HTML)
- The data Nineteen58 was receiving reflected these limitations

### Old Exit Reasons (OEPE-1820)

| # | Reason |
|---|--------|
| 1 | I need more information before I sign up |
| 2 | I don't have time right now |
| 3 | I'm not interested in opening an account |
| 4 | There was an error or technical issue |
| 5 | Other (free text) |

---

## Section 3 — What Changed (5–7 min)

### 3A. 8 Granular Exit Reasons (was 4)

| Constant | Reason Text | Category |
|----------|-------------|----------|
| ExitReason1 | Just exploring, not interested right now | Casual browser |
| ExitReason2 | I need to know more before I commit | Information gap |
| ExitReason3 | I don't have time right now | Time constraint |
| ExitReason4 | I want to login not register | Wrong flow — existing user |
| ExitReason5 | I already have an account | Potential duplicate |
| ExitReason6 | I'm stuck on choosing a username | UX friction |
| ExitReason7 | I'm stuck on creating a password | UX friction |
| ExitReasonOther | Other | Free-text input |

> **Talking point:** "Reasons 6 and 7 are particularly interesting for your analysis — they pinpoint exactly WHERE in the form the user got stuck."

> **Talking point:** "These are defined as constants in the C# backend (`Constants.ExitReason1` through `ExitReason7` + `ExitReasonOther`), so frontend and database stay consistent."

### 3B. Conditional Self-Service Messages

Two reasons trigger helpful in-context messages:

| Reason | Message Shown | Effect |
|--------|---------------|--------|
| "I want to login not register" | "Already one of us? Good news — your account is waiting." + login link | Redirects user to login |
| "I already have an account" | Links to recover username or reset password | Reduces unnecessary drop-offs |

> **Talking point:** "This should actually REDUCE some of your drop-off counts — because these users are being redirected to the right flow instead of just leaving."

### 3C. Contact Me Toggle + Email & Phone

New section: **"Would you like us to get in touch with you?"** with a toggle switch.

When toggled ON, two fields appear:

| Field | Details |
|-------|---------|
| **Email** | Pre-populated from the registration form if user already typed one |
| **Phone** | Hardcoded `+27` (South Africa) prefix, expects 9 digits without leading zero |

**Validation rules:**

| When | Behavior |
|------|----------|
| On blur (email) | Regex: `^([a-zA-Z0-9_'\-.]+)@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.)|(([a-zA-Z0-9\-]+\.)+))([a-zA-Z]{2,63}|[0-9]{1,3})$` |
| On blur (phone) | Regex: `^[1-9]\d{8}$` — 9 digits, no leading zero |
| On blur (empty) | No error shown (user hasn't typed anything yet) |
| On submit | If contact toggle ON, at least one of email or phone is required |
| Visual feedback | Green left border = valid, Red = invalid |

> **Talking point:** "Users who opt in are warm leads — this gives you a re-engagement signal."

### 3D. X-Close Button Behavior (**IMPORTANT**)

This is a critical behavioral detail:

| Scenario | What Gets Saved |
|----------|----------------|
| User selected a reason, then clicks X | **Saves the selected reason** (NOT "Closed without feedback") |
| User entered contact info, then clicks X | **Saves the contact data** alongside the reason |
| User selected nothing, clicks X | Saves `ExitReason = "Closed without feedback"` |
| Contact validation fails on X-close | Modal stays open (doesn't close) |

> **Talking point:** "This means you'll get richer data even from users who don't explicitly click 'Send Feedback'. You'll see fewer 'Closed without feedback' entries and more specific reasons."

---

## Section 4 — Data Model (3 min)

**Table:** `ExitRegistration` (same table the API already reads from)

| Column | Type | In API? | Notes |
|--------|------|---------|-------|
| ExitReasonId | bigint (PK, auto-incr) | **Yes** | Internal record identifier |
| UserDeviceId | string | **No** | From DeviceId cookie or auto-generated GUID. In DB since OEPE-1820, never exposed |
| ExitReason | string | **Yes** | Selected reason text, or "Closed without feedback". NOW 8 values instead of 4 |
| ExitReasonOther | string (nullable) | **Yes** | Free-text, only when user selects "Other" |
| ReferralPartner | string (nullable) | **Yes** | `productid` from landing URL (e.g., "capitec", "easyequities") |
| EmailAddress | string (nullable) | **Yes** | Validated email. Sanitized to null if format invalid |
| **PhoneNumber** | **string (nullable)** | **No** | **NEW (OEPE-2263).** SA phone 9-digit. Sanitized to null if invalid. **NOT YET IN API** |
| DateCreatedUTC | datetime | **Yes** | UTC timestamp (ISO 8601) |

### Server-Side Validation Philosophy

> "We never reject a feedback submission due to bad contact data. The exit reason is the primary data; contact info is secondary."

- **Approach:** Sanitization over rejection — invalid email/phone formats are set to `null`, the record is still saved
- Both email and phone regexes are **pre-compiled** (`RegexOptions.Compiled`) for performance

---

## Section 5 — Your Existing API (OEPE-2165 by KG) (**Critical Section**)

> **Talking point:** "I want to acknowledge the API that KG already built for you under OEPE-2165."

### API Overview

| Aspect | Detail |
|--------|--------|
| Endpoint | `GET /api/exit-feedback` |
| Auth | OAuth 2.0 client credentials |
| Client ID | `nineteen58-exit-feedback` |
| Scope | `idp_exit_feedback_endpoint` |
| Token expiry | 1 hour — cache and reuse, do not request per call |
| UAT | `https://uatidentity.openeasy.io/api/exit-feedback` |
| Production | `https://identity.openeasy.io/api/exit-feedback` **(TBC — pending sign-off)** |
| Token endpoint (UAT) | `https://uatidentity.openeasy.io/connect/token` |

### Query Parameters

| Parameter | Format | Default | Notes |
|-----------|--------|---------|-------|
| `fromDate` | YYYY-MM-DD | `toDate` minus 365 days | Inclusive |
| `toDate` | YYYY-MM-DD | Today UTC | Inclusive |

- Case-insensitive parameter names
- Max 1-year lookback (silently capped if exceeded)
- Unrecognized params silently ignored

### Current API Response (what Nineteen58 sees today)

```json
{
  "records": [
    {
      "exitReasonId": 150,
      "exitReason": "I'm stuck on choosing a username",
      "exitReasonOther": null,
      "referralPartner": "easyequities",
      "emailAddress": "user@example.com",
      "dateCreatedUtc": "2026-07-10T14:23:01.5"
    },
    {
      "exitReasonId": 149,
      "exitReason": "Other",
      "exitReasonOther": "The page loaded slowly",
      "referralPartner": "capitec",
      "emailAddress": null,
      "dateCreatedUtc": "2026-07-10T09:45:12.3"
    }
  ],
  "count": 2
}
```

### Proposed API Response (after follow-up update)

```json
{
  "records": [
    {
      "exitReasonId": 150,
      "exitReason": "I'm stuck on choosing a username",
      "exitReasonOther": null,
      "referralPartner": "easyequities",
      "emailAddress": "user@example.com",
      "phoneNumber": "821234567",
      "userDeviceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "dateCreatedUtc": "2026-07-10T14:23:01.5"
    }
  ],
  "count": 1
}
```

> **Note:** `phoneNumber` and `userDeviceId` are proposed additions — pending confirmation from Nineteen58 in this meeting.

### What the Good News Is

> "The new exit reasons will appear automatically in your API responses because they come from the same `ExitRegistration` table. **No changes needed on your side for that.**"

### What Won't Appear Yet

> "The API response model (`ExitFeedbackRecord`) doesn't include `phoneNumber`. Similarly, `userDeviceId` has been in the DB since day one but was never exposed."

> "We have a follow-up action to add these fields. I'd like to confirm with you today: **would phoneNumber and userDeviceId be useful for your models?**"

### API Source Files (for internal reference)

| File | Path |
|------|------|
| Controller | `src/.../Features/Api/Controllers/ExitFeedbackController.cs` |
| Response Model | `src/.../Features/Api/Models/ExitFeedbackResponse.cs` |
| Query Service | `src/.../Features/Api/Services/ExitFeedbackQueryService.cs` |
| Query Interface | `src/.../Features/Api/Services/IExitFeedbackQueryService.cs` |
| API Startup | `src/.../Features/Api/ApiStartupExtensions.cs` |
| Controller Tests | `test/.../Api/Controllers/ExitFeedbackControllerTests.cs` |
| Service Tests | `test/.../Api/Services/ExitFeedbackQueryServiceTests.cs` |

---

## Section 6 — Technical Architecture (2–3 min)

### Write Path (User → Database)

```
1. User is on BasicRegistration.cshtml (sign-up page for non-OTP tenants)
2. User clicks "Cancel Registration"
3. JavaScript opens exit modal, resets state, pre-populates email from form
4. User selects a reason, optionally toggles "contact me", enters email/phone
5. On "Send Feedback" click OR X-close → POST /Registration/SaveFeedback
6. RegistrationController delegates to RegistrationService.SaveExitFeedbackToDb
7. Service validates/sanitizes email and phone with pre-compiled Regex
8. Writes to SQL Server via Entity Framework
   (ExitRegistrationDbContext → ExitRegistration table)
```

### Read Path (Nineteen58 → Database)

```
1. Nineteen58 requests OAuth token:
   POST /connect/token (client_credentials, scope: idp_exit_feedback_endpoint)

2. Nineteen58 calls:
   GET /api/exit-feedback?fromDate=YYYY-MM-DD&toDate=YYYY-MM-DD

3. ExitFeedbackController validates date range
   (max 1-year lookback, silently caps if exceeded)

4. ExitFeedbackQueryService queries ExitRegistration table
   via EF Core (AsNoTracking, ordered by dateCreatedUtc DESC)

5. Returns JSON response with records array and count
```

### End-to-End Flow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                     WRITE PATH                                │
│                                                               │
│  User on Registration Page                                    │
│         │                                                     │
│         ▼ clicks "Cancel Registration"                        │
│  ┌─────────────────────────┐                                  │
│  │    Exit Modal            │                                  │
│  │  - 8 reasons             │                                  │
│  │  - Contact toggle        │                                  │
│  │  - Email + Phone fields  │                                  │
│  └──────────┬──────────────┘                                  │
│             │ Submit OR X-close                                │
│             ▼                                                  │
│  POST /Registration/SaveFeedback                              │
│             │                                                  │
│             ▼                                                  │
│  RegistrationService.SaveExitFeedbackToDb                     │
│  (validates email/phone with pre-compiled Regex)              │
│             │                                                  │
│             ▼                                                  │
│  ┌─────────────────────────┐                                  │
│  │  ExitRegistration Table  │◄─── Same table for both paths   │
│  │  (SQL Server via EF)     │                                  │
│  └──────────┬──────────────┘                                  │
└─────────────┼────────────────────────────────────────────────┘
              │
┌─────────────┼────────────────────────────────────────────────┐
│             │            READ PATH                            │
│             ▼                                                  │
│  GET /api/exit-feedback                                       │
│  (OAuth 2.0 — client: nineteen58-exit-feedback)               │
│             │                                                  │
│             ▼                                                  │
│  ExitFeedbackController → ExitFeedbackQueryService            │
│  (EF Core, AsNoTracking, ordered by dateCreatedUtc DESC)      │
│             │                                                  │
│             ▼                                                  │
│  JSON Response to Nineteen58                                  │
│  { records: [...], count: N }                                 │
└──────────────────────────────────────────────────────────────┘
```

---

## Section 7 — Test Coverage (1 min)

> "30 unit tests covering `SaveExitFeedbackToDb` (the write path), all passing. KG also wrote tests for the API side."

### Exit Modal Unit Tests (30 tests)

| Category | Tests |
|----------|-------|
| **Reason saving** | Standard reason saves with null ExitReasonOther; "Other" saves free-text |
| **Contact saving** | Valid email + phone both saved |
| **X-close** | With reason → saves reason; With "Other" → saves other-text; No reason → saves "Closed without feedback"; With email → saves email; Without email → saves null |
| **DeviceId** | Null → generates new GUID; Provided → used as-is |
| **Sanitization** | Null/empty/whitespace email → null; Null/empty/whitespace phone → null; Invalid email format → null; Invalid phone format → null |
| **ReferralPartner** | URL with `productid` → extracts partner; URL without → null |
| **Persistence** | `SaveChangesAsync` is called; `DateCreatedUTC` ≈ `DateTime.UtcNow` |

### API Tests (KG's OEPE-2165)

- `ExitFeedbackControllerTests` — controller authorization, date validation, response structure
- `ExitFeedbackQueryServiceTests` — query logic, date range filtering, ordering

---

## Section 8 — Value for Nineteen58 (2 min)

### Immediate (no action needed on Nineteen58's side)

- **8 specific drop-off reasons** instead of 4 — the `exitReason` field values change automatically in API responses
- **Friction-point signals** — reasons 6 and 7 ("stuck on username/password") are new categories to detect
- **Better X-close granularity** — if a user selected a reason but closed with X, you now get the actual reason instead of just "Closed without feedback"

### Coming Soon (after API update)

- **`phoneNumber` field** — SA mobile numbers for users who opted in to contact
- **`userDeviceId` field** — device-level identifier for correlating multiple exit events from the same browser

### Heads Up for Nineteen58's Models

- Historical records still have the **old 4 reason strings**. New records from ~July 3 onward have the **8 new strings**
- Parsing/categorization logic may need updating to handle the new reason values
- The "Other" reason behavior is unchanged — `exitReason = "Other"` and `exitReasonOther` contains the free text

---

## Section 9 — Action Items & Discussion (3–5 min)

> "I'd like to discuss a few things before we close."

### Items to Confirm

| # | Item | Detail | Owner | Status |
|---|------|--------|-------|--------|
| 1 | **API update: PhoneNumber** | Add `phoneNumber` to `ExitFeedbackRecord` and `ExitFeedbackQueryService`. Quick change. Do you want this field? | EasyEquities (Nathaniel/KG) | Pending confirmation |
| 2 | **API update: UserDeviceId** | Has been in DB since OEPE-1820, never exposed. Useful for correlating repeat drop-offs from same device. Do you want this? | EasyEquities (Nathaniel/KG) | Pending confirmation |
| 3 | **Nineteen58 model updates** | Parsing logic needs to handle 8 new exit reason strings. Old strings will stop appearing in new records. Do you need a mapping document? | Nineteen58 | Discussion |
| 4 | **Production API sign-off** | Integration guide notes production is "TBC — pending sign-off". What's the status? | Carrie / DevOps | Check with Carrie |
| 5 | **Re-engagement flow** | Users who toggle "contact me" are warm leads. How does Nineteen58 plan to use this data? | Nineteen58 / Carrie | Discussion |

### Live Capture (fill in during meeting)

- [ ] PhoneNumber: Confirmed wanted? ___________
- [ ] UserDeviceId: Confirmed wanted? ___________
- [ ] Reason mapping document: Needed? ___________
- [ ] Production timeline: ___________
- [ ] Re-engagement approach: ___________
- [ ] Follow-up meeting date: ___________
- [ ] Jira ticket for API update: ___________

---

## Section 10 — Closing (1 min)

> "To summarize: the exit modal changes are already merged. Your existing API will automatically serve the new exit reasons."
>
> "We have a small follow-up to expose `phoneNumber` (and optionally `userDeviceId`) in the API response."
>
> "Thanks everyone. Any final questions?"

---

## Appendix A — Reason String Mapping (Old → New)

### Direct Mappings

| Old Reason (OEPE-1820) | New Reason (OEPE-2263) | Change Type |
|-------------------------|------------------------|-------------|
| I need more information before I sign up | I need to know more before I commit | **Replaced** (similar intent, new wording) |
| I don't have time right now | I don't have time right now | **Unchanged** |
| I'm not interested in opening an account | Just exploring, not interested right now | **Replaced** (similar intent, new wording) |
| There was an error or technical issue | *(no direct equivalent)* | **Removed** — users can describe via "Other" |

### New Reasons (no old equivalent)

| New Reason | Category |
|------------|----------|
| I want to login not register | Wrong flow — existing user |
| I already have an account | Potential duplicate |
| I'm stuck on choosing a username | UX friction |
| I'm stuck on creating a password | UX friction |

### Cutoff Date

Records with `dateCreatedUtc` **before ~July 3, 2026** will have old reason strings.  
Records **from July 3 onward** will have the new reason strings.

---

## Appendix B — Full Q&A Reference

### API Compatibility

**Q: We're already using the API — will our integration break?**  
A: No. The API contract is backward-compatible. The `exitReason` field will contain new string values (8 new reasons instead of 4), but the JSON structure is identical. Your code won't break, but your categorization/parsing logic should be updated to recognize the new reason strings.

**Q: When will the new exit reasons start appearing?**  
A: They already are. The PR was merged to main on July 3. Any records created after that will have the new reason strings.

**Q: Are the exit reason strings stable? Will they change again?**  
A: They're defined as constants in the C# backend. If we update the wording, we'd coordinate with you in advance.

### Missing Fields

**Q: Where is the PhoneNumber? We don't see it in API responses.**  
A: Correct — PhoneNumber is saved to the database but the API response model (`ExitFeedbackRecord`) hasn't been updated yet. We need to add `phoneNumber` to `ExitFeedbackRecord` and update the Select projection in `ExitFeedbackQueryService`. Quick change — we can do it this sprint.

**Q: What about UserDeviceId? Can we get that too?**  
A: Same situation — it's been in the DB since OEPE-1820 but was never mapped into the API response. If it's useful for your models (e.g., correlating repeat drop-offs from the same browser), we can add it alongside PhoneNumber.

### User Behavior

**Q: What if the user doesn't select any reason and just closes?**  
A: We still save a record with `ExitReason = "Closed without feedback"`. But now, if the user DID select a reason and then clicked X, we save the actual reason — not "Closed without feedback". So you'll see fewer generic entries.

**Q: Can a user submit exit feedback multiple times?**  
A: Yes. Each time they open the exit modal, it creates a new row. Use `exitReasonId` + `dateCreatedUtc` for de-duplication as noted in the integration guide.

**Q: Does the OTP registration flow also have this exit modal?**  
A: No. Only the BasicRegistration flow (non-OTP tenants). OTP registration uses a different view without the exit modal.

### Phone Number

**Q: How does the phone number field work? Is it always South African?**  
A: Currently yes — `+27` prefix, 9-digit SA mobile number (no leading zero). If we expand to other countries, we'd update the prefix and validation rules and communicate the change.

### Other Fields

**Q: What's the ReferralPartner field?**  
A: We extract the `productid` query parameter from the landing URL. If the user came through Capitec, ABSA, or another partner flow, you'll see that partner name. Lets you segment drop-offs by acquisition channel.

**Q: Do you capture how far into registration the user got?**  
A: Not directly in the exit feedback data. We pre-populate the email they entered on the sign-up form, but we don't track which fields they completed. Could be a future enhancement.

### Security & Privacy

**Q: Is there any PII concern with the data you're exposing?**  
A: Email and phone are optional — only captured when the user explicitly toggles "contact me". The DeviceId is a random GUID. The API is already scoped to a dedicated OAuth client (`nineteen58-exit-feedback`) with its own scope (`idp_exit_feedback_endpoint`). No other IDP APIs or tenant data are accessible.

### Operational

**Q: What's the recommended polling cadence?**  
A: Per the integration guide: once every 15–60 minutes. Track the latest `dateCreatedUtc` you processed and use it as `fromDate` on the next call. Cache your OAuth token for the full 1-hour lifetime.

**Q: Is the API in production yet?**  
A: The integration guide says production is "TBC — pending sign-off". UAT is available at `uatidentity.openeasy.io`. Check with Carrie or DevOps on production timeline.

---

## Appendix C — Ticket Reference

| Ticket | Title | Owner | Date | Status |
|--------|-------|-------|------|--------|
| OEPE-1820 | Original exit modal | KG | June 2024 | Merged |
| OEPE-1849 | Form updates | KG | October 2024 | Merged |
| OEPE-1894 | Form fix | KG | November 2024 | Merged |
| OEPE-2165 | Exit feedback API for Nineteen58 | KG | April 2026 | On develop/release (not main) |
| OEPE-2263 | Exit modal redesign | Nathaniel | July 2026 | Merged (PR #624, July 3) |
| OEPE-2266 | Sub-task | Nathaniel | — | Part of OEPE-2263 |
| OEPE-2267 | Sub-task | Nathaniel | — | Part of OEPE-2263 |
| OEPE-2268 | Sub-task | Nathaniel | — | Part of OEPE-2263 |
| OEPE-2269 | Sub-task | Nathaniel | — | Part of OEPE-2263 |
| OEPE-2270 | Sub-task | Nathaniel | — | Part of OEPE-2263 |

---

## Appendix D — Key Files Reference

### Exit Modal (Write Path)

| Component | Path |
|-----------|------|
| View | `src/.../Views/Default/Registration/BasicRegistration.cshtml` |
| Controller | `src/.../Controllers/RegistrationController.cs` |
| Service | `src/.../Services/Registration/RegistrationService.cs` |
| JS (source) | `src/.../src/js/passwordRequirements/PageScripts/exitRegistration.js` |
| JS (built) | `src/.../wwwroot/js/exitRegistration.js` |
| CSS | `src/.../src/css/easyequities.css` |
| Model | `src/.../Models/ExitFeedbackModel.cs` |
| DB Context | `src/.../DbContext/ExitRegistrationDbContext.cs` |
| Constants | `src/.../Constants.cs` |
| Tests | `test/.../Services/Registration/RegistrationServiceTests.cs` |
| DeviceId Middleware | `src/.../Middleware/DeviceIdMiddleware.cs` |

### Nineteen58 API (Read Path)

| Component | Path |
|-----------|------|
| Controller | `src/.../Features/Api/Controllers/ExitFeedbackController.cs` |
| Response Model | `src/.../Features/Api/Models/ExitFeedbackResponse.cs` |
| Query Service | `src/.../Features/Api/Services/ExitFeedbackQueryService.cs` |
| Query Interface | `src/.../Features/Api/Services/IExitFeedbackQueryService.cs` |
| API Startup | `src/.../Features/Api/ApiStartupExtensions.cs` |
| Controller Tests | `test/.../Api/Controllers/ExitFeedbackControllerTests.cs` |
| Service Tests | `test/.../Api/Services/ExitFeedbackQueryServiceTests.cs` |

---

## Appendix E — Quick Reference Cheat Sheet

### Old vs New Summary

| Aspect | Before (OEPE-1820) | After (OEPE-2263) |
|--------|--------------------|--------------------|
| Exit reasons | 4 generic | 8 specific + Other |
| Contact capture | None (email fields commented out) | Toggle with email + phone |
| Phone number | Not captured | +27 prefix, 9-digit SA number (DB only, not yet in API) |
| X-close behavior | Always "Closed without feedback" | Saves selected reason + contact data if entered |
| Validation | None | Client-side blur + server-side Regex sanitization |
| Unit tests | 0 | 30 tests covering all save paths |
| CSS | Basic modal styles | BEM-named classes matching Figma designs |

### API Impact Summary

| Aspect | Detail |
|--------|--------|
| Breaking changes | **NONE** — JSON structure unchanged, new reason strings are additive |
| Auto changes | `exitReason` field values will be different strings for new records |
| Pending changes | `phoneNumber` and `userDeviceId` not yet in API response |
| Nineteen58 action | Update parsing logic to handle 8 new exit reason strings |

### Data Flow (One Sentence)

User clicks Cancel → Modal opens → Selects reason → Optional: toggle contact → Submit or X-close → `POST /Registration/SaveFeedback` → `RegistrationService` → `ExitRegistration` SQL table → `GET /api/exit-feedback` (Nineteen58 polls via OAuth) → `ExitFeedbackQueryService` reads same table

---

**Good luck with the presentation!**

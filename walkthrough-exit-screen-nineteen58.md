# Exit Screen Walkthrough — Nineteen58 Meeting

**Date:** Monday, 13 July 2026 — 4:00 PM  
**Presenter:** Nathaniel Alcantara  
**Audience:** Nineteen58 team  
**Topic:** Exit modal implementation in the EasyEquities Identity Server (IDP)

---

## Quick Reference: What Nineteen58 Does

- Enterprise AI agent company (Johannesburg, SA — founded 2023)
- Builds omnichannel AI agents: WhatsApp, Voice, Web, Email
- Specialises in customer service automation, retention campaigns, win-back
- Outcome-based pricing: they only get paid on measurable results
- Likely use case here: **AI-driven outreach to departing EasyEquities users** using the exit data we capture

---

## 1. Agenda

| # | Topic | Time |
|---|-------|------|
| 1 | Context: Why the exit modal exists | 2 min |
| 2 | Original implementation (OEPE-1820, KG) | 3 min |
| 3 | Redesigned exit modal (OEPE-2263, PR #624) | 10 min |
| 4 | Data model: `ExitRegistration` table | 5 min |
| 5 | Data consumption options for Nineteen58 | 5 min |
| 6 | Historical data & migration notes | 3 min |
| 7 | Q&A / Next steps | Open |

---

## 2. Context: Why the Exit Modal Exists

**Purpose:** When a user initiates account closure / departure from EasyEquities, we present an exit modal to:

- Capture the **reason** for leaving (structured data)
- Optionally capture **contact preference** so someone can reach out
- Feed churn analytics for product and business teams
- (New) Enable **Nineteen58's AI agents** to initiate personalised retention conversations

**Where it lives:** `identityserver` repository — the Identity Provider (IDP) that handles authentication, SSO, consent UI, and login forms.

**Flow:**  
User initiates departure → IDP presents exit modal → User selects reason(s) → Optionally toggles "contact me" → Submits → Data saved to `ExitRegistration` SQL table → Account closure proceeds

---

## 3. Original Implementation — OEPE-1820 (KG's Work)

**Developer:** Kim Geraldine Fabe (KG)  
**Ticket:** OEPE-1820  
**Status:** Merged (this is what's on `main` today before the redesign)

### What KG Built

| Aspect | Detail |
|--------|--------|
| Exit reasons | **4 reasons** (fixed list) |
| Contact fields | **None** — no toggle, no email, no phone |
| Data storage | `ExitRegistration` SQL table |
| UI | Basic modal with radio/checkbox selection |
| Tests | Initial unit test coverage |

### The 4 Original Reasons

1. I'm not using the platform enough
2. I found a better alternative
3. I'm unhappy with the service
4. Other

### KG's Other Nineteen58-Related Tickets

KG was the original developer on the exit modal feature. Her OEPE-1820 work established:

- The `ExitRegistration` database table schema
- The initial exit modal UI component in the IDP
- The backend API endpoint to persist exit data
- The foundational flow where the modal appears during account departure

> **Note for Nineteen58:** If you query historical data, records created before the redesign will only have the original 4 reason strings above. Post-redesign records will use the new 8 reasons.

---

## 4. Redesigned Exit Modal — OEPE-2263 (PR #624)

**Developer:** Nathaniel Alcantara  
**Ticket:** OEPE-2263  
**PR:** #624 (already merged to `main`)  
**Status:** Merged and deployed

### What Changed

| Aspect | Before (OEPE-1820) | After (OEPE-2263) |
|--------|--------------------|--------------------|
| Exit reasons | 4 | **8** |
| Contact toggle | None | **"Contact me" toggle** |
| Contact fields | None | **Phone number field** |
| UI/UX | Basic modal | **Redesigned with better UX** |
| Unit tests | Basic | **30 unit tests** |
| Unsettled cash check | None | **OEPE-2193 integration** |

### The 8 Updated Exit Reasons

1. I'm not using the platform enough
2. I found a better alternative
3. I'm unhappy with the service
4. Fees are too high
5. The platform is too complicated
6. I'm having technical issues
7. I'm consolidating my investments elsewhere
8. Other

### Contact Preference Feature

- **Toggle:** "Would you like us to contact you?" (boolean switch)
- **Phone number:** Appears when toggle is ON — captures a phone number for outreach
- **Behaviour on X (close) button:** Clicking the X **discards** the form entirely — no data is submitted
- **Behaviour on Submit:** Only the explicit **Submit** button sends feedback + contact preference to the backend

> **This is critical for Nineteen58:** The contact toggle + phone number is exactly the data point that enables AI agent outreach. When a user opts in, Nineteen58 can initiate a WhatsApp or voice conversation to understand more and attempt retention.

### Unsettled Cash Modal (OEPE-2193)

If the user has unsettled positions (buys exceed sells), an **"Unsettled Cash" modal** is shown before the exit modal. This prevents premature account closure when funds are still in transit.

### Test Coverage

PR #624 includes **30 unit tests** covering:

- Reason selection (single and multiple)
- Contact toggle state management
- Phone number validation
- Form submission vs. dismissal (X button)
- Edge cases: empty submission, toggle without phone, etc.
- Integration with unsettled cash flow

---

## 5. Data Model: `ExitRegistration` Table

### Schema

| Column | Type | Description |
|--------|------|-------------|
| Id | int / bigint | Primary key |
| UserId | uniqueidentifier | The departing user's ID |
| Reason | nvarchar | Selected exit reason(s) |
| ContactMe | bit | Whether user opted in for contact |
| PhoneNumber | nvarchar | Phone number (if contact opted in) |
| CreatedDate | datetime | Timestamp of exit submission |

> **Note:** Exact column names may vary — confirm with the DBA. The above reflects the logical model.

### Key Points for Nineteen58

- **No direct integration exists today.** There is no API call, webhook, or data push to Nineteen58 from the IDP codebase.
- Data is written to the `ExitRegistration` SQL table and stays there.
- Nineteen58 needs to consume this data somehow — see next section.

---

## 6. Data Consumption Options for Nineteen58

This is an **open discussion point** for the meeting. Options include:

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **Direct DB Read** | Nineteen58 queries the `ExitRegistration` table on a schedule | Simple, fast to implement | Requires DB access, tight coupling |
| **Scheduled Export** | A scheduled job exports new exit records to CSV/JSON and delivers to Nineteen58 (SFTP, S3, email) | Decoupled, auditable | Latency (not real-time), maintenance |
| **API Endpoint** | Build a REST API that Nineteen58 calls to fetch recent exit registrations | Clean, standard integration | Requires development effort |
| **Webhook / Event Push** | On each exit submission, push an event to Nineteen58's endpoint | Real-time, enables immediate outreach | Requires Nineteen58 to expose an endpoint, error handling |
| **Message Queue** | Publish exit events to a queue (e.g., RabbitMQ, Azure Service Bus) that Nineteen58 consumes | Decoupled, reliable, scalable | Infrastructure overhead |

### Recommendation

For **real-time AI agent outreach** (Nineteen58's strength), a **webhook or event-based approach** would be ideal — the moment a user submits their exit with "contact me" = true, an event fires to Nineteen58, and their AI agent can initiate a conversation within minutes.

For a **quick first implementation**, a **scheduled export** or **API endpoint** may be more practical.

> **Action item from this meeting:** Agree on the integration approach and ownership.

---

## 7. Historical Data & Migration Notes

### Data Differences Between Versions

| Aspect | Pre-Redesign Records | Post-Redesign Records |
|--------|---------------------|-----------------------|
| Reasons available | 4 original reasons | 8 updated reasons |
| ContactMe field | Not populated (NULL or false) | Populated based on user toggle |
| PhoneNumber field | Not populated | Populated when contact opted in |
| Volume | All historical exits | New exits going forward |

### What Nineteen58 Should Know

1. **Historical records will NOT have contact preferences** — the contact toggle didn't exist before OEPE-2263
2. **Reason strings changed** — the old 4 reasons and new 8 reasons may overlap but aren't identical; if building analytics/categorisation, account for both sets
3. **No backfill planned** — old records stay as-is with the original reason strings
4. **"Other" reason exists in both versions** — free-text may or may not be captured alongside it (confirm with the team)

---

## 8. Key Q&A — Anticipated Questions

### "How does the exit modal get triggered?"
When a user initiates account departure through the EasyEquities platform, the IDP (identityserver) presents the exit modal as part of the closure flow. It's not a standalone page — it's embedded in the departure UX.

### "Can we get real-time notifications when someone exits?"
Not today. Currently, data is written to the SQL table. Real-time integration would require building a webhook or event pipeline (see Section 6).

### "What if the user closes the modal without submitting?"
The X (close) button **discards everything**. No data is saved. Only the explicit Submit button persists the exit data. This was a deliberate design decision confirmed with Carrie Ann Singh.

### "Can users select multiple exit reasons?"
Yes — the redesigned modal supports multi-select for exit reasons.

### "What about the unsettled cash scenario?"
If a user has unsettled positions (buys > sells), the system shows an "Unsettled Cash" modal (OEPE-2193) **before** the exit modal. This prevents premature departure when money is still in transit.

### "What happens if the user opts in for contact but doesn't enter a phone number?"
The phone number field is validated — if the toggle is ON, a valid phone number is required before submission. The 30 unit tests cover these edge cases.

### "Is email captured too?"
The current redesign captures **phone number** as the contact field, not email. The user's email is already available in the system from their account profile, so it could be joined downstream.

### "Who originally built this?"
KG (Kim Geraldine Fabe) built the original exit modal under OEPE-1820 with 4 reasons and no contact fields. Nathaniel redesigned it under OEPE-2263 (PR #624) with 8 reasons, contact toggle, phone number, and 30 tests.

### "Where does the data live?"
`ExitRegistration` table in the Identity Server database. No data is pushed externally today.

---

## 9. Architecture Overview

```
User clicks "Close Account"
         │
         ▼
┌─────────────────────────┐
│   EasyEquities Platform  │
│   (paymentsDepartureMicro│
│    / main app)           │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐     ┌─────────────────────┐
│   Identity Server (IDP)  │     │  Unsettled Cash      │
│                          │◄────│  Check (OEPE-2193)   │
│  ┌─────────────────────┐│     └─────────────────────┘
│  │   Exit Modal         ││
│  │  - 8 reasons         ││
│  │  - Contact toggle    ││
│  │  - Phone number      ││
│  └──────────┬──────────┘│
│             │ Submit     │
│             ▼            │
│  ┌─────────────────────┐│
│  │  ExitRegistration    ││
│  │  SQL Table           ││
│  └──────────┬──────────┘│
└─────────────┼───────────┘
              │
              ▼
    ┌─────────────────┐
    │   ???            │  ◄── Integration TBD
    │   Nineteen58     │      (This meeting's discussion)
    │   AI Agents      │
    └─────────────────┘
              │
              ▼
    ┌─────────────────┐
    │  WhatsApp / Voice│
    │  / Web outreach  │
    │  to departing    │
    │  user            │
    └─────────────────┘
```

---

## 10. Tickets Reference

| Ticket | Title | Owner | Status | Description |
|--------|-------|-------|--------|-------------|
| OEPE-1820 | Original Exit Modal | KG (Kim Geraldine Fabe) | Merged | Initial exit modal with 4 reasons, no contact fields |
| OEPE-2263 | Exit Modal Redesign | Nathaniel Alcantara | Merged (PR #624) | 8 reasons, contact toggle, phone number, 30 tests |
| OEPE-2193 | Unsettled Cash Modal | Related | Merged | Shows warning when buys exceed sells before exit |

---

## 11. Action Items Template

Use this to capture decisions during the meeting:

- [ ] **Integration approach agreed:** _________________________ (DB read / export / API / webhook / queue)
- [ ] **Data format agreed:** _________________________ (JSON / CSV / direct query)
- [ ] **Frequency / latency requirement:** _________________________ (real-time / hourly / daily)
- [ ] **Who builds the integration?** _________________________ (EE team / Nineteen58 / shared)
- [ ] **Historical data needed?** _________________________ (yes — how far back / no — only new records)
- [ ] **PII handling confirmed:** _________________________ (phone numbers, user IDs — POPIA compliance)
- [ ] **Next meeting / follow-up date:** _________________________
- [ ] **Jira ticket for integration work:** _________________________

---

## 12. Quick Stats (Talking Points)

- EasyEquities has **~1.25 million active clients** (as of Feb 2026 interim results)
- Total client assets: **R94.9 billion** (up 41% YoY)
- Average user age: **32 years old**
- Purple Group's board has approved acquisition of an AI technology business (due diligence underway)
- Platform efficiency ratio improved from 87% (2023) to 52% (2026) — target: 45% within 3 years via AI automation
- Charles Savage (CEO): *"No technology opportunity has excited me more than placing AI at the centre of the intelligence of our operating system"*

---

**Good luck with the presentation!**

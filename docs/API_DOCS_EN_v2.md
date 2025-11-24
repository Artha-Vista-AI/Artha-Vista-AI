# Vibe Code – API Documentation (MVP, English)

This document specifies the REST API for the **Vibe Code / Artha Vista** MVP – an AI-powered credit scoring and insight layer on top of Amartha’s lending process.

All endpoints use JSON over HTTPS.

---

## 0. Authentication & Base URL

**Authentication**  

All requests (except `/health`) must include a valid **Firebase ID Token** in the header:

- `Authorization: Bearer <firebase_id_token>`

The backend verifies the token using the Firebase Admin SDK and maps it to one of these roles:

- `ROLE_FIELD_AGENT`
- `ROLE_OFFICER`
- `ROLE_ADMIN` (for internal ops / debugging)

**Base URLs**  

- Staging: `https://api-staging.vibecode.app`
- Production: `https://api.vibecode.app`

**Error Envelope**  

All error responses use the following structure:

```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "Human readable message",
    "details": null
  }
}
```

Typical error codes:

- `UNAUTHORIZED`
- `FORBIDDEN`
- `NOT_FOUND`
- `INVALID_REQUEST`
- `INTERNAL_ERROR`
- `ALREADY_ASSIGNED`

---

## 1. Health Check

### GET `/health`

Simple liveliness check for monitoring.

**Auth**: none

**Response 200**

```json
{
  "status": "ok",
  "service": "vibecode-api",
  "timestamp": "2025-11-24T12:00:00Z"
}
```

---

## 2. Applications

### 2.1 Ingest Application (Webhook from core system)

### POST `/api/applications/webhook`

**Description**  
Endpoint used by Amartha’s core system (or a simulator) to push new loan applications into Vibe Code.

**Auth**  

- Firebase token with an internal service user, or an additional header such as `X-Internal-Key` (implementation detail).

**Request Body**

```json
{
  "application_id": "APP-2025-0001",
  "borrower": {
    "borrower_id": "BRW-123",
    "name": "Siti",
    "phone": "+628123456789",
    "village": "Desa A",
    "region": "Cirebon"
  },
  "business_profile": {
    "business_type": "warung",
    "business_age_months": 18,
    "monthly_revenue": 8000000,
    "monthly_profit": 2500000
  },
  "loan_request": {
    "amount": 5000000,
    "tenor_months": 12,
    "purpose": "Working capital for inventory"
  }
}
```

**Behavior**  

- Validates the payload.
- Creates or updates a document in the `applications` collection.
- Sets `status = "new"` if it is a new application.

**Response 200**

```json
{
  "application_id": "APP-2025-0001",
  "status": "new"
}
```

---

### 2.2 List Applications (Officer & Admin)

### GET `/api/applications`

**Description**  
Returns a paginated list of applications, with optional filters for status, risk category, and region.

**Auth**  

- `ROLE_OFFICER`  
- `ROLE_ADMIN`

**Query Parameters**

- `status`: one of `new`, `scheduled`, `visited`, `scored`, `approved`, `rejected`, `need_more_detail`
- `risk_category`: `low`, `medium`, or `high`
- `region`: region string (e.g. `"Cirebon"`)
- `limit`: integer, default `20`, max `100`
- `page_token`: opaque cursor for pagination

**Response 200**

```json
{
  "items": [
    {
      "application_id": "APP-2025-0001",
      "borrower_name": "Siti",
      "village": "Desa A",
      "region": "Cirebon",
      "status": "scored",
      "risk_score": 62,
      "risk_category": "medium",
      "created_at": "2025-11-24T12:00:00Z"
    }
  ],
  "next_page_token": null
}
```

---

### 2.3 Get Application Detail

### GET `/api/applications/{application_id}`

**Auth**  

- `ROLE_FIELD_AGENT` (only if `assigned_field_agent_id` equals the current user)
- `ROLE_OFFICER`
- `ROLE_ADMIN`

**Response 200**

```json
{
  "application": {
    "application_id": "APP-2025-0001",
    "borrower_id": "BRW-123",
    "status": "scored",
    "loan_request": {
      "amount": 5000000,
      "tenor_months": 12,
      "purpose": "Working capital for inventory"
    },
    "business_profile": {
      "business_type": "warung",
      "business_age_months": 18,
      "monthly_revenue": 8000000,
      "monthly_profit": 2500000
    },
    "location": {
      "village": "Desa A",
      "region": "Cirebon"
    },
    "assigned_field_agent_id": "FA-123",
    "scheduled_visit_at": "2025-11-25T09:00:00Z",
    "timestamps": {
      "created_at": "2025-11-24T12:00:00Z",
      "visited_at": "2025-11-24T13:00:00Z",
      "scored_at": "2025-11-24T13:05:00Z"
    }
  },
  "survey": {
    "visit_notes": "The shop has been running consistently, inventory is neatly arranged, and there are many repeat customers.",
    "numeric_fields": {
      "inventory_value": 4000000,
      "household_size": 4
    },
    "tags": {
      "has_collateral": true,
      "has_prior_default": false
    },
    "photo_urls": [
      "https://storage.googleapis.com/vibecode-demo/app-2025-0001/front.jpg",
      "https://storage.googleapis.com/vibecode-demo/app-2025-0001/inside.jpg"
    ],
    "visited_at": "2025-11-24T13:00:00Z"
  },
  "score": {
    "risk_score": 62,
    "risk_category": "medium",
    "suggested_loan_limit": 4500000,

    "business_health_summary": "The business is stable with consistent revenue, well-organized inventory, and a steady base of repeat customers.",

    "top_reasons": [
      "Monthly revenue has been relatively stable over the last six months based on interview and observation.",
      "Business age is under two years, which increases risk but is offset by stable customer traffic.",
      "The borrower has no other large active loans in the same institution."
    ],

    "risk_factors": [
      {
        "code": "SHORT_BUSINESS_AGE",
        "label": "Business is relatively new",
        "impact": "negative",
        "weight": 0.25
      },
      {
        "code": "STABLE_REVENUE",
        "label": "Stable revenue trend",
        "impact": "positive",
        "weight": 0.35
      }
    ],

    "supporting_evidence": [
      {
        "source": "FIELD_NOTE",
        "snippet": "The shop is busy during peak hours and most visitors appear to be repeat customers."
      },
      {
        "source": "PHOTO",
        "description": "Shelves are neatly organized and fast-moving products are fully stocked."
      }
    ],

    "mitigation_notes": [
      "Start with a loan limit slightly below the borrower’s requested amount to reduce exposure.",
      "Monitor repayment behavior for the first two instalments before considering a limit increase."
    ],

    "visual_observations": [
      {
        "type": "STORE_CONDITION",
        "value": "well_organized"
      },
      {
        "type": "HOUSE_CONDITION",
        "value": "simple_permanent"
      }
    ],

    "consistency_checks": [
      {
        "field": "monthly_revenue",
        "status": "consistent",
        "details": "Interviewed revenue aligns with inventory volume and observed customer flow."
      },
      {
        "field": "inventory_value",
        "status": "slightly_low",
        "details": "Inventory value is slightly below what would be expected for the claimed revenue; suggest monitoring restocking frequency."
      }
    ]
  },
  "decision": {
    "status": "approved",
    "final_loan_limit": 4500000,
    "officer_notes": "Approve with a slightly lower limit and monitor the first two repayment cycles.",
    "ai_followed": true,
    "decided_at": "2025-11-24T13:10:00Z"
  }
}
```

**Response 404** – returned if the application does not exist or the user does not have access.

---

## 3. Survey (Field Agent)

### 3.1 List Assigned Applications

### GET `/api/field/assigned-applications`

**Description**  
Returns applications assigned to the current Field Agent, including those that are already scheduled.

**Auth**  

- `ROLE_FIELD_AGENT`

**Response 200**

```json
{
  "items": [
    {
      "application_id": "APP-2025-0001",
      "borrower_name": "Siti",
      "village": "Desa A",
      "region": "Cirebon",
      "status": "scheduled",
      "scheduled_visit_at": "2025-11-25T09:00:00Z"
    },
    {
      "application_id": "APP-2025-0002",
      "borrower_name": "Budi",
      "village": "Desa B",
      "region": "Cirebon",
      "status": "new",
      "scheduled_visit_at": null
    }
  ]
}
```

**Notes**  

- `status` can be `new`, `scheduled`, `visited`, `scored`, `approved`, `rejected`, or `need_more_detail`.
- If `status = "scheduled"`, `scheduled_visit_at` must be non-null.

---

### 3.2 Schedule Visit (Book Application)

### POST `/api/field/applications/{application_id}/schedule`

**Description**  
Field Agent books an onsite visit for an application. Once scheduled, the application is **locked** to that agent until it is either visited or manually reassigned by an Officer/Admin.

**Auth**  

- `ROLE_FIELD_AGENT`

**Request Body**

```json
{
  "scheduled_visit_at": "2025-11-25T09:00:00Z"
}
```

**Behavior**  

- Only allowed if:
  - `status` of the application is `new`, and
  - `assigned_field_agent_id` is currently `null`.
- Atomically updates the application document:
  - `status` → `"scheduled"`
  - `assigned_field_agent_id` → current Field Agent’s ID
  - `scheduled_visit_at` → value from the request
- If another Field Agent has already scheduled this application, returns `409 Conflict`.

**Response 200**

```json
{
  "application_id": "APP-2025-0001",
  "status": "scheduled",
  "assigned_field_agent_id": "FA-123",
  "scheduled_visit_at": "2025-11-25T09:00:00Z"
}
```

**Response 409**

```json
{
  "error": {
    "code": "ALREADY_ASSIGNED",
    "message": "Application already scheduled by another field agent.",
    "details": null
  }
}
```

---

### 3.3 Submit Survey & Photos

### POST `/api/applications/{application_id}/survey`

**Description**  
Field Agent submits the result of the onsite visit, including numeric fields, notes, and photo URLs.

**Auth**  

- `ROLE_FIELD_AGENT` (must match `assigned_field_agent_id`)

**Request Body**

```json
{
  "visit_notes": "The shop is frequently visited by local residents, shelves are clean and organized, and the owner records daily revenue in a notebook.",
  "numeric_fields": {
    "estimated_monthly_revenue": 8000000,
    "estimated_monthly_profit": 2500000,
    "business_age_months": 18
  },
  "tags": {
    "has_collateral": true,
    "has_prior_default": false
  },
  "photo_urls": [
    "https://storage.googleapis.com/vibecode-demo/app-2025-0001/front.jpg",
    "https://storage.googleapis.com/vibecode-demo/app-2025-0001/inside.jpg",
    "https://storage.googleapis.com/vibecode-demo/app-2025-0001/owner-and-ledger.jpg"
  ]
}
```

**Behavior**  

- Validates the payload.
- Writes survey to `surveys` collection.
- Updates application status from `scheduled` to `visited`.
- Enqueues an internal scoring job (implemented as a backend service workflow, not as “AI agents”).

**Response 202**

```json
{
  "status": "accepted",
  "next": "scoring_in_progress"
}
```

---

## 4. Scoring (AI)

### 4.1 Get Score

### GET `/api/applications/{application_id}/score`

**Description**  
Returns the AI-generated risk score and the full insight pack for a given application.

**Auth**  

- `ROLE_FIELD_AGENT` (optional, depending on business rules)
- `ROLE_OFFICER`
- `ROLE_ADMIN`

**Response 200**

```json
{
  "status": "ready",
  "score": {
    "risk_score": 62,
    "risk_category": "medium",
    "suggested_loan_limit": 4500000,

    "business_health_summary": "The business shows a stable revenue pattern with sufficient inventory and strong repeat customer behavior. The main risk driver is short operating history, partially mitigated by consistent daily sales.",

    "top_reasons": [
      "Monthly revenue estimates are consistent with observed inventory and customer traffic.",
      "The business has been operating for less than two years, which increases risk.",
      "No significant competing loans are recorded; debt burden appears manageable."
    ],

    "risk_factors": [
      {
        "code": "SHORT_BUSINESS_AGE",
          "label": "Business is relatively new",
          "impact": "negative",
          "weight": 0.25
      },
      {
        "code": "STABLE_REVENUE",
        "label": "Stable revenue trend",
        "impact": "positive",
        "weight": 0.35
      },
      {
        "code": "INVENTORY_MATCH",
        "label": "Inventory value aligns with claimed revenue",
        "impact": "positive",
        "weight": 0.20
      }
    ],

    "supporting_evidence": [
      {
        "source": "FIELD_NOTE",
        "snippet": "Daily revenue is written in a notebook and matches the owner’s oral explanation for the last few weeks."
      },
      {
        "source": "PHOTO",
        "description": "Shelves contain a variety of fast-moving products with minimal empty space."
      }
    ],

    "mitigation_notes": [
      "Start with a moderate loan limit and consider an increase only after 3 on-time repayments.",
      "Encourage the borrower to continue simple bookkeeping, which can be digitized later.",
      "If repayment behavior remains strong, reevaluate the risk category in the next cycle."
    ],

    "visual_observations": [
      {
        "type": "STORE_CONDITION",
        "value": "well_organized"
      },
      {
        "type": "EQUIPMENT_PRESENCE",
        "value": "basic_but_sufficient"
      }
    ],

    "consistency_checks": [
      {
        "field": "monthly_revenue",
        "status": "consistent",
        "details": "Reported monthly revenue matches the pattern observed in the revenue notebook and stock rotation."
      },
      {
        "field": "inventory_value",
        "status": "slightly_low",
        "details": "Inventory is slightly below what would be expected for the claimed revenue; advise monitoring stock replenishment."
      }
    ]
  }
}
```

**Response 202**

```json
{
  "status": "pending"
}
```

---

### 4.2 Trigger Re-score

### POST `/api/applications/{application_id}/rescore`

**Description**  
Request a re-run of the scoring workflow (for example, after new survey data is added).

**Auth**  

- `ROLE_OFFICER`
- `ROLE_ADMIN`

**Response 202**

```json
{
  "status": "rescore_requested"
}
```

---

## 5. Decisions (Officer)

### 5.1 Submit Decision

### POST `/api/applications/{application_id}/decision`

**Description**  
Officer records the final credit decision for an application.

**Auth**  

- `ROLE_OFFICER`

**Request Body**

```json
{
  "decision": "approved",
  "final_loan_limit": 4500000,
  "officer_notes": "Approve at a slightly reduced limit; monitor first three instalments closely.",
  "ai_followed": true
}
```

**Response 200**

```json
{
  "application_id": "APP-2025-0001",
  "status": "approved",
  "final_loan_limit": 4500000
}
```

---

### 5.2 Get Decision

### GET `/api/applications/{application_id}/decision`

**Auth**  

- `ROLE_OFFICER`
- `ROLE_ADMIN`

**Response 200**

```json
{
  "decision": {
    "status": "approved",
    "final_loan_limit": 4500000,
    "officer_notes": "Approve at a slightly reduced limit; monitor first three instalments closely.",
    "ai_followed": true,
    "decided_at": "2025-11-24T13:10:00Z"
  }
}
```

---

## 6. Role Summary

- **Field Agent**
  - Views assigned applications.
  - Schedules onsite visits.
  - Submits survey data and photos.

- **Officer**
  - Reviews scored applications.
  - Reads AI Insight Pack and photos.
  - Records final decisions.

- **Admin**
  - Operational and debugging access.

This API spec focuses on a clear scoring workflow and rich, explainable AI output, without using any “agentic AI” concept in the runtime architecture.

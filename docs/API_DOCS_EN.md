# Vibe Code – API Documentation (MVP, English Version)

This document describes the REST API for the **Vibe Code / Artha Vista** MVP – an AI-powered credit scoring and insight layer that sits on top of Amartha’s lending flow.

All endpoints are JSON over HTTPS.

---

## Auth & Base URL

Auth:  
- All requests (except `/health`) must include a **Firebase ID Token** in the header:
  - `Authorization: Bearer <firebase_id_token>`
- The backend verifies the token using the Firebase Admin SDK and maps it to an internal role:
  - `ROLE_FIELD_AGENT`
  - `ROLE_OFFICER`
  - `ROLE_ADMIN` (optional, for internal ops)

Base URL examples:
- Staging: `https://api-staging.vibecode.app`
- Production: `https://api.vibecode.app`

Common response envelope for errors:

```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "Human readable message",
    "details": null
  }
}
```

Typical `code` values:
- `UNAUTHORIZED`
- `FORBIDDEN`
- `NOT_FOUND`
- `INVALID_REQUEST`
- `INTERNAL_ERROR`
- `ALREADY_ASSIGNED`

---

## 1. Health Check

### GET `/health`

**Description**  
Simple liveliness check.

**Auth**  
- No auth required.

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

### 2.1 Ingest Application (Webhook from external system)

### POST `/api/applications/webhook`

**Description**  
Endpoint used by Amartha’s core system (or a simulator) to push a new loan application into Vibe Code.

**Auth**
- Typically protected by an internal token (e.g. `x-internal-key`) in addition to standard auth, depending on deployment.

**Request Body (example)**

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
    "purpose": "Modal tambahan stok"
  }
}
```

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
List applications with basic filters on status, risk category, and region.

**Auth**
- `ROLE_OFFICER` or `ROLE_ADMIN`

**Query Params (optional)**

- `status` – `new|scheduled|visited|scored|approved|rejected|need_more_detail`
- `risk_category` – `low|medium|high`
- `region` – string
- `limit` – default `20`
- `page_token` – opaque cursor for pagination

**Response 200 (example)**

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
- `ROLE_FIELD_AGENT` (only if the application is assigned to them)
- `ROLE_OFFICER`
- `ROLE_ADMIN`

**Response 200 (example)**

```json
{
  "application": {
    "application_id": "APP-2025-0001",
    "borrower_id": "BRW-123",
    "status": "scored",
    "loan_request": {
      "amount": 5000000,
      "tenor_months": 12,
      "purpose": "Modal tambahan stok"
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
    "visit_notes": "The shop has been running consistently, shelves are neat, regular customers are present.",
    "numeric_fields": {
      "inventory_value": 4000000,
      "household_size": 4
    },
    "tags": {
      "has_collateral": true,
      "has_prior_default": false
    },
    "photo_urls": [
      "https://storage.googleapis.com/.../photo1.jpg",
      "https://storage.googleapis.com/.../photo2.jpg"
    ]
  },
  "score": {
    "risk_score": 62,
    "risk_category": "medium",
    "suggested_loan_limit": 4500000,

    "business_health_summary": "Business is stable, inventory is well-organized, and monthly revenue has been consistent for the last six months.",
    "top_reasons": [
      "Monthly revenue has been relatively stable in the last six months.",
      "Business age is under two years, creating a moderate risk profile.",
      "No other large active loans detected."
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
        "label": "Stable revenue",
        "impact": "positive",
        "weight": 0.35
      }
    ],

    "supporting_evidence": [
      {
        "source": "FIELD_NOTE",
        "snippet": "The shop is busy during peak hours with many repeat customers."
      },
      {
        "source": "PHOTO",
        "description": "Shelves are neatly organized with fast-moving goods available."
      }
    ],

    "mitigation_notes": [
      "Start with a limit slightly below the originally requested amount.",
      "Monitor the first two repayment cycles before increasing the limit."
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
        "details": "Field estimate is aligned with the declared profile."
      },
      {
        "field": "inventory_value",
        "status": "slightly_low",
        "details": "Inventory value is slightly below the claimed revenue; should be monitored."
      }
    ]
  },
  "decision": {
    "status": "approved",
    "final_loan_limit": 4500000,
    "officer_notes": "Approve with slightly lower limit and monitor the first two months.",
    "ai_followed": true
  }
}
```

**404 NOT_FOUND** – returned if the application does not exist or the user does not have access.

---

## 3. Survey (Field Agent)

### 3.1 List Assigned Applications (Field Agent)

### GET `/api/field/assigned-applications`

**Auth**
- `ROLE_FIELD_AGENT`

**Description**  
Returns applications assigned to the current Field Agent (including those already scheduled).

**Response 200 (example)**

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
      "status": "new"
    }
  ]
}
```

**Notes**
- `status` can be `new|scheduled|visited|scored|approved|rejected|need_more_detail`.
- If `status = "scheduled"`, the response includes `scheduled_visit_at` so the agent can see the booked time slot.

---

### 3.2 Schedule Visit (Book Application)

### POST `/api/field/applications/{application_id}/schedule`

**Auth**
- `ROLE_FIELD_AGENT`

**Description**  
Field Agent books/schedules a visit for an application. Once an application is scheduled by one agent, it **cannot** be scheduled by any other agent (unless later unassigned by an Officer/Admin).

**Request Body**

```json
{
  "scheduled_visit_at": "2025-11-25T09:00:00Z"
}
```

**Behavior**

- Can only be called if:
  - Application `status` is `new`, **and**
  - `assigned_field_agent_id` is empty.
- Atomically updates:
  - `status` → `scheduled`
  - `assigned_field_agent_id` → `current_user_id`
  - `scheduled_visit_at` → value from request
- If the application is already assigned to another agent, the server returns **409 Conflict**.

**Response 200**

```json
{
  "application_id": "APP-2025-0001",
  "status": "scheduled",
  "assigned_field_agent_id": "FA-123",
  "scheduled_visit_at": "2025-11-25T09:00:00Z"
}
```

**Response 409 (already booked)**

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

**Auth**
- `ROLE_FIELD_AGENT` (must be the one assigned to the application)

**Description**  
Field Agent submits the result of the onsite visit, including structured numeric data, tags, and photo URLs (uploaded via Firebase/Cloud Storage SDK; the backend only stores the URLs).

**Request Body**

```json
{
  "visit_notes": "The shop is busy, shelves are neat, and customers are mostly regulars.",
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
    "https://storage.googleapis.com/.../warung-front.jpg",
    "https://storage.googleapis.com/.../inside-stock.jpg"
  ]
}
```

**Behavior**

- Persist survey data in Firestore (`surveys` collection).
- Update application status → `visited`.
- Trigger the internal Orchestrator → DataIngestion → Scoring → Explanation pipeline.

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

**Auth**
- `ROLE_FIELD_AGENT` (optional, depending on business rules)
- `ROLE_OFFICER`
- `ROLE_ADMIN`

**Response 200 (if available)**

```json
{
  "status": "ready",
  "score": {
    "risk_score": 62,
    "risk_category": "medium",
    "suggested_loan_limit": 4500000,

    "business_health_summary": "Business is stable, inventory is well-organized, and monthly revenue has been consistent for the last six months.",
    "top_reasons": [
      "Monthly revenue has been relatively stable in the last six months.",
      "Business is younger than two years, which implies moderate risk.",
      "No large active loans are detected."
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
        "label": "Stable revenue",
        "impact": "positive",
        "weight": 0.35
      }
    ],

    "supporting_evidence": [
      {
        "source": "FIELD_NOTE",
        "snippet": "The shop is busy during peak hours with many repeat customers."
      },
      {
        "source": "PHOTO",
        "description": "Shelves are neatly organized with fast-moving goods available."
      }
    ],

    "mitigation_notes": [
      "Start with a limit slightly below the original request.",
      "Monitor the first two repayment cycles before increasing the limit."
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
        "details": "Field estimate matches the declared profile."
      },
      {
        "field": "inventory_value",
        "status": "slightly_low",
        "details": "Inventory value is slightly below the claimed revenue; monitor usage."
      }
    ]
  }
}
```

**Response 202 (if not ready yet)**

```json
{
  "status": "pending"
}
```

---

### 4.2 Trigger Re-score (Officer/Admin)

### POST `/api/applications/{application_id}/rescore`

**Auth**
- `ROLE_OFFICER`
- `ROLE_ADMIN`

**Description**  
Requests a re-run of the scoring pipeline (e.g. after new survey data).

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

**Auth**
- `ROLE_OFFICER`

**Request Body**

```json
{
  "decision": "approved",
  "final_loan_limit": 4500000,
  "officer_notes": "Approve with slightly lower limit and monitor two cycles.",
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
    "officer_notes": "Approve with slightly lower limit and monitor two cycles.",
    "ai_followed": true,
    "decided_at": "2025-11-24T13:10:00Z"
  }
}
```

---

## 6. Notes on Roles

- **Field Agent**
  - Sees assigned applications.
  - Schedules visits.
  - Submits surveys & photos.
- **Officer**
  - Sees scored applications and AI insights.
  - Makes final credit decisions.
- **Admin**
  - Operational & debugging access, optional for MVP.

This English version mirrors the Indonesian API spec, with the main differences being language and the explicit emphasis on the **AI Insight Pack** and **scheduling flow** for Field Agents.

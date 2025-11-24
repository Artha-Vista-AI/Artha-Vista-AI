# Vibe Code – Architecture & Tech Stack (English, Detailed)

This document describes the technical architecture for the **Vibe Code / Artha Vista** MVP. The focus is on a clear scoring pipeline (from application → survey → scoring → decision) without using agentic AI. All orchestration is implemented as standard backend services.

---

## 1. High-level Architecture

Main components:

1. **Borrower Channels (Existing Systems)**
   - Amartha’s existing mobile/web channels where borrowers apply for loans.
   - These systems send loan applications to Vibe Code via a webhook.

2. **Field Agent Web App**
   - Technology: React + Next.js + TypeScript.
   - Authentication: Firebase Auth.
   - Runs on mobile browsers or low-end laptops.
   - Main responsibilities:
     - Show “new” applications in the agent’s region.
     - Allow the agent to **schedule** onsite visits (booking & lock).
     - Capture survey data: numeric fields, notes, and photos.

3. **Officer Dashboard**
   - Also React + Next.js + TypeScript (same codebase as Field Agent app, using role-based views).
   - Main responsibilities:
     - Display lists of applications with their current status and risk scores.
     - Provide a detailed view with AI Insight Pack and photos.
     - Allow Officers to approve/reject or request more information.

4. **Backend API**
   - Technology: Python 3 + FastAPI, deployed on Google Cloud Run.
   - Responsibilities:
     - Verify Firebase ID tokens and enforce role-based access.
     - Expose REST endpoints as defined in the API docs.
     - Orchestrate the scoring workflow: read data → call Gemini → store results.

5. **Data Storage**
   - **Firestore**
     - Collections: `users`, `applications`, `surveys`, `scores`, `decisions`.
     - Acts as the source-of-truth for the whole MVP.
   - **Cloud Storage**
     - Stores photos taken during onsite visits.
     - Backend stores only URLs in Firestore.

6. **AI Layer (Gemini)**
   - Uses Gemini 2.5 Flash / Pro via the Gemini API (or Vertex AI).
   - The backend builds prompts and parses responses.
   - No agent orchestration; instead, the backend contains a single scoring service that:
     - Prepares inputs.
     - Calls Gemini.
     - Converts results into structured AI Insight Pack.

---

## 2. Data Model (Collections)

### 2.1 Users

Collection: `users`

- `user_id` (string, same as Firebase UID)
- `role` (`FIELD_AGENT`, `OFFICER`, `ADMIN`)
- `name`
- `region`
- `created_at`
- `updated_at`

### 2.2 Applications

Collection: `applications`

- `application_id`
- `borrower_id`
- `business_profile` (embedded object)
  - `business_type`
  - `business_age_months`
  - `monthly_revenue`
  - `monthly_profit`
- `loan_request` (embedded object)
  - `amount`
  - `tenor_months`
  - `purpose`
- `location`
  - `village`
  - `region`
- `status` – one of:
  - `new`
  - `scheduled`
  - `visited`
  - `scored`
  - `approved`
  - `rejected`
  - `need_more_detail`
- `assigned_field_agent_id` – nullable; set when an application is booked.
- `scheduled_visit_at` – nullable; desired time of onsite visit.
- `created_at`
- `updated_at`

**Business rule #1 (scheduling lock)**  
If `assigned_field_agent_id` is non-null and `status = "scheduled"`, the application **cannot** be scheduled by any other Field Agent. All scheduling operations must be executed transactionally to enforce this rule.

### 2.3 Surveys

Collection: `surveys`

- `survey_id`
- `application_id`
- `field_agent_id`
- `visit_notes` (string)
- `numeric_fields` (map of numeric values)
- `tags` (map of booleans/strings, e.g. `has_collateral`, `has_prior_default`)
- `photo_urls` (array of strings)
- `visited_at`
- `created_at`

### 2.4 Scores (AI Insight Pack)

Collection: `scores`

- `score_id`
- `application_id`
- `risk_score` (0–100)
- `risk_category` (`low`, `medium`, `high`)
- `suggested_loan_limit`
- `business_health_summary`
- `top_reasons` (array of strings)
- `risk_factors` (array of objects)
  - `code`
  - `label`
  - `impact` (`positive` / `negative`)
  - `weight` (0–1)
- `supporting_evidence` (array of objects)
  - `source` (`FIELD_NOTE`, `PHOTO`, etc.)
  - `snippet` or `description`
- `mitigation_notes` (array of strings)
- `visual_observations` (array of objects)
  - `type` (e.g. `STORE_CONDITION`)
  - `value` (e.g. `well_organized`)
- `consistency_checks` (array of objects)
  - `field` (e.g. `monthly_revenue`)
  - `status` (`consistent`, `slightly_low`, `inconsistent`)
  - `details`
- `raw_ai_payload` (optional JSON, trimmed for size)
- `scored_at`

### 2.5 Decisions

Collection: `decisions`

- `decision_id`
- `application_id`
- `officer_id`
- `decision` (`approved`, `rejected`, `need_more_detail`)
- `final_loan_limit`
- `officer_notes`
- `ai_followed` (boolean)
- `decided_at`

---

## 3. Backend Logical Components

The backend is a single FastAPI application with clear modules, not a collection of AI agents.

1. **Auth Module**
   - Verifies Firebase ID tokens.
   - Extracts `user_id` and `role` from token / Firestore.
   - Provides dependency functions for route protection.

2. **Application Module**
   - Handles `/api/applications/webhook` ingestion.
   - Exposes listing and detail endpoints for Officers/Admin.
   - Implements scheduling-related helpers (ensure lock, transactional updates).

3. **Survey Module**
   - Handles survey submission from Field Agents.
   - Validates numeric fields and tags.
   - Updates application status from `scheduled` → `visited`.

4. **Scoring Module**
   - Implements the scoring workflow as a normal service:
     - Reads data from `applications` and `surveys`.
     - Builds a structured prompt for Gemini (business profile, notes, photo descriptions).
     - Calls Gemini via `gemini_client`.
     - Parses the AI response into the `scores` schema (AI Insight Pack).
   - Provides endpoints for:
     - `GET /api/applications/{application_id}/score`
     - `POST /api/applications/{application_id}/rescore`

5. **Decision Module**
   - Handles decision submission and retrieval.
   - Ensures one decision per application (for MVP).

6. **Job / Worker Module (optional for MVP)**
   - If asynchronous scoring is needed:
     - A small background worker processes a queue of scoring jobs.
     - For hackathon scope, scoring can be done synchronously with simple retry logic.

---

## 4. Backend Directory Structure (Detailed)

Recommended structure:

```text
backend/
  app/
    main.py                 # FastAPI app entrypoint
    core/
      config.py             # Settings (ENV vars, project config)
      logging.py            # Logging configuration
    api/
      v1/
        __init__.py
        applications_router.py   # Routes for application ingestion & listing
        surveys_router.py        # Routes for survey submission
        scoring_router.py        # Routes for score retrieval/rescore
        decisions_router.py      # Routes for decisions
    schemas/
      __init__.py
      application.py        # Pydantic models for request/response
      survey.py
      score.py
      decision.py
      common.py             # Shared types (pagination, error envelope)
    models/
      __init__.py
      application.py        # Internal domain models if needed
      survey.py
      score.py
      decision.py
    repositories/
      __init__.py
      application_repository.py  # Firestore queries for applications
      survey_repository.py       # Firestore queries for surveys
      score_repository.py        # Firestore queries for scores
      decision_repository.py     # Firestore queries for decisions
    services/
      __init__.py
      auth_service.py       # Firebase token verification
      application_service.py
      survey_service.py
      scoring_service.py    # Scoring workflow implementation (no AI agents)
      decision_service.py
      firestore_client.py   # Shared Firestore wrapper
      gemini_client.py      # Wrapper around Gemini API
    utils/
      datetime_utils.py
      id_generator.py
  tests/
    unit/
      test_applications.py
      test_surveys.py
      test_scoring_service.py
      test_decisions.py
    integration/
      test_api_endpoints.py
  Dockerfile
  cloudrun.yaml
```

This structure removes any “agent” terminology and expresses the scoring flow as a set of normal services.

---

## 5. Frontend Directory Structure (Detailed)

Assuming a Next.js 15 + TypeScript + Tailwind setup.

```text
frontend/
  src/
    pages/                      # Route-level components (if using pages router)
      index.tsx                 # Landing / login redirect
      applications/
        index.tsx               # Officer list view
        [id].tsx                # Officer application detail + AI Insights
      field/
        assigned.tsx            # Field Agent list assigned/scheduled applications
        visit/
          [id].tsx              # Field Agent survey & photo upload form
    app/                        # If using app router, equivalent structure here
    components/
      layout/
        AppLayout.tsx
        Header.tsx
        Sidebar.tsx
      applications/
        ApplicationList.tsx
        ApplicationFilters.tsx
        ApplicationSummaryCard.tsx
      field/
        AssignedApplicationList.tsx
        ScheduleVisitModal.tsx
        SurveyForm.tsx
      scoring/
        ScoreSummaryCard.tsx
        RiskFactorsList.tsx
        SupportingEvidenceList.tsx
        MitigationNotesList.tsx
        ConsistencyChecksList.tsx
    lib/
      apiClient.ts              # Axios/fetch wrapper for backend
      auth.ts                   # Firebase Auth helpers
      formatters.ts             # Currency/date formatting helpers
      config.ts                 # FE configuration (API base URL, etc.)
    hooks/
      useAuth.ts
      useApplications.ts
      useAssignedApplications.ts
      useScore.ts
    types/
      application.ts            # TS interfaces aligned with backend schemas
      survey.ts
      score.ts
      decision.ts
      common.ts
  public/
    favicon.ico
    logo.png
  package.json
  next.config.mjs
  tsconfig.json
  tailwind.config.mjs
```

This structure makes it explicit where the AI Insight Pack is rendered (in components under `scoring/`) and how Field Agent vs Officer views are separated.

---

## 6. Scoring Workflow (End-to-End)

1. **Application Ingested**
   - Core system calls `POST /api/applications/webhook`.
   - Backend writes to `applications` with `status = "new"`.

2. **Field Agent Schedules Visit**
   - Field Agent opens the Field view.
   - Calls `GET /api/field/assigned-applications` and sees `new` applications in their region.
   - Calls `POST /api/field/applications/{id}/schedule` to book a timeslot.
   - Backend sets `status = "scheduled"` and locks the application to that agent.

3. **Field Agent Conducts Visit & Submits Survey**
   - On the visit day, Field Agent opens the survey page.
   - Fills in numeric fields, notes, and attaches photos.
   - Calls `POST /api/applications/{id}/survey`.
   - Backend writes to `surveys`, updates `status = "visited"`, and enqueues a scoring job.

4. **Backend Performs Scoring (No Agentic AI)**
   - `scoring_service` reads application + survey data.
   - Builds a prompt with:
     - Business profile (numbers & categories).
     - Visit notes.
     - Photo descriptions (optionally pre-generated or labeled).
   - Calls Gemini (single call per application) via `gemini_client`.
   - Parses the response into a structured AI Insight Pack and writes into `scores`.
   - Updates application `status = "scored"`.

5. **Officer Reviews & Decides**
   - Officer opens the dashboard list.
   - Retrieves scored applications via `GET /api/applications`.
   - Opens detail via `GET /api/applications/{id}` and sees:
     - Borrower & business profile
     - AI Insight Pack from `scores`
     - Photos
   - Officer posts a decision using `POST /api/applications/{id}/decision`.

This architecture keeps all “intelligence” in the AI scoring call while avoiding any runtime concept of AI agents. The system remains a conventional, auditable backend + frontend application.


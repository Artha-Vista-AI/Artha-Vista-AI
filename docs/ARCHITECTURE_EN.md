# Vibe Code – Architecture & Tech Stack (English Version)

---

## 1. High-level Architecture

Main components:

1. **Borrower App (existing / dummy)**
   - Sends loan applications to the backend endpoint `POST /api/applications/webhook`.

2. **Field Agent Web App (Vibe Code FE)**
   - Built with React/Next.js.
   - Authentication via Firebase Auth.
   - Features:
     - List of applications that need to be visited.
     - Scheduling of onsite visits.
     - Survey entry (numeric + notes + photos).

3. **Officer Dashboard**
   - Also React/Next.js (can be the same FE application with role-based views).
   - Features:
     - Filtered list of applications.
     - Detailed view with AI score & insights.
     - Decision making (approve / reject / need more detail).

4. **Backend API (FastAPI on Cloud Run)**
   - Stateless REST API.
   - Handles auth, validation, orchestration, and communication with AI services.

5. **Firestore (Data Store)**
   - Stores users, applications, surveys, scores, and decisions.
   - Used as the main source-of-truth for the MVP.

6. **Cloud Storage**
   - Stores photos from Field Agent visits.
   - The backend only persists public or signed URLs in Firestore.

7. **Gemini Scoring Service**
   - Encapsulates calls to Google Gemini 2.5 (Flash/Pro).
   - Converts raw data (forms, notes, photos) into structured AI insight and a risk score.

Data flow (simplified):

1. Application is ingested from the core system via webhook and stored in Firestore (`applications` collection).
2. Field Agent views “new” applications, schedules visits, and then performs onsite surveys.
3. After survey submission, the backend triggers the AI scoring pipeline:
   - Data ingestion → Gemini prompt → rule-based post-processing.
4. AI output is written into `scores` as an **AI Insight Pack**.
5. Officer dashboard reads from `applications`, `surveys`, and `scores`, and the Officer records a decision into `decisions`.

---

## 2. Tech Stack

- **Frontend**
  - React / Next.js.
  - Tailwind CSS (optional) for rapid styling.
  - Firebase Auth for authentication.
  - Deployed on Vercel, Firebase Hosting, or similar.

- **Backend**
  - Python 3.x + FastAPI.
  - Deployed on Google Cloud Run (containerized).

- **Data & Storage**
  - Firestore (NoSQL) as primary database.
  - Google Cloud Storage for images.

- **AI**
  - Google Gemini 2.5 Flash / Pro via Vertex AI / Gemini API.

- **Observability**
  - Cloud Logging / Stackdriver.
  - Basic metrics (requests, latency, error rates).

---

## 3. Data Model (Firestore Collections)

- `users`
  - `user_id`
  - `role` (`FIELD_AGENT|OFFICER|ADMIN`)
  - `name`
  - `region`
  - etc.

- `applications`
  - `application_id`
  - `borrower_id`
  - `business_profile` (embedded)
  - `loan_request` (amount, tenor, purpose)
  - `location` (village, region, coordinates if needed)
  - `status` (`new|scheduled|visited|scored|approved|rejected|need_more_detail`)
  - `assigned_field_agent_id` (nullable, filled when an application is booked/scheduled)
  - `scheduled_visit_at` (nullable, planned survey time)
  - `created_at`, `updated_at`
  - **Business rule:** when `assigned_field_agent_id` is set and `status = "scheduled"`, the application **must not** be scheduled by any other Field Agent; all scheduling operations must enforce this check (atomic/transactional).

- `surveys`
  - `application_id`
  - `field_agent_id`
  - `visit_notes`
  - `numeric_fields` (flexible map of numeric indicators)
  - `tags` (e.g. has_collateral, has_prior_default)
  - `photo_urls`
  - `visited_at`

- `scores`
  - `application_id`
  - `risk_score`
  - `risk_category`
  - `suggested_loan_limit`
  - `business_health_summary`
  - `top_reasons`
  - `risk_factors`
  - `supporting_evidence`
  - `mitigation_notes`
  - `visual_observations`
  - `consistency_checks`
  - `raw_ai_payload` (optional, trimmed for size)
  - `scored_at`

- `decisions`
  - `application_id`
  - `officer_id`
  - `decision` (`approved|rejected|need_more_detail`)
  - `final_loan_limit`
  - `officer_notes`
  - `ai_followed` (boolean)
  - `decided_at`

This schema makes the AI output rich and explainable, not just a single numeric score.

---

## 4. Logical Components (Backend Services)

At the backend level, the logic is organized into several modules (not necessarily separate microservices for the MVP):

1. **Auth Service**
   - Verifies Firebase ID tokens.
   - Extracts user identity & role.
   - Provides middleware for role-based access control.

2. **Application Service**
   - Handles application ingestion from the webhook.
   - Reads & lists applications for Officers and Admins.
   - Assigns / schedules applications to Field Agents.

3. **Survey Service**
   - Accepts survey payloads from Field Agents.
   - Validates and normalizes numeric fields.
   - Stores survey data in Firestore.

4. **Scoring Service (AI & Rules)**
   - Orchestrates the scoring flow:
     - Fetch relevant `applications` and `surveys`.
     - Build a prompt and call Gemini.
     - Post-process AI output into a structured **AI Insight Pack**.
   - Writes to the `scores` collection:
     - `risk_score`, `risk_category`, `suggested_loan_limit`
     - `business_health_summary`
     - `top_reasons`
     - `risk_factors`, `supporting_evidence`, `mitigation_notes`
     - `visual_observations`, `consistency_checks`
     - `raw_ai_payload` (trimmed) for audit/explainability.

5. **Decision Service**
   - Exposes endpoints for Officers to create and read decisions.
   - Ensures decisions are linked to a specific Officer and timestamp.
   - Optionally logs whether the Officer followed the AI recommendation.

6. **Notification / Status Service (lightweight for MVP)**
   - For now, primarily updates statuses in the dashboard UI.
   - Future: web push or email notifications when scoring or decisions are available.

---

## 5. Deployment Overview

- **Backend**
  - Dockerized FastAPI app.
  - Deployed to Cloud Run with automatic scaling based on HTTP traffic.

- **Frontend**
  - Next.js app deployed to Vercel / Firebase Hosting.
  - Uses environment variables for API base URL and Firebase config.

- **Firestore & Storage**
  - Single GCP project for the hackathon MVP.
  - Proper security rules restricted by Firebase Auth UID and roles (future work).

---

## 6. Directory Structure (Example)

```text
backend/
  app/
    main.py
    api/
      applications.py
      surveys.py
      scoring.py
      decisions.py
    models/
      application.py
      survey.py
      score.py
      decision.py
    services/
      scoring_service.py
      firestore_client.py
      gemini_client.py
      auth_service.py
    utils/
      datetime_utils.py
      id_generator.py
  tests/
    test_applications_api.py
    test_scoring_service.py
  Dockerfile
  cloudrun.yaml
```

This architecture description is the English counterpart of the Indonesian version and emphasizes:
- The **scheduling rule** for Field Agents.
- The rich **AI Insight Pack** stored in `scores`.
- The clear separation between ingestion, survey, scoring, and decision-making.

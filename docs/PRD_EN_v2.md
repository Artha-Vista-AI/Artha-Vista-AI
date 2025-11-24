# Artha Vista – Product Requirements Document (PRD, English)

Version: 0.2  
Owner: Beni Saprulah  
Last updated: 24 Nov 2025

---

## 1. Summary

Artha Vista is a **decision support tool** that helps microfinance institutions like Amartha assess rural MSME borrowers more quickly, more consistently, and more transparently.

The product combines:

- Borrower and business profile data from existing systems.
- Onsite visit data from Field Agents (notes, numeric fields, and photos).
- An AI Scoring Engine (Gemini 2.5) that produces a **risk score and an explainable insight pack**.

The output is used by human Officers to make final credit decisions. The goal is not to fully automate approvals but to **upgrade the quality and speed of human decisions**.

---

## 2. Background & Problem

Challenges in current credit assessment for rural MSMEs:

- Heavy dependence on Field Agent experience; quality varies between individuals and regions.
- Limited formal documentation (no financial statements, informal bookkeeping).
- Assessments are often stored in scattered notes and are hard to review later.
- Officers may not have time to deeply analyze every case, especially in peak periods.

This creates several risks:

- Inconsistent decisions for similar borrowers.
- Missed opportunities for good borrowers.
- Difficulty explaining or auditing decisions after the fact.

Artha Vista addresses these by:

- Centralizing data from applications, surveys, and photos.
- Using AI to turn raw data into structured, explainable risk insights.
- Providing dashboards for Officers and Field Agents that fit into existing workflows.

---

## 3. Product Goals & KPIs

### 3.1 Goals

1. **Speed**: Reduce time from completed survey to preliminary risk score and insights.
2. **Consistency**: Make assessments more standardized across agents and regions.
3. **Inclusion**: Enable better evaluation of borrowers without formal financial records.
4. **Explainability**: Provide clear reasoning and evidence for risk assessments.

### 3.2 Success Indicators (MVP)

- Average time from survey submission to score availability during the pilot.
- Qualitative feedback from Officers on usefulness of AI insights.
- Percentage of applications in the pilot that reach the **scored** state.
- Adoption by Field Agents:
  - Percentage of eligible visits done using the Artha Vista web app.

---

## 4. Scope

### 4.1 In-scope for MVP

1. **Application Integration**
   - Receive loan applications via a webhook from existing systems or a simulator.
   - Store application data in Firestore as `applications`.

2. **Field Agent Workflow**
   - Show a list of “new” applications in the agent’s region.
   - Allow the agent to **schedule** onsite visits (booking & lock to single agent).
   - Provide a survey form for:
     - Numeric inputs (revenue, profit, business age, etc.).
     - Free-text notes.
     - Photo URLs from Cloud Storage.

3. **AI Scoring Engine**
   - Combine application + survey + photos into a prompt.
   - Call Gemini and transform output into a `SCORE` record (AI Insight Pack).

4. **Officer Dashboard**
   - List of applications and their statuses/scores.
   - Detail page showing:
     - Borrower & business profile.
     - AI Insight Pack.
     - Photos.
   - Decision capture: approve / reject / need more detail.

5. **Basic Observability**
   - Logging of scoring requests and responses (with redacted payloads if needed).
   - Simple metrics (number of scored applications, average scoring time).

### 4.2 Out-of-scope for MVP

- Complex multi-stage workflows (reassignments, multi-officer approvals).
- Full integration with production Amartha core systems.
- Advanced retraining pipelines on historical data.

---

## 5. Personas

### 5.1 Field Agent

- Visits borrowers in rural areas.
- Collects information, explains products.
- Needs a tool that is fast, works on low bandwidth, and does not slow down visits.

### 5.2 Credit Officer

- Reviews applications and decides whether to approve or reject.
- Needs a clear snapshot of risk, reasons, and evidence.
- Needs to be able to explain their decision.

### 5.3 Borrower (Indirect User)

- Does not use Artha Vista directly.
- Benefits from faster, clearer, and more consistent decisions.

---

## 6. User Journey (End-to-End)

### 6.1 Borrower Journey

1. Submits a loan application via Amartha’s existing system.
2. Receives confirmation that a Field Agent will visit.
3. Is visited by a Field Agent, who collects information and photos.
4. Later receives the decision from Amartha.

### 6.2 Field Agent Journey

1. Logs into the Artha Vista web app (Firebase Auth).
2. Sees a list of applications with `status = "new"` in their region.
3. **Schedules an onsite visit**:
   - Opens a specific application and chooses date & time.
   - System updates the application:
     - `status = "scheduled"`
     - `assigned_field_agent_id = current agent`
     - `scheduled_visit_at = chosen time`
   - The application no longer appears as “new” to other Field Agents, preventing double visits.
4. On the visit day:
   - Opens the scheduled application.
   - Fills in the survey form:
     - Numeric fields (revenue, profit, business age, etc.).
     - Field notes (condition of business, customer flow, etc.).
     - Uploads 3–5 photos (shop, inventory, house, bookkeeping notebook if any).
5. Submits the survey:
   - Backend stores the survey and updates `status = "visited"`.
   - Backend triggers the scoring workflow.
6. Optionally, later the Field Agent can see a high-level highlight of the AI Insight Pack to support conversations with borrowers or Officers.

### 6.3 Officer Journey

1. Logs into the Officer dashboard.
2. Filters applications by status, region, or risk category.
3. Opens a scored application and sees:
   - Borrower & business profile.
   - AI Insight Pack (score, business health summary, reasons, risk factors, evidence, consistency checks).
   - Photos.
4. Makes a decision:
   - `Approve` – sets a final loan limit (can differ from suggested limit).
   - `Reject` – provides a structured reason.
   - `Need more detail` – indicates missing information (optionally triggers a manual follow-up).

---

## 7. Functional Requirements

### 7.1 Webhook Integration for Applications

- `POST /api/applications/webhook` must accept JSON payloads with:
  - Application ID, borrower info, business profile, loan request.
- On success:
  - The application is stored in Firestore with `status = "new"`.

### 7.2 Application & Status Management

- The system must track application status transitions:
  - `new` → `scheduled` → `visited` → `scored` → (`approved` or `rejected` or `need_more_detail`).
- Scheduling must enforce the **single-agent lock** rule:
  - Once `assigned_field_agent_id` is set and status is `scheduled`, other agents cannot book the same application.

### 7.3 Photo Handling

- Photos are uploaded to Cloud Storage via the frontend.
- Backend receives only `photo_urls` and stores them in the `surveys` collection.
- For the MVP, URLs may be public or signed; they must be reliably viewable in the dashboard.

### 7.4 Scoring Engine (Gemini + Rule-based Post-processing)

**Trigger**  

- When a complete survey has been submitted for an application.

**Input**  

- Application data:
  - Loan amount, tenor, purpose.
  - Business type, business age, revenue, profit.
- Survey data:
  - Numeric fields (e.g., estimated monthly revenue, estimated monthly profit, household size).
  - Field notes (free text, describing the business and environment).
  - Photo descriptions or labels (if provided).

**Processing**  

1. The backend builds a prompt for Gemini that includes:
   - A structured description of the borrower’s profile and business.
   - Field notes, clearly marked.
   - A summary of what each photo shows (if available).
2. Gemini returns:
   - A risk assessment, explanation, and suggestions in natural language.
3. A post-processing step converts the raw Gemini output into the `SCORE` schema.

**Output (Minimum for MVP)**  

- `risk_score` (0–100) and `risk_category` (`low`, `medium`, `high`).
- `suggested_loan_limit` and, optionally, a safe range suggestion.
- `business_health_summary`:
  - A paragraph summarizing business health, including revenue stability, customer base, inventory, and overall sustainability.
- `top_reasons`:
  - 3–7 bullet points outlining the main positive and negative drivers of the score.
- `risk_factors`:
  - Structured objects with `code`, `label`, `impact`, and `weight`, e.g.:
    - `SHORT_BUSINESS_AGE` (negative)
    - `STABLE_REVENUE` (positive)
    - `HIGH_EXISTING_DEBT` (negative)
- `supporting_evidence`:
  - Snippets from field notes or photo-based observations that justify each risk factor.
- `mitigation_notes`:
  - Concrete suggestions such as:
    - “Start with a lower limit than requested.”
    - “Re-evaluate after 3 on-time repayments.”
    - “Request additional documentation if debt is high.”
- `visual_observations`:
  - Items like store condition, inventory organization, visible business assets.
- `consistency_checks`:
  - Observations such as:
    - “Claimed revenue is consistent with inventory and observed customer flow.”
    - “Inventory seems low for the claimed revenue; risk flagged as `slightly_low`.”

All of these fields must be persisted in the `scores` collection and exposed via API for the Officer dashboard.

### 7.5 Officer Dashboard

- **List View**
  - Displays applications with columns:
    - Borrower name, village, region.
    - Status.
    - Risk score & category (if available).
  - Supports filters by status, region, and risk category.

- **Detail View**
  - Displays:
    - Borrower profile and business profile.
    - Full AI Insight Pack:
      - Score, category, suggested limit.
      - Business health summary.
      - Top reasons.
      - Risk factors & evidence.
      - Mitigation notes.
      - Visual observations.
      - Consistency checks.
    - Photos.
  - Provides a decision form:

    - `decision` (`approved`, `rejected`, `need_more_detail`)
    - `final_loan_limit` (numeric, required if approved)
    - `officer_notes` (string, required if rejected or need more detail)
    - `ai_followed` (boolean)

### 7.6 Notifications & Status Feedback

- At minimum, the FE must reflect status changes after major events:
  - After scheduling.
  - After survey submission.
  - After scoring.
  - After decision.
- Optional: show subtle banners or labels like “AI score ready” for scored applications.

---

## 8. Non-functional Requirements

- **Performance**
  - Target average scoring time under 10 seconds per application during the MVP demo.

- **Security**
  - Use Firebase Auth for authentication.
  - Enforce role-based authorization in the backend.
  - Avoid storing raw sensitive personal data in logs.

- **Reliability**
  - If the scoring call to Gemini fails, log the error and mark the application as `visited` but not `scored`.
  - Allow Officers to proceed with manual assessment in such cases.

- **Usability**
  - UI must be usable on mobile devices and low-bandwidth connections:
    - Minimal heavy assets.
    - Clear, text-first layouts.

---

This PRD describes a focused MVP where the core differentiator is the **AI Scoring & Insight Pack** and a **clean, lock-based Field Agent scheduling flow**. There is no runtime concept of agentic AI; the intelligence is encapsulated in a single, auditable scoring service.

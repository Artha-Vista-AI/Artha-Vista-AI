# Artha Vista – Product Requirements Document (PRD)
Version: 0.1  
Owner: Beni Saprulah  
Last updated: 24 Nov 2025

---

## 1. Summary

Artha Vista is a **decision support tool** that helps lending teams assess rural MSME (micro, small, and medium enterprise) borrowers faster, more consistently, and more inclusively.

The product combines:

- Borrower & business profile data
- Onsite visit data from Field Agents (notes + photos)
- AI analysis (Gemini + a simple baseline risk model)

The core output is an **AI risk score and insight pack** that helps Officers make better credit decisions, not a fully-automated approval engine.

---

## 2. Background & Problem

Amartha and similar institutions face several challenges when assessing rural MSME borrowers:

- Strong dependence on Field Agent experience and intuition.
- Quality of assessment varies between regions and agents.
- Many borrowers do not have formal financial records.

Current flows are often:

- Manual and fragmented.
- Hard to audit and explain.
- Time-consuming for Officers.

Artha Vista aims to introduce an AI-powered scoring and insight layer, while still keeping humans in the loop and respecting local context.

---

## 3. Product Goals & KPIs

### 3.1 Goals

1. **Speed up** credit assessment without sacrificing quality.
2. **Standardize** assessments across Field Agents and regions.
3. **Increase inclusion** for borrowers who are difficult to evaluate using formal methods only.
4. Provide **transparent explanations** to both borrowers and internal teams.

### 3.2 Success KPIs (MVP)

- Reduce time from survey completion to preliminary score availability.
- Officers report that AI insights are *useful* and *understandable* (qualitative feedback).
- High adoption:
  - % of Field Agents using the tool during visits.
  - % of applications with AI scores and summaries.

---

## 4. Scope

### 4.1 In-scope (MVP)

1. **Integration with loan application flow**
   - Receive loan applications from the existing system (via webhook or simulator).
   - Store data in Firestore.

2. **Field Agent flow**
   - Field Agent dashboard to see new applications in their region.
   - Ability to **schedule onsite visits** (booking / locking).
   - Simple survey form (numeric fields, notes, photos).

3. **AI scoring**
   - Combine structured data, notes, and photos.
   - Produce a risk score and insight pack using Gemini + light rule-based logic.

4. **Officer dashboard**
   - List of scored applications.
   - Detail page with AI insights and photos.
   - Decision actions: approve / reject / need more detail.

5. **Basic observability & logging**
   - Logs for AI calls and decisions to support the hackathon demo.

### 4.2 Out-of-scope (MVP)

- Real-time integration with production Amartha systems.
- Advanced model training or custom ML pipelines.
- Full role-based access control and security hardening (beyond basic Firebase auth).
- Complex workflow management (reassigning, escalations, etc.).

---

## 5. Personas

### 5.1 Field Agent

- Visits borrowers on-site.
- Collects qualitative and quantitative information.
- Needs a tool that is simple, works on mobile, and does not slow them down.

### 5.2 Credit Officer

- Reviews applications and makes final decisions.
- Needs a quick snapshot: risk score, reasons, and evidence.
- Needs to be able to explain decisions to internal stakeholders and borrowers.

### 5.3 Borrower (indirect user)

- Does not directly use the tool.
- Affected by faster and more transparent decisions.

---

## 6. User Journey (End-to-End)

### 6.1 Borrower

1. Applies for a loan through Amartha’s existing channels.
2. Receives confirmation that a Field Agent will visit.
3. Waits for the visit, then later for the final decision.

### 6.2 Field Agent

1. Logs into the Artha Vista dashboard (React + Firebase Auth).
2. Sees a list of applications with status `new` in their region.
3. **Schedules the onsite visit (schedule / book):**
   - The Field Agent selects an application and chooses a date/time for the visit.
   - The system updates the application status to `scheduled` and fills `assigned_field_agent_id` with that agent.
   - Applications with status `scheduled` **no longer appear** in other Field Agents’ lists (prevents double visits).
4. On the scheduled day, the Field Agent visits the borrower:
   - Fills out a short survey form (revenue, profit, business age, etc.).
   - Writes visit notes.
   - Uploads 3–5 photos (shop, inventory, house, etc.).
5. Submits the visit → backend stores data in Firestore and sends a scoring request to the AI layer.
6. Once scoring is complete, the Field Agent can optionally see a high-level highlight to help explain things to the borrower or to discuss with an Officer.

### 6.3 Officer

1. Logs into the Officer dashboard.
2. Views a list of applications that already have scores.
3. Opens an application detail page:
   - Borrower & business profile.
   - AI score and risk category.
   - AI insight pack (reasoning, evidence, consistency checks).
   - Key photos.
4. Chooses an action:
   - `Approve` → set final loan limit.
   - `Reject` → must provide a reason.
   - `Need More Detail` → sends the case back for additional information (manual step).

---

## 7. Functional Requirements

### 7.1 Application Integration & Webhook

- System must accept application payloads via `POST /api/applications/webhook`.
- Minimal validation: required fields must be present and properly typed.
- Each payload becomes a document in the `applications` collection.

### 7.2 Data Management in Firestore

- Applications, surveys, scores, and decisions are stored in separate collections.
- Status transitions for applications:
  - `new` → `scheduled` → `visited` → `scored` → (`approved` or `rejected` or `need_more_detail`).
- Status updates should be consistent and traceable.

### 7.3 Photo Upload (Cloud Storage)

- Field Agent uploads photos via the FE (Firebase Storage / Cloud Storage SDK).
- Backend only stores `photo_urls` in Firestore.
- For the MVP, public or signed URLs are acceptable (as long as they work for the demo).

### 7.4 Scoring Engine (Gemini + Rule-based)

- **Trigger:**
  - When a Field Agent submits a complete survey.

- **Input to AI:**
  - Business profile data (numeric & categorical).
  - Visit notes (free text).
  - Photo URLs from Cloud Storage.

- **Output (minimum for MVP):**
  - `risk_score` (0–100) and `risk_category` (Low / Medium / High).
  - `suggested_loan_limit` plus a safe range note (e.g., “recommended 4.5–5M IDR”).
  - `summary`: 3–5 sentence summary of the business and borrower situation.
  - `top_reasons`: 3–7 key positive/negative factors explaining the score.
  - `business_health_summary`: dedicated paragraph describing business health (revenue stability, supplier dependency, customer base, etc.).
  - `risk_factors`: structured list of factors with codes, labels, and direction of impact (e.g., `SHORT_BUSINESS_AGE`, `STABLE_REVENUE`, `HIGH_EXISTING_DEBT`).
  - `supporting_evidence`: excerpts from notes and photo descriptions used by the AI as evidence (e.g., “shelves almost empty for all fast-moving SKUs”).
  - `mitigation_notes`: suggested actions Officers can take (e.g., start with a smaller limit, request additional documents, closely monitor early repayments).
  - `visual_observations`: insights derived from photos (shop/house condition, inventory organization, presence of business equipment).
  - `consistency_checks`: comparisons between numeric data and onsite observations (e.g., claimed revenue vs. inventory, claimed customer flow vs. visual evidence).

All of this is stored in the `scores` entity/collection and displayed as an **AI Insights** panel in the Officer dashboard, so that Officers see not just a score but a clear, auditable explanation.

### 7.5 Officer Dashboard

- **List view:**
  - Filter by status, score, and region.
  - Show key badge indicators (e.g., risk category, whether AI was followed).

- **Detail view:**
  - Borrower & business profile.
  - AI score & category.
  - AI insight pack (summary, reasons, factors, evidence, consistency checks).
  - Photos.
  - Decision form (approve / reject / need more detail).

### 7.6 Notifications (MVP)

- Minimal:
  - Status updates in the FE dashboards (no fancy push needed).
- Nice-to-have:
  - Web push or email for key events (scoring done, application needs more details, etc.).

---

## 8. Non-functional Requirements

- **Performance**
  - Target scoring latency < 10 seconds per application (for the hackathon demo).

- **Security**
  - Firebase Auth for user authentication.
  - Basic role-based control in the backend.

- **Reliability**
  - If the AI service fails, the system should log errors and mark the scoring attempt as failed, without blocking the whole application flow.

- **Usability**
  - FE should be mobile-friendly and usable in rural network conditions (simple UI, minimal heavy assets).

---

This English PRD mirrors the original Indonesian one, with two key enhancements emphasized:

1. **AI Scoring is not just a number** – it produces a rich, explainable insight pack that helps Officers reason about risk.
2. **Field Agents must schedule visits first** – applications are locked to one agent once scheduled, preventing double visits and clarifying responsibilities.

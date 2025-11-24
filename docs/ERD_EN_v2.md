# Vibe Code – ERD (MVP, English)

This ERD describes the main entities for the Vibe Code / Artha Vista MVP.

- `USER` – Field Agent, Officer, Admin
- `BORROWER` and `BUSINESS_PROFILE`
- `APPLICATION` – loan request
- `SURVEY` – onsite visit result
- `SCORE` – AI scoring & insight pack
- `DECISION` – Officer decision

```mermaid
erDiagram
  USER {
    string user_id PK
    string name
    string role
    string region
  }

  BORROWER {
    string borrower_id PK
    string name
    string phone
    string village
    string region
  }

  BUSINESS_PROFILE {
    string business_profile_id PK
    string borrower_id FK
    string business_type
    int business_age_months
    float monthly_revenue
    float monthly_profit
  }

  APPLICATION {
    string application_id PK
    string borrower_id FK
    string business_profile_id FK
    float requested_amount
    int tenor_months
    string purpose
    string status
    string assigned_field_agent_id
    datetime scheduled_visit_at
    datetime created_at
    datetime updated_at
  }

  SURVEY {
    string survey_id PK
    string application_id FK
    string field_agent_id FK
    string visit_notes
    json numeric_fields
    json tags
    json photo_urls
    datetime visited_at
  }

  SCORE {
    string score_id PK
    string application_id FK
    float risk_score
    string risk_category
    float suggested_loan_limit
    string business_health_summary
    json top_reasons
    json risk_factors
    json supporting_evidence
    json mitigation_notes
    json visual_observations
    json consistency_checks
    json raw_ai_payload
    datetime scored_at
  }

  DECISION {
    string decision_id PK
    string application_id FK
    string officer_id FK
    string decision
    float final_loan_limit
    string officer_notes
    bool ai_followed
    datetime decided_at
  }

  %% Relationships
  BORROWER ||--o{ BUSINESS_PROFILE : "owns"
  BORROWER ||--o{ APPLICATION : "applies_for"
  BUSINESS_PROFILE ||--o{ APPLICATION : "used_for"
  USER ||--o{ APPLICATION : "assigned_field_agent"
  USER ||--o{ SURVEY : "submits"
  APPLICATION ||--o{ SURVEY : "has"
  APPLICATION ||--o{ SCORE : "has"
  USER ||--o{ DECISION : "makes"
  APPLICATION ||--o{ DECISION : "has"
```

## SCORE Entity as AI Insight Pack

The `SCORE` entity is designed to store a complete AI Insight Pack, not just a number:

- **Core metrics**
  - `risk_score` – normalized score 0–100.
  - `risk_category` – `low`, `medium`, or `high`.
  - `suggested_loan_limit` – recommended loan amount.

- **Narrative summary**
  - `business_health_summary` – paragraph describing revenue stability, inventory, customer base, etc.
  - `top_reasons` – bullet-point reasons that explain the score (mix of positive & negative drivers).

- **Structured factors & evidence**
  - `risk_factors` – encoded factors with `code`, `label`, `impact`, and `weight`.
  - `supporting_evidence` – snippets from field notes and photo descriptions used by the AI.

- **Mitigation & observations**
  - `mitigation_notes` – concrete suggestions for Officers (e.g., start with lower limit, monitor first cycles).
  - `visual_observations` – structured observations derived from photos (store condition, equipment, etc.).

- **Consistency checks**
  - `consistency_checks` – comparison between numeric data (revenue, inventory, etc.) and onsite observations.

This makes `SCORE` the central place for **explainable AI scoring**, usable in Officer dashboards, audits, and potential retraining in the future.

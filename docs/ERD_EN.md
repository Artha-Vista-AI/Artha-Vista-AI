# Vibe Code – ERD (MVP, English Version)

This diagram shows the main entities in the Vibe Code / Artha Vista MVP:

- User (Field Agent, Officer, Admin)
- Borrower & Business Profile
- Application (loan request)
- Survey (onsite visit result)
- Score (AI scoring & analysis)
- Decision (Officer decision)

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

## Notes on the SCORE Entity (AI Insight Pack)

The `SCORE` entity does not only store a numeric risk score; it also contains a full AI-generated insight pack produced by Gemini, including:

- `risk_score` and `risk_category` (Low / Medium / High).
- `suggested_loan_limit` – recommended loan amount for the borrower.
- `business_health_summary` – a narrative summary of business health (revenue stability, customer diversity, etc.).
- `top_reasons` – key positive and negative drivers that explain the score.
- `risk_factors` – structured list of risk factors (code, label, direction of impact).
- `supporting_evidence` – snippets from visit notes and descriptions of photos used by the AI as evidence.
- `mitigation_notes` – suggested follow-up or risk mitigation steps for the Officer.
- `visual_observations` – insights extracted from photos (store/house condition, inventory organization, etc.).
- `consistency_checks` – checks between numeric data and onsite observations (e.g., revenue vs. inventory, claims vs. visual proof).

Because of this, the `SCORE` table acts as the center of **explainable AI scoring**, instead of just being a place to store a single 0–100 number.

# Vibe Code – ERD (MVP)

Diagram ini menggambarkan relasi utama antara entity dalam sistem Vibe Code untuk MVP:
- User (Field Agent, Officer, Admin)
- Borrower & Business Profile
- Application (pengajuan)
- Survey (hasil kunjungan lapangan)
- Score (hasil AI scoring & analysis)
- Decision (keputusan officer)

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
    string district
    string province
  }

  BUSINESS_PROFILE {
    string business_profile_id PK
    string borrower_id FK
    string business_name
    string business_type
    int business_age_months
    int monthly_revenue
    int monthly_profit
  }

  APPLICATION {
    string application_id PK
    string borrower_id FK
    string business_profile_id FK
    int loan_amount_requested
    int tenor_months
    string purpose
    string status
    string assigned_field_agent_id FK
    datetime scheduled_visit_at
    datetime created_at
    datetime updated_at
  }

  SURVEY {
    string survey_id PK
    string application_id FK
    string field_agent_id FK
    text visit_notes
    json numeric_fields
    json tags
    json photo_urls
    datetime visited_at
  }

  SCORE {
    string score_id PK
    string application_id FK
    int risk_score
    string risk_category
    int suggested_loan_limit
    text summary
    json top_reasons
    text business_health_summary
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
    int final_loan_limit
    text officer_notes
    bool ai_followed
    datetime decided_at
  }

  %% Relationships
  BORROWER ||--o{ BUSINESS_PROFILE : "owns"
  BORROWER ||--o{ APPLICATION : "applies"
  BUSINESS_PROFILE ||--o{ APPLICATION : "used_for"
  USER ||--o{ APPLICATION : "assigned_field_agent"
  USER ||--o{ SURVEY : "submits"
  APPLICATION ||--o{ SURVEY : "has"
  APPLICATION ||--o{ SCORE : "has"
  USER ||--o{ DECISION : "makes"
  APPLICATION ||--o{ DECISION : "has"
```


## Catatan Entity SCORE (AI Insight Pack)

Entity `SCORE` tidak hanya menyimpan angka skor risiko, tetapi juga paket insight lengkap yang dihasilkan oleh Gemini, antara lain:

- `risk_score` dan `risk_category` (Low / Medium / High).
- `suggested_loan_limit` yang direkomendasikan untuk borrower.
- `business_health_summary` yang merangkum kesehatan usaha (stabilitas omzet, keragaman pelanggan, dsb).
- `risk_factors` terstruktur (kode faktor, label, arah pengaruh).
- `supporting_evidence` dari catatan kunjungan & foto lapangan.
- `mitigation_notes` berisi saran tindak lanjut bagi Officer.
- `visual_observations` hasil analisis foto (kondisi warung/rumah, kerapihan stok, dll).
- `consistency_checks` untuk melihat konsistensi antara data numerik dan observasi lapangan.

Dengan begitu, tabel `SCORE` berperan sebagai pusat **explainable AI scoring**, bukan sekadar menyimpan angka 0–100.

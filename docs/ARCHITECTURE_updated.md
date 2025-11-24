# Vibe Code – Architecture & Tech Stack

---

## 1. High-level Architecture

Komponen utama:

1. **Borrower App (existing / dummy)**
   - Mengirimkan pengajuan pinjaman ke endpoint backend `POST /api/applications/webhook`.

2. **Field Agent Web App (Vibe Code FE)**
   - Built with React/Next.js.
   - Login via Firebase Auth.
   - Fitur:
     - List aplikasi yang perlu dikunjungi.
     - Form survey & upload foto.
     - Lihat status skor (opsional).

3. **Agent Officer Web App**
   - Bisa jadi satu aplikasi dengan Field Agent (role-based UI).
   - Fitur:
     - List aplikasi dengan filter skor/status.
     - View detail skor & summary.
     - Masukkan keputusan.

4. **Backend API (Vibe Code API)**
   - Python 3.x + FastAPI.
   - Berjalan di Google Cloud Run.
   - Tanggung jawab:
     - Verifikasi Firebase token.
     - Orkestrasi agent (Orchestrator, DataIngestion, Scoring, Explanation, Feedback).
     - CRUD untuk applications / surveys / scores / decisions.
     - Integrasi ke Gemini API.
     - Integrasi ke Firestore & Cloud Storage.

5. **Data Layer**
   - **Firestore** sebagai primary database (NoSQL, document).
   - **Cloud Storage** untuk menyimpan foto.

6. **AI Layer**
   - Gemini 2.5 Flash (multimodal).
   - Dipanggil dari backend via HTTP (REST API).

7. **Observability**
   - Google Cloud Logging.
   - Cloud Trace / Error Reporting.

---

## 2. Tech Stack & Tools

### Backend

- Bahasa: **Python 3.11+**
- Framework: **FastAPI**
- Schema / validation: **Pydantic**
- HTTP client: `httpx` atau `requests`
- Google Cloud SDK:
  - `google-cloud-firestore`
  - `google-cloud-storage`
- Auth:
  - `firebase-admin` (verify ID token)
- Testing: `pytest`
- Container: Docker, deploy ke Cloud Run

### Frontend

- **Next.js 15** (React 19) atau React + Vite
- Bahasa: TypeScript
- Styling: Tailwind CSS
- UI kit (optional): shadcn/ui
- State/data fetching: React Query / TanStack Query
- Auth: Firebase Auth (SDK)
- Storage upload (opsional): Firebase Storage SDK atau direct upload ke GCS via signed URL.

### DevOps & Infra

- Source control: GitHub / GitLab
- CI/CD: GitHub Actions -> build & deploy ke Cloud Run + Firebase Hosting
- IaC (opsional, kalau sempat): Terraform untuk:
  - Cloud Run service
  - Firestore database
  - Storage bucket
  - IAM & service account

---

## 3. Data Model (Firestore Collections)

- `users`
  - `user_id`
  - `role` (`FIELD_AGENT|OFFICER|ADMIN`)
  - `name`, `region`, dst.

- `applications`
  - `application_id`
  - `borrower_id`
  - `business_profile` (embedded)
  - `loan_request` (amount, tenor, purpose)
  - `location`
  - `status` (`new|scheduled|visited|scored|approved|rejected|need_more_detail`)
  - `assigned_field_agent_id` (nullable, diisi ketika aplikasi di-book / dijadwalkan)
  - `scheduled_visit_at` (nullable, waktu survei yang direncanakan)
  - `created_at`, `updated_at`
  - **Business rule:** ketika `assigned_field_agent_id` terisi dan `status = scheduled`, aplikasi tidak boleh di-schedule oleh Field Agent lain; semua operasi schedule harus melakukan cek kondisi ini.

- `surveys`
  - `application_id`
  - `field_agent_id`
  - `visit_notes`
  - `numeric_fields`
  - `tags`
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
  - `raw_ai_payload` (opsional, trimmed)
  - `scored_at`

- `decisions`
  - `application_id`
  - `officer_id`
  - `decision`
  - `final_loan_limit`
  - `officer_notes`
  - `ai_followed` (bool)
  - `decided_at`

---

## 4. Logical Components (Backend Services)

Di level backend, arsitektur logika disusun sebagai beberapa **service / modul** biasa (bukan agent LLM):

- **Auth Service**
  - Verifikasi Firebase ID token.
  - Ekstrak informasi user & role.
  - Middleware untuk proteksi endpoint.

- **Application Service**
  - CRUD untuk `applications`.
  - Endpoint webhook eksternal (`/api/applications/webhook`).

- **Survey Service**
  - CRUD untuk `surveys` (terutama create dari Field Agent).
  - Validasi konsistensi dengan `applications` (aplikasi harus ada, status valid).

- **Scoring Service (AI & Rules)**
  - Mengambil data `application` + `survey` dari Firestore.
  - Menyiapkan feature terstruktur + konteks text + (opsional) URL foto.
  - Memanggil Gemini Flash untuk menghasilkan:
    - `risk_category`
    - `suggested_loan_limit`
    - `summary`
    - `business_health_summary`
    - `risk_factors`
    - `supporting_evidence`
    - `mitigation_notes`
  - Menyimpan hasil ke collection `scores`.

- **Decision Service**
  - Menerima keputusan dari Officer (`approve/reject/need_more_detail`).
  - Update status di `applications`.
  - Menulis record ke `decisions` dan flag apakah keputusan mengikuti rekomendasi AI atau tidak.

- **Reference Data Service**
  - Menyajikan data referensi (jenis usaha, dsb).

- **Logging & Audit**
  - Menulis log ke Cloud Logging.
  - Mencatat siapa mengubah apa & kapan (untuk kebutuhan future audit).
## 5. Folder Structure (Monorepo)

```text
vibecode/
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── api/
│   │   │   ├── routes_applications.py
│   │   │   ├── routes_surveys.py
│   │   │   ├── routes_scores.py
│   │   │   ├── routes_decisions.py
│   │   │   └── routes_health.py
│   │   ├── core/
│   │   │   ├── config.py
│   │   │   ├── security.py    # Firebase token verify, role extraction
│   │   │   ├── logging.py
│   │   │   └── exceptions.py
│   │   ├── models/            # Firestore models (Python classes)
│   │   │   ├── application_model.py
│   │   │   ├── survey_model.py
│   │   │   ├── score_model.py
│   │   │   └── decision_model.py
│   │   ├── schemas/           # Pydantic request/response schemas
│   │   │   ├── application_schema.py
│   │   │   ├── survey_schema.py
│   │   │   ├── score_schema.py
│   │   │   └── decision_schema.py
│   │   ├── services/
│   │   │   ├── firestore_service.py
│   │   │   ├── storage_service.py
│   │   │   ├── gemini_client.py
│   │   │   └── auth_service.py
│   │   ├── agents/
│   │   │   ├── orchestrator_agent.py
│   │   │   ├── data_ingestion_agent.py
│   │   │   ├── scoring_agent.py
│   │   │   ├── explanation_agent.py
│   │   │   └── feedback_logging_agent.py
│   │   └── utils/
│   │       ├── datetime_utils.py
│   │       └── id_generator.py
│   ├── tests/
│   │   ├── test_applications_api.py
│   │   ├── test_scoring_agent.py
│   │   └── test_explanation_agent.py
│   ├── pyproject.toml
│   ├── Dockerfile
│   └── README.md
│
├── frontend/
│   ├── app/                    # Next.js App Router
│   │   ├── layout.tsx
│   │   ├── page.tsx            # Landing / login
│   │   ├── field-agent/
│   │   │   ├── page.tsx        # list assigned applications
│   │   │   └── [applicationId]/page.tsx  # survey form
│   │   ├── officer/
│   │   │   ├── page.tsx        # list applications
│   │   │   └── [applicationId]/page.tsx  # detail + decision
│   │   └── api/                # Next.js API routes (opsional)
│   ├── components/
│   │   ├── layout/
│   │   ├── forms/
│   │   └── cards/
│   ├── lib/
│   │   ├── api-client.ts       # wrapper fetch ke backend (dengan token)
│   │   └── firebase.ts         # init Firebase SDK
│   ├── hooks/
│   │   ├── useAuth.ts
│   │   └── useApplications.ts
│   ├── styles/
│   │   └── globals.css
│   ├── public/
│   ├── package.json
│   ├── next.config.mjs
│   └── tailwind.config.cjs
│
├── infra/
│   ├── terraform/
│   │   ├── main.tf
│   │   ├── cloud_run.tf
│   │   ├── firestore.tf
│   │   └── storage.tf
│   ├── cloudbuild.yaml         # atau github-actions workflows
│   ├── firestore.rules
│   └── storage.rules
│
├── docs/
│   ├── PRD.md
│   ├── AGENT.md
│   ├── API_DOCS.md
│   └── ARCHITECTURE.md
│
└── README.md
```

---

## 6. Alur Deployment (High-level)

1. Developer push ke branch `main` / `staging`.
2. CI:
   - Running unit test (backend & frontend).
   - Build Docker image untuk backend -> push ke Container Registry.
   - Deploy ke Cloud Run (staging/prod).
   - Build frontend -> deploy ke Firebase Hosting / Vercel.
3. Secrets (API key Gemini, service account):
   - Disimpan di Google Secret Manager.
   - Diinject ke Cloud Run sebagai environment variable.

---

## 7. Security & Compliance (MVP level)

- Semua API HTTPS only.
- Firebase ID token diverifikasi di backend.
- Access kontrol ketat berdasarkan role:
  - Field Agent hanya boleh mengakses aplikasi yang di-assign.
  - Officer hanya pada area/wilayah yang di-mapping (bisa pakai field `region` di user).
- Data borrower tidak pernah keluar dari environment GCP kecuali log yang sudah dianonimkan.

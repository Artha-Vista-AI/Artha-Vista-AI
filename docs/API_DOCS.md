# Vibe Code – API Documentation (MVP)

Auth:  
- Semua request (kecuali `/health`) pakai **Firebase ID Token** di header:
  - `Authorization: Bearer <firebase_id_token>`
- Backend verifikasi token via Firebase Admin SDK, dan menentukan role:
  - `ROLE_FIELD_AGENT`
  - `ROLE_OFFICER`
  - `ROLE_ADMIN` (opsional, buat internal ops)

Base URL (contoh):
- Staging: `https://api-staging.vibecode.app`
- Prod: `https://api.vibecode.app`

---

## 1. Health Check

### GET `/health`

**Description:**  
Cek apakah service hidup dan bisa konek ke dependency utama.

**Auth:**  
Public (tanpa token, untuk liveness probe).

**Response 200:**
```json
{
  "status": "ok",
  "service": "vibecode-api",
  "version": "0.1.0",
  "dependencies": {
    "firestore": "ok",
    "storage": "ok",
    "gemini": "ok"
  }
}
```

---

## 2. Applications

### 2.1 Ingest Application (Webhook dari sistem eksternal)

### POST `/api/applications/webhook`

**Description:**  
Dipanggil oleh sistem peminjam (misal: Amartha / dummy borrower app) ketika ada pengajuan pinjaman baru.

**Auth:**  
- Option A (MVP): API key di header, misal `X-API-Key: <key>`
- Option B (future): signed JWT / HMAC signature.

**Request Body (example):**
```json
{
  "application_id": "APP-2025-0001",
  "borrower_id": "BRW-123",
  "business_profile": {
    "business_name": "Warung Bu Siti",
    "business_type": "retail",
    "business_age_months": 18,
    "monthly_revenue": 8000000,
    "monthly_profit": 2500000
  },
  "loan_request": {
    "loan_amount_requested": 5000000,
    "tenor_months": 12,
    "purpose": "Tambah stok"
  },
  "location": {
    "village": "Desa A",
    "district": "Kecamatan B",
    "province": "Jawa Barat"
  }
}
```

**Response 201:**
```json
{
  "id": "APP-2025-0001",
  "status": "new"
}
```

---

### 2.2 List Applications (Officer & Admin)

### GET `/api/applications`

**Description:**  
List aplikasi dengan filter status, skor, wilayah.

**Auth:**  
- `ROLE_OFFICER` atau `ROLE_ADMIN`.

**Query Params (optional):**
- `status` = `new|visited|scored|approved|rejected|need_more_detail`
- `risk_category` = `low|medium|high`
- `region` (string)
- `limit` (default 20)
- `offset` atau `page_token`

**Response 200:**
```json
{
  "items": [
    {
      "application_id": "APP-2025-0001",
      "borrower_id": "BRW-123",
      "status": "scored",
      "risk_category": "medium",
      "risk_score": 62,
      "loan_amount_requested": 5000000,
      "suggested_loan_limit": 4500000,
      "created_at": "2025-11-24T12:00:00Z"
    }
  ],
  "next_page_token": null
}
```

---

### 2.3 Get Application Detail

### GET `/api/applications/{application_id}`

**Auth:**  
- `ROLE_FIELD_AGENT` (hanya jika assigned ke dia)
- `ROLE_OFFICER`
- `ROLE_ADMIN`

**Response 200:**
```json
{
  "application": {
    "application_id": "APP-2025-0001",
    "borrower_id": "BRW-123",
    "status": "scored",
    "loan_request": { },
    "business_profile": { },
    "location": { },
    "timestamps": {
      "created_at": "2025-11-24T12:00:00Z",
      "visited_at": "2025-11-24T13:00:00Z",
      "scored_at": "2025-11-24T13:05:00Z"
    }
  },
  "survey": {
    "visit_notes": "Usaha sudah berjalan konsisten, stok rapi...",
    "numeric_fields": {
      "inventory_value": 4000000,
      "household_size": 4
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
    "top_reasons": [
      "Omzet stabil 6 bulan terakhir",
      "Bisnis masih < 2 tahun"
    ],
    "summary": "Warung sembako dengan omzet stabil, namun usia usaha masih menengah..."
  },
  "decision": {
    "status": "approved",
    "final_loan_limit": 4500000,
    "officer_id": "OFF-1",
    "officer_notes": "Ikuti rekomendasi AI",
    "decided_at": "2025-11-24T13:10:00Z"
  }
}
```

---

## 3. Survey (Field Agent)

### 3.1 List Assigned Applications (Field Agent)

### GET `/api/field/assigned-applications`

**Auth:**  
- `ROLE_FIELD_AGENT`.

**Response 200:**
```json
{
  "items": [
    {
      "application_id": "APP-2025-0001",
      "borrower_name": "Siti",
      "village": "Desa A",
      "status": "new"
    }
  ]
}
```

---

### 3.2 Submit Survey & Photos

### POST `/api/applications/{application_id}/survey`

**Auth:**  
- `ROLE_FIELD_AGENT` (yang assigned ke aplikasi tersebut).

**Description:**  
Field Agent submit hasil kunjungan + URL foto (upload via Firebase/Storage SDK, backend hanya simpan URL).

**Request Body:**
```json
{
  "visit_notes": "Warung ramai, stok rapi, pelanggan tetap cukup banyak.",
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

**Behavior:**
- Simpan survey di Firestore (`surveys` collection).
- Update status aplikasi → `visited`.
- Trigger OrchestratorAgent → DataIngestionAgent → ScoringAgent → ExplanationAgent.

**Response 202:**
```json
{
  "status": "accepted",
  "next": "scoring_in_progress"
}
```

---

## 4. Scoring

### 4.1 Get Score

### GET `/api/applications/{application_id}/score`

**Auth:**  
- `ROLE_FIELD_AGENT` (optional)
- `ROLE_OFFICER`
- `ROLE_ADMIN`

**Response 200 (if available):**
```json
{
  "status": "ready",
  "score": {
    "risk_score": 62,
    "risk_category": "medium",
    "suggested_loan_limit": 4500000,
    "top_reasons": [ "..." ],
    "summary": "..."
  }
}
```

**Response 202 (if not ready):**
```json
{
  "status": "pending"
}
```

---

### 4.2 Trigger Re-score (Admin/Officer)

### POST `/api/applications/{application_id}/rescore`

**Auth:**  
- `ROLE_OFFICER` atau `ROLE_ADMIN`.

**Description:**  
Untuk kasus data di-update dan butuh re-skor.

**Response 202:**
```json
{
  "status": "rescore_requested"
}
```

---

## 5. Decisions (Officer)

### 5.1 Submit Decision

### POST `/api/applications/{application_id}/decision`

**Auth:**  
- `ROLE_OFFICER`.

**Request Body:**
```json
{
  "decision": "approved", 
  "final_loan_limit": 4500000,
  "officer_notes": "Ikuti rekomendasi skor, potensi bagus.",
  "reason_tags": [
    "follow_ai_recommendation"
  ]
}
```

**Decision enum:**
- `approved`
- `rejected`
- `need_more_detail`

**Behavior:**
- Simpan keputusan + alasan.
- Update status aplikasi (`approved|rejected|need_more_detail`).
- Panggil FeedbackLoggingAgent untuk simpan info override/align dengan AI.

**Response 200:**
```json
{
  "application_id": "APP-2025-0001",
  "status": "approved"
}
```

---

## 6. Reference Data

### GET `/api/reference/business-types`

**Description:**  
List jenis usaha untuk dipakai di form.

**Response 200:**
```json
{
  "items": [
    { "code": "retail", "label": "Warung / Retail" },
    { "code": "fnb", "label": "Makanan & Minuman" },
    { "code": "service", "label": "Jasa" }
  ]
}
```

---

## 7. Error Format

Semua error gunakan format:

```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "business_age_months must be >= 0",
    "details": null
  }
}
```

Contoh `code`:
- `UNAUTHORIZED`
- `FORBIDDEN`
- `NOT_FOUND`
- `INVALID_REQUEST`
- `INTERNAL_ERROR`

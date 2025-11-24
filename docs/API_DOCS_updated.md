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

    "business_health_summary": "Usaha stabil, stok rapi, omzet konsisten 6 bulan terakhir.",
    "top_reasons": [
      "Omzet relatif stabil dalam 6 bulan terakhir",
      "Usaha masih < 2 tahun sehingga risiko moderat",
      "Tidak ada pinjaman aktif lain yang besar"
    ],

    "risk_factors": [
      {
        "code": "SHORT_BUSINESS_AGE",
        "label": "Usaha relatif baru",
        "impact": "negative",
        "weight": 0.25
      },
      {
        "code": "STABLE_REVENUE",
        "label": "Omzet stabil",
        "impact": "positive",
        "weight": 0.35
      }
    ],

    "supporting_evidence": [
      {
        "source": "FIELD_NOTE",
        "snippet": "Warung ramai di jam sibuk, pelanggan tetap cukup banyak."
      },
      {
        "source": "PHOTO",
        "description": "Rak stok tersusun rapi, barang fast moving tersedia."
      }
    ],

    "mitigation_notes": [
      "Mulai dengan limit sedikit di bawah pengajuan awal.",
      "Pantau pembayaran 2 bulan pertama sebelum menaikkan limit."
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
        "details": "Estimasi lapangan sejalan dengan profil di aplikasi."
      },
      {
        "field": "inventory_value",
        "status": "slightly_low",
        "details": "Nilai stok sedikit di bawah klaim omzet, perlu dipantau."
      }
    ]
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

**Catatan:**
- `status` dapat bernilai `new|scheduled|visited|scored|approved|rejected|need_more_detail`.
- Jika `status = "scheduled"`, response juga menyertakan field `scheduled_visit_at` sehingga Field Agent bisa melihat jadwal kunjungan yang sudah di-book.


---


### 3.2 Schedule Visit (Book Application)

### POST `/api/field/applications/{application_id}/schedule`

**Auth:**  
- `ROLE_FIELD_AGENT`.

**Description:**  
Field Agent melakukan **booking**/scheduling kunjungan. Setelah aplikasi di-schedule oleh satu Field Agent, aplikasi tersebut **tidak bisa di-schedule** oleh Field Agent lain (kecuali di-unassign oleh Officer/Admin).

**Request Body:**
```json
{
  "scheduled_visit_at": "2025-11-25T09:00:00Z"
}
```

**Behavior:**
- Hanya bisa dipanggil jika:
  - `status` aplikasi = `new` **dan**
  - `assigned_field_agent_id` masih kosong.
- Secara atomik update:
  - `status` → `scheduled`
  - `assigned_field_agent_id` → `current_user_id`
  - `scheduled_visit_at` → dari request.
- Jika aplikasi sudah di-assign ke agent lain, return **409 Conflict**.

**Response 200:**
```json
{
  "application_id": "APP-2025-0001",
  "status": "scheduled",
  "assigned_field_agent_id": "FA-123",
  "scheduled_visit_at": "2025-11-25T09:00:00Z"
}
```

**Response 409 (sudah di-book orang lain):**
```json
{
  "error": {
    "code": "ALREADY_ASSIGNED",
    "message": "Application already scheduled by another field agent.",
    "details": null
  }
}
```

### 3.3 Submit Survey & Photos

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

    "business_health_summary": "Usaha stabil, stok rapi, omzet konsisten 6 bulan terakhir.",
    "top_reasons": [
      "Omzet relatif stabil dalam 6 bulan terakhir",
      "Usaha masih < 2 tahun sehingga risiko moderat",
      "Tidak ada pinjaman aktif lain yang besar"
    ],

    "risk_factors": [
      {
        "code": "SHORT_BUSINESS_AGE",
        "label": "Usaha relatif baru",
        "impact": "negative",
        "weight": 0.25
      },
      {
        "code": "STABLE_REVENUE",
        "label": "Omzet stabil",
        "impact": "positive",
        "weight": 0.35
      }
    ],

    "supporting_evidence": [
      {
        "source": "FIELD_NOTE",
        "snippet": "Warung ramai di jam sibuk, pelanggan tetap cukup banyak."
      },
      {
        "source": "PHOTO",
        "description": "Rak stok tersusun rapi, barang fast moving tersedia."
      }
    ],

    "mitigation_notes": [
      "Mulai dengan limit sedikit di bawah pengajuan awal.",
      "Pantau pembayaran 2 bulan pertama sebelum menaikkan limit."
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
        "details": "Estimasi lapangan sejalan dengan profil di aplikasi."
      },
      {
        "field": "inventory_value",
        "status": "slightly_low",
        "details": "Nilai stok sedikit di bawah klaim omzet, perlu dipantau."
      }
    ]
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

# LifeRoute – Contract Doc (v1)

This document freezes **scope, data models, API interfaces, and permissions**.
No backend or frontend work should diverge from this without updating this file.

---

## 1. Frozen scope (must-have features)
- Doctor–patient communication (chat + queries)
- Prescription creation + updates in-app
- Emergency management: ambulance navigation using vacancy + budget + quality expectation
- Daily timed medication reminders + long-term protocols
- OPD booking through app
- Classify problems/diseases into online vs offline consult (suggestion, not diagnosis)
- Recovery progress shown through app

---

## 2. Frozen constants
- Roles: `patient`, `doctor`, `dispatcher`, `admin`
- Consent modes: `none`, `doctors`, `custom`
- API base prefix: `/api/v1`

---

## 3. Work division (ownership, not permissions)
- Auth & Identity — Arham
- Patient Profile & Consent — Arham
- Doctor–Patient Communication — Dev
- Prescriptions & Medication — Dev
- Emergency Management — Arham
- Appointments (OPD) — Dev
- Online Consultation — Arham
- Recovery & Progress Tracking — Dau
- Notifications & Reminders — Arham

---

## 4. Core entities (v1)
IDs are strings. All timestamps are ISO-8601.

### User
- id, role, name, phone, email?, created_at

### PatientProfile
- patient_id, age, sex?, conditions[], allergies[], emergency_contacts[]

### DoctorProfile
- doctor_id, specialization, clinic_name, hospital_id?

### Hospital
- id, name, location_text, lat, lon, price_level, quality_score, vacancy_beds, vacancy_icu, phone

### Consent
- patient_id, mode, allowed_doctor_ids[], updated_at

### Conversation
- id, patient_id, doctor_id, created_at, last_message_at

### Message
- id, conversation_id, sender_user_id, text, created_at

### Prescription
- id, patient_id, doctor_id, diagnosis_text, notes, status, created_at, updated_at

### Medication
- name, dosage_text, frequency_text, duration_days, instructions

### Appointment
- id, patient_id, provider_type, provider_id, mode, slot_time, status, reason_text, created_at

### EmergencyCase
- id, patient_id?, pickup_lat, pickup_lon, pickup_text, budget, quality_expectation, status, assigned_dispatcher_id?, recommended_hospital_id?, created_at, updated_at

### Reminder
- id, patient_id, title, message, schedule_times[], timezone, enabled, created_at, updated_at

### RecoveryLog
- id, patient_id, metric_type, value_text, recorded_at, notes?

---

## 5. Global API rules

### Response format
All responses are JSON.

### Error format
```json
{ "error": { "code": "STRING", "message": "STRING" } }
Authentication
All endpoints require authentication except:

/auth/register

/auth/login

Header:

makefile
Copy code
Authorization: Bearer <token>
Consent logic
none → doctor cannot access patient medical data

doctors → any doctor can access

custom → only doctors in allowed_doctor_ids

Consent applies to:

prescriptions

recovery logs

6. API endpoints (v1)
6.1 Auth & Identity
POST /auth/register
json
Copy code
{ "role": "patient", "name": "A", "phone": "9xxx", "password": "..." }
json
Copy code
{ "token": "...", "user": { "id": "u1", "role": "patient" } }
POST /auth/login
json
Copy code
{ "phone": "9xxx", "password": "..." }
json
Copy code
{ "token": "...", "user": { "id": "u1", "role": "patient" } }
GET /auth/me
json
Copy code
{ "user": { "id": "u1", "role": "patient" } }
6.2 Patient Profile & Consent
GET /patient/profile
json
Copy code
{ "age": 17, "conditions": [], "emergency_contacts": [] }
PUT /patient/profile
json
Copy code
{ "age": 17, "conditions": ["asthma"] }
json
Copy code
{ "status": "updated" }
GET /patient/consent
json
Copy code
{ "mode": "custom", "allowed_doctor_ids": ["d1"] }
PUT /patient/consent
json
Copy code
{ "mode": "doctors", "allowed_doctor_ids": [] }
json
Copy code
{ "status": "updated" }
6.3 Doctor–Patient Communication
POST /conversations
json
Copy code
{ "doctor_id": "d1" }
json
Copy code
{ "conversation_id": "c1" }
GET /conversations
json
Copy code
{ "conversations": [{ "id": "c1" }] }
GET /conversations/{id}/messages
json
Copy code
{ "messages": [{ "id": "m1", "text": "Hi" }] }
POST /conversations/{id}/messages
json
Copy code
{ "text": "I have fever" }
json
Copy code
{ "message_id": "m2" }
6.4 Prescriptions & Medication (CONSENT-PROTECTED)
POST /prescriptions
json
Copy code
{
  "patient_id": "u1",
  "diagnosis_text": "Viral fever",
  "medications": [
    { "name": "Paracetamol", "dosage_text": "500mg", "frequency_text": "2x/day", "duration_days": 3 }
  ]
}
json
Copy code
{ "prescription_id": "p1" }
PATCH /prescriptions/{id}
json
Copy code
{ "notes": "If fever persists, visit clinic" }
json
Copy code
{ "status": "updated" }
GET /prescriptions
json
Copy code
{ "prescriptions": [{ "id": "p1", "status": "active" }] }
GET /prescriptions/{id}
json
Copy code
{ "id": "p1", "medications": [] }
6.5 Notifications & Reminders
POST /reminders
json
Copy code
{ "title": "Medicine", "schedule_times": ["09:00"] }
json
Copy code
{ "reminder_id": "r1" }
GET /reminders
json
Copy code
{ "reminders": [{ "id": "r1" }] }
PATCH /reminders/{id}
json
Copy code
{ "enabled": false }
6.6 Appointments (OPD)
POST /appointments
json
Copy code
{ "provider_id": "d1", "mode": "offline", "slot_time": "2025-12-21T10:00" }
json
Copy code
{ "appointment_id": "a1" }
GET /appointments
json
Copy code
{ "appointments": [{ "id": "a1", "status": "requested" }] }
PATCH /appointments/{id}
json
Copy code
{ "status": "confirmed" }
6.7 Online Consultation (Suggestion Only)
POST /consultation/triage
json
Copy code
{ "age": 17, "symptoms": ["fever"], "duration_days": 2 }
json
Copy code
{ "suggested_mode": "online", "confidence": 0.62 }
6.8 Emergency Management
POST /emergency/cases
json
Copy code
{ "pickup_lat": 22.7, "pickup_lon": 75.9, "budget": "medium" }
json
Copy code
{ "case_id": "e1" }
GET /emergency/cases/{id}
json
Copy code
{ "status": "triaged", "recommended_hospital_id": "h1" }
POST /emergency/cases/{id}/recommendations
json
Copy code
{ "recommendations": [{ "hospital_id": "h1", "score": 0.88 }] }
6.9 Recovery & Progress Tracking (CONSENT-PROTECTED)
POST /recovery/logs
json
Copy code
{ "metric_type": "symptom", "value_text": "Improving" }
json
Copy code
{ "log_id": "rl1" }
GET /recovery/logs
json
Copy code
{ "logs": [{ "id": "rl1" }] }
GET /recovery/summary
json
Copy code
{ "trend": "improving" }
7. Permission matrix
Feature	Patient	Doctor	Dispatcher	Admin	Consent
Auth	✅	✅	✅	✅	No
Profile	✅	❌	❌	✅	No
Chat	✅	✅	❌	✅	No
Prescriptions	✅	✅*	❌	✅	Yes
Appointments	✅	✅	❌	✅	No
Reminders	✅	❌	❌	✅	No
Emergency	✅	❌	✅	✅	No
Recovery	✅	✅*	❌	✅
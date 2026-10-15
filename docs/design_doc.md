# Care Plan Generation System — Design Doc

## 1. Background

A specialty pharmacy currently relies on pharmacists to manually review a patient's medical record and write a care plan for each medication order. This takes 20–40 minutes per patient, is required for compliance and Medicare/pharma reimbursement, and the pharmacy is backlogged because it is understaffed.

**Goal:** Build a tool that lets a medical worker enter patient/medication/provider information through a web form, automatically generate a care plan via an LLM, and support downstream reporting.

## 2. Users

- Internal users only: CVS medical workers (medical assistants / pharmacists). Patients do not interact with the system directly.
- The medical worker enters patient info, triggers care plan generation, downloads/prints the result, and hands it to the patient.
- No role-based permission split in this phase (MVP assumption — may change later).

## 3. Scope — Core Entities

| Entity | Key Fields |
|---|---|
| **Patient** | First Name, Last Name, MRN (unique 6-digit), DOB, Primary Diagnosis (ICD-10), Additional Diagnoses (list of ICD-10), Medication History (list) |
| **Provider** | Name, NPI (unique 10-digit) |
| **Order** | Patient (FK), Provider (FK), Medication Name, Patient Records (string or PDF) |
| **Care Plan** | Order (FK, 1:1), Content, Status (pending / processing / completed / failed) |

**Key relationship rule:** One Care Plan maps to exactly one Order, and one Order maps to exactly one medication. A patient with multiple medications will have multiple orders, each generating its own care plan.

## 4. Functional Requirements

| Feature | Priority | Notes |
|---|---|---|
| Web form to submit patient/order info | Must | Primary data entry point |
| Input validation | Must | Format rules below |
| Patient duplicate detection | Must | Must not disrupt existing workflow |
| Order duplicate detection | Must | Prevent redundant submissions |
| Provider duplicate detection | Must | Affects pharma reporting accuracy |
| LLM-generated care plan | Must | Core value of the product |
| Care plan download | Must | User uploads it to their own system |
| Export for pharma reporting | Must | Quick export of structured data |

## 5. Care Plan Output Requirements

A generated care plan must include, at minimum:
- Problem list / drug therapy problems
- Goals (SMART format where applicable)
- Pharmacist interventions / plan
- Monitoring plan & lab schedule

## 6. Validation Rules

- Patient MRN: unique, 6-digit numeric
- Provider NPI: unique, 10-digit numeric
- Diagnosis codes: valid ICD-10 format
- All required fields must be present before an order can be submitted

## 7. Duplicate / Integrity Detection Rules

| Scenario | Action | Rationale |
|---|---|---|
| Same patient + same medication + **same day** | ❌ **ERROR** — block submission | Certain duplicate |
| Same patient + same medication + **different day** | ⚠️ **WARNING** — allow override | Likely a refill |
| MRN matches but name or DOB differs | ⚠️ **WARNING** — allow override | Possible data-entry error |
| Name + DOB match but MRN differs | ⚠️ **WARNING** — allow override | Possibly the same person |
| NPI matches but provider name differs | ❌ **ERROR** — must correct | NPI is a unique national identifier; conflict is never valid |

**Design principle:** When the system cannot be certain whether something is a true duplicate (patient-identity ambiguity, possible refill), it should warn and let the human confirm — not silently decide. Only cases with an unambiguous ground truth (same-day same-med-same-patient; NPI collision) are hard errors.

## 8. Non-Functional / Production-Readiness Requirements

- Every input validated at the API boundary
- Integrity rules enforced consistently across all entry points (not just the web form)
- Errors are safe, clear, and contained (no leaking stack traces/internal state)
- Codebase is modular and navigable (clear separation of concerns)
- Critical business logic (validation, duplicate detection) covered by automated tests
- Project runs end-to-end out of the box (minimal setup, documented steps, reproducible environment)

## 9. Open Questions / Future Considerations

- Role-based permissions (medical assistant vs. pharmacist) — not in MVP, revisit if customer requests it
- Multiple data source ingestion (beyond the web form) — addressed in a later phase via adapters
- Async processing of LLM calls — addressed once synchronous generation proves too slow for real usage
- Monitoring/observability — addressed once the system is deployed

## 10. Out of Scope (for now)

- Patient-facing access
- Automated LLM failure recovery beyond basic retry
- Multi-tenant / multi-pharmacy support

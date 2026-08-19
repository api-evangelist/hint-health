---
name: Write clinical interactions into a Hint patient chart
description: >-
  Post clinical notes, lab results, documents and partner records into a Hint
  patient's chart, notify the practice, and pull attachment download URLs — the
  PHI-bearing write path, with the handling rules that go with it.
api: openapi/hint-health-clinicalnoteinteraction-api-openapi.yml
apis:
  - openapi/hint-health-clinicalnoteinteraction-api-openapi.yml
  - openapi/hint-health-labinteraction-api-openapi.yml
  - openapi/hint-health-documentinteraction-api-openapi.yml
  - openapi/hint-health-partnerinteraction-api-openapi.yml
  - openapi/hint-health-interaction-api-openapi.yml
  - openapi/hint-health-interactionnotifications-api-openapi.yml
  - openapi/hint-health-integrationrecord-api-openapi.yml
operations:
  - ClinicalNoteInteraction.CreateClinicalNoteInteraction
  - LabInteraction.CreateLabInteraction
  - DocumentInteraction.CreateDocumentInteraction
  - PartnerInteraction.CreatePartnerInteraction
  - Interaction.ListAllInteractions
  - InteractionNotifications.CreateInteractionNotification
  - IntegrationRecord.ListAllIntegrationRecords
generated: '2026-08-15'
method: generated
source: >-
  openapi/ (operationIds verified present), data-model/hint-health-data-model.yml,
  conventions/hint-health-conventions.yml
---

# Write clinical interactions into a patient chart

An **interaction** is Hint's unit of clinical record — a note, a lab, a document,
or an opaque partner record. All four are created under a patient and all four
carry PHI. Treat every call in this skill as regulated: you are writing into a
covered entity's chart as a business associate.

## Before you start

- **Auth** practice access token only. A partner API key must never touch
  `/provider/patients/{patient_id}/interactions/*`.
- **Never log a request or response body from this skill.** These payloads carry
  names, diagnoses, and lab values.
- **Idempotency** every interaction type accepts `integration_record_id`, unique
  per object type. Set it from your own record id so a retry cannot double-post a
  note into a chart. A duplicate is rejected, not replayed — on 422, look the
  record up rather than re-posting.
- Each interaction also carries `integration_error_message` and
  `integration_web_link`, which surface your sync state and a deep link back to
  your system inside the Hint UI. Fill them; they are what makes the record
  supportable by the practice.

## Choose the right interaction type

| Type | Operation | Use for |
|---|---|---|
| Clinical note | `ClinicalNoteInteraction.CreateClinicalNoteInteraction` | Narrative documentation. Fields: `title`, `text`, `formatted_text`, `event_timestamp`, `files`, `provider_name`. |
| Lab | `LabInteraction.CreateLabInteraction` | Structured results. Nested `order`, `report`, `results[]` — see below. |
| Document | `DocumentInteraction.CreateDocumentInteraction` | A file that is not a note or a lab. |
| Partner | `PartnerInteraction.CreatePartnerInteraction` | Your own record type, rendered as a partner entry. |

All four are `POST /provider/patients/{patient_id}/interactions/<type>`, with a
matching `Show` and `Update`.

### Lab payload shape

Hint's lab serializer is **FHIR-influenced but not FHIR** — it is a snake_cased
renaming of R4 elements, with no `resourceType` and no CapabilityStatement, so do
not hand it to a FHIR client:

- `order` → `lab_reference_id`, `ordered_from`, `authored_on`, `collected_at`,
  `author`, `author_reference`, `actions[]`, `intent`, `subject`, `lab_account_id`
- `report` → `issued`, `status`, `lab_reference_id`, `order_reference`
- `results[]` → `code {code, display}`, `code_text`, `issued`,
  `value_quantity {value, unit}`, `value_string`, `text`, `interpretation`,
  `ranges [{low, high, text}]`, `subject`
- top level → `status`, `vendor_order_id`, `event_timestamp`

Send `value_quantity` for numeric results and `value_string` for qualitative ones.
Do not put a number in `value_string` — the practice's trending breaks.

## Steps

1. **Resolve the patient.** Use `Patient.MatchPatients` or an
   `integration_record_id` lookup. Do not create a chart entry against a guessed id.
2. **Post the interaction** with the operation from the table above.
3. **Notify the practice** when it needs eyes on it:
   `InteractionNotifications.CreateInteractionNotification`
   (`POST /provider/interactions/{interaction_id}/notifications`). Notify on
   abnormal results and time-sensitive documents; do not notify on routine syncs,
   or the practice will mute you.
4. **Reconcile.** `IntegrationRecord.ListAllIntegrationRecords`
   (`GET /provider/patients/{patient_id}/integration_records`) tells you what your
   integration has already written for this patient — run it before a backfill.

## Reading back

`Interaction.ListAllInteractions` (`GET /provider/interactions`) lists the
practice's interactions. Attachments are **not** direct URLs: request short-lived,
single-use download URLs from the interaction's file endpoints
(`/provider/interactions/{interaction_id}/files/download_urls` and
`.../files/{id}/download_url`, in the 2026-07-01 harvest at
`openapi/_original/hint-health-partner-endpoints-2026-07-01-openapi.yml`). They
expire; fetch immediately and never cache or forward the URL. A non-PDF file
returns 422 rather than a URL.

## Events

`patient.created/updated/destroyed/inactive` fire on the patient. There is no
`interaction.*` event in Hint's published catalogue, so an integration that needs
to know when the practice writes back must poll `Interaction.ListAllInteractions`
with an `updated_at[gt]` delta rather than wait for a webhook.

## Failure handling

Standard `{"status", "message"}` envelope. 403 can mean either insufficient scope
or a security anomaly block on a PHI endpoint — if the token is right, stop and
contact devsupport@hint.com rather than retrying into the block. 429 is 20 req/s /
500,000 per day per partner with no `Retry-After`; a chart backfill must be
throttled deliberately and scheduled off-hours.

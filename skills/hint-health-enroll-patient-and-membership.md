---
name: Enroll a patient and start a membership in Hint
description: >-
  End-to-end enrollment against the Hint Provider API — de-duplicate the person,
  create the patient, price the plan, capture a payment method through the
  practice's processor, then create and confirm the membership.
api: openapi/hint-health-patient-api-openapi.yml
apis:
  - openapi/hint-health-patient-api-openapi.yml
  - openapi/hint-health-plan-api-openapi.yml
  - openapi/hint-health-quote-api-openapi.yml
  - openapi/hint-health-paymentmethod-api-openapi.yml
  - openapi/hint-health-membership-api-openapi.yml
  - openapi/hint-health-membershipmember-api-openapi.yml
operations:
  - Patient.MatchPatients
  - Patient.CreatePatient
  - Plan.ListAllPlans
  - Quote.CreateQuote
  - PaymentMethod.CreateSetupIntent
  - PaymentMethod.CreatePaymentMethod
  - Membership.CreateMembership
  - Membership.ConfirmMembership
  - MembershipMember.CreateMembershipMember
generated: '2026-08-15'
method: generated
source: >-
  openapi/ (operationIds verified present), conventions/hint-health-conventions.yml,
  errors/hint-health-problem-types.yml, sandbox/hint-health-sandbox.yml
---

# Enroll a patient and start a membership

Membership enrollment is the flow the Hint platform exists to run. Get it wrong and
you create duplicate patients, orphaned draft memberships, or a membership that
never bills because no payment method was attached.

## Before you start

- **Base URL** `https://api.hint.com/api`. The same host serves sandbox and live —
  a key prefixed `sbx-` selects sandbox. Do not swap hosts when you promote.
- **Auth** `Authorization: Bearer <practice access token>`. This flow is entirely
  on `/provider/*`, so you must use the practice-scoped token from
  `api_keys[0].token`, never the partner API key. Using the partner key here leaks
  across practices.
- **Money** every amount is an integer in cents (`*_in_cents`). There is no
  currency field.
- **Idempotency** set `integration_record_id` on every create. It is unique per
  object type and is the only duplicate protection Hint offers. There is no
  `Idempotency-Key` header, and a retry with the same value returns a validation
  error rather than replaying the original response — so treat 422-on-retry as
  "already created", then look the record up by `integration_record_id`.

## Steps

1. **De-duplicate first.** `Patient.MatchPatients`
   (`POST /provider/patients/matching`) against name, date of birth and email.
   Hint's own list endpoints exclude archived patients by default, so a person you
   cannot find with `Patient.ListAllPatients` may still exist — re-check with
   `?filter=all` before concluding they are new.

2. **Create the patient.** `Patient.CreatePatient`
   (`POST /provider/patients`). Send `integration_record_id`. Addresses are flat
   `address_*` fields on the patient, not a nested object; state values are
   normalized to two-letter codes. Phones are an array of
   `{number, type, us_formattable}` and `type` is canonicalized (`cell` → `mobile`,
   `work phone` → `office`).

3. **Choose the plan.** `Plan.ListAllPlans` (`GET /provider/plans`) returns the
   practice's published plans with their periods and rates. Plan ids are `pln-`
   prefixed.

4. **Price it before you commit.** `Quote.CreateQuote` (`POST /provider/quotes`)
   takes the plan, the member roster (`type`, `age`, `uses_tobacco`, `dob`,
   `start_date`), the period, and an optional coupon, and returns the computed
   price. Show this to the patient; do not compute the price yourself from plan
   rates — age banding and tobacco loading are Hint's to apply.

5. **Capture a payment method.** `PaymentMethod.CreateSetupIntent`
   (`POST /provider/patients/{patient_id}/payment_methods/setup`) returns the
   processor handshake for whichever processor this practice uses — read
   `practice.payment_processor` first:
   - **Hint Payments (Rainforest)** → `payment_method_config_id`, `session_key`,
     `allowed_methods`; render Rainforest's payment component.
   - **Stripe** → `stripe_client_secret`; render Stripe Elements.

   Then `PaymentMethod.CreatePaymentMethod`
   (`POST /provider/patients/{patient_id}/payment_methods`) with **exactly one** of
   `rainforest_id` or `stripe_id`. Never post a card number — Hint's API never
   accepts a PAN, which is how the platform stays out of PCI scope.

6. **Create the membership.** `Membership.CreateMembership`
   (`POST /provider/memberships`) with `membership_patients[]` (each
   `{patient, start_date, member_type, fixed_period_rate_in_cents}`), `plan`,
   `owner`, `start_date`, `period_in_months`, and any `sponsorship` for an
   employer-paid membership.

7. **Add dependents** with `MembershipMember.CreateMembershipMember`
   (`POST /provider/memberships/{membership_id}/members`) if the family was not
   supplied in step 6.

8. **Confirm it.** `Membership.ConfirmMembership`
   (`POST /provider/memberships/{id}/confirm`). Until you confirm, the membership
   is not live and will not bill. This is the step integrations most often skip.

## Verify

- `GET /provider/memberships/{id}` and check `status`, `plan`, and that
  `membership_patients` holds everyone you intended.
- Webhooks: expect `patient.created` and `membership.created` on your registered
  endpoint. Payloads are always **unexpanded** — `?expand=` has no effect on
  delivery, so re-fetch the resource if you need the nested graph.

## Failure handling

| Status | What it means here | Do |
|---|---|---|
| 401 | Wrong token family | You are almost certainly holding the partner key. Switch to the practice access token. |
| 403 | Scope, or a security anomaly block | Check the token; if volume is high, back off and mail devsupport@hint.com. |
| 404 | Id wrong — or the record is archived | Retry the lookup with `?filter=all`. |
| 422 | Validation, or a duplicate `integration_record_id` | Read `message`. On duplicate, look the record up instead of re-creating. |
| 429 | 20 req/s or 500,000/day, per partner | Exponential backoff **with jitter**. There is no `Retry-After` and no quota header — the server tells you nothing about when to retry. |

The error body is always `{"status": <int>, "message": "<human string>"}`. `message`
is not a stable machine identifier; do not branch on its text.

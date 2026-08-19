---
name: Bill a patient and collect a Hint customer invoice
description: >-
  Build a customer invoice from charges, issue it, take payment against it, and
  handle the void / return-to-draft paths — the Hint billing lifecycle as an agent
  can safely drive it.
api: openapi/hint-health-customerinvoice-api-openapi.yml
apis:
  - openapi/hint-health-customerinvoice-api-openapi.yml
  - openapi/hint-health-customerinvoicecharge-api-openapi.yml
  - openapi/hint-health-customerinvoicepayment-api-openapi.yml
  - openapi/hint-health-charge-api-openapi.yml
  - openapi/hint-health-membership-api-openapi.yml
  - openapi/hint-health-credit-api-openapi.yml
operations:
  - CustomerInvoice.CreateCustomerInvoice
  - CustomerInvoiceCharge.CreateCustomerInvoiceCharge
  - CustomerInvoice.IssueCustomerInvoice
  - CustomerInvoicePayment.CreatePayment
  - CustomerInvoice.CreatePDF
  - CustomerInvoice.ReturnToDraft
  - CustomerInvoice.VoidCustomerInvoice
  - CustomerInvoice.ListAllCustomerInvoices
  - Membership.BillMembership
  - Credit.CreateCredit
generated: '2026-08-15'
method: generated
source: >-
  openapi/ (operationIds verified present), conventions/hint-health-conventions.yml,
  errors/hint-health-problem-types.yml
---

# Bill a patient and collect a customer invoice

Hint invoices move through a state machine — **draft → issued → paid**, with
`void` and `return_to_draft` as the escape hatches. Every transition is its own
operation. There is no `PATCH status` shortcut, and trying to invent one is the
most common integration mistake.

## Before you start

- **Auth** practice access token, `/provider/*` surface only.
- **Money** integers in cents throughout — `amount_in_cents`, `paid_in_cents`,
  `price_in_cents`, `past_due_in_cents`. Divide by 100 for display only.
- **Idempotency** `integration_record_id` on every create.
- **Dates** bucket reporting on `paid_at` / `date`, never `created_at`. In sandbox
  every seeded record shares one `created_at`, so any `created_at` time series is a
  flat line with one spike.

## Steps

1. **Open a draft.** `CustomerInvoice.CreateCustomerInvoice`
   (`POST /provider/customer_invoices`) with the `owner` (the paying patient) and
   `location`. It lands in draft.

2. **Add lines.** `CustomerInvoiceCharge.CreateCustomerInvoiceCharge`
   (`POST /provider/customer_invoices/{customer_invoice_id}/charges`) once per
   line. Reference a `charge_item` where the practice has a catalogued item so the
   line inherits its code and category; otherwise supply `description`,
   `quantity`, `price_in_cents`. `CustomerInvoiceCharge.UpdateCustomerInvoiceCharge`
   and `...DeleteCustomerInvoiceCharge` work only while the invoice is a draft.

   > Ad-hoc charges that should run through Hint's pricing engine instead of onto a
   > specific invoice go to `Charge.CreateCharge` (`POST /provider/charges`) — Hint
   > then places them on the right invoice itself.

3. **Apply credit if the patient has any.** `Credit.CreateCredit`
   (`POST /provider/patients/{patient_id}/credits`) against a credit category.

4. **Issue it.** `CustomerInvoice.IssueCustomerInvoice`
   (`POST /provider/customer_invoices/{id}/issue`). Lines are frozen from here.

5. **Take payment.** `CustomerInvoicePayment.CreatePayment`
   (`POST /provider/customer_invoices/{customer_invoice_id}/payments`) against a
   payment method already on file for the owner (see the enroll skill for
   `PaymentMethod.CreateSetupIntent`). Confirm with
   `CustomerInvoicePayment.ListAllPayments`.

6. **Render a PDF** for the patient with `CustomerInvoice.CreatePDF`
   (`POST /provider/customer_invoices/{customer_invoice_id}/pdf`).

## Corrections

- **Wrong lines, not yet paid** → `CustomerInvoice.ReturnToDraft`
  (`POST /provider/customer_invoices/{id}/return_to_draft`), fix the charges, issue
  again.
- **Should never have existed** → `CustomerInvoice.VoidCustomerInvoice`
  (`POST /provider/customer_invoices/{id}/void`). Void, do not delete — deleting a
  billed invoice destroys the practice's audit trail.
- **Recurring membership dues** are Hint's job, not yours.
  `Membership.BillMembership` (`POST /provider/memberships/{id}/bill`) triggers a
  membership bill on demand; do not hand-build a monthly invoice for a membership
  that Hint already bills on `next_bill_date`.

## Reconciliation

`CustomerInvoice.ListAllCustomerInvoices` is the endpoint Hint points at as its
full advanced-query example. Filter with the operator forms Hint documents —
`?date={"gte":"2026-01-01","lt":"2026-02-01"}` or the bracket form
`?date[gte]=2026-01-01&date[lte]=2026-01-31` — plus `limit`/`offset` (max 100) and
`sort=-paid_at`. Read `x-total-count` from the response headers to size the walk;
the body itself is a **bare JSON array**, not `{data: [...]}`.

Advanced querying is enabled per endpoint. If an operator filter is silently
ignored somewhere else, that endpoint has not been enabled for your integration —
mail devsupport@hint.com rather than assuming the parameter name is wrong.

## Events

Subscribe to `customer_invoice.draft`, `.issued`, `.paid`, `.cancelled`,
`.created`, `.updated`, `.destroyed`. Endpoints receive the practice's **full**
event stream — Hint has no per-endpoint event filter in the API — so ignore
unknown `type` values rather than erroring: Hint adds `resource.action` pairs
without a version bump.

## Failure handling

`{"status": <int>, "message": "<string>"}`, always. 422 also covers domain
impossibilities, not just field validation — e.g. emailing an invoice whose owner
has no email address, or has previously bounced. 429 carries no `Retry-After`;
back off exponentially with jitter.

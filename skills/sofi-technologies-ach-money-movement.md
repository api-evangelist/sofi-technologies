---
name: sofi-technologies-ach-money-movement
description: >-
  Move money on the SoFi Tech Solutions Program API over ACH - link an external bank account,
  originate a transaction, read history, and cancel inside the stated window.
api: SoFi Tech Solutions Program API 4.0
spec: openapi/sofi-technologies-program-api-openapi.json
generated: '2026-09-06'
method: generated
source: >-
  Grounded in openapi/sofi-technologies-program-api-openapi.json and
  https://docs.tech.sofi.com/pro/reference/post_createachtransaction
operations:
  - post_addachaccount
  - post_addachaccountcorporate
  - post_getachaccounts
  - post_createachtransaction
  - post_cancelachtransaction
  - post_getachtranshistory
  - post_verifyaccount
  - post_getbalance
  - post_createsimulatedachtransaction
  - post_cancelsimulatedachtransaction
  - post_getallsimulatedachtransactions
---

# Move money over ACH

Same transport rules as every Program API call: form-encoded `POST` to
`/intserv/4.0/{endpointName}`, four auth parameters in the body, a fresh UUID `transactionId`.

## 1. Link the external bank account

`post_addachaccount` for a consumer account, `post_addachaccountcorporate` for a corporate one.
Read back with `post_getachaccounts`. The linked account is addressed afterwards by
`achAccountNo`.

## 2. Check the account can take the movement

`post_verifyaccount` returns the current balance, the maximum amount that can currently be loaded
under the product parameters, the external account ID, the account PRN and the balance ID. Calling
it first is cheaper than handling a limit violation.

## 3. Originate

`post_createachtransaction`. Two things changed recently and will bite an older integration:

- `companyEntryDesc` became **required** in August 2026.
- As of September 2026 the SEC code classification for B2C debits is sharpened — online or mobile
  authorization should carry `WEB`; other B2C transactions default to `PPD`.

This endpoint is idempotent on `transactionId`.

## 4. Cancel — and know the window

`post_cancelachtransaction` reverses a transaction you created, **only before the ACH binaries
have processed it**. After that you get `431-05` (ACH transaction has been processed and cannot
be canceled). The other outcomes are `431-01` not found, `431-02` in error status, `431-03`
already canceled, `431-04` cannot be canceled.

There is no time window published for this — the boundary is a processing state, not a clock, so
poll rather than assume.

## 5. Read history

`post_getachtranshistory`. Large result sets page with `recordCnt` and `page`; the response
carries `total_record_count`, `number_of_pages` and `page`. The default per-page maximum is 200
and anything larger is clamped to 200.

## 6. Returns

An ACH return arrives as an event, not as a response to your call — `ach_return`,
`ach_credit_return`, `ach_credit_fail`, `ach_debit_fail` on the Events API webhook. Nacha return
reason codes are enumerated at
`https://docs.tech.sofi.com/pro/reference/api-reference-ach-return-codes`. See
`asyncapi/sofi-technologies-webhooks.yml`.

If you want to be the one deciding on incoming ACH debits rather than just hearing about them,
that is the External Trans API webhook (`webhook_ach_debit_post`) — you answer with a
`response_code` in the body.

## Testing

In the Sandbox, simulate the inbound side with `post_createsimulatedachtransaction`, list what you
have created with `post_getallsimulatedachtransactions`, and tear it down with
`post_cancelsimulatedachtransaction`. There is no test clock, so settlement timing cannot be
fast-forwarded.

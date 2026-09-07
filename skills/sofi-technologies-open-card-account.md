---
name: sofi-technologies-open-card-account
description: >-
  Open a SoFi Tech Solutions card account for a new customer on the Program API - enrollment,
  customer ID verification, account creation, card activation and the void path if it goes wrong.
api: SoFi Tech Solutions Program API 4.0
spec: openapi/sofi-technologies-program-api-openapi.json
generated: '2026-09-06'
method: generated
source: >-
  Grounded in openapi/sofi-technologies-program-api-openapi.json (every operationId below was
  grepped out of that file) and https://docs.tech.sofi.com/pro/docs/creating-an-account
operations:
  - post_startenrollment
  - post_getenrollmentinfo
  - post_completeenrollment
  - post_createaccount
  - post_voidcreateaccount
  - post_getaccountbyid
  - post_getaccountcards
  - post_activatecard
  - post_getbalance
  - post_getcallstatus
---

# Open a card account

Every call is a `POST` with `Content-Type: application/x-www-form-urlencoded` to
`https://api-{corename}.{env}.gpsrv.com/intserv/4.0/{endpointName}`. Do not omit `/intserv/4.0/`
and do not double a slash — any non-standard URI returns 404. Set `response-content-type: json`.

## Before you start

Every request carries four parameters in the body: `apiLogin`, `apiTransKey`, `providerId` (all
issued by SoFi Tech Solutions and bound to your source IP) and `transactionId`, which you
generate. Use a fresh UUID per request. `transactionId` is also the idempotency key on the write
steps below — same `transactionId` + `providerId` + endpoint inside 90 days returns
`status_code: 24` (Duplicate transaction) instead of acting twice.

Dates and times you send are interpreted in Arizona Standard Time (GMT-0700, no daylight saving),
not UTC.

## 1. Decide which entry point you need

- `post_createaccount` creates the customer record **and** the account in one call, and can run
  identity verification if the EIVS provider parameter is set. Use it for the ordinary path.
- `post_startenrollment` creates only the customer record and runs the legacy CIP. Use it when you
  need to verify a person before committing to an account.

## 2. If you enrolled first, finish the enrollment

- `post_getenrollmentinfo` — pass the `transactionId` from the original Start Enrollment (or from
  a Create Account call sent with `cipStatus: 1`) to read back what was submitted and the CIP
  outcome.
- `post_completeenrollment` — pass the **same** `transactionId` as the original call to finalise.

## 3. Create the account

`post_createaccount` with `prodId` for the product you are opening against. Depending on product
settings this also creates a card and loads funds. Most input parameters are optional by design;
if you need some to be mandatory, that is set with the PINED product parameter, not in the call.

Read the result from `status_code`, not from the HTTP status — the HTTP status is 200 on business
failures too. A code that does not match this endpoint's own table is probably an enrollment
status code.

## 4. Read the account back

- `post_getaccountbyid` — supply exactly one of the mutually exclusive lookup keys.
- `post_getaccountcards` — returns the cards, the PAN and the current ship-to address.
- `post_getbalance` — note that this returns balances for **every** account sharing the same
  balance ID (the Galileo account number), not just the one you asked about.

## 5. Activate the card

`post_activatecard`. This is one of the 35 endpoints with `transactionId` idempotency, so a retry
after a timeout is safe.

## 6. If it went wrong

`post_voidcreateaccount` moves the account to `status: B` (Voided). Pass the original Create
Account `transactionId`, or the optional `accountNo`.

The window is narrow and stated:

- instant-issue (prepaid) card accounts only;
- `412-03` if secondary or related cards have already been added;
- `412-04` if any transaction has posted against the account;
- `412-05` if the product has been changed since creation.

For any other account type the reversal is not an API call at all — it goes through the Customer
Service Tool.

## Recovering from an ambiguous write

If a write times out and you do not know whether it landed, do **not** retry blindly with a new
`transactionId`. Call `post_getcallstatus` with the original `transactionId` as
`requestTransactionId` and the endpoint name in camelCase as `requestMethodName`. It returns the
status code of the original call.

## Status codes worth branching on

| Code | Meaning |
| --- | --- |
| `0` | Success |
| `1` / `2` | Missing / invalid parameters — read the `errors[]` list |
| `4` | Failed API login |
| `21` | Unregistered IP address — your source IP is not the one the credentials were issued for |
| `24` | Duplicate transaction — this `transactionId` already succeeded within 90 days |
| `54` | Endpoint deprecated and will be removed |

The full common table is in `errors/sofi-technologies-problem-types.yml`.

## Testing

The Sandbox (`https://sandbox-api.gpsrv.com/intserv/4.0/`) carries all of the operations above
under program `6914` / product `2769`, capped at 1000 requests per 10 minutes. Sandbox credentials
expire every 30 days. SoFi Tech Solutions publishes no test card numbers — you create your own by
calling Create Account in the Sandbox.

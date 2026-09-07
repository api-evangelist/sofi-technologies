---
name: sofi-technologies-file-dispute-claim
description: >-
  File and track a cardholder dispute on the SoFi Tech Solutions Dispute API 3.0 - intake,
  reasons, transactions, questionnaire, documents, submit, then list and act on tasks.
api: SoFi Tech Solutions Dispute API 3.0 (25.02)
spec: openapi/sofi-technologies-dispute-api-3-0-openapi.json
generated: '2026-09-06'
method: generated
source: >-
  Grounded in openapi/sofi-technologies-dispute-api-3-0-openapi.json and
  https://docs.tech.sofi.com/pro/reference/api-reference-about-dispute-api-30
operations:
  - postClaimCreate
  - postRetrieveReasons
  - postClaimReasonsAdd
  - postRetrieveTransactions
  - postClaimTransactionsAdd
  - postRetrieveSimilarTransactions
  - postClaimTransactionsAddSimilar
  - postRetrieveQuestionnaire
  - postClaimQuestionnaireAdd
  - postRetrieveDocRequirements
  - postClaimIntakeDocumentsAdd
  - postRetrieveClaimIntakeSummary
  - postClaimSubmit
  - postRetrieveClaimIntakeConfirmation
  - postClaimList
  - postClaimDetailsRetrieve
  - postClaimReopenRequest
  - postClaimWithdrawRequest
  - postTaskSummary
  - postTaskList
  - postTaskAction
---

# File a dispute claim

Dispute API 3.0 is a different shape from the Program API: it is path-structured
(`/claim/intake/create`, `/task/action`) on
`{corename}.gft-dispute-api.{env}.gpsrv.com/gft-dispute-api/1.0/`, and it splits cleanly into an
**Intake API** that builds a claim and an **Interaction API** that works it afterwards.

Dispute API 2.0 still exists and is still published
(`openapi/sofi-technologies-dispute-api-2-0-openapi.json`, 22 operations). It is superseded and no
sunset date is published. Build new work on 3.0.

## Intake — build the claim, then submit it

The intake flow is deliberately staged. Each retrieve step tells you what the next add step is
allowed to contain, so do not skip the retrieves.

1. `postClaimCreate` — start the claim.
2. `postRetrieveReasons` → `postClaimReasonsAdd` — get the valid claim reasons for this card and
   attach the ones that apply.
3. `postRetrieveTransactions` → `postClaimTransactionsAdd` — pull the disputable transactions and
   attach the ones being disputed.
4. `postRetrieveSimilarTransactions` → `postClaimTransactionsAddSimilar` — the platform surfaces
   transactions that look related; attach any the cardholder also disputes. Skipping this is the
   most common cause of a partial claim.
5. `postRetrieveQuestionnaire` → `postClaimQuestionnaireAdd` — the questionnaire is reason-driven,
   so it must be fetched after the reasons are attached, not before.
6. `postRetrieveDocRequirements` → `postClaimIntakeDocumentsAdd` — document requirements likewise
   depend on the reasons and the answers.
7. `postRetrieveClaimIntakeSummary` — read back the whole claim before committing.
8. `postClaimSubmit` — commit it.
9. `postRetrieveClaimIntakeConfirmation` — the confirmation to show the cardholder.

For high volume, `postBulkCreate` takes claims in bulk.

## After submission

- `postClaimList` / `postClaimDetailsRetrieve` / `postClaimRetrieve` — find and read claims.
- `postTransactionsList` — the transactions attached to a claim.
- `postClaimAttachmentsAdd` and `postClaimDocumentsAdd` — add evidence to a live claim.
- `postClaimWithdrawRequest` — the cardholder no longer wants to pursue it.
- `postClaimReopenRequest` — reopen a closed claim.

## Work the queue

- `postTaskSummary` — counts by task type; use it to decide whether there is anything to do at all
  before pulling lists.
- `postTaskList` / `postTaskDetailsList` — the tasks themselves.
- `postTaskAction` — act on one.

## Reason and status enumerations

Claim reasons, deny reasons, final statuses and resolution outcomes are published as reference
enumerations:

- `https://docs.tech.sofi.com/pro/reference/api-reference-dispute-claim-reasons`
- `https://docs.tech.sofi.com/pro/reference/api-reference-dispute-deny-reasons`
- `https://docs.tech.sofi.com/pro/reference/api-reference-dispute-final-statuses`
- `https://docs.tech.sofi.com/pro/reference/api-reference-dispute-resolution`

## Regulatory note

Dispute handling on consumer deposit accounts in the United States is Regulation E work, and the
platform ships a Reg E RDF alongside the Dispute and Chargeback RDF. The clock on a Reg E
investigation is set by regulation, not by this API, and SoFi Tech Solutions does not publish that
timing in the developer docs — do not infer a deadline from anything here.

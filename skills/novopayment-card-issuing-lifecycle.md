---
name: novopayment-card-issuing-lifecycle
description: Issue, activate, fund, limit, block and replace NovoPayment cards, and reverse a cash movement when one goes wrong.
api: NovoPayment Cards API, Accounts API
provider: novopayment
generated: '2026-09-17'
method: generated
source: openapi/novopayment-cards-openapi.yml, openapi/novopayment-accounts-openapi.yml
operations:
  - CardAndAccountBundled
  - CardInformation
  - CardActiveStatus
  - CardPINAssignment
  - CardPINUpdate
  - VirtualToPhysicalCard
  - CardReplacement
  - BlockCard
  - UnblockCard
  - CashIn
  - CashInReverse
  - CashOut
  - CashOutReverse
  - CustomizeOperationLimits
  - AccountBalanceByCardId
  - CardMovementsQuery
  - TransactionDetail
  - TransactionSummary
  - AssociateTransactionalRules
  - ListTransactionRules
  - EnableDisableTransactionalRules
  - CardCardholdersAssociation
---

# Run a NovoPayment card program

Base: `https://sandbox-api.novopayment.com/api/v1.3` (Cards API v1.3). Token and headers as in the
onboarding skill — OAuth2 client credentials, `X-Tenant-Id` on every call, `X-Transaction-Id` for
tracing only.

## Issue

- `CardAndAccountBundled` (`POST /cards/issuance`) issues a card and its account together.
- `CardCardholdersAssociation` (`POST /cards/cardholders`) attaches the cardholder;
  `CardCardholdersUpdate` (`PUT /cards/cardholders`) edits them.
- On the Accounts API, `CardCreation` (`POST /customers/{customerId}/cards`) and
  `CardToAccountAssociation` (`POST /customers/{customerId}/cards/{cardId}/associate`) are the
  account-first equivalents.

## Activate and secure

- `CardActiveStatus` (`PUT /cards/{cardId}/status`) moves the card into an active state.
- `CardPINAssignment` (`POST /cards/{cardId}/pin`) sets the PIN, `CardPINUpdate` (`PUT`) changes it.
- `CardSecurityCodeCVV2` on the Accounts API returns the CVV2 for a virtual card.
- `VirtualToPhysicalCard` (`PUT /cards/{cardId}/tophysical`) promotes a virtual card.

## Control

- `CustomizeOperationLimits` (`PATCH /cards/cardholders/{cardId}/operationlimits`) sets per-card
  operation limits.
- `AssociateTransactionalRules` (`POST /cards/{cardId}/transactionrules`),
  `ListTransactionRules` (`GET`) and `EnableDisableTransactionalRules` (`PATCH`) manage the rule set.
- `BlockCard` (`POST /cards/{cardId}/block`) and `UnblockCard` (`POST /cards/{cardId}/unblock`).
  Note `400.01.409 The card is permanently blocked` is terminal — unblock will not clear it.

## Move money

- `CashIn` (`POST /cards/{cardId}/cashin`) and `CashOut` (`POST /cards/{cardId}/cashout`).
- `SendMoney` (`POST /sendmoney`) for card-to-card transfer.
- **Reversal:** `CashInReverse` (`POST /cards/{cardId}/reversecashin`) and `CashOutReverse`
  (`POST /cards/{cardId}/reversecashout`). NovoPayment documents the reversal operations but **not
  the window** in which they remain valid — never promise a customer a reversal deadline the docs do
  not state.

## Read

`AccountBalanceByCardId` (`GET /cards/{cardId}/balance`), `CardMovementsQuery`
(`GET /cards/{cardId}/transactions`), `TransactionDetail`
(`GET /cards/{cardId}/transactions/{transactionIdentifier}`) and `TransactionSummary`.

## Failure handling

Decline reasons come back in the `code` field and follow ISO 8583 detail numbers:
`400.01.044` insufficient funds, `400.01.008` card is blocked, `400.08.511`–`400.08.520` pick-up and
fraud conditions. The catalogue is in `errors/novopayment-decline-codes.yml`. NovoPayment publishes
no guidance on which of these may be shown to a cardholder — treat fraud/pick-up reasons as
issuer-only.

Money movement has no idempotency key. After a timeout on `CashIn`, `CashOut` or `SendMoney`, call
`CardMovementsQuery` for the card and match on your own reference before retrying.

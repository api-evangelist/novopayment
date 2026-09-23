---
name: novopayment-alias-real-time-payment
description: Register an alias and its payment instruments, then push, pull, query and reverse a real-time payment between Visa/Mastercard branded products.
api: NovoPayment Alias Directory API, Real-Time Payments API
provider: novopayment
generated: '2026-09-17'
method: generated
source: openapi/novopayment-alias-directory-openapi.yml, openapi/novopayment-real-time-payments-openapi.yml
operations:
  - createAlias
  - findAlias
  - GetAliasByAliasId
  - UpdateAlias
  - DeleteAliasByAliasId
  - createPayment
  - SetPrimaryPaymentInstrument
  - UpdatePaymentInstrumentStatus
  - DeletePaymentInstrument
  - CreateFavorite
  - GetFavorite
  - DeleteFavorite
  - sendmoney
  - push
  - pull
  - reverse
  - adjustment
  - query
  - summary
---

# Send a real-time payment by alias

Two contracts cooperate: the Alias Directory
(`https://sandbox-api.novopayment.com/aliasdirectory/v1`) holds who can be paid, and Real-Time
Payments (`https://sandbox-api.novopayment.com/realtimepayments/v1`) moves the money. The point of
the alias is that the transaction carries the alias, not the PAN.

## Register the payee

1. `createAlias` (`POST /alias`) creates the alias; `findAlias` (`POST /alias/find`) looks one up
   before you create a duplicate.
2. `createPayment` (`POST /alias/paymentinstruments`) attaches a payment instrument;
   `SetPrimaryPaymentInstrument` (`PUT /alias/paymentinstruments/primary`) picks the default and
   `UpdatePaymentInstrumentStatus` (`PUT /alias/paymentinstruments/status`) enables or disables one.
3. `createAccount` (`POST /alias/account`) links an account to the alias.
4. Favourites for a payer: `CreateFavorite` (`POST /alias/favorites`), `GetFavorite`
   (`GET /alias/{aliasId}/favorites`), `DeleteFavorite`.

`GetAliasByAliasId` (`GET /alias/{aliasId}`) reads the alias back — always do this before paying, so
you pay the instrument that is primary now.

## Move the money

- `sendmoney` (`POST /moneytransfer`) for a full transfer.
- `push` (`POST /moneytransfer/transactions/funds/push`) and `pull`
  (`POST /moneytransfer/transactions/funds/pull`) for the two directions separately.
- NovoPayment states funds are available within 30 minutes. That is a settlement statement, not a
  reversal window.

## Check and correct

- `query` (`GET /query/transactions`) and `summary` (`POST /transactions/summary`) read state. Use
  `query` before any retry — there is no idempotency key on this API, so a blind retry of `push` can
  move the money twice.
- `reverse` (`POST /moneytransfer/transactions/funds/reverse/{transactionId}`) reverses a transfer.
- `adjustment` (`POST /moneytransfer/transactions/adjustments/{transactionId}`) reverses an
  adjustment.
- No reversal deadline is published. If a deadline matters to your user, get it from NovoPayment in
  writing rather than from the docs.

## Removing a payee

`DeletePaymentInstrument` (`DELETE /alias/{aliasId}/paymentinstruments/{paymentInstrumentId}`) and
`DeleteAliasByAliasId` (`DELETE /alias/{aliasId}`). Note `400.09.539 Deletion of the primary payment
instrument is not allowed` — move the primary flag first. Deletion has no published restore path;
treat it as final and re-register if needed.

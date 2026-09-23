---
name: novopayment-merchant-payment-acceptance
description: Authorize a gateway payment, read it back, refund or reverse it, and accept a merchant-presented QR payment with its status notification.
api: NovoPayment Payment Authorizer API, Merchant Presented QR API
provider: novopayment
generated: '2026-09-17'
method: generated
source: openapi/novopayment-payment-authorizer-openapi.yml, openapi/novopayment-merchant-presented-qr-openapi.yml
operations:
  - PaymentDetail
  - notificationsPayment
---

# Accept and settle a merchant payment

Payment Authorizer base: `https://sandbox-api.novopayment.com/paymentauthorizer/v1`.
Merchant Presented QR base: `https://sandbox-api.novopayment.com/api/v1/mpqr`.

## Authorize

`POST /payments` ("Payment Authorizer") authorizes a transaction arriving from a payment gateway.
The request body takes `notifyUrl` — "service address to receive information of the request
(webhook)", max 100 characters. NovoPayment will POST transaction information there, but **the
callback payload shape, retry behaviour and signature scheme are not published**, so build the
receiver defensively and verify every notification by calling back into the API.

`PaymentDetail` (`GET /payments/{referenceCode}`) is the authoritative read.

## Refund and reverse

- `PATCH /payments/{transactionId}` ("Payment Refund") refunds a captured payment.
- `POST /paymentreverse` ("Payment Reverse") reverses an authorization.
- Neither operation declares an `operationId` in the published spec, and neither carries a stated
  time window. Reconcile with `PaymentDetail` first; there is no idempotency key, so a duplicate
  refund is a real risk on a retry.

## Merchant-presented QR

The QR contract adds two credentials on top of the OAuth2 bearer token: an `apikey` query parameter
("public API key, which is different from the shared secret") and an `X-Pay-Token` header that
**expires in 480 seconds for all clients**. `createApiKey` is documented as "do not set to true".

- Generate the code with the QR generation operation, then let the wallet read it.
- `notificationsPayment` (`POST /notifications/payment`) carries the payment status event with
  `srcClientId`, `srcDigitalCardId` and `eventTimestamp`.

## Failures

Authorization failures arrive in the `code` field: `400.08.044` insufficient funds, `400.08.046`
invalid card information, `400.08.101` card not allowed, `400.08.511`–`400.08.520` fraud and
pick-up conditions. See `errors/novopayment-decline-codes.yml`. Show a buyer a generic decline;
the fraud conditions are issuer information.

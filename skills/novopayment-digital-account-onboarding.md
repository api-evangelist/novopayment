---
name: novopayment-digital-account-onboarding
description: Open a digital bank account with NovoPayment end to end — identity document check, onboarding session, OTP, personal and regulatory data capture, KYC, customer record and account creation.
api: NovoPayment Onboarding API, Compliance API, Customers API, Accounts API
provider: novopayment
generated: '2026-09-17'
method: generated
source: openapi/novopayment-onboarding-openapi.yml, openapi/novopayment-compliance-openapi.yml, openapi/novopayment-customers-openapi.yml, openapi/novopayment-accounts-openapi.yml
operations:
  - DocIdVerification
  - DeviceRegistration
  - TokenUpdate
  - OnboardingSession
  - OTPRequestViaSMSOrEmail
  - OTPVerification
  - PersonalInformation
  - ResidenceAddress
  - Occupation
  - IncomeSource
  - ExpectedAccountActivity
  - RegulatoryQuestions
  - UploadSupportDocument
  - GetTermsAndConditions
  - AcceptTermsAndConditions
  - UserCreation
  - ApproveAccount
  - CreateIndividualCustomer
  - AccountCreation
---

# Open a NovoPayment digital bank account

Every call below is a real operation in NovoPayment's published OpenAPI documents. Base URLs are the
sandbox hosts named in those specs; UAT and production hosts are issued by NovoPayment on approval.

## Before you start

1. Get a token. `POST https://sandbox-api.novopayment.com/oauth2/token`, form-encoded, with
   `grant_type=client_credentials`, `client_id` and `client_secret` from your project dashboard.
   The response is a `BearerToken` with `expires_in` 1799 seconds — refresh on ~25 minutes, do not
   assume an hour.
2. Every request carries `X-Tenant-Id`. Most also accept or require `X-Transaction-Id`, a
   client-generated trace id. **It is not an idempotency key** — NovoPayment publishes no replay
   protection, so never blind-retry a POST in this flow; query first (see Recovery).
3. Onboarding base: `https://sandbox-api.novopayment.com/onboarding/v1`.

## The flow

1. **Screen the document** — `DocIdVerification` (`POST /personal/documentverify`) confirms the
   identity document is not already registered with an active status. A rejection here ends the
   flow; do not proceed to create a session.
2. **Register the device** — `DeviceRegistration` (`POST /device`), then `TokenUpdate`
   (`POST /device/pushtoken`) to store the Android/iOS push token used for transactional and
   non-transactional notifications.
3. **Open the session** — `OnboardingSession` (`GET /session`). Use `SessionSummary`
   (`POST /session/resume`) to pick a session back up rather than starting a new one; a second
   session for the same applicant is a duplicate you cannot cancel.
4. **Verify the applicant** — `OTPRequestViaSMSOrEmail` (`POST /personal/otp/type/{typeId}`) then
   `OTPVerification` (`POST /personal/otp/verify/{typeId}`).
5. **Capture the profile** — `PersonalInformation` (`POST /personal/info`), `ResidenceAddress`
   (`POST /personal/address`), `Occupation` (`POST /occupation/dependent`) or
   `OtherTypeOfOccupation` (`POST /occupation/other`), `IncomeSource` (`POST /income`).
6. **Capture the regulatory answers** — `ExpectedAccountActivity` (`POST /expectedactivity`) and
   `RegulatoryQuestions` (`POST /regulatoryassessment`). Biometrics run through
   `GetBiometricApplicantId` (`POST /personal/applicant`).
7. **Supporting documents** — `UploadSupportDocument` (`POST /documents`); read back with
   `GetCustomersSupportDocuments` (`GET /documents`).
8. **Terms** — `GetTermsAndConditions` (`GET /terms`) then `AcceptTermsAndConditions`
   (`POST /terms`). Record what version was accepted; NovoPayment does not version it for you.
9. **KYC** — `POST /customers/{customerId}/info` on the Compliance API
   (`https://sandbox-api.novopayment.com/kyc/v1`) for facial, document and address verification.
10. **Create the records** — `UserCreation` (`POST /usercreation`), then `ApproveAccount`
    (`POST /customers/{customerId}/onboarding/decision`). For a customer created outside the
    onboarding funnel use `CreateIndividualCustomer` (`POST /individuals`),
    `CreateCommercialCustomers` (`POST /commercial`) or `CreateMerchantCustomers` (`POST /merchant`)
    on the Customers API.
11. **Open the account** — `AccountCreation`
    (`POST /customers/{customerId}/accounts` on `https://sandbox-api.novopayment.com/accounts/v1`),
    which can create the account and its card together.

## Reading the result

Every response — success or failure — carries `{code, message, datetime}`. The ten-character `code`
is the outcome, not the HTTP status: `200.12.000` is a processed onboarding call, `400.xx.xxx` is a
rejection. The full registry is in `errors/novopayment-error-codes.yml`.

## Recovery

There is no idempotency key and no dry-run mode anywhere in this API. If a POST times out:

- for a customer, call `CustomerList` (`GET /customers`) or
  `CustomerGeneralInformation` (`GET /customers/{customerId}/generalinfo`) before retrying;
- for an account, call `AccountList` (`GET /customers/{customerId}/accounts`);
- for a session, call `SessionSummary` rather than `OnboardingSession`.

Onboarding has no published cancel or rollback operation. `UnsubscribeCustomer`
(`POST /customers/{customerId}/unsubscribe`) exists on the Customers API but NovoPayment does not
document it as an undo of onboarding, so do not treat it as one.

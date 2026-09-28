---
title: "LTPS COF refunds need mutually exclusive selector forms"
date: "2026-09-23"
last_updated: "2026-09-23"
category: "integration-issues"
module: "plugin-eft-impl"
problem_type: "integration_issue"
component: "payments"
symptoms:
  - "External LTPS card-on-file refunds fail when the member and card-account identifiers are unavailable."
  - "Sending incomplete account/member selectors cannot identify the registered LTPS transaction."
root_cause: "wrong_api"
resolution_type: "code_fix"
severity: "medium"
tags: [ltps, card-on-file, refunds, transaction-identity, selector-routing]
related_components: ["server-core-impl", "plugin-eft", "api-soap-endpoint"]
---

# LTPS COF refunds need mutually exclusive selector forms

## Problem

An external LTPS card-on-file payment can retain its provider transaction identity while lacking recoverable member and card-account identifiers at refund time. The existing external COF refund path always used the account/member form for `retail-refund`, so the fallback case could not be refunded even though the original LTPS transaction was known.

## Symptoms

- Full refunds that are still voidable must continue to use `void-sale`; the fallback applies only when the flow reaches `retail-refund`.
- Partial or already-processed refunds need a provider request identified by the original transaction ID when account/member selectors are absent or non-numeric.

## What Didn't Work

- Reusing the existing account/member overload for every COF refund leaves the provider with incomplete selectors when `PersonDO.getExternalId()` or the external account ID is missing.
- Matching by refund amount would not establish the identity of the original payment and would risk targeting the wrong transaction. The existing clearing-house reference already supplies the LTPS transaction ID.

## Solution

Keep the persisted external reference as the transaction identity and make the LTPS client expose two distinct refund construction paths:

```java
// Existing selector form
ltpsClient.createRefund(amount, profitCenter, memberId, accountId);

// Fallback form: tranId only
ltpsClient.createRefund(amount, profitCenter, originalTransactionId);
```

`LtpsAPIPaymentService.executeCardOnFileRefund` first parses and validates the original clearing-house reference, then preserves the existing void decision. On the `retail-refund` branch it calls the transaction-ID overload when either selector is missing, blank, or not numeric (`plugin-eft-impl/src/main/java/com/exerp/clublead/gl/payment/creditcard/ltps/LtpsAPIPaymentService.java:162-219`). Otherwise it retains the account/member overload (`:194-199`).

`LtpsClient` builds the fallback request with `LtpsAuthRequest(terminal, amount, tranId)` (`plugin-eft-impl/src/main/java/com/exerp/clublead/gl/payment/creditcard/ltps/client/LtpsClient.java:62-75,161-172`). That request constructor sets `cardHolderMemberId` and `accountId` to `null` and sets `tranId` (`plugin-eft-impl/src/main/java/com/exerp/clublead/gl/payment/creditcard/ltps/client/LtpsAuthRequest.java:98-114`), so the serialized request has one selector shape instead of a mixture of fields.

The original identity is already carried through the external COF request as `CardOnFileRefundRequestDO.originalExternalReference` (`plugin-eft/src/main/java/dk/procard/eclub/gl/payment/eft/dataobjects/CardOnFileRefundRequestDO.java:11-37`). The SOAP registration contract exposes the source external reference (`api-soap-endpoint/src/main/java/dk/procard/eclub/api/v5/RegisterExternalPaymentParameters.java:19,59-65`), and the registration implementation maps it into `PaymentInformation.paymentReference` (`api-soap-impl/src/main/java/dk/procard/eclub/api/ShoppingBasketAPIImpl.java:114-122`).

## Why This Works

LTPS accepts different identifiers for the same refund endpoint, but the selector forms are mutually exclusive. The overloads encode that contract at the Java request-construction boundary: the account/member constructor nulls `tranId`, while the transaction-ID constructor nulls both account/member fields. Routing chooses the fallback when either existing selector is missing, blank, or non-numeric, while full voidable refunds remain on the pre-existing void path. Provider failures still become `RefundNotAddedException` in the external COF flow (`LtpsAPIPaymentService.java:177-205`), so a rejected request cannot be reported as a successful refund.

## Prevention

- Test both serialized request shapes and assert absence of the mutually exclusive selector fields, not only the presence of the selected field.
- Keep selector choice in the LTPS client/service boundary; do not make callers assemble a request containing nullable combinations.
- Treat missing, blank, and non-numeric identifiers as unavailable and fail closed if the original transaction identity is also invalid (`LtpsAPIPaymentService.java:164-175,208-218`).
- Preserve the ordering of transaction lookup, void eligibility, and refund selection so a full voidable refund is not changed into a `retail-refund`.
- The focused regression coverage includes `LtpsAPIPaymentServiceTest` for fallback routing and `LtpsClientTest` for LTPS request construction; the fallback service test verifies the transaction-ID overload and that the four-argument overload is not called (`plugin-eft-impl/src/test/java/com/exerp/clublead/gl/payment/creditcard/ltps/LtpsAPIPaymentServiceTest.java:285-299`).
- `LtpsClientTest.create_refund_success_with_transaction_id_selector` asserts that the fallback JSON contains `tranId` and omits both account/member selector fields (`plugin-eft-impl/src/test/java/com/exerp/clublead/gl/payment/creditcard/ltps/client/LtpsClientTest.java:214-228`).

## Verification Notes

The current tree confirms the learning's key boundaries: `LtpsAPIPaymentService` keeps lookup and void evaluation ahead of selector choice (`plugin-eft-impl/src/main/java/com/exerp/clublead/gl/payment/creditcard/ltps/LtpsAPIPaymentService.java:177-205`), and the focused service and client tests cover the fallback interaction and serialized-field exclusion (`plugin-eft-impl/src/test/java/com/exerp/clublead/gl/payment/creditcard/ltps/LtpsAPIPaymentServiceTest.java:284-299`, `plugin-eft-impl/src/test/java/com/exerp/clublead/gl/payment/creditcard/ltps/client/LtpsClientTest.java:214-228`). No test command was run as part of this documentation capture.

The refreshed checkout-level verification also confirms that `hasUsableSelectors` rejects null, blank, and non-numeric member/account values before choosing the transaction-ID overload (`LtpsAPIPaymentService.java:191-218`), while the transaction-ID request constructor explicitly nulls both account/member fields before serialization (`LtpsAuthRequest.java:98-114`). The repository's mechanical claims validator could not compare these paths with `origin/main` because that ref is unavailable in this checkout; the paths are present and tracked in the current tree.

## Related Issues

- [DEV-77094 implementation plan](../../plans/DEV-77094-ltps-cof-refund-research-plan.md)
- The implementation and documentation are present in the current tree under the `DEV-77094` work item.

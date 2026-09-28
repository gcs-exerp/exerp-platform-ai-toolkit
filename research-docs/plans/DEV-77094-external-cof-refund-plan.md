---
artifact_contract: ce-unified-plan/v1
product_contract_source: ce-brainstorm
jira_key: DEV-77094
title: "DEV-77094: Refund external COF online transactions"
status: requirements-only
---

# DEV-77094: Refund external COF online transactions

## Goal Capsule

### Objective

Allow an external integrator to fully or partially refund an online payment made
with LTPS card on file (COF) when the integrator registered the payment through
`ShoppingBasketAPI.registerExternalPayment` and cannot provide the LTPS account
ID or card-holder member ID at refund time.

### Primary actor

The primary actor is an external integrator using Exerp's ShoppingBasket API.
LTPS Clearinghouse is the external payment system involved in the refund.

### Success signal

For an eligible external COF payment, Exerp can identify the original LTPS
transaction and complete the requested full or partial refund through the LTPS
retail refund flow without requiring account ID or card-holder member ID.

## Product Contract

### Summary

External COF payments registered through the ShoppingBasket API must remain
refundable after the sale is committed, even when the external integrator does
not have the LTPS account ID and card-holder member ID. The original LTPS
transaction identifier is the refund identity for this case.

### Problem Frame

DEV-77094 reports that the external integrator can register an external payment
with `externalCardOnFile: true`, but cannot supply the account ID and
`cardHolderMemberId` expected by the current LTPS refund request. The Jira
description states that LTPS added an optional `tranId` field to the retail
refund request so it can be used instead of those two identifiers.

Repository evidence shows the current path stores a credit-card transaction ID
and an external payment reference during external COF capture, then constructs
`CardOnFileRefundRequestDO` with an external account ID. The LTPS implementation
subsequently calls `createRefund` with member/account identifiers. The existing
path therefore does not carry the original transaction identifier through to
the LTPS refund request.

### Requirements

#### Refund identity

- **R1.** For an external COF payment whose original LTPS transaction identifier
  is available, the refund flow MUST be able to use that identifier as the
  primary LTPS refund identity.
- **R2.** Supplying the transaction identifier MUST allow a refund without an
  LTPS account ID or card-holder member ID.
- **R3.** The LTPS refund request MUST NOT combine the transaction identifier
  with the account/member identifier pair in the same request, matching the
  Jira-confirmed LTPS contract.

#### Refund coverage

- **R4.** The behavior MUST support a full refund of an eligible external COF
  transaction.
- **R5.** The behavior MUST support a partial refund of an eligible external
  COF transaction.
- **R6.** Refund requests MUST retain enough original-payment context to
  identify the payment accurately; the amount alone MUST NOT be the only
  identifying information sent to LTPS.

#### Compatibility and eligibility

- **R7.** The change MUST apply to the LTPS external COF refund path described
  by the Jira issue when LTPS Clearinghouse is enabled and the system property
  for fetching COF details from LTPS is enabled.
- **R8.** Existing refund behavior for ordinary card-on-file payments that use
  a payment agreement, and for non-COF payment methods, MUST remain unchanged.
- **R9.** Existing external-payment registration through the ShoppingBasket API
  MUST remain usable with `externalCardOnFile: true`; the refund capability must
  not depend on the external integrator supplying unavailable account/member
  identifiers at registration or refund time.

### Actors and responsibilities

- The external integrator registers the online sale and indicates that the
  payment was made using an external COF.
- Exerp records and later selects the original payment context while creating a
  credit note/refund.
- The LTPS integration accepts the refund using either the transaction identity
  or the existing account/member identity, without mixing the two modes.

### Key flows

#### Full external COF refund

1. An external integrator registers and commits an online payment through the
   ShoppingBasket API with external COF enabled.
2. Exerp retains the original LTPS transaction identity associated with the
   payment.
3. A full refund is requested for the eligible invoice.
4. Exerp sends an LTPS refund request identified by the original transaction
   identity, without account/member identifiers.
5. LTPS accepts the refund and Exerp records the refund result.

#### Partial external COF refund

1. The same payment is selected for a refund smaller than the original amount.
2. Exerp sends the partial refund amount together with the original transaction
   identity.
3. LTPS processes the partial refund and Exerp records the refund result.

#### Existing payment modes

1. A refund is requested for a non-external-COF payment or an ordinary COF
   payment backed by a payment agreement.
2. Exerp continues using the existing applicable refund identity and behavior.

### Acceptance examples

#### AE1 — Full refund without account/member identifiers

Given LTPS Clearinghouse and external COF detail fetching are enabled, and an
online payment was registered through the ShoppingBasket API with external COF
enabled, when a full refund is requested, then the LTPS refund request uses the
original transaction identifier and does not require account ID or
card-holder member ID.

#### AE2 — Partial refund without account/member identifiers

Given the same payment conditions, when a partial refund is requested, then the
LTPS refund request uses the original transaction identifier with the partial
amount and the refund can complete without account ID or card-holder member ID.

#### AE3 — No mixed LTPS identity modes

Given an external COF refund identified by an original transaction identifier,
when the LTPS request is built, then it contains the transaction identity mode
only and does not also contain the account/member identity mode.

#### AE4 — Existing refund behavior remains isolated

Given a refund that is not for an external COF payment, when the refund is
processed, then the request follows the existing payment-method path rather
than being forced through the transaction-identifier mode.

#### AE5 — Accurate payment matching

Given two payments with the same amount, when one is refunded, then the request
contains the original-payment identity needed to distinguish the selected
transaction rather than relying only on amount matching.

### Scope boundaries

In scope:

- External COF payments registered through the ShoppingBasket API.
- LTPS retail refund requests using the original transaction identifier.
- Full and partial refunds.
- The request-mode separation between transaction identity and account/member
  identity.
- Regression coverage for existing refund modes.

Out of scope:

- Changing LTPS endpoints or the external LTPS service contract.
- Changing how non-external-COF payments are refunded.
- Introducing a new external payment provider or a new payment method.
- Changing the ShoppingBasket API's sale-registration purpose or general
  payment validation beyond the information needed to preserve refundability.
- Free credit-note refunds; the current external COF service explicitly rejects
  those, and the Jira issue does not request changing that rule.

### Success criteria

- Full and partial refunds for the Jira-described external COF scenario can be
  completed using the original LTPS transaction identity without account/member
  identifiers.
- The LTPS request never mixes the transaction-identifier and account/member
  modes.
- Existing payment modes retain their current refund behavior.
- The behavior is covered by focused service/client tests for full refund,
  partial refund, identity selection, and regression cases.

### Evidence and known limitations

- Jira `DEV-77094` is an In Progress, P1 Story with labels
  `POS_September_2026`, `September_2026`, and `next`; it has no comments.
- The Jira description states that LTPS added optional `tranId` support and that
  a request must not mix `tranId` with `cardHolderMemberId`/`accountId`.
- The issue's only link is `ST-17070`, a blocking Story with the same summary.
  No linked Confluence page was provided.
- Repository research found no `STRATEGY.md` or `CONCEPTS.md` at the repository
  root and no declared Compound Packs in `.compound-engineering/config.yaml`.
- The exact LTPS request-field serialization for the new `tranId` field is not
  present in the current repository and remains an implementation/planning
  detail to verify against the LTPS client contract.

### Outstanding questions

- **Resolve Before Planning:** Verify the exact LTPS `tranId` JSON type and
  serialization contract, including whether the value is numeric or textual.
- **Deferred to Planning:** Determine the smallest existing data-object and
  client API changes that carry the original transaction identity while keeping
  the existing account/member refund mode compatible.
- **Deferred to Planning:** Identify the precise regression-test fixtures and
  test modules for end-to-end external COF full and partial refunds.

### 🧠 Engineering Decision Log

1. **What was the hardest decision you made here?**

   The hardest decision was to frame this as a refund-identity selection change
   rather than a broader external-payment redesign. The Jira evidence identifies
   one concrete LTPS capability (`tranId`) and one incompatibility (the
   integrator cannot provide account/member identifiers). Repository research
   shows that registration and refund already have an external COF path, so the
   narrowest requirements that solve the reported failure are to preserve and
   use the original transaction identity while leaving other payment modes
   intact.

2. **What alternatives were rejected?**

   Requiring the external integrator to provide `accountId` and
   `cardHolderMemberId` was rejected because the Jira description explicitly
   says those values are unavailable to the integrator for
   `registerExternalPayment`, while LTPS explicitly supports `tranId` instead.
   Replacing the existing refund path for all COF payments was also rejected
   because the repository has a separate payment-agreement-based COF path and
   the Jira scope is limited to external COF payments.

3. **What is the fatal flaw?**

   The fatal flaw is assuming that the transaction identifier currently
   available inside Exerp is exactly the LTPS `tranId` expected by the new
   retail-refund contract. The repository stores several transaction/reference
   values across the cash-register, credit-card, and payment-note layers, and
   the exact LTPS field serialization is not yet present. If that mapping is
   wrong, the request may be structurally valid but refund the wrong payment or
   be rejected by LTPS.

---
artifact_contract: ce-unified-plan/v1
product_contract_source: ce-brainstorm
title: "DEV-73574 - Revoke refund request research plan"
type: research-plan
topic: DEV-73574
created_at: 2026-09-21
execution: code
---

## Goal Capsule

- **Objective:** Establish the verified current behavior, affected surfaces, unresolved product rules, and smallest safe implementation boundary for allowing authorized staff to revoke refund payment requests and restore the member's payment-account credit.
- **Product authority:** Jira DEV-73574, linked to EC-11359, with repository behavior treated as the source of truth for existing flows.
- **Research outcome:** A decision-ready evidence set covering domain service behavior, payment-request UI, Apply Step/API exposure, authorization, transaction matching, state transitions, regression coverage, and documentation impact.
- **Open blocker:** DEV-73574 conflicts with itself on eligible states: the original acceptance text says `Done`, while the kickoff notes include `Done`, `Done Manual`, and `Sent`. The research must not silently choose between them.

## Product Contract

### Summary

Research how to support full revocation of refund payment requests for the DLL use case, preserving existing payment-request revocation behavior while preventing duplicate credit restoration.

### Problem Frame

DEV-73574 reports that an already-completed refund request can fail at the bank or bounce back, leaving the member's payment account without the credit needed to raise the refund again. The ticket identifies an existing negative-amount failure when the current payment-revocation path is used for a refund.

The repository already contains a refund-specific `PaymentRequestFacade.revokeRefundRequest` path and a refund-revocation service branch that uses the absolute received amount for the account-receivable reversal. It also contains a generic `Revoke payment request` Apply Step that currently invokes the ordinary payment-revocation service. The research must verify how these paths interact with state transitions, settlement matching, authorization, and UI enablement before proposing changes.

The linked EC-11359 issue identifies BACS Simple UK and SEPA direct debit as the originating business context. The Jira discussion does not establish that the behavior should be limited to either clearing house, so clearing-house-specific assumptions remain out of scope unless repository evidence requires them.

### Key Decisions

- **Research-first scope:** Produce evidence and a bounded recommendation before implementation planning; do not alter existing behavior during this work.
- **Full-amount refund revocation:** The requested scope excludes partial refund revocation and rejection fees. Governs R9.
- **Separate refund semantics from ordinary payment semantics:** Preserve the existing payment-request revoke flow and investigate a refund-specific path for refund requests.
- **Authorization remains explicit:** Evaluate the requested `Revoke Refund Request Manual` action and its Finance Manager association without assuming that the existing `RevokePaymentRequestManual` action should authorize both behaviors.

### Actors

- **Finance staff:** Authorized staff initiate a refund revocation from the Payment Requests view or the Apply Step flow.
- **Payment-request domain services:** Validate request type/state, update request status, and create the account-receivable transaction.
- **Apply Step/API consumers:** Select persons and execute the refund-revocation operation through the existing Apply Step framework and `ApplyStepAPI.executeApplyStep` path.
- **Clearing-house integrations:** Provide the original refund request context; the research verifies whether their behavior differs for this operation.

### Requirements

#### Research boundaries

- R1. The research must map all existing entry points for refund payment-request revocation, including the Payment Requests UI, the payment-request facade/service, the Apply Step registry, and the Apply Step API.
- R2. The research must trace the current account-receivable transaction and settlement behavior for a completed refund request, including the sign of the original request amount and the reversal amount.
- R3. The research must identify the existing state predicates and state-transition methods relevant to `Done`, `Done Manual`, `Sent`, and revoked or rejected states.
- R4. The research must compare the ticket's requested refund behavior with the existing ordinary payment-revocation behavior and record any compatibility risk.

#### Product and security rules to validate

- R5. A successful full refund revocation must restore the member's payment-account credit with a positive transaction that can support re-raising the refund.
- R6. A repeated revocation attempt must not create a second credit-restoration transaction.
- R7. Multiple independent refund revocations for one member must remain separate transactions with the correct amount per request.
- R8. The final eligible-state rule must be resolved from the DEV-73574 conflict before implementation planning: original acceptance says `Done`; kickoff notes say `Done`, `Done Manual`, and `Sent`, with `Sent` described as status-only rejection behavior.
- R9. Rejection fees and partial refund revocations must not become part of the requested refund-revocation behavior.
- R10. The research must define whether a person without a payment account, a missing request for the requested date, an already-revoked request, or an ineligible state receives the existing error behavior or a refund-specific message.
- R11. The research must verify center-scoped authorization for the proposed refund-revocation action and confirm that unauthorized staff retain the existing missing-privilege behavior.

#### Surfaces and compatibility

- R12. The Payment Requests UI must expose the correct refund-specific label and enablement behavior while preserving ordinary payment-request revocation and its rejection-fee flow.
- R13. The Apply Step must be discoverable and executable through the existing Apply Step framework with the required request-date or equivalent selection semantics documented by the ticket.
- R14. The research must verify whether public API contracts or generated API representations are affected, especially `ApplyStepAPI.executeApplyStep` and available-Apply-Step metadata.
- R15. Regression coverage must include existing ordinary payment revocation, Odin behavior, refund revocation, repeated revocation, multiple refunds for one member, missing account/request, state rejection, authorization, and the documented clearing-house context.
- R16. Documentation impact must be identified for the new manual action and the Apply Step page, without assuming that a feature video or external documentation artifact is produced by implementation.

### Research Questions

- Q1. Which state set is authoritative for refund revocation, and should `Sent` create a transaction or only transition to `Failed, Bank rejected` as described in the kickoff notes?
- Q2. Should the new refund action be a distinct action with a Finance Manager default association, or can the existing action be safely reused without widening ordinary payment-revocation permissions?
- Q3. Should the Apply Step select one refund request per person by request date, process all matching refund requests, or use a different disambiguation rule when several requests match?
- Q4. What exact persisted status/resource name should represent an operator-revoked refund, given the ticket's references to `Failed, refund revoked` and `revoked_by_debitor`?
- Q5. Are any clearing-house-specific constraints required for BACS Simple UK or SEPA direct debit, or is the service behavior clearing-house agnostic?
- Q6. Which API contract, if any, is externally observable enough to require compatibility documentation or explicit regression coverage?

### Research Method and Evidence Targets

- **Domain behavior:** Inspect `server-core-impl/src/main/java/dk/procard/eclub/gl/payment/PaymentRequestServiceBean.java`, `server-shared/src/main/java/dk/procard/eclub/gl/payment/PaymentRequestService.java`, and `common-base/src/main/java/dk/procard/eclub/gl/payment/eft/states/PaymentRequestState.java` to document current amount handling, transaction collection states, idempotency guards, and legal state transitions.
- **Existing entry points:** Inspect `server-facade/src/main/java/dk/procard/eclub/gl/payment/PaymentRequestFacade.java` and `server-facade-impl/src/main/java/dk/procard/eclub/gl/payment/PaymentRequestFacadeBean.java` to distinguish ordinary payment revocation from the existing refund-specific facade method.
- **Apply Step:** Inspect `server-core-impl/src/main/java/dk/procard/eclub/reports/exap/ApplyRevokePaymentRequest.java`, `server-core-impl/src/main/java/dk/procard/eclub/reports/exap/ApplyPluginRegistry.java`, and the Apply Step execution/API classes to determine whether reuse is safe or a new step is required.
- **UI and authorization:** Inspect `client/src/main/java/com/exerp/clublead/client/gl/ar/PaymentRequestView.java`, `client/src/main/java/com/exerp/clublead/client/gl/ar/PaymentRequestController.java`, `client/src/main/java/dk/procard/eclub/gl/ar/InvoiceCollectionsUI.java`, and `common-base/src/main/java/dk/procard/eclub/base/Actions.java` for labels, enablement, role checks, and ordinary-flow compatibility.
- **Existing tests:** Inspect `test-behaviour/src/main/resources/com/exerp/clublead/cucumber/capabilities/billing/payment_requests/revoke_payment_request.feature`, its step definitions, and related payment/account-receivable tests to identify reusable fixtures and missing acceptance scenarios.
- **Integration behavior:** Inspect refund and payment message handlers and their tests to determine whether webhook-driven `revoked_by_debitor` behavior shares the same transaction/idempotency invariants.
- **External ticket evidence:** Use DEV-73574, its seven comments, and linked EC-11359 as context; preserve the distinction between ticket assertions and verified repository behavior.

### Acceptance Examples

- AE1. **Given** a refund request is eligible under the resolved state rule, **when** an authorized staff member revokes the full refund, **then** the request reaches the resolved revoked outcome and the payment account receives exactly one positive restoration transaction for the refunded amount. **Covers** R5, R8, R11.
- AE2. **Given** the same refund request has already been revoked, **when** staff attempt revocation again, **then** no additional restoration transaction is created. **Covers** R6.
- AE3. **Given** two separate refund requests exist for one member, **when** both are revoked, **then** each produces its own correctly sized restoration transaction. **Covers** R7.
- AE4. **Given** an ordinary payment request, **when** staff use its existing revoke flow, **then** its current rejection-fee and state behavior remains unchanged. **Covers** R4, R12, R15.
- AE5. **Given** a person lacks a payment account or the requested refund cannot be found or is ineligible, **when** the Apply Step executes, **then** the operation fails with the resolved error contract and does not create a transaction. **Covers** R10, R13.
- AE6. **Given** staff lack the refund-revocation privilege, **when** they view or execute the refund-revocation operation, **then** authorization prevents the operation and preserves the existing privilege feedback pattern. **Covers** R11, R12.

### Scope Boundaries

#### In scope

- Researching full refund-request revocation from the Payment Requests view and Apply Step path.
- Verifying positive payment-account credit restoration, state handling, duplicate protection, authorization, API exposure, and test/documentation impact.
- Covering the BACS Simple UK and SEPA direct-debit context as evidence targets.

#### Deferred to planning

- Exact class/file changes, database changes, endpoint schema changes, registry identifiers, and test command selection.
- The concrete implementation mechanism for idempotency and transaction matching.
- Final UI wording where the ticket provides alternatives such as `refund revoked` and `revoked_by_debitor`.

#### Outside this plan

- Partial refund revocations.
- Rejection fees for refund revocations.
- Changes to the existing ordinary payment-revocation Apply Step or ordinary payment-revocation business rules.
- Clearing-house import-file support; EC-11359 explicitly describes manual processing from a list.

### Resolve Before Planning

- Resolve Q1: eligible refund-request states and the transaction-versus-status-only behavior for `Sent`.
- Resolve Q2: distinct refund authorization action versus reuse of the existing payment-revocation action.
- Resolve Q4: canonical persisted/display status for an operator-revoked refund.

### Sources / Research

- Jira: DEV-73574, “Core Development - Revoke payment request of type refund”; status `In progress`, priority `P1`, updated 2026-09-21.
- Linked Jira: EC-11359, “DLL: Core Development Pilot Phase 2 - Revoke payment request of type refund”; status `In progress`.
- Repository evidence: `server-core-impl/src/main/java/dk/procard/eclub/gl/payment/PaymentRequestServiceBean.java`, `server-core-impl/src/main/java/dk/procard/eclub/reports/exap/ApplyRevokePaymentRequest.java`, `client/src/main/java/com/exerp/clublead/client/gl/ar/PaymentRequestView.java`, and `test-behaviour/src/main/resources/com/exerp/clublead/cucumber/capabilities/billing/payment_requests/revoke_payment_request.feature`.

<!-- ce-section: work-relationships -->

### How This Work Fits Together

This plan covers the research and product-boundary definition for DEV-73574. It does not commit the repository to implementation choices until the three listed product questions are resolved.

- **Enables:** A subsequent implementation plan for refund revocation and its regression coverage.
- **Shares:** Existing payment-request domain services, UI authorization patterns, Apply Step infrastructure, and account-receivable transaction matching.
- **Can proceed independently of:** Documentation wording and feature-video production, once the canonical behavior and status terminology are approved.

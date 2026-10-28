---
title: "Preserve clipcard credit access while gating partial quantities"
date: "2026-09-30"
category: "security-issues"
module: "common-base, eclub2-srv, client, server-core-impl"
problem_type: "security_issue"
component: "authorization"
symptoms:
  - "A new partial-credit action granted only to a selected role would omit staff who already hold BUYOUT_CLIPCARD through other roles."
  - "Employees with direct BUYOUT_CLIPCARD assignments would lose the new quantity capability unless those assignments were migrated too."
root_cause: "missing_permission"
resolution_type: "migration"
severity: "medium"
tags: [clipcard, authorization, action-migration, partial-credit]
related_components: ["client", "server-core-impl", "eclub2-srv"]
---

# Preserve clipcard credit access while gating partial quantities

## Problem

The local DEV-76818 implementation branch adds a separate staff action for selecting a partial clipcard credit quantity. The requirement was later clarified: staff who already have `BUYOUT_CLIPCARD` must also receive the new action, so their existing credit workflow remains available. The initial plan's opt-in-only migration rule was superseded by that correction.

## Symptoms

- Granting the action only to Membership support would leave other existing buyout users without the new quantity capability.
- A role-only migration would miss employees assigned `BUYOUT_CLIPCARD` directly.
- Disabling the client selector alone would not enforce the quantity permission at the server mutation boundary.

## What Didn't Work

The first action registration used ID 654; the repository's reserved ID was 655, so the registration was corrected before the final migration. The initial [implementation plan](../../plans/DEV-76818-clip-credit-quantity-permission-plan.md) also proposed not propagating the permission to existing buyout users. The later compatibility requirement replaced that policy, and the Membership-only implied-role migration was removed.

## Solution

In the local DEV-76818 implementation branch, define `SELECT_PARTIAL_CLIPCARD_CREDIT_QUANTITY` as action ID 655. Keep `BUYOUT_CLIPCARD` as the general clipcard-credit capability and `OVERRIDE_BUYOUT_CLIPCARD` as the independent buyout-fee override capability. The action's documentation reference is [the supplied permission guide](https://clublead.atlassian.net/wiki/x/AQBucAE).

In the versioned upgrade, register action 655 as implied by action 499 (`BUYOUT_CLIPCARD`):

```sql
SELECT newaction(655, 'SelectPartialClipcardCreditQuantity', 499);
```

Also backfill employees with direct assignments of action 499:

```sql
CALL assign_action_to_employee_with_existing_action(655, 499);
```

The local DEV-76818 branch records both statements in PostgreSQL upgrade files `76818.01.postgres.sql` and `76818.02.postgres.sql`; those files are not present in the active `develop` checkout. The same branch adds the action to `Roles.MEMBERSHIPS_SUPPORT` in [Roles.java](../../../../common-base/src/main/java/dk/procard/eclub/base/Roles.java). The migration is intended to cover role-implied grants and direct employee assignments.

The DEV-76818 implementation branch also gates the staff quantity selector using the new action scoped to the credited center and disables partial selection for camp clipcards. Its `ClipcardServiceBean.creditClips` change checks the action for partial non-camp credits, so the client control is not the authorization boundary. In that branch, camp clipcards continue to credit all remaining clips, and fee override continues to require `OVERRIDE_BUYOUT_CLIPCARD`. Relevant source files are [Actions.java](../../../../common-base/src/main/java/dk/procard/eclub/base/Actions.java), [BuyoutClipcardDialog.java](../../../../client/src/main/java/dk/procard/eclub/subscription/clipcards/BuyoutClipcardDialog.java), [ClipcardServiceBean.java](../../../../server-core-impl/src/main/java/com/exerp/clublead/server/subscription/ClipcardServiceBean.java), and [SubscriptionServiceBean.java](../../../../server-core-impl/src/main/java/com/exerp/clublead/server/subscription/SubscriptionServiceBean.java).

## Why This Works

Existing grants can come from role implications or direct employee assignments. The two migration statements are intended to preserve both kinds of existing access: action implication for role-derived grants and explicit backfill for employees who received the old action directly. A role-only migration would leave direct assignments without the new action.

The new permission narrows only quantity selection. Keeping the existing buyout and fee-override actions separate avoids changing the meaning of either capability. Server-side enforcement covers callers that do not use the staff dialog.

## Prevention

- Before splitting an existing permission, inspect both the role implication graph and direct employee-action assignments.
- Migrate each existing grant source explicitly, then test both role-derived and direct-assignment users.
- Verify a reserved action ID before registration and keep the source role catalog and upgrade scripts aligned.
- Pair UI permission gating with a check at the shared server mutation boundary.
- Cover full credits, authorized partial credits, unauthorized partial credits, camp-specific behavior, and independent fee overrides.

## Verification and Status

The local DEV-76818 implementation branch records the action registration, UI and service checks, and tests for staff and REST credit paths. The active checkout for this documentation run is `develop`, which does not contain those changes; this learning describes the implementation-branch history and does not claim the change is merged. Earlier verification recorded in the implementation session reports `RolesAndActionsTest` passing and the affected client, server, integration, and behavior modules compiling. The integration runner could not start under Java 21 because it uses the disabled SecurityManager, and the behavior scenarios were not run without a local server. Runtime verification of the final migration scenarios is therefore not claimed here.

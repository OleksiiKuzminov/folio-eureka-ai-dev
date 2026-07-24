---
name: ecs-user-setup
description: Expert guide for setting up users in Enhanced Consortia Support (ECS) tests. Use when generating or reviewing Cypress tests where ecs_enabled is true in TestRail, or when user mentions consortia/ECS/multi-tenant tests.
---

# ECS User Setup

Use for tests where TestRail has `ecs_enabled: true`, the user mentions consortia / ECS / multi-tenant, the test spans Central / College / University, the test needs affiliation switching or cross-tenant operations, or you are debugging user creation/deletion in a consortia test.

**Do not use for regular non-ECS tests.**

Pattern variants (multi-user, capabilities-based, no-switch, troubleshooting): `references/patterns.md`.

## Constants

| Constant | Tenant |
|---|---|
| `cy.resetTenant()` | Central (Consortia) — the default |
| `Affiliations.College` | member-1 |
| `Affiliations.University` | member-2 |
| `tenantNames.central` | `"Consortia"` |
| `tenantNames.college` | `"College"` |
| `tenantNames.university` | `"University"` |

## Rules

**0. TestRail preconditions are law.** Create each user in the exact tenant the preconditions name — "User A has been created in member-1 tenant" means College, "in central tenant" means Central. ECS rules supplement TestRail requirements; they never override them. When in doubt, follow the preconditions literally.

**1. Creation tenant sets the primary affiliation, and primary affiliation is the login tenant — always.** A user created in College logs into College. So if a test must work in a given tenant from the start without a UI switch, **create the user in that tenant**. Only add an affiliation switch when TestRail actually specifies a login tenant different from the creation tenant.

**2. Central affiliation is automatic; member affiliations are manual.** Users created by admin get Central (Consortia) automatically. Never assign it by hand — `cy.assignAffiliationToUser(Affiliations.Consortia, ...)` is wrong. Use `cy.assignAffiliationToUser()` for College and University only.

**3. Cross-tenant permissions are assigned from inside the target tenant.** Switch context with `cy.resetTenant()` / `cy.setTenant()`, then `cy.assignPermissionsToExistingUser()` (or `cy.assignCapabilitiesToExistingUser()` on Eureka).

**4. Delete each user from the tenant it was created in** — not from where it logged in or worked. A mismatch here is what produces 404s in cleanup.

**5. An affiliation switch is a UI operation, a tenant context switch is an API operation.** `ConsortiumManager.switchActiveAffiliation()` is not interchangeable with `cy.setTenant()`.

## Canonical setup

Two users in different tenants, one of which switches affiliation mid-test.

```javascript
describe('ECS Test Example', () => {
  const testData = {};
  let userA;
  let userB;

  before('Create test data', () => {
    cy.getAdminToken();

    // TestRail: "User A has been created in member-1 tenant with permissions X, Y"
    // Created in College → primary affiliation College → logs into College
    cy.setTenant(Affiliations.College);
    cy.createTempUser([
      Permissions.inventoryAll.gui,
      Permissions.uiQuickMarcQuickMarcBibliographicEditorAll.gui,
    ]).then((userProperties) => {
      userA = userProperties;

      // Central permissions (Central affiliation itself is automatic)
      cy.resetTenant();
      cy.assignPermissionsToExistingUser(userA.userId, [
        Permissions.consortiaSettingsConsortiumManagerView.gui,
      ]);

      // Extra member affiliations, only if TestRail asks for them
      cy.assignAffiliationToUser(Affiliations.University, userA.userId);
    });

    // TestRail: "User B has been created in central tenant with permissions Z"
    cy.resetTenant();
    cy.createTempUser([Permissions.settingsUsersView.gui]).then((userBProperties) => {
      userB = userBProperties;
      cy.assignAffiliationToUser(Affiliations.College, userB.userId);
    });
  });

  it('C123456: Test requiring affiliation switch', {
    tags: ['extendedPathECS', 'spitfire', 'C123456']
  }, () => {
    cy.resetTenant();
    cy.login(userA.username, userA.password, {
      path: TopMenu.inventoryPath,
      waiter: InventoryInstances.waitContentLoading,
    });

    // Logged into College (creation tenant) — switch to Central in the UI
    ConsortiumManager.switchActiveAffiliation(tenantNames.college, tenantNames.central);
  });

  after('Delete test data', () => {
    cy.getAdminToken();

    cy.setTenant(Affiliations.College); // where User A was created
    Users.deleteViaApi(userA.userId);

    cy.resetTenant();                   // where User B was created
    Users.deleteViaApi(userB.userId);
  });
});
```

## Imports

```javascript
import Affiliations, { tenantNames } from '../../../support/dictionary/affiliations';
import ConsortiumManager from '../../../support/fragments/consortium-manager/consortiumManager';
import Users from '../../../support/fragments/users/users';
import Permissions from '../../../support/dictionary/permissions';

// Eureka capabilities approach:
import Capabilities from '../../../support/dictionary/capabilities';
import CapabilitySets from '../../../support/dictionary/capabilitySets';
```

## Validation checklist

- [ ] Every user created in the exact tenant TestRail's preconditions name
- [ ] Central affiliation not assigned manually anywhere
- [ ] Member affiliations assigned where TestRail requires them
- [ ] Affiliation switch present only where TestRail specifies a login tenant ≠ creation tenant
- [ ] Every deletion runs in the user's creation tenant
- [ ] Tags carry the `ECS` suffix (`smokeECS`, `criticalPathECS`)
- [ ] Tenant context switches use `cy.setTenant()` / `cy.resetTenant()`, not `switchActiveAffiliation()`
- [ ] `Affiliations`, `ConsortiumManager`, `tenantNames` imported

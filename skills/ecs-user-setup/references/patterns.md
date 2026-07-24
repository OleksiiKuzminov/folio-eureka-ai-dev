# ECS Setup Pattern Variants

Read this when the scenario is not the standard single-user shape in `SKILL.md`.

## Test works entirely in the user's primary affiliation (no switch)

TestRail: "User created in member-1, works in member-1."

```javascript
before('Create test data', () => {
  cy.getAdminToken();

  // Primary affiliation = College
  cy.setTenant(Affiliations.College);
  cy.createTempUser([permissions_for_college]).then((userProperties) => {
    testUser = userProperties;

    // Central permissions if the test also touches Central (affiliation is automatic)
    cy.resetTenant();
    cy.assignPermissionsToExistingUser(testUser.userId, [permissions_for_central]);
  });
});

it('Test in College tenant', () => {
  cy.setTenant(Affiliations.College);
  cy.login(testUser.username, testUser.password, {
    path: TopMenu.inventoryPath,
    waiter: InventoryInstances.waitContentLoading,
  });

  // Already in College — no affiliation switch
  ConsortiumManager.checkCurrentTenantInTopMenu(tenantNames.college);
});
```

The Central-only variant is the same with `cy.resetTenant()` in place of `cy.setTenant(...)`.

## Test requires an affiliation switch

TestRail specifies a login tenant different from the creation tenant.

```javascript
it('Test switches from College to Central', () => {
  cy.resetTenant();
  cy.login(testUser.username, testUser.password, {
    path: TopMenu.inventoryPath,
    waiter: InventoryInstances.waitContentLoading,
  });

  // Logged into College (primary affiliation) — switch in the UI
  ConsortiumManager.switchActiveAffiliation(tenantNames.college, tenantNames.central);
  ConsortiumManager.checkCurrentTenantInTopMenu(tenantNames.central);
});
```

## Multiple users across different tenants

Each user is created in — and deleted from — its own tenant.

```javascript
let centralUser;
let collegeUser;
let universityUser;

cy.resetTenant();
cy.createTempUser([centralPermissions]).then((user) => {
  centralUser = user;
});

cy.setTenant(Affiliations.College);
cy.createTempUser([collegePermissions]).then((user) => {
  collegeUser = user;
});

cy.setTenant(Affiliations.University);
cy.createTempUser([universityPermissions]).then((user) => {
  universityUser = user;
});

after('Delete test data', () => {
  cy.getAdminToken();
  cy.resetTenant();
  Users.deleteViaApi(centralUser.userId);
  cy.setTenant(Affiliations.College);
  Users.deleteViaApi(collegeUser.userId);
  cy.setTenant(Affiliations.University);
  Users.deleteViaApi(universityUser.userId);
});
```

## Eureka capabilities instead of permissions

```javascript
before('Create test data', () => {
  cy.getAdminToken();

  const capabsToAssign = [Capabilities.settingsEnabled];
  const capabSetsToAssign = [CapabilitySets.uiAuthorizationRolesSettingsView];

  // TestRail: "User has been created in member-1 tenant"
  cy.setTenant(Affiliations.College);
  cy.createTempUser([]).then((userProperties) => {
    testData.user = userProperties;

    cy.assignCapabilitiesToExistingUser(
      testData.user.userId,
      capabsToAssign,
      capabSetsToAssign,
    );

    // Central capabilities (Central affiliation is automatic)
    cy.resetTenant();
    cy.assignCapabilitiesToExistingUser(
      testData.user.userId,
      [Capabilities.settingsEnabled],
      [CapabilitySets.uiConsortiaSettingsView],
    );
  });
});
```

## Troubleshooting

**User switches to a tenant but can't see the expected data.** Check all three: permissions assigned *in that tenant*; affiliation assigned (automatic for Central, manual for members); the UI switch actually performed via `ConsortiumManager.switchActiveAffiliation()`.

**`Users.deleteViaApi()` returns 404 or a permission error.** The tenant context is wrong. Set it to where the user was *created*, not where the user logged in or worked.

## ECS vs non-ECS at a glance

| Aspect | Non-ECS test | ECS test |
|---|---|---|
| User creation | Single tenant (default) | Specific tenant via `cy.setTenant()` |
| Permissions | Assigned during creation | May span multiple tenants |
| Affiliations | Not relevant | Central automatic, members manual |
| Login | Direct to application | May require affiliation switch |
| Deletion | Simple `deleteViaApi()` | Must match creation tenant |
| Tags | `smoke`, `criticalPath` | `smokeECS`, `criticalPathECS` |

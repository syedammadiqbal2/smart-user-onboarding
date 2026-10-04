# Smart User Onboarding (SUO)

Free Salesforce AppExchange app, intended as a 2GP managed package. Solo developer.

## Specs

The specs live outside this repo, in `../.claude/`:

- `PROJECT_KNOWLEDGE.md`: consolidated spec. **Start here.**
- `suotechinicaldocument.md`: detailed v1.1 technical spec (data dictionary, security, test matrix, defects R-01 to R-35).
- `suo details.md`: business overview. Its "Implemented" claims are false; treat it as requirements only.

Where they disagree, the technical spec wins over the business overview, and the corrections below win over both.

## Environment

- Build org: `syedammadiqbal21@gmail.com.suo` (Developer Edition). Deploy straight to it with `sf project deploy start`.
- That org owns the `suo` namespace, so deployed code runs namespaced (`suo__`). `sfdx-project.json` namespace stays empty until packaging.
- No Dev Hub or scratch orgs during development.
- Only Salesforce Platform licenses are free in the org (3); use profile `Standard Platform User` for any real test user.
- Don't create a package until a Partner Business Org exists; the Dev Hub that creates a managed package owns it.
- `org-retrieve/` is a git-ignored metadata retrieve of the org, for reference only. Never deploy it.

## Rules

- Build in small vertical slices: one class plus its test class, deployed and passing before the next.
- Never reference API names through runtime strings, so the code works once the namespace is added.
- No `without sharing`, no passwords or `System.setPassword()`, no callouts, no SOQL/DML in loops, user-facing strings in Custom Labels.
- Treat Salesforce behaviour in the specs as unverified until it has been checked in the org.

## Spec corrections

Verified in Phase 1; details in PROJECT_KNOWLEDGE.md, "Phase 0 and Phase 1 results".

- Native permissions: query `UserPermissionAccess`. `FeatureManagement.checkPermission()` is only for the custom permission.
- LoginHistory: `Status` can't be filtered. Use `GROUP BY UserId, Status` and keep only `Success` counts in memory.
- Outcome recording: a `System.Finalizer` on `SUO_ProvisioningQueueable` replaces `SUO_HistoryFinalizerQueueable` (fixes R-21).
- Tests of the async chain: wrap `startTest`/`stopTest` in `System.runAs`, and catch the exception `stopTest` rethrows on failure.
- "Manage Internal Users" may not be a real standard permission; still unverified.

## Plan

- [x] Phase 0: repo and org setup
- [x] Phase 1: platform checks in the build org
- [ ] Phase 2: metadata (objects, custom metadata, custom permission, permission sets, labels, app/tabs/page)
- [ ] Phase 3: read-side Apex (constants, exceptions, authorization, selectors, search, ranking, candidates, policy, suggestion, validation, preflight, controller reads)
- [ ] Phase 4: provisioning and audit (request service, provisioning Queueable and Finalizer, recovery, submit/status)
- [ ] Phase 5: LWC wizard with Jest tests
- [ ] Phase 6: release readiness, Partner Program, namespace, package, security review

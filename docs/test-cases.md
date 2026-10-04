# Smart User Onboarding: Test Cases

Test cases for v1.1, derived from the specs in `../.claude/` and the Phase 1 platform checks (2026-10-04).

**Levels**

- **Apex**: automated Apex test, runs in the build org.
- **Jest**: automated LWC unit test, runs locally.
- **Manual**: run by hand in the build org (UAT).

**Priority**

- **P1**: security or data integrity. Must pass before any release.
- **P2**: core behaviour.
- **P3**: edge cases and polish.

**Status** starts as `Not run`. Update it to `Pass`, `Fail` or `Blocked` as tests are built and run.

## Test data

Apex tests create all their own data in `@TestSetup` or the test method. They never rely on data from other tests or the org (R-26).

| Fixture                | Purpose                                                                                                     |
| ---------------------- | ----------------------------------------------------------------------------------------------------------- |
| Operator               | User with permission set `SUO_Onboarding_Operator` plus Manage Users, Assign Permission Sets and View Setup |
| Package-only operator  | User with `SUO_Onboarding_Operator` only, no native permissions                                             |
| Auditor                | User with `SUO_Onboarding_Auditor` only                                                                     |
| No-access user         | User with neither                                                                                           |
| Reference user         | Active Standard user with a Role, 2 direct permission sets, 1 PSG, 1 Queue, 1 Public Group                  |
| Role with N users      | Roles holding 0, 3, 5, 12 and 201+ active users                                                             |
| PSG                    | Created in test, made assignable with `Test.calculatePermissionSetGroup()`                                  |
| Queue / Public Group   | `Group` with `Type = 'Queue'` / `'Regular'`                                                                 |
| Excluded / Warn groups | Groups matched by `SUO_Membership_Policy__mdt` test records                                                 |

Known constraints:

- The build org has no free full Salesforce licenses. Real users for manual tests must use the `Standard Platform User` profile (3 licenses free).
- Async-chain Apex tests must wrap `Test.startTest()`…`Test.stopTest()` in `System.runAs(new User(Id = UserInfo.getUserId()))`, otherwise the Finalizer hits `MIXED_DML_OPERATION`. On failure, `Test.stopTest()` rethrows the job's exception; catch it.
- `LoginHistory` cannot be inserted in tests. Ranking logic is tested with a stubbed selector.

---

## 1. Authorization (AUTH)

| ID      | Scenario                                   | Steps / Input                                                                    | Expected                                                                                | Level  | Priority | Status  |
| ------- | ------------------------------------------ | -------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- | ------ | -------- | ------- |
| AUTH-01 | No custom permission                       | As No-access user, call each `@AuraEnabled` method                               | Every call throws the authorization exception with a safe message                       | Apex   | P1       | Not run |
| AUTH-02 | Custom permission via permission set       | As Operator, call `getPreflight()`                                               | Succeeds                                                                                | Apex   | P1       | Not run |
| AUTH-03 | Custom permission via Permission Set Group | Grant `SUO_Manage_User` only through a PSG; call `getPreflight()`                | Succeeds (proves `FeatureManagement.checkPermission` is used, not a PSA query; R-04)    | Apex   | P1       | Not run |
| AUTH-04 | Auditor cannot onboard                     | As Auditor, call `submitOnboarding()`                                            | Authorization exception; no request record created                                      | Apex   | P1       | Not run |
| AUTH-05 | Package-only operator submits              | As Package-only operator, submit a valid request                                 | Blocked with "missing native permission" message; no User created                       | Apex   | P1       | Not run |
| AUTH-06 | Native permission read                     | As Operator, run preflight                                                       | Reports Manage Users, Assign Permission Sets and View Setup from `UserPermissionAccess` | Apex   | P1       | Not run |
| AUTH-07 | Queueable re-authorizes                    | Enqueue provisioning for a user whose custom permission was removed after submit | Job fails safely; request ends `Failed` with an authorization error code                | Apex   | P1       | Not run |
| AUTH-08 | Low-privilege user cannot create an admin  | As Package-only operator, submit with System Administrator profile               | Platform refuses; no User created; safe message                                         | Manual | P1       | Not run |

## 2. Preflight (PRE)

| ID     | Scenario                     | Steps / Input                             | Expected                                                   | Level | Priority | Status  |
| ------ | ---------------------------- | ----------------------------------------- | ---------------------------------------------------------- | ----- | -------- | ------- |
| PRE-01 | All checks pass              | Operator with all native permissions      | Preflight status Ready                                     | Apex  | P2       | Not run |
| PRE-02 | Missing Manage Users         | Operator without Manage Users             | Preflight flags it with remediation text; wizard blocked   | Apex  | P2       | Not run |
| PRE-03 | Missing View Setup           | Operator without View Setup               | Warning "Login frequency unavailable"; wizard still usable | Apex  | P2       | Not run |
| PRE-04 | Ranking disabled in settings | `Enable_Login_History_Ranking__c = false` | Preflight reports recency-only ranking                     | Apex  | P3       | Not run |

## 3. User search (SRCH)

| ID      | Scenario                       | Steps / Input                  | Expected                                          | Level | Priority | Status  |
| ------- | ------------------------------ | ------------------------------ | ------------------------------------------------- | ----- | -------- | ------- |
| SRCH-01 | Blank term                     | `''`, `null`, `'   '`          | Empty list; no query error                        | Apex  | P2       | Not run |
| SRCH-02 | Match by name, username, email | Term matching each field       | Matching user returned in each case               | Apex  | P2       | Not run |
| SRCH-03 | Inactive users excluded        | Matching inactive user         | Not returned                                      | Apex  | P2       | Not run |
| SRCH-04 | External users excluded        | Matching portal/community user | Not returned (`UserType = 'Standard'` only; R-13) | Apex  | P1       | Not run |
| SRCH-05 | Result cap                     | 25 matching users              | Exactly 20 returned, ordered by Name              | Apex  | P2       | Not run |
| SRCH-06 | `%` and `_` treated literally  | Term `50%` and `a_b`           | Only literal matches; no wildcard behaviour       | Apex  | P1       | Not run |
| SRCH-07 | Quote in term                  | `O'Brien`                      | Matches; no SOQL error                            | Apex  | P1       | Not run |
| SRCH-08 | Very long term                 | 10,000 characters              | Truncated to 100; no error                        | Apex  | P2       | Not run |
| SRCH-09 | Role search                    | Term matching a Role name      | Matching Roles returned, capped at 20             | Apex  | P2       | Not run |

## 4. Ranking (RANK)

Recency boundaries: the spec's "within 1 day" etc. is ambiguous (calendar days vs 24-hour periods). Decide in Phase 3 and update these cases.

| ID      | Scenario                       | Steps / Input                                                               | Expected                                                                                                                             | Level | Priority | Status  |
| ------- | ------------------------------ | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ----- | -------- | ------- |
| RANK-01 | Recency, never logged in       | `LastLoginDate = null`                                                      | 0 points                                                                                                                             | Apex  | P2       | Not run |
| RANK-02 | Recency thresholds             | Last login 0.5, 1, 1+ε, 7, 7+ε, 30, 30+ε, 90, 90+ε days ago                 | 60, 60, 50, 50, 35, 35, 15, 15, 5                                                                                                    | Apex  | P2       | Not run |
| RANK-03 | Past dates score correctly     | Last login 3 days ago                                                       | 50, not 60 or 0 (R-07 `daysBetween` direction)                                                                                       | Apex  | P1       | Not run |
| RANK-04 | Datetime, not Date             | Last login 23 hours ago, crossing midnight                                  | Scored from the Datetime (R-08)                                                                                                      | Apex  | P2       | Not run |
| RANK-05 | Frequency thresholds           | 0, 1, 3, 4, 9, 10, 19, 20, 500 successful logins                            | 0, 10, 10, 20, 20, 30, 30, 40, 40                                                                                                    | Apex  | P2       | Not run |
| RANK-06 | Only successful logins counted | Selector returns rows grouped by `UserId, Status` including failed statuses | Only `Success` rows counted; failed counts discarded                                                                                 | Apex  | P1       | Not run |
| RANK-07 | Tie-break order                | Equal scores with different `LastLoginDate`, login counts, names            | Sorted by score, then last login, then count, then Name; deterministic (R-06)                                                        | Apex  | P2       | Not run |
| RANK-08 | LoginHistory unreadable        | Selector throws or user lacks View Setup                                    | Recency-only ranking; `frequencyAvailable = false`; no privilege escalation                                                          | Apex  | P1       | Not run |
| RANK-09 | Query shape                    | Inspect the LoginHistory selector query                                     | Uses `GROUP BY UserId, Status` with a Datetime cutoff; never selects SourceIp, Browser, Platform, LoginGeoId, LoginType, Application | Apex  | P1       | Not run |

## 5. Role candidates (CAND)

| ID      | Scenario                 | Steps / Input                                 | Expected                                  | Level | Priority | Status  |
| ------- | ------------------------ | --------------------------------------------- | ----------------------------------------- | ----- | -------- | ------- |
| CAND-01 | Empty Role               | Role with 0 active users                      | Empty list; friendly message              | Apex  | P2       | Not run |
| CAND-02 | Fewer than display limit | Role with 3 users                             | 3 candidates                              | Apex  | P2       | Not run |
| CAND-03 | Display limit            | Role with 12 users                            | Top 5 by score                            | Apex  | P2       | Not run |
| CAND-04 | Pool truncation          | Role with 201+ users                          | Pool capped at 200; `truncated = true`    | Apex  | P2       | Not run |
| CAND-05 | Inactive users           | Role with active and inactive users           | Inactive never returned                   | Apex  | P2       | Not run |
| CAND-06 | Display limit setting    | `Role_Candidate_Display_Limit__c` = 3, then 9 | 3 shown; 9 is invalid and falls back to 5 | Apex  | P3       | Not run |

## 6. Access suggestions (SUGG)

| ID      | Scenario                  | Steps / Input                                                                                                            | Expected                                                      | Level | Priority | Status  |
| ------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------- | ----- | -------- | ------- |
| SUGG-01 | Profile and Role          | Reference user                                                                                                           | Suggested Profile and Role match reference user               | Apex  | P2       | Not run |
| SUGG-02 | Direct permission sets    | Reference user with 2 direct PS                                                                                          | Both suggested with Label shown, Name kept for audit (R-16)   | Apex  | P2       | Not run |
| SUGG-03 | Profile-owned PS excluded | Reference user's profile PS                                                                                              | Not suggested                                                 | Apex  | P1       | Not run |
| SUGG-04 | No PS duplicated from PSG | Reference user has PSG containing PS-A                                                                                   | PS-A not listed as a direct PS (R-05)                         | Apex  | P1       | Not run |
| SUGG-05 | Expired PS assignment     | PSA with past `ExpirationDate`                                                                                           | Not suggested                                                 | Apex  | P2       | Not run |
| SUGG-06 | PSG suggested             | Reference user with PSG                                                                                                  | PSG suggested with `MasterLabel` (R-15) and status            | Apex  | P2       | Not run |
| SUGG-07 | PSG statuses              | PSG in Updated / Outdated / Updating / Failed                                                                            | Assignable / warn / blocked / blocked                         | Apex  | P2       | Not run |
| SUGG-08 | PSLs                      | Reference user with PSL assignments                                                                                      | PSLs suggested; PSL required by a chosen PS marked `Required` | Apex  | P2       | Not run |
| SUGG-09 | Queues and Public Groups  | Reference user in Queue and Public Group                                                                                 | Both suggested in their own lists                             | Apex  | P2       | Not run |
| SUGG-10 | Other group types ignored | Reference user in Role / Territory groups                                                                                | Not suggested                                                 | Apex  | P1       | Not run |
| SUGG-11 | Excluded groups hidden    | Reference user in a policy-Excluded group                                                                                | Not suggested                                                 | Apex  | P1       | Not run |
| SUGG-12 | High-privilege warnings   | PS with Modify All Data, View All Data, Manage Users, Customize Application, Author Apex, Manage Profiles/PS, View Setup | Each flagged with a warning; not blocked                      | Apex  | P2       | Not run |
| SUGG-13 | High privilege inside PSG | PSG containing a Modify All Data PS                                                                                      | PSG flagged                                                   | Apex  | P2       | Not run |
| SUGG-14 | Bulk queries              | Reference user with 50 PS, 20 PSG, 30 groups                                                                             | Fixed number of SOQL queries regardless of count (R-18)       | Apex  | P2       | Not run |

## 7. Server-side validation and tampering (VAL)

| ID     | Scenario                             | Steps / Input                                                                                | Expected                          | Level | Priority | Status  |
| ------ | ------------------------------------ | -------------------------------------------------------------------------------------------- | --------------------------------- | ----- | -------- | ------- |
| VAL-01 | Wrong sObject type                   | Account Id submitted as Profile Id                                                           | Rejected: invalid reference       | Apex  | P1       | Not run |
| VAL-02 | Nonexistent Id                       | Well-formed but deleted PS Id                                                                | Rejected                          | Apex  | P1       | Not run |
| VAL-03 | Profile-owned PS submitted           | PS with `IsOwnedByProfile = true`                                                            | Rejected                          | Apex  | P1       | Not run |
| VAL-04 | Role-type group                      | Group `Type = 'Role'` submitted as Public Group                                              | Rejected (R-11)                   | Apex  | P1       | Not run |
| VAL-05 | Territory / Organization group       | Group of those types submitted                                                               | Rejected                          | Apex  | P1       | Not run |
| VAL-06 | Queue in Public Group list           | Queue Id in the Public Group list, and vice versa                                            | Rejected: list/type mismatch      | Apex  | P1       | Not run |
| VAL-07 | Excluded group submitted directly    | Policy-Excluded group Id                                                                     | Rejected at validation (R-12)     | Apex  | P1       | Not run |
| VAL-08 | Warn group                           | Policy-Warn group Id                                                                         | Accepted with warning message     | Apex  | P2       | Not run |
| VAL-09 | Blocked PSG status                   | PSG in Updating or Failed                                                                    | Rejected                          | Apex  | P2       | Not run |
| VAL-10 | Inactive PSL / no seats              | PSL with no available licenses                                                               | Rejected with license message     | Apex  | P2       | Not run |
| VAL-11 | PS incompatible with profile licence | PS whose license doesn't match the chosen profile                                            | Rejected or warned per spec       | Apex  | P2       | Not run |
| VAL-12 | Missing required user fields         | Each of first/last name, email, username, alias, time zone, locale, language, encoding blank | Rejected with field-level message | Apex  | P2       | Not run |
| VAL-13 | External profile                     | Customer Community profile chosen                                                            | Rejected: internal users only     | Apex  | P1       | Not run |

## 8. Membership policy (POL)

| ID     | Scenario                | Steps / Input                                                   | Expected                              | Level | Priority | Status  |
| ------ | ----------------------- | --------------------------------------------------------------- | ------------------------------------- | ----- | -------- | ------- |
| POL-01 | No policy               | Group with no matching record                                   | Allow                                 | Apex  | P2       | Not run |
| POL-02 | Allow / Warn / Exclude  | One active record each                                          | Allow; allow + message; hide + reject | Apex  | P1       | Not run |
| POL-03 | Inactive policy ignored | Exclude record with `Is_Active__c = false`                      | Allow                                 | Apex  | P2       | Not run |
| POL-04 | Type must match         | Exclude record for Queue `X`; Public Group also named `X`       | Only the Queue is excluded            | Apex  | P2       | Not run |
| POL-05 | Enforced three times    | Excluded group: suggestion, `validateSelection()`, provisioning | Excluded at all three layers          | Apex  | P1       | Not run |

## 9. Submission and provisioning (PROV)

| ID      | Scenario                           | Steps / Input                                              | Expected                                                                                                                   | Level  | Priority | Status  |
| ------- | ---------------------------------- | ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- | ------ | -------- | ------- |
| PROV-01 | Transaction A                      | Valid submit                                               | Request `Queued` + one snapshot per item (`Requested`); job enqueued; no setup DML                                         | Apex   | P1       | Not run |
| PROV-02 | Full success                       | Valid request with PSL, PS, PSG, Queue, Public Group       | User created; every assignment present; request `Succeeded`; snapshots `Assigned`; `New_User__c` and `Completed_At__c` set | Apex   | P1       | Not run |
| PROV-03 | Assignment order                   | Inspect DML order                                          | User → PSL → PS → PSG → Queue → Public Group                                                                               | Apex   | P2       | Not run |
| PROV-04 | PSG via PermissionSetAssignment    | PSG selected                                               | `PermissionSetAssignment.PermissionSetGroupId` set; no `PermissionSetGroupMember` used (R-03)                              | Apex   | P1       | Not run |
| PROV-05 | Duplicate username                 | Username already in use                                    | No User; request `Failed` with `DUPLICATE_USERNAME` mapped message                                                         | Apex   | P1       | Not run |
| PROV-06 | Failure at each step               | Force failure at PSL, PS, PSG, Queue, Public Group in turn | Full rollback each time: no User, no assignments; request `Failed`; snapshots `Not Assigned`                               | Apex   | P1       | Not run |
| PROV-07 | Idempotency                        | Submit the same `Client_Request_Id__c` twice               | Second submit rejected; one request, one job                                                                               | Apex   | P1       | Not run |
| PROV-08 | Activation email                   | Option on / off                                            | `triggerUserEmail` set accordingly; no password set anywhere                                                               | Apex   | P1       | Not run |
| PROV-09 | Finalizer records outcome          | Success and failure runs                                   | Finalizer updates request and snapshots in its own transaction; no `MIXED_DML_OPERATION`                                   | Apex   | P1       | Not run |
| PROV-10 | Provisioning re-validates          | Change a group to Excluded between submit and job          | Job rejects it; request `Failed`                                                                                           | Apex   | P1       | Not run |
| PROV-11 | Subscriber automation side effects | Org has a User trigger creating a record                   | Documented: package rollback can't undo it; noted in release docs                                                          | Manual | P3       | Not run |

## 10. Audit, errors and recovery (AUD)

| ID     | Scenario                         | Steps / Input                                       | Expected                                                                                                              | Level | Priority | Status  |
| ------ | -------------------------------- | --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- | ----- | -------- | ------- |
| AUD-01 | No stack traces                  | Any failure                                         | `Error_Message__c` and UI payload contain no stack trace, raw exception text, schema names or inaccessible Ids (R-10) | Apex  | P1       | Not run |
| AUD-02 | Stable error codes               | Each failure type                                   | Mapped to a fixed `Error_Code__c`                                                                                     | Apex  | P2       | Not run |
| AUD-03 | No new-user personal data copied | After success                                       | Request and snapshots hold no name/email of the new user, only the `New_User__c` lookup                               | Apex  | P1       | Not run |
| AUD-04 | Snapshot readable after rename   | Rename a PS after onboarding                        | Snapshot still shows the original label, developer name and namespace                                                 | Apex  | P3       | Not run |
| AUD-05 | Stale request recovery           | Request `Queued` older than the timeout             | Recovery scheduler sets `Requires Review`                                                                             | Apex  | P2       | Not run |
| AUD-06 | Fresh request untouched          | Request `Queued` within the timeout                 | Unchanged                                                                                                             | Apex  | P2       | Not run |
| AUD-07 | Operator sees own requests only  | Two operators                                       | Each sees only their own (Private sharing)                                                                            | Apex  | P1       | Not run |
| AUD-08 | Manager and Auditor see all      | Manager / Auditor user                              | Can read all requests and snapshots                                                                                   | Apex  | P2       | Not run |
| AUD-09 | CRUD/FLS enforced                | User without object access reads history            | Blocked (`WITH USER_MODE` / user-mode DML; R-35)                                                                      | Apex  | P1       | Not run |
| AUD-10 | Request status endpoint          | `getRequestStatus()` for another operator's request | No data returned                                                                                                      | Apex  | P1       | Not run |

## 11. Settings (CFG)

| ID     | Scenario                | Steps / Input                                       | Expected                                                      | Level | Priority | Status  |
| ------ | ----------------------- | --------------------------------------------------- | ------------------------------------------------------------- | ----- | -------- | ------- |
| CFG-01 | Defaults                | Default settings record                             | 30-day window, 60-minute timeout, display limit 5, ranking on | Apex  | P2       | Not run |
| CFG-02 | Out-of-range values     | Window 0 and 181; timeout 4 and 1441; limit 0 and 6 | Each falls back to its default                                | Apex  | P2       | Not run |
| CFG-03 | Missing settings record | No record                                           | Defaults used; no error                                       | Apex  | P2       | Not run |

## 12. LWC wizard (UI)

| ID    | Scenario                           | Steps / Input                                        | Expected                                                                                      | Level  | Priority | Status  |
| ----- | ---------------------------------- | ---------------------------------------------------- | --------------------------------------------------------------------------------------------- | ------ | -------- | ------- |
| UI-01 | Preflight states                   | Mock Ready / warning / not authorized                | Correct panel; wizard blocked when not authorized                                             | Jest   | P2       | Not run |
| UI-02 | Mode selection                     | Choose Existing User, then Role                      | Correct search component shown                                                                | Jest   | P2       | Not run |
| UI-03 | Debounced search                   | Type quickly; type 1 character                       | One Apex call after debounce; no call below minimum length                                    | Jest   | P2       | Not run |
| UI-04 | Handler doesn't shadow Apex import | Trigger user search                                  | Apex `searchUsers` called once; no recursion (R-25)                                           | Jest   | P2       | Not run |
| UI-05 | Candidate cards                    | Mock 5 candidates, frequency available / unavailable | Scores shown; frequency hidden with notice when unavailable; truncation notice when truncated | Jest   | P2       | Not run |
| UI-06 | Required user details              | Leave each required field blank                      | Next disabled; field errors shown                                                             | Jest   | P2       | Not run |
| UI-07 | Edit suggestions                   | Remove a PS, add a PS, change Profile                | State and review panel reflect changes                                                        | Jest   | P2       | Not run |
| UI-08 | Required PSL locked                | PSL marked `Required`                                | Cannot be removed while dependent PS selected                                                 | Jest   | P3       | Not run |
| UI-09 | Warnings shown                     | High-privilege and Warn-policy items                 | Warning panel lists each                                                                      | Jest   | P2       | Not run |
| UI-10 | Single submit                      | Double-click Submit                                  | One Apex call; button disabled immediately                                                    | Jest   | P1       | Not run |
| UI-11 | Async status                       | Mock Queued → Succeeded / Failed                     | Status updates; links to User and request on success; safe message on failure                 | Jest   | P2       | Not run |
| UI-12 | Uses returned request Id           | After submit                                         | Status polling uses the server's request Id, not the client UUID (R-22)                       | Jest   | P2       | Not run |
| UI-13 | Labels, not hardcoded strings      | Render each step                                     | Text comes from Custom Labels (R-28)                                                          | Jest   | P3       | Not run |
| UI-14 | Accessibility                      | Keyboard-only run through wizard                     | All steps reachable; focus visible; labels read by screen reader                              | Manual | P2       | Not run |

## 13. End-to-end manual tests (UAT)

Run in the build org with real users on the `Standard Platform User` profile. Deactivate test users afterwards to free the license.

| ID     | Scenario                 | Steps                                                        | Expected                                                   | Priority | Status  |
| ------ | ------------------------ | ------------------------------------------------------------ | ---------------------------------------------------------- | -------- | ------- |
| UAT-01 | Unauthorized user        | Open the Onboard User tab without the package permission set | Clear block message                                        | P1       | Not run |
| UAT-02 | Package permission alone | Operator PS only, no native permissions                      | Preflight shows missing Manage Users; submit impossible    | P1       | Not run |
| UAT-03 | Existing User flow       | Search reference user → details → review → submit            | User created with matching access; request `Succeeded`     | P1       | Not run |
| UAT-04 | Role flow                | Pick Role → candidate → details → review → submit            | Same as UAT-03; candidates ranked by activity              | P1       | Not run |
| UAT-05 | Suggestions accurate     | Compare suggestions with reference user in Setup             | Profile, Role, PSL, PS, PSG, Queue, Public Group all match | P1       | Not run |
| UAT-06 | Edits respected          | Remove one suggestion, add one PS                            | New user has exactly the edited set                        | P2       | Not run |
| UAT-07 | Warnings                 | Choose high-privilege PS and Warn group                      | Warnings shown; submission allowed                         | P2       | Not run |
| UAT-08 | Tampering                | Submit an Excluded group Id via the browser console          | Rejected on the server                                     | P1       | Not run |
| UAT-09 | Duplicate username       | Use an existing username                                     | Safe failure; no partial user                              | P1       | Not run |
| UAT-10 | Audit trail              | Open the request record                                      | Correct status, operator, reference, snapshots             | P1       | Not run |
| UAT-11 | Auditor                  | Log in as Auditor                                            | Sees history; no Onboard User tab; cannot submit           | P1       | Not run |
| UAT-12 | LoginHistory access lost | Remove View Setup from operator                              | Ranking falls back to recency-only with notice             | P2       | Not run |
| UAT-13 | Activation email         | Submit with email option on                                  | New user receives Salesforce's activation email            | P2       | Not run |

## 14. Release gates (REL)

| ID     | Check           | Expected                                                                                 | Priority | Status  |
| ------ | --------------- | ---------------------------------------------------------------------------------------- | -------- | ------- |
| REL-01 | Apex coverage   | ≥ 90% overall, ≥ 80% per class, every test asserts behaviour                             | P1       | Not run |
| REL-02 | Jest coverage   | ≥ 70%                                                                                    | P2       | Not run |
| REL-03 | Lint and format | `npm run lint`, `npm run prettier:verify` pass                                           | P2       | Not run |
| REL-04 | Code Analyzer   | No unexplained High/Critical findings                                                    | P1       | Not run |
| REL-05 | Code search     | No `without sharing`, `System.setPassword`, callouts, `innerHTML`, `eval`, hardcoded Ids | P1       | Not run |
| REL-06 | Clean install   | Package version installs and works in a fresh org                                        | P1       | Not run |
| REL-07 | Uninstall       | Uninstall leaves created users and assignments; documented                               | P2       | Not run |

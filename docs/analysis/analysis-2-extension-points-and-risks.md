# Analysis 2: Extension Points, Reuse, and Risks for a Reconciliation Exceptions Workbench

## Key Takeaways

- Copy the newer standard screens and pop-up grids; avoid the older hand-built pages.
- Existing permissions, alerts and scheduled jobs can be reused almost as they are.
- Bank lines in `FIN_BANKSTATEMENTLINE` have no owner, reason code or status field.
- Import, matching and completing a reconciliation have no automated tests.
- First build a cross-account worklist, exception fields and unattended matching.

## 1. UI Framework and Patterns to Follow

The workbench serves one objective: "reduce manual effort and cycle time in bank reconciliation. Most statement lines are matched without human touch, and every unmatched item has an owner, age, and reason code." Here a statement line is one entry on the statement the bank sends, matching links it to the transaction it settles (the ERP's record of money expected in or out of a financial account, which is the ERP record for one bank account), and a reconciliation is the period document confirming the ERP balance agrees with the bank.

### 1.1 Technology Baseline

- **Server:** Java (the server's programming language), version 11 or later as `build.xml` enforces, built with Apache Ant (a build tool), version 1.9.2 or later.
- **Screens:** JavaScript (the browser's programming language) on Isomorphic SmartClient (a library of browser grids and forms), in `modules/org.openbravo.userinterface.smartclient`. Each screen is described in the Application Dictionary (the metadata that defines screens, tables and processes), for example `AD_WINDOW`, `AD_TAB`, `AD_FIELD` and `OBUIAPP_PROCESS`.
- **Data:** PostgreSQL (the database), with each table's structure kept as XML (structured text) files under `src-db/database/model` and in each module's (a separately packaged part of the ERP) own database folder.
- **Java tests:** JUnit 4.13.2 (a Java test framework) and Mockito 4.11.0 (a tool that fakes collaborators in tests), both in `lib/test`, with tests in `src-test/src` and in module folders such as `modules/org.openbravo.client.application/src-test`.
- **Test runs:** `src-test/build.xml` runs two test suites (named bundles of tests): `run.all.tests` runs `StandaloneTestSuite`, `run.webservice.tests` runs `WebserviceTestSuite`, and `run.test` runs any single test, including those in neither suite.
- **User-interface tests:** Jest 29.7 (a JavaScript test runner), declared in `package.json`, runs three utility test files in `modules/org.openbravo.client.application/web-test`. `ViewGeneration`, in `StandaloneTestSuite`, only checks that screen definitions generate without errors. No test clicks through a screen, so new workbench screens start with no automated screen-interaction tests.

### 1.2 Patterns for Windows, Views and Dashboards

| Pattern | Workbench use | Example in the code |
|---------|---------------|---------------------|
| Standard window (a data-entry screen with a header and sub-tabs) defined in the Application Dictionary | Exception records with their owner, reason and status | `Financial Account` window |
| Process definition, Pick and Execute (a pop-up grid where the user selects rows and runs an action) | Bulk assign, set reason, resolve | `Match Statement` (`OBUIAPP_PickAndExecute`), built on `BaseProcessActionHandler` |
| HQL-based table (a read-only list built from a database query) | One worklist across all financial accounts, showing age | `aprm_matchstatement`, the Match Statement grid |
| Manual view (a custom-built screen registered in the Application Dictionary) | Only if a bespoke layout is needed | `OBUIAPP_AlertManagement` |
| Workspace and Query/List widgets (small panels on the Workspace dashboard, the latter showing a query's result) | Counts by age, owner and reason | `OBKMO_WIDGET_CLASS`, `OBCQL_WIDGET_QUERY` |
| Classic servlet (an older, hand-coded web page) | Avoid for new work; Analysis 1, Section 1.2 shows where these sit today | `ImportBankFile`, `Reconciliation` |

Screens defined in the Application Dictionary get filtering, grid behaviour and role-based access (a role is a named set of screen and process permissions) without custom code. Servlets built on `HttpSecureAppServlet` need every page written by hand, which makes them slower to build and harder to keep consistent.

## 2. Reusable Components

Reuse fit is High when the component meets the workbench need through configuration or new records only, Medium when it exists but must be switched on or extended, and Low when it would need replacing.

| Component | What exists | Reuse fit (High/Medium/Low) |
|-----------|-------------|-----------------------------|
| Roles and permissions | Each role is granted windows, processes and process definitions (`AD_WINDOW_ACCESS`, `AD_PROCESS_ACCESS`, `OBUIAPP_PROCESS_ACCESS`), and role inheritance (one role taking over another's grants) is handled by `RoleInheritanceManager`. The standard Finance role in `modules/org.openbravo.v3.datasets` holds the reconciliation windows (Bank File Format, Matching Algorithm, Financial Account) and processes (Reconciliation Details, Reconcile, Import Statement, Reconciliation Summary). Roles are also granted organizations (companies or branches) in `AD_ROLE_ORGACCESS`, limiting which records they see, but no grant assigns records an owner, so ownership must come from the data | High |
| Audit trail | An audit trail (a log of who changed which field, and when) is kept in `AD_AUDIT_TRAIL` and shown in the `Audit Trail` window and per record by `AuditTrailPopup`. It is switched on table by table (`ISFULLYAUDITED`) and is off by default. Reconciliation tables such as `FIN_BANKSTATEMENTLINE` already record who created and last changed each row, and when (`CREATED`, `UPDATED`) | Medium (see the note below the table) |
| Notifications | Alerts (system-generated to-do messages sent to a user or role) come from rules written as database queries (`AD_ALERTRULE`) and carry a status, user, role and note (`AD_ALERT`), with optional email (`AD_ALERTRECIPIENT`). `AlertProcess`, scheduled in the core setup data, raises one alert per new item and closes it once the rule stops returning that item, so an alert on an unmatched line would clear itself when the line is matched. Users acknowledge or suppress alerts in the Alert Management view; a WebSocket (live browser connection) client, `MessageClientManager`, and free-text record notes, `OBUIAPP_NOTE`, also exist | High |
| Background processes and scheduling | Background processes (jobs the system runs on a schedule without a user) run on a Quartz-backed scheduler (built on the Quartz job library), `OBScheduler`, through process requests (saved schedules for one job) in `AD_PROCESS_REQUEST`, set in the `Process Request` window, each run logged in `AD_PROCESS_RUN`, reviewed in `Process Monitor`. `FIN_BankStatementProcess` and `FIN_ReconciliationProcess` are already written in the form the scheduler runs. Automatic matching, however, sits inside a screen action, `MatchStatementOnLoadActionHandler`, so it must be moved out before it can run without a user | High |
| Reporting | The reconciliation PDF, `ReportReconciliation`, lists outstanding payments, outstanding deposits and unreconciled statement lines. Newer reports run as process definitions through `BaseReportActionHandler`, and Query/List widgets (`OBCQL_WIDGET_QUERY`) can show counts. No report breaks unmatched lines down by age, owner or reason | Medium |

**Unconfirmed:** whether reconciliation tables are audited today, because auditing is switched on table by table in each installation and is off by default (consulted `AD_TABLE`, `ISFULLYAUDITED`, `AD_AUDIT_TRAIL`).

## 3. Data Model: Required Changes vs Current Support

Analysis 1, Section 4.3 describes the gaps; this table pairs each need with what the schema (the database structure) already holds and what must be added, naming needs rather than designs.

| Current schema already supports | Change required |
|---------------------------------|-----------------|
| Unmatched state: a line in `FIN_BANKSTATEMENTLINE` with an empty `FIN_FINACC_TRANSACTION_ID` | An explicit exception status, for example open, assigned or resolved |
| Age, derived from the bank date `DATETRX` or the record creation time `CREATED` | None; age is calculated, not stored |
| Who created or last changed a line (`CREATEDBY`, `UPDATEDBY`) | An owner (a user) for each unmatched item |
| Free text in `DESCRIPTION` | A list of reason codes and a reason on each item |
| Match strength in `MATCHINGTYPE`: AU strong (found by the algorithm's primary rules), AP weak (found only by its looser fallback rules), MA matched by hand, and AD linked to a transaction added by hand; plus what the line was matched against in `MATCHED_DOCUMENT` | A confirmation flag, needed only for the weak-match queue in Section 5.2 of this document |
| Record notes (`OBUIAPP_NOTE`) and change history (`AD_AUDIT_TRAIL`) | Auditing switched on for the new fields |
| Three on/off switches and a pluggable rule program (`CLASSNAME`) in `FIN_MATCHING_ALGORITHM`, the matching algorithm record (a saved set of matching rules that one or more financial accounts use) | Amount and date tolerance settings, needed only for tolerance matching in Section 5.2 of this document |
| A trigger (a database rule run on every change), `APRM_FIN_BNKSTM_LINE_CHECK_TRG`, which locks only the original bank fields once a statement is processed | None; new exception fields stay editable after processing |

## 4. Integration Points and Regression Risks

An area is tested when a test in the repository exercises that area's own behaviour, directly or through helper code, wherever the test sits; otherwise it is untested. A shared mechanism tested on another table or process does not make a reconciliation area tested, so each label says in brackets what the test covers, whether a suite runs it, and what stays uncovered.

| Area | Integration point | Regression risk | Tested |
|------|-------------------|-----------------|--------|
| Statement import | `ImportBankFile`, `FIN_BankStatementImport`, installed bank file format modules (add-ons that each read one bank's file layout) | New line fields must not break the format modules that create lines | untested (no test refers to either class) |
| Statement processing | `FIN_BankStatementProcess`, `APRM_FIN_BNKSTM_LINE_CHECK_TRG` | Date checks and line locks can block exception updates that alter bank data or add or remove processed lines | untested (no test refers to either) |
| Automatic matching | `StandardMatchingAlgorithm`, `MatchTransactionDao`; see Analysis 1, Section 3 | Moving or changing matching alters which lines clear and can create wrong strong matches; editing a shared `FIN_MATCHING_ALGORITHM` record changes every account that uses it | untested (no test refers to the matching classes) |
| Manual match, split, unmatch | `APRM_MatchingUtility` | Splits, merges and deletion of algorithm-created records must keep exception data consistent | untested (no test refers to it) |
| Match Statement grid | `aprm_matchstatement` | Extra query columns can slow or break the grid | untested (`HQLDataSourceTest`, in `WebserviceTestSuite`, covers HQL-based tables only on `Aprm_Credit_To_Use`; the Match Statement query and its speed are not covered) |
| Reconciliation completion and balances | `FIN_ReconciliationProcess`, `FIN_ReconciliationEventListener` | Holding completion for open exceptions changes how later balances are recalculated | untested (no test refers to either class) |
| Posting | `DocFINReconciliation`, `AcctServerProcess` | Write-off or adjustment actions create entries that posting (creating the accounting entries in the general ledger) must handle | tested (`RecordID2Test`, in `StandaloneTestSuite`, removes and recreates one sample reconciliation's entries and checks balances; posting of new adjustment entries is not covered) |
| Payments feeding candidates | `FIN_PaymentProcess`, `FIN_TransactionProcess` | Payment status changes alter the transactions matching can choose from | tested (payment processing, by `CancelAndReplaceTest` in `StandaloneTestSuite` and by the payment tests through `TestUtility`, runnable only through `run.test`; how matching uses the resulting transactions is not covered) |
| Roles and access | `AD_WINDOW_ACCESS`, `OBUIAPP_PROCESS_ACCESS` | New screens need grants for the Finance role | tested (role inheritance and access checks, `RoleInheritanceTestSuite` in `StandaloneTestSuite`; grants for new screens are not covered) |
| Alerts | `AlertProcess`, `AD_ALERTRULE` | Errors in a rule's query appear only when it runs | untested (alert rules appear only as stored data, for example in `DalLockingTest`; no test runs `AlertProcess`) |
| Scheduling | `OBScheduler`, `ProcessRunner`, `AD_PROCESS_REQUEST` | A scheduled matching run can collide with users editing the grid | tested (scheduler only: `ProcessSchedulingTest` with test-only jobs, `TriggerProviderTest` and `MisfirePolicyTest`, all in `StandaloneTestSuite`; matching alongside user edits is not covered) |
| Reporting | `ReportReconciliation` and its JasperReports (report-layout engine) templates | Template changes stop the report from building | tested (building only: `AllJrxmlCompilation`, in `StandaloneTestSuite`; the reconciliation report's content is not covered) |
| Audit trail | `AD_AUDIT_TRAIL`, `ISFULLYAUDITED` | Auditing busy tables adds database write load | untested (`ETagGeneration` switches auditing on `C_ORDER` only to test browser page caching; no test reads the audit trail) |

Import, matching, manual resolution and completion, the areas the workbench touches most, are all untested, so no test would catch a change breaking them.

## 5. Recommended Capability Set

Impact is High when a capability itself raises the share of lines matched with no person acting, or itself gives unmatched items an owner, age or reason code; Medium when it cuts the effort or cycle time of work people still do; Low when it only measures, records or displays the work. Effort S means configuration or metadata on an existing component; M means new fields, a screen, a process definition or a background process built from existing patterns; L means new or changed matching logic in an untested area.

Each tier is ranked by impact (High, Medium, Low), then effort (S, M, L); ties keep the order of the effort sources in Analysis 1, Section 5, lowest item number first, then the order shown. Item numbers below refer to those effort sources, and each capability names the existing part it reuses. **Assumption:** effort sizes assume reuse of the patterns and components in Sections 1 and 2 of this document.

### 5.1 Minimum-Viable Tier

These capabilities give every unmatched item an owner, age and reason, and match lines without waiting for a user.

| Rank | Capability | Impact | Effort |
|------|------------|--------|--------|
| 1 | Cross-account exceptions worklist showing each unmatched line's age (items 1 and 4; an HQL-based table like `aprm_matchstatement`) | High | M |
| 2 | Owner, reason code and status on every unmatched line (item 4; new fields on `FIN_BANKSTATEMENTLINE`) | High | M |
| 3 | Automatic matching after each import as a scheduled background process (item 5; `OBScheduler`) | High | M |
| 4 | Alerts to owners for aging items and to the Finance role for unassigned items (item 4; `AD_ALERTRULE`, `AlertProcess`, `AD_ALERTRECIPIENT`) | Medium | S |
| 5 | Dashboard widget with exception counts by age, owner and reason (item 4; `OBCQL_WIDGET_QUERY`) | Low | S |

### 5.2 Full-Scope Tier

These capabilities add to the minimum-viable tier.

| Rank | Capability | Impact | Effort |
|------|------------|--------|--------|
| 1 | Tolerance matching: amount and date windows and partial partner names (item 2; a new rule set through `FIN_MatchingAlgorithm`) | High | L |
| 2 | Matching rules learned from manual resolutions (item 2; the partner-guessing approach in `FIN_BankStatementImport`) | High | L |
| 3 | Bulk actions: assign, set reason, create bank-fee transactions (item 1; `AddTransactionActionHandler`) | Medium | M |
| 4 | Queue to accept or reject weak matches in bulk (item 3; `MATCHINGTYPE` in a Pick and Execute grid) | Medium | M |
| 5 | Audit history on the exception fields (item 4; `AD_AUDIT_TRAIL`) | Low | S |
| 6 | Reporting of the share of lines matched untouched and of cycle time (items 1 and 5; `BaseReportActionHandler`) | Low | M |

The weak-match queue is Medium, not High, because weak matches are already applied automatically (`MatchStatementOnLoadActionHandler`), so the queue cuts correction effort rather than raising the share of lines matched with no person acting.

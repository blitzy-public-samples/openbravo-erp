# Analysis 1: Current-State Reconciliation Workflow and Manual Effort

## Key Takeaways

- A bank file reaches the general ledger in eight steps, most of them started by a person.
- People act at 17 points, and most recurring ones sit in the `Match Statement` screen.
- The automatic rules need exact values, allow no tolerance and take the first candidate.
- Unmatched lines carry no owner, reason code or status in `FIN_BANKSTATEMENTLINE`.
- The biggest effort is resolving, one line at a time, every bank line the rules miss.

## 1. End-to-End Flow from Statement Import to Posted Reconciliation

### 1.1 How the Pieces Fit

The business objective is to "reduce manual effort and cycle time in bank reconciliation. Most statement lines are matched without human touch, and every unmatched item has an owner, age, and reason code." Apart from its core tables and posting, the flow lives in one module (a separately packaged part of the ERP), Advanced Payables and Receivables Management, `modules/org.openbravo.advpaymentmngt`.

Everything hangs off the financial account (the ERP record for one bank account), kept in the `Financial Account` window (a data-entry screen). It holds bank statements and their bank statement lines (one entry on the statement the bank sends) in `FIN_BANKSTATEMENT` and `FIN_BANKSTATEMENTLINE`, transactions (the ERP's record of money expected in or out of a financial account) in `FIN_FINACC_TRANSACTION`, and reconciliations (period documents confirming the ERP balance agrees with the bank) in `FIN_RECONCILIATION`. Every list, rule and document is therefore scoped to one account at a time, which matters for any screen meant to work exceptions across accounts.

### 1.2 Step-by-Step Flow

1. A finance user clicks Import Bank File on the `Financial Account` window, picks a bank file format (one bank's file layout, kept in `FIN_BANKFILE_FORMAT`) and uploads the file through `ImportBankFile`, an older hand-built screen.
2. The format's importer (the program that reads that layout), built on `FIN_BankStatementImport`, creates the statement and its lines in `FIN_BANKSTATEMENT` and `FIN_BANKSTATEMENTLINE`, dropping zero-amount lines. It guesses each line's business partner (customer or supplier) from earlier lines, the exact name, or the closest name once the words in `FIN_BANKFILE_EXCEPTION` are removed.\
   **Unconfirmed:** which bank file formats an installation can import, because no code in this repository implements the importer, so formats come from separately installed modules (consulted `FIN_BankStatementImport`, `ImportBankFile`, `APRM_NoBankFileAvailableInfo`).
3. The import processes the statement at once (`FIN_BankStatementProcess`); a line dated on or before the latest line of earlier processed statements (or, if none, the last reconciliation's end date) raises a warning and leaves the statement unprocessed.
4. The user opens `Match Statement`, a process definition (Pick and Execute: a pop-up grid where the user selects rows and runs an action); `MatchStatementOnLoadActionHandler` reuses the account's draft reconciliation in `FIN_RECONCILIATION` or creates one.
5. A prompt (`APRM_AlgorithmConfirm`) offers to run matching (linking a statement line to the transaction it settles); if accepted, the account's rules, shipped as `StandardMatchingAlgorithm`, link lines to transactions in `FIN_FINACC_TRANSACTION`, saving each match at once.
6. The user resolves the remaining lines in the `Match Statement P&E` grid: search and link (`FindTransactionsToMatchActionHandler`), add a missing transaction (`AddTransactionActionHandler`), or undo a wrong link (unmatch).
7. When the user clicks Reconcile, `MatchStatementActionHandler` calls `FIN_ReconciliationProcess`, which checks that each cleared (linked) transaction falls in an open accounting period (a span open for bookings) and completes the reconciliation, even with lines still unmatched.
8. Posting (creating the accounting entries in the general ledger) records the reconciliation's entries in `FACT_ACCT` through `DocFINReconciliation`, from the Post button or from `Acct Server Process`, a background process (a job the system runs on a schedule without a user).\
   **Unconfirmed:** whether posting a reconciliation produces entries, because that depends on how each payment method books cleared items and on the configured accounts (consulted `DocFINReconciliation`, `DocFINBankStatement`).\
   **Unconfirmed:** whether and how often posting runs on a schedule, because the data shipped with the core product sets no schedule for it and only the QA (test) sample data runs it every five minutes, so each installation sets its own (consulted `AcctServerProcess`, `src-db/database/sourcedata/referencedData/AD_PROCESS_REQUEST.xml`, `referencedata/sampledata/QA_Testing/AD_PROCESS_REQUEST.xml`).

## 2. Manual User Actions and Their Triggers

**Assumption:** users holding the standard Finance role (a named set of screen and process permissions) perform every step below except the format-module install. **Unconfirmed:** which people hold that role, because role assignments are installation data (consulted `modules/org.openbravo.v3.datasets/referencedata/standard/V3_Roles.xml`, `AD_USER_ROLES`).

| Step | Who acts | Trigger | Window / class |
|------|----------|---------|----------------|
| 1. Install a bank file format module and apply its dataset (packaged setup data) | System Administrator | Import warns that no format is installed | `ImportBankFile` |
| 2. Maintain words excluded from partner guessing | Finance user | Partners guessed wrongly at import | `Bank File Format`, `FIN_BANKFILE_EXCEPTION` |
| 3. Choose the account's matching algorithm record (a saved set of matching rules that one or more financial accounts use) | Finance user | New account, or it should match differently | `Financial Account` |
| 4. Edit a record's three on/off switches, which changes every account that selects it | Finance user | Matching too strict or too loose | `Matching Algorithm` |
| 5. Import the bank file | Finance user | Statement received from the bank | `Financial Account`, `ImportBankFile` |
| 6. Enter or correct a statement by hand, then process it | Finance user | No file, or file rejected | `Imported Bank Statements`, `Bank Statement Lines` |
| 7. Force-process a statement despite earlier dates | Finance user | Date warning when processing | `EM_APRM_PROCESS_BS_FORCE` |
| 8. Accept or decline the automatic matching run | Finance user | Prompt each time the screen opens | `Match Statement` |
| 9. Unmatch a wrong match, one row or a selection | Finance user | Match judged wrong; the only cue is the Affinity column (the grid's match-strength code) | `Match Statement P&E` |
| 10. Search for and link transactions, splitting the line when amounts differ | Finance user | Line left unmatched | `APRM_MatchingUtility` |
| 11. Create and link a missing bank fee, payment or GL item (a general-ledger category for amounts with no invoice) | Finance user | No transaction exists for the line | `AddTransactionActionHandler` |
| 12. Reconcile, or save and leave the draft open | Finance user | Matching finished for the period | `MatchStatementActionHandler` |
| 13. Reconcile without a statement: enter date and ending balance, tick cleared transactions, optionally book a difference to a GL item | Finance user | Account reconciled without a bank file | `EM_APRM_RECONCILE` |
| 14. Reactivate (reopen for editing) a statement or the latest reconciliation | Finance user | Error found after completion | `FIN_BankStatementProcess`, `FIN_ReconciliationProcess` |
| 15. Force-reactivate an earlier reconciliation that later ones follow | Finance user | Correction in a processed, non-latest, non-manual reconciliation, the only case the button shows | `Reconciliation Process Force` |
| 16. Print the reconciliation details or summary | Finance user | Printout needed | `ReportReconciliationDetail`, `ReportReconciliationSummary` |
| 17. Post the reconciliation | Finance user | Completed reconciliation not yet posted | `DocFINReconciliation` |

Steps 5, 8 and 12 recur with every statement and steps 9 to 11 with every line the rules miss, so they drive effort; the rest are setup or corrections.

## 3. Existing Matching Logic

### 3.1 Criteria and Algorithm

The standard rules (`StandardMatchingAlgorithm`, searching through `MatchTransactionDao`) try these passes in order and keep the first candidate found. A strong match is a candidate found by the algorithm's primary rules (passes 1 and 3), a weak match one found only by its looser fallback rules (passes 2 and 4).

| Pass | Applies when | What must agree | Result |
|------|--------------|-----------------|--------|
| 1 | Line carries a GL item | Same GL item, exact amount, transaction dated on or before the line | Strong (`AU`) |
| 2 | Pass 1 found nothing | Same GL item and exact amount, any date; if this also fails, the line goes on to pass 3 | Weak (`AP`) |
| 3 | Every line not matched by pass 1 or 2 | Exact amount, plus each criterion whose switch is on: exact date (date switch); line reference equal to the payment's reference or document number (reference switch, only if the line's reference is neither empty nor a double asterisk); identical partner name (name switch). A switched-off criterion is not compared, so a difference in it does not prevent a strong match | Strong (`AU`) |
| 4 | Pass 3 found nothing | Exact amount, plus exact date if the date switch is on | Weak (`AP`) |
| 5 | Nothing found | — | No match (`NO`) |

Candidates are processed, unreconciled transactions of the same account that are not already cleared (status `RPPC`), minus those matched earlier in the run; the result levels come from `FIN_MatchedTransaction`.

### 3.2 Configurability

- Each financial account selects one matching algorithm record (`FIN_MATCHING_ALGORITHM_ID` on `FIN_FINANCIAL_ACCOUNT`). The record, kept in the `Matching Algorithm` window and `FIN_MATCHING_ALGORITHM`, names the program that applies the rules (`CLASSNAME`) and holds the three switches (`MATCHBPNAME`, `MATCHREFERENCE`, `MATCHTRANSACTIONDATE`). Choosing another record changes one account, but editing a record changes every account that selects it, so one edit can silently shift matching for many accounts.
- A different rule set can be plugged in through the `FIN_MatchingAlgorithm` interface (the contract every rule set follows) without changing the flow.
- New records default to name on, reference on and date off; the F&B International Group sample `Standard` record (`referencedata/sampledata/F_B_International_Group/FIN_MATCHING_ALGORITHM.xml`) turns all three off, so any same-amount candidate becomes a strong match.
- **Unconfirmed:** which record each account selects and that record's switches, because both are configuration data that can be edited at any time (consulted `FIN_FINANCIAL_ACCOUNT`, `FIN_MATCHING_ALGORITHM`, `StandardMatchingAlgorithm`).

### 3.3 Limits

- No amount or date tolerance, so a one-cent or one-day difference in a compared field sends the line to weak matching or manual work.
- With the name switch on, partner names must be identical, so spelling differences defeat the strong rule.
- The first candidate wins, with no scoring between same-amount candidates, so a plausible but wrong link can be made.
- Automatic matching links one line to one transaction; linking one line to several needs a manual search that splits it (`APRM_MatchingUtility`).
- Matching runs only when a user opens `Match Statement` and accepts the prompt, so lines wait for a person.
- Weak matches are applied with no confirmation step and only the Affinity column as a cue, so wrong links surface only on review.
- Every grid change is saved at once (`APRM_GRID_PERSIST_MESSAGE`), so there is no batch undo.

## 4. Current Handling of Unmatched and Exception Items

### 4.1 What Counts as Unmatched

An unmatched item is a line of a processed statement with no linked transaction (`FIN_FINACC_TRANSACTION_ID` empty). Such lines show in the Match Statement grid, in the `Bank Statement Lines` tab (all lines, with no filter on matched status) and in the unreconciled-lines part of the reconciliation PDF (`ReportReconciliation`). None of these lists unmatched lines across accounts, so no one sees the whole backlog or its age.

### 4.2 How Exceptions Are Resolved

- Search and link, with an automatic split when amounts differ; splitting is blocked once the reconciliation is posted (`APRM_SplitBSLReconciliationPosted`).
- Add a transaction, which is linked with the match level `AD` (manually added).
- Unmatch, which merges split parts back and deletes any transaction or payment a matching algorithm created, with its accounting entries if posted (`APRM_MatchingUtility`).
- Carry forward: a reconciliation completes with unmatched lines left, and they reappear in the next reconciliation's pending list (`FIN_ReconciliationProcess`) with nothing marking them as old.

### 4.3 What Is Not Recorded

- `FIN_BANKSTATEMENTLINE` has no owner, reason code, exception status or due date; age can only be derived from `DATETRX` or `CREATED`, and the line's only free-text field is `DESCRIPTION`. This is the gap against "every unmatched item has an owner, age, and reason code".
- No active alert (a system-generated to-do message sent to a user or role) watches statement lines; the one shipped statement rule, "Old not processed Bank Statements", is inactive and reads the older `C_BANKSTATEMENT` table.
- Checks stop work rather than queue it: a closed period (`APRM_PeriodNotAvailableClearedItem`), date order (`APRM_BankStatementLineWrongDateWarning`, bypassed by force processing), mixing manual and statement reconciliation (`APRM_ReconciliationMixed`), and reactivation limits for a statement with a matched line (`APRM_BSLineReconciled`), an open draft (`APRM_DraftReconciliationExists`) or a later reconciliation (`APRM_FutureReconciliationExists`, bypassed by the force process).

## 5. Five Largest Sources of Manual Effort

**Assumption:** the ranking reflects how many statement lines each source touches along the code paths, not measured volumes. **Unconfirmed:** actual match rates, line volumes and cycle times, because the repository stores no metrics and they depend on production data (consulted `MatchStatementOnLoadActionHandler`, which only shows the matched count as a message).

1. **Line-by-line resolution of every line the rules miss** — each search, add or split acts on one statement line (Section 2, steps 9 to 11), so every miss needs a person and directly blocks "matched without human touch" (`FindTransactionsToMatchActionHandler`, `AddTransactionActionHandler`).
2. **Exact-equality rules with no tolerance, where the first candidate wins** — any difference in a field the applicable pass compares (Section 3.1) sends the line to weak matching or manual work, capping how many lines are "matched without human touch" (`MatchTransactionDao`, `StandardMatchingAlgorithm`, `FIN_MATCHING_ALGORITHM`).
3. **Correcting wrong automatic matches** — weak matches are applied unconfirmed with only the Affinity column as a cue (Section 3.3), and each wrong one costs a manual unmatch that may also delete records the algorithm created (Section 4.2), adding to the manual effort the objective aims to reduce (`FIN_MatchedTransaction`, `MatchStatementOnLoadActionHandler`, `UnMatchSelectedTransactionsActionHandler`).
4. **No structured exception tracking** — unmatched lines carry only free text, with no owner, reason code or status (Section 4.3), and roll into the next reconciliation unflagged, so "every unmatched item has an owner, age, and reason code" cannot be met (`FIN_BANKSTATEMENTLINE`, `FIN_ReconciliationProcess`).
5. **Matching only on demand, behind a confirmation prompt** — nothing is matched between import and the next user session, which adds waiting time to the cycle time the objective aims to cut (`APRM_AlgorithmConfirm`, `Match Statement`).

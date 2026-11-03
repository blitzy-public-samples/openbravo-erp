# Analysis 1: Current-State Reconciliation Workflow and Manual Effort

## Key Takeaways

- A bank file reaches the accounting records (`FACT_ACCT`) in eight steps, most started by a person.
- People act at 17 points; the line-by-line ones sit in the `Match Statement` screen.
- The automatic rules (`StandardMatchingAlgorithm`) need exact values and take the first match found.
- Unmatched lines carry no owner, cause or status in `FIN_BANKSTATEMENTLINE`.
- The biggest effort is resolving, one at a time, bank lines the rules never match (`APRM_MatchingUtility`).

## 1. End-to-End Flow from Statement Import to Posted Reconciliation

### 1.1 How the Pieces Fit

The business objective is to "reduce manual effort and cycle time in bank reconciliation. Most statement lines are matched without human touch, and every unmatched item has an owner, age, and reason code." A reconciliation is the period document confirming the ERP (business-management software) balance agrees with the bank (`FIN_RECONCILIATION`), and a reason code records why an item is unresolved.

Apart from its data tables (stores of records) and accounting step (step 8), both in the core product, the flow lives in one module (a separately packaged part of the ERP), Advanced Payables and Receivables Management, `modules/org.openbravo.advpaymentmngt`. Everything hangs off the financial account (the ERP record for one bank account), kept in the `Financial Account` window (a data-entry screen): bank statements and their bank statement lines (one entry on the statement the bank sends) in `FIN_BANKSTATEMENT` and `FIN_BANKSTATEMENTLINE`, transactions (the ERP's record of money expected in or out of a financial account) in `FIN_FINACC_TRANSACTION`, and reconciliations in `FIN_RECONCILIATION`. These records and their lists therefore cover one account at a time (`FIN_FINANCIAL_ACCOUNT_ID`; rule records can be shared, Section 3.2), which matters for any screen meant to work exceptions (unmatched lines) across accounts.

### 1.2 Step-by-Step Flow

1. A finance user clicks Import Bank File on the `Financial Account` window, picks a bank file format (one bank's file layout, kept in `FIN_BANKFILE_FORMAT`) and uploads the file through `ImportBankFile`, an older hand-built screen.
2. The format's importer (the program that reads that layout), built on the `FIN_BankStatementImport` class (a program unit), creates the statement and its lines in `FIN_BANKSTATEMENT` and `FIN_BANKSTATEMENTLINE`, dropping zero-amount lines. It guesses each line's business partner (customer or supplier) from earlier lines, the exact name, or, ignoring the words in `FIN_BANKFILE_EXCEPTION`, the only partner sharing a name word; with several such partners, closest-name selection fails, leaving the partner blank.\
   **Unconfirmed:** which bank file formats an installation can import, because no code in this repository implements the importer, so formats come from separately installed modules (consulted `FIN_BankStatementImport`, `ImportBankFile`, `APRM_NoBankFileAvailableInfo`).
3. The import processes (confirms) the statement at once (`FIN_BankStatementProcess`); a line dated on or before the latest line of earlier processed statements (or, if none, the last reconciliation's end date) raises a warning and leaves the statement unprocessed.
4. The user opens `Match Statement`, a process definition (Pick and Execute: a pop-up grid, or on-screen table, where the user selects rows and runs an action); `MatchStatementOnLoadActionHandler` reuses the account's draft (unfinished) reconciliation in `FIN_RECONCILIATION` or creates one.
5. A prompt (`APRM_AlgorithmConfirm`) offers to run matching (linking a statement line to the transaction it settles); if accepted, the account's rules, shipped as `StandardMatchingAlgorithm`, link lines to transactions in `FIN_FINACC_TRANSACTION`, saving each match at once.
6. The user resolves the remaining lines in the `Match Statement P&E` grid: search and link (`FindTransactionsToMatchActionHandler`), add a missing transaction (`AddTransactionActionHandler`), or undo a wrong link (unmatch).
7. When the user clicks Reconcile, `MatchStatementActionHandler` calls `FIN_ReconciliationProcess`, which checks, for organizations set up for accounting, that each cleared (linked) transaction falls in an open accounting period (a span open for bookings) and completes the reconciliation, even with lines still unmatched.
8. Posting (creating the accounting entries in the general ledger, the company's books) records the reconciliation's entries in `FACT_ACCT` through `DocFINReconciliation`, from the Post button or `Acct Server Process`, a background process (a job the system runs on a schedule without a user).\
   **Unconfirmed:** whether posting a reconciliation produces entries, because that depends on how each payment method books cleared items and on the configured accounts (consulted `DocFINReconciliation`, `DocFINBankStatement`).\
   **Unconfirmed:** whether and how often posting runs on a schedule, because the data shipped with the core product sets no schedule for it and only the QA (test) sample data runs it every five minutes, so each installation sets its own (consulted `AcctServerProcess`, `src-db/database/sourcedata/referencedData/AD_PROCESS_REQUEST.xml`, `referencedata/sampledata/QA_Testing/AD_PROCESS_REQUEST.xml`).

## 2. Manual User Actions and Their Triggers

**Assumption:** users holding the standard Finance role (a named set of screen and process permissions) perform every step below except the format-module install and steps 2 and 4 (view-only for this role in `AD_WINDOW_ACCESS`). **Unconfirmed:** which people hold that role, because role assignments are installation data (consulted `modules/org.openbravo.v3.datasets/referencedata/standard/V3_Roles.xml`, `AD_USER_ROLES`).

| Step | Who acts | Trigger | Window / class |
|------|----------|---------|----------------|
| 1. Install a bank file format module and apply its dataset (packaged setup data) | System Administrator | Import warns that no format is installed | `ImportBankFile` |
| 2. Maintain words excluded from partner guessing | User with edit access | Partners guessed wrongly at import | `Bank File Format`, `FIN_BANKFILE_EXCEPTION` |
| 3. Choose the account's matching algorithm record (a saved set of matching rules that one or more financial accounts use) | Finance user | New account, or it should match differently | `Financial Account` |
| 4. Edit a record's three on/off switches, which changes every account that selects it | User with edit access | Matching too strict or too loose | `Matching Algorithm` |
| 5. Import the bank file | Finance user | Statement received from the bank | `Financial Account`, `ImportBankFile` |
| 6. Enter or correct a statement by hand, then process it | Finance user | No file, or file rejected | `Imported Bank Statements`, `Bank Statement Lines` |
| 7. Force-process a statement despite earlier dates | Finance user | Date warning when processing | `EM_APRM_PROCESS_BS_FORCE` |
| 8. Accept or decline the automatic matching run | Finance user | Prompt each time the screen opens | `Match Statement` |
| 9. Unmatch a wrong match, one row or a selection | Finance user | Linked transaction judged wrong; only the Affinity column (the grid's match-strength code) flags looser matches | `Match Statement P&E` |
| 10. Search for and link transactions, splitting the line into parts when amounts differ | Finance user | Line left unmatched | `APRM_MatchingUtility` |
| 11. Create and link a missing transaction for a bank fee, existing payment or GL item (a general-ledger category for amounts with no invoice) | Finance user | No transaction exists for the line | `AddTransactionActionHandler` |
| 12. Reconcile, or save and leave the draft open | Finance user | Matching finished for the period | `MatchStatementActionHandler` |
| 13. Reconcile without a statement: enter date and ending balance, tick cleared transactions, optionally book a difference to a GL item, then save the draft or process it | Finance user | Account reconciled without a bank file | `EM_APRM_RECONCILE` |
| 14. Reactivate (reopen for editing) a statement or the latest reconciliation | Finance user | Error found after completion | `FIN_BankStatementProcess`, `FIN_ReconciliationProcess` |
| 15. Force-reactivate an earlier reconciliation that later ones follow | Finance user | Correction in a processed, non-latest, non-manual reconciliation, the only case the button shows | `Reconciliation Process Force` |
| 16. Print the reconciliation details or summary | Finance user | Printout needed | `ReportReconciliationDetail`, `ReportReconciliationSummary` |
| 17. Post the reconciliation | Finance user | Completed reconciliation not yet posted | `DocFINReconciliation` |

Steps 5, 8 and 12 recur with every statement, step 9 with every wrong match and steps 10 and 11 with every line the rules never match (`ImportBankFile`, `Match Statement`), so they drive effort; the rest are setup or corrections.

## 3. Existing Matching Logic

### 3.1 Criteria and Algorithm

The standard rules (`StandardMatchingAlgorithm`, searching through `MatchTransactionDao`) try these passes (rounds of checks) in order and keep the first candidate, a transaction that could settle the line. A strong match is a candidate found by the algorithm's primary rules (passes 1 and 3), a weak match one found only by its looser fallback rules (passes 2 and 4; `FIN_MatchedTransaction`).

| Pass | Applies when | What must agree | Result |
|------|--------------|-----------------|--------|
| 1 | Line carries a GL item | Same GL item, exact amount, transaction dated on or before the line | Strong (`AU`) |
| 2 | Pass 1 found nothing | Same GL item and exact amount, any date; if this also fails, the line goes on to pass 3 | Weak (`AP`) |
| 3 | Every line not matched by pass 1 or 2 | Exact amount, plus each criterion whose switch is on: exact date (date switch); the bank's identifier for the line equal to the identifier entered on the payment or its ERP-assigned number (reference switch, only if the bank's identifier, a mandatory field, is neither empty text nor a double asterisk); identical partner name (name switch). A switched-off criterion is not compared, so a difference in it does not prevent a strong match | Strong (`AU`) |
| 4 | Pass 3 found nothing | Exact amount, plus exact date if the date switch is on | Weak (`AP`) |
| 5 | Nothing found | — | No match (`NO`) |

Candidates are processed, unreconciled transactions of the same account that are not already cleared (status `RPPC`), minus those matched earlier in the run.

### 3.2 Configurability

- Each financial account can select one matching algorithm record (`FIN_MATCHING_ALGORITHM_ID` on `FIN_FINANCIAL_ACCOUNT`). The record, kept in the `Matching Algorithm` window and `FIN_MATCHING_ALGORITHM`, names the program that applies the rules (`CLASSNAME`) and holds the three switches (`MATCHBPNAME`, `MATCHREFERENCE`, `MATCHTRANSACTIONDATE`). Choosing another record changes one account, but editing a record changes every account that selects it (`FIN_MATCHING_ALGORITHM_ID`), so one edit can silently shift matching for many accounts.
- A different rule set can be plugged in through the `FIN_MatchingAlgorithm` interface (the contract every rule set follows) without changing the flow.
- Left unset, a record's switches default to name and reference on, date off, though the window starts new records with all three on; the F&B International Group sample `Standard` record (`referencedata/sampledata/F_B_International_Group/FIN_MATCHING_ALGORITHM.xml`) turns all three off, so pass 3 makes any same-amount candidate a strong match.
- **Unconfirmed:** which record each account selects and that record's switches, because both are configuration data that can be edited at any time (consulted `FIN_FINANCIAL_ACCOUNT`, `FIN_MATCHING_ALGORITHM`, `StandardMatchingAlgorithm`).

### 3.3 Limits

- No amount or date tolerance (allowed difference) in `MatchTransactionDao`, so a one-cent difference, or a one-day difference in an exact-date pass, leaves the line unmatched for manual work.
- With the name switch (`MATCHBPNAME`) on, pass 3 needs identical partner names, so spelling differences defeat that strong rule.
- The first candidate wins (`StandardMatchingAlgorithm`), with no scoring between same-amount candidates, so a plausible but wrong link can be made.
- Automatic matching links one line to one transaction; linking one line to several needs a manual search or add that splits it (`APRM_MatchingUtility`).
- Automatic matching runs only when a user opens `Match Statement` and accepts the prompt, so lines wait for a person.
- Weak matches are applied with no confirmation step and only the Affinity column as a cue, so wrong links surface only on review (`MatchStatementOnLoadActionHandler`).
- Every grid change is saved at once (`APRM_GRID_PERSIST_MESSAGE`), so changes cannot be undone together.

## 4. Current Handling of Unmatched and Exception Items

### 4.1 What Counts as Unmatched

An unmatched item is a line of a processed statement with no linked transaction (`FIN_FINACC_TRANSACTION_ID` empty). Such lines show in the Match Statement grid, in the `Bank Statement Lines` tab (all lines, matched or not) and in the unreconciled-lines part of the reconciliation printout (`ReportReconciliation`). None of these lists unmatched lines across accounts (`FIN_FINANCIAL_ACCOUNT_ID`), so no one sees the whole backlog or its age.

### 4.2 How Exceptions Are Resolved

- Search and link, with an automatic split when amounts differ; splitting is blocked once the reconciliation is posted (`APRM_SplitBSLReconciliationPosted`).
- Add a transaction, which is linked with the match level `AD` (manually added).
- Unmatch, which merges the line with its still-unmatched split parts and, under the standard rules, deletes an algorithm-created transaction, and its payment if also algorithm-created, once reactivated, with accounting entries if posted (`APRM_MatchingUtility`, `StandardMatchingAlgorithm`).
- Carry forward: a reconciliation completes with unmatched lines left, and they reappear in the next reconciliation's pending list (`FIN_ReconciliationProcess`) with nothing marking them as old.

### 4.3 What Is Not Recorded

- `FIN_BANKSTATEMENTLINE` has no owner, reason code, exception status or due date, the gap against "every unmatched item has an owner, age, and reason code"; age can only be derived from the line's transaction date (`DATETRX`) or entry date (`CREATED`), and its only general-purpose note field is `DESCRIPTION`.
- No active alert (a system-generated to-do message sent to a user or role) watches statement lines; the one shipped statement rule, "Old not processed Bank Statements", is inactive and reads the older `C_BANKSTATEMENT` table.
- Checks stop work rather than queue it: closed period (`APRM_PeriodNotAvailableClearedItem`, where accounting applies), date order (`APRM_BankStatementLineWrongDateWarning`, bypassed by force processing), mixing manual and statement reconciliation (`APRM_ReconciliationMixed`, blocking manual entry; Match Statement repairs it first), and reactivation limits for a statement with a matched line (`APRM_BSLineReconciled`, bypassed by force processing unless posted), an open draft (`APRM_DraftReconciliationExists`) or a later reconciliation (`APRM_FutureReconciliationExists`, bypassed by the force process).

## 5. Five Largest Sources of Manual Effort

**What this section tells you:** the five sources of manual effort that the code makes largest, in ranked order. Each item gives its evidence, who is affected, the part of the objective it blocks and why it ranks above the next item. The items build on the findings of Sections 2 to 4 and point back to them rather than repeating them.

**Assumption:** the ranking reflects how many statement lines the code lets each source affect, not measured volumes. **Unconfirmed:** actual match rates, line volumes and cycle times, because the repository stores no metrics and they depend on production data (consulted `MatchStatementOnLoadActionHandler`, which only shows the matched count as a message).

1. **Line-by-line resolution of lines the rules never match** — every line the automatic run leaves open needs a person, one line at a time.
   - **Evidence:** each search-and-link or add-transaction request carries a single statement line (`FindTransactionsToMatchActionHandler`, `AddTransactionActionHandler`), and a search that picks several transactions splits that one line into parts (`APRM_MatchingUtility`). Only unmatching works on a selection of lines (`UnMatchSelectedTransactionsActionHandler`), so the search and add actions in Section 2 have no bulk form.
   - **Who is affected:** finance users working the `Match Statement P&E` grid, who hold the Finance role under the Assumption in Section 2. Readers of the reconciliation printout also see every line left over as unreconciled (`ReportReconciliation`).
   - **Business impact:** each line no later run matches needs a person, so this source directly blocks "matched without human touch". Every such line adds at least one manual action to the effort the objective aims to reduce.
   - **Why it ranks above the next item:** it reaches every line that ends unmatched for any reason, including lines with no transaction to find, such as a bank fee, which no rule can match until a person adds one (`AddTransactionActionHandler`). The exact-value rules of item 2 are one of several causes that feed it.
2. **Exact-equality rules with no tolerance, where the first candidate wins** — any field that fails the applicable pass's test (Section 3.1) sends the line to weak matching or manual work.
   - **Evidence:** every amount comparison in the candidate search is an exact equality, and so are the date, reference and partner-name comparisons the switches turn on (`MatchTransactionDao`). The rules keep the first candidate returned, with no scoring (`StandardMatchingAlgorithm`), and the only settings are the three on/off switches of a shared record (`FIN_MATCHING_ALGORITHM`, Section 3.2).
   - **Who is affected:** finance users, who receive every line a rule rejects. Users with edit access to the `Matching Algorithm` window are affected too, because each switch they change applies to every account that selects the record.
   - **Business impact:** these rules cap how many lines are "matched without human touch", since the limits in Section 3.3 turn a small difference into manual work. Each line they reject adds to item 1, and each wrong first-candidate pick adds to item 3.
   - **Why it ranks above the next item:** these rules decide the outcome of every line in every automatic run, whereas wrong matches arise only on the lines where a same-amount candidate was found (Section 3.1).
3. **Correcting wrong automatic matches** — each wrong link (Section 3.3) must be found and undone by a person (Section 4.2).
   - **Evidence:** the automatic run saves each match at once with no confirmation step (`MatchStatementOnLoadActionHandler`), weak matches included (`FIN_MatchedTransaction`). Undoing one is a manual unmatch, row by row or by selection (`UnMatchSelectedTransactionsActionHandler`), which clears the line's match level (the code recording how it was matched, `MATCHINGTYPE`), so no record keeps the fact that the rules got it wrong (`APRM_MatchingUtility`).
   - **Who is affected:** finance users, who have only the Affinity column (Section 2) to tell weak matches from strong ones while reviewing. Accounting readers are affected too, because a wrong link marks the wrong transaction as cleared (status `RPPC`) and places it in the reconciliation (`APRM_MatchingUtility`).
   - **Business impact:** each wrong match costs a review, an unmatch and a fresh search or add, adding to the manual effort the objective aims to reduce. Lines the rules matched can therefore still need a person, and the share of wrong matches is not determinable from code.
   - **Why it ranks above the next item:** it adds manual actions to lines the rules did match, and any automatic match may need checking because a wrong link looks like a right one. Item 4 adds no action per line and bites only on lines still open after the work of items 1 to 3.
4. **No structured exception tracking** — the gaps in Sections 4.2 and 4.3 leave each unmatched line with no accountable person.
   - **Evidence:** the statement-line table holds no owner, reason code, exception status or due date (`FIN_BANKSTATEMENTLINE`, Section 4.3). A reconciliation completes with lines still unmatched, and they carry into the next one unflagged (`FIN_ReconciliationProcess`, Section 4.2).
   - **Who is affected:** finance users who inherit carried-forward lines in the next reconciliation (`FIN_ReconciliationProcess`). Anyone answerable for the backlog is affected too, since no record names an owner and no active alert reaches a user or role (Section 4.3).
   - **Business impact:** "every unmatched item has an owner, age, and reason code" cannot be met, because no field exists to hold the owner or the reason and age is never stated (Section 6).
   - **Why it ranks above the next item:** it affects every line that stays open beyond a reconciliation and blocks a whole part of the objective that no current action can satisfy (`FIN_BANKSTATEMENTLINE`). Item 5 costs one prompt answer per opening of the screen and delays lines rather than leaving them untracked.
5. **Matching only on demand, behind a confirmation prompt** — nothing is matched between import and the next user session.
   - **Evidence:** the rules run only inside the opening step of `Match Statement` (`MatchStatementOnLoadActionHandler`), after the user accepts the prompt shown each time the screen opens (`APRM_AlgorithmConfirm`, `modules/org.openbravo.advpaymentmngt/web/org.openbravo.advpaymentmngt/js/ob-aprm-matchStatement.js`). Neither the import (`ImportBankFile`) nor statement processing (`FIN_BankStatementProcess`) starts them, and no background process does.
   - **Who is affected:** finance users, who must open the screen and answer the prompt for each financial account. Everyone waiting on that account's matched lines and completed reconciliation is affected too (`Match Statement`).
   - **Business impact:** waiting time is added to the cycle time the objective aims to cut, and an automatic step still needs a human touch to start.
   - **Why it ranks above the next item:** no sixth item is ranked, and this one outranks the candidates left below the five because it applies to every statement on every account (`MatchStatementOnLoadActionHandler`). Date-order warnings and force processing arise only for statements with a line dated on or before earlier processed lines (`APRM_BankStatementLineWrongDateWarning`), and partner-name exclusion words are upkeep per bank file format (`FIN_BANKFILE_EXCEPTION`). Posting configuration shapes accounting entries, not matching (`DocFINReconciliation`).

**Requirement implications:** the workbench must first cut per-line resolution and widen automatic matching (items 1 and 2), then make automatic matches checkable (item 3), give each open line the owner, age and reason code no field holds today (item 4), and start matching without waiting for a person (item 5).

## 6. Gaps Against the Objective

**What this section tells you:** how far today's code supports each element of the objective quoted in Section 1.1. Owner and reason code have no support, while the auto-match rate (the share of statement lines matched with no person acting) and age are only partly supported. Each row cites its evidence and points to the section that holds the detail.

Support levels: Full means the code delivers the element and records it; Partial means it delivers or records only part of it; None means nothing in the code supports it.

| Objective element | Current support (Full/Partial/None) | Evidence | Where detailed |
|-------------------|-------------------------------------|----------|----------------|
| Auto-match rate: most statement lines matched without human touch | Partial | The rules match lines automatically (`StandardMatchingAlgorithm`) but need exact values and start only when a person accepts the prompt (`APRM_AlgorithmConfirm`). Each line keeps its match level (`AU`, `AP`, manual `MA` or added `AD`) in `MATCHINGTYPE`, shown line by line as the Affinity column (`aprm_matchstatement`), yet no screen or report counts the share matched by the rules. The opening message gives one run's count and is not stored (`APRM_AutomaticMatchedLines`), and unmatching clears the level (`APRM_MatchingUtility`), so the actual rate is not determinable from code. | Sections 3.1, 3.3 and 5 (items 1, 2, 3 and 5) |
| Owner (the person accountable for resolving an unmatched item) | None | `FIN_BANKSTATEMENTLINE` has no owner column, and its `CREATEDBY` and `UPDATEDBY` record who entered and last changed a line, not who must resolve it. Access is granted per role and window (`AD_WINDOW_ACCESS`), never per line. | Sections 4.3 and 5 (item 4) |
| Age (how long an unmatched item has been open) | Partial | Age can be derived from the line's `DATETRX` or `CREATED` (`FIN_BANKSTATEMENTLINE`), and the Match Statement grid shows the line's date (`aprm_matchstatement`), but no field, list or report states the age. Lines carried into the next reconciliation carry no marker (`FIN_ReconciliationProcess`). | Sections 4.1, 4.2 and 4.3 |
| Reason code | None | No reason column exists on `FIN_BANKSTATEMENTLINE`, and `MATCHINGTYPE` records how a line was matched, not why one is not. The free-text `DESCRIPTION` and the general notes any record can carry (`OBUIAPP_NOTE`) offer no fixed list of reasons to count or report. | Section 4.3 |
| Cycle time (from statement import to completed reconciliation) | None | No field keeps when a line was matched or when a reconciliation completed: `FIN_RECONCILIATION` holds the period end (`DATETO`) and the statement date (`STATEMENTDATE`), and the `UPDATED` stamps on it and on each line change with any later edit. Actual cycle times are not determinable from code. | Sections 3.3 and 5 (item 5) |
| Manual effort | Partial | The automatic run, a search that links several transactions to one line, and unmatching a selection each save manual actions (`APRM_MatchingUtility`, `UnMatchSelectedTransactionsActionHandler`). Every line the rules miss still needs its own search or add, and only the current match levels `MA` and `AD` show manual work, so staff time is not determinable from code. | Sections 2 and 5 (items 1 and 3) |

**Requirement implications:** the workbench must add owner and reason code as new recorded data, turn the derivable age into a shown and tracked value, and measure the auto-match rate and cycle time that the code cannot report today.

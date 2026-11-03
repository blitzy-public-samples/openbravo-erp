# Analysis 1: Current-State Reconciliation Workflow and Manual Effort

## Key Takeaways

- A bank file reaches posted entries (`FACT_ACCT`) in twelve steps, most started by a person.
- People act at 40 points; the per-line ones sit in the `Match Statement` screen.
- The rules (`StandardMatchingAlgorithm`) need exact amounts, take the first match and wait for a user.
- No unmatched line has an owner, reason code or status, and age is only derivable (`FIN_BANKSTATEMENTLINE`).
- Per-line resolution is the top effort (`APRM_MatchingUtility`); match rates are not determinable from code.

## 1. End-to-End Flow from Statement Import to Posted Reconciliation

**What this section tells you:** How a bank file becomes a completed, posted reconciliation today, step by step. Every stage up to completion waits for a person, and a reconciliation can be completed while lines are still unmatched (Section 1.2, Figures 1 to 3).

### 1.1 How the Pieces Fit

The business objective is to "reduce manual effort and cycle time in bank reconciliation. Most statement lines are matched without human touch, and every unmatched item has an owner, age, and reason code." A reconciliation is the period document confirming the ERP (business-management software) balance agrees with the bank (`FIN_RECONCILIATION`), and cycle time runs from statement import to completed reconciliation. An unmatched item's owner is the person accountable for resolving it, its age is how long it has been open, and a reason code records why it is unresolved; Appendix A collects every defined term.

Apart from its data tables (stores of records) and accounting step (step 12), both in the core product, the flow lives in one module (a separately packaged part of the ERP), Advanced Payables and Receivables Management, `modules/org.openbravo.advpaymentmngt`. Everything hangs off the financial account (the ERP record for one bank account), kept in the `Financial Account` window (a data-entry screen): bank statements and their bank statement lines (one entry on the statement the bank sends) in `FIN_BANKSTATEMENT` and `FIN_BANKSTATEMENTLINE`, transactions (the ERP's record of money expected in or out of a financial account) in `FIN_FINACC_TRANSACTION`, and reconciliations in `FIN_RECONCILIATION`. So these records and lists cover one account at a time (`FIN_FINANCIAL_ACCOUNT_ID`; rule records can be shared, Section 3.2), which matters for any screen working exceptions (unmatched lines) across accounts.

Two account settings decide whether an account uses matching (linking a statement line to the transaction it settles). Only a bank-type account with a matching algorithm record (a saved set of matching rules that one or more financial accounts use) selected in `FIN_MATCHING_ALGORITHM_ID` shows the `Import Statement` and `Match Statement` buttons, and Section 1.2 follows these accounts. A cash account, or one with no record selected, is reconciled by hand instead (Section 1.3).

### 1.2 Step-by-Step Flow

The walkthrough has twelve steps in three stages: import (steps 1 to 4, Figure 1), matching (steps 5 to 9, Figure 2), and completion and posting (creating the accounting entries in the general ledger, the company's books; steps 10 to 12, Figure 3). Each step gives what happens, the window or class (a program unit) doing it, the tables and what the user sees; Sections 2 to 4 detail actions, matching and exceptions.

**Figure 1. Import and Statement Processing**

```mermaid
flowchart TD
    A["User clicks Import Statement (ImportBankFile)"] --> B{"Is a file format installed?"}
    B -->|No| C["Warning: install a format module (APRM_NoBankFileAvailableInfo)"]
    B -->|Yes| D["User picks format and file (FIN_BANKFILE_FORMAT)"]
    D --> E["File read into a new statement (FIN_BankStatementImport)"]
    E --> F{"Read without error?"}
    F -->|No| G["Error shown, nothing saved (APRM_InvalidOrMissingValues)"]
    F -->|Yes| H["Zero-amount lines dropped, partners guessed (FIN_BANKSTATEMENTLINE)"]
    H --> I["Statement processed at once (FIN_BankStatementProcess)"]
    I --> J{"Any line dated on or before the last processed date?"}
    J -->|Yes| K["Left unprocessed with a date warning (APRM_BankStatementLineWrongDateWarning)"]
    J -->|No| L["Ready for matching (FIN_BANKSTATEMENT)"]
```

**How to read this:** Follow the arrows down; diamonds are system checks, and each path ends in a message or a statement ready for matching.

1. **Start the import.**
   - **What happens:** A finance user clicks `Import Statement` (`EM_APRM_IMPORTBANKFILE`), and the system checks that a bank file format (one bank's file layout) is installed. `ImportBankFile` accepts only a file the user uploads, so how files arrive from the bank is not determinable from code.
   - **Done by:** the `Financial Account` window and `ImportBankFile`, an older hand-built screen.
   - **Tables:** reads `FIN_FINANCIAL_ACCOUNT` and `FIN_BANKFILE_FORMAT`.
   - **User sees:** a pop-up with a file chooser and format list or, with none installed, `APRM_NoBankFileAvailableInfo` asking an administrator to install a format module.
2. **Load the file into a new statement.**
   - **What happens:** On OK, the system creates the statement header, named with its number and file name, and the format's importer (the program that reads that layout) turns each entry into a bank statement line; any failure undoes the whole import (`ImportBankFile`).
   - **Done by:** `ImportBankFile`, running the format's importer, built on `FIN_BankStatementImport`.
   - **Tables:** reads `FIN_BANKFILE_FORMAT` and the statement document type (the setting that numbers a kind of document) in `C_DOCTYPE`; writes `FIN_BANKSTATEMENT` and `FIN_BANKSTATEMENTLINE`.
   - **User sees:** on failure, `APRM_InvalidOrMissingValues` (unreadable values), `APRM_DocumentTypeNotFound` (no numbering) or `APRM_WrongBankFileFormat` (importer not installed), with nothing kept.
   - **Unconfirmed:** which bank file formats an installation can import, because no code in this repository implements the importer, so formats come from separately installed modules (consulted `FIN_BankStatementImport`, `ImportBankFile`, `APRM_NoBankFileAvailableInfo`).
3. **Drop zero-amount lines and guess partners.**
   - **What happens:** Lines with zero credit and debit are discarded. Each other line takes its business partner (customer or supplier) and GL item (a general-ledger category for amounts with no invoice, such as bank fees) from the latest earlier line on the account with the same name, ignoring spaces and differences between digits; else from the partner with exactly that name; else, ignoring the words in `FIN_BANKFILE_EXCEPTION` and words of three letters or fewer, from the only partner sharing a name word. With several such partners, closest-name selection fails and the partner stays blank, so `Add Transaction` opens with no partner filled in (`AddTransactionFilterExpression`).
   - **Done by:** `FIN_BankStatementImport`.
   - **Tables:** reads earlier lines in `FIN_BANKSTATEMENTLINE`, partners in `C_BPARTNER` and excluded words in `FIN_BANKFILE_EXCEPTION`; writes partner and GL item on the new lines.
   - **User sees:** nothing; the final message counts only lines inserted (`RowsInserted`), and the prepared note on dropped zero-amount lines (`APRM_ZeroAmountNotInserted`) is never shown.
4. **Process the statement at once.**
   - **What happens:** The import processes (confirms) the statement at once so matching can use it. A line dated on or before the latest line of earlier processed statements (or, if none, the last reconciliation's end date) leaves it unprocessed (`FIN_BankStatementProcess`) until the user corrects the dates or force-processes it (Section 2).
   - **Done by:** `FIN_AddPayment`, which runs `FIN_BankStatementProcess`.
   - **Tables:** reads earlier lines in `FIN_BANKSTATEMENTLINE` and the last completed reconciliation in `FIN_RECONCILIATION`; writes the processed flag on `FIN_BANKSTATEMENT`.
   - **User sees:** the statement number and lines inserted (`APRM_BankStatementNo`), or the date warning with the cut-off date and affected lines (`APRM_BankStatementLineWrongDateWarning`, `APRM_BankStatementLineWrongDateWarning2`).

**Figure 2. Matching in the Match Statement Screen**

```mermaid
flowchart TD
    A["User clicks Match Statement (Match Statement)"] --> B{"Run the matching rules now? (APRM_AlgorithmConfirm)"}
    B -->|Yes or No| C["Draft reconciliation reused or created (FIN_RECONCILIATION)"]
    C -->|Yes, nothing unmatched| D["Message: nothing to match (APRM_NoStatementsToMatch)"]
    C -->|Yes| E["Unmatched lines matched, each saved at once (StandardMatchingAlgorithm)"]
    C -->|No| F["Grid of statement lines (Match Statement P&E)"]
    D --> F
    E --> F
    F --> G["User resolves a line: search, add or unmatch (APRM_MatchingUtility)"]
    G --> F
    F --> H{"Which button?"}
    H -->|OK| I["Screen closes, draft stays open (MatchStatementActionHandler)"]
    H -->|Reconcile All| J["On to completion and posting (MatchStatementActionHandler)"]
```

**How to read this:** The user's answer at the top decides whether automatic matching runs before the list of lines appears; the loop below is line-by-line work until a button is pressed.

5. **Open Match Statement and answer the prompt.**
   - **What happens:** The user clicks `Match Statement` (`EM_APRM_MATCHTRANSACTIONS`), opening a process definition (Pick and Execute: a pop-up grid, or on-screen table, where the user selects rows and runs an action). Before any data loads, a prompt (a yes-or-no question) asks, at every opening, whether to run the matching rules.
   - **Done by:** the `Match Statement` process definition and its browser code, `modules/org.openbravo.advpaymentmngt/web/org.openbravo.advpaymentmngt/js/ob-aprm-matchStatement.js`.
   - **Tables:** reads the user's saved preference (a saved system setting) in `AD_PREFERENCE`.
   - **User sees:** the prompt `APRM_AlgorithmConfirm`, warning that the run may create transactions, and a notice that every grid change is saved at once (`APRM_GRID_PERSIST_MESSAGE`) unless that preference hides it (`APRM_NoPersistInfoMessageInMatching`); with no Cancel or close button, only the screen's own buttons leave it.
6. **Prepare the draft reconciliation.**
   - **What happens:** Whatever the answer, the system reuses the account's draft (unfinished) reconciliation or creates one (`APRM_MatchingUtility`). A reused draft is re-dated, first repairing any transaction shown as cleared (linked) with no statement line, the trace of mixed manual and statement reconciliation (Section 4.4); a new draft is numbered from its document type, dated at the latest processed line and opened at the last reconciliation's balance.
   - **Done by:** `MatchStatementOnLoadActionHandler`, using `APRM_MatchingUtility`.
   - **Tables:** reads `FIN_BANKSTATEMENTLINE`, `C_DOCTYPE` and earlier reconciliations in `FIN_RECONCILIATION`; writes the draft in `FIN_RECONCILIATION` and any repaired transaction in `FIN_FINACC_TRANSACTION`.
   - **User sees:** nothing, unless `APRM_NoDocTypeRec` (no reconciliation numbering) or `APRM_ReconciliationMixed` (repair failed) stops it.
7. **Run automatic matching, if accepted.**
   - **What happens:** If no processed statement line is unmatched, the system only says so. Otherwise it takes each unmatched line in date order (for an earlier draft, only lines up to its end date, Section 1.3), asks the account's rules for one transaction and saves each match at once. A matched transaction becomes cleared (`RPPC`) and joins the draft; a line the rules cannot evaluate stays unmatched (Section 4.1).
   - **Done by:** `MatchStatementOnLoadActionHandler`, applying the rules the account's record names (shipped as `StandardMatchingAlgorithm`, through `FIN_MatchingAlgorithm`; Section 3.1).
   - **Tables:** reads `FIN_MATCHING_ALGORITHM`, `FIN_BANKSTATEMENTLINE` and candidate transactions (ones that could settle the line) in `FIN_FINACC_TRANSACTION`, with `FIN_PAYMENT` and `C_BPARTNER` details; writes the link and match level (the code recording how a line was matched, `MATCHINGTYPE`) on the line, and cleared status and draft link on `FIN_FINACC_TRANSACTION` and `FIN_PAYMENT`.
   - **User sees:** the match count (`APRM_AutomaticMatchedLines`), `APRM_NoAutomaticMatchedLines` or `APRM_NoStatementsToMatch`, then the grid.
8. **Review the grid.**
   - **What happens:** The grid lists the account's processed statement lines still unmatched, including those left from earlier statements, plus those matched in the current draft, in date order; lines cleared in earlier reconciliations do not appear (`MatchStatementTransformer`).
   - **Done by:** the `Match Statement P&E` window, an HQL-based table (a read-only list built from a database query, `aprm_matchstatement`) filtered by `MatchStatementTransformer`.
   - **Tables:** reads `FIN_BANKSTATEMENT`, `FIN_BANKSTATEMENTLINE`, `FIN_FINACC_TRANSACTION`, `C_BPARTNER` and `C_GLITEM`; writes nothing.
   - **User sees:** each line's date, partner, reference, amount and match strength (Section 3.1), and the linked transaction's date, partner, amount and GL item. Search and add work only while a row is unmatched, unmatch only once it is matched.
9. **Resolve the remaining lines.**
   - **What happens:** For each line still unmatched, the user searches for and links one or more transactions, splitting the line when amounts differ, or adds a missing transaction, created and linked at once; unmatch undoes a wrong link. Each row action first checks that no one changed the line since the grid loaded (`CheckRecordChangedActionHandler`). Actions are in Section 2, side effects in Section 4.2.
   - **Done by:** `FindTransactionsToMatchActionHandler`, `AddTransactionActionHandler` and the unmatch actions (`UnMatchSelectedTransactionsActionHandler`), all through `APRM_MatchingUtility`.
   - **Tables:** reads and writes `FIN_BANKSTATEMENTLINE` (a split adds a line for the remainder), `FIN_FINACC_TRANSACTION` and `FIN_PAYMENT`.
   - **User sees:** a pop-up grid of candidates or an entry form, then the refreshed grid, or `APRM_StaleDate` if the line changed meanwhile.

**Figure 3. Completing and Posting the Reconciliation**

```mermaid
flowchart TD
    A["User presses Reconcile All (MatchStatementActionHandler)"] --> B["Draft re-dated, balances recalculated (MatchTransactionDao)"]
    B --> C{"Every cleared item in an open period?"}
    C -->|No| D["Closed-period error, draft stays open (APRM_PeriodNotAvailableClearedItem)"]
    C -->|Yes| E["Completed; unmatched lines wait for the next one (FIN_ReconciliationProcess)"]
    E --> F{"Who starts posting?"}
    F -->|A user| G["User clicks Post (Posted)"]
    F -->|A schedule| H["Background accounting job runs (AcctServerProcess)"]
    G --> I{"Any cleared item needing entries? (DocFINReconciliation)"}
    H --> I
    I -->|No| J["No entries; button shows Document Disabled (DocFINReconciliation)"]
    I -->|Yes| K["Entries written to the general ledger (FACT_ACCT)"]
```

**How to read this:** From Reconcile All, the first diamond can refuse completion and the second shows posting started by a user or a schedule.

10. **Save and leave, or press Reconcile All.**
    - **What happens:** OK closes the screen, leaving the draft open with every match saved. `Reconcile All` re-dates the draft to the latest processed line (for an earlier draft, its latest cleared line), recalculates its balances and asks for completion.
    - **Done by:** `MatchStatementActionHandler`, using `APRM_MatchingUtility` and `MatchTransactionDao`.
    - **Tables:** reads `FIN_BANKSTATEMENTLINE` and `FIN_FINACC_TRANSACTION`; writes dates and balances on `FIN_RECONCILIATION`.
    - **User sees:** the `Financial Account` window, with a success message or the error that stopped completion.
11. **Complete the reconciliation.**
    - **What happens:** For organizations set up for accounting, each cleared transaction's accounting date must fall in an open accounting period (a span open for bookings); if one does not, nothing is completed (`FIN_ReconciliationProcess`). Otherwise this and any later reconciliation get recalculated balances, and it is marked complete even with lines still unmatched. Balances add every processed line, matched or not, so the document shows the bank's balance while unmatched lines wait for the next reconciliation (Section 4.2).
    - **Done by:** `FIN_ReconciliationProcess`, with balances from `MatchTransactionDao`.
    - **Tables:** reads the cleared items in `FIN_RECONCILIATIONLINE_V`, their transactions in `FIN_FINACC_TRANSACTION` and the open periods in `C_PERIODCONTROL`; writes status and balances on `FIN_RECONCILIATION`.
    - **User sees:** success, or `APRM_PeriodNotAvailableClearedItem` naming the item in a closed period; the completed reconciliation then shows in the `Reconciliations` tab, its items in `Cleared items`.
12. **Post the reconciliation.**
    - **What happens:** Posting books only cleared transactions for which the account's accounting setup, and for payments the payment method (a way of paying, such as a bank transfer), names a ledger account for clearing, so a reconciliation with none gets no entries; a closed period also refuses it (`DocFINReconciliation`).
    - **Done by:** the `Posted` button on the `Reconciliations` tab, shown once complete, or `Acct Server Process`, a background process (a job the system runs on a schedule without a user) run by `AcctServerProcess`; both use `DocFINReconciliation`.
    - **Tables:** reads `FIN_FINACC_TRANSACTION`, `FIN_FINACC_PAYMENTMETHOD` and `FIN_FINANCIAL_ACCOUNT_ACCT`; writes entries to `FACT_ACCT` and the posting status on `FIN_RECONCILIATION`.
    - **User sees:** the new entries in the General Ledger Journal report (`ReportGeneralLedgerJournal`), or a button caption giving the outcome, such as `Post: Document Disabled` or `Post: Period Closed`. Once posted, the button reads `Unpost`, which deletes the entries (`ResetAccounting`).
    - **Unconfirmed:** whether posting a reconciliation produces entries, because that depends on how each payment method books cleared items and on the configured accounts (consulted `DocFINReconciliation`, `DocFINBankStatement`).
    - **Unconfirmed:** whether and how often posting runs on a schedule, because core product data sets no schedule and only the QA (test) sample data runs it every five minutes, so each installation sets its own (consulted `AcctServerProcess`, `src-db/database/sourcedata/referencedData/AD_PROCESS_REQUEST.xml`, `referencedata/sampledata/QA_Testing/AD_PROCESS_REQUEST.xml`).

### 1.3 Accounts and Periods Outside the Main Path

- **Accounts reconciled without a bank file:** a cash account, or a bank account with no matching algorithm record, shows the `Reconcile` button (`EM_APRM_RECONCILE`) instead. It opens the older `Reconciliation` screen, where the user ticks transactions as cleared without linking statement lines, so a workbench tracking unmatched lines would miss these accounts (rows 29 and 30 in Section 2.1; mixing ban in Section 4.4).
- **Reopening an earlier period:** after an earlier reconciliation is force-reopened (Section 2), the account shows `Match Transactions Force` instead of `Match Statement`, working only on lines up to that reconciliation's end date (`MatchStatementTransformer`); completing it recalculates later balances (step 11).

**Requirement implications:** A workbench must start from processed statements' lines on each bank account, act on the draft reconciliation `Match Statement` uses, and expect completed reconciliations to leave lines unmatched; accounts reconciled without a bank file need their own treatment (Section 1.3).

## 2. Manual User Actions and Their Triggers

**What this section tells you:** Section 2.1 lists the 40 actions that need a person, with trigger, recurrence, volume sensitivity and automation potential (`modules/org.openbravo.advpaymentmngt`). Section 2.2 draws them as swimlanes (diagrams with one lane per actor) beside the system's steps. The actions repeated per statement line sit in `Match Statement` and set most of the effort.

### 2.1 Every Manual Action and What Drives It

Frequency driver is the event that brings an action back; Data volume sensitivity is High per line or transaction, Medium per statement, session or period, Low for rare corrections and None for one-time setup; real volumes are not determinable from code (Section 7). Could be automated is Yes when code already does or can skip the step, Partly when it does part, No when a person must judge.

**Assumption:** users holding the standard Finance role (a named set of screen and process permissions), shown as Finance, perform every row except 1 (System Administrator), 2, 4 and 5 (windows this role may only view in `AD_WINDOW_ACCESS`), and 9 and 10 (windows no standard role is granted). **Unconfirmed:** which people hold that role, because role assignments are installation data (consulted `modules/org.openbravo.v3.datasets/referencedata/standard/V3_Roles.xml`, `AD_USER_ROLES`).

| Step | Who acts | Trigger | Window / class | Frequency driver | Data volume sensitivity | Could be automated |
|------|----------|---------|----------------|------------------|-------------------------|--------------------|
| 1. Install a bank file format module and apply its dataset (packaged setup data) | System Administrator | Import warns that no format is installed (`APRM_NoBankFileAvailableInfo`) | `ImportBankFile` | One-time setup | None: one-time | No: the message assigns it to administrators |
| 2. Maintain words excluded from partner guessing | User with edit access | Partners guessed wrongly at import | `Bank File Format` (`FIN_BANKFILE_EXCEPTION`) | Setup, revisited after wrong guesses | Low: one list per format | Partly: the importer reuses earlier lines' partners (`FIN_BankStatementImport`); the list stays manual |
| 3. Choose the account's matching algorithm record | Finance | New bank account or different matching wanted; until chosen, import and matching buttons stay hidden (`FIN_MATCHING_ALGORITHM_ID`) | `Financial Account` | One-time setup | None: one-time | No: a business choice |
| 4. Edit a record's three on/off switches, changing every account that selects it | User with edit access | Matching too strict or too loose | `Matching Algorithm` (`FIN_MATCHING_ALGORITHM`) | Setup, revisited when results disappoint | Low: one record, many accounts | No: no match results are stored to tune from (Section 3.2) |
| 5. Create a record naming a different rule program | User with edit access | A module with other rules is installed (`FIN_MatchingAlgorithm`) | `Matching Algorithm` (`CLASSNAME`) | One-time setup | None: one-time | No: the rules must first be written |
| 6. Set, per payment method, whether payments create their transactions | Finance | Payments missing from matching candidates | `Payment Method` tab (`AUTOMATIC_DEPOSIT`, `AUTOMATIC_WITHDRAWN`) | One-time setup | None, but each method left off adds rows 18 and 24 per payment | No: the switch is what lets `FIN_PaymentProcess` automate this |
| 7. Set the account's GL item for differences | Finance | Manual reconciliations book differences (row 29) | `Financial Account` (`EM_APRM_GLITEM_DIFF`) | One-time setup | None: one-time | No: an accounting choice |
| 8. Set up posting: statement posting switch, cleared-item accounts, each method's clearing use | Finance | Posting creates no entries (Section 1.2) | `Accounting Configuration`, `Payment Method` (`ENABLEBANKSTATEMENT`, `INUPONCLEARINGUSE`) | One-time setup | None: one-time | No: accounting policy |
| 9. Schedule background posting | User granted the `Process Request` window | Posting should run without a person | `Acct Server Process` (`AD_PROCESS_REQUEST`) | One-time setup | None: one-time | No: the schedule is the automation (Section 1.2) |
| 10. Switch off the partial-match question (row 22) | User granted the `Preference` window | The question repeats at every split | `APRM_MATCHSTATEMENT_HIDE_PARTIALMATCH_POPUP`, a preference | One-time setup | None: one-time | No: a display choice |
| 11. Import the bank file | Finance | Statement received from the bank | `Financial Account` (`EM_APRM_IMPORTBANKFILE`), `ImportBankFile` | Per statement | Medium: one upload per file | Partly: reading and processing run alone after upload (`FIN_BankStatementImport`); no code fetches files |
| 12. Enter or correct a statement and its lines by hand | Finance | No file, or file rejected (`APRM_InvalidOrMissingValues`) | `Imported Bank Statements`, `Bank Statement Lines` | Per statement without a usable file | High: per line | Partly: a format module reads files instead (row 1) |
| 13. Process a hand-entered statement | Finance | Statement complete | `EM_APRM_PROCESS_BS`, `FIN_BankStatementProcess` | Per hand-entered statement | Medium: per statement | Yes: import already processes what it creates (`FIN_BankStatementImport`) |
| 14. Force-process a statement despite earlier dates | Finance | Date warning (`APRM_BankStatementLineWrongDateWarning`) | `EM_APRM_PROCESS_BS_FORCE`, `FIN_BankStatementProcess` | Per warned statement | Medium: per statement | Partly: a fixed date rule only a person overrides (Section 4.4) |
| 15. Open Match Statement | Finance | Statement processed; shown for bank accounts with a record and no older draft | `Financial Account` (`EM_APRM_MATCHTRANSACTIONS`), `Match Statement` | Per opening of Match Statement | Medium: per session | Partly: draft handling and matching are programs only this screen starts (`MatchStatementOnLoadActionHandler`) |
| 16. Accept or decline the automatic matching run | Finance | Prompt at every opening (`APRM_AlgorithmConfirm`) | `Match Statement` | Per opening of Match Statement | Medium: per session | Yes: the prompt only gates a program needing no input (Section 3.3) |
| 17. Dismiss the immediate-save notice for good | Finance | Notice at every opening until dismissed (`APRM_GRID_PERSIST_MESSAGE`) | `MatchStatementOnLoadPreferenceActionHandler` (`APRM_NoPersistInfoMessageInMatching`) | Once per user | None: once per person | No: a personal choice |
| 18. Add several payments to the account as transactions | Finance | Payments lack transactions (row 6) | `Add Multiple Payments`, `AddMultiplePaymentsHandler` | Per batch of payments | Medium: one run, many payments | Partly: unneeded where row 6 applies (`FIN_PaymentProcess`) |
| 19. Enter and process a transaction by hand | Finance | An amount, such as a fee, must exist before matching | `Transaction` tab (`EM_APRM_PROCESSED`), `FIN_TransactionProcess` | Per transaction | High: per amount | Partly: as row 23 |
| 20. Set or correct a processed line's GL item or partner | Finance | Line unmatched for want of a GL item (Section 3.1) | `Bank Statement Lines` (`C_GLITEM_ID`, `C_BPARTNER_ID`, still editable under `APRM_FIN_BNKSTM_LINE_CHECK_TRG`) | Per unmatched line | High: per line | Partly: later imports copy both from lines with the same name (`FIN_BankStatementImport`) |
| 21. Search for and link transactions, splitting the line when amounts differ | Finance | Line left unmatched | `Find Transactions to Match`, `FindTransactionsToMatchActionHandler`, `APRM_MatchingUtility` | Per unmatched line | High: per line | Partly: code lists candidates but ranks none (Section 3.3) |
| 22. Answer the partial-match question (whether to split the line) | Finance | Chosen amount differs from the line (`APRM_SplitBankStatementLineConfirm`) | `Find Transactions to Match`, `Add Transaction` | Per split | High: per split line | Yes: row 10 removes it |
| 23. Add and link a bank-fee or GL-item transaction | Finance | No transaction exists for the line | `Add Transaction`, `AddTransactionActionHandler` | Per unmatched line | High: per line | Partly: plugged-in rules may create them (`CREATEDBYALGORITHM`); standard rules create none (`StandardMatchingAlgorithm`) |
| 24. Link an existing payment through Add Transaction | Finance | Payment has no transaction | `Add Transaction`, `AddTransactionActionHandler` | Per unmatched line | High: per line | Partly: unneeded where row 6 applies (`FIN_PaymentProcess`) |
| 25. Create a new payment from Add Transaction | Finance | The money has no payment in the ERP | `Add Payment` (`OBUISEL_AddNewRecord`) | Per unmatched line | High: per line | Partly: only plugged-in rules could create payments (`CREATEDBYALGORITHM`, `FIN_PAYMENT`) |
| 26. Unmatch a wrong match, one row or a selection | Finance | Link judged wrong; only the Affinity column (the grid's match-strength code) flags looser matches | `Match Statement P&E`, `UnMatchTransactionActionHandler`, `UnMatchSelectedTransactionsActionHandler` | Per wrong match | High: per matched line | No: nothing in code marks a link as wrong (Section 3.3) |
| 27. Reconcile with Reconcile All | Finance | Matching finished for the period | `Match Statement`, `MatchStatementActionHandler` | Per period | Medium: per period | Partly: completion is one program call (Section 1.2); only the decision is manual |
| 28. Leave the draft open with OK | Finance | Work to continue later | `MatchStatementActionHandler` | Per opening of Match Statement | Low: per session | Yes: OK changes nothing, as grid changes are already saved (`APRM_GRID_PERSIST_MESSAGE`) |
| 29. Reconcile without a statement: enter date, ending balance and any difference for row 7's GL item; save or process | Finance | Cash account, or account with no record, the only cases the button shows | `Financial Account` (`EM_APRM_RECONCILE`), `Reconciliation` | Per period | Medium: per period | No: no statement lines exist to compare, and an unbooked difference is refused (`APRM_ReconciliationDiscrepancy`) |
| 30. Tick each cleared transaction | Finance | Manual reconciliation open | `Reconciliation` | Per transaction, each tick saved at once | High: per transaction | No: as row 29 |
| 31. Reactivate (reopen for editing) a statement or the latest reconciliation | Finance | Error found after completion; refusals in Section 4.4 | `FIN_BankStatementProcess`, `FIN_ReconciliationProcess` | Per correction | Low: rare | No: a person decides what to correct |
| 32. Unpost a reconciliation or statement | Finance | Reactivation refused while posted (`PostedDocument`) | `Posted` button, `ResetAccounting` | Per correction after posting | Low: rare | No: a person decides to undo the books |
| 33. Force-reactivate a statement with a matched line | Finance | Reactivation refused (`APRM_BSLineReconciled`) | `EM_APRM_PROCESS_BS_FORCE` | Per correction | Low: rare | No: as row 31 |
| 34. Force-reactivate an earlier reconciliation that later ones follow | Finance | Correction in a processed, non-latest, non-manual reconciliation, the only case the button shows | `Reconciliation Process Force` | Per correction | Low: rare | No: as row 31 |
| 35. Re-match that earlier draft | Finance | The open draft is not the latest, which hides row 15's button | `Match Transactions Force` (`EM_APRM_MATCHTRANS_FORCE`) | Per forced correction | Low: rare | No: follows row 34 |
| 36. Delete a wrong, unprocessed statement | Finance | Duplicate or wrong file, after row 31; processed statements cannot be deleted (`APRM_FIN_BANK_STMT_CHECK_TRG`) | `Imported Bank Statements` | Per wrong import | Low: rare | No: a person judges the file wrong |
| 37. Delete a draft or later reconciliation | Finance | A later reconciliation blocks a correction (`APRM_FutureReconciliationExists`) | `Reconciliations` (`APRM_FIN_RECON_CHECK_TRG`) | Per correction | Low: rare | No: as row 31 |
| 38. Print the reconciliation details or summary | Finance | Printout needed | `ReportReconciliationDetail`, `ReportReconciliationSummary` | Per reconciliation | Low: per printout | Partly: code builds the report (`ReportReconciliation`); only a button starts it |
| 39. Post the reconciliation | Finance | Completed reconciliation not yet posted | `Posted` button, `DocFINReconciliation` | Per reconciliation | Medium: per period | Yes: `Acct Server Process` posts it once scheduled (row 9), unless switched off (`ISBACKGROUNDDISABLED`) |
| 40. Post the bank statement | Finance | Processed statement, with statement posting on (row 8) | `Posted` button, `DocFINBankStatement` | Per statement | Medium: per statement | Yes: as row 39 |

Effort comes from the recurring rows: per statement or session, rows 11, 15 and 16 (`ImportBankFile`, `APRM_AlgorithmConfirm`); per line the rules never match, rows 20 to 25 (`FindTransactionsToMatchActionHandler`, `AddTransactionActionHandler`); per wrong match, row 26; per transaction ticked, row 30; and per period, rows 27 and 29 (`MatchStatementActionHandler`). The rest are setup, corrections, printing and posting; a schedule can take over posting (row 9), and Appendix B lists buttons left out.

### 2.2 Swimlanes of Manual Touchpoints

Figure 4 shows setup, Figure 5 import, Figure 6 matching and Figure 7 corrections, each manual touchpoint a bold box naming its Section 2.1 rows once. Every arrow back into the user lane, such as an unmatched line (`Match Statement`) or a refusal (`PostedDocument`), is manual effort.

**Figure 4. Swimlane of Setup Actions**

```mermaid
flowchart LR
  subgraph People["Finance user or administrator"]
    M1["Rows 1-2: install a format module, keep exclusion words (Bank File Format)"]
    M3["Rows 3-5: choose, tune or replace the rules (Matching Algorithm)"]
    M6["Row 6: decide which payments create transactions (Payment Method)"]
    M7["Row 7: set the difference GL item (GL Item Difference)"]
    M8["Rows 8-9: set up and schedule posting (Accounting Configuration, Process Request)"]
    M10["Row 10: switch off the partial-match question (Preference)"]
  end
  subgraph System["System"]
    S1["Reads files, guesses partners (FIN_BankStatementImport)"]
    S2["Shows matching buttons, applies the rules (StandardMatchingAlgorithm)"]
    S3["Creates a transaction per payment (FIN_PaymentProcess)"]
    S4["Books the difference (Reconciliation)"]
    S5["Posts on request or schedule (Acct Server Process)"]
    S6["Splits without asking (Find Transactions to Match, Add Transaction)"]
  end
  M1 --> S1
  M3 --> S2
  M6 --> S3
  M7 --> S4
  M8 --> S5
  M10 --> S6
  classDef manual stroke-width:3px
  class M1,M3,M6,M7,M8,M10 manual
```

**How to read this:** Bold boxes are manual setup steps, and each arrow points to the behaviour it enables.

**Figure 5. Swimlane of Import and Opening**

```mermaid
flowchart TB
  subgraph People["Finance user"]
    M11["Row 11: import the file (ImportBankFile)"]
    M12["Rows 12-13: enter, fix and process a statement (Imported Bank Statements)"]
    M14["Row 14: force-process (Bank Statement Process Force)"]
    M15["Rows 15-16: open Match Statement, answer the prompt (APRM_AlgorithmConfirm)"]
    M17["Row 17: dismiss the save notice (APRM_GRID_PERSIST_MESSAGE)"]
    M18["Rows 18-19: add payments or transactions first (Add Multiple Payments)"]
    M20["Row 20: set a line's GL item or partner (Bank Statement Lines)"]
  end
  subgraph System["System"]
    S1["Reads the file, creates lines (FIN_BankStatementImport)"]
    S2["Checks dates, processes (FIN_BankStatementProcess)"]
    S3["Warns, leaves unprocessed (APRM_BankStatementLineWrongDateWarning)"]
    S4["Prepares the draft, runs the rules if accepted (MatchStatementOnLoadActionHandler)"]
    S5["Shows the lines (Match Statement)"]
  end
  M11 --> S1
  S1 -->|"rejected"| M12
  S1 --> S2
  M12 --> S2
  S2 -->|"earlier dates"| S3
  S3 -->|"fix dates"| M12
  S3 -->|"override"| M14
  S2 -->|"processed"| M15
  M14 --> M15
  M15 -->|"notice"| M17
  M15 --> S4
  M18 -->|"candidates"| S4
  S4 --> S5
  S5 -->|"no GL item"| M20
  M20 -->|"next opening"| M15
  classDef manual stroke-width:3px
  class M11,M12,M14,M15,M17,M18,M20 manual
```

**How to read this:** Bold boxes are manual steps, and arrow labels name the condition that hands work back.

**Figure 6. Swimlane of Matching and Closing**

```mermaid
flowchart LR
  subgraph System["System"]
    S1["Line left unmatched (Match Statement)"]
    S2["Link saved at once, split if amounts differ (APRM_MatchingUtility)"]
    S3["Link removed, parts merged (APRM_MatchingUtility)"]
    S4["Periods checked, completed (FIN_ReconciliationProcess)"]
    S5["Difference, dates, period checked (Reconciliation)"]
  end
  subgraph People["Finance user"]
    M21["Row 21: search and link (Find Transactions to Match)"]
    M22["Row 22: answer the partial-match question (APRM_SplitBankStatementLineConfirm)"]
    M23["Rows 23-25: add a transaction or payment (Add Transaction, Add Payment)"]
    M26["Row 26: unmatch (UnMatchSelectedTransactionsActionHandler)"]
    M27["Rows 27-28: Reconcile All, or OK to leave the draft (MatchStatementActionHandler)"]
    M29["Rows 29-30: reconcile by ticking items (Reconciliation)"]
  end
  S1 --> M21
  S1 --> M23
  M21 -->|"amounts differ"| M22
  M23 -->|"amounts differ"| M22
  M22 --> S2
  M21 -->|"equal"| S2
  M23 -->|"equal"| S2
  S2 -->|"wrong"| M26
  M26 --> S3
  S3 --> S1
  S2 -->|"period done"| M27
  M27 -->|"Reconcile All"| S4
  M29 --> S5
  classDef manual stroke-width:3px
  class M21,M22,M23,M26,M27,M29 manual
```

**How to read this:** Bold boxes are manual steps, and the unmatch loop returns a wrong link to unmatched.

**Figure 7. Swimlane of Corrections, Printing and Posting**

```mermaid
flowchart LR
  subgraph People["Finance user"]
    M31["Row 31: reactivate a statement or latest reconciliation (EM_APRM_PROCESS_BS, EM_APRM_PROCESS_REC)"]
    M32["Row 32: unpost first (ResetAccounting)"]
    M33["Rows 33-34: force-reactivate (Bank Statement Process Force, Reconciliation Process Force)"]
    M35["Row 35: re-match the earlier draft (Match Transactions Force)"]
    M36["Rows 36-37: delete a wrong statement or reconciliation (Imported Bank Statements, Reconciliations)"]
    M38["Row 38: print details or summary (Reconciliations)"]
    M39["Rows 39-40: post (Posted)"]
  end
  subgraph System["System"]
    S1["Refuses while posted (PostedDocument)"]
    S2["Refuses: matched line or later one (APRM_BSLineReconciled, APRM_FutureReconciliationExists)"]
    S3["Reopens the document (FIN_BankStatementProcess, FIN_ReconciliationProcess)"]
    S4["Creates entries (DocFINReconciliation, DocFINBankStatement)"]
    S5["Prints (ReportReconciliationDetail, ReportReconciliationSummary)"]
  end
  M31 -->|"posted"| S1
  S1 --> M32
  M32 --> M31
  M31 -->|"matched or later"| S2
  S2 -->|"force"| M33
  S2 -->|"later one"| M36
  M31 -->|"allowed"| S3
  M33 --> S3
  S3 -->|"earlier one reopened"| M35
  S3 -->|"wrong statement"| M36
  M38 --> S5
  M39 --> S4
  classDef manual stroke-width:3px
  class M31,M32,M33,M35,M36,M38,M39 manual
```

**How to read this:** Bold boxes are manual steps, and arrows from a refusal lead to the correction it forces.

**Requirement implications:** A workbench cuts the most effort by removing the line-level actions (rows 20 to 26) and the per-session prompt (row 16) and by using the switches that already automate steps (rows 6, 9 and 10), not by adding setup or correction screens (`Match Statement`, `FIN_PaymentProcess`).

## 3. Existing Matching Logic

**What this section tells you:** Automatic rules link a statement line to a transaction only when the amounts agree exactly, and the first candidate wins; other details count only where a pass compares them, and a GL-item line accepts an earlier-dated transaction. Fixed gates and a few mostly shared settings decide what is compared, each with its default, how an administrator changes it and a failure example (Sections 3.1 and 3.2).

### 3.1 Criteria and Algorithm

The standard rules (`StandardMatchingAlgorithm`) try these passes (rounds of checks) in order. A strong match is a candidate found by the algorithm's primary rules, and a weak match is a candidate found only by its looser fallback rules (`FIN_MatchedTransaction`). Figure 8 shows the passes for one line.

| Pass | Applies when | What must agree | Result |
|------|--------------|-----------------|--------|
| 1 | Line carries a GL item | Same GL item, exact amount, transaction dated on or before the line | Strong (`AU`) |
| 2 | Pass 1 found nothing | Same GL item and exact amount, any date | Weak (`AP`) |
| 3 | Every line not matched by pass 1 or 2 | Exact amount, plus each switched-on criterion below: date, reference unless the line's is empty or a double asterisk, and partner name; a difference in a switched-off or skipped criterion cannot block a strong match | Strong (`AU`) |
| 4 | Pass 3 found nothing | Exact amount, plus exact date if the date switch is on | Weak (`AP`) |
| 5 | Nothing found | — | No match (`NO`) |

**Figure 8. Matching Decision Flow for One Bank Line**

```mermaid
flowchart TD
    accTitle: Figure 8. Matching Decision Flow for One Bank Line
    accDescr: Each line runs the passes in order until one finds a candidate, which sets the match strength.
    L["Next eligible line, oldest first (APRM_MatchingUtility)"] --> G{"Line carries a GL item?"}
    G -->|Yes| P1{"Pass 1: same GL item, exact amount, dated on or before the line?"}
    G -->|No| P3{"Pass 3: exact amount and every switched-on criterion, skipping an empty or double-asterisk reference?"}
    P1 -->|Yes| S["Strong match (AU)"]
    P1 -->|No| P2{"Pass 2: same GL item, exact amount, any date?"}
    P2 -->|Yes| W["Weak match (AP)"]
    P2 -->|No| P3
    P3 -->|Yes| S
    P3 -->|No| P4{"Pass 4: exact amount, plus same date if the switch is on?"}
    P4 -->|Yes| W
    P4 -->|No| N["No match, left for a person (NO)"]
    S --> K["Linked if saved; either way counted and its transaction no longer offered (MatchStatementOnLoadActionHandler)"]
    W --> K
```

**How to read this:** Follow one line down the Yes and No answers; the box where its path ends is how the automatic run leaves it.

These fixed gates, which no setting changes, decide which lines are tried and which transactions they may take (`APRM_MatchingUtility`, `MatchTransactionDao`).

- Lines: unmatched lines of the account's processed statements (Section 4.1), and, when the draft is not the latest reconciliation (Section 1.3), only those dated on or before its end date (`APRM_MatchingUtility`, `MatchTransactionDao`); for example, with March reopened after April's completion, a 2 April line is not tried.
- Line order: oldest first, then line number and partner name (`APRM_MatchingUtility`), so earlier lines get first pick of shared candidates.
- Candidates: the line's account only (`MatchTransactionDao`); for example, a payment into the company's other bank account is never offered.
- Candidates: processed only; for example, a fee entered but not processed waits until processed (`FIN_TransactionProcess`).
- Candidates: not already cleared, checked as no reconciliation and status not `RPPC`, which every link or tick sets together (`APRM_MatchingUtility`, `Reconciliation`); for example, a deposit ticked in last month's reconciliation is not offered again.
- Candidates: not taken earlier in the run (`MatchStatementOnLoadActionHandler`); for example, of two 100.00 lines and one 100.00 payment, the older line wins it even when it belongs to the newer.

| Criterion | Default behaviour | How an admin changes it | What defeats it, for example |
|-----------|-------------------|-------------------------|------------------------------|
| Same GL item (passes 1 and 2) | Only lines with a GL item (`C_GLITEM_ID`), set by the file format, by hand or at import (Section 1.2) | No setting | A line for an unrecorded fee falls to passes 3 and 4, which ignore the GL item, and can take an unrelated same-amount payment (`StandardMatchingAlgorithm`) |
| Dated on or before the line (pass 1) | Always, whatever the date switch says (`StandardMatchingAlgorithm`) | No setting | A fee recorded a day after the bank charged it fails pass 1 and is only a weak match in pass 2 (`MatchTransactionDao`) |
| Exact amount (every pass) | Money in minus money out must equal the transaction's deposit minus payment, to the cent (`MatchTransactionDao`) | No setting; only another rule set could relax it | A 1,000.00 receipt booked as 997.50 after a bank charge finds no candidate and waits for a person |
| Exact date (passes 3 and 4) | Compared only with the date switch (`MATCHTRANSACTIONDATE`) on; both passes then need the same day | Date switch (Section 3.2) | With the switch on, passes 3 and 4 miss a payment entered on the 30th that the bank books on the 1st |
| Reference (pass 3) | Only with the reference switch on and a line reference (`REFERENCENO`, mandatory) neither empty nor a double asterisk, which must equal the payment's reference or ERP-assigned document number (`FIN_PAYMENT`) | Reference switch (Section 3.2) | With the switch on, a customer quoting the invoice number, which is never compared, misses pass 3 and gets at best a weak match; a transaction with no payment, such as a bank fee, never passes it |
| Partner name (pass 3) | With the name switch on, the name the bank sent (`BPARTNERNAME`) must equal the payment partner's name (`C_BPARTNER`); the import's guessed partner (`C_BPARTNER_ID`) is ignored | Name switch (Section 3.2) | "ACME LTD" from the bank against partner "Acme Limited" fails pass 3; a line with no partner name never passes it against a partner's payment |

One rule set ships, others can follow the interface (the contract every rule set follows, `FIN_MatchingAlgorithm`), and manual search is a separate path.

| Rule set or path | Default behaviour | How an admin changes it | Limit or failure scenario, for example |
|------------------|-------------------|-------------------------|----------------------------------------|
| Standard rules (`StandardMatchingAlgorithm`) | The only shipped program following the interface; it links existing transactions, never creating one, though the prompt warns it may (`APRM_AlgorithmConfirm`), and undoing its link deletes only algorithm-created records (Section 4.2) | Select a record naming it, then set its switches (Section 3.2) | A rule error counts as no match (Section 4.1), but a link that fails to save still counts as matched and its transaction is no longer offered, though the line stays unmatched; a run failing part-way keeps saved links and reports a normal count (`APRM_MatchingUtility`, `MatchStatementOnLoadActionHandler`) |
| Another rule set | None ships; the interface asks for one transaction and match level per line and an undo step | Install a module with one and enter its name as the record's program (`CLASSNAME`) | The `Matching Algorithm` window then hides the three switches, and a line covering several transactions stays manual |
| Manual search (`Find Transactions to Match`) | Lists the account's processed transactions not yet in a reconciliation, by default only those dated on or before the line, with no amount, name or reference filter (`TransactionsToMatchTransformer`) | No setting; the date filter is part of the screen's definition (`AD_TAB`) | A payment entered a day after the bank date stays hidden until the user removes the filter (Section 4.2) |

### 3.2 Configurability

- Each financial account can select one matching algorithm record (`FIN_MATCHING_ALGORITHM_ID`), kept in the `Matching Algorithm` window (`FIN_MATCHING_ALGORITHM`) with the rules program and three switches (table below). Choosing another record changes one account, but editing a record silently shifts matching for every account selecting it; in the F&B International Group sample, two bank accounts share one (`referencedata/sampledata/F_B_International_Group/FIN_FINANCIAL_ACCOUNT.xml`).
- Another rule set can plug in through the interface (`FIN_MatchingAlgorithm`) without changing the flow (Section 3.1).
- New records, from the window or a dataset such as the module's own `Standard` record (which sets no switch), take the application's switch defaults (`AD_COLUMN`, `DataImportService`, `modules/org.openbravo.advpaymentmngt/referencedata/standard/Standard_Algorithm.xml`); the database's apply only to rows inserted directly (`FIN_MATCHING_ALGORITHM`), and reloading a dataset keeps an existing record's switches. The F&B sample `Standard` record (`referencedata/sampledata/F_B_International_Group/FIN_MATCHING_ALGORITHM.xml`) turns all three off, so pass 3 makes any same-amount candidate a strong match and pass 4 can find nothing pass 3 missed.
- **Unconfirmed:** which record each account selects and that record's switches, because both are configuration data that can be edited at any time (consulted `FIN_FINANCIAL_ACCOUNT`, `FIN_MATCHING_ALGORITHM`, `StandardMatchingAlgorithm`).

Every setting read by matching and its screens (`StandardMatchingAlgorithm`, `MatchStatementOnLoadActionHandler`):

| Parameter | Default behaviour | How an admin changes it | Limit or failure scenario, for example |
|-----------|-------------------|-------------------------|----------------------------------------|
| Account's matching algorithm record (`FIN_MATCHING_ALGORITHM_ID`) | Empty: optional, no default, shown only for bank accounts (Section 1.3) | `Matching Algorithm` field of the `Financial Account` window | A new bank account saved without one never offers automatic matching, so its statements are reconciled by ticking |
| Program name (`CLASSNAME`) | Mandatory, with no default; every shipped record names the standard rules (`StandardMatchingAlgorithm`) | `Java Class Name` field in the `Matching Algorithm` window | A mistyped name, or one from a module not installed, stops automatic matching and every unmatch for every sharing account (`NoAlgorithmFoundException`); a valid new name also changes how earlier automatic links are undone, since unmatching always asks the account's current record (`APRM_MatchingUtility`) |
| Name switch (`MATCHBPNAME`) | On by application and database default | `Match BP Name` check box | Off, two customers' same-amount payments can be swapped as strong matches; on, see Section 3.1 |
| Reference switch (`MATCHREFERENCE`) | On by application and database default | `Match Reference` check box | Off, an exact reference no longer beats another payment of the same amount; on, see Section 3.1 |
| Date switch (`MATCHTRANSACTIONDATE`) | On by application default, off by database default | `Match Transaction Date` check box | Off, a payment dated months away from the line can be linked; on, see Section 3.1 |
| Automatic deposit or withdrawal on the account's payment method (`AUTOMATIC_DEPOSIT`, `AUTOMATIC_WITHDRAWN`) | Off: processing a payment creates no transaction unless the user picks the action that also deposits or withdraws (`FIN_PaymentProcess`, `FIN_Utility`) | `Automatic Deposit` and `Automatic Withdrawn` on the `Payment Method` tab | A customer receipt processed without a deposit leaves its bank line no candidate until the deposit is recorded |
| Partial-match question (`APRM_MATCHSTATEMENT_HIDE_PARTIALMATCH_POPUP`) | Asked when the chosen transactions do not total the line (`APRM_SplitBankStatementLineConfirm`), unless this preference is Y | Y in the `Preference` window, for one window or all (Section 2.1) | Y skips the question, not the split: a 90.00 payment picked for a 100.00 line leaves a 10.00 remainder line unannounced |
| Grid-save notice (`APRM_NoPersistInfoMessageInMatching`) | Shown at every opening (`APRM_GRID_PERSIST_MESSAGE`) | Each user can hide it with the never-show link (`OBUIAPP_NeverShowMessageAgain`), a personal preference read by `MatchStatementOnLoadGetPreferenceActionHandler`; deleting it in the `Preference` window restores it | Once hidden, nothing on screen reminds the user that a wrong link is saved immediately |

### 3.3 Limits

- No amount or date tolerance (allowed difference) in `MatchTransactionDao` (exact-amount and exact-date criteria, Section 3.1).
- With the name switch (`MATCHBPNAME`) on, pass 3 needs identical partner names (partner-name criterion, Section 3.1).
- The first candidate wins (`StandardMatchingAlgorithm`), with no scoring between same-amount candidates, so a plausible but wrong link can be made. No setting changes this, and the searches set no order, so the winner is not determinable from code (`MatchTransactionDao`); for example, two open payments of 100.00 can each be linked to the other's line.
- Automatic matching links one line to one transaction; linking it to several needs a manual search or add that splits it (`APRM_MatchingUtility`). For example, one deposit covering three customer cheques is never matched automatically unless one transaction holds the total (`StandardMatchingAlgorithm`).
- Automatic matching runs only when a user opens `Match Statement` and accepts the prompt (`APRM_AlgorithmConfirm`), which no setting removes, so lines wait for a person; for example, a user declining it to reach the grid sooner may hand-resolve lines the rules would have matched. Nothing signals a waiting statement either (Section 4.3).
- Weak matches are applied unconfirmed, with only the Affinity column as a cue, so wrong links surface only on review (`MatchStatementOnLoadActionHandler`); the closing message counts matched lines without separating weak ones (`APRM_AutomaticMatchedLines`).
- Every grid change and every automatic link is saved at once (`APRM_GRID_PERSIST_MESSAGE`, `MatchStatementOnLoadActionHandler`), so neither a session's changes nor a run's wrong links can be undone in one step.

**Requirement implications:** A workbench must show the shared record and switches behind each automatic link and let people confirm weak matches, which are saved unconfirmed (Sections 3.2 and 3.3).

## 4. Current Handling of Unmatched and Exception Items

**What this section tells you:** This section traces each exception type from detection to resolution (Section 4.4, Appendix C). A person clears each by hand; none records an owner, age or reason code (Section 4.3).

### 4.1 What Counts as Unmatched

An unmatched item is a processed statement's line with no linked transaction (`FIN_FINACC_TRANSACTION_ID` empty, `MatchTransactionDao`); a matching error counts as no match, so the line looks rejected (`MatchStatementOnLoadActionHandler`). Lines of unprocessed statements, such as one held by the date check (Section 4.4), do not count: matching, the grid and the printout read only processed ones (`APRM_MatchingUtility`, `aprm_matchstatement`). Unmatched lines show in the Match Statement grid (Section 1.2, step 8), the `Bank Statement Lines` tab (one statement's lines, no unmatched-only filter) and the reconciliation printout's unreconciled-lines part (`ReportReconciliation`), a snapshot of uncleared lines up to the end date, not a worklist (`modules/org.openbravo.advpaymentmngt/src/org/openbravo/advpaymentmngt/ad_reports/UnreconciledBankStatement.jrxml`). None spans accounts (`FIN_FINANCIAL_ACCOUNT_ID`) or gives age, owner or cause, so no one sees the whole backlog.

### 4.2 How Exceptions Are Resolved

Apart from a later automatic run, each resolution starts from one row of the Match Statement grid (`Match Statement P&E`).

- Search and link, splitting when amounts differ: the line keeps the matched amount and a new unmatched remainder with the same line number and date carries the rest (`APRM_MatchingUtility`); splitting is blocked once the reconciliation is posted (`APRM_SplitBSLReconciliationPosted`).
- Add a transaction, linked with match level `AD` (manually added); unless the user types a description, a fee or GL item transaction reads only "Bank Fee" or "GL Item" and the item's name (`APRM_BankFee`, `APRM_GLItem`).
- Unmatch, which merges the line with its still-unmatched split parts and, under the standard rules, deletes an algorithm-created transaction, and its payment if also algorithm-created, once reactivated, with accounting entries if posted (`APRM_MatchingUtility`, `StandardMatchingAlgorithm`). Because search and add work only on unmatched rows (Section 1.2), correcting a link always takes an unmatch first (`modules/org.openbravo.advpaymentmngt/web/org.openbravo.advpaymentmngt/js/ob-aprm-matchStatement.js`).
- Carry forward: a reconciliation completes with unmatched lines left, and they reappear in the next reconciliation's pending list (`FIN_ReconciliationProcess`) with nothing marking them as old. Reconcile gives no warning that lines remain (`MatchStatementActionHandler`), and each later automatic run retries them (`APRM_MatchingUtility`).

### 4.3 What Is Not Recorded

- `FIN_BANKSTATEMENTLINE` has no owner, reason code, exception status or due date (gap levels in Section 6); age can only be derived from the line's transaction date (`DATETRX`) or entry date (`CREATED`). Its only general-purpose note field, `DESCRIPTION`, holds free text, which cannot be counted by cause and so cannot stand in for a reason code.
- `CREATEDBY` and `UPDATEDBY` record who entered and last changed a line, so after resolution they show who acted, never who was responsible while it was open (`FIN_BANKSTATEMENTLINE`). Record notes (free-text comments attachable to any record, `OBUIAPP_NOTE`) are the only other place for a cause, and the reconciliation module never reads them (`modules/org.openbravo.advpaymentmngt`).
- No active alert (a system-generated to-do message sent to a user or role) watches statement lines; the one shipped statement rule, "Old not processed Bank Statements", is inactive and reads the older `C_BANKSTATEMENT` table. It flags unprocessed statements over ten days old, so alerts can already express an age limit (`src-db/database/sourcedata/referencedData/AD_ALERTRULE.xml`).
- Checks stop work rather than queue it: each in Section 4.4 and Appendix C shows at most one message to whoever acted and keeps no visible record, so blocked time cannot be measured within cycle time (`FIN_BankStatementProcess`, `FIN_ReconciliationProcess`).

### 4.4 Exception Types from Detection to Resolution

No type records an owner (Section 4.3): "Finance, assumed" names Section 2's assumed operator; actual responsibility is not determinable from code (Section 7).

| Exception type | Detected and seen | Resolved today | Owner today | Age and reason captured; gap |
|----------------|-------------------|----------------|-------------|------------------------------|
| Line left unmatched | No candidate, a declined run, or a rule error (Sections 3.1, 4.1) | Search and link, add, or carry forward (Section 4.2) | Finance, assumed | Age derivable, reason free text only (Section 4.3); not "matched without human touch" |
| Wrong automatic match (`AP`, `AU`) | Only on review (Sections 3.3, 5) | Unmatch, then search or add; failures listed once by reference (`APRM_ErrorOnUnmatchingRecords`) | Finance, assumed; no reviewer recorded | Neither: unmatching clears `MATCHINGTYPE`, so causes cannot be counted or fed back into the rules |
| Amount difference resolved by split | Only by a person's search, amounts being exact (Section 3.3, `FindTransactionsToMatchActionHandler`) | Split and merge (Section 4.2) | Finance, assumed; remainder unowned | Remainder keeps the original `DATETRX`; fee, partial payment or error is never said |
| Missing transaction: fee, GL item or payment | Search finds none; the standard rules never create one (`StandardMatchingAlgorithm`) | Add Transaction, level `AD` (Section 4.2) | Finance, assumed | Age derivable (`DATETRX`); after resolution, type and GL item (`C_GLITEM`) say what, not why |
| Carried-forward line | Reconcile completes with it unmatched (Section 4.2, `FIN_ReconciliationProcess`) | As a line left unmatched, later or never | Finance, assumed | Age derivable (`DATETRX`) but unmarked, no reason; with no due date (Section 4.3), nothing bounds cycle time |
| Line changed by someone else | `APRM_StaleDate`: search and add check only before their pop-up opens (`CheckRecordChangedActionHandler`), unmatch on submission (`UnMatchTransactionActionHandler`), so a change during an open pop-up goes uncaught | Refresh and retry | Finance, assumed; lines unassigned | Neither: colleagues learn of each other by refusal, if at all |
| Cleared item in a closed accounting period | Reconcile and reactivation stop, naming the item, if its accounting date (`DATEACCT`) is in a period not open (`C_PERIODCONTROL`, `APRM_PeriodNotAvailableClearedItem`) for organizations with accounting (`FIN_ReconciliationProcess`); manual checks differ (Appendix C) | Draft: unmatch the item or reopen the period (Open/Close Period Control window); completed: reactivate only after reopening | Not determinable from code: no standard role holds the period window | Neither: the unstored block hides the wait in cycle time |
| Statement dates out of order | Statement left unprocessed, failing lines listed (Section 1.2, `APRM_BankStatementLineWrongDateWarning`) | Correct the dates and reprocess, or force-process (Section 2) | Finance, assumed | Age derivable (`IMPORTDATE`), no reason; lines are in no unmatched list (Section 4.1), and the one age rule is inactive (Section 4.3) |
| Manual and statement reconciliation mixed | Match Statement clears a draft's hand ticks, telling only the server log (a technical record users do not see), or fails (`APRM_ReconciliationMixed`, `APRM_MatchingUtility`); manual reconciliation refuses a draft with statement lines (`Reconciliation`) | Finish the draft in one mode | Finance, assumed | Neither: hand work is reversed unannounced |
| Split after posting | Blocked once the reconciliation is posted (Section 4.2) | Link an exact amount, or unpost first (`ResetAccounting`) | Finance, assumed | Neither: the line waits unexplained |
| Reactivation limits | Processed statements, lines and reconciliations refuse key-field edits, insertion and deletion (`20501`); reopening is refused if posted, if a statement has a matched line (`APRM_BSLineReconciled`), or if a later reconciliation (`APRM_FutureReconciliationExists`) or another draft exists (`APRM_DraftReconciliationExists`) | Reopen first: force-reactivation skips only the matched-line or later-reconciliation check; else unpost, finish the draft or delete later reconciliations (Section 2) | Finance, assumed | Neither: correcting a completed period is a chain of unassigned steps |

Figure 9 traces one line to reconciliation or unmarked carry-forward.

**Figure 9. Lifecycle of an Unmatched Statement Line**

```mermaid
flowchart TD
    accTitle: Figure 9. Lifecycle of an Unmatched Statement Line
    accDescr: A line moves from processing to reconciliation or carry-forward.
    A["Statement processed (FIN_BankStatementProcess)"] --> B{"Matching accepted and a candidate found? (StandardMatchingAlgorithm)"}
    B -->|"Yes, strong or weak, and saved"| L["Line linked (FIN_FINACC_TRANSACTION)"]
    B -->|"No, declined, rules failed or save failed"| U["Line unmatched, no owner or reason (FIN_BANKSTATEMENTLINE)"]
    U --> G["Waits in one account's grid (MatchStatementTransformer)"]
    G -->|"Search finds one"| S["Search and link, split if amounts differ (FindTransactionsToMatchActionHandler)"]
    G -->|"None exists"| D["Add a fee, GL item or payment (AddTransactionActionHandler)"]
    S --> L
    S -->|"Split remainder"| U
    D --> L
    L --> W{"Wrong link spotted?"}
    W -->|"Yes"| X["Unmatch (UnMatchTransactionActionHandler)"]
    X --> U
    W -->|"No"| R["Period reconciled (FIN_ReconciliationProcess)"]
    G -->|"Still unmatched"| R
    R -->|"Linked lines"| C["Line reconciled"]
    R -->|"Unmatched lines"| F["Carried forward, unflagged"]
    F --> G
```

**How to read this:** Boxes are steps, diamonds decisions; each arrow back into "Line unmatched" is another manual pass.

**Requirement implications:** A workbench must give every exception type in Section 4.4 an assignee, a visible age and a coded reason across all accounts, and turn each blocked action into an owned item rather than a message shown once.

## 5. Five Largest Sources of Manual Effort

**What this section tells you:** This section ranks the five largest sources of manual effort by how many statement lines each can reach (Sections 2 to 4). It separates the work of getting lines matched (items 1, 2 and 5) from correcting and tracking them (items 3 and 4), so readers can test the order against installation data (Section 7).

**Assumption:** the ranking reflects how many statement lines the code lets each source affect, not measured volumes. **Unconfirmed:** actual match rates, line volumes and cycle times, because the repository stores no metrics and they depend on production data (consulted `MatchStatementOnLoadActionHandler`, which only shows the matched count as a message).

1. **Line-by-line resolution of lines the rules never match** — each line no later automatic run matches needs a person, one line at a time (Section 4.2).
   - **Evidence:** each search, add or split request carries a single statement line (`FindTransactionsToMatchActionHandler`, `AddTransactionActionHandler`, `APRM_MatchingUtility`). Only unmatching works on a selection (`UnMatchSelectedTransactionsActionHandler`), so the search and add actions of Section 2.1 have no bulk form.
   - **Who is affected:** finance users working the `Match Statement P&E` grid (Section 2), and readers of the reconciliation printout, which lists every line left over (`ReportReconciliation`).
   - **Business impact:** it blocks "matched without human touch", since each such line takes at least one manual action to resolve (Section 6).
   - **Why it ranks above the next item:** it reaches every line no automatic run matches, whatever the cause, including a bank fee with no transaction to find (`AddTransactionActionHandler`); item 2's exact-value rules are only one cause feeding it.
2. **Exact-equality rules with no tolerance, where the first candidate wins** — any field that fails the applicable pass's test (Section 3.1) sends the line to weak matching or manual work.
   - **Evidence:** Sections 3.1 to 3.3 give the criteria, switches and first-candidate choice (`MatchTransactionDao`, `StandardMatchingAlgorithm`, `FIN_MATCHING_ALGORITHM`).
   - **Who is affected:** finance users, who work the lines the rules reject, and users permitted to edit `Matching Algorithm` (Sections 2.1 and 3.2).
   - **Business impact:** they cap how many lines are "matched without human touch" (Section 3.1); lines they keep rejecting feed item 1, and wrong first picks feed item 3.
   - **Why it ranks above the next item:** they decide every line in every automatic run (`StandardMatchingAlgorithm`), whereas wrong matches arise only where a same-amount candidate was found.
3. **Correcting wrong automatic matches** — each wrong link (Section 3.3) must be found and undone by a person (Section 4.2).
   - **Evidence:** Sections 3.3 and 4.4 give the review and correction evidence (`FIN_MatchedTransaction`, `UnMatchSelectedTransactionsActionHandler`).
   - **Who is affected:** finance users, who must detect and correct wrong links (Sections 3.3 and 4.2), and accounting readers, because a wrong link marks the wrong transaction cleared (`RPPC`, `APRM_MatchingUtility`).
   - **Business impact:** it adds to the manual effort the objective aims to reduce, because each wrong match costs a review and an unmatch, after which the line waits for a search, an add or a later automatic run (Section 4.2). The share of wrong matches is not determinable from code (Section 7).
   - **Why it ranks above the next item:** it adds actions to lines the rules did match, and any automatic match may need checking because a wrong link looks right (`FIN_MatchedTransaction`); item 4 adds no action per line and bites only on lines still open after items 1 to 3.
4. **No structured exception tracking** — the gaps in Sections 4.2 and 4.3 leave each unmatched line with no accountable person.
   - **Evidence:** Sections 4.2 and 4.3 give the missing tracking fields and the unflagged carry-forward (`FIN_BANKSTATEMENTLINE`, `FIN_ReconciliationProcess`).
   - **Who is affected:** finance users who inherit carried-forward lines, and anyone answerable for the backlog, which no record or active alert assigns (Section 4.3).
   - **Business impact:** "every unmatched item has an owner, age, and reason code" cannot be met (Section 6).
   - **Why it ranks above the next item:** it affects every line still open after a reconciliation and blocks a part of the objective no current action can satisfy (`FIN_BANKSTATEMENTLINE`); item 5 costs one prompt answer per opening and delays lines rather than leaving them untracked.
5. **Matching only on demand, behind a confirmation prompt** — nothing is matched between import and the next user session (Section 3.3).
   - **Evidence:** Section 3.3 gives the matching-start limit (`MatchStatementOnLoadActionHandler`, `APRM_AlgorithmConfirm`); neither import nor processing starts matching (Section 1.2, `ImportBankFile`, `FIN_BankStatementProcess`).
   - **Who is affected:** finance users working each bank account that uses matching, and people waiting for its reconciliation (`Match Statement`).
   - **Business impact:** waiting time is added to the cycle time the objective aims to cut, and an automatic step still needs a human touch to start (Section 6).
   - **Why it ranks above the next item:** it outranks the candidates left unranked because it applies to every processed statement on each bank account that uses matching (Section 1.1, `MatchStatementOnLoadActionHandler`). Date-order warnings and force processing arise only for statements dated on or before earlier lines (`APRM_BankStatementLineWrongDateWarning`), exclusion words are upkeep per file format (`FIN_BANKFILE_EXCEPTION`), and posting configuration shapes accounting entries, not matching (`DocFINReconciliation`).

**Requirement implications:** Requirements must cover both preventing manual work and accountable resolution of the exceptions that remain, and delivery priorities should be checked against workload and outcome data before this ranking is treated as a delivery sequence (Section 7).

## 6. Gaps Against the Objective

**What this section tells you:** This section assesses how far today's code supports each element of the objective quoted in Section 1.1. Owner and reason code have no support, while the auto-match rate (the share of statement lines matched with no person acting) and age are partly supported.

Support levels: Full, the code delivers and records the element; Partial, only part; None, nothing supports it.

| Objective element | Current support (Full/Partial/None) | Evidence | Where detailed |
|-------------------|-------------------------------------|----------|----------------|
| Auto-match rate: most statement lines matched without human touch | Partial | Automatic matching exists (`StandardMatchingAlgorithm`), within the limits of Sections 3.1 and 3.3. Each line keeps its match level (`AU`, `AP`, manual `MA` or added `AD`) in `MATCHINGTYPE`, but no screen or report counts the share and the run's count is not stored (`APRM_AutomaticMatchedLines`). The actual rate is not determinable from code. | Sections 3.1, 3.3 and 5 (items 1, 2, 3 and 5) |
| Owner | None | No owner field (Section 4.3, `FIN_BANKSTATEMENTLINE`); role grants (`AD_WINDOW_ACCESS`) never assign individual lines. | Sections 4.3, 4.4 and 5 (item 4) |
| Age | Partial | Derivable from the line's dates, which the grid shows, but never shown as an age (Sections 4.3 and 4.4, `FIN_BANKSTATEMENTLINE`, `aprm_matchstatement`). | Sections 4.1 to 4.4 |
| Reason code | None | No reason field, and free text cannot be counted (Section 4.3, `FIN_BANKSTATEMENTLINE`, `OBUIAPP_NOTE`); `MATCHINGTYPE` records how a line was matched, not why one is not. | Sections 4.3 and 4.4 |
| Cycle time | None | No field keeps when a line was matched or a reconciliation completed: `FIN_RECONCILIATION` holds the period end (`DATETO`) and statement date (`STATEMENTDATE`), and its `UPDATED` stamp changes with any edit. Actual cycle times are not determinable from code. | Sections 3.3 and 5 (item 5) |
| Manual effort | Partial | Sections 2.1, 4.2 and 5 (items 1 and 3) give the actions the code already spares users and the per-line work left on lines no automatic run matches, since later runs retry open lines (`APRM_MatchingUtility`, `UnMatchSelectedTransactionsActionHandler`); `MA` and `AD` identify manual matching methods, not time spent, so staff time is not determinable from code. | Sections 2 and 5 (items 1 and 3) |

**Requirement implications:** The workbench must record owner and reason code as new data, show and track the derivable age (Section 4.3), and measure the auto-match rate and cycle time the code cannot report today (Section 7).

## 7. Open Questions

**What this section tells you:** Code alone cannot establish the workload, operating results or installation settings needed to judge the objective (Section 1.1). These questions identify the operational evidence required before setting targets or validating the ranking and gaps in Sections 5 and 6.

| Question | Why the code cannot answer it | Code consulted | Section it affects |
|----------|-------------------------------|----------------|--------------------|
| How many statement lines arrive per account and period, and so how much each ranked source weighs | Not determinable from code: volumes are production data, and the ranking rests on code paths (Unconfirmed and Assumption, Section 5) | `FIN_BANKSTATEMENTLINE`, `MatchStatementOnLoadActionHandler` | Sections 2 and 5 |
| What share of lines is matched without human touch, and what share of those matches is wrong | Not determinable from code: no report counts either, run counts are not stored and unmatching clears the match level (Unconfirmed, Section 5) | `APRM_AutomaticMatchedLines`, `MATCHINGTYPE`, `APRM_MatchingUtility` | Sections 5 and 6 |
| How long lines take from import to completed reconciliation | Not determinable from code: no match or completion time is stored (Unconfirmed, Section 5) | `FIN_RECONCILIATION` | Section 6 |
| What "most" lines and the cycle-time target mean in numbers | Not determinable from code: the objective sets no figures and no setting holds a target | `FIN_MATCHING_ALGORITHM` | Section 1.1 |
| How much staff time each manual action takes | Not determinable from code: no action is timed | `MATCHINGTYPE` | Sections 2 and 6 |
| Which bank file formats are installed | Not determinable from code: formats are separate modules (Unconfirmed, Section 1.2) | `FIN_BankStatementImport` | Section 1.2 |
| How files reach users, including bank-side delivery and connectivity | Not determinable from code: import accepts only an uploaded file | `ImportBankFile` | Section 1.2 |
| Whether posting produces entries, and whether and how often it is scheduled | Not determinable from code: both depend on each installation's setup (two Unconfirmed findings, Section 1.2) | `DocFINReconciliation`, `AcctServerProcess` | Section 1.2 |
| Which record each account selects, with its switches | Not determinable from code: configuration data (Unconfirmed, Section 3.2) | `FIN_MATCHING_ALGORITHM_ID`, `FIN_MATCHING_ALGORITHM` | Section 3.2 |
| Which of several same-amount candidates wins | Not determinable from code: the searches set no order | `MatchTransactionDao` | Section 3.3 |
| Who holds the Finance role, who may reopen a closed period, and who answers for each open item or blocked check | Not determinable from code: role assignments are installation data (Unconfirmed and Assumption, Section 2), no standard role holds the period window, and nothing records an owner | `AD_USER_ROLES`, `C_PERIODCONTROL` | Sections 2 and 4.4 |

**Requirement implications:** Targets and measures for the workbench must come from a live installation, and the workbench should record the match, age and timing data the code cannot report today (Section 6).

## Appendix A. Glossary

Each technical term as defined at its first use.

| Term | Plain definition |
|------|------------------|
| Accounting period | a span open for bookings |
| Affinity column | the grid's match-strength code |
| Age | how long an unmatched item has been open |
| Alert | a system-generated to-do message sent to a user or role |
| Auto-match rate | the share of statement lines matched with no person acting |
| Background process | a job the system runs on a schedule without a user |
| Bank file format | one bank's file layout |
| Bank statement line | one entry on the statement the bank sends |
| Business partner | a customer or supplier |
| Candidate | a transaction that could settle the line |
| Class | a program unit |
| Cleared | linked, said of a transaction |
| Cycle time | the time from statement import to completed reconciliation |
| Data table | a store of records |
| Dataset | packaged setup data |
| Document type | the setting that numbers a kind of document |
| Draft | an unfinished reconciliation |
| ERP | business-management software |
| Exception | an unmatched line |
| Financial account | the ERP record for one bank account |
| General ledger | the company's books |
| GL item | a general-ledger category for amounts with no invoice, such as bank fees |
| Grid | an on-screen table |
| HQL-based table | a read-only list built from a database query |
| Importer | the program that reads a bank file format |
| Interface | the contract every rule set follows |
| Match level | the code recording how a line was matched |
| Matching | linking a statement line to the transaction it settles |
| Matching algorithm record | a saved set of matching rules that one or more financial accounts use |
| Module | a separately packaged part of the ERP |
| Owner | the person accountable for resolving an unmatched item |
| Pass | a round of checks |
| Payment method | a way of paying, such as a bank transfer |
| Posting | creating the accounting entries in the general ledger |
| Preference | a saved system setting |
| Process (a statement) | confirm it |
| Process definition (Pick and Execute) | a pop-up grid where the user selects rows and runs an action |
| Prompt | a yes-or-no question |
| QA | test |
| Reactivate | reopen for editing |
| Reason code | a record of why an unmatched item is unresolved |
| Reconciliation | the period document confirming the ERP balance agrees with the bank |
| Record notes | free-text comments attachable to any record |
| Role | a named set of screen and process permissions |
| Server log | a technical record users do not see |
| Strong match | a candidate found by the algorithm's primary rules |
| Swimlane | a diagram with one lane per actor |
| Tolerance | an allowed difference |
| Transaction | the ERP's record of money expected in or out of a financial account |
| Unpost | delete the accounting entries of a posted document |
| Weak match | a candidate found only by its looser fallback rules |
| Window | a data-entry screen |

## Appendix B. Buttons and Checks Without a Row

Supports Section 2.1: flow buttons and checks with no row of their own, and why.

- `Funds Transfer` (`EM_APRM_FUNDS_TRANS`) moves money between accounts, creating candidates rather than resolving lines, and transaction posting from the `Transaction` tab and the older `C_BANKSTATEMENT` window lie outside this flow.
- Hidden columns `EM_APRM_ADDTRANSACTIONPD` and `EM_APRM_FINDTRANSACTIONSPD` only hold the process definitions of the search and add actions (Section 2.1, rows 21 and 23 to 25), and `EM_APRM_PROCESS_RECONCILIATION` is never shown.
- The stale-line check (`CheckRecordChangedActionHandler`) has no row because it only refuses an action on a line someone else changed; Section 4.4 traces it.

## Appendix C. Further Checks That Interrupt Exception Work

Supports Section 4.4, in its columns (`modules/org.openbravo.advpaymentmngt/src-db/database/sourcedata/AD_MESSAGE.xml`).

| Check | Detected and seen | Resolved today | Owner today | Age and reason captured; gap |
|-------|-------------------|----------------|-------------|------------------------------|
| Matched line edited or deleted | Deletion, or a change to date, amounts, reference, partner name text (`BPARTNERNAME`), line number, statement, organization or active flag while matched, is refused (`APRM_BSTLine_Matched`, `APRM_FIN_BNKSTM_LINE_CHECK_TRG`); description, GL item and partner link stay editable. Processed statements refuse these first (`20501`), so it surfaces after force-reactivation. Splitting it is refused too (`APRM_SplitBSLAlreadyMatched`, `APRM_MatchingUtility`), a safeguard the screens never reach | Unmatch, then edit or split | Finance, assumed | Neither: why the line needed correcting is lost |
| Manual reconciliation out of balance | Refused while the difference, less any GL-item difference entered, is not zero (`APRM_ReconciliationDiscrepancy`, `Reconciliation`); any manual refusal closes the screen with the message on the tab, keeping the draft's date and balance | Tick or untick, correct the ending balance, or enter the difference, booked to the account's difference GL item (`EM_APRM_GLITEM_DIFF`) | Finance, assumed | Neither: the booking never says why |
| Manual reconciliation date out of range | Dates before the last completed reconciliation's end date, or after tomorrow, are refused; tomorrow passes (`APRM_ReconcileInFutureOrPast`) | Enter a date in range | Finance, assumed | Neither: the block is unrecorded |
| Manual reconciliation in a closed period | Only if posting is configured for a ticked transaction and the organization has accounting: the statement date (`PeriodNotAvailable`), then each ticked transaction's date (`STATEMENTDATE`, `APRM_PeriodNotAvailableClearedItem`), must be in an open period | Pick another date, untick the item, or reopen the period | Not determinable from code (Section 4.4) | Neither: the draft waits |
| Setup missing | With no reconciliation numbering, Match Statement (`APRM_NoDocTypeRec`, `APRM_NoDocNumberDocType`) and manual reconciliation (`APRM_DocumentTypeNotFound`) create no draft; import saves nothing without a format (`APRM_NoBankFileAvailableInfo`) or after a Section 1.2 failure; program name: Section 3.2 | Set numbering in `Document Type`; install a format, fix the program name, document type or file, or hand-enter (Section 2) | Not determinable from code: Company Admin holds `Document Type`, a permission only | Neither: nothing exists to assign or age |
| Search or add refused | The pop-up stays open, saving nothing, if Find has nothing ticked (`APRM_SELECT_RECORD_ERROR`), Add lacks a payment or GL item (`APRM_INVALID_TRANSACTION`), or the new transaction's period is closed while posting is configured (`PeriodNotAvailable`) or its payment awaits automatic execution (`APRM_AutomaticExecutionProcess`, `FIN_TransactionProcess`) | Complete the entry, reopen the period, or execute the payment first (`Payment In`, `Payment Out`) | Finance, assumed | Neither: the line stays unmatched, unexplained |
| Unmatch clean-up refused | Deleting an algorithm-created transaction on unmatch (Section 4.2) fails if its payment method has left the account (`APRM_NoPaymentMethod`, `FIN_TransactionProcess`); the transaction stays, reported only to the server log (`APRM_MatchingUtility`) | Add the method back, then reopen and delete it | Finance, assumed | Neither: no one learns a leftover remains |

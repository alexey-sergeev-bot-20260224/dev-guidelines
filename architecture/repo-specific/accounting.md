# Accounting Repo — Review Notes

## Journal Entry Line Metadata Propagation

When new JournalEntryLine objects are created — especially synthetic/correction lines
(rounding discrepancy, exchange gain/loss) — verify that product/clazz/description
fields are set appropriately.

### Why it matters

These fields are consumed downstream by:
- **Register/OpenSearch indexing** (`RegisterEntryMappingRepository`) — product, clazz,
  description appear in search results and UI
- **Summary reports & CSV export** (`JpaSummarizedJournalRecordRepository`,
  `CsvRawDataReportService`) — product/clazz used for grouping; description in raw data
- **Dimension flows** (`JournalEntryLine.getFieldValue()`) — PRODUCT, PRODUCT_NAME,
  DESCRIPTION fields drive dimension-based automation

### Rules

- Lines derived from a transaction line (income, discount, tax, asset) → copy
  productReference, clazzReference, description from source line
- Synthetic lines (rounding, exchange gain/loss) → at minimum set description
  (e.g. "Exchange rate rounding discrepancy"). Product/clazz may be null.
- Null is valid (adjustments/tips/shipping often omit product/clazz), but be
  intentional — understand that null means the amount won't appear in
  per-product/per-class reports

## Accounting Equation in Code

The balancing logic in `LedgerUtil.getBalancingAccountMoney()` implements:

```
Assets + Expenses = Liabilities + Equity + Income
```

- Same side of equation → opposite sign
- Different sides → same sign

When reviewing methods that create JE lines, verify the sign logic matches
this rule. Key method: `createOffsettingLineFrom()` (or equivalent).

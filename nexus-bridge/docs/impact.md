# Impact

## The problem today

A support agent who gets a question about a merchant's payout has to look the merchant up in Passport, MXM, CPX and ACH one after another. Each system has its own ID for the merchant and its own version of the name. The agent works out which records belong together by eye, usually from the business name. That is slow, and it goes wrong exactly where it matters most, with near-duplicates like "Acme Retail LLC" and "Acme Retail L.L.C.", which may or may not be the same business.

Finance runs into the same problem during reconciliation. Risk has no reliable record of why two records were ever treated as one merchant.

## What changes with Nexus Bridge

**For Support.** One search with any legacy ID gives one canonical record: linked IDs, sub-accounts, payment instruments, balances by currency, and a single ledger across all four systems. An agent no longer has to open four systems to answer one question.

**For Finance.** Every transaction carries a source and a destination canonical ID. Reconciliation can group activity by merchant without joining four data sets by hand. Balances in different currencies stay separate.

**For Risk and Compliance.** Nothing is merged on a hunch. If an identifier is missing, conflicts, or matches more than one entity, the transaction is blocked and a person decides. Every decision is logged with who made it and why.

**For Engineering.** Legacy systems don't change. Stamping only adds fields, so existing consumers keep working, and it can be switched off with a flag. New services get one stable key to build on.

## What we are and aren't claiming

The prototype shows the behaviour end to end on fictional data. We haven't measured production time savings yet, and the benchmark tiles in the prototype are labelled as simulated. The plan is to measure during a shadow-mode pilot:

- time for Support to answer a merchant identity question, before and after
- share of transactions that resolve with exactly one mapping
- number of records sent to review each week, and how long they wait
- number of wrong links found in audit (the target is zero)

These are the numbers we'd bring back to decide whether to move on to live stamping.

---
layout: post
title: "Measuring Salesforce Code Performance, Part 1: The Query Plan Tool"
date: 2026-06-26
tags: [Salesforce, Performance]
excerpt: "Most developers never touch the performance tools sitting right inside their org — until a query that worked fine in sandbox hits a timeout in production."
---

Most Salesforce developers never touch the performance tools sitting right inside their org — not because the tools are hidden, but because nobody needs them until data volume turns a fast query into a query timeout. By then, the fix is reactive instead of planned.

This is the first post in a series on tools that let you measure and fix code performance *before* it breaks in production — including cases where the code was written by AI. Part 1 covers the most foundational of these: the **Query Plan Tool**.

## What the Query Plan Tool Does

In large enterprise orgs, data volumes climb into the millions of records. A SOQL query that runs instantly against a sandbox with a thousand rows can hit a query timeout against a million-row production table.

The Query Plan Tool tells you, upfront, how a query will actually be executed — whether the optimizer will use an index, fall back to a full table scan, or rely on sharing rules — so you can fix a bad query plan before it ships, not after it fails.

## How to Enable It

1. Open the **Developer Console**.
2. Go to **Help → Preferences**.
3. Enable **"Enable Query Plan."**

Once enabled, a **Query Plan** tab appears next to the Query Editor.

## Key Terms You Need to Know

| Term | What It Means |
|---|---|
| **Cardinality** | Estimated number of records the leading operation will return |
| **Fields** | The indexed field(s) the optimizer chose to use, if any |
| **Leading Operation Type** | How the optimizer plans to execute the query — via an index, sharing rules, a table scan, or other internal optimizations |
| **Cost** | A score representing query efficiency (more below) |
| **sObject Cardinality** | Total approximate record count for the queried object |
| **sObject Type** | The primary object the query runs against |

Salesforce continuously runs statistics on your objects so the optimizer can calculate these costs using real, current data distribution — not guesswork.

## How Query Cost Is Actually Calculated

Here's the part most developers skip past without understanding — and it's the part that matters most.

Imagine a custom object, `Support_Ticket__c`, with 1 million records. You run:

```sql
SELECT Id, Subject__c FROM Support_Ticket__c
WHERE Priority__c = 'High' AND Status__c = 'Closed'
```

The optimizer doesn't just run the query — it checks current data distribution and scores multiple possible execution paths.

**Plan A: Index on `Priority__c`**
600,000 of the 1 million tickets are marked "High." A custom index is only considered selective if it returns under 10% of the table. 600,000 records is 60% of the data — six times over the threshold. Cost: **6.0** (60% ÷ 10%). Verdict: non-selective, unsafe to use.

**Plan B: Index on `Status__c`**
Only 30,000 of 1 million tickets are "Closed" — 3% of the table, well under the 10% threshold. Cost: **0.3** (3% ÷ 10%). Verdict: highly selective — this is the plan the optimizer should pick.

| Leading Operation Type | Fields | Cost | Verdict |
|---|---|---|---|
| Index | Status__c | 0.3 | **Winner** — highly selective, jumps straight to 30K records |
| TableScan | *null* | 1.0 | Fallback baseline — scans all 1M records |
| Index | Priority__c | 6.0 | **Loser** — six times over the selectivity threshold |

**The rule of thumb: anything with a Cost under 1.0 is selective and safe. Anything at or above 1.0 means you're not getting real index benefit.**

If you want the platform-level detail on how selectivity thresholds are derived, Salesforce publishes a [query optimization cheatsheet](https://resources.docs.salesforce.com/194/0/en-us/sfdc/pdf/salesforce_query_search_optimization_developer_cheatsheet.pdf) (search for it by title if the link has moved).

## Using It in Practice

1. Enable the tool in Developer Console (steps above).
2. Run your actual query in the Query Editor — don't bother substituting in literal record IDs for `WHERE` clauses; ID fields are indexed by default, so they're not your risk area. Focus your attention on **non-ID fields** in `WHERE` clauses.
3. Read the output: sObject cardinality, leading operation type, and cost.
4. Target a cost below 1.0. If you're above it, look for a more selective field to filter or index on.

**A few field-level gotchas worth knowing up front:** `CreatedDate`, `SystemModstamp`, and `LastModifiedDate` are *not* indexed by default. If your query relies heavily on date filtering, plan for that early rather than discovering it after a timeout in production.

## The Goal

The point of this tool isn't to memorize terminology — it's to get every query you ship below a cost of 1.0, so it scales with your org's data instead of breaking when data volume grows.

---

*Next in this series: Perspective Manager, the Apex Log Analyzer VS Code extension, and Scale Center — three more tools for diagnosing performance once a query plan alone isn't enough to tell you what's slow.*

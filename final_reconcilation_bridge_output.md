# Reconciliation Bridge

| Step | Description | Result | Reason |
|---|---|---|---|
| 0 | Naive count of all `communication_log` rows | 30 | Starting point — the "obvious" query someone would run without knowing how campaigns/retries work |
| 1 | Exclude ineligible campaigns | 26 | Campaign 9004 is `approval_awaiting`. Sends already went out under it, but per the data dictionary a campaign only counts toward reporting once both `creation_status` is finalized and `processing_status = 'processed'` — 9004's creation workflow never cleared, so its 4 rows (C11–C14) are dropped |
| 2 | Collapse retry chains to distinct customers per family | 22 | Campaigns 9001→9002→9003 and 9201→9202 are retry chains (linked via `parent_id`) representing the same underlying communication re-attempted. Customers retried within a chain (C2, C3, D1) are counted once per chain, not once per attempt. Standalone campaign 9101 (no parent, nothing points at it) is left uncollapsed — a repeated customer there (C20) is two legitimate separate sends, not a retry |

Final: target_base = 22
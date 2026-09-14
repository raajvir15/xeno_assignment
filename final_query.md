# Final SQL Query — target_base = 22

    WITH RECURSIVE chain(campaign_id, root_id) AS (
        -- Base case: only campaigns with no parent are true roots
        SELECT id, id
        FROM campaign
        WHERE merchant_id = 501
          AND parent_id IS NULL
          AND creation_status IN ('approved','aborted','resumed','stopped')
          AND processing_status = 'processed'

        UNION ALL

        -- Recursive case: a campaign whose parent is already mapped inherits its root
        SELECT c.id, chain.root_id
        FROM campaign c
        JOIN chain ON c.parent_id = chain.campaign_id
        WHERE c.creation_status IN ('approved','aborted','resumed','stopped')
          AND c.processing_status = 'processed'
    ),
    mapped AS (
        -- Attach every eligible log row to its family root
        SELECT cl.id AS log_id, cl.customer_id, ch.root_id
        FROM communication_log cl
        JOIN chain ch ON cl.communication_id = ch.campaign_id
        WHERE cl.merchant_id = 501
    ),
    chain_roots AS (
        -- Roots that actually have children are real retry chains
        SELECT DISTINCT parent_id AS root_id
        FROM campaign
        WHERE parent_id IS NOT NULL
    ),
    per_root AS (
        SELECT
          m.root_id,
          CASE WHEN cr.root_id IS NOT NULL
               THEN COUNT(DISTINCT m.customer_id)
               ELSE COUNT(*)
          END AS qualifying_count
        FROM mapped m
        LEFT JOIN chain_roots cr ON m.root_id = cr.root_id
        GROUP BY m.root_id, cr.root_id
    )
    SELECT SUM(qualifying_count) AS target_base
    FROM per_root;

Result: target_base = 22
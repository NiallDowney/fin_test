-- =============================================================================
-- MQL vs Trial Analysis: Website Change or Upmarket Shift?
-- =============================================================================
-- All queries use Marketing w-shaped attribution model.
-- Core web leads = Demo + Contact Sales + Livechat (via Intercom) + Trial
-- S2 pipeline filtered to core_noncore = 'core' to match core lead types.
-- Period: Sep 2024 – Mar 2026 (18 months)
-- =============================================================================


-- ---------------------------------------------------------------------------
-- 1. SALES-LED SHARE OF CORE WEB LEADS
-- Sales-led (Demo + Contact Sales + Livechat) / Total core web leads
-- ---------------------------------------------------------------------------
SELECT
    DATE_TRUNC('month', day)::DATE AS month,
    first_person_source,
    SUM(metric_value) AS leads
FROM analytics.mart_sales_funnel_metrics_uncohorted_attributed
WHERE metric = 'leads'
  AND first_person_source IN ('Demo', 'Contact Sales', 'Intercom', 'Trial')
  AND day >= '2024-09-01'
  AND day < '2026-04-01'
GROUP BY 1, 2
ORDER BY 1, 2;

-- Note: first_person_source = 'Intercom' maps to Livechat leads.
-- This is confirmed in dim_leads.sql line 97 where livechat → 'Intercom'.


-- ---------------------------------------------------------------------------
-- 2. FIN.AI vs INTERCOM.COM LEAD SPLIT
-- Website field only populated from ~Apr 2025 onward
-- ---------------------------------------------------------------------------
SELECT
    DATE_TRUNC('month', day)::DATE AS month,
    COALESCE(website, 'intercom.com') AS website,
    SUM(metric_value) AS leads
FROM analytics.mart_sales_funnel_metrics_uncohorted_attributed
WHERE metric = 'leads'
  AND first_person_source IN ('Demo', 'Contact Sales', 'Intercom', 'Trial')
  AND day >= '2024-09-01'
  AND day < '2026-04-01'
GROUP BY 1, 2
ORDER BY 1, 2;


-- ---------------------------------------------------------------------------
-- 3. LEAD VOLUME BY SOURCE TYPE (stacked composition)
-- Shows Demo stability, Contact Sales growth, Livechat surge/fade, Trial trends
-- ---------------------------------------------------------------------------
SELECT
    DATE_TRUNC('month', day)::DATE AS month,
    first_person_source,
    SUM(metric_value) AS leads
FROM analytics.mart_sales_funnel_metrics_uncohorted_attributed
WHERE metric = 'leads'
  AND first_person_source IN ('Demo', 'Contact Sales', 'Intercom', 'Trial')
  AND day >= '2024-09-01'
  AND day < '2026-04-01'
GROUP BY 1, 2
ORDER BY 1, 2;


-- ---------------------------------------------------------------------------
-- 4. LEAD COMPANY SIZE MIX
-- company_reporting_segment: VSB, SB, MM, ENT
-- Grouped as SB = VSB + SB, MME = MM + ENT for analysis
-- ---------------------------------------------------------------------------
SELECT
    DATE_TRUNC('month', day)::DATE AS month,
    company_reporting_segment,
    SUM(metric_value) AS leads
FROM analytics.mart_sales_funnel_metrics_uncohorted_attributed
WHERE metric = 'leads'
  AND first_person_source IN ('Demo', 'Contact Sales', 'Intercom', 'Trial')
  AND day >= '2024-09-01'
  AND day < '2026-04-01'
GROUP BY 1, 2
ORDER BY 1, 2;


-- ---------------------------------------------------------------------------
-- 5. TRIAL COMPANY SIZE MIX
-- Uses self-serve funnel table for trial-specific segmentation
-- ---------------------------------------------------------------------------
SELECT
    DATE_TRUNC('month', day)::DATE AS month,
    company_reporting_segment,
    SUM(metric_value) AS trials
FROM analytics.mart_self_serve_cohorted_funnel_attributed
WHERE metric = 'trial'
  AND day >= '2024-09-01'
  AND day < '2026-04-01'
GROUP BY 1, 2
ORDER BY 1, 2;


-- ---------------------------------------------------------------------------
-- 6. S2 ACV PERCENTILE DISTRIBUTION (Core Pipeline Only)
-- P25, P50, P75, P90 of s2_date_arr across all core S2 deals
-- core_noncore = 'core' ensures only MQL-Demo, MQL-Contact Sales,
-- MQL-Livechat, MQL-Trial sourced pipeline
-- ---------------------------------------------------------------------------
SELECT
    DATE_TRUNC('month', day)::DATE AS month,
    COUNT(*) AS deal_count,
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY s2_date_arr) AS p25,
    PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY s2_date_arr) AS p50,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY s2_date_arr) AS p75,
    PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY s2_date_arr) AS p90
FROM analytics.mart_sales_funnel_metrics_uncohorted_attributed
WHERE metric = 's2_pipeline'
  AND core_noncore = 'core'
  AND day >= '2024-09-01'
  AND day < '2026-04-01'
GROUP BY 1
ORDER BY 1;


-- ---------------------------------------------------------------------------
-- 7. S2 ACV PERCENTILE BY SEGMENT (MME vs SB)
-- MME = MM + ENT, SB = VSB + SB
-- Shows whether ACV growth is broad-based or segment-specific
-- ---------------------------------------------------------------------------
SELECT
    DATE_TRUNC('month', day)::DATE AS month,
    CASE
        WHEN company_reporting_segment IN ('MM', 'ENT') THEN 'MME'
        ELSE 'SB'
    END AS segment_group,
    COUNT(*) AS deal_count,
    PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY s2_date_arr) AS p50
FROM analytics.mart_sales_funnel_metrics_uncohorted_attributed
WHERE metric = 's2_pipeline'
  AND core_noncore = 'core'
  AND day >= '2024-09-01'
  AND day < '2026-04-01'
GROUP BY 1, 2
ORDER BY 1, 2;

# DataAnalytics-Assessment
-- Assessment_Q1.sql
-- Customers with at least one funded savings & investment plan
SELECT
  u.id AS owner_id,
  u.name,
  COUNT(DISTINCT s.id) AS savings_count,
  COUNT(DISTINCT i.id) AS investment_count,
  SUM(sa.confirmed_amount / 100.0) AS total_deposits
FROM users_customuser u
JOIN plans_plan s ON u.id = s.owner_id AND s.is_regular_savings = 1
JOIN plans_plan i ON u.id = i.owner_id AND i.is_a_fund = 1
JOIN savings_savingsaccount sa ON sa.owner_id = u.id
GROUP BY u.id, u.name
HAVING savings_count >= 1 AND investment_count >= 1
ORDER BY total_deposits DESC;

-- Assessment_Q2.sql
-- Analyze average number of transactions per customer per month

WITH txn_stats AS (
  SELECT
    owner_id,
    COUNT(*) AS total_txns,
    DATE_DIFF(MAX(created_at), MIN(created_at), MONTH) + 1 AS tenure_months
  FROM savings_savingsaccount
  GROUP BY owner_id
),
categorized AS (
  SELECT
    owner_id,
    total_txns,
    tenure_months,
    SAFE_DIVIDE(total_txns, tenure_months) AS avg_txn_per_month,
    CASE
      WHEN SAFE_DIVIDE(total_txns, tenure_months) >= 10 THEN 'High Frequency'
      WHEN SAFE_DIVIDE(total_txns, tenure_months) >= 3 THEN 'Medium Frequency'
      ELSE 'Low Frequency'
    END AS frequency_category
  FROM txn_stats
)
SELECT
  frequency_category,
  COUNT(*) AS customer_count,
  ROUND(AVG(avg_txn_per_month), 1) AS avg_transactions_per_month
FROM categorized
GROUP BY frequency_category;

-- Assessment_Q3.sql
-- Flag inactive plans with no inflow for over a year
WITH last_txn_dates AS (
  SELECT
    owner_id,
    MAX(created_at) AS last_transaction_date
  FROM savings_savingsaccount
  GROUP BY owner_id
),
recent_plans AS (
  SELECT
    p.id AS plan_id,
    p.owner_id,
    CASE 
      WHEN p.is_regular_savings = 1 THEN 'Savings'
      WHEN p.is_a_fund = 1 THEN 'Investment'
      ELSE 'Other'
    END AS type,
    l.last_transaction_date,
    DATE_DIFF(CURRENT_DATE(), l.last_transaction_date, DAY) AS inactivity_days
  FROM plans_plan p
  LEFT JOIN last_txn_dates l ON p.owner_id = l.owner_id
  WHERE (p.is_regular_savings = 1 OR p.is_a_fund = 1)
)
SELECT *
FROM recent_plans
WHERE inactivity_days >= 365;

-- Assessment_Q4.sql
-- Estimate Customer Lifetime Value (CLV)

WITH customer_txns AS (
  SELECT
    sa.owner_id,
    COUNT(*) AS total_transactions,
    SUM(confirmed_amount) / 100.0 AS total_amount,
    AVG(confirmed_amount) / 100.0 AS avg_txn_value,
    DATE_DIFF(CURRENT_DATE(), MIN(created_at), MONTH) AS tenure_months
  FROM savings_savingsaccount sa
  GROUP BY sa.owner_id
)
SELECT
  u.id AS customer_id,
  u.name,
  c.tenure_months,
  c.total_transactions,
  ROUND((c.total_transactions / NULLIF(c.tenure_months, 0)) * 12 * (0.001 * c.avg_txn_value), 2) AS estimated_clv
FROM customer_txns c
JOIN users_customuser u ON u.id = c.owner_id
ORDER BY estimated_clv DESC;

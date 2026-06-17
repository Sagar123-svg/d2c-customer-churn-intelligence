# Data Quality Audit Report: D2C Customer Churn Project
**Prepared by:** Lead Data Analyst  
**Snapshot Date Reference:** 2025-09-30  

This document details the critical data-quality anomalies, structural inconsistencies, and target-leakage risks discovered during the initial audit of the raw D2C datasets.

## 1. Critical Target Leakage Risk (High Risk)
* **Affected Dataset:** `orders.csv`
* **Anomaly Details:** The file contains transactional data extending up to `2025-11-29`. Any order line where `order_date > 2025-09-30` belongs strictly to the 60-day post-snapshot performance window.
* **Impact on Modeling:** If these rows are aggregated to build features (e.g., total spend, frequency), the model will inadvertently learn the target label itself, leading to perfect but completely artificial training performance.
* **Remediation:** Apply a strict filter during feature engineering: `df_orders = df_orders[df_orders['order_date'] <= '2025-09-30']`. Post-snapshot records must only be used to validate the labels in `churn_labels.csv`.

## 2. Duplicate and Near-Duplicate Order Records
* **Affected Dataset:** `orders.csv`
* **Anomaly Details:** A significant number of keys in the `order_id` column contain the suffix `_DUP` (e.g., `ORD000412_DUP`). These rows represent intentional duplicate entries with matching or near-matching attributes.
* **Impact on Modeling:** Blindly calculating purchase counts or summing `gross_amount` will artificially inflate customer monetary values and frequencies, distorting RFM segmentation and model weights.
* **Remediation:** In the data pipeline, identify and isolate `_DUP` records. Prior to aggregation, execute a deduplication step keeping the original transaction or drop records matching the `*_DUP` regex pattern after verifying their true origin.

## 3. Structural Missingness in Profiles and Transactions
* **`customers.csv` (`loyalty_tier`):** ~1,386 missing rows. This is an informative null representing customers who did not enroll in the loyalty program. It must be explicitly filled with a category string like `'Unenrolled'` rather than dropped.
* **`customers.csv` (`skin_type`):** ~401 missing rows. This represents a missing-at-random (MAR) opt-in feature from registration. Dropping these rows would discard over 16% of the customer universe. Remediation: Impute as `'Not Provided'`.
* **`orders.csv` (`rating`):** ~80 missing values. Customers frequently skip leaving reviews. Dropping these rows will skew average satisfaction tracking. Remediation: Group by `customer_id` and map to a neutral flag, or handle via an indicator column (`rating_missing`).

## 4. Extreme Financial Outliers
* **Affected Dataset:** `orders.csv`
* **Column:** `gross_amount`
* **Anomaly Details:** While the baseline transactions start around ₹149.0, multiple transactions peak up to an extreme value of ₹24,789.38. Given this is a personal-care brand selling items like skincare and cosmetics, single-order amounts this high are highly statistical anomalies or represent B2B wholesale purchases mixed into D2C channels.
* **Impact on Modeling:** Linear and distance-based models will be severely warped by these values, leading to poor generalization across standard consumer profiles.
* **Remediation:** Clip these values at the 99th percentile or apply a log transformation ($Y = \log(X)$) to compress the distribution tail during feature generation.

## 5. Referential Integrity & Key Alignments
* **Analysis:** A cross-reference checks of `customer_id` across all relational tables confirmed that the customer universe matches exactly 2,400 distinct records in `customers.csv`, `web_events_snapshot.csv`, `churn_labels.csv`, `rfm_modeling_snapshot.csv`, and `intervention_history.csv`.
* **Exception:** `support_tickets.csv` contains only 2,398 unique customers. This is structurally sound, as not all acquired customers have submitted a support request. All joins must use a structural `LEFT JOIN` starting from `customers.csv`.

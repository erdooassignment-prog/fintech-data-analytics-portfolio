# Banking Customer & Deposit Analytics

## Project Overview
A Business Intelligence project analyzing **5,000,000 customer and deposit records** simulating a mid-sized Nigerian commercial bank operating across all 36 states + FCT. The dataset covers savings, current, domiciliary, and fixed deposit accounts, with fields such as BVN (Bank Verification Number, anonymized), account tenor, branch code, KYC tier, account officer, and monthly balance history from Jan 2022–Dec 2024. The end product is a Tableau dashboard used by branch heads and the Retail Banking team to track deposit mobilization and customer attrition.

## Objective
- Build a single BI view of deposit performance across branches, states, and account types.
- Quantify customer concentration risk (how much of total deposits sits with a small number of customers).
- Flag dormant and attrition-risk accounts using balance and transaction-recency rules.
- Give branch and regional managers a self-service dashboard instead of monthly static Excel reports.

## Business Problem
The bank's Retail Banking division relied on a manually compiled Excel report, refreshed monthly by branch operations staff, to track deposit growth. At 5M+ records, this process was:
- Too slow — a full national deposit report took 3–4 working days to compile and reconcile.
- Error-prone — inconsistent branch codes and KYC tier labels across systems caused mismatched totals.
- Reactive — dormant accounts (CBN-defined as no customer-initiated activity for 6+ months) were only identified after quarterly compliance sweeps, not in real time.
- Blind to concentration risk — leadership had no consolidated view of what percentage of deposits sat with the top 1–5% of customers, a key liquidity risk metric.

## Entity Relationship Diagram (ERD)
The star-schema data model used for the Tableau dashboard — `fact_account_balance` at the grain of one row per account per monthly snapshot, surrounded by customer, branch, account-type, and date dimensions.

![ERD - Banking Customer & Deposit Analytics](erd_banking_customer_deposit.png)

## Dataset
`/data/banking_customer_deposit_sample.csv` is a **50,500-row sample** extracted from the full 5,000,000-row simulated core-banking export, generated with `/data/generate_banking_data.py`. The generator deliberately injects real-world messiness consistent with the cleaning steps in the Methodology below: inconsistent state naming ("FCT" / "Abuja" / "F.C.T"), currency-formatted balance strings (₦, commas), missing KYC-tier values, mixed date formats, duplicate export rows, and a small number of test/staff accounts. Run the script with `--rows 5000000` to regenerate the full dataset referenced in this README.

## Methodology
1. **Data extraction (SQL)**: Pulled core banking exports (customer master, account master, monthly balance snapshots) via `JOIN`s on customer_id and account_id; used `GROUP BY` and window functions (`ROW_NUMBER()`, `LAG()`) to build month-on-month balance deltas per account.
2. **Data cleaning (Python — pandas)**: Standardized branch codes and state names (e.g., collapsing "FCT", "Abuja", "F.C.T" into one label), removed test/staff accounts, converted balance fields from string (with ₦ and commas) to numeric, and handled ~2.3% missing KYC-tier values by back-filling from the customer master table.
3. **EDA (Python — pandas, matplotlib/seaborn)**: Distribution analysis of balances by account type and tier; identified heavy right-skew in current accounts (a small number of corporate-linked accounts holding disproportionate balances).
4. **Segmentation**: Built RFM-style segments (Recency of transaction, Frequency of deposits, Monetary balance) and CBN KYC-tier cross-tab to classify customers into Mass Market, Affluent, and Priority Banking tiers.
5. **Dormancy & attrition flagging**: Wrote SQL logic to flag accounts with no debit/credit activity in 180+ days, cross-checked against CBN dormant-account guidelines.
6. **Tableau dashboard build**: Modeled a star schema (fact table = monthly balances, dimensions = customer, branch, account type) and built calculated fields (LOD expressions for customer-level deposit share, running totals for YTD growth).
7. **Validation**: Reconciled Tableau-aggregated totals against SQL `SUM()` outputs for three sample branches before rollout.

## Tools
- **SQL** (PostgreSQL/MySQL syntax) – extraction, joins, window functions, dormancy logic
- **Python** – pandas, NumPy for cleaning and EDA; matplotlib/seaborn for exploratory charts
- **Tableau** – star-schema data modeling, LOD expressions, calculated fields, dashboard actions, parameter-driven filters
- **Excel** – initial data dictionary and stakeholder sign-off on metric definitions

## Skills
- SQL window functions and multi-table joins at scale (5M+ rows)
- Data cleaning and standardization across inconsistent source systems
- RFM/customer segmentation
- Tableau LOD expressions, calculated fields, and dashboard UX design
- Requirement gathering with non-technical stakeholders (branch/regional managers)
- Metric definition and documentation (data dictionary)

## Result & Recommendations
*(figures below are computed directly from the 50,500-row sample dataset in `/data`; re-run against the full 5,000,000-row extract to refresh at production scale)*
- The top 5% of customers by balance held **39.4% of total deposits**, confirming a meaningful concentration risk that leadership had not previously quantified.
- Dormancy flagging surfaced **14.7% of accounts** inactive for 180+ days — materially above the ~5% the compliance team had estimated from manual sampling.
- Current accounts in Lagos and FCT branches showed the strongest deposit balances on average, while accounts tagged to smaller regional branches skewed toward Savings products with lower average balances — a segmentation cut the manual Excel report never surfaced.
- **Recommendations**:
  - Launch a targeted reactivation campaign for the flagged dormant accounts before the next compliance write-off cycle.
  - Introduce a tiered relationship-manager program for the top 5% concentration-risk customers to reduce liquidity exposure from any single withdrawal event.
  - Investigate digital-channel adoption gaps in underperforming regions as a lever for deposit growth, not just branch expansion.
  - Replace the manual Excel report with the Tableau dashboard, refreshed weekly, cutting reporting turnaround from 3–4 days to same-day.

## Next Step
- Automate the SQL extraction with a scheduled job (e.g., cron + stored procedure) so Tableau refreshes nightly instead of manually.
- Add a predictive dormancy-risk score (logistic regression on transaction recency/frequency) to flag at-risk accounts 60–90 days before they go dormant.
- Extend the dashboard with row-level security so each branch manager only sees their own branch by default.
- Present concentration-risk findings to Treasury/ALM team to inform liquidity contingency planning.


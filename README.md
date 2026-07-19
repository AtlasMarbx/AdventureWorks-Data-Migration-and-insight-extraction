# AdventureWorks: End-to-End Data Engineering & EDA Insight Generation

![](https://img.shields.io/badge/Python-3.10-blue)
![](https://img.shields.io/badge/MySQL-8.0-orange)
![](https://img.shields.io/badge/Pandas-%20-purple)
![](https://img.shields.io/badge/scikit--learn-GMM-orange)
![](https://img.shields.io/badge/kmodes-KPrototypes-blue)
![](https://img.shields.io/badge/Azure-Blob_Storage-0089D6)

**80+ tables migrated · 12.4M cells · 3 domains analyzed · 19 phases · 19,800+ customers profiled · k=4 clusters**

An end-to-end manual data engineering and exploratory analysis project migrating **AdventureWorks2022** (80+ tables, 12.4M+ cells) from SQL Server to MySQL, followed by structured EDA and unsupervised customer profiling.

**Key findings:** 87.90% online orders · 61.73% foreign revenue · 74.60% of revenue from 6.74% of customers (businesses)

---

### Migration (Phases 1–7)

<details>
<summary>Phase 1: Data Acquisition & Extraction</summary>

**Problem**
The source dataset was a `.bak` (SQL Server native backup) file — not portable to MySQL. No direct conversion tools existed that were both secure and reliable for company-grade data.

**Solution**
- Restored `.bak` into SQL Server Management Studio (SSMS) v19.3
- Used SSMS Export Wizard with flat-file (CSV) format, exporting one table per file
- Deliberately avoided third-party `.bak→CSV` converters to eliminate security risks

**Business Gain**
- Zero reliance on untrusted third-party tools — data sovereignty maintained throughout migration
- No data exfiltration risk from opaque conversion utilities

**Technical Gain**
- Deep understanding of SSMS export mechanics, data type compatibility, and cross-RDBMS migration planning
- Learned that "if it works, don't touch it" — hours were lost reverting driver settings on a working Export Wizard configuration

</details>

<details>
<summary>Phase 2: Cross-RDBMS Migration (SQL Server → CSV → MySQL)</summary>

**Problem**
SQL Server and MySQL use fundamentally different type systems — `MONEY`, `XML`, `NVARCHAR`, `VARBINARY`, `GEOGRAPHY`, `BIT`, and `UNIQUEIDENTIFIER` have no direct MySQL equivalents. CSV is a lossy intermediate format; the Export Wizard alone failed repeatedly due to data type mismatches, nullable constraints, and delimiter conflicts.

**Solution**
- **Intermediate format:** CSV (flat-file) — each of 80+ tables exported individually
- **Data type mapping:**
  - `MONEY` → `DECIMAL(10,5)`
  - `NVARCHAR(n)` → `VARCHAR(n) CHARACTER SET utf8mb4`
  - `BIT` → `TINYINT(1)`
  - `UNIQUEIDENTIFIER` → `CHAR(36)`
  - `XML`, `VARBINARY`, `GEOGRAPHY` — left at source (no analytical value, corruption risk)
- **Bulk loading:** Discovered and adopted `LOAD DATA INFILE` — integrated 2.4M cells in ~5 seconds, replacing row-by-row `INSERT` that previously took ~50+ hours for 12.4M cells
- Discovered that `NULLABLE` in Export Wizard means "column may be NULL" not "keep NULLs" — a single checkbox misinterpretation caused cascading export failures

**Business Gain**
- 99.997% faster data loading (row-by-row INSERT took ~50+ hours; `LOAD DATA INFILE` integrated 2.4M cells in ~5 seconds)
- Complete migration without data loss on analytical columns

**Technical Gain**
- Mastery of `INFORMATION_SCHEMA.COLUMNS` for cross-RDBMS type inspection
- Practical understanding of `DECIMAL(p,s)` precision vs. scale
- Competence with `utf8mb4` for international character support

</details>

<details>
<summary>Phase 3: Data Cleaning & Integrity</summary>

**Problem**
CSV export introduced systematic issues: shifted columns (commas inside strings misinterpreted as delimiters), NULLs exported as empty strings, multi-language strings breaking parsing, and unpredictable structural inconsistencies in views.

**Solution**
- **Google Sheets pipeline:** Each CSV inspected and repaired before MySQL import
- **Key functions used:**
  - `=IF(ISBLANK(...), "NULL", ...)` — normalise empties to NULL
  - `=ARRAYFORMULA(IF(ISBLANK(range), "NULL", range))` — bulk NULL fills across 18.5K+ rows
  - `=COUNTIF()` paired with SQL Server source counts — cross-validated row integrity post-cleaning
  - `=LEN()` vs. `=COUNTA()` — correctly detected single-character strings vs. non-empty cells
- **Manual reconciliation:** ~7 hours for a single 2.5M-cell table; each view required unique cleaning logic due to unpredictable structure
- Conditional formatting used as a visual debugger to catch misplaced cells

**Business Gain**
- Zero analytical columns discarded — every row that could hold insight was preserved and corrected
- Manual data cleaning is ~80% of project effort — automation opportunities identified for future iterations

**Technical Gain**
- Developed "vertical partitioning" intuition naturally (later validated by senior practitioners)
- Learned that data cleaning is the dominant time sink in any analytics pipeline
- Built a reusable mental framework: inspect → hypothesise → test → correct → validate

</details>

<details>
<summary>Phase 4: Data Modeling (EER Diagram & Relationships)</summary>

**Problem**
80+ tables across 5 business domains (Sales, Production, Purchasing, Human Resources, Person) with no formal relationships, no primary/foreign keys, orphan records, and incorrect data types embedded during import.

**Solution**
- **Star-schema-per-domain** design — each domain got its own central fact table with surrounding dimension tables
- Domain isolation prevents spaghetti models — cross-domain joins were deliberately avoided because joining central tables across domains creates key conflicts and an unmaintainable EER diagram
- **Key operations:**
  - Synchronised MySQL EER diagram ↔ physical database via forward engineering
  - Defined primary keys, foreign keys, and UNIQUE constraints
  - Enforced 1:1 cardinality by adding UNIQUE to foreign key columns
  - Handled orphan records via `LEFT JOIN ... WHERE parent IS NULL` — inserted missing parent keys to preserve child data
- **Error catalogue** (compiled from real failures):
  - `1452`: FK column has fewer matching PK values
  - `1822`: FK references non-existent PK
  - `1265` / `1175`: Strict/safe mode blocking operations
  - `1064`: Syntax errors

**Business Gain**
- Ad-hoc analytical queries across domains became possible (revenue per employee, product profitability by territory) via runtime SQL JOINs — no formal cross-domain FK constraints needed
- Star-schema-per-domain keeps the EER diagram clean and maintainable; cross-domain relationships exist only at query time, not in the schema

**Technical Gain**
- Proficient with MySQL Workbench EER diagram sync (model → database and database → model)
- Understanding of indexing: `B-Tree`, `EXPLAIN` for query plan analysis
- Practical distinction between 1:1, 1:n, and many-to-many cardinalities and their enforcement via keys

</details>

<details>
<summary>Phase 5: Performance Optimization</summary>

**Problem**
Cross-table joins on large datasets (e.g., 31K-row sales × 121K-row details) without indexes resulted in full table scans and slow query response.

**Solution**
- **B-tree indexes** on frequently joined/queried columns: `CREATE INDEX idx_sod_product ON SalesOrderDetails(ProductID);`
- Targeted indexing on 5 core tables: `SalesOrderHeader`, `SalesOrderDetails`, `Product`, `Employee`, `Customer`
- Used `EXPLAIN SELECT` to verify index usage and identify bottlenecks

**Business Gain**
- Faster time-to-insight — analytical queries that previously took minutes returned in milliseconds

**Technical Gain**
- Indexes are a trade-off: faster `SELECT`, slower `UPDATE`/`INSERT`, extra disk space
- `SHOW INDEX FROM table` and `DROP INDEX` for lifecycle management

</details>

<details>
<summary>Phase 6: Cloud Backup & Security</summary>

**Problem**
A local-only dataset is a single ransomware event away from total loss. Sensitive employee data (NationalIDNumber, LoginID, hashed passwords) required secure handling.

**Solution**
- **Azure Blob Storage (Gen2, Cool Tier)** — project artefacts, cleaned CSVs, data model backups, and logs stored off-site
- **Access management:** RBAC ownership levels, SAS tokens, and connection strings for programmatic access
- **Sensitive data handling:**
  - Employee PII left flagged for secure-environment processing — holds no analytical value for sales/product analysis
  - Password columns excluded — zero contribution to EDA insights, and storing them adds liability with no return
  - Recommendation documented: "Do not store sensitive production data on local machines — use cloud-native or air-gapped environments"
- **Backup discipline:** Every phase end (cleaning, modeling, relationship-building) triggered an Azure sync
- Discovered that Azure `$logs` is a system container with immutable permissions after hours of RBAC troubleshooting

**Business Gain**
- Full disaster recovery capability — any point in the project can be restored from cloud backup
- Backups saved the project twice (power shutdown during a 29-hour data integration would have been catastrophic)

**Technical Gain**
- Hands-on Azure RBAC and `azcopy` with `az login` vs. SAS-token-based auth
- Learned to read the docs before assuming you can modify privileged resources

</details>

<details>
<summary>Phase 7: Python EDA Integration</summary>

**Problem**
Analysis needed to move from SQL queries to Python (pandas, matplotlib, seaborn) for richer statistical exploration. The connection code needed to be clean and straightforward for local EDA.

**Solution**
- **Credentials:** Stored in `.env`, loaded via `python-dotenv` + `os`
- **Context-managed engine:** `with engine.connect() as connection: df = pd.read_sql_query(...)`
- **No unnecessary abstraction** — `text()` and parameterised placeholders deliberately omitted for local manual EDA. The same pattern is documented as essential for backend services (prevents SQL injection in user-facing applications)
- **Driver insight:** Used `pymysql` with the `cryptography` package — an additional dependency learned when the authentication handshake failed. Unlike Oracle's `mysql-connector-python` (which bundles it), PyMySQL relies on `cryptography` for 1-way hashed authentication
- **Special character encoding:** utilized `quote_plus` on special characters such as `@`, `#`, `$`, to automatically detect password and database special character changes and encode them

**Business Gain**
- Clean, minimal EDA pipeline with no dead abstraction
- Backend-security insight documented and ready to apply when user-facing code is needed

**Technical Gain**
- SQLAlchemy `Engine` as a connection daemon vs. raw connections
- Knowing when *not* to over-engineer — `text()` + `params` is valuable for backend services, unnecessary for local EDA
- `pandas.read_sql_query` with pure SQL strings keeps the code simple when queries are trusted

</details>

### Analysis (Phases 8–18)

<details>
<summary>Phase 8: Data Understanding & Table Selection</summary>

**Problem**
10+ interconnected sales tables across multiple business domains — needed to identify which tables held analytical value and which could be deferred without compromising insight quality.

**Solution**
- Selected 10 target tables: SalesOrderHeader, SalesOrderDetails, Product, ProductCategory, ProductSubCategory, ShipMethod, SalesTerritory, SpecialOffer, Customers, CurrencyRateID
- Designed a robust query infrastructure:
  - `.env` for sensitive variables
  - `quote_plus` for credential encoding before DB connection
  - Parameter dictionary pattern documented as guardrail against SQL injection (omitted for manual EDA, flagged for automated pipelines)
- Adopted additive table strategy — added tables as needed rather than overengineering upfront, given the database's sheer scale

**Business Gain**
- Focused analytical scope on `operational` and `customer-based` insights directly tied to revenue and customer behavior
- Database scale was underestimated — adopted an additive table strategy, pulling in additional tables as needed rather than engineering for exhaustive coverage upfront

**Technical Gain**
- Practical judgment in scoping EDA to business-relevant tables rather than exhaustive coverage
- Understanding that query infrastructure design must balance security (parameterization) with context (manual vs. automated pipeline)

</details>

<details>
<summary>Phase 9: Null Analysis & Initial Business Assumptions</summary>

**Problem**
Several columns in SalesOrderHeader contained null values at varying ratios — needed to determine whether these represented meaningful business signals or data quality issues.

**Solution**
- Profiled null ratios across all 26 columns of SalesOrderHeader (31,465 rows)
- Identified 6 columns with nulls using vectorized `isna().mean()` across the DataFrame
- Cross-referenced null patterns to formulate testable business hypotheses:
  - `SalesPersonID` and `PurchaseOrderNumber` at 87.90% null → online order hypothesis
  - `CreditCardID` and `CreditCardApprovalCode` at 3.59% null → payment method hypothesis
  - `CurrencyRateID` at 55.58% null → domestic vs. foreign transaction hypothesis
- Validated the online order hypothesis against `OnlineOrderFlag` — confirmed at exactly 87.90%

**Business Gain**
- **87.90% of transactions are online orders** — validates digital-first channel strategy; offline sales infrastructure can be deprioritized
- **96% of transactions use credit cards** — card processing partnerships are critical; cash/alternative payment infrastructure is negligible
- **38.27% domestic vs. 61.73% foreign** — the business is predominantly international; local market optimization is secondary to foreign market expansion

**Technical Gain**
- Vectorized null profiling with `df.isna().mean()` for efficient column-level missing data assessment
- Cross-validation technique: using one column's null pattern to validate assumptions about another column
- Connecting data quality signals to testable business hypotheses before statistical modeling

</details>

<details>
<summary>Phase 10: Foreign Market Decomposition & Currency Normalization</summary>

**Problem**
Multi-currency financial data across 10 territories required normalization to a common numeraire (USD) before any aggregative or comparative analysis was valid. Additionally, the relationship between territories, currency conversion, and transaction volume needed decomposition.

**Solution**
- Merged SalesOrderHeader with SalesTerritory on `TerritoryID` and CurrencyRate on `CurrencyRateID`
- Segmented territories into domestic (IDs 1-5) and foreign (IDs 6-10) based on business-defined grouping
- Computed domestic/foreign transaction ratios and decomposed foreign market share:
  - Australia 35.23%, Canada 20.94%, UK 16.57%, France 13.76%, Germany 13.50%
- Identified that 100% of domestic currency conversions were to CAD (Canadian dollars) — only 21 transactions (~2% of domestic), indicating cross-border purchasing from Canadian customers
- Normalized all financial columns to USD using `EndOfDayRate`: `SubTotal_USD = SubTotal / EndOfDayRate`, propagating null CurrencyRateID with USD base rate (ID 13520)
- Engineered derived metrics: `TotalDue_USD`, `FreightBurden%`, `TaxRatio%` with zero-division guardrails using `np.where`

**Business Gain**
- **Australia is the dominant foreign market at 35%** — marketing and logistics investment should prioritize this region
- **Canadian cross-border purchasing exists but is minimal (~2% of domestic)** — localized CA operations are not justified
- **Currency-normalized financial data enables unbiased RFM and clustering** — all subsequent monetary aggregations are now apples-to-apples

**Technical Gain**
- Currency normalization pipeline: `fillna(base_rate) → merge → divide → guard` — a reproducible pattern for any multi-currency dataset
- `np.where` for safe ratio engineering (avoids `inf`/`NaN` from zero-division)
- Understanding that `value_counts(normalize=True)` gives immediate market share decomposition without manual percentage calculation

</details>

<details>
<summary>Phase 11: Column Disposition Strategy & Data Retention</summary>

**Problem**
Of 26 columns in SalesOrderHeader, several held zero analytical value or contained sensitive data that should not enter the modeling pipeline. A principled retention/drop decision was needed for each.

**Solution**
- **Dropped columns (zero/low value):**
  - `Comment` — 100% empty, no information content
  - `rowguid` — GUID, no analytical relevance
  - `ModifiedDate` — data entry timestamp, not decision-relevant
  - `CreditCardID`, `CreditCardApprovalCode` — sensitive PII, no EDA value
- **Retained null columns without propagation:**
  - `PurchaseOrderNumber` — 87.90% null, signals online orders (valuable categorical signal)
  - `SalesPersonID` — 87.90% null, same online order signal
  - `CurrencyRateID` — propagated with USD default (13520) for normalization pipeline
- **Observation:** The VEB (Venezuelan Bolívar) currency code in CurrencyRate suggests AdventureWorks data before 2008–2025 was bootstrapped with newer features per version

**Business Gain**
- Reduced feature space from 26 to 21 without losing signal — leaner dataset improves model training efficiency
- Sensitive columns excluded from EDA pipeline — no compliance risk with payment card data

**Technical Gain**
- Principled column disposition framework: (1) empty → drop, (2) identifier → drop, (3) sensitive → drop, (4) null with signal → retain, (5) null with operational use → propagate
- Detecting bootstrapped historical data via currency code anomalies (VEB discontinued in 2008)

</details>

<details>
<summary>Phase 12: Customer-Level Aggregation & Transaction Profiling</summary>

**Problem**
Raw sales data was at the order-line level — needed to aggregate to customer level for RFM analysis and unsupervised customer segmentation, while identifying high-value and outlier accounts.

**Solution**
- Switched merge key from `SalesOrderID` to `CustomerID` after discovering it was already present in the table
- Aggregated two key customer metrics via `groupby('CustomerID')`:
  - `Transactions_total`: sum of `SubTotal_USD` per customer
  - `Transactions_count`: count of transactions per customer
- Sorted and inspected top 30 customers by total spending and transaction count
- Identified outlier customers with transaction counts of **16, 17, 25, 27, 28** — representing extreme purchase frequency
- Analyzed outlier spending profile: transaction cost interval ~$358–$1,120, average ~$669

**Business Gain**
- **Extreme outliers (0.18% of customers) hold $34,501 (0.02% of total)** — small but non-ignorable revenue segment
- Customer-level aggregation is the prerequisite for all downstream clustering and personalized marketing models
- Top-30 analysis revealed initial revenue concentration ($877K–$514K range) but was a biased snapshot — confirmed that unsupervised methods like GMM/k-prototypes were necessary for unbiased segmentation

**Technical Gain**
- `groupby` + `agg` pattern for customer-level feature engineering from transactional data
- Practical outlier identification via distribution inspection — confirmed GMM with `full` covariance as the appropriate model (handles clusters of varying shapes and sizes)
- Recognized that top-30 analysis is a biased heuristic that does not represent the true distribution — reinforced the need for unsupervised, distribution-aware methods

</details>

<details>
<summary>Phase 13: Tax Ratio Hypothesis Testing (Linear vs. Non-Linear)</summary>

**Problem**
A hypothesis emerged: transactions with low `SubTotal_USD` might have a higher average `TaxRatio%`, possibly explaining revenue leakage on small orders. Needed rigorous statistical testing with appropriate model comparison.

**Solution**
- Segmented transactions into Low/Medium/High `SubTotal` categories using quantile thresholds
- Preserved natural class imbalance rather than resampling — rebalancing would introduce bias against real customer behavior patterns
- Compared two models to test linearity of the `SubTotal`→`TaxRatio%` relationship:
  - **LinearRegression** (linear baseline)
  - **RandomForestRegressor** (non-linear capable)
- Analyzed explained variance (R²) gap between models:
  - LinearRegression R²: ~20.86%
  - RandomForestRegressor R²: ~46%
  - Gap: ~20.24% — confirms the relationship is predominantly non-linear
- Verified non-normality of transaction distribution via visualization
- Pearson correlation returned ~51% — moderate linear component within a predominantly non-linear structure

**Business Gain**
- **The low-subtotal-higher-tax hypothesis is partially false** — the relationship is mostly non-linear, meaning tax increases are not systematically penalizing small orders
- **~46% of tax variance remains unexplained** — omitted features (product category, territory tax policy, shipment method) are needed for a complete model
- Linear policies (flat tax adjustments) would address only ~20% of the variance — non-linear modeling is essential for accurate tax impact analysis

**Technical Gain**
- Model comparison framework for linearity assessment: LinearRegression vs. RandomForest R² gap quantifies non-linearity magnitude
- Principled stance on class imbalance: preserving natural distribution is less biased than resampling
- Visual residual analysis as diagnostic tool for model selection

</details>

<details>
<summary>Phase 14: Logistics Timeline Analysis & Shipment Uniformity Discovery</summary>

**Problem**
Time interval features (`TimeInterval_ship`, `TimeInterval_due`, `BufferTime`) were engineered to test whether delivery timelines and buffer windows correlated with tax rate variations across regions.

**Solution**
- Calculated date intervals: `OrderDate → ShipDate`, `OrderDate → DueDate`, `ShipDate → DueDate` (buffer)
- Noted that MySQL timestamps consistently use `00:00:00` — dates are reliable for interval calculation
- Tested correlation between all three time-based features and `TaxRatio%`
- **Key finding:** `BufferTime` returned **NaN** correlation — total absence of variance in the buffer window across 31,465 transactions, with a single 5-day outlier

**Business Gain**
- **Fulfillment scheduling is entirely uniform** — regional tax rate differences have zero impact on delivery expectations
- Operational insight: logistics execution is standardized regardless of destination market tax structure

**Technical Gain**
- A NaN correlation is itself a powerful finding — zero variance in a feature is a statistically meaningful signal
- Feature engineering of date intervals from datetime columns in pandas (vectorized subtraction → `.dt.days`)
- Understanding that "no correlation" and "uncorrelatable" (zero variance) are different diagnostic results

</details>

<details>
<summary>Phase 15: Multicollinearity Diagnosis & Shipment Method Disparity</summary>

**Problem**
Perfect correlations (1.00) between `SubTotal`, `TaxAmt`, `Freight`, and `TotalDue` suggested severe multicollinearity. Additionally, `ShipMethodID` needed analysis for its impact on cost ratios.

**Solution**
- Computed full correlation matrix and confirmed perfect multicollinearity among core financial features
- Used eigenvalue decomposition — features covary in synchrony **99.998%** of the time
- Isolated `ShipMethodID` groups and applied **Mann-Whitney U test** (non-parametric, appropriate for nominal numerical data)
- **Discovery:** Only two shipment methods are actively used in the sales pipeline:
  - **Group 1 — ShipMethodID 1 (Ground Trucks):** 79.42% domestic, 91.57% foreign. Absolute zero variance in `FreightBurden%` and `TaxRate%`. Operates as strict flat-rate pricing matrix.
  - **Group 2 — ShipMethodID 5 (International Cargo):** 20.58% domestic, 8.43% foreign. Right-skewed with extreme outliers up to 22%. Dynamic pricing absorbing international tariffs.
- Other mediums (ZY-EXPRESS, OVERSEAS-DELUXE, OVERNIGHT J-FAST) are reserved for `PurchaseOrderHeader` (business procurement of components, not customer sales)

**Business Gain**
- **Ground trucks handle ~80%+ of all transport** — the flat-rate system is the operational backbone; any disruption here impacts the majority of orders
- **Dynamic cargo pricing absorbs all international volatility** — cost spikes up to 22% are isolated to a small fraction of shipments
- Premium shipping methods are irrelevant to customer sales — not a customer-facing differentiator

**Technical Gain**
- Eigenvalue decomposition for multicollinearity quantification — surpassing simple correlation matrix inspection
- Mann-Whitney U test selection rationale: non-parametric, two-sample, two-tailed — appropriate for nominal-group comparison with non-normal distributions
- Discovery that feature gaps (unused shipment methods) reveal business process boundaries (procurement vs. sales)

</details>

<details>
<summary>Phase 16: Product Portfolio Analysis — Finished Goods vs. Components</summary>

**Problem**
The Product table (504 products) contained mixed product states — finished goods, raw components, and assemblies — without clear differentiation. Needed to segment the product portfolio for meaningful sales analysis.

**Solution**
- Identified that `DiscontinuedDate` is 100% empty — all products are active in the current business model
- Through collaborative audit with an LLM peer, recognized that `SizeUnitMeasureCode` and `Weight` are strictly tied — both capture measurement data, but neither is used as a cost factor in pricing. Eliminated both from freight burden analysis as they carry no explanatory value for cost calculations
- Decomposed `StandardCost` and `ListPrice` placeholder pattern (0.000 / 0.00):
  - **200 products (~40%):** Non-finished raw components with zero placeholder costs
  - **~10 products (~2%):** Non-finished products sold as standalone assemblies
  - **Remaining (~58%):** Finished products with genuine cost data
- The 40% `Weight` non-null ratio aligns with `SizeUnitMeasureCode` — both reflect that measurement tracking exists only for non-raw-component products, not that measurement drives pricing
- **Manufacturing analysis:** 47.42% manufactured internally vs. 52.58% resale
- **Production timeline:** Internal manufacturing takes 2–4 days; resale takes 0–1 days (no raw component preprocessing)
- **Resupply threshold:** Flat 75% across all products — consistent replenishment policy

**Business Gain**
- **~58% of products are revenue-generating finished goods** — the remaining ~42% are operational inputs, not customer-facing
- **Resale products outpace internal manufacturing** (52.58%) — the business model favors curation over production
- Flat 75% resupply threshold means inventory policy is not optimized per product category — opportunity for differentiated just-in-time strategies

**Technical Gain**
- Placeholder cost detection as proxy for product state classification
- Null ratio decomposition to infer business logic (weight tracking = non-raw-component status)
- Cross-referencing manufacture days with product state reveals operational efficiency patterns

</details>

<details>
<summary>Phase 17: Customer Segmentation — Business vs. Individual Analysis</summary>

**Problem**
Customer type distribution (business vs. individual) was unknown but critical for understanding transaction concentration risk and tailoring marketing strategy.

**Solution**
- Identified **701 inactive customers (3.54%)** — never placed an order, no interest in product catalog
- Computed business vs. individual split: **6.74% businesses, 93.26% individual buyers** (based on transaction report)
- Cross-referenced customer type with transaction aggregation:
  - Businesses = ~7% of customer base
  - Businesses = **50% of transaction frequency**
  - Businesses = **74.60% of total spending**
- Discovered invoicing operates on `ProductID` basis (not `OrderQty`) — multiple quantities of the same product generate one invoice; different products generate separate invoices

**Business Gain**
- **74.60% of revenue comes from 6.74% of customers (businesses)** — this is extreme concentration risk; losing top business accounts would be catastrophic
- **Inactive customers (3.54%) are a recovery opportunity** — re-engagement campaigns could reactivate ~700 accounts

**Technical Gain**
- Concentration ratio analysis: minority segment (7%) driving majority of revenue (74.6%)
- Inactive customer identification via transaction history join

</details>

<details>
<summary>Phase 18: Top/Bottom Product Performance & Seasonality Decomposition</summary>

**Problem**
Product-level sales performance and seasonal patterns needed systematic decomposition to identify revenue drivers, inventory planning signals, and underperformers.

**Solution**
- Ranked products by total spending and purchasing frequency at both product and subcategory level
- **Top 10 products (by total spending):** Mountain-200 Silver/Black (38/42/46cm), Road-250 Black (44/48/52cm), Road-350-W Yellow
- **Bottom 10 products (by total spending):** Mountain Bike Socks L, Bike Headset LL, Touring Frame Blue LL-58, Bike Road Seat/Saddle LL, Touring Seat/Saddle LL, Touring Seat/Saddle ML, Mountain Frame Black LL-52, Mountain Frame-W Silver ML-38, Mountain Frame Black LL-40, Touring Handlebars LL
- **Top subcategories:** Mountain Bikes > Road Bikes (by spending); Road Bikes > Tires and Tubes > Mountain Bikes > Helmets > Bottles and Cages (by frequency)
- **Lowest subcategories:** Touring Frames, Socks, Saddles, Mountain Frames, Headsets, Handlebars (by spending); Headsets, Locks, Chains, Bike Stands, Forks (by frequency)
- **Seasonality analysis per product:** Extracted high-selling and lowest-selling months for each of the top 10 and bottom 10 products
- Key seasonal patterns identified:
  - **Golden rule:** Bike demand peaks late-spring to summer (Mar–Jun)
  - **September outlier:** Back-to-school and fitness activity demand
  - Lowest Months: February and, notably, April—which falls inside the peak window but frequently registers drops for individual products. The leading hypothesis is spring rain and weather volatility; however, this remains unvalidated due to a lack of qualitative survey data. To confirm this correlation cost-effectively, integrating historical weather metadata into each transaction log would provide a highly robust, low-expense validation model
  - Component/frame demand mirrors full-bike seasonality but customers prefer completed bikes
  - Product 710 (Mountain Frame Black LL-40) breaks pattern — peaks in October/August

**Business Gain**
- **Mountain-200 and Road-250 are the revenue-critical SKUs** — stockouts here have outsized revenue impact
- **Seasonal inventory strategy:** Ramp up March–June, reduce January–February
- **September back-to-school demand is an underutilized marketing window** for bike and fitness categories
- **Low-selling components (Headsets, Locks, Chains) may be candidates for discontinuation or bundling** — they consume shelf space without proportional revenue

**Technical Gain**
- Product-level seasonality extraction: groupby month → sort → top/bottom N extraction per product
- Subcategory rollup for hierarchical product performance analysis
- Pattern recognition: distinguishing unified seasonal demand (all bikes) from product-specific anomalies (Product 710)

</details>

### Clustering (Phase 19)

<details>
<summary>Phase 19: Dual Clustering — GMM (Monetary/Frequency) & KPrototypes (Monetary + Subcategory)</summary>

**Problem**
Transaction data contained a mix of continuous numeric features (monetary, frequency) and categorical features (product subcategories). A single clustering algorithm could not handle both types natively. Needed separate approaches for each data modality.

**Solution**
- **GMM (Gaussian Mixture Model)** applied to the Monetary/Frequency numeric subset only — accommodated non-normal distributions and overlapping cluster boundaries without assuming spherical clusters
- **KPrototypes** applied to the combined numeric + categorical space (monetary metrics + product subcategory behavior) — handles mixed-type data natively by combining k-means (numeric) and k-modes (categorical) distance metrics
- Used **VIF (Variance Inflation Factor)** with `add_constant` to detect multicollinearity before GMM clustering — justified switching covariance type from `full` to `diag` to reduce computational cost
- BIC elbow at k=4 confirmed by a sharp drop from k=1→4 and a spike upward at k=5 — statistical optimum aligns with business interpretability
- Selected k=4 as both the statistical and the actionable configuration: directly maps to differentiated campaign design, discount tiering, and retention prioritization without over-splitting the 2-dimensional Monetary/Frequency space
- **GMM cluster profiles (Monetary/Frequency, k=4):**

  | Cluster | Customers | Freq_mean | Total_mean | Total_spent | Ratio |
  |---|---|---|---|---|---|
  | 0 | 910 | 3.00 | $5,894 | $5,363,892 | 4.76% |
  | 1 | 5,473 | 2.00 | $2,443 | $13,373,125 | 28.63% |
  | 2 | 11,649 | 1.00 | $574 | $6,687,486 | 60.93% |
  | 3 | 1,087 | 5.65 | $70,987 | $77,162,591 | 5.69% |

- **KPrototypes cluster profiles (Monetary + Subcategory):**

  | Cluster | Top subcategories | Total_revenue |
  |---|---|---|
  | 0 | Road Bikes (41%), Mountain Bikes (35%), Touring Bikes (13%), Mountain Frames (4%), Road Frames (3%) | $98,794,495 |
  | 1 | Jerseys (17%), Helmets (13%), Shorts (7%), Road Frames (6%), Mountain Frames (6%) | $3,380,343 |
  | 2 | Tires and Tubes (53%), Bottles and Cages (14%), Fenders (9%), Gloves (8%), Caps (7%) | $412,216 |

**Business Gain**
- **k=4 enables targeted marketing actions** — each cluster maps to a distinct campaign strategy, discount tier, and retention approach
- Dual approach (GMM + KPrototypes) captures both spending patterns and product category affinities — richer segmentation than either method alone
- Revenue contribution per cluster identifies which segments to defend vs. grow

**Technical Gain**
- VIF-based covariance type selection: `full` → `diag` reduces parameters from O(k×d²) to O(k×d) without sacrificing cluster separation
- BIC elbow (k=4) with spike validation at k=5: confirmed the statistical optimum before accepting it as the business choice
- GMM as a generative model: produces cluster probabilities, not hard assignments — enables nuanced customer treatment
- KPrototypes for mixed-type clustering: bridges the gap between purely numeric and purely categorical segmentation approaches

</details>

---

## Project Summary

|---|---|
| Source tables migrated | 80+ |
| Largest table (SalesOrderDetails) | 121K rows, 12.4M cells |
| Business domains migrated | 5 (Sales, Production, Purchasing, HR, Person) |
| Business domains analyzed (EDA) | 3 (Sales, Production, Purchasing) |
| Data cleaning hours (largest table) | ~7 hours |
| Integration speedup (after `LOAD DATA INFILE`) | 99.997% |
| Cloud backups triggered | At every phase completion |
| EDA tables analyzed in depth | 10 |
| Total customers profiled | ~19,800+ |
| Products analyzed | 504 |
| GMM optimal k (BIC elbow + business) | k=4 |
| Online order ratio | 87.90% |
| Foreign market revenue share | 61.73% |
| Business customer revenue concentration | 74.60% from 6.74% of customers |

---

## Lessons From the Trenches

- **`LOAD DATA INFILE` is ~99.997% faster than row-by-row INSERT** — 50+ hours of integration dropped to seconds. Always research bulk-load options before starting any migration.
- **"If it works, don't touch it"** — hours were lost reverting and re-reverting driver settings. Debugging is valuable; over-debugging a working configuration is a timeline risk.
- **Azure `$logs` is a system container** — its permissions are immutable. Read the docs before assuming you can modify privileged resources.
- **NULLABLE in Export Wizard means "column may be NULL" not "keep NULLs."** A single checkbox misinterpretation caused cascading export failures.
- **Backups saved the project twice** — a power shutdown during a 29-hour data integration would have been catastrophic without the Azure-reserved model file.
- **Domain isolation prevents spaghetti models** — 80+ tables across 5 domains modeled as independent star schemas. Cross-domain joins create key conflicts and unmaintainable EER diagrams.
- **Vertical partitioning is learnable by doing** — splitting tables to resolve CSV export issues was later validated as a standard data modeling technique.
- **Manual data cleaning is the real time sink** (~80% of project effort). Automate early where possible, but manual inspection catches structural issues that scripts miss.
- **NaN correlation is a finding, not a failure** — zero variance in BufferTime revealed uniform logistics execution across all tax zones.
- **BIC elbow with spike validation confirms model selection** — k=4 was the statistical optimum (elbow at 4, spike at 5) and also the business-actionable choice; no need to override the data
- **Minority customer segments can drive majority revenue** — 6.74% businesses accounted for 74.60% of total spending.

---

**Key philosophy:** *"Normalising complexity into clarity."*

Every error was documented, every workaround tested, and every lesson learned is reflected in this repository — so that peers, collaborators, or future employers can see not just the output, but the engineering judgment behind it.

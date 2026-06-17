# AdventureWorks Data Migration and insight extraction: SQL Server -> MySQL -> EDA -> insight generation

An end-to-end manual data engineering project migrating **AdventureWorks2022** (80+ tables, 12.4M+ cells across 121K+ rows in the largest table) from Microsoft SQL Server to MySQL, establishing a clean, industry-standard baseline for downstream analytics.

---

## 1. Data Acquisition & Extraction

**Problem**  
The source dataset was a `.bak` (SQL Server native backup) file — not portable to MySQL. No direct conversion tools existed that were both secure and reliable.

**Solution**  
- Restored `.bak` into SQL Server Management Studio (SSMS) v19.3
- Used SSMS Export Wizard with flat-file (CSV) format, exporting one table per file
- Avoided third-party `.bak→CSV` converters to eliminate security risks with company-grade data

**Business Gain**  
Zero reliance on untrusted third-party tools — data sovereignty maintained throughout migration.

**Technical Gain**  
Deep understanding of SSMS export mechanics, data type compatibility, and the nuances of cross-RDBMS migration planning.

---

## 2. Cross-RDBMS Migration (SQL Server → CSV → MySQL)

**Problem**  
SQL Server and MySQL use fundamentally different type systems — `MONEY`, `XML`, `NVARCHAR`, `VARBINARY`, `GEOGRAPHY`, `BIT`, and `UNIQUEIDENTIFIER` have no direct MySQL equivalents. CSV is a lossy intermediate format; the Export Wizard alone failed repeatedly due to data type mismatches, nullable constraints, and delimiter conflicts.

**Solution**  
- **Intermediate format:** CSV (flat-file) — each of 80+ tables exported individually
- **Data type mapping:**

| SQL Server | MySQL Equivalent |
|---|---|
| `MONEY` | `DECIMAL(10,5)` |
| `NVARCHAR(n)` | `VARCHAR(n) CHARACTER SET utf8mb4` |
| `BIT` | `TINYINT(1)` |
| `UNIQUEIDENTIFIER` | `CHAR(36)` |
| `XML` | Split into separate table or left at source |
| `VARBINARY` | Left at source (encrypted documents) |
| `GEOGRAPHY` | Left at source (spatial data) |

- **Bulk loading:** Discovered and adopted `LOAD DATA INFILE` — integrated 2.4M cells in ~5 seconds, replacing row-by-row `INSERT` that previously took ~50+ hours for 12.4M cells
- **Left incompatible columns** at source (XML docs, encrypted binaries, spatial geography) where they posed corruption risk and held no analytical value

**Business Gain**  
- 99.997% faster data loading (row-by-row INSERT took ~50+ hours for 12.4M cells; `LOAD DATA INFILE` integrated 2.4M cells in ~5 seconds)
- Complete migration without data loss on analytical columns

**Technical Gain**  
- Mastery of `INFORMATION_SCHEMA.COLUMNS` for cross-RDBMS type inspection
- Practical understanding of `DECIMAL(p,s)` precision vs. scale
- Competence with `utf8mb4` for international character support

---

## 3. Data Cleaning & Integrity

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
- **Conditional formatting** used as a visual debugger to catch misplaced cells

**Business Gain**  
Zero analytical columns discarded — every row that could hold insight was preserved and corrected.

**Technical Gain**  
- Developed "vertical partitioning" intuition naturally (later validated by senior practitioners)
- Learned that data cleaning is the dominant time sink in any analytics pipeline (80%+ of project effort)
- Built a reusable mental framework: inspect → hypothesise → test → correct → validate

---

## 4. Data Modeling (EER Diagram & Relationships)

**Problem**  
80+ tables across 5 business domains (Sales, Production, Purchasing, Human Resources, Person) with no formal relationships, no primary/foreign keys, orphan records, and incorrect data types embedded during import.

**Solution**  
- **Star-schema-per-domain** design — each domain got its own central fact table with surrounding dimension tables
- **Section-by-section execution:** Production → Sales → Purchasing → HR → Person
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
Relational querying across domains became possible — revenue per employee, product profitability by territory, churn-adjacent HR insights.

**Technical Gain**  
- Proficient with MySQL Workbench EER diagram sync (model → database and database → model)
- Understanding of indexing: `B-Tree` structure, clustered vs. non-clustered, `EXPLAIN` for query plan analysis
- Practical distinction between 1:1, 1:n, and many-to-many cardinalities and their enforcement via keys

---

## 5. Performance Optimization

**Problem**  
Cross-table joins on large datasets (e.g., 31K-row sales × 121K-row details) without indexes resulted in full table scans and slow query response.

**Solution**  
- **B-tree indexes** on frequently joined/queried columns:
  ```sql
  CREATE INDEX idx_sod_product ON SalesOrderDetails(ProductID);
  ```
- Targeted indexing on 5 core tables: `SalesOrderHeader`, `SalesOrderDetails`, `Product`, `Employee`, `Customer`
- Used `EXPLAIN SELECT` to verify index usage and identify bottlenecks

**Business Gain**  
Faster time-to-insight — analytical queries that previously took minutes returned in milliseconds.

**Technical Gain**  
- Indexes are a trade-off: faster `SELECT`, slower `UPDATE`/`INSERT`, extra disk space
- Secondary (non-clustered) indexes accelerate lookups on non-PK columns by leveraging the primary B-tree
- `SHOW INDEX FROM table` and `DROP INDEX` for lifecycle management

---

## 6. Cloud Backup & Security

**Problem**  
A local-only dataset is a single ransomware event away from total loss. Sensitive employee data (NationalIDNumber, LoginID, hashed passwords) required secure handling.

**Solution**  
- **Azure Blob Storage (Gen2, Cool Tier)** — project artefacts, cleaned CSVs, data model backups, and logs stored off-site
- **Access management:** RBAC ownership levels, SAS tokens, and connection strings for programmatic access
- **Sensitive data handling:**
  - Employee PII left flagged for secure-environment processing
  - Password columns (even if hashed) excluded from local MySQL
  - Recommendation documented: *"Do not store sensitive production data on local machines — use cloud-native or air-gapped environments"*
- **Backup discipline:** Every phase end (cleaning, modeling, relationship-building) triggered an Azure sync

**Business Gain**  
Full disaster recovery capability — any point in the project can be restored from cloud backup.

**Technical Gain**  
- Hands-on Azure RBAC (learned that `$logs` is a system container with immutable permissions)
- `azcopy` with `az login` vs. SAS-token-based auth
- Practical appreciation for infrastructure-as-code principles

---

## 7. Python EDA Integration

**Problem**  
Analysis needed to move from SQL queries to Python (pandas, matplotlib, seaborn) for richer statistical exploration. The connection code needed to be clean and straightforward for local EDA.

**Solution**  
- **Credentials:** Stored in `.env`, loaded via `python-dotenv` + `os`
- **Context-managed engine:**
  ```python
  with engine.connect() as connection:
      df = pd.read_sql_query("SELECT * FROM table", connection)
  ```
- **No unnecessary abstraction** — `text()` and parameterised placeholders were deliberately omitted. For local, manual EDA where you control every query, they add overhead without benefit. The same pattern (`text()` + `:params`) is however a valuable insight for backend design — it forces MySQL to treat string inputs as categorical data, preventing SQL injection in any user-facing application.
- **Driver insight:** Used `pymysql` with the `cryptography` package — an additional dependency learned in practice when the authentication handshake failed. Unlike Oracle's `mysql-connector-python` (which bundles it), PyMySQL is written from scratch and relies on `cryptography` for 1-way hashed authentication.
- **Special character encoding:** Manually encoded `@`, `#`, `$` in connection strings instead of relying on `quote_plus` — URL parsers are designed for web query strings, not DB credentials with complex character sets

**Business Gain**  
Clean, minimal EDA pipeline with no dead abstraction. The backend-security insight is documented and ready to apply when user-facing code is needed.

**Technical Gain**  
- SQLAlchemy `Engine` as a connection daemon vs. raw connections
- Knowing when *not* to over-engineer — `text()` + `params` is valuable for backend services, unnecessary for local EDA
- `pandas.read_sql_query` with pure SQL strings keeps the code simple when queries are trusted

---

## Project Summary

| Metric | Value |
|---|---|
| Source tables migrated | 80+ |
| Largest table (SalesOrderDetails) | 121K rows, 12.4M cells |
| Total cells across Address table | 2.4M cells |
| Business domains covered | 5 (Sales, Production, Purchasing, HR, Person) |
| Data cleaning hours (largest table) | ~7 hours |
| Integration speedup (after `LOAD DATA INFILE`) | 99.997% |
| Cloud backups triggered | At every phase completion |
| Security boundaries documented | PII, encrypted binaries, spatial data |

## Lessons From the Trenches

- **`LOAD DATA INFILE` is ~99.997% faster than row-by-row INSERT** — 50+ hours of integration dropped to seconds. Always research bulk-load options before starting any migration.
- **"If it works, don't touch it"** — hours were lost reverting and re-reverting driver settings on the Export Wizard. Debugging is valuable; over-debugging a working configuration is a timeline risk.
- **Azure `$logs` is a system container** — its permissions are immutable. Discovered after hours of RBAC troubleshooting. Read the docs before assuming you can modify privileged resources.
- **NULLABLE in Export Wizard means "column may be NULL" not "keep NULLs."** A single checkbox misinterpretation caused cascading export failures across the migration.
- **Backups saved the project twice** — a power shutdown during a 29-hour data integration would have been catastrophic without the Azure-reserved model file. Backup before any long-running operation.
- **Domain isolation prevents spaghetti models** — 80+ tables across 5 domains (Production, Sales, Purchasing, HR, Person) were modeled as independent star schemas, each with its own central fact table. Cross-domain connections were deliberately avoided because joining central tables across domains creates key conflicts and an unmaintainable EER diagram.
- **Vertical partitioning is learnable by doing** — splitting tables to resolve CSV export issues was later validated as a standard data modeling technique, discovered without formal training.
- **Manual data cleaning is the real time sink** (~80% of project effort). Automate early where possible, but manual inspection catches structural issues that scripts miss.
- **Data sovereignty matters** — avoiding third-party `.bak→CSV` converters meant slower progress but zero risk of data exfiltration. The same judgment applies in production.

---

**Key philosophy:** *"Normalising complexity into clarity."*

Every error was documented, every workaround tested, and every lesson learned is reflected in this repository — so that peers, collaborators, or future employers can see not just the output, but the engineering judgment behind it.

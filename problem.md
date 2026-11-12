# Data Quality Audit Lab — Flow & Implementation Analysis

This document outlines the issues, discrepancies, and edge cases identified during the review and execution of `R&D.md` (**Building Reliable Data Pipelines with Data Quality Validation**).

---

## 1. [Critical] Validity Check Logic Bug & Output Discrepancy

### Description
In **Chapter 5**, the Validity check logic for `customer_email` and other fields is implemented as:
```python
email_valid = df["customer_email"].astype("string").str.fullmatch(
    r"[^@\s]+@[^@\s]+\.[^@\s]+",
    na=False
)

status_valid = df["status"].isin(
    ["Pending", "Shipped", "Delivered"]
)

quantity_valid = df["quantity"].notna() & (df["quantity"] > 0)
price_valid = df["unit_price"].notna() & (df["unit_price"] >= 0)
amount_valid = df["total_amount"].notna() & (df["total_amount"] >= 0)

validity_issues = ~(
    email_valid
    & status_valid
    & quantity_valid
    & price_valid
    & amount_valid
    & order_dates.notna()
)
```

### The Problem
- In `sales.csv`, record `ORD003` has a missing customer email (`,,`).
- Passing `na=False` into `str.fullmatch` causes `email_valid` to evaluate to `False` for `ORD003`.
- In the negation `validity_issues = ~(...)`, `ORD003` is flagged as a **Validity issue**, in addition to already being flagged as a **Completeness issue**.
- Similarly, requiring `order_dates.notna()`, `df["quantity"].notna()`, etc. inside `validity_issues` means any missing field will fail Validity as well as Completeness.

### Actual Output vs. Documented Output
- **Actual execution result of Chapter 5 code**:
  ```text
  dimension     total_records  issue_count  quality_score
  Validity                 10            4          60.00
  ```
- **Documented Expected Output in Chapter 6**:
  ```text
  dimension     total_records  issue_count  quality_score
  Validity                 10            3          70.00
  ```
- **Documented Findings in Chapter 8**:
  Chapter 8 explicitly states:
  > **Validity** | Invalid email, zero quantity, or unknown status | **ORD006, ORD005, ORD009**
  
  `ORD003` is not supposed to be in the Validity findings; it belongs exclusively under Completeness.

### Conceptual Violation
Chapter 2 states:
> *"A dataset can pass one dimension and fail another... The six dimensions therefore check different aspects of quality and should be evaluated separately."*

Flagging missing values as invalid format violates the separation between Completeness and Validity.

### Recommended Fix
Allow `NaN` values to pass format validation (as nullness is already evaluated under Completeness):
```python
# 6. VALIDITY
email_valid = df["customer_email"].isna() | df["customer_email"].astype("string").str.fullmatch(
    r"[^@\s]+@[^@\s]+\.[^@\s]+",
    na=False
)

status_valid = df["status"].isna() | df["status"].isin(
    ["Pending", "Shipped", "Delivered"]
)

quantity_valid = df["quantity"].isna() | (df["quantity"] > 0)
price_valid = df["unit_price"].isna() | (df["unit_price"] >= 0)
amount_valid = df["total_amount"].isna() | (df["total_amount"] >= 0)
date_format_valid = df["order_date"].isna() | order_dates.notna()

validity_issues = ~(
    email_valid
    & status_valid
    & quantity_valid
    & price_valid
    & amount_valid
    & date_format_valid
)
```
Applying this fix produces exactly 3 issues (`ORD005`, `ORD006`, `ORD009`) and a score of `70.00%`, perfectly matching Chapter 6 and Chapter 8.

---

## 2. [Major] Hardcoded Future Dates & Lack of Determinism in Timeliness Check

### Description
In **Chapter 5**, Timeliness is evaluated using dynamic system time:
```python
today = pd.Timestamp.today().normalize()
oldest_allowed_date = today - pd.Timedelta(days=30)

timeliness_issues = (
    order_dates.isna()
    | (order_dates < oldest_allowed_date)
    | (order_dates > today)
)
```

### The Problem
- The dataset in Chapter 3 uses fixed hardcoded dates in October and November 2026:
  - `ORD001` - `ORD007`, `ORD009` - `ORD010`: `2026-11-01` to `2026-11-09`
  - `ORD008`: `2026-10-01`
- Chapter 6 notes: *"Expected output for the supplied sample dataset, when run on November 12, 2026"*.
- If a user or automated grading script runs this lab on any date other than ~November 10–30, 2026:
  - If run before November 2026: **all 10 records** will be flagged as future dates (`order_dates > today`).
  - If run after December 2026: **all 10 records** will be flagged as older than 30 days (`order_dates < oldest_allowed_date`).
  - The resulting score will be **0.00%** (10 issues) instead of **90.00%** (1 issue), breaking reproducibility and test automation.

### Recommended Fix
Either:
1. Use an explicit reference audit date for consistent evaluation:
   ```python
   # Use reference audit date (or pass as parameter)
   audit_date = pd.Timestamp("2026-11-12")
   oldest_allowed_date = audit_date - pd.Timedelta(days=30)
   timeliness_issues = (
       order_dates.isna()
       | (order_dates < oldest_allowed_date)
       | (order_dates > audit_date)
   )
   ```
2. Or generate dates in `sales.csv` dynamically relative to current date (e.g. `today - 5 days`, `today - 45 days`).

---

## 3. [Major] Shell & Platform Incompatibilities in Chapter 1 Setup

### Description
Chapter 1 lists the following commands:
```bash
mkdir -p datahive-quality-audit/{data,src,output}
cd datahive-quality-audit
python3 -m venv .venv
source .venv/bin/activate
```

### The Problem
1. **Brace Expansion (`{...}`)**:
   - `mkdir -p datahive-quality-audit/{data,src,output}` relies on Bash/Zsh brace expansion.
   - In Windows PowerShell, `mkdir` is an alias for `New-Item -ItemType Directory`. Running this creates a literal directory named `{data,src,output}` instead of three separate subdirectories.
   - In Command Prompt (`cmd.exe`) or strict POSIX `sh`, this command also fails.
2. **Unix-Specific Python Commands**:
   - `python3` is standard on Linux/macOS, but Windows typically uses `python`.
   - `source .venv/bin/activate` fails on Windows terminals (`.venv\Scripts\Activate.ps1` in PowerShell, or `.venv\Scripts\activate.bat` in CMD).
3. Even though the lab introduces VS Code Server (which often runs on Linux), students frequently clone and run labs locally on Windows or other shells.

### Recommended Fix
Provide cross-platform commands or specify standard syntax:
```bash
# Standard cross-platform commands
mkdir -p datahive-quality-audit/data datahive-quality-audit/src datahive-quality-audit/output
cd datahive-quality-audit

# Linux / macOS
python3 -m venv .venv
source .venv/bin/activate

# Windows (PowerShell)
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

---

## 4. [Moderate] Overwriting Flow in Chapter 4 vs. Chapter 5

### Description
- In **Chapter 4**, students create `src/quality_audit.py` with 25 lines of preliminary inspection code (`df.head()`, `df.info()`, `df.shape`, `df.isnull().sum()`) and execute it.
- In **Chapter 5**, students are instructed: *"Replace the code in `src/quality_audit.py` with the complete implementation below."*

### The Problem
- All code written in Chapter 4 is discarded and completely overwritten by Chapter 5.
- This creates an unnatural workflow where a script is created for one task, only to be erased in the very next step.

### Recommended Fix
- Either name the Chapter 4 script `src/inspect_data.py` (or keep it in a separate exploratory script / notebook),
- Or keep the inspection step as an introductory function `inspect_dataset(df)` inside `src/quality_audit.py` that is called prior to running the quality checks.

---

## 5. [Moderate] Pre-Coercion of Numeric Data Masks Validation Failures

### Description
In **Chapter 5**, data types are converted before quality checks are executed:
```python
for column in ["quantity", "unit_price", "total_amount"]:
    df[column] = pd.to_numeric(df[column], errors="coerce")
```

### The Problem
- If a record arrives with a malformed string value (e.g. `quantity = "two"` or `"N/A"`), `errors="coerce"` silently replaces it with `NaN`.
- In the next step, Completeness (`df[required_columns].isna().any(axis=1)`) flags the row as having a **missing** field.
- The root cause (invalid data type / format) is concealed and misclassified as an absence of data.

### Recommended Fix
Preserve raw values or check data types before coercing to distinguish between missing data (`Completeness`) and corrupt/unparseable data (`Validity`).

---

## 6. [Minor] Output Column Discrepancy & Number Formatting in Chapter 6

### Description
In **Chapter 5**, the report is printed via:
```python
print(report.to_string(index=False))
```
The DataFrame `report` includes 6 columns: `dimension`, `total_records`, `issue_count`, `quality_score`, `status`, `details`.

### The Problem
1. **Missing Columns in Documentation**: The expected output shown in Chapter 6 displays only 4 columns (`dimension`, `total_records`, `issue_count`, `quality_score`). The `status` and `details` columns are omitted without mentioning that columns were sliced.
2. **Float Formatting**: Chapter 6 displays formatted floats with two decimals (`90.00`, `80.00`, `70.00`). Default `pandas.DataFrame.to_string()` prints unformatted floats (`90.0`, `80.0`, `60.0`).

### Recommended Fix
Update the print statement in `quality_audit.py` to match the documented display:
```python
display_cols = ["dimension", "total_records", "issue_count", "quality_score"]
print(report[display_cols].to_string(index=False, float_format="%.2f"))
```
Or display all columns in Chapter 6.

---

## 7. [Minor] Placeholder Image Links Throughout the Document

### Description
Across `R&D.md`, 8 image references exist as dummy markdown links:
- Line 87: `![VS Code Server terminal showing...](url)`
- Line 104: `![Reference diagram showing accuracy...](url)`
- Line 128: `![VS Code Server editor displaying sales.csv...](url)`
- Line 172: `![VS Code Server terminal displaying the dataset preview...](url)`
- Line 344: `![VS Code Server editor displaying quality_audit.py...](url)`
- Line 372: `![VS Code Server terminal showing the completed audit report...](url)`
- Line 390: `![VS Code Server editor displaying the generated output/quality_report.csv...](url)`
- Line 407: `![VS Code Server showing sales.csv and quality_report.csv...](url)`

### Impact
None of the referenced images or diagrams will render in markdown readers or web view.

---

## 8. [Minor] Title-Casing in Consistency Check Is Fragile for Real Data

### Description
In Chapter 5:
```python
normalized_product = df["product"].astype("string").str.strip().str.title()
```

### The Problem
While `.str.title()` works for `"laptop"` vs `"Laptop"`, it is inappropriate for retail/product datasets containing acronyms, product codes, or non-title casing (e.g., `"USB Cable"` becomes `"Usb Cable"`, `"iPhone"` becomes `"Iphone"`, `"RAM"` becomes `"Ram"`).

### Recommended Fix
Note in the text that in production environments, consistency is verified against a canonical product taxonomy or lookup table rather than simple `.str.title()`.

---

## 9. [Minor] Skipped Order ID in Sample Dataset

### Description
In `data/sales.csv`, the `order_id` values are:
`ORD001`, `ORD002`, `ORD003`, `ORD004`, `ORD005`, `ORD006`, `ORD006`, `ORD008`, `ORD009`, `ORD010`.

### Note
`ORD007` is skipped (going straight from the duplicate `ORD006` to `ORD008`). While not breaking, it is worth noting in Chapter 3 so students do not suspect a missing record.

---

## Summary Table of Issues

| # | Severity | Category | Issue Description |
|---|---|---|---|
| 1 | **Critical** | Code Bug & Logic | Validity check flags null email (`ORD003`), yielding 4 issues (60.00%) instead of expected 3 issues (70.00%). Conflates Completeness with Validity. |
| 2 | **Major** | Reproducibility | Timeliness check uses dynamic `Timestamp.today()`, breaking entirely when run outside Nov 2026 (score drops from 90% to 0%). |
| 3 | **Major** | Portability | `mkdir` brace expansion and `source` / `python3` fail on Windows and non-Bash shells. |
| 4 | **Moderate** | Workflow / UX | Chapter 4 inspection script is immediately erased and replaced in Chapter 5. |
| 5 | **Moderate** | Pipeline Design | `pd.to_numeric(errors="coerce")` converts invalid strings into `NaN`, causing Validity errors to be misreported as Completeness errors. |
| 6 | **Minor** | Documentation | Terminal output displays 6 columns, while Chapter 6 only shows 4; float formatting differs (`90.0` vs `90.00`). |
| 7 | **Minor** | Content | All 8 diagram and screenshot markdown links use `(url)` placeholder links. |
| 8 | **Minor** | Robustness | `.str.title()` is fragile for products with acronyms or camelCase names. |
| 9 | **Minor** | Data Design | `ORD007` is skipped in the sample dataset. |

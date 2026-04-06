# Complete Beginner's Guide to Data Quality in IICS

> **IICS Service:** Data Quality
> **Difficulty:** Beginner
> **Estimated Reading Time:** 30 minutes
> **Prerequisites:** An IICS organization account (free trial is sufficient), basic understanding of what a database table is, familiarity with common data issues (missing values, duplicates, inconsistent formatting)
> **Learning Objectives:**
> - Explain what the IICS Data Quality service is and why data quality matters for business decisions
> - Run a Data Profiling job to discover the shape, completeness, and patterns in your data
> - Create Data Quality Rules to codify business expectations (e.g., "email must contain @", "age must be between 0 and 120")
> - Build a Scorecard to monitor data quality trends over time
> - Apply Cleansing transformations to standardize and fix common data issues
> - Interpret profiling results and scorecard metrics to communicate data quality to stakeholders

---

## 🧭 Why Data Quality Matters — The Business Case

Before diving into the IICS tools, it's important to understand WHY data quality exists as a dedicated discipline.

**Data Quality** (the measure of how accurate, complete, consistent, timely, and valid data is for its intended use) directly impacts business decisions. Poor data quality leads to:

| Business Impact | Example |
|----------------|---------|
| **Wrong decisions** | A sales report double-counts revenue because of duplicate customer records |
| **Wasted resources** | Marketing sends 10,000 emails to invalid addresses, paying for each send |
| **Compliance risk** | Customer addresses are incomplete, violating regulatory reporting requirements |
| **Lost trust** | An executive notices that two dashboards show different numbers for the same metric |

The IICS **Data Quality service** provides tools to **discover** problems (profiling), **define** expectations (rules), **measure** health (scorecards), and **fix** issues (cleansing).

---

## 🔍 Deep Dive: Data Profiling — Understanding Your Data

**Data Profiling** (the process of examining a dataset to discover its structure, content patterns, completeness, and anomalies without modifying the data) is always the first step in any data quality initiative. You cannot fix what you haven't measured.

### What Profiling Reveals

Think of profiling as a medical check-up for your data. Just as a doctor checks vital signs before prescribing treatment, profiling checks data vitals before you write rules or cleansing logic.

| Profiling Analysis | What It Shows You | Business Value |
|-------------------|------------------|----------------|
| **Column Statistics** | Min, max, average, null count, distinct count for each column | Quickly spots columns with too many nulls or unexpected ranges |
| **Pattern Analysis** | Common formats in a column (e.g., phone numbers that follow "XXX-XXX-XXXX" vs "XXXXXXXXXX") | Reveals inconsistent data entry across systems |
| **Frequency Distribution** | Most and least common values in a column | Identifies dominant categories, rare outliers, or possible default/dummy values |
| **Data Type Inference** | What data type the values actually represent (even if stored differently) | Catches columns stored as VARCHAR that actually contain numbers or dates |
| **Uniqueness Analysis** | How unique the values are in a column or combination of columns | Validates whether supposed primary keys are truly unique |
| **Null Analysis** | Percentage of rows with missing values per column | Highlights incomplete data that may break downstream processes |

### Step-by-Step: Running a Data Profile

1. From the IICS home page, click **Data Quality** in the left navigation [VERIFY: confirm exact navigation — may be under "Data Integration" → profiling options in some IICS versions]
2. Click **New** → **Profile**
3. Name it `profile_customer_data`
4. Under **Source**, select your **Connection** (e.g., a Snowflake or Oracle connection)
5. Select the **Source Object** (e.g., `CUSTOMERS` table)
6. Choose which columns to profile:
   - **All Columns** (recommended for the first run)
   - Or select specific columns if you know what to focus on
7. Configure profiling options:
   - **Row Sampling**: For very large tables, you can profile a sample instead of the full table (e.g., 10% or 100,000 rows). Sampling is faster but may miss rare anomalies
   - **Analysis Types**: Select which analyses to include (column stats, patterns, frequency, etc.)
8. Under **Runtime Environment**, select your Secure Agent
9. Click **Save** → **Run**

### Interpreting Profiling Results

After the profile completes, IICS displays a visual results dashboard. Here's how to read the key metrics:

**Example: Profiling a `CUSTOMERS` table**

| Column | Null % | Distinct Count | Top Pattern | Observation |
|--------|--------|---------------|-------------|-------------|
| `customer_id` | 0% | 50,000 | NNNNN | ✅ Clean — fully populated, unique |
| `email` | 12% | 44,000 | AAA@AAA.AAA | ⚠️ 12% missing emails — investigate |
| `phone` | 5% | 47,500 | Multiple patterns | ⚠️ Inconsistent formats — needs cleansing |
| `country` | 0% | 28 | AAA | ✅ Fully populated, reasonable cardinality |
| `created_date` | 0% | 3,200 | NNNN-NN-NN | ✅ Consistent date format |
| `status` | 0% | 5 | AAAA | ✅ Limited values — good candidate for a rule |

> 💡 **What Does "Pattern" Mean?**
> In profiling, a **pattern** replaces each letter with `A`, each digit with `N`, and keeps special characters. So `john@acme.com` becomes `AAAA@AAAA.AAA`. This lets you see at a glance whether data follows a consistent format without examining every individual value.

---

## 📏 Deep Dive: Data Quality Rules — Codifying Your Expectations

A **Data Quality Rule** (a formal, reusable definition of what "good data" looks like for a specific column or set of columns, expressed as a logical condition) is the bridge between "we know this data has problems" and "we can automatically detect and measure those problems."

### Types of Rules

| Rule Type | What It Checks | Example |
|----------|---------------|---------|
| **Completeness** | Is the value present (not null, not blank)? | `email IS NOT NULL AND email != ''` |
| **Validity** | Does the value conform to an expected format or range? | `email LIKE '%@%.%'` |
| **Consistency** | Do related values agree with each other? | `IF country = 'US' THEN zip_code LIKE 'NNNNN'` |
| **Uniqueness** | Is the value unique across the dataset? | `customer_id has no duplicates` |
| **Referential Integrity** | Does the value exist in a reference dataset? | `country_code exists in the ISO_COUNTRIES reference table` |

### Step-by-Step: Creating a Data Quality Rule

**Scenario:** You want to create a rule that validates that all customer email addresses contain an `@` symbol and at least one `.` after the `@`.

1. Navigate to **Data Quality** → **New** → **Rule** [VERIFY: confirm exact menu path for creating rules — may be under "Rule Specifications" in some versions]
2. Name it `rule_valid_email_format`
3. Define the **Rule Logic**:
   - **Input**: A column named `email` (String)
   - **Condition**: `email LIKE '%@%.%'`
   - **Output**: A Boolean result — `PASS` if the condition is true, `FAIL` if false
4. (Optional) Add a **Severity Level**: Critical, Major, or Minor — this helps prioritize fixes
5. (Optional) Add a **Description**: "Validates that the email field contains a properly formatted email address with an @ symbol and a domain with at least one period"
6. Click **Save**

### Composing Multiple Rules

In practice, a single column may need multiple rules. For the `email` column:

| Rule Name | Condition | Severity |
|----------|-----------|----------|
| `rule_email_not_null` | `email IS NOT NULL` | Critical |
| `rule_valid_email_format` | `email LIKE '%@%.%'` | Major |
| `rule_email_no_spaces` | `email NOT LIKE '% %'` | Minor |

Each rule runs independently, and the results aggregate into a quality score for the column.

> 💡 **Rules Are Reusable**
> A Data Quality Rule defined once can be applied to ANY dataset that has a column matching the input definition. Your `rule_valid_email_format` works on the `CUSTOMERS` table, the `EMPLOYEES` table, and any future table with an `email` column — no need to recreate it.

---

## 📊 Deep Dive: Scorecards — Measuring and Tracking Quality

A **Scorecard** (a dashboard-style report in IICS that aggregates the results of Data Quality Rules across one or more datasets and displays them as metrics, trends, and drill-down views) answers the executive question: "How good is our data, and is it getting better or worse?"

### What a Scorecard Shows

| Metric | Description | Example |
|--------|------------|---------|
| **Overall Score** | The percentage of rows that pass all applicable rules | 87.3% — meaning 87.3% of records have no quality issues |
| **Score by Rule** | Pass rate for each individual rule | "Email not null" = 88%, "Valid email format" = 92% |
| **Score by Column** | Aggregated quality score for each column | `email` column = 84%, `phone` column = 91% |
| **Trend Over Time** | How the scores change across multiple runs | Score improved from 82% → 87% over the last 4 weeks |
| **Drill-Down** | The actual failing rows and values | Click the 12% failure rate on `email` to see the specific rows with bad emails |

### Step-by-Step: Creating a Scorecard

1. Navigate to **Data Quality** → **New** → **Scorecard** [VERIFY: confirm exact menu path for creating Scorecards]
2. Name it `scorecard_customer_data_quality`
3. **Add Data Source**: Select your `CUSTOMERS` table (connection + object)
4. **Add Rules**: Select the rules to evaluate — e.g., `rule_email_not_null`, `rule_valid_email_format`, `rule_email_no_spaces`
5. **Map Rules to Columns**: Associate each rule with the column it should evaluate (e.g., all three email rules → `email` column)
6. Configure **Scoring Method**:
   - **Row-Level**: A row passes only if ALL rules pass for that row
   - **Column-Level**: Each column gets an independent score
7. Click **Save** → **Run**

### Reading the Scorecard Dashboard

After the scorecard runs, the results dashboard presents:

```
┌─────────────────────────────────────────────────┐
│  Customer Data Quality Scorecard                 │
│  Overall Score: 87.3%  ▲ +2.1% from last run    │
│                                                   │
│  ┌─────────────┬──────────┬────────────┐         │
│  │ Rule         │ Pass Rate │ Trend     │         │
│  ├─────────────┼──────────┼────────────┤         │
│  │ Email Not Null│  88.0%  │ ▲ +1.2%   │         │
│  │ Valid Format  │  92.3%  │ ── Same    │         │
│  │ No Spaces     │  99.1%  │ ▲ +0.5%   │         │
│  └─────────────┴──────────┴────────────┘         │
│                                                   │
│  [Click any rule to drill into failing rows]     │
└─────────────────────────────────────────────────┘
```

> 💡 **Schedule Scorecards Regularly**
> The real value of a Scorecard is the **trend**. A one-time score tells you the current state; a weekly score tells you whether your data quality efforts are working. Schedule the Scorecard to run on the same cadence as your data pipelines (e.g., daily or weekly).

---

## 🧹 Deep Dive: Cleansing — Fixing Data Problems

**Cleansing** (the process of detecting and correcting — or removing — inaccurate, incomplete, or improperly formatted data according to defined rules and standards) is where you move from measuring problems to solving them.

IICS provides cleansing capabilities through transformations that can be used inside CDI Mappings. This means cleansing is integrated into your data pipelines — not a separate, disconnected step.

### Common Cleansing Operations

| Operation | What It Does | Example |
|----------|-------------|---------|
| **Standardization** | Converts data to a consistent format | `"new york"`, `"NEW YORK"`, `"New york"` → `"New York"` |
| **Parsing** | Splits a single field into structured parts | `"John Smith"` → `first_name: "John"`, `last_name: "Smith"` |
| **Validation** | Checks values against reference data | Validates that `"US"` is a real country code from an ISO standard list |
| **Deduplication** | Identifies and merges duplicate records | Two records for `"John Smith, 123 Main St"` and `"J. Smith, 123 Main Street"` are flagged as likely duplicates |
| **Enrichment** | Adds missing information from external sources | Given a zip code `"10001"`, adds `city: "New York"`, `state: "NY"` |

### Using Cleansing in a CDI Mapping

Cleansing transformations are available in the Mapping Designer as part of CDI. Here's how to add basic cleansing to a data pipeline:

1. Open your CDI **Mapping** in the Mapping Designer (or create a new one)
2. After your Source transformation, add a **Data Quality** transformation from the palette [VERIFY: confirm exact transformation name — may be called "Cleanse" or "Data Cleansing" transformation]
3. Configure the transformation:
   - **Standardization**: Select the columns you want to standardize (e.g., `country_name`) and the reference dictionary to use (e.g., ISO country names)
   - **Case Conversion**: Set `UPPER`, `LOWER`, or `PROPER` case for text fields
   - **Trimming**: Remove leading/trailing whitespace from string fields
4. Connect the transformation output to your Target or to additional transformations
5. **Validate** and **Save** the Mapping

### A Practical Cleansing Pipeline

```
Source (Raw CUSTOMERS)
    │
    ▼
Expression (Trim whitespace from all string columns)
    │
    ▼
Data Quality Transformation (Standardize country names, parse full names)
    │
    ▼
Filter (Remove rows that fail critical quality rules)
    │
    ▼
Target (Clean CUSTOMERS table)
```

---

## 🔗 How Profiling, Rules, Scorecards, and Cleansing Work Together

These four capabilities form a continuous cycle:

```
     ┌──────────┐
     │ PROFILE  │  ← Step 1: Discover what's in your data
     └────┬─────┘
          │
          ▼
     ┌──────────┐
     │  RULES   │  ← Step 2: Define what "good" looks like
     └────┬─────┘
          │
          ▼
     ┌──────────┐
     │SCORECARD │  ← Step 3: Measure how good the data is
     └────┬─────┘
          │
          ▼
     ┌──────────┐
     │ CLEANSE  │  ← Step 4: Fix the problems
     └────┬─────┘
          │
          ▼
     ┌──────────┐
     │ PROFILE  │  ← Step 5: Re-measure to confirm improvement
     └──────────┘    (cycle repeats)
```

This cycle is not a one-time project — it's an ongoing practice. As new data sources are added and business requirements change, you continuously profile, define rules, measure, cleanse, and re-measure.

---

## ⚠️ Common Mistakes & Troubleshooting

| Mistake | Why It Happens | How to Fix It |
|---------|---------------|---------------|
| Profiling takes hours on a large table | The table has millions of rows and profiling is running against the full dataset | Enable **row sampling** — profiling 10% of rows is usually sufficient to identify patterns and issues |
| Rules all show 100% pass rate but data is clearly bad | The rules are too lenient or check the wrong condition (e.g., checking for NULL but not for empty strings) | Review rule logic. Common miss: `email IS NOT NULL` passes for empty string `''`. Change to `email IS NOT NULL AND TRIM(email) != ''` |
| Scorecard shows different results than a manual SQL count | The Scorecard may use sampling, or the rule condition doesn't exactly match the SQL WHERE clause | Align the rule condition precisely with the SQL logic. Disable sampling if exact counts are needed |
| Cleansing transformation changes data unexpectedly | The standardization dictionary mapped a value to something unexpected (e.g., the abbreviation "MA" was mapped to "Massachusetts" in a column that stores country codes) | Review the dictionary/reference data being used. Ensure the right reference dataset is selected for each column's context |
| Profiling results show many "unknown" patterns | The column contains unstructured or freeform text (like comments or descriptions) that doesn't follow any pattern | This is expected for freeform text fields. Focus profiling and rules on structured columns (IDs, dates, codes, emails) |

---

## 📝 Key Takeaways

- **Data quality is a cycle, not a one-time fix** — profiling, rules, scorecards, and cleansing form a continuous improvement loop
- **Always profile BEFORE writing rules** — profiling reveals the actual state of your data so you can write targeted, relevant rules instead of guessing
- **Rules are reusable across datasets** — define once, apply to any table with matching columns
- **Scorecards make quality visible to stakeholders** — a 92% score with a trend chart is more persuasive than "we think the data is mostly good"
- **Cleansing happens INSIDE your data pipeline** — it's integrated into CDI Mappings, not a disconnected manual process

---

## 🧪 Practice Exercise

**Scenario:** You have a `SUPPLIERS` table with columns: `supplier_id`, `company_name`, `email`, `phone`, `country`, `rating` (1–5). You suspect the data has quality issues and need to investigate, measure, and report.

**Your Task:**
1. **Profile** the `SUPPLIERS` table:
   - Create a new Profile, select all columns, run it
   - Examine the results: What percentage of `email` values are null? What patterns exist in `phone`? What distinct values exist in `country`?
2. **Create three Data Quality Rules**:
   - `rule_supplier_email_required`: `email IS NOT NULL AND TRIM(email) != ''`
   - `rule_supplier_rating_valid`: `rating >= 1 AND rating <= 5`
   - `rule_supplier_country_not_blank`: `country IS NOT NULL AND TRIM(country) != ''`
3. **Create a Scorecard** named `scorecard_supplier_quality`:
   - Add the `SUPPLIERS` table as the data source
   - Associate the three rules with their respective columns
   - Run the Scorecard and note the overall score
4. **Interpret the results**: Write down (on paper or in a text file) which rule has the lowest pass rate and what action you would recommend to fix the underlying data issue

**Expected Outcome:** The Scorecard runs successfully and displays a score for each rule. You can identify the weakest rule and articulate a fix — for example, "The email rule has a 78% pass rate — we need to require email at the point of supplier registration."

---

## 📚 Glossary

| Term | Definition |
|------|-----------|
| **Data Quality** | The measure of how accurate, complete, consistent, timely, and valid data is for its intended use |
| **Data Profiling** | The process of examining a dataset to discover its structure, content patterns, completeness, and anomalies |
| **Data Quality Rule** | A formal, reusable definition of what "good data" looks like, expressed as a logical condition |
| **Scorecard** | A dashboard-style report that aggregates rule results across datasets and displays quality metrics and trends |
| **Cleansing** | The process of detecting and correcting inaccurate, incomplete, or improperly formatted data |
| **Standardization** | A cleansing operation that converts data values to a consistent format (e.g., normalizing country names) |
| **Parsing** | A cleansing operation that splits a single field into structured component parts |
| **Deduplication** | The process of identifying and merging duplicate records in a dataset |
| **Pattern Analysis** | A profiling technique that replaces letters with A, digits with N, and keeps special characters to reveal formatting consistency |
| **Frequency Distribution** | A profiling metric showing the most and least common values in a column |
| **Row Sampling** | A technique that profiles a subset of rows instead of the full table to reduce processing time |
| **Completeness** | A data quality dimension measuring whether required values are present (not null or blank) |
| **Validity** | A data quality dimension measuring whether values conform to expected formats, ranges, or reference data |
| **Referential Integrity** | A data quality dimension measuring whether values in one dataset exist in a related reference dataset |

---

## 🔗 What to Learn Next

- **Advanced Data Quality Rules:** Building complex rules with cross-column logic, lookup-based validation, and custom scoring formulas
- **Integrating Data Quality into CDI Pipelines:** Embedding profiling and cleansing transformations directly into your Mappings so data is validated and fixed as it flows
- **Data Quality Governance and Reporting:** Setting up automated scorecard runs, email alerts for quality drops, and executive dashboards for data stewardship

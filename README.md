# 🏦 Aurora Bank: Customer Risk & Spend Intelligence

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Pandas](https://img.shields.io/badge/Pandas-150458?style=for-the-badge&logo=pandas&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)

---

## 🔗 Live Dashboard

👉 [View Live Power BI Dashboard]([https://app.powerbi.com/view?r=eyJrIjoiYWQzNzExMWYtMzVlNy00NzBiLWFhNjUtMTEyYzM2YTQ1YzJhIiwidCI6IjQyZjRkOWVkLTBiYmItNGU5NS1hYmRjLTM5OGM1M2QzNjkxZCJ9](https://app.powerbi.com/view?r=eyJrIjoiYWQzNzExMWYtMzVlNy00NzBiLWFhNjUtMTEyYzM2YTQ1YzJhIiwidCI6IjQyZjRkOWVkLTBiYmItNGU5NS1hYmRjLTM5OGM1M2QzNjkxZCJ9&pageName=9e32f33b4754dab4d229))

---

## 📌 Project Overview

Aurora Bank's Risk & Customer Analytics team needed a unified view of customer risk — combining **who their customers are** (demographics, financial health, card ownership) with **how they behave** (spending patterns, transaction errors, high-value activity).

Rising personal debt levels increase exposure to default, yet the bank had no systematic way to flag at-risk customers before problems surface. This project builds a **two-pillar analytical solution** that enables the Head of Risk & Customer Analytics to:

- Segment customers by financial risk tier based on Debt-to-Income ratio and credit score
- Identify spending behaviors and transaction error patterns that validate and explain those risk tiers
- Guide proactive intervention (credit limit reviews, targeted outreach, product recommendations)

---

## 🎯 Business Objective

> **Give Aurora Bank a single dashboard to understand customer financial risk and the transaction behavior that confirms it — enabling data-driven decisions before risk turns into default or churn.**

| Pillar | Focus | Business Question |
|---|---|---|
| A — Customer Profiling & Segmentation | Demographics + Financial Health | Who are our customers and how risky are they? |
| B — Transaction & Spend Intelligence | Spending Behavior + Error Analysis | How do customers spend, and where does risk show up in behavior? |

---

## 📊 Dashboard Pages

| Page | Objective |
|---|---|
| Demographics | Customer profile by age, gender, geography, and income bands |
| Financial Health | DTI ratio, credit score distribution, debt analysis by risk tier |
| Transactions | Spend by MCC category, transaction channel, monthly trends |
| Card Details | Credit limit, card brand distribution, failed transactions by card |

**Row-Level Security (RLS)** is implemented across all pages with three roles:

| Role | Access |
|---|---|
| Executive | All customers — full portfolio view |
| Risk Analyst | High-risk customers only |
| Relationship Manager | Low and Medium risk customers only |

---

## 🗃️ Data Sources

Four tables from a synthetic banking dataset (`.xlsx` format, two sheets each — data + description):

| Table | Rows | Key Fields |
|---|---|---|
| `users_data` | 2,000 | Demographics, income, debt, credit score |
| `cards_data` | 6,146 | Card brand, type, credit limit, expiry |
| `transactions_data` | 157,224 | Amount, merchant, MCC, errors, channel |
| `mcc_codes` | 109 | Merchant Category Code descriptions |

**Data period:** January 2022 – December 2024 (3 years)

**Referential integrity:** Validated — all `client_id` values in `cards_data` and `transactions_data` exist in `users_data`. All `mcc` values in `transactions_data` exist in `mcc_codes`.

---

## 🛠️ Tools & Stack

| Tool | Purpose |
|---|---|
| **Python (Pandas)** | Data cleaning, type conversion, anomaly investigation |
| **Power BI (DAX + Power Query)** | Data modelling, calculated columns, KPI measures, RLS, dashboard |

---

## 🔧 Data Cleaning & Transformation

### Step 1 — Currency String to Numeric (Python)

`yearly_income`, `total_debt`, `per_capita_income` (users table) and `credit_limit` (cards table) were stored as strings with `$` prefix (e.g., `"$44,128"`). These were stripped and cast to `float64`:

```python
for col in ['yearly_income', 'total_debt', 'per_capita_income']:
    users_df[col] = users_df[col].replace('[\$,]', '', regex=True).astype(float)

cards_df['credit_limit'] = cards_df['credit_limit'].replace('[\$,]', '', regex=True).astype(float)
```

**Business reason:** All downstream KPIs (DTI ratio, avg income by tier, credit limit utilization) require numeric computation. Leaving these as strings means no calculation is possible.

---

### Step 2 — Latitude Corruption Investigation (Python)

Latitude was loaded as `object` dtype instead of `float64`. Investigation revealed 45 records contained date strings (e.g., `"2024-01-29"`) instead of decimal coordinates:

```python
# Identify corrupted rows before any conversion
bad_lat = users_df[pd.to_numeric(users_df['latitude'], errors='coerce').isna()]
print(len(bad_lat))   # 45 rows confirmed
print(bad_lat['latitude'].head())
```

**Root cause:** Excel auto-formatted decimal latitude values with two decimal places (e.g., `29.01`) as calendar dates ("29-Jan") during the original file creation — a known Excel date auto-formatting trap.

**Resolution:** Latitude values for 45 affected rows were nulled. Longitude was confirmed unaffected on all 45 rows. These customers are excluded from geographic mapping only — all demographic and financial fields remain valid and are retained for segmentation.

```python
users_df['latitude'] = pd.to_numeric(users_df['latitude'], errors='coerce')
users_df['longitude'] = pd.to_numeric(users_df['longitude'], errors='coerce')
```

---

### Step 3 — Missing Values: Structural vs. Data Quality

| Column | Missing Count | Root Cause | Action |
|---|---|---|---|
| `merchant_state` | 17,921 | Online transactions (no physical location) | Left as null — structural, not a quality issue |
| `zip` | 19,106 | Online transactions | Left as null — structural |
| `errors` | 154,486 | Transaction succeeded — null = no error | Left as null — confirmed by cross-check |
| `latitude` | 45 (post-clean) | Excel date auto-format corruption | Nulled — see Step 2 |

**Validation for online transaction nulls:**

```python
# Confirmed: Online Transaction = 100% missing state, Swipe/Chip = ~0% missing
print(tx.groupby('use_chip')['merchant_state'].apply(lambda x: x.isnull().mean()))
```

Output:
```
use_chip
Chip Transaction      0.001231
Online Transaction    1.000000
Swipe Transaction     0.000000
```

**Business reason:** Filling these nulls with a placeholder would fabricate merchant location data that never existed. Documenting them as structural preserves analytical integrity.

---

### Step 4 — Error Field: Multi-Label Structure

The `errors` column is comma-separated and multi-label — a single transaction can carry more than one error simultaneously (e.g., `"Bad CVV, Insufficient Balance"`). Distribution:

| Error Type | Count |
|---|---|
| Insufficient Balance | 1,760 |
| Bad PIN | 388 |
| Technical Glitch | 324 |
| Bad Card Number | 93 |
| Bad Expiration | 73 |
| Bad CVV | 65 |
| Bad Zipcode | 17 |
| Combined (multi-label) | 13 |

Total error transactions: **2,738 out of 157,224 (1.74%)**

**Note:** `errors` being null means the transaction succeeded — null is not missing data, it is the "no error" state.

---

### Step 5 — MCC Category Mapping (Power Query M)

109 raw Merchant Category Codes were too granular for dashboard filtering. A `Category` column was added to `mcc_codes` using a Power Query custom column, mapping all 109 descriptions into 14 business-readable categories:

```
if Text.Contains([Description], "Eating Places") or Text.Contains([Description], "Restaurants") or Text.Contains([Description], "Fast Food") then "Food"
else if Text.Contains([Description], "Service Stations") or Text.Contains([Description], "Gas") or Text.Contains([Description], "Tolls") or Text.Contains([Description], "Car Washes") or Text.Contains([Description], "Automotive") or Text.Contains([Description], "Truck") then "Automotive"
else if Text.Contains([Description], "Amusement") or Text.Contains([Description], "Carnivals") or Text.Contains([Description], "Theaters") or Text.Contains([Description], "Sports") or Text.Contains([Description], "Recreational") or Text.Contains([Description], "Clubs") then "Entertainment"
else if Text.Contains([Description], "Supermarkets") or Text.Contains([Description], "Grocery") or Text.Contains([Description], "Food Stores") or Text.Contains([Description], "Wholesale") or Text.Contains([Description], "Discount Stores") or Text.Contains([Description], "Department Stores") then "Retail"
else if Text.Contains([Description], "Utilities") or Text.Contains([Description], "Electric") or Text.Contains([Description], "Gas") or Text.Contains([Description], "Water") or Text.Contains([Description], "Sanitary") or Text.Contains([Description], "Telecommunication") or Text.Contains([Description], "Cable") or Text.Contains([Description], "Satellite") then "Utilities"
else if Text.Contains([Description], "Doctors") or Text.Contains([Description], "Physicians") or Text.Contains([Description], "Hospitals") or Text.Contains([Description], "Dentists") or Text.Contains([Description], "Pharmacies") or Text.Contains([Description], "Chiropractors") or Text.Contains([Description], "Medical") or Text.Contains([Description], "Podiatrists") then "Healthcare"
else if Text.Contains([Description], "Book Stores") or Text.Contains([Description], "Books") or Text.Contains([Description], "Periodicals") or Text.Contains([Description], "Music") or Text.Contains([Description], "Digital Goods") then "Media"
else if Text.Contains([Description], "Cleaning") or Text.Contains([Description], "Maintenance") or Text.Contains([Description], "Laundry") or Text.Contains([Description], "Tax Preparation") or Text.Contains([Description], "Legal Services") or Text.Contains([Description], "Accounting") or Text.Contains([Description], "Auditing") or Text.Contains([Description], "Insurance") or Text.Contains([Description], "Travel Agencies") or Text.Contains([Description], "Security Services") or Text.Contains([Description], "Detective") then "Services"
else if Text.Contains([Description], "Florists") or Text.Contains([Description], "Gardening") or Text.Contains([Description], "Nursery Stock") or Text.Contains([Description], "Flowers") then "Floristry"
else if Text.Contains([Description], "Building Materials") or Text.Contains([Description], "Lumber") or Text.Contains([Description], "Hardware Stores") or Text.Contains([Description], "Lighting") or Text.Contains([Description], "Fixtures") or Text.Contains([Description], "Floor Covering") or Text.Contains([Description], "Home Furnishing") or Text.Contains([Description], "Drapery") then "Home"
else if Text.Contains([Description], "Steel") or Text.Contains([Description], "Metal") or Text.Contains([Description], "Machinery") or Text.Contains([Description], "Fabricated Products") or Text.Contains([Description], "Welding") or Text.Contains([Description], "Ironwork") or Text.Contains([Description], "Semiconductors") then "Manufacturing"
else if Text.Contains([Description], "Airlines") or Text.Contains([Description], "Passenger Railways") or Text.Contains([Description], "Bus Lines") or Text.Contains([Description], "Railroad") or Text.Contains([Description], "Motor Freight") or Text.Contains([Description], "Ship Chandlers") or Text.Contains([Description], "Postal Services") or Text.Contains([Description], "Cruise Lines") or Text.Contains([Description], "Towing Services") then "Transport"
else if Text.Contains([Description], "Beauty") or Text.Contains([Description], "Barber") or Text.Contains([Description], "Cosmetic") or Text.Contains([Description], "Women's Ready-To-Wear") or Text.Contains([Description], "Clothing") or Text.Contains([Description], "Shoe Stores") or Text.Contains([Description], "Sports Apparel") or Text.Contains([Description], "Leather Goods") then "Apparel"
else "Miscellaneous"
```

| Category | Example Descriptions |
|---|---|
| Food | Eating Places, Fast Food Restaurants, Restaurants |
| Automotive | Service Stations, Car Washes, Auto Body Repair |
| Entertainment | Theaters, Amusement Parks, Recreational Sports |
| Retail | Grocery Stores, Department Stores, Discount Stores |
| Utilities | Electric/Gas/Water, Telecommunication, Cable |
| Healthcare | Doctors, Hospitals, Pharmacies, Dentists |
| Media | Book Stores, Digital Goods, Music Stores |
| Services | Legal, Accounting, Insurance, Travel Agencies |
| Floristry | Florists, Gardening Supplies, Nursery Stock |
| Home | Lumber, Hardware Stores, Home Furnishings |
| Manufacturing | Steel, Metalwork, Machinery, Semiconductors |
| Transport | Airlines, Railways, Bus Lines, Cruise Lines |
| Apparel | Clothing, Shoe Stores, Beauty, Cosmetics |
| Miscellaneous | All remaining codes |

---

### Step 6 — Power Query Transformations (Power BI)

| Table | Transformation |
|---|---|
| `cards_data` | `expires` and `acct_open_date` converted to Date type |
| `transactions_data` | `date` column split into separate `Date` and `Time` columns |

**Note on `expires`:** Card expiry is month/year only by nature — no real day component exists. Power BI assigns the 1st of the month when converting to Date type. This is acceptable for expiry-window analysis but `expires` should not be used for day-level calculations.

---

## 📐 Data Model (Star Schema)

```
                    ┌─────────────┐
                    │  MccData    │
                    │  (1 side)   │
                    └──────┬──────┘
                           │ mcc_id → mcc
                           │
┌─────────────┐    ┌───────┴──────────┐    ┌─────────────┐
│  UsersData  │    │ TransactionData  │    │  CardsData  │
│  (1 side)   ├───►│   (Fact Table)   │◄───┤  (1 side)   │
│             │    │   157,224 rows   │    │             │
└─────────────┘    └──────────────────┘    └─────────────┘

```

- `TransactionData` is the **fact table** (events, measures)
- `UsersData`, `CardsData`, `MccData` are **dimension tables** (context)
- All relationships: One-to-Many, Single cross-filter direction
- RLS defined on `UsersData` — cascades to all connected tables via direct relationships

---

## 📐 Key DAX Measures & Columns

### Calculated Columns (UsersData)

```dax
-- Debt-to-Income Ratio
DTI = DIVIDE(UsersData[total_debt], UsersData[yearly_income], 0)

-- Risk Tier (industry-standard DTI thresholds used by lenders)
Loan Risk Cat =
IF(UsersData[DTI] < 0.36, "Low Risk",
    IF(UsersData[DTI] <= 0.43, "Medium Risk", "High Risk"))
```

**Why these thresholds?** DTI < 0.36 (Low), 0.36–0.43 (Medium), > 0.43 (High) mirrors standard mortgage-lending DTI guidelines — giving the segmentation a defensible, industry-grounded rationale rather than arbitrary bucketing.

---

## 📋 Data Limitations & Assumptions

| Limitation | Impact | Decision |
|---|---|---|
| No default/delinquency label in data | Risk tier is a **proxy** (DTI + credit score), not a validated prediction of who actually defaults | Framed as risk segmentation, not a predictive model |
| 45 corrupted latitude records (2.25%) | Geographic analysis excludes these customers | Nulled — not dropped. Financial/demographic fields retained |
| Missing `merchant_state`/`zip` for online transactions | No geographic analysis possible for online spend (100% of Online Transaction rows) | Documented as structural, not a data quality issue |
| `errors` field is multi-label (comma-separated) | Simple GROUP BY gives misleading counts for combined errors | Documented; 13 multi-label rows flagged |
| 2022–2024 covers a high-interest-rate environment | "Rising debt" findings may reflect macro conditions, not individual customer behavior alone | Acknowledged in interpretation; no causal claims made |
| Transaction errors ≠ confirmed fraud | "Bad PIN" or "Bad CVV" could be innocent mistakes, not fraudulent activity | Error analysis framed as anomaly flags, not fraud detection |

---

## 🔐 Row-Level Security (RLS)

Implemented in Power BI using Manage Roles (Modeling tab).

```dax
-- Role: Risk_Analyst (High-risk customers only)
[Loan Risk Cat] = "High Risk"

-- Role: Relationship_Manager (Low and Medium risk customers only)
[Loan Risk Cat] = "Low Risk" || [Loan Risk Cat] = "Medium Risk"

-- Role: Executive (no filter — sees full dataset)
-- No DAX filter applied
```

RLS is defined on `UsersData`. Because `TransactionData` and `CardsData` both connect directly to `UsersData` with Single cross-filter direction, the row filter cascades automatically — no additional DAX needed on fact or card tables.

---

## 📁 Repository Structure

```
Aurora-Bank-Customer-Risk-Spend-Intelligence/
│
├── data/
│   ├── users_data.xlsx
│   ├── cards_data.xlsx
│   ├── transactions_data.xlsx
│   └── mcc_codes.xlsx
│
├── notebooks/
│   └── data_cleaning.ipynb
│
├── powerbi/
│   └── Aurora_Bank_Customer_Risk_and_Spend_Intelligence.pbix
│
├── assets/
│   └── screenshots/
│
└── README.md
```

---

## 💡 Key Business Insights

**Demographics**
- Aurora Bank serves **2,000 customers** — 984 Male, 1,016 Female — with an average age of **45 years**
- Average yearly income is **$45,716** (range: $1 – $307,018); average per capita income is **$23,142**
- Income is heavily skewed: **70.3% of customers fall in the Low income band**, only 2.6% in High — meaning the majority of the portfolio is financially vulnerable by income alone
- Credit score distribution: Good 32%, Excellent 26.7%, Fair 24.6%, Poor 16.7% — **over 41% of customers carry a Fair or Poor credit score**

**Risk Segmentation (Pillar A)**
- **22% of customers (440/2,000) are classified as High Risk** based on the bank's risk scoring model
- Moderate Risk accounts for 44% (879 customers); Low Risk for 34% (681 customers)
- High Risk customers collectively carry **$22M in total debt** vs $63M for Moderate and $43M for Low Risk — Severe Risk DTI tier alone drives **$62M in combined debt exposure**
- Average Risk Score across all customers: **56.24** (range: 35–83); Average Debt-to-Income Ratio: **139.36%**, with a maximum of **497.86%** — indicating some customers carry debt nearly 5× their annual income

**Transaction & Spend Behavior (Pillar B)**
- **157,224 total transactions** processed; **98.3% passed** ($6,706,705), **1.7% failed** ($167,779)
- **Retail dominates** with 45,000 transactions (highest volume category), followed by Automotive (36K) and Entertainment (19K)
- Transaction channel breakdown: **Online 71.3%**, Chip 17.4%, Swipe 11.3% — the majority of spend is card-not-present, which aligns with the missing merchant location data confirmed during cleaning
- Peak transaction hours: **6 AM–12 PM and 12 PM–6 PM** dominate volume across all months; late-night (12 AM–6 AM) is consistently the lowest
- **California, Texas, and New York** are the top 3 states by transaction volume; **Houston** is the highest-volume merchant city

**Error & Risk Analysis**
- **2,738 failed transactions** out of 157,224 total (1.74% error rate)
- **Insufficient Balance is the dominant error type** — accounting for the largest share of failures, signaling genuine financial stress rather than card fraud or technical issues
- **Mastercard accounts for ~50% of all failed transactions** despite having the most issued cards (2,868 chip-enabled) — warranting further monitoring
- Failed transaction value: **$167,779** representing spend that could not be completed

**Card Portfolio**
- Mastercard leads in every dimension: highest transaction amount ($3.37M), most cards issued (471), most transactions (82,000), highest credit limit exposure (**$47M**)
- Visa is second across all metrics: $2.72M transactions, 345 cards, $34M credit limit
- Amex ($5M) and Discover ($2M) represent a small but present share of the portfolio
- **Mastercard carries disproportionate risk concentration** — largest exposure + highest share of failed transactions (49.96%)

---

## 👤 Author

**Subhabrata Sahoo**
Senior Executive – Operations & Analytics → Business & Data Analyst

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/subhabrata99)
[![GitHub](https://img.shields.io/badge/GitHub-100000?style=for-the-badge&logo=github&logoColor=white)](https://github.com/SpartanSubha)
[![Email](https://img.shields.io/badge/Email-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:kitusahoo@gmail.com)

---

## 📖 Glossary

| Term | Plain-Language Definition |
|---|---|
| DTI (Debt-to-Income Ratio) | What % of a person's income is already committed to debt repayment. A higher DTI = less financial buffer = higher risk |
| MCC (Merchant Category Code) | A 4-digit code assigned to every merchant that describes what type of business they are (e.g., 5411 = Grocery Stores) |
| RLS (Row-Level Security) | A Power BI feature that restricts which rows of data each user role can see — the same report shows different data to different roles |
| Star Schema | A database/model design where one central fact table (transactions) connects to multiple dimension tables (customers, cards, categories) |
| Risk Tier | A classification bucket (Low / Medium / High) derived from a customer's DTI ratio, indicating their relative financial risk level |
| Cardinality | In a data relationship, the ratio of matching rows — One-to-Many (1:*) means one customer can have many transactions |
| Fact Table | The central table in a star schema — contains measurable events (transactions). Large row count, connects to all dimension tables |
| Dimension Table | Supporting tables that add context to fact data — who the customer is, what card was used, what category the merchant belongs to |
| Null (Missing Value) | An empty cell. In this project, null in `errors` means the transaction succeeded. Null in `merchant_state` means the transaction was online |
| Power Query M | The formula language Power BI uses inside the Query Editor for data transformation steps before loading into the model |

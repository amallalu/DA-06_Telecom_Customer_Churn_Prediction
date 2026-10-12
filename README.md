# DA-06 · Telecom Customer Churn Prediction

**Category:** Data Analytics · **Tools:** Power BI Desktop, Power Query, DAX, Python (scikit-learn) · **Author:** Amal Lalu K K

![Dashboard preview](preview/dashboard_preview.png)

## 1. Business problem

A telecom operator loses about a quarter of its subscribers a year. The retention team needs to know why customers leave and which current customers are most likely to leave next, so it can call them with the right offer before they cancel.

**Questions this dashboard answers**

- What is the churn rate and how much monthly revenue does it cost?
- Which contract types, services and payment methods have the highest churn?
- Which active customers does the model flag as high risk, and how much revenue do they represent?
- How accurate is the model?

> **About the data:** Synthetic subscriber data shaped like the public IBM Telco Customer Churn sample (including its quirk of blank TotalCharges for brand-new customers).

## 2. What's in this folder

| Path | What it is |
|---|---|
| `data/churn_predictions.csv` | Output of `python/churn_model.py`: churn probability per customer (5,099 rows) |
| `data/customers.csv` | One row per subscriber (5,241 rows) |
| `data/feature_importance.csv` | Model drivers from `python/churn_model.py` (15 rows) |
| `data/model_metrics.csv` | Test-set scores for both models (8 rows) |
| `powerquery/*.m` | Power Query code for each table, ready to paste into the Advanced Editor |
| `dax/measures.dax` | Every DAX measure used in the report |
| `dax/calculated_columns_and_tables.dax` | Calculated columns and tables |
| `python/churn_model.py` | Trains logistic regression and random forest models, scores every customer, exports drivers and metrics |
| `theme/AmalAnalytics_theme.json` | Report colour theme (colourblind-safe palette) |
| `preview/dashboard_preview.png` | What the finished dashboard looks like |
| `STEPS.html` | This guide, formatted to open in any browser |

## 3. Data dictionary

### Predictions — `data/churn_predictions.csv`

Output of `python/churn_model.py`: churn probability per customer

| Column | Type | Description |
|---|---|---|
| `CustomerID` | Text | Unique identifier / key column |
| `ChurnProbability` | Decimal number | Model probability (0–1) that the customer leaves |
| `RiskBand` | Text | High ≥ 0.50, Medium ≥ 0.25, else Low |

### Customers — `data/customers.csv`

One row per subscriber

| Column | Type | Description |
|---|---|---|
| `CustomerID` | Text | Unique identifier / key column |
| `Gender` | Text |  |
| `SeniorCitizen` | Text |  |
| `Partner` | Text |  |
| `TenureMonths` | Whole number | Months as a customer |
| `Contract` | Text | Monthly, One Year or Two Year |
| `InternetService` | Text | DSL, Fiber Optic or No Internet |
| `TechSupport` | Text | Has the tech-support add-on |
| `PaymentMethod` | Text |  |
| `PaperlessBilling` | Text |  |
| `MonthlyCharges` | Decimal number | Current monthly bill in AED |
| `TotalCharges` | Decimal number | Lifetime billed in AED (blank for brand-new customers) |
| `Churn` | Text | Yes = cancelled in the last month |
| `ChurnProbability` | Decimal number | Merged from Predictions |
| `RiskBand` | Text | Merged from Predictions |

*Raw file contains deliberate data-quality issues to clean in Power Query: 5,241 raw rows → 5,099 clean rows.*

### FeatureImportance — `data/feature_importance.csv`

Model drivers from `python/churn_model.py`

| Column | Type | Description |
|---|---|---|
| `Feature` | Text |  |
| `Coefficient` | Decimal number | Standardised logistic-regression coefficient; positive raises churn |
| `ForestImportance` | Decimal number | Random-forest importance (0–1) |
| `Direction` | Text |  |
| `AbsImpact` | Decimal number | Size of the effect |

### ModelMetrics — `data/model_metrics.csv`

Test-set scores for both models

| Column | Type | Description |
|---|---|---|
| `Model` | Text |  |
| `Metric` | Text |  |
| `Value` | Decimal number |  |

`FeatureImportance` and `ModelMetrics` are standalone tables with no relationships; they feed their own visuals.

## 4. Build it step by step

### Step 1 · (Optional) Re-run the Python analysis

The three model output files are already in `data/`. To retrain:

1. `pip install pandas numpy scikit-learn`
2. From this folder run `python python/churn_model.py`
3. It cleans the data the same way as Power Query, one-hot encodes the categories, holds back 25% of customers as a test set, trains a **logistic regression** (explainable) and a **random forest** (benchmark), prints the test-set scores and writes the probabilities for every customer.

### Step 2 · Set up a new report

1. Copy this whole folder somewhere permanent, e.g. `C:\PowerBI\DA-06_Telecom_Customer_Churn_Prediction\`.
2. Open **Power BI Desktop** → **Blank report**.
3. **File → Options and settings → Options → Current file → Data Load** → untick **Auto date/time** (we use our own Date table). Also untick **Autodetect new relationships after data is loaded** under *Relationships*; you will create them yourself in a later step.
4. **View → Themes → Browse for themes** → pick `theme/AmalAnalytics_theme.json`.
5. **File → Save as** → `DA-06_Telecom_Customer_Churn_Prediction.pbix` in this folder. Save often (Ctrl+S).

### Step 3 · Create the folder path parameter

All queries read files relative to one parameter, so the report works on any PC.

1. **Home → Transform data** to open the Power Query Editor.
2. **Home → Manage Parameters → New Parameter**.
3. Name: `FolderPath` · Type: **Text** · Current Value: the full path of this folder **ending with a backslash**, e.g. `C:\PowerBI\DA-06_Telecom_Customer_Churn_Prediction\`.
4. Click **OK**.

### Step 4 · Load and clean the data in Power Query

Repeat for each table below: **Home → New Source → Blank Query** → **Home → Advanced Editor** → delete what's there → paste the code → **Done** → in *Query Settings* on the right, rename the query to the name shown.

#### Query `Predictions`  (file `powerquery/Predictions.m`)

What it does:

- reads `data\churn_predictions.csv` and promotes the first row to headers
- sets the data type of every column (using the en-US locale so dates and decimals parse the same on any PC)

```powerquery
let
    Source = Csv.Document(File.Contents(FolderPath & "data\churn_predictions.csv"), [Delimiter=",", Encoding=65001, QuoteStyle=QuoteStyle.Csv]),
    #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers", {{"CustomerID", type text}, {"ChurnProbability", type number}, {"RiskBand", type text}}, "en-US")
in
    #"Changed Type"
```

Right-click this query → untick **Enable load** (it is only used by other queries).

#### Query `Customers`  (file `powerquery/Customers.m`)

What it does:

- reads `data\customers.csv` and promotes the first row to headers
- sets the data type of every column (using the en-US locale so dates and decimals parse the same on any PC)
- trims stray spaces in `PaymentMethod`
- fixes inconsistent capitalisation (e.g. `north`, `NORTH` → `North`) in `PaymentMethod`
- removes incomplete rows where `TotalCharges` is blank
- removes exact duplicate rows
- merges the `Predictions` query on `CustomerID` and expands `ChurnProbability` and `RiskBand`

```powerquery
let
    Source = Csv.Document(File.Contents(FolderPath & "data\customers.csv"), [Delimiter=",", Encoding=65001, QuoteStyle=QuoteStyle.Csv]),
    #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers", {{"CustomerID", type text}, {"Gender", type text}, {"SeniorCitizen", type text}, {"Partner", type text}, {"TenureMonths", Int64.Type}, {"Contract", type text}, {"InternetService", type text}, {"TechSupport", type text}, {"PaymentMethod", type text}, {"PaperlessBilling", type text}, {"MonthlyCharges", type number}, {"TotalCharges", type number}, {"Churn", type text}}, "en-US"),
    #"Trimmed Text" = Table.TransformColumns(#"Changed Type", {{"PaymentMethod", Text.Trim, type text}}),
    #"Capitalized Each Word" = Table.TransformColumns(#"Trimmed Text", {{"PaymentMethod", Text.Proper, type text}}),
    #"Removed Blank TotalCharges" = Table.SelectRows(#"Capitalized Each Word", each [TotalCharges] <> null),
    #"Removed Duplicates" = Table.Distinct(#"Removed Blank TotalCharges"),
    #"Merged Predictions" = Table.NestedJoin(#"Removed Duplicates", {"CustomerID"}, Predictions, {"CustomerID"}, "Pred", JoinKind.LeftOuter),
    #"Expanded Predictions" = Table.ExpandTableColumn(#"Merged Predictions", "Pred", {"ChurnProbability", "RiskBand"})
in
    #"Expanded Predictions"
```

#### Query `FeatureImportance`  (file `powerquery/FeatureImportance.m`)

What it does:

- reads `data\feature_importance.csv` and promotes the first row to headers
- sets the data type of every column (using the en-US locale so dates and decimals parse the same on any PC)

```powerquery
let
    Source = Csv.Document(File.Contents(FolderPath & "data\feature_importance.csv"), [Delimiter=",", Encoding=65001, QuoteStyle=QuoteStyle.Csv]),
    #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers", {{"Feature", type text}, {"Coefficient", type number}, {"ForestImportance", type number}, {"Direction", type text}, {"AbsImpact", type number}}, "en-US")
in
    #"Changed Type"
```

#### Query `ModelMetrics`  (file `powerquery/ModelMetrics.m`)

What it does:

- reads `data\model_metrics.csv` and promotes the first row to headers
- sets the data type of every column (using the en-US locale so dates and decimals parse the same on any PC)

```powerquery
let
    Source = Csv.Document(File.Contents(FolderPath & "data\model_metrics.csv"), [Delimiter=",", Encoding=65001, QuoteStyle=QuoteStyle.Csv]),
    #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers", {{"Model", type text}, {"Metric", type text}, {"Value", type number}}, "en-US")
in
    #"Changed Type"
```

When all queries show data, click **Home → Close & Apply**.

### Step 5 · Calculated tables and columns

Select table **Customers** → **Table tools → New column** — `Tenure Band`: groups tenure

```dax
Tenure Band =
SWITCH ( TRUE (), Customers[TenureMonths] <= 12, "1-12 m", Customers[TenureMonths] <= 24, "13-24 m", Customers[TenureMonths] <= 48, "25-48 m", "49+ m" )
```

Then **Column tools → Sort by column → Tenure Band Sort**.

Select table **Customers** → **Table tools → New column** — `Tenure Band Sort`: sort order (create first)

```dax
Tenure Band Sort =
SWITCH ( TRUE (), Customers[TenureMonths] <= 12, 1, Customers[TenureMonths] <= 24, 2, Customers[TenureMonths] <= 48, 3, 4 )
```

### Step 6 · Create the measures

1. **Home → Enter data** → name the table `_Measures` → **Load**. (A dedicated table keeps all measures in one place.)
2. Select `_Measures` in the Data pane → **Home → New measure**, paste one measure, press Enter. Repeat for each.
3. After each one, set the format in **Measure tools → Format** as listed.
4. Once you have measures, delete the empty `Column1` from `_Measures`; the table moves to the top of the Data pane.

| Measure | Format | What it means |
|---|---|---|
| `Customers` | Whole number, thousands separator | Subscribers in scope |
| `Churned Customers` | Whole number, thousands separator | Subscribers who cancelled |
| `Churn Rate` | Percentage, 1 decimal | Churned ÷ all customers |
| `Active Customers` | Whole number, thousands separator | Still subscribed |
| `Monthly Revenue Lost` | Currency, 0 decimals | Monthly bills of customers who left |
| `High-Risk Active Customers` | Whole number, thousands separator | Active customers the model flags as High risk |
| `Monthly Revenue at Risk` | Currency, 0 decimals | Monthly bills of High-risk active customers |
| `Avg Tenure (months)` | Decimal number, 1 decimal | Average months as a customer |
| `Model ROC AUC` | Decimal number, 3 decimals | How well the logistic model ranks churners (1 = perfect, 0.5 = random) |

```dax
Customers =
    COUNTROWS ( Customers )

Churned Customers =
    CALCULATE ( [Customers], Customers[Churn] = "Yes" )

Churn Rate =
    DIVIDE ( [Churned Customers], [Customers] )

Active Customers =
    CALCULATE ( [Customers], Customers[Churn] = "No" )

Monthly Revenue Lost =
    CALCULATE ( SUM ( Customers[MonthlyCharges] ), Customers[Churn] = "Yes" )

High-Risk Active Customers =
    CALCULATE ( [Customers], Customers[Churn] = "No", Customers[RiskBand] = "High" )

Monthly Revenue at Risk =
    CALCULATE ( SUM ( Customers[MonthlyCharges] ), Customers[Churn] = "No", Customers[RiskBand] = "High" )

Avg Tenure (months) =
    AVERAGE ( Customers[TenureMonths] )

Model ROC AUC =
    CALCULATE ( MAX ( ModelMetrics[Value] ), ModelMetrics[Model] = "Logistic Regression", ModelMetrics[Metric] = "ROC AUC" )
```

### Step 7 · Build the report pages

Set the page size first: **View → Page view → Fit to page**, and in **Format page → Canvas settings** use 16:9. Add a text box at the top of each page with the page title.

#### Page: Churn Overview

| # | Visual | Fields to drag in | Settings |
|---|---|---|---|
| 1 | Card (x5) | Customers · Churn Rate · Monthly Revenue Lost · High-Risk Active Customers · Monthly Revenue at Risk |  |
| 2 | Clustered column chart | X-axis: Customers[Contract] · Y-axis: Churn Rate |  |
| 3 | Clustered column chart | X-axis: Customers[Tenure Band] · Y-axis: Churn Rate |  |
| 4 | Clustered bar chart | Y-axis: Customers[PaymentMethod] · X-axis: Churn Rate |  |
| 5 | Clustered bar chart | Y-axis: Customers[InternetService] · X-axis: Churn Rate |  |
| 6 | Slicers | Customers[SeniorCitizen] · Customers[Contract] |  |

#### Page: Model & Drivers

| # | Visual | Fields to drag in | Settings |
|---|---|---|---|
| 1 | Clustered bar chart | Y-axis: FeatureImportance[Feature] · X-axis: Sum of Coefficient · Legend: FeatureImportance[Direction] | Top N = 10 by AbsImpact; shows what raises or lowers churn |
| 2 | Matrix | Rows: ModelMetrics[Metric] · Columns: ModelMetrics[Model] · Values: Sum of Value | Compare logistic regression against random forest |
| 3 | Card | Model ROC AUC |  |

#### Page: Retention Call List

| # | Visual | Fields to drag in | Settings |
|---|---|---|---|
| 1 | Table | CustomerID, Contract, TenureMonths, MonthlyCharges, ChurnProbability, RiskBand | Filters: Churn = No, RiskBand = High. Sort by ChurnProbability descending. Data bars on ChurnProbability |
| 2 | Donut chart | Legend: Customers[RiskBand] · Values: Active Customers |  |

### Step 8 · Check your numbers

With **no slicers or filters applied**, your cards should show these values. If one is different, re-check that measure and the cleaning steps.

| Measure | Expected value |
|---|---|
| `Customers` | 5,099 |
| `Churned Customers` | 1,552 |
| `Churn Rate` | 30.4% |
| `Active Customers` | 3,547 |
| `Monthly Revenue Lost` | AED 113,621 |
| `High-Risk Active Customers` | 406 |
| `Monthly Revenue at Risk` | AED 34,425 |
| `Avg Tenure (months)` | 28.3 |
| `Model ROC AUC` | 0.848 |

### Step 9 · Save and share

1. **File → Save** — your finished `DA-06_Telecom_Customer_Churn_Prediction.pbix`.
2. **File → Export → Export to PDF** for a static copy, or take a screenshot of each page for your portfolio and GitHub.
3. Optional: **Home → Publish** to Power BI Service (needs a work or school account) to get a shareable link.

## 5. Key findings from this data

- **Churn rate by contract:** highest is Monthly (48.3%), lowest is Two Year (4.9%).
- **Churn rate by tenure:** highest is 1-12 m (53.6%), lowest is 49+ m (6.2%).
- **Top churn drivers (standardised coefficient):** highest is InternetService = Fiber Optic (0.42), lowest is Contract = Two Year (-0.90).
- **Active customers by risk band:** Low is the largest share (63.3%).
- Positive coefficients raise churn risk: InternetService = Fiber Optic, MonthlyCharges, PaymentMethod = Electronic Check. Negative ones lower it: Contract = Two Year, TenureMonths, Contract = One Year.

## 6. Skills shown

- Python machine learning: logistic regression and random forest, train/test split, ROC AUC
- Power Query merge of model scores into the customer table
- DAX: CALCULATE with multiple filters, revenue-at-risk
- Turning a model into an actionable call list

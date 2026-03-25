# Supply Chain Command Center

> A director-level supply chain analytics dashboard delivering single-pane-of-glass operational visibility across global supply chain functions — built for speed of insight, not speed of development.

![Python](https://img.shields.io/badge/Python-3.10%2B-blue?style=flat-square)
![Streamlit](https://img.shields.io/badge/Streamlit-1.x-red?style=flat-square)
![Plotly](https://img.shields.io/badge/Plotly-5.x-purple?style=flat-square)
![SCOR Model](https://img.shields.io/badge/Framework-SCOR%20Model-orange?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)

---

## The Problem This Solves

In most enterprises, supply chain performance data lives across 4–6 disconnected systems:
- OTD in the ERP (SAP / Oracle)
- Inventory in the WMS
- Supplier scorecards in a procurement portal
- Risk events in a spreadsheet someone emails on Fridays
- Cost data in Finance's BI tool

A director preparing for a Monday operations review spends 2–3 hours pulling this together manually — and the result is already stale.

This dashboard consolidates all of it into a single, filtered, always-consistent view. Change the region, the product line, or the fiscal period — every KPI, every chart, every risk event updates simultaneously from the same data slice.

---

## Functional Coverage

| Module | What It Shows | Business Question Answered |
|---|---|---|
| **KPI Cards** | OTD, Perfect Order, Inventory Turns, Fill Rate, Savings, Risk Index | *Are we performing above or below target this period?* |
| **Trend Charts** | Monthly performance + cost savings overlay | *Are we improving or declining — and how fast?* |
| **Supplier Intelligence** | Scorecard + capability radar for top suppliers | *Which suppliers are underperforming and where is our spend concentrated?* |
| **Cost Structure** | Actual vs. budget by logistics category | *Where are we over-budget and by how much?* |
| **Demand–Supply Balance** | Volume gap + backlog by product and period | *Where is demand outpacing supply and building backlog?* |
| **Risk Register** | Severity-coded events + probability-impact heatmap | *What is our highest-exposure risk right now and what is the financial impact?* |

---

## KPI Methodology — SCOR Model

All six KPIs follow the **SCOR model** (Supply Chain Operations Reference) — the APICS industry standard used by Gartner, McKinsey, and most Fortune 500 operations teams.

### 1. On-Time Delivery (OTD)
**Measures:** Reliability of delivery against customer promise dates.

```
OTD % = (Orders delivered on or before promised date ÷ Total orders) × 100
```

**Source table:** `orders` — requires `promised_date`, `actual_delivery_date`

**Benchmark:** World-class ≥ 95%. Below 90% triggers customer churn risk.

---

### 2. Perfect Order Rate
**Measures:** End-to-end order quality — the hardest KPI to move because all four
conditions must be true simultaneously.

```
Perfect Order % = OTD % × Complete % × Damage-Free % × Invoice Accuracy %

Where:
  Complete %        = units shipped ÷ units ordered
  Damage-Free %     = orders without damage claims ÷ total orders
  Invoice Accuracy% = invoices without disputes ÷ total invoices
```

**Source tables:** `orders`, `claims`, `invoices`

**Benchmark:** World-class ≥ 90%. Every 1pp improvement typically reduces
cost-to-serve by 0.5–1.2%.

---

### 3. Inventory Turns
**Measures:** Capital efficiency — how fast inventory converts to revenue.

```
Inventory Turns = Annual COGS ÷ Average Inventory Value

Where:
  Average Inventory Value = (Opening Stock Value + Closing Stock Value) ÷ 2
```

**Source tables:** `financials` (COGS), `inventory_snapshots` (monthly values)

**Benchmark:** Industry average 8.4×. Below 6× signals excess stock and
working capital drag. Above 12× risks stockout exposure.

---

### 4. Fill Rate
**Measures:** Demand fulfillment from available stock on first attempt —
without backorder or substitution.

```
Fill Rate % = (Units shipped on first attempt ÷ Units ordered) × 100
```

**Source table:** `orders` — requires `units_ordered`, `units_shipped_first_attempt`

**Benchmark:** SLA threshold typically 97%. Below 95% directly drives
customer escalations and lost revenue.

---

### 5. Procurement Savings
**Measures:** Cost reduction achieved vs. baseline pricing across the period.

```
Saving per line = (Baseline Price − Negotiated Price) × Quantity Purchased
Total Savings   = Σ saving per line across all purchase orders in period
```

**Source table:** `purchase_orders` — requires `baseline_price`,
`negotiated_price`, `quantity`

**Note:** Cumulative across the period (sum, not point-in-time).
Q4 will always show lower savings than FY — this is correct behaviour.

---

### 6. Supplier Risk Index
**Measures:** Forward-looking composite risk across the supplier portfolio.
Unlike other KPIs, this is a leading indicator — it tells you what is
coming, not what has happened.

```
Risk Index (0–100) = weighted composite of:

  Financial Risk     (30%) = f(credit rating, payment delay days)
  Concentration Risk (25%) = f(% total spend with single supplier)
  Geographic Risk    (20%) = f(country risk score — political, port, customs)
  Lead Time Risk     (15%) = f(standard deviation of delivery lead times)
  Quality Risk       (10%) = f(defect rate, quality incident count)

Portfolio Index = spend-weighted average across all active suppliers
```

**Source table:** `suppliers` — requires financial, geographic, and
performance attributes per supplier

**Benchmark:** Target < 20. Above 35 = elevated risk requiring mitigation
plan. Above 50 = board-level escalation threshold.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Filter State                          │
│         region │ product_line │ period                  │
└──────────────────────┬──────────────────────────────────┘
                       │  shared across all modules
          ┌────────────┼────────────┐
          ▼            ▼            ▼
    get_monthly()  get_inventory() get_suppliers()
    get_demand_supply()  get_risks()  get_costs()
          │            │            │
          └────────────┼────────────┘
                       ▼
              Plotly chart objects
              KPI scalar cards
              HTML tables
                       │
                       ▼
              Streamlit render layer
```

**Key design principle:** Every data function receives the same filter
state and produces a filtered result independently. There is no global
mutable state — changing a filter triggers a full Streamlit re-run,
which re-executes every function with the new filter values. This
guarantees all 15+ visuals always reflect the same data slice.

---

## Data Model

The dashboard is designed to connect to these source tables. The current
implementation uses procedurally generated data that matches this schema —
replacing the generation layer with real queries requires no changes to
the presentation layer.

```
orders
  order_id          STRING    PK
  region            STRING    FK → regions
  product_line      STRING    FK → products
  order_date        DATE
  promised_date     DATE
  actual_delivery   DATE
  units_ordered     INTEGER
  units_shipped     INTEGER
  damaged           BOOLEAN
  invoice_correct   BOOLEAN

inventory_snapshots
  snapshot_date     DATE
  product_line      STRING
  region            STRING
  units_on_hand     INTEGER
  inventory_value   DECIMAL

purchase_orders
  po_id             STRING    PK
  supplier_id       STRING    FK → suppliers
  product_line      STRING
  item_id           STRING
  baseline_price    DECIMAL
  negotiated_price  DECIMAL
  quantity          INTEGER
  purchase_date     DATE

suppliers
  supplier_id       STRING    PK
  name              STRING
  region            STRING
  product_line      STRING
  credit_score      INTEGER   (0-100)
  spend_share_pct   DECIMAL
  country_risk      INTEGER   (0-100)
  avg_lead_time     INTEGER   (days)
  lead_time_stddev  DECIMAL
  defect_rate       DECIMAL
  quality_incidents INTEGER

financials
  period            DATE      (monthly)
  product_line      STRING
  region            STRING
  cogs              DECIMAL
  inventory_open    DECIMAL
  inventory_close   DECIMAL
```

---

## Connecting Real Data

Replace the simulation layer with your data source:

```python
# Current (simulation)
d = gen_monthly(**REGION_KPI_BASE[reg])

# Replace with — CSV / Excel
orders_df      = pd.read_csv("data/orders.csv", parse_dates=["promised_date", "actual_delivery"])
suppliers_df   = pd.read_csv("data/suppliers.csv")
inventory_df   = pd.read_csv("data/inventory_snapshots.csv")
financials_df  = pd.read_csv("data/financials.csv")
purchase_df    = pd.read_csv("data/purchase_orders.csv")

# Or — SQL database
import sqlalchemy
engine = sqlalchemy.create_engine("postgresql://user:pass@host/scdb")
orders_df = pd.read_sql("SELECT * FROM orders WHERE region = %s", engine, params=[sel_region])

# Or — SAP / Oracle via API
# (wrap your ERP API client here — presentation layer unchanged)
```

---

## Project Structure

```
supply-chain-command-center/
│
├── supply_chain_dashboard.py   # Main application entry point
│
├── data/                       # Data layer (swap for real sources)
│   ├── simulation.py           # Procedural data generation (current)
│   ├── loaders.py              # CSV / database loaders (template)
│   └── schema.sql              # Reference schema for real deployment
│
├── kpis/                       # KPI calculation modules
│   ├── delivery.py             # OTD + Perfect Order Rate
│   ├── inventory.py            # Inventory Turns + Fill Rate
│   ├── procurement.py          # Savings calculation
│   └── risk.py                 # Supplier Risk Index composite
│
├── components/                 # Reusable chart components
│   ├── cards.py                # KPI card renderer
│   ├── charts.py               # Plotly chart builders
│   └── tables.py               # HTML table renderers
│
├── tests/
│   ├── test_kpis.py            # Unit tests for KPI formulas
│   └── test_filters.py         # Filter state integration tests
│
├── requirements.txt
└── README.md
```

> **Note:** Current implementation is a single-file prototype. The
> structure above represents the target architecture for production
> deployment.

---

## Setup

```bash
# Clone
git clone https://github.com/yourname/supply-chain-command-center.git
cd supply-chain-command-center

# Install dependencies
pip install -r requirements.txt

# Run
streamlit run supply_chain_dashboard.py
```

**requirements.txt**
```
streamlit>=1.28.0
plotly>=5.18.0
pandas>=2.0.0
numpy>=1.24.0
```

---

## Roadmap

| Priority | Feature | Business Value |
|---|---|---|
| P1 | Connect to CSV / Excel data sources | Make KPIs real, not simulated |
| P1 | Unit tests for all KPI formulas | Ensure calculation accuracy before director review |
| P2 | ERP connector (SAP / Oracle) | Live data, eliminate manual refresh |
| P2 | Email / Teams alerting when KPIs breach threshold | Proactive issue management |
| P3 | ML demand forecasting layer | Shift from reactive to predictive operations |
| P3 | Supplier risk scoring automation | Replace manual risk assessment |
| P3 | Mobile-responsive layout | Field operations access |

---

## Why These Design Decisions

**Why Streamlit over Power BI / Tableau?**
Full code control. Every calculation is transparent, version-controlled,
and auditable. No licensing cost. Deployable anywhere Python runs.

**Why simulate data instead of mocking?**
Procedural generation with region-specific baselines produces data that
behaves like real supply chain data — regional performance gaps, seasonal
variance, improvement trends. A static mock would not respond correctly
to filter changes.

**Why SCOR model specifically?**
It is the only supply chain framework with universal industry adoption.
Using SCOR terminology means this dashboard speaks the same language as
Gartner benchmarks, consultant deliverables, and board reporting —
reducing translation overhead when presenting to senior leadership.


---

*Confidential — Director Use Only*

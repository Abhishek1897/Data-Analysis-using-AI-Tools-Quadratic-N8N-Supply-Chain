# 📦 End-to-End Supply Chain Data Automation & Analytics

**Gmail → n8n → Supabase (PostgreSQL) → Quadratic AI**

An automated data pipeline for an FMCG supply chain. Order data arrives by email as CSV files. n8n saves it to a PostgreSQL database on Supabase, and Quadratic AI analyses delivery performance (**On-Time, In-Full and OTIF**).

![n8n](https://img.shields.io/badge/n8n-Workflow_Automation-EA4B71?logo=n8n&logoColor=white)
![Supabase](https://img.shields.io/badge/Supabase-PostgreSQL-3ECF8E?logo=supabase&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Database-4169E1?logo=postgresql&logoColor=white)
![Gmail API](https://img.shields.io/badge/Gmail_API-OAuth_2.0-EA4335?logo=gmail&logoColor=white)
![Quadratic](https://img.shields.io/badge/Quadratic_AI-Analytics-6B46C1)
![Python](https://img.shields.io/badge/Python-Data_Cleaning-3776AB?logo=python&logoColor=white)

---

## 📑 Table of Contents
- [Problem Statement](#-problem-statement)
- [Architecture](#-architecture)
- [Tech Stack](#-tech-stack)
- [Data Model](#-data-model)
- [Setup Guide](#-setup-guide)
- [n8n Workflow](#-n8n-workflow)
- [Analysis in Quadratic AI](#-analysis-in-quadratic-ai)
- [Supply Chain KPIs](#-supply-chain-kpis)
- [Results](#-results)
- [Key Learnings](#-key-learnings)
- [Future Improvements](#-future-improvements)
- [Acknowledgements](#-acknowledgements)

---

## 🎯 Problem Statement

FMCG supply chain teams need to know every day whether customer orders are delivered **on time** and **in full**. The raw order data often arrives as CSV attachments in email. Someone then downloads the files by hand and pastes them into spreadsheets, which is slow and error-prone.

**Goal:** automate the move from email to database, then use AI-assisted analysis to measure delivery reliability and find problem customers and products.

---

## 🏗 Architecture

```
┌──────────┐     ┌──────────────────┐     ┌───────────────────┐     ┌────────────────┐     ┌──────────────┐
│  Gmail   │ ──▶ │       n8n        │ ──▶ │     Supabase      │ ──▶ │  Quadratic AI  │ ──▶ │   Insights   │
│ CSV mail │     │ Trigger + Extract│     │ PostgreSQL tables │     │ SQL + Python   │     │ OTIF, Fill % │
└──────────┘     └──────────────────┘     └───────────────────┘     └────────────────┘     └──────────────┘
      └──────────── Ingestion (automated) ─────────────┘                 └──── Analysis (AI-assisted) ────┘
```

---

## 🛠 Tech Stack

| Tool | Purpose |
|------|---------|
| **Node.js & npm** | Runtime used to install and run n8n |
| **n8n** (self-hosted, `localhost:5678`) | Workflow automation: Gmail trigger, CSV extraction, Postgres inserts |
| **Google Cloud Console** | Gmail API enabled, OAuth consent screen and OAuth 2.0 Client ID |
| **Gmail / Gmail API** | Source of the data (emails with CSV attachments) |
| **Supabase (PostgreSQL)** | Cloud database that stores the fact and dimension tables |
| **Quadratic AI** | AI spreadsheet with SQL and Python for cleaning, KPIs and analysis |
| **Open Exchange Rates API** | Historical USD → INR rates for currency conversion |

---

## 🗄 Data Model

A star schema in the `public` schema on Supabase:

| Table | Type | Columns |
|-------|------|---------|
| `fact_order_line` | Fact (line level) | order_id, order_placement_date, customer_id, product_id, order_qty, agreed_delivery_date, actual_delivery_date, delivery_qty, In Full, On Time, On Time In Full |
| `fact_aggregate` | Fact (order level) | order_id, customer_id, order_placement_date, on_time, in_full, otif |
| `dim_customers` | Dimension | customer_id, customer_name, city, currency |
| `dim_products` | Dimension | product_name, product_id, category, price_INR, price_USD |
| `dim_targets_orders` | Dimension | customer_id, ontime_target%, infull_target%, otif_target% |

![Supabase Schema](images/supabase_schema.png)

---

## ⚙️ Setup Guide

### 1. Install n8n locally
```bash
# Install Node.js first (Windows .msi installer, tick "Automatically install the necessary tools")
npm install -g n8n
n8n          # press "o" to open in the browser → http://localhost:5678
```
Create your n8n owner account when prompted.

### 2. Create the PostgreSQL database in Supabase
1. Sign up at [supabase.com](https://supabase.com), then create an **organization** and a **project**. Save the database password.
2. Go to **Database → Create a new table → Import data from CSV** for each table:
   `fact_aggregate`, `dim_customers`, `dim_products`, `dim_targets_orders`, `fact_order_line`.
3. Change date columns to the `date` type.
4. Click **Connect** and choose **Session Pooler** (for IPv4 networks). Copy the host, port, database, user and password.

### 3. Create an OAuth Client ID for the Gmail API
1. In [Google Cloud Console](https://console.cloud.google.com), create a new project and enable the **Gmail API**.
2. Configure the **OAuth consent screen**: set the app name and support email, choose **External** as the audience, and add your Gmail as a **test user**.
3. Go to **Credentials → Create client → Web application** and add this redirect URI:
   ```
   http://localhost:5678/rest/oauth2-credential/callback
   ```
4. Paste the **Client ID** and **Client Secret** into n8n's Gmail OAuth2 credential, then sign in and grant the requested scopes.

### 4. Connect n8n to Supabase
Create a **Postgres** credential in n8n with the Session Pooler parameters from step 2.

---

## 🔄 n8n Workflow

![n8n Workflow](images/n8n_workflow.png)

| Node | Operation | Purpose |
|------|-----------|---------|
| Gmail Trigger | Trigger (download attachments) | Fires when the data email arrives |
| Extract from File | Extract From CSV | Parses the first CSV attachment |
| Extract from File1 | Extract From CSV | Parses the second CSV attachment |
| Insert rows in a table | Postgres · insert | Writes branch 1 rows to Supabase |
| Insert rows in a table1 | Postgres · insert | Writes branch 2 rows to Supabase |

One trigger feeds **two parallel branches**, one for each CSV attachment.

---

## 📊 Analysis in Quadratic AI

The Quadratic file **Customer Data** connects to Supabase through a PostgreSQL connection (`n8n proj`). Each table gets its own sheet, loaded live with SQL:

```sql
SELECT * FROM "public"."fact_order_line";
```

![Quadratic AI](images/quadratic_dim_products.png)

### Prompts used
| # | Prompt | Output |
|---|--------|--------|
| 1 | Create a date table from 01-Mar-2025 to 31-May-2025 | `date` table |
| 2 | Create an exchange_rate table (01-Mar to 17-May-2025) from the Open Exchange Rates historical API | Daily USD → INR rates |
| 3 | Clean, merge and summarise into `fact_summary` | Clean IDs, join orders + products + customers + rates, `total_amount` in INR |
| 4 | Create business KPIs | Total Order Lines, LIFR, VOFR, Total Orders, OT %, IF %, OTIF % |
| 5 | Top 5 customers by order value with OTIF %, IF %, OT % | Customer performance table |
| 6 | Top 5 customers in India | India-only view |

**Currency logic used in `fact_summary`:**
- USD customers: `price_USD × USD_INR_Rate × order_qty`
- INR customers: `price_INR × order_qty`

### Business questions explored
- How much revenue is lost to undelivered orders?
- Which customers have the biggest OTIF gaps?
- Which product categories have the lowest In-Full rates, and what does that say about supply bottlenecks?
- What is the average delay for late deliveries?

---

## 📐 Supply Chain KPIs

| KPI | Level | Definition |
|-----|-------|------------|
| **Line Fill Rate** | Line | Lines shipped in full ÷ total lines ordered |
| **Volume Fill Rate** | Line | Quantity shipped ÷ quantity ordered |
| **On-Time %** | Order | Orders with *all* lines delivered by the agreed date ÷ total orders |
| **In-Full %** | Order | Orders with *all* lines delivered in full ÷ total orders |
| **OTIF %** | Order | Orders that are both On Time **and** In Full ÷ total orders |

> **Example:** An order has 4 notebooks and 2 pens. 4 notebooks and 1 pen are shipped.
> Line Fill Rate = 1/2 = **50%** · Volume Fill Rate = 5/6 = **83%** · The order is **not OTIF**.

---

## ✅ Results

- ✔️ **Automated ingestion:** 1 email triggered the workflow. **57 + 109 = 166 rows** were extracted and inserted into PostgreSQL, and every node completed successfully.
- ✔️ **Cloud database:** 5 related tables on Supabase with correct date types.
- ✔️ **Live analysis:** Quadratic sheets pull each table straight from Postgres.
- ✔️ **Clean dataset:** `fact_summary` holds all order values in INR.
- ✔️ **KPI layer:** OT %, IF %, OTIF %, fill rates and top-customer views.

---

## 💡 Key Learnings
- Building trigger-based workflows with parallel branches in n8n
- Setting up OAuth 2.0 end to end (consent screen, test users, redirect URIs)
- Creating Postgres tables from CSV and connecting through a pooler on IPv4
- Designing fact and dimension tables for KPI slicing
- Writing precise prompts that produce reliable Python for data cleaning and KPIs
- The difference between line-level and order-level supply chain metrics

---

## 🚀 Future Improvements
- [ ] Upsert / de-duplication so re-sent emails don't create duplicate rows
- [ ] Filter the Gmail Trigger by sender or subject and add an error-alert branch
- [ ] Daily automated KPI summary by email or Slack
- [ ] Live dashboard tracking OTIF against customer targets

---

## 📁 Repository Structure
```
├── README.md
├── images/
│   ├── n8n_workflow.png
│   ├── supabase_schema.png
│   └── quadratic_dim_products.png
└── docs/
    └── Supply_Chain_Automation_Project_Report.pdf
```

---

## 🙏 Acknowledgements
- [Codebasics](https://codebasics.io) Resume Project Challenge (FMCG | C2)
- [n8n](https://n8n.io) · [Supabase](https://supabase.com) · [Quadratic](https://www.quadratichq.com) · [Open Exchange Rates](https://openexchangerates.org)

---

## 👤 Author
**Abhishek Shinde**

⭐ If you found this project useful, consider giving it a star!

# NeoBank Data Pipeline on Databricks

A beginner-friendly data engineering project. It takes raw banking data from **SQL Server** and **CSV files**, cleans it step by step in **Databricks**, and turns it into tables that a business team can use.

The data is **made-up (synthetic)**. No real customer data is used.

---

## What problem does this solve?

A bank keeps its data in different places: customer details, accounts, transactions, credit reports and payment logs. Raw data like this is hard to use for reports.

This project collects all of it in one place, organizes it and creates ready-to-use tables such as "how many customers does each branch have?" or "how many customers are high risk?".

## How it works (Bronze, Silver, Gold)

The project uses the **Medallion Architecture**, which has three layers:

| Layer | What it means | What happens here |
|---|---|---|
| **Bronze** | Raw data | Data is copied from the sources as it is |
| **Silver** | Clean data | Data is cleaned and made ready to use |
| **Gold** | Business-ready data | Data is summarized into tables for reports and dashboards |

```
SQL Server  ─┐
             ├─►  Bronze  ─►  Silver  ─►  Gold  ─►  Dashboard
CSV files   ─┘
```


## Tools Used

- **Databricks** to build and run the pipeline
- **PySpark and SQL** to process the data
- **Delta Lake** to store the tables
- **SQL Server** as one data source
- **CSV files** as the other data source
- **Databricks Workflows (Jobs)** to run the steps in order


## Data Sources

| Source | Data |
|---|---|
| SQL Server | customers, accounts, transactions, branches |
| CSV files | credit bureau reports, payment gateway logs |

The project has a first (historical) load of data and a second smaller batch of new data. The second run loads only the new data, to show how updates are handled.

## Main Features

- Data comes from two kinds of sources: a database and files.
- The data goes through three layers: Bronze, Silver and Gold.
- The settings for each table (for example, whether to replace or add data) are kept in a small settings table, so they are not written inside the code.
- Each run is recorded in a log table, so we can see what loaded and whether it succeeded.


## Gold Tables

| Table | What it shows |
|---|---|
| `customer_360` | One row per customer with their accounts, balance, transactions and credit risk |
| `branch_performance` | Customers, deposits and transactions for each branch |
| `transaction_channel_summary` | Transactions by channel, gateway and device |
| `daily_bank_kpi` | Daily transaction totals and overall bank numbers |
| `risk_customer_summary` | Number of customers in each credit risk group |



## Project Folders

```
.
├── 00_Source_Files/        # SQL scripts and CSV files (sample data)
├── 01_Metadata/            # creates the settings and log tables
├── 02_Bronze_and_Silver/   # loads data into Bronze and Silver
├── 03_Gold                 # creates the Gold tables
└── README.md
```

## How to Run

1. Create the tables in SQL Server and load the sample data from `00_Source_Files`.
2. Upload the CSV files to the Databricks folder (Volume).
3. Run the setup notebooks in `01_Metadata`.
4. Create a Databricks Job with these steps in order: Source to Bronze, Bronze to Silver, Silver to Gold, Email.
5. Run the job once for the first load.
6. Add the new sample data and run the job again to see the second load.


## What Can Be Improved

This is a learning project, and there are things I would improve:

- The Silver layer does only light cleaning. More data quality checks can be added.
- The Gold tables are rebuilt fully on every run.
- There are no automated tests yet.
- Re-running a step may load some rows twice.

---

## 👨‍💻 About the Author

**Surajkumar Bevnale**
Data Analyst | SQL • Excel • Power BI • Python
B.Tech, Engineering Physics, IIT Ropar (2023–2027)

- 📫 Reach me at: surajk.b168@gmail.com
- 🔗 LinekdIn: [Surajkumar B](https://www.linkedin.com/in/skb168/)
---


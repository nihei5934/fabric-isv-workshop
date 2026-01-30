
# Microsoft Fabric End-to-End Workshop – SaaS Telemetry & AI Analytics

This repository contains a complete, reproducible walkthrough for building a **Microsoft Fabric analytics solution** using synthetic SaaS telemetry data.

The end goal is to deliver **executive-ready Power BI reports** and **AI-enriched insights** (sentiment, entities, trends) in under **2 hours**, optimized for **hands-on SMB / ISV workshops**.

## Prerequisites

- Azure subscription
- Fabric-enabled tenant
- Permissions to:
  - Create Resource Groups
  - Create Storage Accounts
  - Assign RBAC roles
  - Create Fabric capacities
  - Create Power BI / Fabric workspaces

---

## Pre-Steps (Azure Setup)

### 1. Create Resource Group
Create a new resource group in Azure for the workshop.

### 2. Create Fabric Capacity
- Assign yourself as **Capacity Admin**
- Size: **Fabric Large**
- Enable **Large Semantic Model Storage**

### 3. Create Storage Account
- Type: **Azure Blob Storage**
- Performance: **Standard**
- Redundancy: **LRS**
- Networking: **Allow Public Network Access**
- IAM:
  - Assign yourself **Storage Blob Data Owner**

### 4. Create Container & Directory
- Container name: `default`
- Directory: `raw`

### 5. Upload Data Files
Upload the following CSV files into `/default/raw`:
- `feature_usage.csv`
- `tenants.csv`
- `users.csv`
- `support_tickets.csv`

---

## Fabric Workspace Setup

### 6. Create Fabric Workspace
- Go to Power BI / Fabric Portal
- Create new workspace
- Assign workspace to Fabric Capacity
- Enable **Large Semantic Model Storage**

---

## Lakehouse & Shortcut

### 7. Create Lakehouse
- In workspace, create **New Lakehouse**

### 8. Create OneLake Shortcut
- Lakehouse → Get Data → New Shortcut
- Source: **Azure Blob Storage**
- Create new connection:
  - URL: `{storage-account-name}.blob.core.windows.net`
  - Authentication: **Organizational Account**
- Browse to `default/raw`
- Select folder
- **Skip “Revert changes for Delta auto transformation”**

---

## Dataflow Gen2 (Transforms)

### 9. Create Dataflow Gen2
- Create **New Dataflow Gen2**
- Disable Git integration (workshop simplicity)

### 10. Load Data
- Get Data → More → Lakehouse
- Select **all 4 CSV files**
- Apply **First Row as Headers**

---

### 11. Transform `feature_usage` Table

- Confirm data types
- Split column `ts` by **space**
- Delete original `ts`
- Rename:
  - `ts.1` → `date`
  - `ts.2` → `time`

✅ Avoids TODAY() issues and simplifies MAU logic.

---

### 12. Transform `support_tickets` Table

- Split `resolved_at` by **space**
- Delete original `resolved_at`
- Rename:
  - `resolved_at.1` → `resolved_date`
  - `resolved_at.2` → `resolved_time`
- Confirm data types

---

### 13. Merge Tenants Data

- Merge `feature_usage` with `tenants`
- Join key: `tenant_id`
- Expand needed tenant columns
- Drop duplicate tenant_id

---

### 14. Copilot Transformation (Feature Usage)

Use **Copilot** prompt:

> Create a column that buckets latency_ms as:
> - low: 0–20
> - medium: 21–100
> - high: 100+

---

### 15. Output to Lakehouse

- Set **Default Destination**
- Select Lakehouse from OneLake Catalog
- Remove `csv_` prefix from table names
- Publish Dataflow

---

## Notebook – AI Enrichment

### 16. Create Notebook
- Name: **AI Enrichment**

### 17. Configure Environment
- Environment dropdown → New Environment
- External Libraries:
  - Source: PyPI
  - Package: `openai`
- Publish environment
- Re-open notebook
- Attach environment
- Attach Lakehouse as data source

---

### 18. Entity Extraction via Data Wrangler

- Load `support_tickets`
- Select `body` column
- Extract Entities:
  - `app_version`
  - `operating_system`
- Convert extracted columns to **String**
- Copilot prompt:
  > Remove square brackets and quotes from entity columns

⚠️ Delete any auto-generated `raw` variable lines.

---

### 19. Save AI-Enriched Table

```python
df_clean.write \
  .format("delta") \
  .option("overwriteSchema", "true") \
  .mode("overwrite") \
  .saveAsTable("support_tickets_ai")
```
---

## Semantic Model (continued)

### 20. Create Semantic Model (continued)

- From the Lakehouse, open **SQL Analytics Endpoint**
- Select **New Semantic Model**
- Name the model (example: `SaaS Telemetry Semantic Model`)
- Select the following tables:
  - `feature_usage`
  - `tenants`
  - `users` (optional)
  - `support_tickets_ai`
- **Do NOT select** the original `support_tickets` table (raw, non-AI)

Click **Create**.

---

### 21. Open Semantic Model for Editing

- Navigate back to the workspace
- Click the Semantic Model
- Select **Open Semantic Model**
- Toggle **Editing mode** ON

---

### 22. Create Relationships

In **Model view**, create these relationships:

| From Table            | Column      | To Table | Column     |
|-----------------------|-------------|----------|------------|
| `tenants`             | tenant_id   | feature_usage | tenant_id |
| `tenants`             | tenant_id   | support_tickets_ai | tenant_id |
| `users` (optional)    | user_id     | feature_usage | user_id |

- Cardinality: **One-to-many**
- Cross-filter direction: **Single**

---

### 23. Create Measures (Direct Lake Safe)

Create each measure **one at a time**  
(Modeling → New Measure)

#### DAU (Daily Active Users)
```DAX
DAU =
DISTINCTCOUNT ( feature_usage[user_id] )


MAU =
VAR MaxDate =
    MAX ( feature_usage[date] )
RETURN
CALCULATE (
    DISTINCTCOUNT ( feature_usage[user_id] ),
    FILTER (
        ALL ( feature_usage ),
        feature_usage[date] >= MaxDate - 30
            && feature_usage[date] <= MaxDate
    )
)


Stickiness =
DIVIDE ( [DAU], [MAU] )


Error Rate % =
DIVIDE (
    CALCULATE (
        COUNTROWS ( feature_usage ),
        feature_usage[success] = 0
    ),
    COUNTROWS ( feature_usage )
)
```
### 24. Create PowerBI Reports using CoPilot



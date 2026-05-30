# Azure-Automated-Backup-System#

🔄 Azure Automated Backup System

> **Problem:** Backups depend on someone remembering — until data is just gone.  
> **Solution:** Fully automated daily backup pipeline using Azure Blob Storage, Azure SQL, and Logic Apps.

---

## 📐 Architecture

```
Daily Trigger (Logic App)
        ↓
Execute SQL Query (BackupLog table)
        ↓
Write to Blob Storage (/backups)
        ↙              ↘
✅ Success Email     ❌ Failure Alert Email
```

---

## ☁️ Azure Resources Used

| Resource | Name | Purpose |
|---|---|---|
| Storage Account | `mikekingstorage1` | Hosts the backup blob container |
| Blob Container | `backups` | Stores daily backup JSON files |
| SQL Server | `mikekingbackupserver` | Logical SQL server |
| SQL Database | `mikekingbackupdb` | Contains the `BackupLog` table |
| Logic App | `daily-backup-confirmation` | Orchestrates the entire workflow |
| Resource Group | `BackupSystem-RG` | Contains all resources |

---

## 🔧 Components

### 1. Azure Blob Storage + Versioning
- Storage account with blob versioning enabled
- Private `backups` container
- Access via storage account access key

### 2. Lifecycle Management Policy
Blobs in the `backups` container automatically transition:

| Age | Action |
|---|---|
| 30 days | Move to Cool storage |
| 90 days | Move to Archive storage |
| 365 days | Delete |
| Versions > 180 days | Delete old versions |

### 3. Azure SQL Database
- Server: `mikekingbackupserver.database.windows.net`
- Database: `mikekingbackupdb`
- Table: `dbo.BackupLog`

```sql
CREATE TABLE dbo.BackupLog (
    Id INT IDENTITY(1,1) PRIMARY KEY,
    BackupDate DATETIME DEFAULT GETUTCDATE(),
    FileName NVARCHAR(255),
    Status NVARCHAR(50),
    Notes NVARCHAR(500)
);
```

### 4. Logic App Workflow
Runs daily at **2:00 AM UTC** and executes:

1. **Execute SQL Query** — `SELECT * FROM dbo.BackupLog`
2. **Create Blob** — writes results to `/backups/backup-YYYY-MM-DD.json`
3. **Send Success Email** — confirmation to team
4. **Send Failure Email** (parallel branch) — alert if blob write fails

---

## 🚀 Deployment

### Prerequisites
- Azure subscription
- Azure CLI installed (`az --version`)

### Step 1 — Deploy Lifecycle Policy
```bash
az deployment group create \
  --resource-group BackupSystem-RG \
  --template-file blob-lifecycle-policy.json \
  --parameters storageAccountName=mikekingstorage1
```

### Step 2 — Deploy Logic App
```bash
az deployment group create \
  --resource-group BackupSystem-RG \
  --template-file logicapp-daily-backup.json \
  --parameters \
      storageAccountName=mikekingstorage1 \
      storageAccountKey=<your-key> \
      notificationEmail=your@email.com
```

### Step 3 — Authorize Office 365 Connection
1. Azure Portal → Logic Apps → `daily-backup-confirmation`
2. API connections → `office365-connection` → Edit → **Authorize**
3. Sign in with Microsoft 365 account

### Step 4 — Configure SQL Firewall
In `mikekingbackupserver` → Networking:
- ✅ Allow Azure services and resources to access this server
- ✅ Add your client IP address

---

## 📁 Repository Structure

```
├── README.md
├── blob-lifecycle-policy.json          # ARM template for lifecycle rules
├── logicapp-daily-backup.json          # ARM template for Logic App
├── logicapp-updated-with-sql-and-failover.json  # Full Logic App with SQL + failure branch
└── screenshots/
    ├── containers.png                  # Blob container view
    ├── lifecycle-policy.png            # Lifecycle rules configuration
    ├── logic-app-designer.png          # Final Logic App flow
    ├── run-history.png                 # Successful run history
    └── confirmation-email.png          # Sample confirmation email
```

---

## 📧 Email Alerts

**Success:**
> Subject: `✅ Daily Backup Completed — 2026-05-29`  
> Body: Backup successfully written to `mikekingstorage1/backups`

**Failure:**
> Subject: `❌ BACKUP FAILED - Action Required`  
> Body: The daily backup job failed. Check Logic App run history immediately.

---

## 💰 Cost Estimate

| Resource | Estimated Monthly Cost |
|---|---|
| Blob Storage (LRS, Cool tier) | ~$0.01/GB |
| Azure SQL (Serverless, Dev) | ~$5–10 (free tier available) |
| Logic App (Consumption) | ~$0.000025/action run |
| **Total (light usage)** | **~$5–15/month** |

> Apply the free serverless offer on SQL for 100,000 vCore seconds/month free.

---

## 🔒 Security Notes

- Storage access key should be rotated every 90 days
- Consider switching to **Managed Identity** for keyless authentication
- SQL firewall restricts access to Azure services and your IP only
- All containers set to **Private** access level

---

## 📊 Monitoring

- **Logic App run history**: Azure Portal → `daily-backup-confirmation` → Run history
- **Blob inventory**: `mikekingstorage1` → Containers → `backups`
- **SQL query**: Run `SELECT * FROM dbo.BackupLog` in Query Editor

---

## ⚠️ Should You Delete Everything?

If this was a **learning/demo project**, yes — delete to avoid ongoing charges:

```bash
# Delete the entire resource group (removes ALL resources inside it)
az group delete --name BackupSystem-RG --yes --no-wait
```

> ⚠️ This deletes: storage account, SQL server, database, Logic App, and all data permanently.

If you want to **keep it running**, it costs roughly **$5–15/month** at light usage with the free SQL tier applied.

---

## 📸 Screenshots

Built and deployed entirely through the Azure Portal with step-by-step guidance. Key milestones:

- ✅ Blob container created and lifecycle policy applied
- ✅ SQL database and `BackupLog` table created
- ✅ Logic App wired up with SQL → Blob → Email flow
- ✅ Failure alert branch configured as parallel step
- ✅ Successful test runs confirmed via email and run history

---

*Built on Azure — May 2026*

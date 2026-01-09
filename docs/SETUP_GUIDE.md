# Azure Sales Analytics - Detailed Setup Guide

**Author**: Mehul Mittal  
**Last Updated**: January 2026

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Azure Resource Setup](#azure-resource-setup)
3. [SQL Server Configuration](#sql-server-configuration)
4. [Azure Data Factory Setup](#azure-data-factory-setup)
5. [Azure Databricks Configuration](#azure-databricks-configuration)
6. [Azure Synapse Analytics Setup](#azure-synapse-analytics-setup)
7. [Power BI Configuration](#power-bi-configuration)
8. [Testing & Validation](#testing--validation)
9. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Software
- SQL Server Management Studio (SSMS) 2019 or later
- Power BI Desktop (latest version)
- Azure CLI (optional, for command-line deployment)
- Git (for version control)

### Required Knowledge
- Basic SQL querying
- Understanding of data warehousing concepts
- Familiarity with cloud computing
- Python basics (for PySpark notebooks)

### Azure Subscription
- Active Azure subscription
- Contributor or Owner role
- Estimated monthly cost: $40-105

---

## Azure Resource Setup

### Step 1: Create Resource Group

```bash
# Using Azure CLI
az group create \
  --name sales-analytics-rg \
  --location eastus

# Or using Azure Portal:
# 1. Navigate to Resource Groups
# 2. Click "+ Create"
# 3. Enter name: sales-analytics-rg
# 4. Select region: East US
# 5. Click "Review + Create"
```

### Step 2: Create Azure Data Lake Storage Gen2

```bash
# Create storage account
az storage account create \
  --name salesanalyticsadls \
  --resource-group sales-analytics-rg \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hierarchical-namespace true

# Create containers
az storage container create \
  --name bronze \
  --account-name salesanalyticsadls

az storage container create \
  --name silver \
  --account-name salesanalyticsadls

az storage container create \
  --name gold \
  --account-name salesanalyticsadls
```

**Azure Portal Steps:**
1. Search for "Storage accounts"
2. Click "+ Create"
3. Fill in details:
   - Resource group: sales-analytics-rg
   - Storage account name: salesanalyticsadls
   - Region: East US
   - Performance: Standard
   - Redundancy: LRS
4. Advanced tab:
   - Enable "Hierarchical namespace" ✓
5. Create containers: bronze, silver, gold

### Step 3: Create Azure Key Vault

```bash
az keyvault create \
  --name sales-analytics-kv \
  --resource-group sales-analytics-rg \
  --location eastus
```

**Store secrets:**
```bash
# SQL Server password
az keyvault secret set \
  --vault-name sales-analytics-kv \
  --name sql-password \
  --value "YourSecurePassword123!"

# Service principal credentials
az keyvault secret set \
  --vault-name sales-analytics-kv \
  --name sp-client-secret \
  --value "your-client-secret"
```

### Step 4: Create Service Principal

```bash
# Create service principal
az ad sp create-for-rbac \
  --name "sales-analytics-sp" \
  --role Contributor \
  --scopes /subscriptions/<subscription-id>/resourceGroups/sales-analytics-rg

# Output will show:
# {
#   "appId": "xxxx-xxxx-xxxx-xxxx",
#   "tenant": "xxxx-xxxx-xxxx-xxxx",
#   "password": "xxxx-xxxx-xxxx-xxxx"
# }

# Save these values - you'll need them for Databricks mounting
```

### Step 5: Assign Storage Permissions

```bash
# Get service principal object ID
SP_OBJECT_ID=$(az ad sp show --id <appId> --query id -o tsv)

# Assign Storage Blob Data Contributor role
az role assignment create \
  --assignee $SP_OBJECT_ID \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/<subscription-id>/resourceGroups/sales-analytics-rg/providers/Microsoft.Storage/storageAccounts/salesanalyticsadls
```

---

## SQL Server Configuration

### Step 1: Restore AdventureWorksLT2017 Database

```sql
-- In SQL Server Management Studio (SSMS)

-- Restore database from backup file
RESTORE DATABASE AdventureWorksLT2017
FROM DISK = 'C:\Backups\AdventureWorksLT2017.bak'
WITH 
    MOVE 'AdventureWorksLT2017_Data' TO 'C:\Data\AdventureWorksLT2017.mdf',
    MOVE 'AdventureWorksLT2017_Log' TO 'C:\Data\AdventureWorksLT2017_log.ldf',
    REPLACE;
GO

-- Verify database restoration
SELECT name, database_id, create_date 
FROM sys.databases 
WHERE name = 'AdventureWorksLT2017';
```

### Step 2: Create Database User

```sql
-- Create SQL login
USE master;
GO

CREATE LOGIN usr1 WITH PASSWORD = 'SecurePass123!';
GO

-- Create database user
USE AdventureWorksLT2017;
GO

CREATE USER usr1 FOR LOGIN usr1;
GO

-- Grant read permissions
ALTER ROLE db_datareader ADD MEMBER usr1;
GO

-- Verify permissions
SELECT 
    dp.name AS UserName,
    r.name AS RoleName
FROM sys.database_principals dp
LEFT JOIN sys.database_role_members rm ON dp.principal_id = rm.member_principal_id
LEFT JOIN sys.database_principals r ON rm.role_principal_id = r.principal_id
WHERE dp.name = 'usr1';
```

### Step 3: Enable TCP/IP Protocol

1. Open "SQL Server Configuration Manager"
2. Expand "SQL Server Network Configuration"
3. Click "Protocols for MSSQLSERVER"
4. Right-click "TCP/IP" → Enable
5. Restart SQL Server service

### Step 4: Configure Firewall

```powershell
# Windows Firewall - Allow SQL Server port
New-NetFirewallRule -DisplayName "SQL Server" `
  -Direction Inbound `
  -LocalPort 1433 `
  -Protocol TCP `
  -Action Allow
```

---

## Azure Data Factory Setup

### Step 1: Create Data Factory

```bash
az datafactory factory create \
  --resource-group sales-analytics-rg \
  --factory-name sales-analytics-adf \
  --location eastus
```

**Azure Portal:**
1. Search for "Data factories"
2. Click "+ Create"
3. Fill in details and create

### Step 2: Install Self-Hosted Integration Runtime

**On your local machine:**

1. Download Integration Runtime installer from Azure Portal
2. Install the application
3. In Azure Portal:
   - Go to Data Factory → Manage → Integration runtimes
   - Click "+ New"
   - Select "Self-Hosted"
   - Copy authentication key
4. Paste key in Integration Runtime Configuration Manager
5. Verify connection status shows "Running"

### Step 3: Create Linked Services

**SQL Server Linked Service:**

```json
{
  "name": "SqlServerSource",
  "type": "SqlServer",
  "typeProperties": {
    "connectionString": {
      "type": "AzureKeyVaultSecret",
      "store": {
        "referenceName": "AzureKeyVault",
        "type": "LinkedServiceReference"
      },
      "secretName": "sql-connection-string"
    }
  },
  "connectVia": {
    "referenceName": "SelfHostedIR",
    "type": "IntegrationRuntimeReference"
  }
}
```

**ADLS Gen2 Linked Service:**

```json
{
  "name": "ADLSGen2Storage",
  "type": "AzureBlobFS",
  "typeProperties": {
    "url": "https://salesanalyticsadls.dfs.core.windows.net",
    "accountKey": {
      "type": "AzureKeyVaultSecret",
      "store": {
        "referenceName": "AzureKeyVault",
        "type": "LinkedServiceReference"
      },
      "secretName": "storage-account-key"
    }
  }
}
```

### Step 4: Create Datasets

**Source Dataset (SQL Server):**
```json
{
  "name": "SqlServerTables",
  "properties": {
    "linkedServiceName": {
      "referenceName": "SqlServerSource",
      "type": "LinkedServiceReference"
    },
    "parameters": {
      "TableName": {
        "type": "string"
      }
    },
    "type": "SqlServerTable",
    "schema": [],
    "typeProperties": {
      "schema": "SalesLT",
      "table": {
        "value": "@dataset().TableName",
        "type": "Expression"
      }
    }
  }
}
```

**Sink Dataset (ADLS Gen2):**
```json
{
  "name": "ADLSBronzeSink",
  "properties": {
    "linkedServiceName": {
      "referenceName": "ADLSGen2Storage",
      "type": "LinkedServiceReference"
    },
    "parameters": {
      "TableName": {
        "type": "string"
      }
    },
    "type": "Parquet",
    "typeProperties": {
      "location": {
        "type": "AzureBlobFSLocation",
        "fileName": "data.parquet",
        "folderPath": {
          "value": "@concat('bronze/SalesLT/', dataset().TableName)",
          "type": "Expression"
        },
        "fileSystem": "bronze"
      },
      "compressionCodec": "snappy"
    }
  }
}
```

### Step 5: Create Pipeline

**Main ETL Pipeline:**

```json
{
  "name": "MainETLPipeline",
  "properties": {
    "activities": [
      {
        "name": "GetTableNames",
        "type": "Lookup",
        "typeProperties": {
          "source": {
            "type": "SqlServerSource",
            "sqlReaderQuery": "SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = 'SalesLT'"
          },
          "dataset": {
            "referenceName": "SqlServerSource",
            "type": "DatasetReference"
          },
          "firstRowOnly": false
        }
      },
      {
        "name": "ForEachTable",
        "type": "ForEach",
        "dependsOn": [
          {
            "activity": "GetTableNames",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "items": {
            "value": "@activity('GetTableNames').output.value",
            "type": "Expression"
          },
          "activities": [
            {
              "name": "CopyTableData",
              "type": "Copy",
              "typeProperties": {
                "source": {
                  "type": "SqlServerSource"
                },
                "sink": {
                  "type": "ParquetSink"
                }
              }
            }
          ]
        }
      },
      {
        "name": "RunDatabricksNotebook",
        "type": "DatabricksNotebook",
        "dependsOn": [
          {
            "activity": "ForEachTable",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "notebookPath": "/Workspace/bronze-to-silver"
        }
      }
    ]
  }
}
```

---

## Azure Databricks Configuration

### Step 1: Create Databricks Workspace

```bash
az databricks workspace create \
  --resource-group sales-analytics-rg \
  --name sales-analytics-dbx \
  --location eastus \
  --sku standard
```

### Step 2: Create Cluster

**Cluster Configuration:**
```json
{
  "cluster_name": "sales-etl-cluster",
  "spark_version": "12.2.x-scala2.12",
  "node_type_id": "Standard_DS3_v2",
  "autoscale": {
    "min_workers": 2,
    "max_workers": 8
  },
  "autotermination_minutes": 30,
  "spark_conf": {
    "spark.databricks.delta.preview.enabled": "true"
  }
}
```

### Step 3: Mount ADLS Gen2

**Create Databricks Secret Scope:**

```bash
# Using Databricks CLI
databricks secrets create-scope --scope sales-analytics-scope

# Add secrets
databricks secrets put --scope sales-analytics-scope \
  --key client-id \
  --string-value "<service-principal-app-id>"

databricks secrets put --scope sales-analytics-scope \
  --key client-secret \
  --string-value "<service-principal-password>"

databricks secrets put --scope sales-analytics-scope \
  --key tenant-id \
  --string-value "<tenant-id>"
```

**Mount Storage (storagemount.ipynb):**

```python
# Configuration
configs = {
  "fs.azure.account.auth.type": "OAuth",
  "fs.azure.account.oauth.provider.type": "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider",
  "fs.azure.account.oauth2.client.id": dbutils.secrets.get(scope="sales-analytics-scope", key="client-id"),
  "fs.azure.account.oauth2.client.secret": dbutils.secrets.get(scope="sales-analytics-scope", key="client-secret"),
  "fs.azure.account.oauth2.client.endpoint": f"https://login.microsoftonline.com/{dbutils.secrets.get(scope='sales-analytics-scope', key='tenant-id')}/oauth2/token"
}

# Mount Bronze
dbutils.fs.mount(
  source = "abfss://bronze@salesanalyticsadls.dfs.core.windows.net/",
  mount_point = "/mnt/bronze",
  extra_configs = configs
)

# Mount Silver
dbutils.fs.mount(
  source = "abfss://silver@salesanalyticsadls.dfs.core.windows.net/",
  mount_point = "/mnt/silver",
  extra_configs = configs
)

# Mount Gold
dbutils.fs.mount(
  source = "abfss://gold@salesanalyticsadls.dfs.core.windows.net/",
  mount_point = "/mnt/gold",
  extra_configs = configs
)

# Verify mounts
display(dbutils.fs.ls("/mnt/bronze"))
```

### Step 4: Create Transformation Notebooks

See notebooks in `Data_Bricks-Notebooks/` folder for complete code.

---

## Azure Synapse Analytics Setup

### Step 1: Create Synapse Workspace

```bash
az synapse workspace create \
  --name sales-analytics-synapse \
  --resource-group sales-analytics-rg \
  --storage-account salesanalyticsadls \
  --file-system gold \
  --sql-admin-login-user sqladminuser \
  --sql-admin-login-password SecurePass123! \
  --location eastus
```

### Step 2: Configure Firewall

```bash
# Allow Azure services
az synapse workspace firewall-rule create \
  --name AllowAllWindowsAzureIps \
  --workspace-name sales-analytics-synapse \
  --resource-group sales-analytics-rg \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Allow your IP
az synapse workspace firewall-rule create \
  --name AllowMyIP \
  --workspace-name sales-analytics-synapse \
  --resource-group sales-analytics-rg \
  --start-ip-address <your-ip> \
  --end-ip-address <your-ip>
```

### Step 3: Create Database and Views

See SQL scripts in README.md for detailed code.

---

## Power BI Configuration

### Step 1: Install Power BI Desktop

Download from: https://powerbi.microsoft.com/desktop/

### Step 2: Connect to Azure Synapse

1. Open Power BI Desktop
2. Click "Get Data" → "Azure" → "Azure Synapse Analytics SQL"
3. Enter server name: `sales-analytics-synapse-ondemand.sql.azuresynapse.net`
4. Database: `gold_db`
5. Select "DirectQuery" mode
6. Authenticate with Azure AD

### Step 3: Import Views

Select the following views:
- vw_customer_analytics
- vw_product_analytics
- vw_sales_analytics
- vw_date_dimension

### Step 4: Create Data Model

Establish relationships:
- fact_sales[customer_id] → dim_customer[customer_id]
- fact_sales[product_id] → dim_product[product_id]
- fact_sales[order_date] → dim_date[date]

### Step 5: Create Measures

See DAX measures in README.md.

### Step 6: Design Dashboard

Follow the enhanced UI/UX design in the provided .pbix file.

---

## Testing & Validation

### Test 1: Pipeline Execution

```bash
# Trigger pipeline manually
az datafactory pipeline create-run \
  --resource-group sales-analytics-rg \
  --factory-name sales-analytics-adf \
  --name MainETLPipeline
```

### Test 2: Data Validation

```sql
-- Check row counts across layers
SELECT 'Bronze' as Layer, COUNT(*) as RowCount 
FROM OPENROWSET(BULK '/bronze/SalesLT/Customer', FORMAT='PARQUET') AS bronze

UNION ALL

SELECT 'Silver' as Layer, COUNT(*) as RowCount 
FROM OPENROWSET(BULK '/silver/Customer', FORMAT='DELTA') AS silver

UNION ALL

SELECT 'Gold' as Layer, COUNT(*) as RowCount 
FROM OPENROWSET(BULK '/gold/dim_customer', FORMAT='DELTA') AS gold;
```

### Test 3: Power BI Refresh

1. Open Power BI report
2. Click "Refresh"
3. Verify data updates
4. Test interactivity (cross-filtering)

---

## Troubleshooting

See main README.md for comprehensive troubleshooting guide.

---

**Last Updated**: January 2026  
**Author**: Mehul Mittal

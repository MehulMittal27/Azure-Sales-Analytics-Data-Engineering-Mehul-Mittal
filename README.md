# Azure Sales Analytics - End-to-End Data Engineering Pipeline

**Author**: Mehul Mittal  
AI/ML Engineer | Data Engineer  
📧 mehulmittal1299@gmail.com

[![Azure](https://img.shields.io/badge/Azure-Data%20Engineering-0078D4?logo=microsoft-azure)](https://azure.microsoft.com)
[![Databricks](https://img.shields.io/badge/Databricks-FF3621?logo=databricks&logoColor=white)](https://databricks.com)
[![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?logo=power-bi&logoColor=black)](https://powerbi.microsoft.com)
[![Python](https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white)](https://python.org)

---

## Project Overview

An enterprise-grade **End-to-End Azure Data Engineering Solution** implementing modern data lakehouse architecture with medallion pattern (Bronze-Silver-Gold). This project demonstrates production-ready ETL/ELT pipelines, cloud data migration, and advanced analytics using Microsoft Azure services.

### Business Problem

Organizations with on-premise databases need scalable, cloud-based analytics solutions. This project solves:
- **Data Migration**: Moving SQL Server databases to Azure cloud
- **Data Transformation**: Implementing medallion architecture for data quality
- **Real-time Analytics**: Enabling business intelligence through Power BI dashboards
- **Automation**: Scheduled pipelines for continuous data processing

### Key Achievements

- ✅ **100% Automated ETL**: Zero-touch data pipeline from source to visualization
- ✅ **Scalable Architecture**: Handles millions of records with Azure Databricks
- ✅ **Modern UI/UX**: Professional Power BI dashboard with interactive visualizations
- ✅ **Production Ready**: Implements best practices for security, monitoring, and governance

---

## System Architecture

### Data Flow Pipeline

```
On-Premise SQL Server (SSMS)
         ↓
    Integration Runtime
         ↓
Azure Data Lake Gen2 [Bronze Layer] → Raw Parquet Files
         ↓
Azure Databricks (PySpark) → Data Cleaning
         ↓
Azure Data Lake Gen2 [Silver Layer] → Clean Delta Tables
         ↓
Azure Databricks (PySpark) → Business Logic
         ↓
Azure Data Lake Gen2 [Gold Layer] → Analytics-Ready
         ↓
Azure Synapse Analytics → SQL Views
         ↓
Power BI Dashboard → Interactive Visualizations
```

---

## Technologies Used

### Cloud Services
| Service | Purpose |
|---------|---------|
| **Azure Data Factory** | Pipeline orchestration & Integration Runtime |
| **Azure Data Lake Gen2** | Hierarchical storage with medallion architecture |
| **Azure Databricks** | PySpark data transformations & Apache Spark |
| **Azure Synapse Analytics** | Serverless SQL pools & analytics |
| **Azure Key Vault** | Secrets and credentials management |
| **Microsoft Entra ID** | Access control & security groups |
| **Power BI** | Interactive dashboards & reporting |

---

## Project Structure

```
Azure-Sales-Analytics/
├── Data_Bricks-Notebooks/
│   ├── storagemount.ipynb        # ADLS mount configuration
│   ├── bronze-to-silver.ipynb    # Data cleaning transformations
│   └── silver-to-gold.ipynb      # Business logic transformations
├── PowerBI files/
│   ├── Data Reporting.pbix       # Enhanced dashboard design
│   └── Data Reporting.pbit       # Template file
├── Dataset/
│   └── AdventureWorksLT2017.bak  # Sample database
├── pix/
│   └── architecture diagrams & screenshots
└── README.md
```

---

## Implementation Guide

### Part 1: Data Ingestion (Bronze Layer)

**Setup:**
1. Restore AdventureWorksLT2017 database in SQL Server
2. Install Self-Hosted Integration Runtime
3. Create Azure Data Factory pipeline

**Source Database:**
![Source Database](./pix/SOURCE%202017LTv1.png)

**Pipeline Configuration:**
- **Source**: SQL Server (via Integration Runtime)
- **Sink**: ADLS Gen2 Bronze container
- **Format**: Parquet (optimized columnar storage)
- **Tables**: All SalesLT schema tables (Customer, Product, SalesOrder, etc.)

---

### Part 2: Data Transformation (Silver Layer)

**Bronze to Silver Notebook:**

```python
# Read from Bronze (Parquet)
bronze_df = spark.read.parquet("/mnt/bronze/SalesLT/Customer")

# Data Quality Transformations
from pyspark.sql.functions import col, trim, upper, to_date

silver_df = bronze_df \
    .dropDuplicates() \
    .na.drop(subset=["CustomerID"]) \
    .withColumn("FirstName", trim(col("FirstName"))) \
    .withColumn("EmailAddress", upper(col("EmailAddress"))) \
    .withColumn("ModifiedDate", to_date(col("ModifiedDate")))

# Write to Silver (Delta format)
silver_df.write \
    .format("delta") \
    .mode("overwrite") \
    .save("/mnt/silver/Customer")
```

**Key Transformations:**
- Remove duplicates
- Handle null values
- Standardize data types
- Trim whitespace
- Date/time formatting

---

### Part 3: Data Curation (Gold Layer)

**Silver to Gold Notebook:**

```python
# Read from Silver (Delta)
silver_df = spark.read.format("delta").load("/mnt/silver/Customer")

# Business Transformations
gold_df = silver_df \
    .select(
        col("CustomerID").alias("customer_id"),
        col("FirstName").alias("first_name"),
        col("LastName").alias("last_name"),
        concat(col("FirstName"), lit(" "), col("LastName")).alias("full_name"),
        col("EmailAddress").alias("email"),
        col("Phone").alias("phone_number")
    )

# Write to Gold
gold_df.write \
    .format("delta") \
    .mode("overwrite") \
    .save("/mnt/gold/dim_customer")
```

**Business Logic:**
- Standardized naming conventions
- Calculated fields (full_name)
- Dimensional modeling (star schema)
- Aggregations for analytics

---

### Part 4: Analytics Layer (Azure Synapse)

**Create External Tables & Views:**

```sql
-- External data source pointing to Gold layer
CREATE EXTERNAL DATA SOURCE gold_storage
WITH (LOCATION = 'abfss://gold@salesanalyticsadls.dfs.core.windows.net/');

-- Create view for Power BI
CREATE VIEW vw_customer_analytics AS
SELECT 
    customer_id,
    full_name,
    email,
    phone_number
FROM OPENROWSET(
    BULK '/dim_customer',
    DATA_SOURCE = 'gold_storage',
    FORMAT = 'DELTA'
) AS customer_data;
```

**Automated Stored Procedure:**
```sql
CREATE PROCEDURE sp_CreateOrUpdateViews @TableName NVARCHAR(100)
AS
BEGIN
    EXEC('CREATE OR ALTER VIEW vw_' + @TableName + ' AS SELECT * FROM ext_' + @TableName);
END;
```

---

### Part 5: Power BI Dashboard

#### Enhanced Dashboard Design

The dashboard has been redesigned with modern UI/UX principles:

**Design Features:**
- **Clean Layout**: Organized visual hierarchy with hero metrics
- **Azure Color Palette**: Professional blue/white theme
- **Interactive Charts**: Cross-filtering and drill-through
- **Responsive Design**: Optimized for desktop and mobile
- **Performance**: DirectQuery with optimized DAX measures

**Screenshot: Enhanced Dashboard - Overview**
![Dashboard Overview](./PowerBI%20files/PowerBI%20Reporting%20output.png)

**Key Visualizations:**

1. **KPI Cards**
   - Total Customers (847)
   - Total Products (295)
   - Revenue metrics
   - Growth indicators

2. **Sales Analysis**
   - Donut chart: Total Count Values breakdown
   - Sunburst chart: Price distribution by Color & Name
   - Revenue trends over time
   - Geographic distribution

3. **Product Performance**
   - Top 10 products by revenue
   - Category analysis
   - Color preferences
   - Inventory status

4. **Customer Insights**
   - Customer segmentation
   - Purchase patterns
   - Lifetime value analysis
   - Retention metrics

**DAX Measures:**

```dax
// Total Revenue
Total_Revenue = SUM(fact_sales[SalesAmount])

// Revenue Growth %
Revenue_Growth = 
VAR CurrentRevenue = [Total_Revenue]
VAR PreviousRevenue = CALCULATE([Total_Revenue], DATEADD(dim_date[Date], -1, YEAR))
RETURN DIVIDE(CurrentRevenue - PreviousRevenue, PreviousRevenue, 0)

// Customer Lifetime Value
Customer_LTV = SUMX(VALUES(dim_customer[CustomerID]), CALCULATE(SUM(fact_sales[SalesAmount])))
```

---

### Part 6: Pipeline Automation

**Scheduled Trigger Configuration:**

![Trigger Configuration](https://github.com/Shashi42/Azure-End-to-End-Sales-Data-Analytics-Pipeline/assets/26250463/d28f9c77-0027-4bb5-96f4-104109346f82)

**Daily ETL Schedule:**
- Runs at 2:00 AM UTC daily
- Automates: Data ingestion → Transformation → Loading
- Power BI refreshes automatically via DirectQuery

**Before Trigger:**
![Before Trigger](https://github.com/Shashi42/Azure-End-to-End-Sales-Data-Analytics-Pipeline/assets/26250463/51405c5f-331a-4bbd-83cf-439f91ca2525)

**After Trigger:**
![After Trigger](https://github.com/Shashi42/Azure-End-to-End-Sales-Data-Analytics-Pipeline/assets/26250463/578aca35-89b1-4a31-b1e0-27fb7fd923ed)

---

## Security & Governance

### Access Control
- **Microsoft Entra ID**: Role-based access control (RBAC)
- **Security Groups**: Team-based permissions
- **Azure Key Vault**: Credential management
- **Network Security**: Private endpoints for Azure services

### Data Governance
- **Data Lineage**: Track data flow through layers
- **Audit Logging**: Monitor all access and changes
- **Data Quality**: Validation rules in notebooks
- **Compliance**: GDPR-ready architecture

---

## Performance Optimization

### Databricks Optimization
```python
# Enable adaptive query execution
spark.conf.set("spark.sql.adaptive.enabled", "true")

# Cache frequently accessed tables
customer_df.cache()

# Optimize file sizes
spark.conf.set("spark.sql.files.maxPartitionBytes", "128MB")
```

### Synapse Optimization
```sql
-- Create statistics
CREATE STATISTICS stat_customer_id ON vw_customer_analytics(customer_id);

-- Materialize expensive queries
CREATE MATERIALIZED VIEW mv_monthly_sales AS
SELECT YEAR(order_date) as year, MONTH(order_date) as month, SUM(sales_amount) as total
FROM fact_sales GROUP BY YEAR(order_date), MONTH(order_date);
```

---

## Cost Optimization

**Estimated Monthly Cost:** $40-105

| Service | Monthly Cost |
|---------|-------------|
| Azure Data Factory | $5-15 |
| Azure Databricks | $20-50 |
| Azure Data Lake Gen2 | $5-10 |
| Azure Synapse Analytics | $10-30 |

**Optimization Strategies:**
- Auto-pause Databricks clusters after 30 min idle
- Use serverless SQL pools in Synapse
- Lifecycle management for ADLS (Cool tier after 30 days)
- Right-size cluster nodes based on workload

---

## Learning Outcomes

Through this project, I gained expertise in:

### Technical Skills
✅ **Azure Data Engineering**: End-to-end pipeline design  
✅ **Databricks**: PySpark for distributed data processing  
✅ **Data Modeling**: Star schema and dimensional modeling  
✅ **ETL/ELT**: Medallion architecture (Bronze-Silver-Gold)  
✅ **Power BI**: Advanced DAX and dashboard design  
✅ **SQL**: Query optimization and stored procedures  

### Cloud Architecture
✅ **Scalability**: Designing for growth and performance  
✅ **Security**: Implementing enterprise access controls  
✅ **Cost Management**: Optimizing cloud resource usage  
✅ **Monitoring**: Setting up alerts and logging  

### Software Engineering
✅ **Version Control**: Managing notebooks and code  
✅ **Documentation**: Technical writing and diagrams  
✅ **Testing**: Data validation and quality checks  
✅ **DevOps**: Automation and scheduling  

---

## Troubleshooting

### Common Issues

**Integration Runtime Connection Failed:**
- Check firewall rules
- Verify SQL Server authentication
- Test network connectivity

**Databricks Mount Fails:**
- Verify App Registration permissions
- Check Key Vault access policies
- Validate OAuth configuration

**Power BI DirectQuery Timeout:**
- Optimize Synapse views
- Create materialized views
- Increase query timeout settings

---

## Future Enhancements

### Planned Features
- [ ] Real-time streaming with Azure Event Hubs
- [ ] Machine Learning with Azure ML
- [ ] Data quality framework (Great Expectations)
- [ ] CI/CD pipeline with GitHub Actions
- [ ] Multi-region disaster recovery

### Advanced Analytics
- [ ] Customer segmentation (clustering)
- [ ] Sales forecasting (time-series)
- [ ] Anomaly detection
- [ ] Recommendation engine

---

## Use Cases

This architecture applies to:

**Industries:**
- Retail: Sales analytics, inventory management
- Finance: Transaction processing, fraud detection
- Healthcare: Patient analytics, outcomes tracking
- E-commerce: Customer behavior, recommendations

**Business Functions:**
- Sales: Pipeline tracking, forecasting
- Marketing: Campaign ROI, attribution
- Product: Feature usage, adoption
- Customer Success: Churn prediction, satisfaction

---

## Contact

**Mehul Mittal**  
AI/ML Engineer | Data Engineer  
Location: Fürth, Bavaria, Germany  
Email: mehul.mittal@example.com  
GitHub: github.com/mehulmittal

---

## License

MIT License - Copyright (c) 2026 Mehul Mittal

---

## Acknowledgments

- Microsoft Azure for cloud infrastructure
- Databricks for unified analytics platform
- AdventureWorks for sample dataset
- Data engineering community for best practices

---

*This project demonstrates production-ready Azure data engineering skills including ETL/ELT pipelines, data lakehouse architecture, distributed computing, and business intelligence.*

**Built with 💙 by Mehul Mittal**

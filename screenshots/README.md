# Screenshots Folder

This folder contains screenshots documenting the Azure Sales Analytics pipeline.

## Screenshot List

Due to the proprietary nature of Azure environments, screenshots should be added after deployment:

### Required Screenshots:

1. **01-source-database.png** - SQL Server AdventureWorksLT2017 database
2. **02-adls-gen2-setup.png** - Azure Data Lake Storage Gen2 configuration
3. **03-integration-runtime.png** - Self-hosted Integration Runtime setup
4. **04-adf-pipeline.png** - Azure Data Factory pipeline design
5. **05-bronze-layer.png** - Bronze layer data in ADLS
6. **06-storage-mount.png** - Databricks storage mount configuration
7. **07-bronze-to-silver.png** - Bronze to Silver transformation notebook
8. **08-silver-to-gold.png** - Silver to Gold transformation notebook
9. **09-gold-layer.png** - Gold layer Delta tables
10. **10-synapse-analytics.png** - Azure Synapse Analytics workspace
11. **11-synapse-pipeline.png** - Synapse pipeline for view creation
12. **12-powerbi-data-model.png** - Power BI data model relationships
13. **13-dashboard-overview.png** - Enhanced Power BI dashboard overview
14. **14-dashboard-sales.png** - Sales analysis dashboard page
15. **15-dashboard-customers.png** - Customer insights dashboard page
16. **16-dashboard-products.png** - Product performance dashboard page
17. **17-dax-measures.png** - DAX measures in Power BI
18. **18-triggers.png** - Data Factory scheduled triggers
19. **19-monitoring.png** - Azure Monitor dashboard
20. **20-pipeline-success.png** - Successful pipeline execution

## How to Add Screenshots

1. Deploy the Azure resources following the setup guide
2. Take screenshots of each component
3. Name them according to the list above
4. Place them in this folder
5. Screenshots will automatically appear in README.md

## Screenshot Guidelines

- **Resolution**: 1920x1080 or higher
- **Format**: PNG (for clarity)
- **Content**: Ensure no sensitive data (credentials, keys) is visible
- **Annotations**: Use red boxes/arrows to highlight important areas
- **Consistency**: Use same browser/theme for all screenshots

## Tools for Screenshots

- **Windows**: Snipping Tool, Win + Shift + S
- **Mac**: Cmd + Shift + 4
- **Browser Extensions**: Awesome Screenshot, Nimbus Screenshot
- **Annotation**: Greenshot, ShareX, Snagit

---

**Note**: The existing Power BI dashboard screenshots are already included in the `PowerBI files/` folder.

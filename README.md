[![Releases](https://img.shields.io/badge/Release-Download-blue?style=for-the-badge&logo=github)](https://github.com/hola22311/Business-Insights/releases)

# Global Hardware BI: Galaxy Schema, DAX, Power BI Reports Suite

[![Power BI](https://img.shields.io/badge/Power%20BI-v2.0-yellow?style=flat-square&logo=power-bi)](https://powerbi.microsoft.com)
![Data](https://img.shields.io/badge/Data-Analytics-green?style=flat-square&logo=tableau)

![Power BI Hero](https://upload.wikimedia.org/wikipedia/commons/c/cf/PowerBI-logo.png)

Contents
- Quick links
- Project snapshot
- Architecture and data model (Galaxy Schema)
- ETL / Power Query patterns
- Advanced DAX library
- Five report pages: design and logic
- Visual choices and UX
- Deployment and releases
- Installation and run instructions
- Performance and optimization
- Testing and validation
- Data governance and security
- Contribution and development workflow
- Glossary of terms
- References and learning resources
- Credits

Quick links
- Releases and download: https://github.com/hola22311/Business-Insights/releases — download and execute the release asset provided there.
- Project topics / tags: business-analytics, business-intelligence, data-analysis, data-modeling, data-visualization, dax, etl, financial-analysis, galaxy-schema, power-query, powerbi, star-schema, supply-chain-analytics

Project snapshot
This repository contains a complete business intelligence solution built for a global hardware company. The solution sits on a Tabular model in Power BI Desktop. It uses a Galaxy Schema to model complex relationships across multiple business domains. The report provides five interactive pages. Each page targets a core stakeholder group: Finance, Sales, Marketing, Supply Chain, and Executives.

Key highlights
- Galaxy Schema data model with shared lookup dimensions and separate fact groups.
- Advanced DAX measures for allocation, time intelligence, and multi-currency finance.
- Power Query ETL patterns to keep data clean and performant.
- Five report pages with role-focused KPIs, drill paths, and bookmarks.
- Deployable PBIX and dataset with clear release packages. Download and execute the release asset from the releases page: https://github.com/hola22311/Business-Insights/releases

Architecture and data model (Galaxy Schema) 🧭
This solution uses a Galaxy Schema. The Galaxy Schema groups several star schemas that share dimension tables. It fits organizations that have multiple fact tables which need shared dimensions like Date, Product, and Location.

High-level diagram
![Galaxy Schema Diagram](https://raw.githubusercontent.com/microsoft/PowerBI-visuals/master/docs/images/Architecture.png)

Core entities
- Dimensions (shared)
  - DimDate — full date calendar with fiscal flags, week/month/quarter keys.
  - DimProduct — product master with hierarchy (SKU, Model, Family, Category).
  - DimLocation — geography map with plant, region, country, and DC.
  - DimCustomer — customer master with account type, channel, and tier.
  - DimVendor — vendor master for supply chain analytics.
  - DimCurrency — currency codes and conversion rates.
  - DimEmployee — simple sales owner and finance approver table.
- Fact groups
  - FactSales — invoice-level sales with line items, price, cost, discounts.
  - FactPurchases — purchase orders and receipts for supplier analytics.
  - FactInventory — stock snapshots and movements.
  - FactFinancials — GL-level balances and journal entries for finance reconciliation.
  - FactMarketing — campaign spends, leads, and conversions.

Relationships
- Shared dimensions connect to multiple fact tables via key fields.
- Bridge tables handle many-to-many relationships where required (for example, product bundles and multi-SKU promotions).
- Role-playing Date tables allow independent time analysis for sales date, invoice date, ship date, receipt date, and GL posting date.
- Use surrogate keys for all joins to avoid NULL alignment issues.

Modeling rules used
- Keep facts narrow and tall. Avoid wide tables with sparse columns.
- Separate transactional facts (FactSales) from periodic snapshot facts (FactInventory).
- Use conformed dimensions across facts to support cross-fact joins and measures.
- Apply grain enforcement: each fact has a well-defined grain and a unique key.
- Add low-cardinality columns to dimension tables to support slicers and filter panes.

Data quality and design notes
- Normalize master data for product and location to avoid duplicate rows.
- Use natural keys for traceability but join with surrogate keys for performance.
- Flag slow-changing attributes and create type-2 history for attributes that change over time when the business needs see historical context.
- Add soft deletes or active flags rather than purging rows for auditability.

ETL / Power Query patterns 🔁
Power Query drives the ETL layer inside the PBIX and for any external dataset refresh pipelines. The solution uses a layered ETL approach: source extract → staging transforms → conformed entities → modeled dataset.

Common ETL steps
- Source extraction
  - Use native connectors where possible (SQL Server, Oracle, SAP, flat files, REST APIs).
  - Capture extract metadata: timestamp, row count, source checksum.
- Staging transformations
  - Rename columns to standard names.
  - Trim and cleanse strings.
  - Cast data types explicitly.
  - Replace sentinel values (e.g., 'N/A', 'NULL') with real nulls.
  - Use query folding when possible to push transforms to the source.
- Conformed entities
  - Merge duplicate masters using fuzzy matching where needed.
  - Deduplicate facts by using last update timestamp.
  - Generate surrogate keys using index columns.
  - Create Date table with fiscal weeks and rolling flags.
- Load and optimize
  - Reduce column count before loading into the model.
  - Convert binary or JSON into scalar columns where possible.
  - Keep calculated columns to a minimum; prefer measures.

Power Query tips
- Preserve query folding: apply filter and projection steps early.
- Use Table.Buffer sparingly and only when necessary to break folding.
- Use function queries for repeated logic (for example, standardize phone format or parse product codes).
- Build incremental refresh policies for large fact tables to limit refresh time and cost.

Sample M pattern (pseudo)
let
  Source = Sql.Database("dbserver", "db"),
  Sales = Source{[Schema="dbo", Item="InvoiceLines"]}[Data],
  Renamed = Table.RenameColumns(Sales, {{"InvDate", "InvoiceDate"}, {"SKU", "ProductKey"}}),
  Cleaned = Table.TransformColumns(Renamed, {{"Price", each Number.Round(_, 2)}}),
  Filtered = Table.SelectRows(Cleaned, each [IsVoid] = false),
  AddedKey = Table.AddIndexColumn(Filtered, "SurrogateKey", 1, 1, Int64.Type)
in
  AddedKey

Advanced DAX library 🧮
This solution packs a DAX library to address finance-level aggregation, time-based comparatives, allocation, and advanced segmentation.

DAX patterns included
- Time intelligence
  - Custom YearToDate, QuarterToDate, MonthToDate that respect fiscal calendars.
  - Parallel period comparators for YoY and QoQ with fiscal support.
- Currency conversion
  - Dynamic currency conversion using DimCurrency rates with daily rate lookup.
  - Measures that allow users to switch report currency via a slicer.
- Allocation and proportional measures
  - Cost allocation across departments based on driver tables (e.g., headcount, area).
  - Weighted average cost across SKU families.
- Inventory metrics
  - Days of Supply: uses lead time and average daily consumption.
  - Stock out rate: measures periods below safety stock.
- Sales analytics
  - Net revenue, gross margin, and channel profitability measures.
  - Churn and cohort analysis measures for customer retention.
- Custom rolling aggregates
  - Rolling 12 months and variable window metrics using DATESINPERIOD and custom filters.
- Performance-safe measures
  - Use SUMX over filtered tables only when necessary.
  - Replace EARLIER patterns with variables and iterators.

Sample measures
Total Revenue =
SUM(FactSales[NetAmount])

Revenue YTD (Fiscal) =
VAR _fiscalYearStart = MAX(DimDate[FiscalYearStart])
RETURN
CALCULATE([Total Revenue], DATESBETWEEN(DimDate[Date], _fiscalYearStart, MAX(DimDate[Date])))

Currency Converted Sales =
VAR _rate = LOOKUPVALUE(DimCurrency[Rate], DimCurrency[Date], MAX(DimDate[Date]), DimCurrency[CurrencyCode], SELECTEDVALUE(SlicerCurrency[CurrencyCode]))
RETURN
[Total Revenue] * _rate

Allocation to Departments =
VAR TotalDriver = SUM(DriverTable[DriverValue])
VAR DeptDriver = SUMX(FILTER(DriverTable, DriverTable[Department] = SELECTEDVALUE(Department[Department])), DriverTable[DriverValue])
RETURN
IF(TotalDriver = 0, BLANK(), DeptDriver / TotalDriver * [Total Cost])

Design principles for DAX
- Use variables to break down complex logic and to reduce repeated evaluation.
- Favor measure branching: build base measures and derive complex ones from them.
- Avoid iterator-heavy patterns on large tables; use summarization tables where needed.
- Test measures with DAX Studio and PROFILE functions for heavy scans.

Five report pages — design and logic 📊
The report contains five pages. Each page focuses on core needs and follows a consistent layout: top KPIs, trend charts, detailed tables, and action widgets.

1) Finance page 💰
Purpose: Reconcile financials with transactional data and surface risk.
Main visuals
- Top KPIs: Net Revenue, Gross Margin, Operating Expenses, Net Income.
- Waterfall: movement from Gross Revenue to Net Income.
- GL Reconciliation table: links FactFinancials to FactSales and FactPurchases.
- Currency toggle to present values in local or consolidated currency.
- Drill-through to invoice-level transactions.
Business logic
- Use FactFinancials as a ground truth and reconcile revenue and cost at month-end.
- Provide variance measures (Actual vs Budget, Actual vs Forecast).
- Include cash flow indicators and AR / AP aging snapshots.

2) Sales page 🛒
Purpose: Track performance by product, channel, region, and sales rep.
Main visuals
- KPI grid: Sales, Units, Average Price, Win Rate.
- Geographic map with sales density.
- Product hierarchy slicer with Top N products table.
- Sales funnel chart from lead to closed-won.
- Sales rep leaderboard with quota attainment.
Business logic
- Use FactSales and DimProduct for SKU-level analysis.
- Apply customer segmentation logic to split accounts by tier.
- Support cross-highlight between map and product table.

3) Marketing page 📣
Purpose: Show campaign ROI and lead-to-revenue path.
Main visuals
- Campaign spend vs conversions chart.
- Cohort chart for lead aging and conversion velocity.
- Attribution table with multi-touch logic.
- Paid vs organic funnel and cost per acquisition.
Business logic
- Use FactMarketing for spends and leads.
- Connect campaigns to sales via UTM tags and lead keys.
- Provide multi-touch attribution measures and last-touch fallbacks.

4) Supply Chain page 🚚
Purpose: Monitor inventory health, supplier performance, and logistics.
Main visuals
- Inventory heatmap by warehouse and SKU.
- Days of Supply and Stockout trends.
- Supplier lead-time analysis and on-time delivery metric.
- Purchase order aging and receipts vs orders chart.
Business logic
- Use FactInventory for snapshots and FactPurchases for receipts.
- Compute projected stock levels using consumption forecasts.
- Highlight items with slow turns and high carrying cost.

5) Executive page 📈
Purpose: Provide a one-page summary for leadership with drill paths.
Main visuals
- High-level KPIs and trend sparkline tiles.
- Cross-domain summary: finance, sales, supply chain, marketing.
- Scenario switcher that applies alternate forecasts.
- Bookmark set for common executive views.
Business logic
- Use aggregated measures and contextual slicers.
- Offer quick access to detailed pages via drill-through and report tooltips.

Visual choices and UX 🎨
- Use consistent color palette aligned with corporate branding.
- Employ accessible color contrast and tooltips for data points.
- Limit the number of visuals per page to reduce cognitive load.
- Use small multiples for rapid comparison across dimensions.
- Use conditional formatting to surface exceptions in tables.
- Create an interactive filter pane with pinned slicers for cross-report control.
- Use report bookmarks and buttons to guide users to common actions.

Custom visuals and images
- Prefer native visuals for performance.
- Use certified custom visuals only when native visuals cannot meet design needs.
- Images and icons used in the report come from licensed icon packs and the company asset library.

Deployment and releases 🚀
The release channel contains PBIX and deployment artifacts. The releases page hosts the final PBIX file, documentation, and optional SQL scripts.

Important: the release link includes a path. Download the release asset from https://github.com/hola22311/Business-Insights/releases and execute the package as described in the release notes. The package contains:
- Business-Insights.pbix — the Power BI Desktop file.
- README-release.md — release notes and known issues.
- config.sample.json — environment configuration for dataset connection.
- migration-scripts.sql — optional scripts to seed reporting schemas.

Release badge
[![Download Release](https://img.shields.io/badge/Download%20Release-v1.0.0-blue?style=for-the-badge&logo=github)](https://github.com/hola22311/Business-Insights/releases)

Installation and run instructions ⚙️
Prerequisites
- Power BI Desktop (latest stable release).
- Access to source systems or a test dataset provided in the release.
- Optional: Power BI Premium or Pro workspace for publishing and scheduled refresh.

Local use (open PBIX)
1. Download the PBIX from the releases page: https://github.com/hola22311/Business-Insights/releases
2. Open Business-Insights.pbix in Power BI Desktop.
3. Open File → Options → Preview features and enable any required features listed in the release notes.
4. Edit data source credentials for each data source in Transform Data → Data source settings.
5. If you do not have production data, use the bundled sample database or mock dataset.
6. Refresh the dataset. If you see long refresh times, enable Query Diagnostics to track expensive steps.

Publish to Power BI Service
1. Save the PBIX.
2. Sign in to Power BI Service with your workspace access.
3. Publish → choose target workspace.
4. Configure dataset settings: gateway, credentials, scheduled refresh frequency.
5. Assign workspace roles and publish the report as an App for wider consumption.

Connection modes
- Live connection to Analysis Services for enterprise Tabular models.
- Import mode for PBIX packaged datasets.
- Composite models for combining import and direct query sources where necessary.

Performance and optimization ⚡
Model sizing
- Monitor model size in Power BI Desktop Model view.
- Remove unused columns and tables.
- Use integer surrogate keys and avoid GUID keys where possible.

Measures and query performance
- Replace highly selective filters with summarization tables.
- Avoid row-level functions inside large iterators.
- Use SUMMARIZECOLUMNS for grouped queries where appropriate.
- Use storage mode settings and aggregations.

Refresh optimization
- Configure incremental refresh for large fact tables with a sensible partition window.
- Use parallel loading for independent tables.
- Avoid full refresh for large history when not needed.

Indexing and source-side improvements
- Ensure source tables include indexes on join keys.
- Push computation to the source using query folding.
- Pre-aggregate heavy joins in a staging schema if repeated often.

Testing and validation ✅
Data validation
- Use reconciliations between FactFinancials and FactSales to validate totals.
- Implement row counts and checksum validation during ETL.
- Use a set of validation queries that run after refresh to verify critical KPIs.

Measure testing
- Create a DAX test workbook with expected values for sample slices.
- Use Tabular Editor to run measure unit tests where possible.
- Use DAX Studio to profile query execution time and examine query plans.

Automated checks
- Use CI pipelines to validate schema changes (for example, run JSON schema checks for config files).
- Use Power BI REST API to automate dataset deployment and run refreshes.

Data governance and security 🔒
Row-level security (RLS)
- Implement RLS for operational data with role tables.
- Use USERPRINCIPALNAME or USERNAME() to map logged-in users to roles.
- Test RLS with the "View as role" feature in Power BI Desktop.

Column-level security and sensitivity
- Mark sensitive columns with sensitivity labels in Power BI Service.
- Use Power BI admin settings to restrict export and sharing where required.

Audit and lineage
- Use dataset lineage features to map report → dataset → data source relationships.
- Maintain a change log for model changes and schema versions.

Data retention and archival
- Keep historic transaction data in a long-term store and expose summaries to the model.
- Implement purge policies with audit trails.

Troubleshooting and common fixes 🛠️
High refresh time
- Identify heavy queries with Query Diagnostics.
- Move heavy joins to the source or staging schema.
- Reduce the number of visuals that run on load.

Measure returns blanks
- Check for filter context mismatches.
- Verify joins and key uniqueness.
- Ensure measures handle DIVIDE by zero using DIVIDE function.

Incorrect totals in visuals
- Confirm that the model uses the correct granularity.
- Use measures with explicit aggregations rather than implicit sums on calculated columns.
- Check for inactive relationships that need USERELATIONSHIP in measures.

Gateway authentication issues
- Update gateway credentials and ensure privacy levels allow connections.
- Use a dedicated service account for gateway connections.

Contribution and development workflow 🤝
How to contribute
- Fork the repository.
- Create a feature branch with a descriptive name.
- Commit changes with clear messages: what and why.
- Open a pull request describing the change and any model impact.

Guidelines
- Keep measure names and folders consistent.
- Add documentation for any new table, measure, or visual.
- Use tags in commit messages for releases and breaking changes.

Review process
- Peer review for new measures and performance-sensitive changes.
- Approve model changes after passing test validations in DAX Studio.
- Merge only after a successful data refresh in a staging environment.

Branching strategy
- main: stable release artifacts and tagged PBIX.
- dev: active development and feature branches.
- feature/*: experimental changes or new pages.

Analytics CI/CD (optional)
- Use scripts to export model metadata and run pre-flight checks.
- Automate PBIX publishing with Power BI REST API in CI pipelines.
- Run dataset refresh and smoke tests in staging.

Glossary 📘
- Galaxy Schema: A multi-star schema with shared dimensions across multiple fact groups.
- Fact: A table that stores measures and numeric events at a defined grain.
- Dimension: A descriptive table that provides context to measures.
- DAX: Data Analysis Expressions, the formula language used in Power BI.
- Query folding: The process by which Power Query pushes transformations back to the data source.
- RLS: Row-Level Security, a way to restrict data access based on user identity.
- PBIX: Power BI Desktop report file.
- Composite model: A Power BI model that combines import and direct query tables.

References and learning resources 📚
- Power BI documentation — https://docs.microsoft.com/power-bi
- DAX Guide — https://dax.guide/
- SQL Server documentation for query folding concepts.
- Tabular Editor — advanced modeling and scripting.
- DAX Studio — query profiling and performance tuning.

Images and assets credits
- Power BI logo — Wikimedia Commons.
- Architecture diagrams — Microsoft sample assets.
- Icons — public domain and licensed icon packs.

License
This repository includes code and sample assets. Check the LICENSE file in the repository for exact terms.

If you cannot find a release or the provided link does not work, check the "Releases" section on the project page for the latest assets and install notes.

Contact and support
- Use GitHub issues to report bugs or request features.
- Create clear issue templates for model bugs, PBIX errors, and data discrepancies.

Appendix: Sample DAX snippets and patterns
1) Rolling 12 months revenue
Rolling 12M Revenue =
CALCULATE(
  [Total Revenue],
  DATESINPERIOD(DimDate[Date], MAX(DimDate[Date]), -12, MONTH)
)

2) Dynamic currency selection
Selected Currency Rate =
VAR _currency = SELECTEDVALUE(SlicerCurrency[CurrencyCode], "USD")
RETURN
CALCULATE(AVERAGE(DimCurrency[Rate]), DimCurrency[CurrencyCode] = _currency, DimCurrency[Date] = MAX(DimDate[Date]))

3) Customer cohort retention (simplified)
Cohort Month =
MINX(VALUES(FactSales[OrderDate]), FactSales[OrderDate])

Retention Rate =
VAR CohortStart = [Cohort Month]
RETURN
CALCULATE(
  DIVIDE(DISTINCTCOUNT(FactSales[CustomerKey]), [Cohort Customers]),
  FILTER(ALL(DimDate), DimDate[Date] >= CohortStart)
)

Appendix: ETL checklist
- Confirm source schema and column types.
- Create staging schema and test reconciliations.
- Implement row counts and checksums.
- Build conformed dimension merges.
- Test incremental refresh for all large tables.
- Document transform logic in repo.

Maintenance schedule
- Monthly: refresh and reconcile key KPIs.
- Quarterly: review model for unused fields and remove them.
- Annually: refresh fiscal calendar and budge forecasts.

Governance tasks
- Review access groups and RLS mappings quarterly.
- Archive older PBIX versions and keep releases tagged.
- Document any data model breaking changes in release notes.

Troubleshooting appendix
- If PBIX fails to open due to version mismatch, update Power BI Desktop to the version used to create the PBIX.
- If a scheduled refresh fails with authentication errors, re-enter credentials in dataset settings and test connection.
- If a visual times out in service, reduce the number of visuals on the page or add filters to limit initial query size.

Deployment checklist
- Validate PBIX opens locally and refreshes with production credentials.
- Update dataset parameters and connection strings in config.sample.json.
- Publish to staging workspace and run full refresh.
- Run reconciliation and smoke tests.
- Promote to production and update release tag.

Credits and acknowledgements
- Built with Power BI and open source tooling.
- Modeling patterns inspired by industry best practices in analytics and finance.
- Visual and UX guidance from internal design system assets.

If you need the release file, download and execute the package from the releases page: https://github.com/hola22311/Business-Insights/releases

End of README content.
---
title: "Move data between databases and Excel with DbaClientX and PSWriteOffice"
description: "Practical PowerShell examples for exporting database rows to Excel, importing reviewed workbooks back to database staging tables, and choosing the fast path when moving tabular data."
date: "2026-05-23"
language: "en"
authors:
  - przemyslaw-klys
categories:
  - PowerShell
  - Databases
  - Excel
tags:
  - dbaclientx
  - pswriteoffice
  - officeimo
  - excel
  - sql-server
  - powershell
image: "./cover.png"
image_alt: "Database rows moving through PowerShell into an Excel workbook and back"
draft: true
---

PowerShell is often the glue between systems that were never designed to talk to each other. A database stores operational data, Excel is where people review and correct that data, and sooner or later somebody needs a reliable path both ways.

[DbaClientX](https://github.com/EvotecIT/DbaClientX) is a lightweight database client for PowerShell and .NET. It can query SQL Server, PostgreSQL, MySQL, Oracle, and SQLite, and it exposes provider bulk insert paths for fast table loading.

[PSWriteOffice](https://github.com/EvotecIT/PSWriteOffice) creates and reads Office files from PowerShell. For this workflow, the important part is Excel: export rows into real `.xlsx` workbooks, import reviewed sheets back as PowerShell objects or `DataTable`, and do it without requiring Microsoft Excel on the machine.

[OfficeIMO](https://github.com/EvotecIT/OfficeIMO) is the engine underneath PSWriteOffice. It owns the workbook implementation so the PowerShell commands can stay concise while the same workflow scales from a quick export to explicit workbook composition.

The result is a practical data movement story:

- You want database rows in Excel? Query with DbaClientX, export with PSWriteOffice.
- You want a reviewed workbook back in a database? Import with PSWriteOffice, bulk write with DbaClientX.
- You want the fast path? Move tabular data as `DataTable` or `IDataReader` instead of reshaping every row by hand.
- You want the flexible path? Use normal PowerShell objects and let the commands convert them.

## Connect to the database

For SQL Server with integrated security, let DbaClientX build the connection string:

```powershell
$connectionString = New-DbaXConnectionString `
    -Provider SqlServer `
    -Server 'sql01' `
    -Database 'Operations' `
    -Ssl
```

For SQL Server with a credential, DbaClientX query cmdlets can accept `-Credential`:

```powershell
$credential = Get-Credential

Invoke-DbaXQuery `
    -Server 'sql01' `
    -Database 'Operations' `
    -Credential $credential `
    -Query 'SELECT TOP 10 Id, Name, Status FROM dbo.WorkQueue'
```

For providers that require a database login, retrieve a `PSCredential` from your normal secret store and give it to the same builder:

```powershell
$databaseCredential = Get-Secret -Name 'Operations-Database-Credential'

$postgresConnectionString = New-DbaXConnectionString `
    -Provider PostgreSql `
    -Server 'pg01' `
    -Database 'Operations' `
    -Credential $databaseCredential `
    -Ssl
```

SQLite needs no credential:

```powershell
$sqliteConnectionString = New-DbaXConnectionString `
    -Provider SQLite `
    -Database '.\operations.db'
```

`Get-Secret` comes from Microsoft.PowerShell.SecretManagement. In CI, the credential or connection string can come from the runner's secret provider instead. The important boundary is that passwords do not live in the article, script, repository, or command history.

## Export database rows to Excel

If you want a workbook that people can open, filter, review, and send back, query the database and export the result as an Excel table:

```powershell
Import-Module DbaClientX
Import-Module PSWriteOffice

$connectionString = New-DbaXConnectionString -Provider SqlServer -Server 'sql01' -Database 'Operations' -Ssl

$rows = Invoke-DbaXQuery `
    -Server 'sql01' `
    -Database 'Operations' `
    -Query 'SELECT Id, Name, Status, ModifiedUtc FROM dbo.WorkQueue' `
    -ReturnType DataTable

Export-OfficeExcel `
    -InputObject $rows `
    -Path .\WorkQueue.xlsx `
    -WorksheetName 'Work Queue' `
    -TableName 'WorkQueue' `
    -AutoFit `
    -FreezeTopRow
```

That produces a native `.xlsx` workbook, not a CSV renamed to Excel. The table can be filtered, formatted, and opened by Excel without requiring Excel on the machine that created it.

For smaller jobs, object pipelines are fine:

```powershell
Invoke-DbaXQuery `
    -Server 'sql01' `
    -Database 'Operations' `
    -Query 'SELECT Id, Name, Status FROM dbo.WorkQueue' |
    Export-OfficeExcel `
        -Path .\WorkQueue-Simple.xlsx `
        -WorksheetName 'Work Queue' `
        -TableName 'WorkQueue' `
        -AutoFit
```

That is the flexible path. It is easy to read and easy to extend with `Where-Object`, `Select-Object`, and calculated properties.

For large exports, prefer tabular input such as `DataTable` or `IDataReader`. That gives the Excel engine a shape it can write efficiently and avoids paying the cost of rebuilding the table from loose PowerShell objects.

## Import Excel rows to a database table

When the workbook comes back from review, import it as a `DataTable` and write it to a staging table:

```powershell
Import-Module DbaClientX
Import-Module PSWriteOffice

$connectionString = New-DbaXConnectionString -Provider SqlServer -Server 'sql01' -Database 'Operations' -Ssl

$table = Import-OfficeExcel `
    -Path .\WorkQueue-Reviewed.xlsx `
    -WorksheetName 'Work Queue' `
    -AsDataTable

$table | Write-DbaXTableData `
    -Provider SqlServer `
    -ConnectionString $connectionString `
    -DestinationTable 'dbo.WorkQueue_Stage' `
    -BatchSize 5000 `
    -PassThru
```

The important detail is `-AsDataTable`. That gives DbaClientX a tabular shape it can send to provider-native bulk insert APIs.

From there, validate and merge in SQL:

```powershell
Invoke-DbaXNonQuery `
    -Server 'sql01' `
    -Database 'Operations' `
    -Query @'
MERGE dbo.WorkQueue AS target
USING dbo.WorkQueue_Stage AS source
    ON target.Id = source.Id
WHEN MATCHED THEN
    UPDATE SET
        target.Name = source.Name,
        target.Status = source.Status,
        target.ModifiedUtc = SYSUTCDATETIME();
'@
```

Staging tables are the recommended default. They give you a place to validate types, required columns, row counts, and business rules before changing production data.

## Write directly when the workflow is trusted

Sometimes the workbook is not a human review artifact. Maybe it is generated by another system, already validated, and part of a repeatable import job. In that case, writing directly to the destination table can be acceptable:

```powershell
$table = Import-OfficeExcel `
    -Path .\ApprovedProducts.xlsx `
    -WorksheetName 'Products' `
    -AsDataTable

$table | Write-DbaXTableData `
    -Provider SqlServer `
    -ConnectionString $connectionString `
    -DestinationTable 'dbo.Products' `
    -BatchSize 10000 `
    -BulkCopyTimeout 120
```

If humans edited the workbook, use staging. If automation produced and validated the workbook, direct writes can be the shorter path.

## Use the same shape for other providers

The same command shape can target other providers by changing the provider and connection string:

```powershell
$customers = Import-OfficeExcel `
    -Path .\Customers.xlsx `
    -WorksheetName Customers `
    -AsDataTable

$customers | Write-DbaXTableData `
    -Provider PostgreSql `
    -ConnectionString $postgresConnectionString `
    -DestinationTable 'public.customer_stage' `
    -BatchSize 5000
```

SQLite is useful for local tools, test fixtures, and portable handoff files:

```powershell
$audit = Import-OfficeExcel `
    -Path .\Audit.xlsx `
    -WorksheetName Events `
    -AsDataTable

$audit | Write-DbaXTableData `
    -Provider SQLite `
    -ConnectionString 'Data Source=C:\Data\audit.db' `
    -DestinationTable 'audit_events' `
    -BatchSize 1000
```

For SQL Server, PostgreSQL, MySQL, Oracle, and SQLite, the workflow stays the same: import or build tabular data, choose the provider, give DbaClientX the connection string, and write to the target table.

## Pick the fast path or the flexible path

There are two practical ways to move data.

Use the flexible path when the dataset is small or you need PowerShell transformations:

```powershell
Invoke-DbaXQuery -Server 'sql01' -Database 'Operations' -Query 'SELECT Id, Status FROM dbo.WorkQueue' |
    Where-Object Status -ne 'Closed' |
    Select-Object Id, Status, @{ Name = 'ExportedUtc'; Expression = { [DateTime]::UtcNow } } |
    Export-OfficeExcel -Path .\OpenWork.xlsx -WorksheetName Open -TableName OpenWork
```

Use the faster tabular path when the dataset is large or the script runs often:

```powershell
$rows = Invoke-DbaXQuery `
    -Server 'sql01' `
    -Database 'Operations' `
    -Query 'SELECT Id, Status, ModifiedUtc FROM dbo.WorkQueue' `
    -ReturnType DataTable

Export-OfficeExcel `
    -InputObject $rows `
    -Path .\WorkQueue.xlsx `
    -WorksheetName 'Work Queue' `
    -TableName 'WorkQueue' `
    -AutoFit
```

The commands still look like normal PowerShell, but the data stays in a form that database and workbook libraries can process efficiently.

## Common scenarios

This pattern fits a lot of everyday operational work:

- Replace CSV handoffs with real Excel workbooks.
- Export SQL Server review queues to business owners.
- Load reviewed workbook changes into staging tables.
- Build audit handoff workbooks from PostgreSQL, MySQL, Oracle, SQLite, or SQL Server.
- Use SQLite as a local database for workbook-driven tools.
- Keep import/export scripts reusable instead of rewriting them per report.

The short version:

```powershell
# Database to Excel
$rows = Invoke-DbaXQuery ... -ReturnType DataTable
Export-OfficeExcel -InputObject $rows -Path .\Data.xlsx -WorksheetName Data -TableName Data

# Excel to database
$table = Import-OfficeExcel -Path .\Data.xlsx -WorksheetName Data -AsDataTable
$table | Write-DbaXTableData -Provider SqlServer -ConnectionString $connectionString -DestinationTable dbo.Data_Stage
```

That is the workflow: query, export, review, import, bulk write, validate, merge.

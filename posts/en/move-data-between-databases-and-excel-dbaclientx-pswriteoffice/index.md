---
title: "Move data between databases, CSV, and Excel with DbaClientX and PSWriteOffice"
description: "Practical PowerShell examples for moving database rows through CSV and Excel, loading reviewed files into staging tables, and choosing between objects, DataTable, and streaming reader paths."
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
  - mailozaurr
  - powershell
image: "./cover.png"
image_alt: "Analyst reviewing a workbook while data moves between two database systems"
draft: true
---

PowerShell is often the glue between systems that were never designed to talk to each other. A database stores operational data, CSV is the common machine-to-machine handoff, Excel is where people review and correct data, and sooner or later somebody needs reliable paths in both directions.

[DbaClientX](https://github.com/EvotecIT/DbaClientX) is a provider-neutral database client for PowerShell and .NET. It can query SQL Server, PostgreSQL, MySQL, Oracle, and SQLite, and it exposes provider-native bulk insert paths for tabular data.

[PSWriteOffice](https://github.com/EvotecIT/PSWriteOffice) creates and reads Office files from PowerShell. For this workflow, the important parts are Excel and CSV: export rows into real `.xlsx` workbooks or bounded-delimiter files, import reviewed data as PowerShell objects, `DataTable`, or `IDataReader`, and do it without requiring Microsoft Excel on the machine.

[OfficeIMO](https://github.com/EvotecIT/OfficeIMO) is the engine underneath PSWriteOffice. It owns the workbook implementation so the PowerShell commands can stay concise while the same workflow scales from a quick export to explicit workbook composition.

The result is a practical data movement story:

- You want database rows in Excel? Query with DbaClientX, export with PSWriteOffice.
- You want a CSV handoff without turning every row into a `PSCustomObject`? Pass the same reader to `Export-OfficeCsv`.
- You want a reviewed workbook back in a database? Import with PSWriteOffice, bulk write with DbaClientX.
- You want to avoid materializing every row as a PowerShell object? Hand an `IDataReader` directly from the database client to the workbook writer.
- You want the flexible path? Use normal PowerShell objects and let the commands convert them.

## How this fits with dbatools and ImportExcel

This is not an argument that existing scripts need to move.

[dbatools](https://github.com/dataplat/dbatools) is an established SQL Server automation toolkit. If a job already uses its instance, backup, migration, or administrative commands, keeping data movement in that ecosystem may be the clearest choice. DbaClientX becomes interesting when the same provider-neutral data contract needs to work from PowerShell and .NET, or when a live reader should pass directly into another library.

[ImportExcel](https://github.com/dfinke/ImportExcel) is the familiar choice for many PowerShell-to-Excel scripts. If `Export-Excel` and `Import-Excel` already produce the workbook you need, there is no migration prize for changing them. PSWriteOffice is useful when the workbook is one part of a wider Word, PowerPoint, PDF, CSV, or document-inspection workflow, or when direct `DataTable` and `IDataReader` handoffs matter.

The modules also compose. You can keep dbatools for the surrounding SQL Server job, use ImportExcel for a workbook that already depends on it, and introduce DbaClientX or PSWriteOffice only where their public contracts solve a specific problem.

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

$databaseServer = 'sql01'
$databaseName = 'Operations'
$trustServerCertificate = $false

$connectionString = New-DbaXConnectionString `
    -Provider SqlServer `
    -Server $databaseServer `
    -Database $databaseName `
    -Ssl `
    -TrustServerCertificate:$trustServerCertificate

$reader = Invoke-DbaXQuery `
    -Server $databaseServer `
    -Database $databaseName `
    -TrustServerCertificate:$trustServerCertificate `
    -Query 'SELECT Id, Name, Status, ModifiedUtc FROM dbo.WorkQueue' `
    -AsDataReader

try {
    Export-OfficeExcel `
        -InputObject $reader `
        -Path .\WorkQueue.xlsx `
        -WorksheetName 'Work Queue' `
        -TableName 'WorkQueue' `
        -AutoFit `
        -FreezeTopRow
} finally {
    $reader.Dispose()
}
```

The query uses DbaClientX's direct SQL Server parameter set, which enables encryption and applies the explicit certificate-trust decision. The generated connection string is kept for the streaming example and the later bulk-write path.

That produces a native `.xlsx` workbook, not a CSV renamed to Excel. The table can be filtered, formatted, and opened by Excel without requiring Excel on the machine that created it.

For smaller jobs, object pipelines are fine:

```powershell
Invoke-DbaXQueryStream `
    -Provider SqlServer `
    -ConnectionString $connectionString `
    -Query 'SELECT Id, Name, Status FROM dbo.WorkQueue' |
    Export-OfficeExcel `
        -Path .\WorkQueue-Simple.xlsx `
        -WorksheetName 'Work Queue' `
        -TableName 'WorkQueue' `
        -AutoFit
```

That is the flexible path. It is easy to read and easy to extend with `Where-Object`, `Select-Object`, and calculated properties.

For a pass-through SQL Server export, the owned `IDataReader` above keeps the rows streaming until PSWriteOffice has consumed them. Dispose it in `finally` because the reader owns the live command and connection. Use `DataTable` instead when the script needs to inspect, validate, or reshape the complete dataset before writing the workbook.

## Import Excel rows to a database table

When the workbook comes back from review, import it as a `DataTable` and write it to a staging table:

```powershell
Import-Module DbaClientX
Import-Module PSWriteOffice

$databaseServer = 'sql01'
$databaseName = 'Operations'
$trustServerCertificate = $false

$connectionString = New-DbaXConnectionString `
    -Provider SqlServer `
    -Server $databaseServer `
    -Database $databaseName `
    -Ssl `
    -TrustServerCertificate:$trustServerCertificate

$table = Import-OfficeExcel `
    -Path .\WorkQueue-Reviewed.xlsx `
    -WorksheetName 'Work Queue' `
    -AsDataTable

$duplicateIds = @($table.Rows | Group-Object Id | Where-Object Count -gt 1)
if ($duplicateIds.Count -gt 0) {
    throw "The workbook contains duplicate WorkQueue IDs: $($duplicateIds.Name -join ', ')."
}

$stageTable = 'dbo.WorkQueue_Stage_' + ([guid]::NewGuid()).ToString('N')

try {
    $writeResult = $table | Write-DbaXTableData `
        -Provider SqlServer `
        -ConnectionString $connectionString `
        -DestinationTable $stageTable `
        -AutoCreateTable `
        -BatchSize 5000 `
        -PassThru

    $mergeResult = Invoke-DbaXNonQuery `
        -Server $databaseServer `
        -Database $databaseName `
        -TrustServerCertificate:$trustServerCertificate `
        -Query @"
MERGE dbo.WorkQueue WITH (HOLDLOCK) AS target
USING $stageTable AS source
    ON target.Id = source.Id
WHEN MATCHED THEN UPDATE SET
    target.Name = source.Name,
    target.Status = source.Status,
    target.ModifiedUtc = SYSUTCDATETIME()
WHEN NOT MATCHED BY TARGET THEN INSERT (Id, Name, Status, ModifiedUtc)
    VALUES (source.Id, source.Name, source.Status, SYSUTCDATETIME());
"@

    [pscustomobject]@{
        StagedRows = $writeResult.Rows
        ChangedRows = $mergeResult
    }
} finally {
    $cleanupResult = Invoke-DbaXNonQuery `
        -Server $databaseServer `
        -Database $databaseName `
        -TrustServerCertificate:$trustServerCertificate `
        -Query "DROP TABLE IF EXISTS $stageTable;"
}
```

The important detail is `-AsDataTable`. That gives DbaClientX a tabular shape it can send to provider-native bulk insert APIs. The bulk writer uses the generated connection string; the merge and cleanup commands use the same server, database, and certificate-trust decision, and DbaClientX enables SQL Server encryption for those direct non-query connections. If the workflow uses a database credential, pass the same `$databaseCredential` to both `New-DbaXConnectionString` and `Invoke-DbaXNonQuery`.

A unique staging table isolates retries and concurrent imports; the `finally` block removes it even when validation or merge fails. Validate types, required columns, row counts, duplicate keys, and business rules before changing production data.

## Use the same ownership boundary for CSV

CSV does not need a second database client hidden inside file-format commands. DbaClientX can keep owning the query and connection lifetime while PSWriteOffice owns delimiters, quoting, encoding, compression, and the CSV artifact.

For database-to-CSV streaming, pass the live reader as one input object and dispose it after the writer finishes:

```powershell
$reader = Invoke-DbaXQuery `
    -Server 'sql01' `
    -Database 'Operations' `
    -Query 'SELECT Id, Name, Status, ModifiedUtc FROM dbo.WorkQueue' `
    -AsDataReader

try {
    Export-OfficeCsv `
        -InputObject $reader `
        -Path .\WorkQueue.csv `
        -Delimiter ',' `
        -Encoding ([System.Text.UTF8Encoding]::new($false))
} finally {
    $reader.Dispose()
}
```

For CSV-to-database streaming, describe the important column types at the file boundary, then hand the resulting reader to DbaClientX as a single object:

```powershell
$reader = Import-OfficeCsv `
    -Path .\ApprovedProducts.csv `
    -AsDataReader `
    -ColumnType @{
        ProductId = [int]
        UnitPrice = [decimal]
        Approved = [bool]
        ApprovedUtc = [datetime]
    }

try {
    Write-DbaXTableData `
        -Provider SqlServer `
        -ConnectionString $connectionString `
        -DestinationTable 'dbo.ApprovedProduct_Stage' `
        -InputObject (, $reader) `
        -BatchSize 5000
} finally {
    $reader.Dispose()
}
```

The leading comma in `-InputObject (, $reader)` is intentional: it prevents PowerShell from trying to enumerate the reader before DbaClientX receives it. For SQL Server, that reader can flow into the bulk-copy path without first becoming a full in-memory object array. For providers whose bulk API needs a materialized table, DbaClientX performs the provider-specific handoff.

Use `-InferSchema` when a representative sample is good enough, or `-ColumnType` when the staging contract is known. Explicit types are usually the safer choice for IDs, money, booleans, and timestamps. PSWriteOffice also exposes compressed CSV, null values, date formats, duplicate-header policy, strict or lenient quote handling, decompression limits, and parse-error behavior; those remain file-format concerns instead of becoming provider-specific switches in DbaClientX.

This composition is also why four extra commands such as `Export-DbaXQueryCsv` or `Import-DbaXCsv` would add little. The two modules already meet at standard .NET tabular contracts, and keeping that seam visible lets each one remain useful on its own.

## Write directly only for append-only rows

Sometimes the workbook contains guaranteed-new rows for an append-only import log. In that narrow case, writing directly to the destination table can be acceptable:

```powershell
$table = Import-OfficeExcel `
    -Path .\ApprovedProducts.xlsx `
    -WorksheetName 'Products' `
    -AsDataTable

$table | Write-DbaXTableData `
    -Provider SqlServer `
    -ConnectionString $connectionString `
    -DestinationTable 'dbo.ProductImportLog' `
    -BatchSize 10000 `
    -BulkCopyTimeout 120
```

Validation alone does not make a direct bulk write repeatable. If the workbook can contain an existing key, replace a row, or be retried, use the isolated staging-and-merge workflow instead. Direct writes are for rows whose keys are guaranteed to be new and whose destination is intentionally append-only.

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

## Pick the shape that fits the job

There are two practical ways to move data.

Use the flexible path when the dataset is small or you need PowerShell transformations:

```powershell
Invoke-DbaXQuery -Server 'sql01' -Database 'Operations' -Query 'SELECT Id, Status FROM dbo.WorkQueue' |
    Where-Object Status -ne 'Closed' |
    Select-Object Id, Status, @{ Name = 'ExportedUtc'; Expression = { [DateTime]::UtcNow } } |
    Export-OfficeExcel -Path .\OpenWork.xlsx -WorksheetName Open -TableName OpenWork
```

Use a buffered tabular path when the script needs the whole result for validation or transformation:

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

The commands still look like normal PowerShell, but the data stays in a form that database and workbook libraries understand directly. When the rows only need to pass through, use the `IDataReader` example from the export section to avoid buffering the complete result.

## What the benchmarks actually measure

We built these paths while trying to shorten our own reporting jobs, but the useful question is not which module wins in the abstract. It is whether the exact job is equivalent, the output is correct, and the result repeats on the machine that will run it.

The [DbaClientX repository includes PowerForge suites](https://github.com/EvotecIT/DbaClientX#sql-server-benchmarks) for direct SQL Server reads and writes. In the current committed 25,000-row snapshot, the comparable `DataTable` lanes reported these medians:

| Operation | DbaClientX | dbatools |
| --- | ---: | ---: |
| Read all rows into a `DataTable` | 32 ms | 43 ms |
| Write a `DataTable` with provider bulk copy | 40 ms | 60 ms |

Those are client-side data-movement lanes, not a comparison of the much broader dbatools command set. The runner seeds isolated tables outside the measurement and validates row counts plus identifier and score sums afterward.

A separate dated run on 10 August 2026 measured the complete 25,000-row SQL Server to XLSX to SQL Server job. The DbaClientX lane streamed through PSWriteOffice and OfficeIMO; the comparison used ImportExcel 7.8.10 through its public object pipeline. Both produced the same table contract and passed strict SQL-side row, schema, and value checks.

| Engine path | L3 domain 0 median | L3 domain 1 median |
| --- | ---: | ---: |
| DbaClientX + PSWriteOffice + OfficeIMO | 218.62 ms | 174.12 ms |
| ImportExcel 7.8.10 | 3,519.73 ms | 2,926.71 ms |

The two columns are deliberate. This workstation uses an AMD Ryzen 9 9950X3D2 with two performance domains, so the full comparison was repeated with one 16-logical-processor affinity mask at a time. The run used PowerShell 7.6.4, SQL Server 17.0.1125.2, High process priority, five warmups, fifteen rotated measured iterations, and no removed outliers. All 60 measured round trips passed validation.

The gap is meaningful for that source-linked fixture; it is not a promise that every workbook, database, CPU, or module version will produce the same ratio. Read the [SQL Server benchmark notes](https://github.com/EvotecIT/DbaClientX/blob/master/docs/sqlserver-benchmark-notes.md), keep both processor domains visible on heterogeneous machines, and rerun the matrix with the row shape and workbook features used in production.

## Publish The Review Pack

A database export often needs two forms: an editable workbook for analysts and a fixed-layout copy for approval or archival. PSWriteOffice can create the workbook and explicitly export it to PDF; Mailozaurr can then deliver both without making the data-access script own SMTP:

```powershell
Import-Module DbaClientX
Import-Module PSWriteOffice
Import-Module Mailozaurr

$outputDirectory = (New-Item -ItemType Directory -Path (Join-Path $PSScriptRoot 'Output') -Force).FullName
$workbookPath = Join-Path $outputDirectory 'WorkQueue.xlsx'
$pdfPath = Join-Path $outputDirectory 'WorkQueue.pdf'

$rows = Invoke-DbaXQuery `
    -Server 'sql01' `
    -Database 'Operations' `
    -Query 'SELECT Id, Name, Status, ModifiedUtc FROM dbo.WorkQueue' `
    -ReturnType DataTable

Export-OfficeExcel `
    -InputObject $rows `
    -Path $workbookPath `
    -WorksheetName 'Work Queue' `
    -TableName WorkQueue `
    -AutoFit `
    -FreezeTopRow

Export-OfficeDocumentPdf `
    -InputPath $workbookPath `
    -Path $pdfPath

$mailCredential = Get-Secret -Name 'Reporting-Smtp-Credential'
Send-EmailMessage `
    -From 'reports@example.com' `
    -To 'reviewers@example.com' `
    -Subject 'Work queue review pack' `
    -Text 'The editable workbook and PDF review copy are attached.' `
    -Attachment $workbookPath, $pdfPath `
    -Server 'smtp.example.com' `
    -Credential $mailCredential `
    -UseSsl
```

`InputPath` is intentional on `Export-OfficeDocumentPdf` because `Path` names the PDF being produced. Elsewhere, PSWriteOffice uses `Path` for the primary file, `OutputPath` for a transformed copy, and `DestinationPath` for a copy destination.

The ownership stays simple: DbaClientX queries and writes database data, PSWriteOffice creates document artifacts, and Mailozaurr handles credentials, message transport, and attachments. `Get-Secret` comes from Microsoft.PowerShell.SecretManagement; use the same secret provider your scheduled job or CI runner already trusts.

## Common scenarios

This pattern fits a lot of everyday operational work:

- Use compact CSV for machine handoffs and real Excel workbooks for human review.
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

# Database to CSV without object materialization
$reader = Invoke-DbaXQuery ... -AsDataReader
try { Export-OfficeCsv -InputObject $reader -Path .\Data.csv } finally { $reader.Dispose() }

# Excel to database
$table = Import-OfficeExcel -Path .\Data.xlsx -WorksheetName Data -AsDataTable
$table | Write-DbaXTableData -Provider SqlServer -ConnectionString $connectionString -DestinationTable dbo.Data_Stage
```

That is the workflow: query, export, review where needed, import, bulk write, validate, merge.

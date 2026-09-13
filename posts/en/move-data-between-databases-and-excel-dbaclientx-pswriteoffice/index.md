---
title: "Move data between databases, CSV, and Excel with DbaClientX and PSWriteOffice"
description: "Move database rows through CSV or Excel, bring reviewed changes back safely, and choose the right PowerShell data shape for each step."
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

A data movement job often starts as "export this table to Excel." A month later the workbook has to come back into the database, somebody needs a CSV version, and the script is holding every row in memory even when nobody wants to change it.

[DbaClientX](https://github.com/EvotecIT/DbaClientX) is the database side of the workflow. It can query SQL Server, PostgreSQL, MySQL, Oracle, and SQLite from PowerShell or .NET, and it uses provider-native bulk insert paths for tabular data.

[PSWriteOffice](https://github.com/EvotecIT/PSWriteOffice) owns the file side. It writes real `.xlsx` workbooks and delimited files, then reads reviewed data back as PowerShell objects, `DataTable`, or `IDataReader`. Microsoft Excel does not need to be installed on the machine running the job.

[OfficeIMO](https://github.com/EvotecIT/OfficeIMO) provides the workbook engine underneath PSWriteOffice. The PowerShell commands stay short, while the same implementation can handle a quick export or a deliberately composed workbook.

Together, the modules cover the paths I normally need:

- You want database rows in Excel? Query with DbaClientX, export with PSWriteOffice.
- You want a CSV handoff without turning every row into a `PSCustomObject`? Pass the same reader to `Export-OfficeCsv`.
- You want a reviewed workbook back in a database? Import with PSWriteOffice, bulk write with DbaClientX.
- You want to avoid materializing every row as a PowerShell object? Hand an `IDataReader` directly from the database client to the workbook writer.
- You want the flexible path? Use normal PowerShell objects and let the commands convert them.

## Before you start

Use PowerShell 7 and install the current public modules used for this article:

```powershell
Install-Module PSWriteOffice -Scope CurrentUser -Force
Import-Module PSWriteOffice
Install-Module DbaClientX -Scope CurrentUser -Force
Import-Module DbaClientX
```

Run the examples from a folder where you can write the generated files. Replace the server names, databases, credentials, SMTP settings, and sample tables before using any part in a real environment.

The credential examples also require Microsoft.PowerShell.SecretManagement and a registered vault provider. The final email example additionally uses Mailozaurr:

```powershell
Install-Module Microsoft.PowerShell.SecretManagement -Scope CurrentUser
Install-Module Mailozaurr -Scope CurrentUser
Import-Module Microsoft.PowerShell.SecretManagement
```

Install and register the vault provider used by your environment, select its default vault, and store `Operations-Database-Credential` and `Reporting-Smtp-Credential` as `PSCredential` secrets before running those sections. `Get-SecretVault` lists registered vaults; `Test-SecretVault -Name '<your-vault-name>'` verifies access. Installing SecretManagement alone does not create a vault or add either secret.

## How this fits with dbatools and ImportExcel

I am not suggesting that a working dbatools or ImportExcel script should be rewritten for the sake of it.

[dbatools](https://github.com/dataplat/dbatools) is an established SQL Server automation toolkit. If a job already uses its instance, backup, migration, or administrative commands, keeping data movement there may be the clearest choice. I reach for DbaClientX when the same provider-neutral contract must work from PowerShell and .NET, or when a live reader should pass directly into another library.

[ImportExcel](https://github.com/dfinke/ImportExcel) is the familiar choice for many PowerShell-to-Excel scripts. If `Export-Excel` and `Import-Excel` already produce the workbook you need, there is no prize for changing them. PSWriteOffice becomes useful when the workbook is part of a wider Word, PowerPoint, PDF, CSV, or document-inspection workflow, or when direct `DataTable` and `IDataReader` handoffs matter.

The modules can also be mixed. Keep dbatools around the SQL Server administration, keep ImportExcel for a workbook that already depends on it, and introduce DbaClientX or PSWriteOffice only where it solves a specific problem.

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

`Get-Secret` comes from Microsoft.PowerShell.SecretManagement. In CI, the credential or connection string can come from the runner's secret provider. Keep passwords out of the script, repository, and command history.

## Export database rows to Excel

For a workbook that people can filter, review, and send back, query the database and export the result as an Excel table:

The editable round trip needs one reliable way to identify rows and one way to detect concurrent changes. In this example, `dbo.WorkQueue` has an application-assigned, non-`IDENTITY` `Id` primary key and a SQL Server `rowversion` column. New workbook rows need unique IDs assigned by the application. A table with database-generated identity keys needs a separate insert and key-mapping step.

I export the concurrency token as hexadecimal text in `ExpectedVersion` and leave it unchanged while editing `Name` and `Status`. Because SQL Server changes a rowversion on every update, the import can detect a change without depending on timestamp precision surviving the Excel round trip. Rows whose editable values did not change keep their existing `ModifiedUtc` and token.

```powershell
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
    -Query 'SELECT Id, Name, Status, ModifiedUtc, CONVERT(varchar(18), CAST(RowVersion AS binary(8)), 1) AS ExpectedVersion FROM dbo.WorkQueue' `
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

The query uses DbaClientX's direct SQL Server parameter set. It enables encryption and makes the certificate-trust decision explicit. I keep the generated connection string for the streaming example and the later bulk-write path.

The result is a native `.xlsx` workbook rather than a CSV with another extension. It contains a filterable Excel table, even though Excel was not installed on the machine that created it.

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

I use the object pipeline when I want the script to stay simple or need `Where-Object`, `Select-Object`, and calculated properties between the query and export.

For a pass-through SQL Server export, the owned `IDataReader` keeps rows streaming until PSWriteOffice consumes them. Dispose it in `finally` because it owns the live command and connection. I switch to `DataTable` when the script must inspect, validate, or reshape the complete dataset before writing the workbook.

## Import Excel rows to a database table

When the workbook comes back from review, I import it as a `DataTable` and write it to a staging table. Existing rows keep their `ExpectedVersion`; an intentionally new row has a new `Id` and an empty token. The sample treats blank `Name` and `Status` cells as SQL `NULL`. Change that rule if your schema requires values or distinguishes an empty string from null:

```powershell
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

foreach ($column in 'Id', 'Name', 'Status', 'ExpectedVersion') {
    if (-not $table.Columns.Contains($column)) {
        throw "The reviewed workbook is missing the required '$column' column."
    }
}
foreach ($row in $table.Rows) {
    foreach ($column in 'Name', 'Status') {
        if ([string]::IsNullOrEmpty([string]$row[$column])) {
            $row[$column] = [DBNull]::Value
        }
    }
    $token = [string]$row.ExpectedVersion
    if ($token.Length -gt 0 -and $token -notmatch '^0x[0-9a-fA-F]{16}$') {
        throw "WorkQueue ID '$($row.Id)' has an invalid ExpectedVersion. Re-export it before reviewing."
    }
}

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
        -ErrorAction Stop `
        -PassThru

    $mergeResult = Invoke-DbaXNonQuery `
        -Server $databaseServer `
        -Database $databaseName `
        -TrustServerCertificate:$trustServerCertificate `
        -ErrorAction Stop `
        -Query @"
SET XACT_ABORT ON;
BEGIN TRY
    BEGIN TRANSACTION;

    IF EXISTS (
        SELECT 1
        FROM $stageTable AS source
        LEFT JOIN dbo.WorkQueue AS target WITH (UPDLOCK, HOLDLOCK)
            ON target.Id = source.Id
        WHERE (
            NULLIF(source.ExpectedVersion, '') IS NOT NULL
            AND (target.Id IS NULL
                OR target.RowVersion <> CONVERT(binary(8), source.ExpectedVersion, 1))
        ) OR (NULLIF(source.ExpectedVersion, '') IS NULL AND target.Id IS NOT NULL)
    )
        THROW 51000, 'The workbook conflicts with current WorkQueue rows. Re-export and resolve the conflicts.', 1;

    UPDATE target
    SET Name = source.Name,
        Status = source.Status,
        ModifiedUtc = SYSUTCDATETIME()
    FROM dbo.WorkQueue AS target
    JOIN $stageTable AS source ON target.Id = source.Id
    WHERE EXISTS (
        SELECT CONVERT(varbinary(max), CONVERT(nvarchar(max), target.Name)), CONVERT(varbinary(max), CONVERT(nvarchar(max), target.Status))
        EXCEPT
        SELECT CONVERT(varbinary(max), CONVERT(nvarchar(max), source.Name)), CONVERT(varbinary(max), CONVERT(nvarchar(max), source.Status))
    );

    INSERT dbo.WorkQueue (Id, Name, Status, ModifiedUtc)
    SELECT source.Id, source.Name, source.Status, SYSUTCDATETIME()
    FROM $stageTable AS source
    WHERE NOT EXISTS (SELECT 1 FROM dbo.WorkQueue AS target WHERE target.Id = source.Id);

    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
    THROW;
END CATCH;
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

`-AsDataTable` gives DbaClientX a tabular shape it can send to the provider's bulk insert API. The bulk writer uses the generated connection string. The merge and cleanup commands use the same server, database, and certificate-trust decision, and DbaClientX enables SQL Server encryption for those direct non-query connections. If the job uses a database credential, pass the same `$databaseCredential` to both `New-DbaXConnectionString` and `Invoke-DbaXNonQuery`.

A unique staging table keeps two imports from colliding and gives a retry its own workspace. The `finally` block removes it even when validation or import fails. Inside the transaction, SQL Server locks the relevant keys while the script compares version tokens and applies changes.

If a source row changed or was deleted, a new row took an ID in the meantime, or the same workbook already inserted or updated data, the whole batch is rejected. Replaying a workbook that made no changes is a harmless no-op. Re-export and resolve a conflict instead of overwriting newer values. Rows missing from the workbook are left alone; this example does not delete them. Add your own type, required-value, row-count, duplicate-key, and business-rule checks before using the pattern on production data.

## Use the same ownership boundary for CSV

CSV uses the same ownership split. DbaClientX keeps control of the query and connection lifetime, while PSWriteOffice handles delimiters, quoting, encoding, compression, and the file itself.

For database-to-CSV streaming, pass the live reader as one input object and dispose it when the writer finishes:

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

For the return path, describe the important column types at the CSV boundary and pass the resulting reader to DbaClientX as one object:

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

The leading comma in `-InputObject (, $reader)` prevents PowerShell from enumerating the reader before DbaClientX receives it. With SQL Server, the reader can flow into bulk copy without becoming a complete in-memory object array. If another provider needs a materialized table, DbaClientX performs that handoff internally.

Use `-InferSchema` when a representative sample is enough, or `-ColumnType` when you know the staging contract. I prefer explicit types for IDs, money, booleans, and timestamps. PSWriteOffice also controls compressed CSV, null values, date formats, duplicate headers, strict or lenient quote handling, decompression limits, and parse errors. Those options stay with the file reader rather than becoming database-provider switches.

There is no need for another set of commands such as `Export-DbaXQueryCsv` or `Import-DbaXCsv`. The modules already meet at standard .NET tabular contracts, and each remains useful without the other.

## Write directly only for append-only rows

If a workbook contains guaranteed-new rows for an append-only log, a direct write can be reasonable:

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

Validation alone does not make that write safe to repeat. If the workbook may contain an existing key, update a row, or be retried, use staging and merge. I reserve direct writes for new keys and a destination that is deliberately append-only.

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
    -ConnectionString $sqliteConnectionString `
    -DestinationTable 'audit_events' `
    -BatchSize 1000
```

The outer workflow stays the same for SQL Server, PostgreSQL, MySQL, Oracle, and SQLite: build or import tabular data, select the provider, supply the connection string, and write to the target table.

## Pick the shape that fits the job

I choose between PowerShell objects, an in-memory `DataTable`, and a streaming `IDataReader` based on what the script needs to change and how much it can buffer.

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

The commands still look like normal PowerShell, while the data remains in a shape the database and workbook libraries understand directly. When rows only need to pass through, use the `IDataReader` example from the export section and avoid buffering the complete result.

## What the benchmarks actually measure

I built these paths while trying to shorten real reporting jobs. The useful benchmark question is not which module "wins." It is whether both sides performed the same job, whether the files and database rows are correct, and whether the result repeats on the machine that will run it.

The [DbaClientX benchmark snapshot](https://github.com/EvotecIT/DbaClientX/blob/fa7a2a0ec3bb79e19cb2ba9838e556706cf8be1c/README.md#sql-server-benchmarks) includes PowerForge suites for direct SQL Server reads and writes. In that committed 25,000-row snapshot, the comparable `DataTable` lanes reported these medians:

| Operation | DbaClientX | dbatools |
| --- | ---: | ---: |
| Read all rows into a `DataTable` | 32 ms | 43 ms |
| Write a `DataTable` with provider bulk copy | 40 ms | 60 ms |

These measurements cover client-side data movement, rather than the much broader dbatools command set. The runner seeds isolated tables before the measurement and checks row counts plus identifier and score sums afterward.

On 10 August 2026 I also measured the complete 25,000-row SQL Server to XLSX to SQL Server round trip. The DbaClientX path streamed through PSWriteOffice and OfficeIMO; the comparison used ImportExcel 7.8.10 through its public object pipeline. Both produced the same table contract and passed strict SQL-side row, schema, and value checks.

| Engine path | L3 domain 0 median | L3 domain 1 median |
| --- | ---: | ---: |
| DbaClientX + PSWriteOffice + OfficeIMO | 218.62 ms | 174.12 ms |
| ImportExcel 7.8.10 | 3,519.73 ms | 2,926.71 ms |

The two columns are deliberate. My workstation uses an AMD Ryzen 9 9950X3D2 with two performance domains, so I repeated the comparison with one 16-logical-processor affinity mask at a time. The run used PowerShell 7.6.4, SQL Server 17.0.1125.2, High process priority, five warmups, fifteen rotated measured iterations, and no removed outliers. All 60 measured round trips passed validation.

That gap belongs to this source-linked fixture. It is not a promise that another workbook, database, CPU, or module version will produce the same ratio. Read the [SQL Server benchmark notes at the measured revision](https://github.com/EvotecIT/DbaClientX/blob/fa7a2a0ec3bb79e19cb2ba9838e556706cf8be1c/docs/sqlserver-benchmark-notes.md), keep both processor domains visible on heterogeneous machines, and rerun the matrix with the row shape and workbook features you plan to use.

## Publish the review pack

I often need two versions of a database export: an editable workbook for analysts and a fixed-layout copy for approval or archival. PSWriteOffice creates the workbook and exports the PDF; Mailozaurr delivers both without adding SMTP code to the data-access module:

```powershell
Import-Module Mailozaurr

$outputDirectory = (New-Item -ItemType Directory -Path (Join-Path (Get-Location).Path 'Output') -Force).FullName
$workbookPath = Join-Path $outputDirectory 'WorkQueue.xlsx'
$pdfPath = Join-Path $outputDirectory 'WorkQueue.pdf'

$rows = Invoke-DbaXQuery `
    -Server 'sql01' `
    -Database 'Operations' `
    -Query 'SELECT Id, Name, Status, ModifiedUtc, CONVERT(varchar(18), CAST(RowVersion AS binary(8)), 1) AS ExpectedVersion FROM dbo.WorkQueue' `
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

`Export-OfficeDocumentPdf` uses `InputPath` for the workbook because `Path` names the PDF being produced. Elsewhere, PSWriteOffice uses `Path` for the primary file, `OutputPath` for a transformed copy, and `DestinationPath` for a copy destination.

DbaClientX queries and writes the database, PSWriteOffice creates the files, and Mailozaurr handles credentials, transport, and attachments. `Get-Secret` comes from Microsoft.PowerShell.SecretManagement; use the secret provider your scheduled job or CI runner already trusts.

## Common scenarios

I use this pattern for work such as:

- Use compact CSV for machine handoffs and real Excel workbooks for human review.
- Export SQL Server review queues to business owners.
- Load reviewed workbook changes into staging tables.
- Build audit handoff workbooks from PostgreSQL, MySQL, Oracle, SQLite, or SQL Server.
- Use SQLite as a local database for workbook-driven tools.
- Keep import/export scripts reusable instead of rewriting them per report.

For a small one-way job, the script can still be short:

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

Once edits need to come back, add staging, validation, and concurrency checks before the write. The few extra steps are cheaper than discovering that a returned workbook silently replaced newer database changes.

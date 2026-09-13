---
title: "Turn HTML tables into Excel workbooks with PSParseHTML and PSWriteOffice"
description: "Parse useful tables from an HTML page and turn them into native Excel workbooks from either PowerShell or C#."
date: "2026-05-15"
language: "en"
authors:
  - przemyslaw-klys
categories:
  - PowerShell
  - Reporting
tags:
  - powershell
  - psparsehtml
  - pswriteoffice
  - htmltinkerx
  - officeimo
  - excel
image: "./cover.png"
image_alt: "Analyst selecting a useful web table beside an editable workbook version"
draft: true
---

Sooner or later I end up with useful data trapped in an HTML table. It may come from a vendor portal, a monitoring page, an exported report, an old intranet, or a dashboard that never received an API.

The PowerShell version of the requirement is usually short:

> Get the table from HTML and push it into Excel.

That is still the goal. What changed is where the work happens. The parser and workbook code now live in reusable .NET libraries, while the PowerShell modules provide commands on top.

As a result, the same implementation works for two audiences:

- C# developers can use the .NET libraries directly.
- PowerShell users can keep using pipeline-friendly commands.

I can fix the table parser or workbook writer once and both paths benefit.

![The four-row HTML sample in desktop Excel's print view, with typed values and extracted evidence URLs](./images/html-tables-excel-preview.png)

## Before you start

Use PowerShell 7 and install the current public modules used for this article:

```powershell
Install-Module PSWriteOffice -Scope CurrentUser -Force
Import-Module PSWriteOffice
Install-Module PSParseHTML -Scope CurrentUser -Force
Import-Module PSParseHTML
```

Run the examples from a folder where you can write the generated files. The sample HTML is included below, so you can try the complete path before pointing it at a real page.

## The pieces

Four project names appear in the examples. Here is the practical split before we get to the code.

`HtmlTinkerX` is the .NET HTML engine. It parses tables, including headers, row and column spans, captions, metadata, link text, and optional link URLs, and returns reusable table models.

`PSParseHTML` wraps HtmlTinkerX for PowerShell. Its `ConvertFrom-HtmlTable` command means a PowerShell script does not need custom C# just to read a table.

`OfficeIMO.Excel` creates and reads `.xlsx` files without Excel COM automation. It owns worksheets, tables, data tables, datasets, charts, formatting, and workbook structure.

`PSWriteOffice` exposes that Excel engine to PowerShell through commands such as `Export-OfficeExcel`, `Get-OfficeExcel`, `Add-OfficeExcelChart`, and `Close-OfficeExcel`.

The flow looks like this:

```text
HTML page or file
  -> HtmlTinkerX or PSParseHTML
  -> DataTable / DataSet / row objects
  -> OfficeIMO.Excel or PSWriteOffice
  -> native .xlsx workbook
```

This workflow extracts table data; it does not try to reproduce the webpage in Excel. The result remains rows and columns that Excel can filter, chart, format, and edit.

## Why I keep parsing and Excel separate

Putting HTML import directly into OfficeIMO.Excel would make this one demo shorter, but it would also make the Excel library responsible for web scraping, CSS, JavaScript, and every other format somebody might want to import next.

I want OfficeIMO to work with normal .NET data shapes such as `DataTable`, `DataSet`, object sequences, and data readers. HtmlTinkerX has a different job: turn HTML into structured information. Neither library needs to know about the other.

The connection happens in the calling code. A C# service joins HtmlTinkerX with OfficeIMO.Excel; a PowerShell script pipes PSParseHTML into PSWriteOffice.

## If ImportExcel already fits

The parser does not require a particular workbook module. Its output is ordinary row objects, `DataTable`, or `DataSet`, so an existing [ImportExcel](https://github.com/dfinke/ImportExcel) workflow can stay exactly where it is useful.

This article uses PSWriteOffice because the later examples add workbook structure, charts, and read-back, and the wider series also creates other Office formats. A simple HTML-table export does not require a new Excel tool. The [PSWriteOffice comparison page](https://github.com/EvotecIT/PSWriteOffice/blob/main/Website/content/project-docs/docs/compare-importexcel-excelfast.md) shows command shapes, project scope, and validated benchmark scenarios when you need to compare them.

## For C# developers

Save the sample as `service-status.html` in your working directory. Both examples use it. The links are fictional evidence URLs, so replace them when adapting the script.

```html
<!doctype html>
<html lang="en">
<head><meta charset="utf-8"><title>Service status</title></head>
<body>
<table id="services">
  <thead><tr><th>Service</th><th>Health</th><th>Incidents</th><th>Owner</th><th>Evidence</th></tr></thead>
  <tbody>
    <tr><td>Identity Sync</td><td>98</td><td>1</td><td>Platform</td><td><a href="https://example.com/identity">Details</a></td></tr>
    <tr><td>Remote Access</td><td>76</td><td>7</td><td>Security</td><td><a href="https://example.com/access">Details</a></td></tr>
    <tr><td>Backup</td><td>92</td><td>2</td><td>Operations</td><td><a href="https://example.com/backup">Details</a></td></tr>
    <tr><td>Monitoring</td><td>95</td><td>1</td><td>Operations</td><td><a href="https://example.com/monitoring">Details</a></td></tr>
  </tbody>
</table>
</body>
</html>
```

In a .NET console project, add `HtmlTinkerX` 3.0.1 and `OfficeIMO.Excel` 3.4.3 with `dotnet add package`. HtmlTinkerX converts the table into a `DataTable`, and OfficeIMO.Excel writes the workbook.

```csharp
using HtmlTinkerX;
using OfficeIMO.Excel;
using System;
using System.Collections.Generic;
using System.Data;
using System.IO;
using System.Linq;

string html = File.ReadAllText("service-status.html");

List<HtmlTableResult> htmlTables = HtmlParser.ParseTablesWithAngleSharpDetailed(
    html,
    replaceContent: null,
    replaceHeaders: null,
    allProperties: false,
    skipFooter: false,
    cleanHeaders: true,
    emptyValuePlaceholder: null,
    cellTextFormat: HtmlCellTextFormat.Compact,
    includeLinkUrls: true);

HtmlTableResult serviceTable = htmlTables.Single(table =>
    string.Equals(table.Metadata.Id, "services", StringComparison.OrdinalIgnoreCase));
DataTable services = serviceTable.ToDataTable("Services", inferTypes: true);

using var workbook = ExcelDocument.Create("ServiceStatus.xlsx");
ExcelSheet sheet = workbook.AddWorksheet("Services");

sheet.InsertDataTableAsTable(
    services,
    tableName: "Services",
    style: ExcelTableStyle.TableStyleMedium2,
    includeAutoFilter: true);

sheet.AutoFitColumnsFor(Enumerable.Range(1, services.Columns.Count));

workbook.Save();
```

If the HTML contains several useful tables, convert them to a `DataSet` and let OfficeIMO.Excel create one worksheet per table.

```csharp
DataSet dataSet = htmlTables.ToDataSet("ServiceStatus", inferTypes: true);

using var workbook = ExcelDocument.Create("ServiceStatus.xlsx");

workbook.InsertDataSet(
    dataSet,
    createTables: true,
    tableStyle: ExcelTableStyle.TableStyleMedium2,
    includeHeaders: true,
    includeAutoFilter: true,
    autoFit: true);

workbook.Save();
```

The two libraries meet at a standard `DataTable`; PowerShell is not involved in this version.

## For PowerShell users

In PowerShell, the same handoff becomes a pipeline.

```powershell
ConvertFrom-HtmlTable `
    -Path .\service-status.html `
    -TableId 'services' `
    -AsDataTable `
    -IncludeLinkUrls `
    -InferTypes |
    Export-OfficeExcel `
        -Path .\ServiceStatus.xlsx `
        -WorksheetName 'Services' `
        -TableName 'Services' `
        -AutoFit `
        -FreezeTopRow `
        -BoldTopRow
```

The command reads one HTML table, returns a `DataTable`, and writes a native Excel table. The `.xlsx` contains rows, columns, headers, filters, and values rather than a screenshot of the page.

For all tables in the HTML file, use `-AsDataSet`.

```powershell
$tables = ConvertFrom-HtmlTable `
    -Path .\service-status.html `
    -AsDataSet `
    -IncludeLinkUrls `
    -InferTypes

$tables | Export-OfficeExcel `
    -Path .\ServiceStatus.xlsx `
    -AutoFit `
    -FreezeTopRow `
    -BoldTopRow
```

The PowerShell commands are shorter, but they use the same parser and workbook engine as the C# example.

## Add Excel behavior after import

Once the table is in Excel, I can add workbook behavior rather than stopping at the export.

```powershell
$workbook = Get-OfficeExcel -Path .\ServiceStatus.xlsx
ExcelSheet -Document $workbook 'Services' {
    ExcelColumn -ColumnName 'A' -Width 22
    ExcelColumn -ColumnName 'B' -Width 12
    ExcelColumn -ColumnName 'C' -Width 14
    ExcelColumn -ColumnName 'D' -Width 18
    ExcelColumn -ColumnName 'E' -Width 14
    ExcelColumn -ColumnName 'F' -Width 45
}
Add-OfficeExcelTableOfContents `
    -Document $workbook `
    -SheetName 'Index' `
    -AddBackLinks

Add-OfficeExcelChart `
    -Document $workbook `
    -Sheet 'Services' `
    -Range 'A1:B5' `
    -Row 8 `
    -Column 1 `
    -Type BarClustered `
    -Title 'Service health score'

$workbook | Save-OfficeExcel
$workbook | Close-OfficeExcel
```

The `Close-OfficeExcel` command is intentional. I do not want a public PowerShell example to require `.Dispose()` when the module can expose a normal `Close-*` command and handle the object lifetime itself.

## When to use this

I use this approach when the table already contains the data I need:

- service status pages
- product comparison tables
- vendor export pages
- documentation tables
- release matrices
- monitoring summaries
- internal HTML reports

I choose another tool when the job is:

- rendering a whole webpage into Excel
- preserving CSS layout
- running JavaScript
- screen scraping a browser-only application
- replacing an API when a proper API exists

If the site has a useful API, use it. When the only useful interface is an HTML table, this pipeline gives me a repeatable extraction without pretending to be a browser.

## Why the modules changed

The older PowerShell-only modules were convenient, but parsing, transformation, workbook creation, formatting, and file handling often ended up in the same script module. That made improvements harder to reuse from C# and harder to test independently.

The current split is:

- .NET libraries own reusable mechanics.
- PowerShell modules expose those mechanics in a friendly way.
- C# users do not need PowerShell.
- PowerShell users do not need to care that C# is underneath.

That lets me improve the HTML parser without teaching it about Excel, and improve the workbook writer without teaching it about the web.

For this specific workflow, the clean split is:

- HtmlTinkerX improves table parsing.
- PSParseHTML improves PowerShell selection and output options.
- OfficeIMO.Excel improves workbook and tabular-data support.
- PSWriteOffice improves the reporting experience.

For the reader, the workflow stays simple: select a table and write a native workbook. Underneath, C# and PowerShell share the same mechanics without Excel COM or a browser runtime.

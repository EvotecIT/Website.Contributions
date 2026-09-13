---
title: "Turn HTML tables into Excel workbooks with PSParseHTML and PSWriteOffice"
description: "Learn how HtmlTinkerX, PSParseHTML, OfficeIMO.Excel, and PSWriteOffice fit together so C# and PowerShell users can parse HTML tables and create native Excel workbooks."
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

HTML tables show up in many places: vendor portals, monitoring pages, exported reports, documentation sites, product status pages, old intranet systems, and simple dashboards that were never meant to become APIs.

If you are a PowerShell user, the old instinct is often:

> Get the table from HTML and push it into Excel.

That is still the goal, but the architecture has changed. The newer modules in this family are no longer just piles of PowerShell functions doing all the work directly. The heavy lifting has moved into reusable .NET engines, with PowerShell modules acting as friendly command surfaces on top.

That matters because it gives two groups a clean path:

- C# developers can use the .NET libraries directly.
- PowerShell users can keep using pipeline-friendly commands.

The same core mechanics power both.

![The four-row HTML sample in desktop Excel's print view, with typed values and extracted evidence URLs](./images/html-tables-excel-preview.png)

## Before you start

Use PowerShell 7 and install the current public modules used for this article:

```powershell
Install-Module PSWriteOffice -Scope CurrentUser
Import-Module PSWriteOffice
Install-Module PSParseHTML -Scope CurrentUser
Import-Module PSParseHTML
```

Run examples from a working folder where you can write the generated files. Supply your own inputs wherever a later example references an existing file or service.

## The Pieces

There are four names involved, so it is worth separating them before looking at code.

`HtmlTinkerX` is the .NET HTML engine. It parses HTML tables, understands headers, row and column spans, captions, table metadata, link text, optional link URLs, and can return reusable table models.

`PSParseHTML` is the PowerShell module on top of HtmlTinkerX. It exposes commands such as `ConvertFrom-HtmlTable`, so PowerShell users do not need to write C# just to parse a table.

`OfficeIMO.Excel` is the .NET Excel engine. It creates and reads `.xlsx` files without Excel COM automation. It knows about worksheets, tables, data tables, datasets, charts, formatting, and workbook structure.

`PSWriteOffice` is the PowerShell module on top of OfficeIMO. It exposes commands such as `Export-OfficeExcel`, `Get-OfficeExcel`, `Add-OfficeExcelChart`, and `Close-OfficeExcel`.

The flow looks like this:

```text
HTML page or file
  -> HtmlTinkerX or PSParseHTML
  -> DataTable / DataSet / row objects
  -> OfficeIMO.Excel or PSWriteOffice
  -> native .xlsx workbook
```

This is not "render the page exactly as a browser sees it." It is table extraction. The output remains data, which means Excel can filter it, chart it, format it, and keep it editable.

## Why The Split Matters

It would be tempting to put HTML import directly into OfficeIMO.Excel. That would make one demo shorter, but it would make the library worse over time.

OfficeIMO should be excellent at Office documents. It should consume normal .NET shapes such as `DataTable`, `DataSet`, object sequences, and data readers. It should not become a web scraper, browser renderer, CSS engine, JavaScript host, or SQL client.

Likewise, HtmlTinkerX should not need to know what Excel is. Its job is to turn HTML into structured information.

That boundary is what makes the stack reusable. If you are building a C# service, you can connect HtmlTinkerX to OfficeIMO.Excel. If you are writing PowerShell, you can pipe PSParseHTML into PSWriteOffice.

## If ImportExcel already fits

The table parser does not require a particular workbook module. Its output is ordinary row objects, `DataTable`, or `DataSet`, so an existing [ImportExcel](https://github.com/dfinke/ImportExcel) workflow can remain exactly where it is useful.

This article uses PSWriteOffice because the companion examples continue into workbook structure, charts, read-back, and other Office formats. That is a workflow choice, not a claim that every HTML-table export needs a new Excel tool. The [PSWriteOffice comparison page](https://github.com/EvotecIT/PSWriteOffice/blob/main/Website/content/project-docs/docs/compare-importexcel-excelfast.md) shows the public command shapes, project scope, and correctness-validated benchmark lanes when the distinction matters.

## For C# Developers

Save this complete sample as `service-status.html` in your working directory. Both language examples below use it. The links are illustrative evidence URLs; replace them with your own service pages.

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

In a .NET console project, add `HtmlTinkerX` 3.0.1 and `OfficeIMO.Excel` 3.4.3 with `dotnet add package`. Use HtmlTinkerX to parse the HTML table and convert it into a `DataTable`, then let OfficeIMO.Excel create the workbook.

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

That is the C# story: use .NET data shapes between libraries. No PowerShell required.

## For PowerShell Users

In PowerShell, the same idea becomes a pipeline.

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

That command reads a selected HTML table, returns a `DataTable`, and writes it as a native Excel table. The workbook is not a bitmap. It is a real `.xlsx` file with rows, columns, headers, filters, and values.

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

That is the PowerShell story: use commands and pipeline data, but still benefit from the .NET engines underneath.

## Add Excel Behavior After Import

Once the table is in Excel, PSWriteOffice can continue working with the workbook.

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

The cleanup command is intentional. Published PowerShell should not ask users to call `.Dispose()` directly when a module can provide a normal `Close-*` command. The script should read like PowerShell, while the module handles object lifetime.

## When To Use This

This approach is useful when the HTML table contains the data you actually need:

- service status pages
- product comparison tables
- vendor export pages
- documentation tables
- release matrices
- monitoring summaries
- internal HTML reports

It is not meant for:

- rendering a whole webpage into Excel
- preserving CSS layout
- running JavaScript
- screen scraping a browser-only application
- replacing an API when a proper API exists

If a page has a real API, use the API. If the useful data is already in an HTML table, this pipeline is practical and repeatable.

## Why This Replaces The Old Shape

Older PowerShell-only modules were convenient, but they often mixed too many responsibilities in one place. Parsing, transformation, workbook creation, formatting, and file handling could all live inside PowerShell script code.

The newer direction is different:

- .NET libraries own reusable mechanics.
- PowerShell modules expose those mechanics in a friendly way.
- C# users do not need PowerShell.
- PowerShell users do not need to care that C# is underneath.

That is the reason to keep improving HtmlTinkerX, OfficeIMO.Excel, PSParseHTML, and PSWriteOffice together instead of pushing every feature into one giant module.

For this specific workflow, the clean split is:

- HtmlTinkerX improves table parsing.
- PSParseHTML improves PowerShell selection and output options.
- OfficeIMO.Excel improves workbook and tabular-data support.
- PSWriteOffice improves the reporting experience.

HTML tables in, native Excel workbooks out. Same mechanics, two audiences, no Excel COM, no browser rendering burden, and no need to make the Office engine understand the whole web.

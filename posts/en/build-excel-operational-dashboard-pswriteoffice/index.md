---
title: "Build an Excel operational dashboard from PowerShell"
description: "Generate a multi-sheet Excel dashboard with navigation, KPI formulas, tables, validation, conditional formatting, charts, evidence links, print setup, hidden notes, and workbook summary validation using PSWriteOffice."
date: "2026-05-11"
language: "en"
authors:
  - przemyslaw-klys
categories:
  - PowerShell
  - Reporting
tags:
  - powershell
  - pswriteoffice
  - officeimo
  - excel
  - dashboard
image: "./cover.png"
image_alt: "Microsoft Excel showing the generated operational dashboard workbook with tables, charts, KPI tiles, links, and navigation"
draft: true
---

Good Excel automation is not about pushing two rows into a workbook. A useful workbook gives readers a starting point, a way to drill into details, and enough formatting to find risk quickly without losing the underlying data.

This showcase builds a multi-sheet operational dashboard from PowerShell objects. It uses the same service-health scenario as the Word report and PowerPoint brief: services, owners, health scores, incidents, trend data, and remediation actions. It creates a summary page, service detail table, trend sheet, owner summary, hidden notes sheet, generated table of contents, internal backlinks, evidence hyperlinks, formulas, validation, conditional formatting, charts, page setup, headers, footers, and a structural summary check.

![Full Excel operational dashboard preview showing KPI tiles, tables, charts, links, and hidden notes](./images/summary-status-chart.png)

## Workbook Shape

The full script lives in `Examples/Showcase/Showcase-Excel-OperationalDashboard.ps1` in the PSWriteOffice repository.

It produces a workbook with these sheets:

- `Index`: generated table of contents with links and backlinks
- `Summary`: KPI formulas and a status legend chart
- `Services`: the main operational table
- `Trend`: month-by-month availability, incident, and automation data
- `Owner Summary`: grouped ownership view for follow-up
- `Notes`: hidden generation notes for audit/debugging

That shape matters because people rarely consume Excel reports in one direction. Some readers start with the summary, some filter the detail table, and some want to jump straight to ownership or trend data.

## Writing From Objects

The workbook starts with normal PowerShell objects. In real usage, those objects might come from REST APIs, Microsoft Graph, monitoring probes, CSV imports, Active Directory queries, or previous PSWriteOffice reads.

```powershell
$services = @(
    [pscustomobject]@{
        Service = 'Identity Sync'
        Owner = 'Platform'
        Health = 98
        Incidents = 1
        Status = 'Healthy'
        Evidence = 'identity-sync'
    }
    [pscustomobject]@{
        Service = 'Remote Access'
        Owner = 'Security'
        Health = 76
        Incidents = 7
        Status = 'Risk'
        Evidence = 'remote-access'
    }
)

$trend = @(
    [pscustomobject]@{ Month = 'Jan'; Availability = 99.1; Incidents = 14; Automation = 62 }
    [pscustomobject]@{ Month = 'Feb'; Availability = 99.2; Incidents = 12; Automation = 66 }
    [pscustomobject]@{ Month = 'Mar'; Availability = 99.4; Incidents = 10; Automation = 71 }
    [pscustomobject]@{ Month = 'Apr'; Availability = 99.3; Incidents = 11; Automation = 74 }
    [pscustomobject]@{ Month = 'May'; Availability = 99.6; Incidents = 8;  Automation = 79 }
    [pscustomobject]@{ Month = 'Jun'; Availability = 99.7; Incidents = 6;  Automation = 83 }
)

$legend = @(
    [pscustomobject]@{ Status = 'Healthy'; Meaning = 'Stable service posture' }
    [pscustomobject]@{ Status = 'Watch';   Meaning = 'Owner follow-up required' }
    [pscustomobject]@{ Status = 'Risk';    Meaning = 'Immediate action required' }
)

$statusMix = $services |
    Group-Object Status |
    ForEach-Object { [pscustomobject]@{ Status = $_.Name; Count = $_.Count } }

$ownerSummary = $services |
    Group-Object Owner |
    ForEach-Object {
        [pscustomobject]@{
            Owner         = $_.Name
            Services      = $_.Count
            AverageHealth = [math]::Round(($_.Group | Measure-Object Health -Average).Average, 1)
            Incidents     = ($_.Group | Measure-Object Incidents -Sum).Sum
        }
    }

$path = '.\Operational-Dashboard.xlsx'
```

The important part is that the script writes the workbook as Excel structure, not as a flat file with decoration. Tables remain tables, formulas remain formulas, hyperlinks remain hyperlinks, and the workbook can keep living after generation.

If a native table is all you need, start with the short pipeline:

```powershell
$services | Export-OfficeExcel `
    -Path '.\ServiceHealth.xlsx' `
    -WorksheetName 'Services' `
    -TableName 'ServiceHealth' `
    -AutoFit `
    -FreezeTopRow
```

The larger DSL below is useful because this workbook needs several sheets, formulas, charts, validation, and navigation. It is an escalation from the simple export, not a requirement for every job.

## Building The Summary Sheet

The summary sheet combines formulas, a styled table, chart formatting, freeze panes, and print defaults.

```powershell
$workbook = New-OfficeExcel -Path $path -NoSave

ExcelSheet -Document $workbook 'Summary' {
        ExcelRow -Row 1 -Values 'Operational Dashboard' -Bold $true
        ExcelCell -Address 'B4' -Formula 'AVERAGE(Services!C2:C3)' -NumberFormat '0.0'
        ExcelCell -Address 'B5' -Formula 'SUM(Services!D2:D3)'
        ExcelCell -Address 'B6' -Formula 'AVERAGE(Trend!D2:D7)/100' -NumberFormat '0%'

        ExcelTable -Data $legend `
            -TableName 'StatusLegend' `
            -StartRow 7 `
            -StartColumn 1 `
            -TableStyle 'TableStyleMedium4' `
            -AutoFit

        ExcelTable -Data $statusMix `
            -TableName 'StatusMix' `
            -StartRow 7 `
            -StartColumn 6 `
            -TableStyle 'TableStyleMedium4' `
            -AutoFit

        ExcelChart -Range 'F7:G10' `
            -Row 7 `
            -Column 9 `
            -Type Doughnut `
            -Title 'Status Mix' `
            -PassThru |
            Set-OfficeExcelChartLegend -Position Right -PassThru |
            Set-OfficeExcelChartDataLabels -ShowValue $true -ShowCategoryName $true -PassThru |
            Set-OfficeExcelChartStyle -StyleId 251 -ColorStyleId 10
}
```

The output remains a normal `.xlsx`: formulas are formulas, tables are tables, charts are charts, and people can keep editing in desktop Excel.

## Detail Sheet: Where The Work Happens

The `Services` sheet is designed for action. It uses a structured table, validation list, color scale, data bars, traffic-light icons, and evidence links generated from a header.

```powershell
ExcelSheet -Document $workbook 'Services' {
    ExcelTable -Data $services `
        -TableName 'ServiceHealth' `
        -StartRow 1 `
        -StartColumn 1 `
        -TableStyle 'TableStyleMedium9' `
        -AutoFit

    ExcelFreeze -TopRows 1
    ExcelValidationList -Range 'E2:E50' -Values 'Healthy','Watch','Risk'
    ExcelConditionalColorScale -Range 'C2:C3' -StartColor '#F8696B' -EndColor '#63BE7B'
    ExcelConditionalDataBar -Range 'D2:D3' -Color '#5B9BD5'
    ExcelConditionalIconSet -Range 'C2:C3' -IconSet ThreeTrafficLights1

    ExcelChart -Range 'A1:D3' `
        -Row 12 `
        -Column 1 `
        -Type BarClustered `
        -Title 'Health Score and Incidents'

    ExcelUrlLinksByHeader `
        -Header 'Evidence' `
        -TableName 'ServiceHealth' `
        -UrlScript { param($text) "https://evotec.xyz/docs/$text" } `
        -TitleScript { param($text) "Open $text" }
}
```

This is the sweet spot for PowerShell-generated Excel: repeatable input data, but still a workbook that feels native when opened by a human.

## Trend And Ownership

The dashboard also includes trend and owner-summary sheets so the report can answer both "what changed?" and "who needs to act?"

![Trend and owner summary preview showing line chart and grouped owner queue](./images/trend-chart.png)

```powershell
ExcelSheet -Document $workbook 'Trend' {
    ExcelTable -Data $trend -TableName 'TrendData' -TableStyle 'TableStyleMedium2' -AutoFit

    ExcelChart -TableName 'TrendData' `
        -Row 10 `
        -Column 1 `
        -Type Line `
        -Title 'Availability, Incidents, and Automation' `
        -PassThru |
        Set-OfficeExcelChartLegend -Position Bottom -PassThru |
        Set-OfficeExcelChartDataLabels -ShowValue $true -Position Top -PassThru |
        Set-OfficeExcelChartStyle -StyleId 251 -ColorStyleId 10
}

ExcelSheet -Document $workbook 'Owner Summary' {
    ExcelTable -Data $ownerSummary -TableName 'OwnerSummary' -TableStyle 'TableStyleMedium5' -AutoFit
    ExcelConditionalDataBar -Range 'D2:D20' -Color '#ED7D31'
    ExcelConditionalIconSet -Range 'C2:C20' -IconSet ThreeTrafficLights1
}
```

The owner summary is intentionally table-based because reviewers need a visible action queue. When the analysis itself needs regrouping, use a real PivotTable; the companion `Recipe-Excel-PivotAndSparklines.ps1` demonstrates pivots and row-level trends in a smaller script.

## Hidden Notes And Navigation

The hidden `Notes` sheet keeps generation details inside the workbook without cluttering the visible report. After all content sheets exist, generate navigation, save, and close the one live workbook.

```powershell
ExcelSheet -Document $workbook 'Notes' {
    ExcelCell -Address 'A1' -Value 'Generation Notes'
    ExcelCell -Address 'A2' -Value 'This sheet is hidden and carries audit/debugging inputs.'
    ExcelCell -Address 'A5' -Value 'Source'
    ExcelCell -Address 'B5' -Value 'Examples/Showcase/Showcase-Excel-OperationalDashboard.ps1'
    ExcelSheetVisibility -Hide
}

ExcelTableOfContents `
    -Document $workbook `
    -SheetName 'Index' `
    -IncludeNamedRanges `
    -AddBackLinks `
    -BackLinkText 'Back to Index'

$workbook | Close-OfficeExcel -Save
```

## Reading And Proving The Workbook Shape

The showcase finishes by reopening the workbook and checking the structure that readers rely on.

```powershell
$workbook = Get-OfficeExcel -Path $path -ReadOnly
$sheets = @($workbook.Sheets)
$summary = [pscustomobject]@{
    SheetCount = $sheets.Count
    TableCount = @(Get-OfficeExcelTable -Document $workbook).Count
    ChartCount = ($sheets | ForEach-Object { $_.Charts.Count } | Measure-Object -Sum).Sum
}
$sheetSummary = $sheets |
    Select-Object Name, UsedRangeA1, @{ Name = 'ChartCount'; Expression = { $_.Charts.Count } }
$workbook | Close-OfficeExcel

$summary
$sheetSummary
```

For the generated dashboard, the shape check reports:

- 6 sheets
- 6 tables
- 3 charts

The generated workbook also includes navigation links, evidence links, and a hidden notes sheet for audit context.

That makes the example useful in demos and CI logs. It also gives you a fast way to explain what a workbook contains without opening Excel.

For data-level checks, the same workbook can be read back with range and table readers:

```powershell
$serviceRows = Get-OfficeExcelRange `
    -Path $path `
    -Sheet 'Services' `
    -Range 'A1:H9'

$usedRange = Get-OfficeExcelUsedRange `
    -Path $path `
    -Sheet 'Services' `
    -AsDataTable

$namedRanges = Get-OfficeExcelNamedRange -Path $path
```

That gives you both kinds of validation: "is the workbook shaped correctly?" and "does the data still say what I expected?"

## From Windows Events To A Delivered Report

The workbook does not care where its objects came from. A scheduled Windows operations job can query PSEventViewer, write the detailed rows to Excel, create a compact PDF summary, and let Mailozaurr deliver both artifacts:

```powershell
Import-Module PSEventViewer
Import-Module PSWriteOffice
Import-Module Mailozaurr

$outputDirectory = (New-Item -ItemType Directory -Path (Join-Path $PSScriptRoot 'Output') -Force).FullName
$excelPath = Join-Path $outputDirectory 'System-Events.xlsx'
$pdfPath = Join-Path $outputDirectory 'System-Events.pdf'

$events = @(
    Get-EVXEvent `
        -LogName System `
        -TimePeriod Last24Hours `
        -ReadMode Message `
        -MaxEvents 500 |
        Select-Object TimeCreated, MachineName, ProviderName, Id, LevelDisplayName, Message
)

$events | Export-OfficeExcel `
    -Path $excelPath `
    -WorksheetName Events `
    -TableName SystemEvents `
    -AutoFit `
    -FreezeTopRow

New-OfficePdf -Path $pdfPath {
    PdfHeading 'System event summary'
    PdfParagraph "Collected $($events.Count) events during the last 24 hours."
    PdfTable -InputObject ($events | Select-Object -First 25 TimeCreated, MachineName, ProviderName, Id, LevelDisplayName)
}

$mailCredential = Get-Secret -Name 'Operations-Smtp-Credential'
Send-EmailMessage `
    -From 'reports@example.com' `
    -To 'operations@example.com' `
    -Subject 'Daily Windows event report' `
    -Text 'The detailed Excel workbook and review PDF are attached.' `
    -Attachment $excelPath, $pdfPath `
    -Server 'smtp.example.com' `
    -Credential $mailCredential `
    -UseSsl
```

Each module keeps one job. PSEventViewer owns bounded event-log queries and message projection. PSWriteOffice owns the editable workbook and fixed-layout summary. Mailozaurr owns authentication, transport, and attachments. None of the modules needs a special adapter for the others because ordinary PowerShell objects and file paths are the integration contract.

`Get-Secret` comes from Microsoft.PowerShell.SecretManagement. In a scheduled task or CI job, use the secret provider that environment already trusts rather than putting credentials in the report script.

## Performance And Scale

The dashboard is intentionally built around table and range operations because they scale better than cell-by-cell scripting.

- Use `ExcelTable -Data $objects` for rectangular datasets.
- Use formulas for values Excel should keep recalculating after the file is opened.
- Use table-based charts so the chart follows the data shape.
- Apply conditional formatting to ranges instead of formatting every cell in a loop.
- Keep read-back validation focused on summary counts, used ranges, table names, and critical values.
- Use hidden sheets for generation notes and audit metadata instead of writing separate sidecar files.

For larger inventories, split visible sheets by workflow: summary, details, ownership, trend, and notes. That keeps the workbook fast to open and easier to filter.

## More Workbook Ideas

Once the pattern is in place, the same commands can generate:

- inventory dashboards with asset detail, owner queue, and stale-data warnings
- security posture workbooks with risk scoring, evidence links, and action tracking
- migration trackers with validation lists, conditional formatting, and grouped owners
- service availability scorecards with trend charts and monthly snapshots
- workbook QA reports where read-back checks verify tables, charts, links, and hidden sheets

## Honest Compatibility Notes

This showcase intentionally avoids PivotTables and sparklines for now. The cmdlets exist, but generated packages still need an OfficeIMO desktop-open compatibility pass before they should appear in a flagship workbook.

The current dashboard focuses on a clean, useful, visually attractive workbook that validates and opens in desktop Excel.

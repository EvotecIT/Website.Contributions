---
title: "Build an Excel operational dashboard from PowerShell"
description: "Generate an Excel workbook with navigation, tables, formulas, charts, validation, conditional formatting, links, print setup, and desktop Office validation using PSWriteOffice."
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
image_alt: "Excel status chart exported from the generated operational dashboard workbook"
draft: true
---

An Excel report becomes useful when it gives people somewhere to start, somewhere to drill in, and enough formatting to spot risk without reading every row.

This showcase builds a multi-sheet operational dashboard from PowerShell objects. It includes a summary sheet, service detail table, trend sheet, owner summary, hidden notes sheet, table of contents, internal backlinks, smart evidence links, formulas, charts, validation, conditional formatting, and print setup.

![Summary status chart exported from Excel](./images/summary-status-chart.png)

## The Workbook

The full script lives in `Examples/Showcase/Showcase-Excel-OperationalDashboard.ps1`.

```powershell
New-OfficeExcel -Path $path {
    ExcelSheet 'Summary' {
        ExcelCell -Address 'A1' -Value 'Operational Dashboard'
        ExcelCell -Address 'B4' -Formula 'AVERAGE(Services!C2:C9)' -NumberFormat '0.0'
        ExcelTable -Data $legend -TableName 'StatusLegend' -StartRow 7 -StartColumn 1

        ExcelChart -Range 'A7:B10' -Row 7 -Column 6 -Type Doughnut -Title 'Status Meaning Mix' |
            Set-OfficeExcelChartLegend -Position Right |
            Set-OfficeExcelChartDataLabels -ShowValue $true -ShowCategoryName $true
    }
}
```

## Detail and Trend

The detail sheet uses table formatting and conditional rules, while the trend sheet gives a fast visual of availability, incidents, and automation:

![Service health chart exported from Excel](./images/service-health-chart.png)

![Trend chart exported from Excel](./images/trend-chart.png)

```powershell
ExcelSheet 'Services' {
    ExcelTable -Data $services -TableName 'ServiceHealth' -TableStyle 'TableStyleMedium9' -AutoFit
    ExcelValidationList -Range 'F2:F50' -Values 'Healthy','Watch','Risk'
    ExcelConditionalColorScale -Range 'C2:C9' -StartColor '#F8696B' -EndColor '#63BE7B'
    ExcelConditionalDataBar -Range 'D2:D9' -Color '#5B9BD5'
    ExcelUrlLinksByHeader -Header 'Evidence' -TableName 'ServiceHealth' -UrlScript {
        param($text) "https://evotec.xyz/docs/$text"
    }
}
```

The script also ends with a workbook summary check so the example proves its own shape:

```powershell
$summary = Get-OfficeExcelSummary -Path $path -IncludeSheets
$summary | Select-Object SheetCount, TableCount, ChartCount, HyperlinkCount, NamedRangeCount
$summary.Sheets | Select-Object Name, UsedRange, TableCount, ChartCount
```

## Wrap-Up

The generated workbook is validated with Open XML and opened with desktop Excel during the showcase pass. Pivot tables and sparklines are intentionally left out of this draft workbook until their desktop Office compatibility is fixed, because a flagship example should open cleanly.

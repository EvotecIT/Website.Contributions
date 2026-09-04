---
title: "Inspect and convert mixed Office archives with PSWriteOffice"
description: "A cautious PowerShell workflow for inspecting incoming files, converting Apple iWork and offline OneNote content, making scanned PDFs searchable, and keeping fidelity evidence beside the results."
date: "2026-09-04"
language: "en"
authors:
  - przemyslaw-klys
categories:
  - PowerShell
  - Office
  - Automation
tags:
  - pswriteoffice
  - officeimo
  - iwork
  - onenote
  - pdf
  - ocr
  - security
  - provenance
image: "./cover.png"
image_alt: "Records specialist comparing a scanned document with a structured inspection report on a laptop"
draft: true
---

Document migrations rarely begin with a tidy folder of current Word and Excel files. They begin with the files people actually kept: a Pages proposal from an old Mac, a Numbers workbook, an offline OneNote section, several scanned PDFs, and a few macro-enabled documents nobody wants to open just to learn what they contain.

The useful first step is not “convert everything.” It is to identify the format, inspect what can be inspected without launching the desktop application, choose an output people can review, and keep notes about anything that could not be reconstructed exactly.

[PSWriteOffice](https://github.com/EvotecIT/PSWriteOffice) provides the PowerShell commands in this workflow. [OfficeIMO](https://github.com/EvotecIT/OfficeIMO) owns the parsers, document models, conversion reports, package policies, and OCR contracts underneath them. That split keeps the script readable while the result still carries structured evidence.

> This draft targets the upcoming PSWriteOffice build backed by OfficeIMO 3.4. Keep it as a draft until those packages and commands are released together.

## Start with an inventory

Begin by separating known document families from files that need a manual decision:

```powershell
$sourceRoot = 'C:\Archive\Incoming'

$inventory = Get-ChildItem -LiteralPath $sourceRoot -File -Recurse | ForEach-Object {
    $extension = $_.Extension.ToLowerInvariant()
    $workflow = switch ($extension) {
        '.pages'   { 'iWork' }
        '.numbers' { 'iWork' }
        '.key'     { 'iWork' }
        '.one'     { 'OneNote' }
        '.onetoc2' { 'OneNote' }
        '.onepkg'  { 'OneNote' }
        '.pdf'     { 'PDF' }
        '.docx'    { 'OpenXml' }
        '.docm'    { 'OpenXml' }
        '.xlsx'    { 'OpenXml' }
        '.xlsm'    { 'OpenXml' }
        '.pptx'    { 'OpenXml' }
        '.pptm'    { 'OpenXml' }
        default    { 'ManualReview' }
    }

    [pscustomobject]@{
        Path = $_.FullName
        Extension = $extension
        Bytes = $_.Length
        Workflow = $workflow
    }
}

$inventory | Group-Object Workflow | Sort-Object Name | Format-Table Name, Count
```

That inventory is intentionally mundane. It gives the migration a stable input list, makes unsupported formats visible, and prevents a broad recursive conversion from quietly skipping files.

## Inspect packages before opening them

For Open XML and compound Office files, inspect the package structure before deciding whether it belongs in an automated conversion lane:

```powershell
$packageReport = Get-OfficePackageSecurity `
    -Path 'C:\Archive\Incoming\Quarterly.xlsm' `
    -Untrusted

$packageReport | Select-Object `
    IsValid,
    ContainerKind,
    MacroPartCount,
    EmbeddedPayloadPartCount,
    ExternalRelationshipCount

$packageReport.Findings |
    Format-Table Severity, Rule, PartName, Message -AutoSize
```

`-Untrusted` applies bounded defaults that reject macros, embedded payloads, ActiveX, and external relationships. The report is an observation about the package, not a malware verdict. A clean structural report does not replace endpoint scanning, content policy, or a protected review environment.

Use `-ThrowOnViolation` when the script should stop instead of routing the file to a review queue:

```powershell
Get-OfficePackageSecurity `
    -Path 'C:\Archive\Incoming\Quarterly.xlsm' `
    -Untrusted `
    -ThrowOnViolation
```

The command reads the package; it does not launch Word, Excel, PowerPoint, macros, or embedded programs.

## Record provenance as evidence, not authorship

Content Credentials and related signals can help explain where a published asset came from, but their presence or absence does not tell the whole story. Keep structural discovery, text-integrity observations, and cryptographic verification separate:

```powershell
$provenance = Get-OfficeProvenance -Path 'C:\Archive\Incoming\cover.png'

$provenance.Structural
$provenance.TextIntegrity
$provenance.Verification
```

The default path is local and inspect-first. If your environment has an approved `c2patool` installation, opt into that verifier explicitly:

```powershell
$provenance = Get-OfficeProvenance `
    -Path 'C:\Archive\Incoming\cover.png' `
    -C2paToolPath 'C:\Tools\c2patool.exe'

$provenance.Verification |
    Select-Object Status, ProviderName, Findings
```

Treat `ProviderUnavailable`, missing credentials, and an unsigned asset as distinct outcomes. None of them should be rewritten as “human-made” or “AI-made.”

## Convert Pages, Numbers, and Keynote with a report

Modern iWork files are packages rather than simple single-document XML files. Inspect the detected kind first:

```powershell
$source = Get-OfficeIWork -Path 'C:\Archive\Incoming\Quarterly.numbers'

$source | Select-Object Kind, ContainerKind, BuildVersions
$source.ReadNumbers().Sheets | Select-Object Name
```

Then convert to the matching editable Office format and keep the conversion report:

```powershell
$report = ConvertFrom-OfficeIWork `
    -Path 'C:\Archive\Incoming\Quarterly.numbers' `
    -OutputPath 'C:\Archive\Review\Quarterly.xlsx' `
    -PassThruReport

$report | Select-Object `
    SourceKind,
    ProjectionKind,
    ReconstructedItemCount,
    HasLoss,
    Diagnostics
```

The mappings are deliberate:

| Source | Editable destination |
| --- | --- |
| Pages | Word `.docx` |
| Numbers | Excel `.xlsx` |
| Keynote | PowerPoint `.pptx` |

“Editable” does not mean pixel-identical. iWork and Microsoft Office have different layout, chart, animation, and object models. `-FailOnLoss` is useful for a strict lane where any flattened, omitted, or preserved-only structure must go to manual review:

```powershell
ConvertFrom-OfficeIWork `
    -Path 'C:\Archive\Incoming\Board.key' `
    -OutputPath 'C:\Archive\Review\Board.pptx' `
    -FailOnLoss
```

For normal migrations, keeping the report is often more useful than pretending every source feature has a direct equivalent.

## Turn offline OneNote into reviewable files

PSWriteOffice can read a `.one` section, `.onetoc2` notebook index, or `.onepkg` archive without automating the OneNote desktop application:

```powershell
$section = Get-OfficeOneNote -Path 'C:\Archive\Incoming\Operations.one'

$section.Pages |
    Select-Object Title, CreatedUtc, LastModifiedUtc
```

Markdown is convenient for search, version control, and text review:

```powershell
$markdownReport = ConvertFrom-OfficeOneNote `
    -Path 'C:\Archive\Incoming\Operations.one' `
    -OutputPath 'C:\Archive\Review\Operations.md' `
    -PassThruReport

$markdownReport | Select-Object HasLoss, Diagnostics
```

HTML is useful for browser review, while PDF provides a fixed-layout handoff:

```powershell
ConvertFrom-OfficeOneNote `
    -Path 'C:\Archive\Incoming\Operations.onepkg' `
    -OutputPath 'C:\Archive\Review\Operations.html'

$pdfEvidence = ConvertFrom-OfficeOneNote `
    -Path 'C:\Archive\Incoming\Operations.onepkg' `
    -OutputPath 'C:\Archive\Review\Operations.pdf' `
    -PassThruReport
```

OneNote is a free-form canvas. Page hierarchy, text, and many common assets can be projected, but exact placement, history, ink, embedded data, or application-specific objects may not have an equivalent in Markdown, HTML, or PDF. The report is part of the output, not an apology hidden in a log.

## Make scanned PDFs searchable

For an image, return recognized text directly or keep the provider and confidence evidence:

```powershell
$ocr = Get-OfficeImageText `
    -Path 'C:\Archive\Incoming\Scan-001.png' `
    -Language English, Polish `
    -PassThru

$ocr | Select-Object Text, Confidence, Provider, Model, Diagnostics
```

For a scanned PDF, preserve the visible pages and add geometry-aligned invisible text:

```powershell
$searchable = ConvertTo-OfficePdfSearchable `
    -Path 'C:\Archive\Incoming\Invoices.pdf' `
    -OutputPath 'C:\Archive\Review\Invoices-Searchable.pdf' `
    -Language English, Polish `
    -MinimumConfidence 0.70 `
    -PassThru

$searchable | Select-Object ModifiedPages, Ocr
```

The OCR path discovers a local Tesseract runtime and records what it used. Recognition quality still depends on scan resolution, skew, language data, handwriting, typography, and page complexity. Searchable text should make review easier; it should not silently become authoritative invoice or identity data.

## Keep a migration manifest

At the end of each file, write one small record that links the source, output, decision, and evidence:

```powershell
$migrationRecord = [pscustomobject]@{
    SourcePath = 'C:\Archive\Incoming\Quarterly.numbers'
    OutputPath = 'C:\Archive\Review\Quarterly.xlsx'
    SourceKind = $report.SourceKind
    HasLoss = $report.HasLoss
    Diagnostics = @($report.Diagnostics)
    ConvertedUtc = [datetime]::UtcNow
}

$migrationRecord |
    ConvertTo-Json -Depth 8 |
    Set-Content -LiteralPath 'C:\Archive\Review\Quarterly.conversion.json' -Encoding utf8
```

For a larger archive, append those records to a database or a proper manifest store instead of one JSON file per document. The important part is that a successful command, a structurally safe package, a verified provenance carrier, and a loss-free conversion remain separate facts.

The resulting workflow is cautious without being ceremonial:

1. Inventory the files.
2. Inspect supported packages before opening active content.
3. Record provenance evidence without guessing authorship.
4. Convert iWork and OneNote into reviewable formats.
5. Add searchable text to scans where it helps.
6. Keep fidelity and diagnostic evidence beside every result.

That gives people useful documents while leaving a trail that explains what the automation actually knew.

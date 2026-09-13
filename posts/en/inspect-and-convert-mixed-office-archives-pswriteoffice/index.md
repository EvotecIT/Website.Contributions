---
title: "Inspect and convert mixed Office archives with PSWriteOffice"
description: "Inspect a mixed document archive before converting iWork, OneNote, and scanned PDF files, and keep a useful record of what changed."
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

Document migration folders are rarely as tidy as the plan. Alongside current Word and Excel files there is usually an old Pages proposal, a Numbers workbook, an offline OneNote section, a stack of scanned PDFs, and a macro-enabled document nobody wants to open just to identify it.

My first step is not "convert everything." I inventory the files, inspect what I can without launching the desktop application, choose an output somebody can review, and record the parts that could not be reconstructed exactly.

[PSWriteOffice](https://github.com/EvotecIT/PSWriteOffice) provides the PowerShell commands used here. [OfficeIMO](https://github.com/EvotecIT/OfficeIMO) contains the parsers, document models, conversion reports, package policies, and OCR contracts behind them. In practice that means the PowerShell script stays readable while each result can still carry structured evidence.

These commands are available in [PSWriteOffice 3.0.7](https://www.powershellgallery.com/packages/PSWriteOffice/3.0.7) and later. Use PowerShell 7 and install the current module before starting:

```powershell
Install-Module PSWriteOffice -Scope CurrentUser -Force
Import-Module PSWriteOffice
```

The paths are examples. Create the review output directory and start with representative copies of your files. OCR needs a local Tesseract runtime plus the requested English and Polish language data. Cryptographic provenance verification uses the optional `c2patool` executable shown later.

## Start with an inventory

I begin by separating known document families from files that need a manual decision:

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

The inventory is deliberately boring. It gives the migration a stable input list, shows unsupported formats, and prevents a recursive conversion from quietly skipping files.

## Inspect packages before opening them

For Open XML and compound Office files, I inspect the package before deciding whether it belongs in the automated lane:

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

`-Untrusted` applies bounded defaults and rejects macros, embedded payloads, ActiveX, and external relationships. The report describes the package structure; it is not a malware verdict. A clean result does not replace endpoint scanning, content policy, or a protected review environment.

Use `-ThrowOnViolation` when the script should stop instead of routing the file to a review queue:

```powershell
Get-OfficePackageSecurity `
    -Path 'C:\Archive\Incoming\Quarterly.xlsm' `
    -Untrusted `
    -ThrowOnViolation
```

The command reads the package without launching Word, Excel, PowerPoint, macros, or embedded programs.

## Record provenance as evidence, not authorship

Content Credentials and related signals can add useful provenance evidence, but their presence or absence is not a complete authorship answer. I keep structural discovery, text-integrity observations, and cryptographic verification as separate results:

```powershell
$provenance = Get-OfficeProvenance -Path 'C:\Archive\Incoming\cover.png'

$provenance.Structural
$provenance.TextIntegrity
$provenance.Verification
```

The default path inspects local structure. `Verification` can be null when no verifier was requested, because finding a manifest and verifying its signature are different operations. If your environment has an approved `c2patool` installation, enable that verifier explicitly:

```powershell
$provenance = Get-OfficeProvenance `
    -Path 'C:\Archive\Incoming\cover.png' `
    -C2paToolPath 'C:\Tools\c2patool.exe'

$provenance.Verification |
    Select-Object Status, ProviderName, Findings
```

Treat `ProviderUnavailable`, missing credentials, and an unsigned asset as different outcomes. None of them proves that the content was "human-made" or "AI-made."

## Convert Pages, Numbers, and Keynote with a report

Modern iWork files are packages rather than one simple XML document. Check the detected kind first:

```powershell
$source = Get-OfficeIWork -Path 'C:\Archive\Incoming\Quarterly.numbers'

$source | Select-Object Kind, ContainerKind, BuildVersions
$source.ReadNumbers().Sheets | Select-Object Name
```

Then convert to the matching editable Office format and keep the report:

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

"Editable" does not mean pixel-identical. iWork and Microsoft Office have different layout, chart, animation, and object models. In a strict migration, `-FailOnLoss` sends any flattened, omitted, or preserved-only structure to manual review:

```powershell
ConvertFrom-OfficeIWork `
    -Path 'C:\Archive\Incoming\Board.key' `
    -OutputPath 'C:\Archive\Review\Board.pptx' `
    -FailOnLoss
```

For a normal migration, I would rather keep an honest report than pretend every source feature has a direct equivalent.

## Turn offline OneNote into reviewable files

PSWriteOffice can read a `.one` section, `.onetoc2` notebook index, or `.onepkg` archive without automating the OneNote desktop application:

```powershell
$section = Get-OfficeOneNote -Path 'C:\Archive\Incoming\Operations.one'

$section.Pages |
    Select-Object Title, CreatedUtc, LastModifiedUtc
```

For search, version control, and text review, Markdown is usually my first output:

```powershell
$markdownReport = ConvertFrom-OfficeOneNote `
    -Path 'C:\Archive\Incoming\Operations.one' `
    -OutputPath 'C:\Archive\Review\Operations.md' `
    -PassThruReport

$markdownReport | Select-Object HasLoss, Diagnostics
```

HTML works well for browser review, while PDF gives me a fixed-layout handoff:

```powershell
ConvertFrom-OfficeOneNote `
    -Path 'C:\Archive\Incoming\Operations.onepkg' `
    -OutputPath 'C:\Archive\Review\Operations.html'

$pdfEvidence = ConvertFrom-OfficeOneNote `
    -Path 'C:\Archive\Incoming\Operations.onepkg' `
    -OutputPath 'C:\Archive\Review\Operations.pdf' `
    -PassThruReport
```

OneNote is a free-form canvas. Page hierarchy, text, and many common assets can be projected, but exact placement, history, ink, embedded data, and application-specific objects may have no equivalent in Markdown, HTML, or PDF. Keep that information in the report where the reviewer can see it, rather than hiding it in a conversion log.

## Make scanned PDFs searchable

For one image, either return the recognized text directly or keep the provider and confidence details:

```powershell
$ocr = Get-OfficeImageText `
    -Path 'C:\Archive\Incoming\Scan-001.png' `
    -Language English, Polish `
    -PassThru

$ocr | Select-Object Text, Confidence, Provider, Model, Diagnostics
```

For a scanned PDF, I preserve the visible pages and add geometry-aligned invisible text:

```powershell
$searchable = ConvertTo-OfficePdfSearchable `
    -Path 'C:\Archive\Incoming\Invoices.pdf' `
    -OutputPath 'C:\Archive\Review\Invoices-Searchable.pdf' `
    -Language English, Polish `
    -MinimumConfidence 0.70 `
    -PassThru

$searchable | Select-Object ModifiedPages, Ocr
```

The OCR path discovers a local Tesseract runtime and records what it used. Results still depend on scan resolution, skew, language data, handwriting, typography, and page complexity. I use the text to make review and search easier, not as authoritative invoice or identity data.

## Keep a migration manifest

For each file, I write one small record that links the source, output, decision, and evidence:

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

For a larger archive, store the records in a database or manifest instead of creating one JSON file per document. Keep the conclusions separate: a successful command, a structurally safe package, a verified provenance carrier, and a loss-free conversion are four different facts.

The workflow I use is:

1. Inventory the files.
2. Inspect supported packages before opening active content.
3. Record provenance evidence without guessing authorship.
4. Convert iWork and OneNote into reviewable formats.
5. Add searchable text to scans where it helps.
6. Keep fidelity and diagnostic evidence beside every result.

The result is a set of files people can review and a record that explains what the automation found, changed, and could not prove.

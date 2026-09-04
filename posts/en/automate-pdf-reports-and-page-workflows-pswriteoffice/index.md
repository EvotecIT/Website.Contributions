---
title: "Compose PDF reports and automate page workflows with PSWriteOffice"
description: "Create flowing PDF reports, position text at exact coordinates, add forms and attachments, reorder pages, and extract content from PowerShell."
date: "2026-08-19"
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
  - pdf
  - reporting
image: "./cover.png"
image_alt: "Operations specialist checking a PDF review pack with forms, attachments, and reordered pages"
draft: true
---

PDF automation usually means one of two things. You are either composing a new fixed-layout document from data, or operating on a PDF that already exists. Those jobs need different tools.

PSWriteOffice exposes both through the OfficeIMO PDF engine. The composition DSL handles headings, paragraphs, rich text, tables, lists, images, forms, headers, footers, bookmarks, and attachments. Focused commands handle existing files: inspect, extract, merge, split, reorder, stamp, overlay, redact, sanitize, optimize, and exchange form data.

The distinction matters because a flowing paragraph should not require coordinates, while a review stamp at an exact page position should not pretend to be ordinary document content.

## Compose A Report In Document Flow

Start with the DSL when the script owns the report. The page layout engine places flowing content and handles page breaks as the report grows.

```powershell
$findings = @(
    [pscustomobject]@{ Severity = 'High'; Finding = 'Dormant administrator account'; Owner = 'Identity' }
    [pscustomobject]@{ Severity = 'Medium'; Finding = 'Missing evidence link'; Owner = 'Operations' }
)

PdfNew -Path '.\Access-Review.pdf' {
    PdfTheme Report
    PdfHeading 'Access review'
    PdfParagraph 'Prepared for the weekly security review.'
    PdfTable -InputObject $findings
    PdfText 'Review status: Draft' -Italic -Color '#B45309'
}
```

This block uses the short PDF aliases consistently. The canonical equivalents are `New-OfficePdf`, `Set-OfficePdfTheme`, `Add-OfficePdfHeading`, `Add-OfficePdfParagraph`, `Add-OfficePdfTable`, and `Add-OfficePdfText`. Pick one command style for a composition block instead of mixing both.

Saved constructors are quiet by default. You do not need `Out-Null` or a suppression switch. Add `-PassThru` only when the next command needs the saved file.

## Format One Line Without Building Paragraphs By Hand

Use a text run when formatting changes inside one line. The columnar form keeps the content readable when values come from an object:

```powershell
$finding = [pscustomobject]@{
    Owner = 'Identity'
    Due = '2026-08-28'
    Severity = 'High'
}

PdfNew -Path '.\Finding-Summary.pdf' {
    PdfHeading 'Finding summary'
    PdfText -Run @{
        Text  = 'Owner: ', $finding.Owner, '    Due: ', $finding.Due, '    Severity: ', $finding.Severity
        Bold  = $true, $false, $true, $false, $true, $false
        Color = $null, $null, $null, $null, $null, 'Crimson'
    }
}
```

A scalar style value applies to every segment. A style array must have one value or the same number of values as `Text`, so a missing entry cannot silently shift formatting onto the wrong content.

## Position Text At Exact Coordinates

`PdfText` and `PdfParagraph` belong to normal document flow. They intentionally do not have `X` and `Y` parameters. When text must start at a fixed coordinate, add canvas content to an existing PDF:

```powershell
Add-OfficePdfCanvas -Path '.\Access-Review.pdf' -OutputPath '.\Access-Review-Positioned.pdf' -Content {
    PdfCanvasText -Run @(
        TextRun 'Owner: ' -Bold
        TextRun 'Platform' -Color '#0F766E'
        TextRun '  |  REVIEW COPY' -Italic
    ) -X 36 -Y 24 -FontSize 10
}
```

Canvas coordinates use PDF points from the visual top-left of the page. Use this surface for page labels, registration marks, fixed headers, signatures, or generated overlays. For a single text or image mark, `Add-OfficePdfStamp` is shorter. For a complete imported page, use `Add-OfficePdfPageOverlay`.

The canvas command accepts normal strings and rich text runs directly. There is no need to create a runtime-typed .NET array in PowerShell.

## Add Forms And Exchange Their Data

Form fields can be part of the original composition:

```powershell
PdfNew -Path '.\Change-Request.pdf' {
    PdfHeading 'Change request'
    PdfText 'Owner'
    PdfFormField -Name Owner -Type Text -Value 'Platform' -Width 280
    PdfText 'Decision'
    PdfFormField -Name Decision -Type Choice -Options Approve,Reject,Defer -Value Defer -Width 220
}
```

XFDF keeps the field values separate from the document when another system needs to exchange or archive them:

```powershell
Export-OfficePdfXfdf -Path '.\Change-Request.pdf' -OutputPath '.\Change-Request.xfdf'
Import-OfficePdfXfdf -Path '.\Change-Request.pdf' -XfdfPath '.\Change-Request.xfdf' -OutputPath '.\Change-Request-Updated.pdf'
```

PSWriteOffice can also inspect fields, flatten them deliberately, and work with annotations. Flatten only when the delivery copy should no longer be interactive.

## Attach Evidence To The Delivery Copy

An attachment keeps supporting material with the report:

```powershell
PdfNew -Path '.\Audit-Report.pdf' {
    PdfTheme Report
    PdfHeading 'Audit report'
    PdfParagraph 'The supporting evidence is embedded in this PDF.'
    PdfAttachment `
        -Path '.\Evidence-Summary.pdf' `
        -Name 'evidence-summary.pdf' `
        -MimeType 'application/pdf' `
        -Relationship Data `
        -Description 'Supporting review evidence'
}
```

Attachments are useful for evidence packs, electronic invoices, source data, or signed supporting documents. They remain separate files inside the PDF rather than being rendered as visible pages.

## Reorder, Merge, And Split Existing Pages

Page operations do not require recomposing the source document. This moves the approval page to the front and writes a new file:

```powershell
Move-OfficePdfPage `
    -Path '.\Review-Pack.pdf' `
    -PageRange '3' `
    -BeforePage 1 `
    -OutputPath '.\Review-Pack-Reordered.pdf'
```

Use `Join-OfficePdf` to assemble a pack and `Split-OfficePdf` to separate it by page range or pages per document. Keep the source file while validating a transformation; a successful write does not prove that every advanced PDF feature was preserved.

## Export An Office Document To PDF Deliberately

Word, Excel, PowerPoint, Markdown, and RTF creation no longer hide a PDF side effect inside `New-*` or `Save-*`. Create the source artifact first, then request the delivery format explicitly:

```powershell
New-OfficeWord -Path '.\Service-Review.docx' {
    WordParagraph -Text 'Service review' -Style Heading1
    WordParagraph 'This editable report is the source artifact.'
    WordTable -InputObject $findings -Layout AutoFitToWindow
}

Export-OfficeDocumentPdf `
    -InputPath '.\Service-Review.docx' `
    -Path '.\Service-Review.pdf'
```

This is the one deliberate `InputPath` exception in the PSWriteOffice public API. `Path` names the PDF being produced, so `InputPath` makes the source role unambiguous. Other commands use `Path` for their primary file, `OutputPath` for a transformed copy, and `DestinationPath` for a copy destination.

## Extract And Inspect Before Changing

Read-only commands make PDFs useful in search, compliance, and ingestion workflows:

```powershell
$pages = Get-OfficePdfText -Path '.\Policy.pdf' -ByPage
$pages | Select-Object PageNumber, Text
```

Other commands inspect document information, fonts, images, attachments, form fields, annotations, signatures, compliance, interactions, optimization opportunities, and rewrite safety. Use that evidence before redaction, sanitization, optimization, or destructive page changes.

For sensitive content, build a redaction plan from detected text and write a new delivery copy. Do not confuse a visual rectangle with removing the underlying text. `ConvertTo-OfficePdfRedacted` applies actual redaction through the PDF engine.

## Deliver The Result With Mailozaurr

Once the PDF is accepted, Mailozaurr can send the delivery copy without making PSWriteOffice own SMTP or Microsoft Graph:

```powershell
$mailCredential = Get-Secret -Name 'Reporting-Smtp-Credential'

Send-EmailMessage `
    -From 'reports@example.com' `
    -To 'reviewers@example.com' `
    -Subject 'Access review' `
    -Text 'The editable source and PDF delivery copy are attached.' `
    -Attachment '.\Service-Review.docx', '.\Service-Review.pdf' `
    -Server 'smtp.example.com' `
    -Credential $mailCredential `
    -UseSsl
```

Mailozaurr owns message composition, authentication, transport, and mailbox operations. PSWriteOffice owns the generated document artifacts. That boundary also keeps provider choices out of reporting code: the same files can be delivered through SMTP, Microsoft Graph, Gmail, SendGrid, Mailgun, or Amazon SES by changing the Mailozaurr connection layer.

`Get-Secret` comes from Microsoft.PowerShell.SecretManagement. Keep the credential in the secret provider used by the scheduled job or CI runner, not in the report script.

## Choose The Smallest Surface

| Job | Surface |
| --- | --- |
| New report with flowing content | PDF DSL |
| Mixed formatting inside a line | `PdfText -Run` |
| Exact page coordinates | Canvas or stamp |
| Embed supporting files | Attachment DSL |
| Interactive input | Form fields and XFDF |
| Combine or rearrange supplied PDFs | Join, split, move, copy, or remove page commands |
| Search or compliance evidence | Text extraction, inspection, diagnostics, and preflight |
| Safe delivery copy | Redaction, sanitization, optimization, flattening, or signing commands |

The [PSWriteOffice PDF recipes](https://github.com/EvotecIT/PSWriteOffice/tree/main/Examples/Pdf) contain complete scripts for invoices, audit reports, forms, attachments, extraction, page reordering, merging, splitting, positioned content, redaction, sanitization, and preflight. The examples use simple local file names so you can run them first and replace the sample data afterward.

PDF is a fixed-layout destination, but the PowerShell workflow does not need to feel fixed. Use semantic composition for the report, exact coordinates only where they carry meaning, and focused commands for the document operations that follow.

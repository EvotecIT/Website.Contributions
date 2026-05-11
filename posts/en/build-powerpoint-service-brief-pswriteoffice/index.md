---
title: "Build a beautiful PowerPoint service brief from PowerShell"
description: "Create an editable PowerPoint service brief from PowerShell objects using PSWriteOffice and the OfficeIMO PowerPoint designer deck-plan APIs."
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
  - powerpoint
  - presentations
image: "./cover.png"
image_alt: "Title slide exported from the generated PowerPoint service brief"
draft: true
---

PowerPoint automation is painful when every slide is a pile of coordinates. The newer OfficeIMO PowerPoint designer APIs make it possible to describe a deck semantically, then let the engine compose the slides.

This showcase builds an editable service brief from PowerShell objects. It uses a deck plan for section, process, card grid, coverage, capability, and case-study slides, then adds chart and table slides with speaker notes, transitions, and sections.

![Generated PowerPoint title slide](./cover.png)

## A Semantic Deck Plan

The full script lives in `Examples/Showcase/Showcase-PowerPoint-ServiceBrief.ps1`.

```powershell
$plan = PptDeckPlan {
    PptPlanSection -Title 'PSWriteOffice Showcase' -Subtitle 'Beautiful, useful Office artifacts from PowerShell'
    PptPlanProcess -Title 'Showcase Delivery Path' -Steps $process
    PptPlanCardGrid -Title 'Product Coverage' -Cards $cards
    PptPlanCoverage -Title 'Artifact Map' -Locations $coverage
    PptPlanCapability -Title 'What the DSL should make easy' -Sections $capabilities
}
```

The deck is rendered through the designer bridge:

```powershell
New-OfficePowerPoint -Path $path {
    PptSlideSize -Preset Screen16x9
    PptDesignerDeck -Plan $plan `
        -AccentColor '#008C95' `
        -Seed 'pswriteoffice-showcase' `
        -Purpose 'technical service brief' `
        -CreativeDirectionPack TechnicalMap `
        -LayoutStrategy ContentFirst
}
```

![Generated process slide](./images/process-slide.png)

## Still Editable

After the designer plan slides are created, the script adds ordinary PowerPoint chart and table slides:

![Generated chart slide](./images/chart-slide.png)

That split matters. The deck gets a polished starting point, but the output remains an editable `.pptx` with real slides, charts, notes, and sections.

## Wrap-Up

`PSWriteOffice` now exposes an initial bridge to the OfficeIMO PowerPoint designer layer. The next useful polish is richer diagnostics, metric/visual-frame helpers, and shape-layout commands for manual slide refinement.

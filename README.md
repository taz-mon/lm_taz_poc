# Publishing PDFs from an Oxygen DCPP Sample

This is a small DITA documentation project that builds two product spec PDFs through Oxygen XML Editor's CSS-based publishing pipeline (DCPP), with a GitHub Actions workflow that runs the same build headlessly in CI.

I built this repo as a job-search writing sample to demonstrate structured authoring and publishing-pipeline skills end to end. It includes:

- DITA content design
- custom CSS-based PDF styling
- A CI/CD workflow 

It requires two Oxygen licenses, one for an Oxygen Editor on your local system to create your artifacts, and one for the Oxygen Publishing Engine so you can publish your artifacts using a GitHub workflow as I demonstrate here. I want to shout out to `sales@oxygenxml.com` for providing trial licenses to anyone who wants to sharpen their DITA skills without making the purchasing leap.

## What's in this repo

| Output | Layout | Chapters | Notable feature |
|--------|--------|----------|------------------|
| `passage_l20.pdf` | Single-column | 5 | Standard chapter flow with dynamic footer |
| `passage_l200.pdf` | Three-column specs page | 3 | `outputclass="three_columns"` on `specifications.dita` |

The project lets you publish both PDFs from the same DITA source set and the same publishing template, using one CSS/template pair driving two distinct layouts.

## Repo structure

```
.
├── maps/
│   ├── passage_l20.ditamap       # Bookmap, 5 chapters, single-column
│   └── passage_l200.ditamap      # Bookmap, 3 chapters, three-column specs page
├── common/
│   └── global_variables.ditamap  # Key definitions: trademarks (plain + -tm variants),
│                                  # product names, reused across both maps
├── topics/
│   └── passage_l200/
│       ├── overview.dita
│       ├── specifications.dita   # outputclass="three_columns"
│       └── variants.dita
├── templates/
│   └── lightmatter/
│       ├── lightmatter.opt           # Single publishing template that both products reference
│       ├── lightmatter-pdf.css       # Single CSS file driving both PDFs
│       ├── lightmatter@logotyp.us.svg
│
└── .github/
    └── workflows/
        └── publish.yml            # Headless DCPP build, manually triggered
```

## Why DITA + DCPP

This project uses DITA's map/key-reference model and Oxygen's CSS-based publishing pipeline (DCPP). Here's why (for anyone evaluating my structured-authoring choices):

**Single source, two outputs.** Both bookmaps reference the same `common/global_variables.ditamap` through `<mapref>` in `<frontmatter>`, so the trademark string I reference (plain and `-tm` variants) and product name are defined once and used in topics with `<keyword>` references. This approach makes it to easy manage trademarking and for managing product references that appear many times in a topic.

**Layout as metadata.** The three-column specs layout is triggered by the `<topic outputclass="three_columns">` attribute. For this project you can see it in the `topics/passage_l200/specifications.dita` file. The CSS defines an `@page three_column_page` rule with `column-count: 3`; the DITA source doesn't know or care about columns. This keeps the content model (topic) clean and the layout decision entirely in the presentation layer (css file).

**Dynamic footers from content.** The footer pulls the product name directly from the bookmap title at render time using `oxy_xpath('//*[contains(@class, "bookmap/mainbooktitle")]/text()')`. Neither PDF needs product-specific CSS overrides because of this. One limitation: this only works if mainbooktitle is literal text. When I tried referencing it through a keyref variable instead, the XPath returned nothing, so for now both bookmaps use literal titles rather than key references for this one element.

**Path A vs. Path B.** I chose the DCPP process (CSS-based, Oxygen v28+) over an older XSL-FO `pdf2` transform that I knew about in a previous job. For more information, see [Customizing PDF Output Using CSS](https://www.oxygenxml.com/doc/versions/28.1/ug-ope/topics/dcpp_the_customization_css.html). This latest CSS approach works really well and is easy to implement without the complexity you need to grasp in XSL-FO's fine-grained pagination control.

## The CI/CD workflow

`publish.yml` runs the identical DCPP build that runs locally in Oxygen XML Editor, but headless on a GitHub-hosted runner, triggered on demand:

```bash
gh workflow run publish.yml
gh run watch
```

See [`PUBLISHING_WORKFLOW.md`](./PUBLISHING_WORKFLOW.md) for the full breakdown of what the workflow does and the licensing prerequisites. Note that the Oxygen Publishing Engine requires a separate license from the Oxygen XML Editor desktop app, which is the single biggest gotcha for anyone trying to reuse this pattern.

### What this workflow demonstrates

Getting from "DITA source and a local Oxygen transformation scenario" to "headless, reproducible PDF builds in CI" surfaced a series of real, undocumented failure modes rather than a clean first-try setup. That troubleshooting trail is part of the portfolio value here:

- No prebuilt GitHub Action exists for Oxygen's Publishing Engine; the workflow downloads and drives the engine directly, the same way Oxygen's own build tooling does.
- The correct download URL, launcher path (`bin/dita`, not a top-level `dita` command), and transtype (`pdf-css-html5`) all came from inspecting the actual engine distribution and Oxygen's own `build.gradle`, not from published CI examples.
- The Oxygen Publishing Engine license is a separate product from the Oxygen XML Editor license, and the failure mode when this is missing (`You must have a valid Oxygen PDF Chemistry license key`) gives no indication of *why*.
- The license file format is a full multi-line metadata block, not the bare signature string you paste into your local Oxygen application to activate the license.
- You need the `-Dpdf.publishing.template` flag, otherwise the final output fails silently: the build succeeds, but the output reverts to unstyled defaults with no error, which defeats the purpose of having a customized, nicely-stylized PDF.

## Running it yourself

**Locally in Oxygen XML Editor (v28+):**
Open either `.ditamap` in the Project view and run the corresponding transformation scenario (`LightmatterPDF_L20` or `LightmatterPDF-l200`), both configured to use `templates/lightmatter/lightmatter.opt`.

**In CI:**
Requires an `OXYGEN_LICENSE_KEY` repository secret (see [`PUBLISHING_WORKFLOW.md`](./PUBLISHING_WORKFLOW.md) for the exact format), then:

```bash
gh workflow run publish.yml
gh run watch
gh run download <run-id>
```

Both PDFs are uploaded as downloadable workflow artifacts on successful completion.

## Background

Source content for both products is adapted from [lightmatter.co](https://lightmatter.co) for sample/portfolio purposes only; this is not an official Lightmatter publication.

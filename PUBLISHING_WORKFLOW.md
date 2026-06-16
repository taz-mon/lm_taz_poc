# Publishing PDFs via GitHub Actions

I built a GitHub Actions workflow (`.github/workflows/publish.yml`) with Claude for this repo as part of a job application writing sample I did. It builds two product spec PDFs using Oxygen's DCPP (CSS-based) publishing pipeline, the same engine used by Oxygen XML Editor's transformation scenarios, running headless on a GitHub-hosted runner.

## What the workflow does

1. Checks out the repo.
2. Sets up Java 17 (required by the Oxygen Publishing Engine).
3. Downloads and unzips the Oxygen Publishing Engine.
4. Writes a license key file required to run Chemistry (the PDF rendering engine behind DCPP).
5. Runs two PDF builds using a shared publishing template.
6. Uploads both PDFs as downloadable workflow artifacts.

Reference: [Building, Validating, and Publishing Using GitHub Actions](https://blog.oxygenxml.com/topics/building_validating_and_publishing_using_github_actions.html) — Oxygen's own guidance on the underlying download-license-build mechanism that this workflow is based on.

## Prerequisite: Oxygen Publishing Engine license

This is the part most likely to trip up anyone reusing this workflow, so it's worth explaining in full.

**Your Oxygen XML Editor license does not work here.** Oxygen XML Editor (the desktop GUI app) and the Oxygen Publishing Engine (the headless CLI/CI tool) are licensed separately, even though they share the same underlying DCPP/Chemistry rendering code. An Editor trial or paid seat license will not activate Chemistry when run from the command line or in CI. Attempting to do so fails with:

```
com.oxygenxml.chemistry.c.d: You must have a valid Oxygen PDF Chemistry license key.
Please place your license key in a file named 'licensekey.txt' in the directory '...'.
If you don't have a license key or you need more details, please contact sales@oxygenxml.com.
```

If you hit this, you need a **separate Publishing Engine license**. Oxygen's sales team (sales@oxygenxml.com) can issue a trial license for evaluation or portfolio purposes; explain what you're building and ask whether a trial or limited-use key is available.

### License key format

The license Oxygen sends back is not a single string. It's a multi-line block that looks like this:

```
Registration_Name=your.email @ example . com
Company=your-company-or-self-employed
Category=Enterprise
Component=Publishing-Engine
Version=28
Number_of_Licenses=1
Date=MM-DD-YYYY
Trial=30
SGN=<long base64-style signature string>
```

The engine expects **the entire block**, not just the `SGN=` line. A license file containing only the SGN line will fail license validation even though the SGN line looks like "the actual key."

### Storing it as a GitHub secret

1. Go to the repo's **Settings → Secrets and variables → Actions → New repository secret**.
2. Name it `OXYGEN_LICENSE_KEY`.
3. Paste the **entire multi-line block** above (including `Registration_Name` through `SGN`) as the secret value. GitHub secrets support multi-line values without any special escaping.

The workflow writes this secret directly to `licensekey.txt` in the Publishing Engine's root directory:

```yaml
- name: License Oxygen Publishing Engine
  env:
    OXYGEN_LICENSE_KEY: ${{ secrets.OXYGEN_LICENSE_KEY }}
  run: |
    echo "$OXYGEN_LICENSE_KEY" > "$OPE_HOME/licensekey.txt"
```

If your trial expires (typically 30 days), the build will fail with the same license error above until the secret is updated with a new key.

## Key technical details worth knowing if something breaks

- For transtype, it uses `pdf-css-html5`. This is the DITA-OT transformation type for Oxygen's CSS-based PDF pipeline (DCPP), as opposed to the legacy XSL-FO-based `pdf2` transform.
- It references the `bin/dita` publishing launcher, not a top-level `dita` command, which I got by downloading the Publishing Engine zip and extracting it to `oxygen-publishing-engine/`.
- The publishing template flag is mandatory for correct output: `-Dpdf.publishing.template=<path-to-.opt-file>`. For my project transforms, I customized the CSS to create custom footers, suppress chapter numbering, and support a three-column specs layout. In my experience, a build can complete without errors but look completely unstyled. That's the signal that this flag has been omitted.
- There's no separate "activate" or "licensekey.sh" step. Searches for guidance often describe such a step, but for version 28.1, refer to the doc I reference above; in this distribution there isn't one. Writing the correctly formatted `licensekey.txt` file into the engine's root directory is the entire licensing mechanism.

## Running the workflow

```bash
gh workflow run publish.yml
gh run watch
```

Once it completes, download the artifacts:

```bash
gh run list --workflow=publish.yml
gh run download <run-id>
```

Or download them from the Actions tab in the GitHub UI under the specific run's Artifacts section.

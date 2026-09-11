# GitHub Actions consumer example

`boris-content-audit` is a **consumer-side** tool: a content repository that
uses Boris can run it to audit poetry coverage without any changes to Boris
CI. This document shows a workflow for a consumer repository.

## What the workflow does

1. Checks out the consumer content repository.
2. Acquires a pinned `boris-content-audit` binary by **building it from a
   pinned checkout of this repository** (the default path below), or by
   downloading a pinned, verified binary kit published from this repository.
3. Runs the poetry audit into a temporary output directory (never into the
   content tree), **capturing the exit status** so findings never lose the
   report.
4. Uploads the complete report as an Actions artifact **even when the audit
   fails** (`if: always()`).
5. Writes the Markdown executive summary to `$GITHUB_STEP_SUMMARY`
   **even when the audit fails** (`if: always()`).
6. Fails the job in a **final, always-run step** when the captured audit
   status requires failure (`--fail-on`).
7. On main or manual dispatch, uploads `site/` as a GitHub Pages artifact
   using the **same pinned tool acquisition**.
8. Never commits generated reports back into the consumer content repository.
9. Uses read-only repository permissions except for the Pages deployment job.

## Acquisition paths

**Path A (recommended): build from a pinned checkout of this repository.**

Pin the exact tag of `boris-content-audit`, check it out, build it, and copy
the binary to `PATH`. Pinning both the tool tag and the Zig toolchain version
keeps the consumer audit reproducible. No Boris checkout is needed: the tool
has no Boris build dependency, and the grammar it validates is pinned by tag
in [the README](README.md#boris-pins).

```yaml
- name: Install Zig (pinned)
  uses: mlugg/setup-zig@v2
  with:
    version: 0.16.0

- name: Check out pinned boris-content-audit
  uses: actions/checkout@v4
  with:
    repository: drawmeanelephant/boris-content-audit
    ref: v0.8.2              # pin the tool tag, not a moving branch
    path: content-audit

- name: Build boris-content-audit from source
  run: |
    zig build --build-file content-audit/build.zig
    install -m 0755 \
      content-audit/zig-out/bin/boris-content-audit \
      /usr/local/bin/boris-content-audit
```

**Path B: download a pinned, verified binary kit** (only when this repository
publishes one).

The tool ships as source today, so Path A is the supported acquisition. If a
release kit is published from this repository, it carries the binary plus its
content-addressed metadata; download the artifact for the exact pinned release
and verify it before use:

```yaml
- name: Download pinned binary kit
  uses: actions/download-artifact@v4
  with:
    name: boris-content-audit-v0.8.2-linux-x86_64
    path: /tmp/boris-kit

- name: Verify and install binary kit
  run: |
    # The kit contains the manifest and checksums published with the release.
    shasum -a 256 -c /tmp/boris-kit/SHA256SUMS
    install -m 0755 /tmp/boris-kit/boris-content-audit /usr/local/bin/boris-content-audit
```

Pick exactly one path and delete the other from the workflow you copy. The
**echo-only placeholder is never an operative step** — the audit must run a
real binary.

## Workflow file

`.github/workflows/content-audit.yml` in the consumer repository:

```yaml
name: content-audit

on:
  pull_request:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read   # read-only everywhere; Pages job overrides below

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - name: Check out content repository
        uses: actions/checkout@v4

      # ---- Pinned acquisition (Path A; swap for Path B if you distribute a kit) ----
      - name: Install Zig (pinned)
        uses: mlugg/setup-zig@v2
        with:
          version: 0.16.0

      - name: Check out pinned boris-content-audit
        uses: actions/checkout@v4
        with:
          repository: drawmeanelephant/boris-content-audit
          ref: v0.8.2
          path: content-audit

      - name: Build boris-content-audit from source
        run: |
          zig build --build-file content-audit/build.zig
          install -m 0755 \
            content-audit/zig-out/bin/boris-content-audit \
            /usr/local/bin/boris-content-audit

      # ---- Audit with exit capture ----
      - name: Run poetry audit
        id: audit
        continue-on-error: true   # findings must not discard the report
        run: |
          set +e
          boris-content-audit \
            --mode=poetry \
            --root=. \
            --content-root=content \
            --policy=content-audit-policy.json \
            --out=/tmp/poetry-audit \
            --format=all \
            --fail-on=structural
          audit_status=$?
          echo "audit_status=$audit_status" >> "$GITHUB_OUTPUT"
          echo "## Audit exit status" >> "$GITHUB_STEP_SUMMARY"
          echo "The content audit exited with **${audit_status}** (0 = clean, 1 = selected findings, 3 = output/ownership error, 4 = contract error)." >> "$GITHUB_STEP_SUMMARY"
          exit 0

      # ---- Report is uploaded even when the audit fails ----
      - name: Upload report artifact
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: poetry-audit-report
          path: /tmp/poetry-audit
          if-no-files-found: error

      # ---- Summary is written even when the audit fails ----
      - name: Write executive summary to step summary
        if: always()
        run: |
          {
            echo "## Poetry content audit"
            echo ""
            sed -n '1,40p' /tmp/poetry-audit/REPORT.md
          } >> "$GITHUB_STEP_SUMMARY"

      # ---- Fail the job last, only when the captured status requires it ----
      - name: Enforce audit status
        if: always()
        run: |
          status="${{ steps.audit.outputs.audit_status }}"
          if [ -z "${status}" ]; then
            echo "no audit status captured; failing" >&2
            exit 1
          fi
          if [ "${status}" = "0" ]; then
            echo "audit passed"
          elif [ "${status}" = "1" ]; then
            echo "audit completed with selected findings; report uploaded" >&2
            exit 1
          else
            echo "audit failed with status ${status}; report uploaded" >&2
            exit 1
          fi

  publish-pages:
    # Main branch or manual dispatch only: publish the static site.
    if: github.event_name == 'push' && github.ref == 'refs/heads/main' || github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Check out content repository
        uses: actions/checkout@v4

      # Same pinned acquisition as the audit job.
      - name: Install Zig (pinned)
        uses: mlugg/setup-zig@v2
        with:
          version: 0.16.0

      - name: Check out pinned boris-content-audit
        uses: actions/checkout@v4
        with:
          repository: drawmeanelephant/boris-content-audit
          ref: v0.8.2
          path: content-audit

      - name: Build boris-content-audit from source
        run: |
          zig build --build-file content-audit/build.zig
          install -m 0755 \
            content-audit/zig-out/bin/boris-content-audit \
            /usr/local/bin/boris-content-audit

      - name: Run poetry audit
        run: |
          boris-content-audit \
            --mode=poetry \
            --root=. \
            --content-root=content \
            --policy=content-audit-policy.json \
            --out=/tmp/poetry-audit

      - name: Setup Pages
        uses: actions/configure-pages@v5

      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: /tmp/poetry-audit/site

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

## Notes

- The audit output directory is always a temporary location
  (`/tmp/poetry-audit`); generated reports are **never** committed back into
  the consumer content repository.
- **Exit capture never discards the report.** The `Run poetry audit` step uses
  `continue-on-error: true` and records the status to `GITHUB_OUTPUT`; the
  artifact upload and step summary run with `if: always()`, and a final
  always-run step fails the job only when the captured status requires it.
- `--fail-on` defaults to `structural`, so a run with only missing or
  placeholder poetry still succeeds — use `--fail-on=policy` if the
  repository wants policy-level findings to fail CI.
- The report site is static and self-contained; it can be served from
  `file://` or published to Pages without any build step.
- The tool has no Boris build dependency. Only the grammar it validates is
  pinned, by Boris release tag, in [the README](README.md#boris-pins).
- Boris CI does **not** clone private or external dogfood repositories; this
  workflow lives in the consumer repository, not in Boris.

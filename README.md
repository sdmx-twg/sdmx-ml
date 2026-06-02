# Overview

This repository is used for maintaining the SDMX-ML messages specifications.

This includes:
* Normative documentation and XSD schemas for the SDMX-ML structure, data, metadata and registry interface messages
* Samples for core structures including Data Structure Definition, Code List and Concept Scheme
* Samples illustrating VTL and geospatial structures
* Data message samples

To test the samples, copy the ‘samples’ and ‘schemas’ folders into the same local directory.

The schemas are published to https://xml.sdmx.org

## Repository Structure

-   `docs/` — Markdown content pages for the SDMX-ML specification.
-   `mkdocs.yml` — MkDocs configuration integrating this component into the
    `sdmx-docs` site.

## Version Branches

Each minor release of this component is maintained on a dedicated documentation
branch following the naming convention `docs_vX.Y` (e.g., `docs_v2.1`,
`docs_v3.0`).

These branches exist solely to support the documentation website and are not
used for regular development. Changes to the specification continue to go
through the normal development and release process (via `develop`). Older
documentation branches may additionally require file reorganization and
formatting adaptations for MkDocs.

The branch tracked by the
[`sdmx-docs`](https://github.com/sdmx-twg/sdmx-docs) parent repository is
declared in `.gitmodules` at the root of that repo. Switching the tracked
branch in the parent repository is how a new version of this component is
published on the documentation site.

## Formatting Conventions

For Markdown and MkDocs formatting conventions that apply to content in `docs/`,
see the [`sdmx-docs` README](https://github.com/sdmx-twg/sdmx-docs#readme).


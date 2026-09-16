# Disaster recovery runbook

This site is stateless. Its authoritative source is the public Git repository; GitHub Pages is a replaceable generated deployment.

## Recovery objectives

- RPO: one accepted Git commit.
- RTO: 30 minutes after GitHub and Pages are available.

## Prerequisites

- Git 2.x.
- A GitHub account with write access to the repository.
- No runtime secrets, database, DNS record, or uploaded user content is required.

## Restore on a clean machine

1. Clone the repository.
2. Confirm the expected revision with `git rev-parse HEAD`.
3. Verify that `index.html`, `analysis.md`, `runtime-code-agent.md`, `diagrams/`, and `assets/previews/` exist.
4. In repository settings, configure GitHub Pages to deploy from the `main` branch and repository root.
5. Wait for the Pages deployment, then open the published URL and verify the landing page, report, Runtime article, and five diagrams.

## Validation

- All reader-facing links return HTTP 200.
- The landing page shows all three preview cards.
- The report identifies the analyzed upstream commit.
- The Runtime article renders at `/runtime-code-agent/`, with two previews and links to both interactive diagrams. All URLs must include the project-site base path.
- Formal publication artifacts contain no credentials or machine-local paths.

## Diagram recovery

The checked standalone HTML diagrams, PNG previews, and their editable JSON specifications are versioned together. Restoring the website requires no diagram-generation tool: Pages publishes these assets unchanged. To revise diagrams, use Archify 2.17 with the corresponding JSON input, then repeat its validation and visual checks before replacing HTML and previews. The analyzed upstream revision is pinned in the article; do not silently substitute a newer checkout. Local visual-check receipts are intentionally excluded because they may contain machine-specific paths.

## Rollback

Revert the faulty commit on `main` and push. GitHub Pages will rebuild the previously known-good content.

## Secrets, DNS, and state

There are no application secrets or stateful services. The default GitHub Pages domain is used, so no custom DNS or TLS recovery is needed.

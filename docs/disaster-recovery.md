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
3. Verify that `index.html`, `analysis.md`, `diagrams/`, and `assets/previews/` exist.
4. In repository settings, configure GitHub Pages to deploy from the `main` branch and repository root.
5. Wait for the Pages deployment, then open the published URL and verify the landing page, report, and three diagrams.

## Validation

- All reader-facing links return HTTP 200.
- The landing page shows all three preview cards.
- The report identifies the analyzed upstream commit.
- Formal publication artifacts contain no credentials or machine-local paths.

## Rollback

Revert the faulty commit on `main` and push. GitHub Pages will rebuild the previously known-good content.

## Secrets, DNS, and state

There are no application secrets or stateful services. The default GitHub Pages domain is used, so no custom DNS or TLS recovery is needed.


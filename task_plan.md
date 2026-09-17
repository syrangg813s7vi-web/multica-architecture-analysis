# Publication plan

## Runtime component diagram publication (TIN-734)
Scope: publish the validated Archify runtime component and communication diagram to the existing static Pages site without replacing the overview diagram. The versioned JSON is the editable source; HTML and PNG are derived artifacts. Dependencies: checked source diagram and the current Pages repository. Acceptance: homepage entry, six-diagram count, privacy and link checks, successful Pages build and live diagram/preview/JSON access. Rollback: revert this publication commit. No service or data migration.
- [x] Select the existing Pages repository and record the deployment boundary.
- [x] Add diagram, preview, homepage navigation, and recovery documentation.
- [x] Validate locally, publish, and verify the live deployment.

## Runtime article publication
Scope: Chinese article on Runtime, remote daemon and Code Agent integration; explicitly omit Docker. Reuse checked diagrams, add homepage/report links, publish to existing Pages repository. No service changes. Markdown owns narrative; checked JSON owns diagrams; static assets are derived. Acceptance: pinned source references, no credentials/local paths, internal links and live article/images/diagrams verified. Rollback: revert publication commit. Follow existing recovery runbook.
- [x] Author and audit article/assets.
- [x] Publish and verify Pages deployment.

## Goal

Publish the existing Multica source analysis in a new public GitHub repository and deploy it as a directly browsable static website.

## Phases

- [x] 1. Inspect source artifacts, GitHub authentication, and publication constraints.
- [x] 2. Audit and sanitize public content.
- [x] 3. Build a reproducible static documentation site.
- [x] 4. Validate links, rendering, and generated artifacts locally.
- [x] 5. Create the public GitHub repository and push the initial revision.
- [x] 6. Enable GitHub Pages and verify the public URL.
- [x] 7. Record recovery/rebuild instructions and final evidence.

## Decisions

- Publish analysis only; do not republish the full upstream source tree.
- Keep the analyzed upstream repository and commit SHA visible for reproducibility.
- Use a static site with no secrets or runtime database.
- Use GitHub Pages for low-friction public browsing.
- Keep the publication repository independent from the upstream source repository.

## Errors encountered

| Error | Attempt | Resolution |
|---|---:|---|
| Initial skill-read command used an invalid working directory | 1 | Re-ran from the actual workspace root. |
| Unneeded workspace-dependency lookup stalled | 1 | Terminated it; no dependency lookup is required for a static site. |
| Plain local HTTP server displayed Jekyll front matter in `index.html` | 1 | Removed front matter from the static landing page and kept Jekyll only for Markdown rendering. |
| `curl` is not installed in the workspace runtime | 1 | Used real-browser navigation and snapshots for the landing page and all three diagrams. |

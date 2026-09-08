# Publication plan

## Goal

Publish the existing Multica source analysis in a new public GitHub repository and deploy it as a directly browsable static website.

## Phases

- [x] 1. Inspect source artifacts, GitHub authentication, and publication constraints.
- [x] 2. Audit and sanitize public content.
- [x] 3. Build a reproducible static documentation site.
- [x] 4. Validate links, rendering, and generated artifacts locally.
- [ ] 5. Create the public GitHub repository and push the initial revision.
- [ ] 6. Enable GitHub Pages and verify the public URL.
- [ ] 7. Record recovery/rebuild instructions and final evidence.

## Decisions

- Publish analysis only; do not republish the full upstream source tree.
- Keep the analyzed upstream repository and commit SHA visible for reproducibility.
- Use a static site with no secrets or runtime database.
- Use GitHub Pages for low-friction public browsing.

## Errors encountered

| Error | Attempt | Resolution |
|---|---:|---|
| Initial skill-read command used an invalid working directory | 1 | Re-ran from the actual workspace root. |
| Unneeded workspace-dependency lookup stalled | 1 | Terminated it; no dependency lookup is required for a static site. |
| Plain local HTTP server displayed Jekyll front matter in `index.html` | 1 | Removed front matter from the static landing page and kept Jekyll only for Markdown rendering. |
| `curl` is not installed in the workspace runtime | 1 | Used real-browser navigation and snapshots for the landing page and all three diagrams. |

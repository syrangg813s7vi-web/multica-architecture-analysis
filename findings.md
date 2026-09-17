# Findings

## Runtime component diagram

- GitHub Pages built publication commit `983473b413d46d18142c54a20c0bc6aa6f7bbbd9`. The live homepage, diagram, preview, and JSON returned HTTP 200; live HTML and JSON hashes match the validated local artifacts.
- Browser navigation from the live homepage opened the new diagram. The only observed console error was a default `/favicon.ico` request at the domain root, unrelated to the diagram; the project-site favicon file remains available under its project path.
- The existing site already has a broader system overview diagram, so the new runtime view uses a distinct URL and leaves earlier diagrams unchanged.
- The Archify HTML and JSON hashes match the validated source artifacts; the PNG preview is versioned with them.
- The static homepage link reaches the interactive diagram in Chromium at mobile width. The diagram renders at desktop width with no browser console errors; local HTML and PNG endpoints return HTTP 200.
- The new public assets contain no machine-local path, localhost URL, or credential marker in the targeted privacy scan.

## Runtime article

- Live publication succeeded at commit `8959deaffa541790ec4da023ddcac7d32f5052d6`; all five article/asset endpoints returned HTTP 200. Jekyll expanded the project-site base path correctly.

- The article distinguishes platform-to-daemon control from Backend-to-CLI protocol adaptation, with immutable source references at the existing analysis baseline.
- Reused two previously validated Archify diagrams and their editable JSON specifications; excluded machine-specific visual-check receipts.
- New public article and diagrams passed the local-path, credential-query, and excluded-topic scan. Recovery remains a stateless Pages rebuild from versioned inputs.

## Source artifacts

- Publish the 273-line repository report, three standalone Archify HTML diagrams, three source JSON specs, and three representative preview images.
- Exclude visual-check harnesses and visual-check JSON because they contain machine-local paths and are not reader-facing artifacts.
- Analysis baseline is upstream commit `c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd`.

## Publication audit

- GitHub CLI is authenticated with permission to create a public repository and push through SSH.
- The report and formal diagrams contain no real email address, access token, secret, private hostname, or machine-local absolute path.
- CodeHub appears only as a generic compatibility example; no company endpoint or credential is present.
- Existing relative source references must be rewritten to immutable upstream commit links before publication.

## Deployment

- The static landing page and all three interactive diagrams load successfully in a real Chromium browser.
- JSON source specs parse successfully and every required publication artifact exists.
- The landing page has responsive CSS and a direct link to the immutable upstream source baseline.
- GitHub Pages will process `analysis.md` through Jekyll into `/analysis/`; the landing page and diagrams are standalone HTML.
- The public repository was created successfully and the first Pages build completed from `main` at the repository root.
- Public Chromium verification confirmed the landing page, rendered Markdown report, and all three diagrams.
- The verified Pages build for publication content completed successfully at commit `e9984bed352ab2f179359326a77764fd7715b70e`.
- The repository is the authoritative source; the hosted site is stateless and can be rebuilt from `main` without secrets.

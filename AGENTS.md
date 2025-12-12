# Repository Guidelines

## Project Structure & Module Organization
- Root holds the static site entry point `index.html`; content is currently lightweight and in Japanese.  
- `README.md` documents the Markdown learnings that accompany the portfolio.  
- `.github/workflows/static.yml` deploys the entire repo to GitHub Pages on pushes to `main`. Keep additional assets (images, CSS, JS) alongside `index.html` or in subfolders like `assets/` for clarity.

## Build, Test, and Development Commands
- No build step is required; the site ships as plain HTML.  
- Preview locally: `python3 -m http.server 8000` from the repo root, then open `http://localhost:8000/index.html`.  
- Validate HTML quickly with `npx htmlhint index.html` (optional; install once via `npm install -g htmlhint` if you prefer).

## Coding Style & Naming Conventions
- HTML: two-space indentation, lowercase tags/attributes, and UTF-8 encoding.  
- Keep copy edits in Japanese to match the current tone; add brief English comments only when clarifying structure.  
- Use semantic elements when expanding (e.g., `header`, `main`, `section`, `footer`) and prefer external CSS files under `assets/` over inline styles as the site grows.

## Testing Guidelines
- There are no automated tests today; rely on manual verification in at least one modern desktop and one mobile browser.  
- Check for broken links, visible layout regressions, and encoding issues. Capture before/after screenshots when altering layout or typography.

## Commit & Pull Request Guidelines
- Follow the existing short, present-tense style (Japanese or concise English is fine), e.g., `index.htmlを作った！` or `Add hero section`.  
- One change per commit when possible; include context if touching multiple files.  
- Pull requests should include: a short summary, screenshots for visual changes, steps to reproduce/verify, and links to related issues or tasks. Keep diffs small and focused.

## Deployment Notes
- GitHub Actions deploys on every push to `main` via `.github/workflows/static.yml`; no manual steps are needed.  
- Verify the Pages URL from the workflow run output after merges and roll back by reverting the offending commit if a deploy introduces a regression.

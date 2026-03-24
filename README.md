# Ledewire — Development Overview

Central developer documentation hub for Ledewire, published via GitHub Pages.

## Viewing the Site

Once GitHub Pages is enabled, the site will be at:

**`https://ledewire.github.io/overview/`**

> The URL path (`/overview/`) is set by the repository name — the repo must be named `overview` inside the `ledewire` GitHub organization.

### Enable GitHub Pages

1. Go to **Settings → Pages** in this repository
2. Under **Source**, select **Deploy from a branch**
3. Choose `main` branch, `/ (root)` folder
4. Click **Save**

The site will be live within a minute or two.

## Local Preview

No build step required — it’s plain HTML. Use any static file server:

```bash
# Python (built-in on macOS/Linux)
python3 -m http.server 8080

# Node.js via npx
npx serve .

# VS Code: right-click index.html → "Open with Live Server"
```

Then open `http://localhost:8080`.

## Site Structure

```
├── index.html                  # Landing / hub page
├── roadmap.html                # Roadmap & strategic direction
├── assets/
│   └── css/
│       └── style.css           # Shared stylesheet
├── docs/
│   ├── getting-started.html    # Onboarding & local setup
│   ├── architecture.html       # System architecture overview
│   ├── contributing.html       # PR process & coding standards
│   ├── decisions.html          # Architecture Decision Records index
│   ├── services.html           # Per-service documentation
│   └── adr/                    # Individual ADR files (add as decisions are made)
├── .nojekyll                   # Tells GitHub Pages to skip Jekyll processing
└── README.md
```

## Adding Content

- **Update placeholder text**: Search for `<!-- TODO` comments throughout the HTML files.
- **Add a new ADR**: Create a file in `docs/adr/NNN-short-title.md` and add a card to `docs/decisions.html`.
- **Add a new page**: Copy the header/sidebar/footer pattern from any existing `docs/*.html` page.
- **Update the roadmap**: Edit the timeline items in `roadmap.html`.

## Tech

Pure HTML + CSS — no build tools, no dependencies, no framework. Intentionally simple so that any contributor can edit docs without setup.

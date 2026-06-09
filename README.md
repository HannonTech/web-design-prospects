# Oak Forest Prospects

Prospect research, leads tracking, outreach templates, and demo sites for the Oak Forest web design business.

## Structure
```
prospects/
├── leads.csv              ← master prospect list (update this as you work)
├── outreach/              ← email and phone templates by business type
├── research/              ← chamber notes, competitor research
└── [business-slug]/       ← demo site for each prospect
    ├── index.html
    └── assets/
```

## Workflow
1. Add prospect to `leads.csv` with status `researching`
2. Build demo in `[business-slug]/index.html`
3. Push to GitHub — demo goes live at `https://[username].github.io/oak-forest-prospects/[business-slug]/`
4. Update status to `demo-built`, paste demo URL in notes
5. Send outreach email from `outreach/` template
6. Update status through: `outreach-sent` → `follow-up` → `negotiating` → `client` or `lost`

## When a Prospect Becomes a Client
1. Create a new GitHub repo: `[business-name]-website`
2. Copy their demo folder as the starting point
3. Build out the full site
4. Update `leads.csv` status to `client`
5. Remove their demo folder from this repo (or leave it — your call)

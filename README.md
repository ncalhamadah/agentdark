# AgentDark

AgentDark is an **open‑source**, **production‑grade** credential exposure monitor focused on email/password leaks for your organisation.  It is designed for internal security teams who need an easy way to discover when employee accounts appear in public or dark‑web breach dumps.  The project is split into two parts:

* A **frontend** built with [React](https://reactjs.org/) and [Vite](https://vitejs.dev/) that renders a public, read‑only dashboard on **GitHub Pages**.  This dashboard lists every email in your watchlist along with any associated leaks, severity scores and remediation suggestions.
* A **collector** built with [Netlify Functions](https://docs.netlify.com/functions/overview/) that runs on a scheduled cadence (daily by default) via Netlify’s scheduled functions.  The collector pulls data from third‑party services like Have I Been Pwned (HIBP) and lawful open breach indexes, deduplicates and risk‑scores the findings, and publishes sanitised JSON that the frontend consumes.  A simple admin API is also provided for maintaining the list of watched email addresses.

## Features

* **Watchlist management** – add or remove email addresses via a protected API endpoint or by editing a CSV file.  Only addresses belonging to verified domains can be added.
* **Scheduled scanning** – a daily Netlify scheduled function queries breach APIs and TOR‑only data sources (when enabled) for each address in the watchlist.
* **Risk scoring** – each finding is assigned a severity based on exposure type (plaintext vs hash), recency and duplication across sources.
* **Redaction options** – by default no password characters are displayed in the dashboard; admins can temporarily enable a “last‑character redacted” mode for legal review.
* **Public dashboard** – results are published as static JSON and rendered via a responsive React dashboard with filters, search and CSV/PDF export.
* **90‑day retention** – findings older than 90 days are automatically purged by the collector.
* **Domain verification** – additional domains may be monitored after a DNS TXT challenge proves ownership.

## Repository structure

```
agentdark/
├── README.md             — this file
├── LICENSE               — MIT license
├── netlify.toml          — Netlify configuration (build, functions and schedule)
├── package.json          — root package with workspaces for frontend and collector
├── .gitignore            — ignores dependencies, builds and secrets
├── .github/workflows     — GitHub Actions CI configuration
│   └── node.yml
├── frontend              — React/Vite dashboard served from GitHub Pages
│   ├── package.json
│   ├── index.html
│   ├── vite.config.js
│   └── src
│       ├── main.jsx
│       ├── App.jsx
│       └── api.js
└── collector             — Netlify Functions and supporting libraries
    ├── package.json
    ├── functions
    │   ├── scan.js            — scheduled function to perform daily scans
    │   ├── public-findings.js — HTTP function serving public findings
    │   └── admin-emails.js    — protected API for managing the watchlist
    ├── connectors
    │   ├── hibp.js            — HIBP connector (real or mocked)
    │   └── mock.js            — fallback/mock connector
    └── lib
        ├── db.js              — simple file‑based store for emails and findings
        └── risk.js            — risk scoring helper
```

## Running locally

To run both the collector and the frontend locally you need Node.js >=18 and npm installed.  Each package manages its own dependencies:

```sh
# install all dependencies
npm install

# run the Netlify functions locally on port 8888
npm run dev:collector

# run the frontend with hot reloading on port 5173
npm run dev:frontend
```

For local development the collector will use the **mock connector** by default when no `HIBP_API_KEY` is set.  This produces synthetic findings so the dashboard has data to display.  When you obtain a real HIBP key, set it in a `.env` file or your Netlify environment to enable live data.

## Deployment overview

1. **Create a public GitHub repository** named **agentdark** and push this code.  Initialise the repository with the MIT licence.
2. **Connect the repository to Netlify** and enable [Netlify Functions](https://docs.netlify.com/functions/overview/).  The `netlify.toml` file instructs Netlify to build the frontend (`npm run build --prefix frontend`) and deploy the output to GitHub Pages.  It also configures the functions directory and schedules the `scan` function to run every day at 22:00 UTC (02:00 Asia/Dubai).
3. **Add environment variables** in the Netlify dashboard:
   - `ADMIN_TOKEN` – a long random string used to authorise admin API requests.
   - `HIBP_API_KEY` – optional; required for live HIBP scanning.
   - `ALLOWED_DOMAINS` – comma‑separated list of domains allowed in the watchlist (e.g. `eden.ae`).
4. **Configure GitHub Pages**: in your repository settings enable GitHub Pages and set the source to the `gh-pages` branch created by the build.  The `vite.config.js` file sets the base path correctly.
5. **Verify deployment**: once Netlify builds successfully, open your GitHub Pages URL (`https://<username>.github.io/agentdark/`) to view the dashboard.  You should see sample findings if no HIBP key is configured.

For a detailed runbook and day‑0 checklist, consult the comments in `netlify.toml` and the scripts within the `collector/functions` folder.

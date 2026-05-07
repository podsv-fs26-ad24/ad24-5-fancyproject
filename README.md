# ThurNaturGau Marktübersicht

**Documentation website:** https://podsv-fs26-ad24.github.io/ad24-5-fancyproject/

Interactive choropleth dashboard for **ThurNaturGau Handel** (fictional agricultural trader, Kanton Thurgau) showing livestock distribution, GVE intensity, and feed-purchase potential per municipality. Built as a project deliverable for the ZHAW IUNR PODSV course (Programming and Data Visualization for Sustainability) by Christian Haag.

The project follows the visualization product development process taught at ZHAW: **Project Understanding → Data Acquisition & Exploration → Visual Encoding & Design → Evaluation → Deployment**. Every phase has a corresponding documentation page (rendered to a Quarto website) and, where applicable, a code folder.

The end deliverable is a single self-contained HTML file (`deployment/output/thurnatur_dashboard.html`) that the field-sales team can open in any browser, online or offline.

## Project Organisation

| Phase | Code folder | Documentation section | `docs`-File |
|:-------|:---|:---|:---|
| Project Understanding | – | Project Charta | `project_charta.qmd` |
| Data Acquisition and Exploration | `eda` | Data Report | `data_report.qmd` |
| Visual Encoding and Design | `viz_design` | Visual Encoding and Design | `viz_encoding_design.qmd` |
| Evaluation | `evaluation` | Evaluation | `evaluation.qmd` |
| Deployment | `deployment` | Deployment | `deployment.qmd` |

The `data-raw/` folder holds the raw inputs (swissBOUNDARIES3D shapefile, ThurNaturGau logo). The `data/` folder is where the data report writes processed snapshots (`tg_gemeinden.geojson`, `tierbestaende_<year>.json`) that the deployment step then bakes into the dashboard. Both folders' contents are gitignored except for the README markers; only the source files in `data-raw/` belong in version control. The `deployment/template/` folder contains the HTML scaffold with placeholders that the build step substitutes.

## Python Environment Setup and Management with uv

Make sure you have `uv` installed: <https://docs.astral.sh/uv/getting-started/installation/>

After cloning the repository, create the Python environment with all dependencies based on the `.python-version`, `pyproject.toml` and `uv.lock` files by running:

```bash
uv sync
```

To add new dependencies, use `uv add <package>`; to remove, `uv remove <package>`. Commit changes to `pyproject.toml` and `uv.lock` into version control. Run `uv sync` after pulling changes to update the local environment.

Whenever the Python environment is used, prefix every command with `uv run`, e.g. `uv run python script.py`. Alternatively, activate the project environment in the current shell:

```bash
source .venv/bin/activate
```

## Runtime Configuration with Environment Variables

Environment variables are specified in a `.env` file, which is **never** committed (it may contain secrets). The repo only contains `.env.template` to demonstrate the format — copy or rename it to `.env` for local use. The content is read by the `python-dotenv` dependency.

For this project, the relevant variables are:

```bash
TG_API_BASE=https://data.tg.ch/api/explore/v2.1/catalog/datasets
TG_DATA_YEAR=          # leave empty for auto-detect (newest available)
GEO_SIMPLIFY_TOLERANCE=0.0002
```

## Quarto Setup and Usage

### Setup Quarto

1. [Install Quarto](https://quarto.org/docs/get-started/) (v1.4 or newer)
2. Optional: [Quarto extension for VS Code](https://marketplace.visualstudio.com/items?itemName=quarto.quarto)
3. If working with svg files and pdf output you will need to install rsvg-convert:
    * macOS: `brew install librsvg`
    * Windows (chocolatey): `choco install rsvg-convert`

Source `*.qmd` files and the Quarto config (`_quarto.yml`) are in the `docs/` folder.

### Working on the Documentation

1. Make changes to the `.qmd` source files in `docs/`
2. Activate the Python environment (see above)
3. Preview locally: `quarto preview` from the `docs` folder
4. Build the documentation website: `uv run quarto render` from the `docs` folder. This renders to `docs/build/`
5. Open `docs/build/index.html` in a browser to verify

### Building the Dashboard

The data report (`data_report.qmd`) and deployment page (`deployment.qmd`) are *executable* — they actually run the data pipeline and write outputs:

* `data_report.qmd` → loads the shapefile, filters Thurgau, reprojects, simplifies, fetches API data, writes `data/tg_gemeinden.geojson` and `data/tierbestaende_<year>.json`
* `deployment.qmd` → reads the snapshots, embeds them plus the logo into `deployment/template/dashboard_template.html`, writes `deployment/output/thurnatur_dashboard.html`

So the full pipeline is just:

```bash
cd docs && uv run quarto render
```

The first render takes about a minute (loading the 44 MB shapefile and hitting the API). Subsequent renders are cached via `_freeze` (see below) and are near-instant.

### Deployment of the Documentation to GitHub Pages

The documentation website is deployed to GitHub Pages via a GitHub Actions workflow (`.github/workflows/publish.yml`). Every push to `main` triggers the workflow, which renders the Quarto project and deploys the result.

The setting `execute: freeze: auto` in `_quarto.yml` ensures Python computations only execute locally. Results are cached in `docs/_freeze` and committed to the repository, so the GitHub Actions runner doesn't need Python — it uses the pre-computed results.

#### Initial Setup (once)

1. In the GitHub repository settings, go to **Settings > Pages** and set the source to **GitHub Actions**
2. Render locally so that `_freeze` contains cached computation results:
   ```bash
   cd docs && uv run quarto render
   ```
3. Commit and push the resulting `_freeze/` directory along with the workflow file `.github/workflows/publish.yml`

#### Publishing Updates

1. Build locally: `cd docs && uv run quarto render`. This updates `docs/build/` (gitignored) and `docs/_freeze/` (checked in)
2. Open `docs/build/index.html` to verify
3. Commit and push all updated files (including `docs/_freeze`) to `main`. The workflow will publish the site automatically.

## Replacing Inputs

* **Newer Gemeinde boundaries**: drop a fresh swissBOUNDARIES3D download into `data-raw/swissboundaries/` (overwrite the existing files; the build expects the same filenames). Get it from <https://www.swisstopo.admin.ch/de/geodata/landscape/boundaries3d.html>.
* **New logo**: replace `data-raw/logo.png`. Aim for under 100 KB; it gets base64-embedded.
* **New data year**: leave `TG_DATA_YEAR` empty in `.env` — the build auto-detects the newest year available on `data.tg.ch`.

After replacing inputs, re-render: `cd docs && uv run quarto render`.

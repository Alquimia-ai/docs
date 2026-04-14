# Alquimia documentation hub (`docs`)

Documentation hub for the Alquimia platform. Root `docs.json` configures branding, the product switcher, and hub navigation. Deep product manuals live under `products/<product>/` and are populated by each product’s sync workflow.

## Config files (what matters)

| File | Role |
| --- | --- |
| **`docs.json`** (repo root) | **Only** file Mintlify uses for site-wide config, the product switcher, and the **Overview** product’s inline navigation. |
| **`products/studio/navigation.json`** | Canonical **Studio** sidebar for the hub: flat `product`, `description`, and `groups`. Pulled into root `docs.json` with `{ "$ref": "products/studio/navigation.json", "icon": "…" }`. |
| **`products/insight-hub/navigation.json`** | Same pattern for **InsightHub** (and future products use **`products/<id>/navigation.json`**). |
| **`products/studio/docs.json`** | Shipped by Studio for **standalone** Studio docs; **ignored** here via **`.mintignore`** so it never competes with the root config. |

Each `navigation.json` under `products/<id>/` must match what Mintlify expects for a single **`navigation.products[]`** entry (at minimum **`product`**, **`description`**, and **`groups`**). Icons and other allowed siblings live beside **`$ref`** in root **`docs.json`**.

## Language policy

- **Hub-authored** MDX at the repo root and under `platform/` is written in **English** only.
- Corporate and marketing content may stay bilingual on [alquimia.ai](https://alquimia.ai).

## Ownership

| Path | Owner |
| --- | --- |
| `docs.json` (root), `index.mdx`, `platform/**` | Platform / docs maintainers |
| `products/studio/**` | **Studio** automation only (sync PRs), including **`navigation.json`**, MDX, and assets. |
| `products/insight-hub/**` | **InsightHub** automation only (sync PRs), including **`navigation.json`**, MDX, and assets. |
| Future `products/<id>/**` | Matching product repo automation |

Do not commit manual changes inside `products/<id>/` except through that product’s sync pull requests.

## Product navigation without editing root `docs.json`

Mintlify resolves **`$ref`** to a JSON file and **merges** any **sibling** keys from root `docs.json` on top of the referenced object (for example **`icon`**).

- **Studio** in `navigation.products` uses **`$ref`: `products/studio/navigation.json`** plus an **`icon`** (see root **`docs.json`**).
- **InsightHub** uses **`$ref`: `products/insight-hub/navigation.json`** the same way.

When a product’s navigation or pages change, update **`products/<id>/navigation.json`** (and MDX) in that product’s sync. Root **`docs.json`** does **not** need to change for sidebar updates, as long as the **`$ref`** path stays the same.

### Optional automation in this repo

You can add a GitHub Action on pull requests that runs **`mint validate`** and **`mint broken-links`** (or JSON validation on `navigation.json`) for faster review. That does **not** require a second generated navigation file.

## Reviewing a product sync PR (Studio, InsightHub, …)

1. Read the PR description and confirm it matches the expected product workflow.
2. Inspect the diff under the product directory: MDX, assets, **`navigation.json`**, and **`docs.json`** if present (hub builds ignore `products/studio/docs.json` when it is listed in **`.mintignore`**).
3. Confirm **`navigation.json`** stays valid for Mintlify (paths under **`products/<id>/`** match real MDX files, structure is consistent).
4. Run **`mint validate`** (and optionally **`mint broken-links`**) before merging.

## Mintlify

Connect this GitHub repository in the Mintlify dashboard; default branch `main`; content root is the repository root where `docs.json` lives.

## Branding

Theme, colors, favicon, and logos follow the shared tokens (`theme: mint`, primary `#7C3AED`, and logo paths under `/logo/`).

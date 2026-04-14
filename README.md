# Alquimia documentation hub (`docs`)

Documentation hub for the Alquimia platform. Root `docs.json` configures branding, the product switcher, and hub navigation. Deep product manuals live under `products/<product>/` and are populated by each product’s sync workflow.

## Config files (what matters)

| File | Role |
| --- | --- |
| **`docs.json`** (repo root) | **Only** file Mintlify uses for site config, navigation, and the product switcher for this hub. |
| **`products/studio/hub-mintlify-product.json`** | Studio sidebar fragment; `$ref`’d from root `docs.json`; generated from `navigation.json` in Studio CI. |
| **`products/studio/navigation.json`** | Studio’s canonical nav export for tooling and the `jq` step above—not read directly by Mintlify for the hub sidebar. |
| **`products/studio/docs.json`** | Shipped by Studio for **standalone** Studio docs; **ignored** here via **`.mintignore`** so it never competes with the root config. |

## Language policy

- **Hub-authored** MDX at the repo root and under `platform/` is written in **English** only.
- Corporate and marketing content may stay bilingual on [alquimia.ai](https://alquimia.ai).

## Ownership

| Path | Owner |
| --- | --- |
| `docs.json` (root), `index.mdx`, `platform/**` | Platform / docs maintainers |
| `products/studio/**` | **Studio** automation only (sync PRs), including **`hub-mintlify-product.json`** (Mintlify nav fragment for the central hub). |
| Future `products/<id>/**` | Matching product repo automation |

Do not commit manual changes inside `products/studio/` except through Studio’s sync pull requests.

## Studio navigation without editing root `docs.json`

Root `docs.json` pulls the **Alquimia Studio** sidebar from a file under the synced tree using Mintlify’s **`$ref` merge**:

- In `docs.json`, the Studio entry is: `{ "$ref": "products/studio/hub-mintlify-product.json", "icon": "layout-dashboard" }`.
- Mintlify loads that JSON and merges sibling keys (`icon`) on top of the referenced object.

You **cannot** `$ref` `navigation.json` directly: its shape is `{ "product": { "product", "description" }, "groups" }`, while one `navigation.products[]` item must be flat: `{ "product", "description", "groups" }`.

### What to do in the Studio repo workflow

After you write `products/studio/navigation.json`, emit **`products/studio/hub-mintlify-product.json`** in the same job (so every push stays consistent). Example with `jq`:

```bash
jq '{ product: .product.product, description: .product.description, groups: .groups }' \
  products/studio/navigation.json > products/studio/hub-mintlify-product.json
```

Commit or upload both files to the docs repo in the same sync. Root `docs.json` then stays stable; only files under `products/studio/` change.

### Other options

- **Workflow in this (`docs`) repo** that runs when `products/studio/navigation.json` changes and patches or regenerates `hub-mintlify-product.json` (or inlines into `docs.json`). Useful if you truly cannot change the Studio pipeline yet.
- **Keep manual merges** only if you refuse both `$ref` and any automation; it does not scale when navigation changes often.

## Reviewing a Studio sync PR

1. Read the PR description and verify it is the expected Studio workflow.
2. Inspect the diff under `products/studio/` (MDX, assets, `navigation.json`, **`hub-mintlify-product.json`**, and `docs.json` if present—hub builds ignore it when listed in `.mintignore`).
3. Confirm **`hub-mintlify-product.json`** matches **`navigation.json`** (if the PR updates nav, the fragment should change in the same PR). If only `navigation.json` changed, ask for a regeneration step—the hub reads the fragment via `$ref`, not `navigation.json`.
4. Run `mint validate` (and optionally `mint broken-links`) before merging.

## Mintlify

Connect this GitHub repository in the Mintlify dashboard; default branch `main`; content root is the repository root where `docs.json` lives.

## Branding

Theme, colors, favicon, and logos match Studio’s platform docs tokens (`theme: mint`, primary `#7C3AED`, and logo paths under `/logo/`).

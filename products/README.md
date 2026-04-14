# `products/` directory

Each subdirectory under `products/` is owned by **that product’s repository** and its automation (for example, Studio sync opens pull requests that update `products/studio/`).

## Rules

1. **Do not hand-edit** files under `products/studio/` (or future `products/<id>/`) except by merging the product’s sync PRs. Local edits will be overwritten or will conflict with the next sync.
2. The hub root `docs.json` references **`products/studio/hub-mintlify-product.json`** via `$ref` for the Studio product sidebar. Your Studio sync job should generate that file from **`navigation.json`** every time (see root `README.md` for a `jq` example). Do not rely on hand-editing root `docs.json` when navigation changes.
3. This documentation site is **English-only** for hub-authored pages; marketing may remain bilingual on [alquimia.ai](https://alquimia.ai).

The root **`.mintignore`** lists **`products/studio/docs.json`** so Mintlify only uses the hub’s root `docs.json` and does not pick up the synced Studio standalone config.

# `products/` directory

Each subdirectory under `products/` is owned by **that product’s repository** and its automation (for example, Studio sync opens pull requests that update `products/studio/`).

## Rules

1. **Do not hand-edit** files under `products/studio/` (or future `products/<id>/`) except by merging the product’s sync PRs. Local edits will be overwritten or will conflict with the next sync.
2. The hub root **`docs.json`** pulls the **Studio** product sidebar via **`$ref`** to **`products/studio/navigation.json`**, with **`icon`** (and any other allowed siblings) set next to that **`$ref`** in **`docs.json`**. Keep **`navigation.json`** in sync with your MDX tree when you change the sidebar. See the repository root **[README.md](../README.md)** for the full multi-product pattern (InsightHub and future products use the same approach).
3. This documentation site is **English-only** for hub-authored pages; marketing may remain bilingual on [alquimia.ai](https://alquimia.ai).

The root **`.mintignore`** lists **`products/studio/docs.json`** so Mintlify only uses the hub’s root `docs.json` and does not pick up the synced Studio standalone config.

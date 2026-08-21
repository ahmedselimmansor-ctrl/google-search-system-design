# assets

This folder holds generated output. Nothing here is required to read the repository — GitHub renders every diagram natively from its Mermaid source.

## `rendered/`

Created on demand when you export diagrams to SVG or PNG. It is git-ignored, so exports stay local.

```bash
npm install -g @mermaid-js/mermaid-cli
```

```bash
mkdir -p assets/rendered && for f in diagrams/*.mmd; do mmdc -i "$f" -o "assets/rendered/$(basename "${f%.mmd}").svg" -b transparent; done
```

Add `-t dark` for a dark-theme variant, or change the extension to `.png` for raster output.

Diagram sources live in [`../diagrams/`](../diagrams/README.md).

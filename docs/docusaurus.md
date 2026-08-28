# CarMate Blueprint — Docusaurus Integration

The documents in this folder are included in the [Eclipse SDV Blueprints website](https://sdv-blueprints.eclipse.dev).
The website is built with [Docusaurus](https://docusaurus.io) and deployed as GitHub Pages.

- **Blueprints website repository:** https://github.com/eclipse-sdv-blueprints/blueprints-website
- **Blueprints website URL:** https://sdv-blueprints.eclipse.dev

## Including CarMate docs in the website build

Add the following entries to `blueprints-website/docusaurus.config.js` under the `plugins` array:

```javascript
plugins: [
  [
    "docusaurus-plugin-remote-content",
    {
      name: "carmate",
      sourceBaseUrl: "https://raw.githubusercontent.com/eclipse-sdv-blueprints/carmate/main/docs",
      outDir: "docs/carmate",
      documents: [
        "introduction.md",
        "architecture.md",
        "getting-started.md",
        "components.md",
      ],
      requestConfig: { responseType: "arraybuffer" },
    },
  ],
  [
    "docusaurus-plugin-remote-content",
    {
      name: "carmate-img",
      sourceBaseUrl: "https://raw.githubusercontent.com/eclipse-sdv-blueprints/carmate/main/docs/img",
      outDir: "docs/carmate/img",
      documents: ["tech_arch.png"],
      requestConfig: { responseType: "arraybuffer" },
    },
  ],
],
```

## Updating the website

After editing any file in this `docs/` folder, trigger the GitHub Action that builds the Eclipse SDV Blueprints website to publish the changes.

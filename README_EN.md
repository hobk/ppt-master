<p align="center"><img src="./public/favicon.png" alt="ppt-master" width="72" /></p>
<h1 align="center">ppt-master</h1>
<p align="center">A clean AI presentation studio · turn ideas into decks in minutes</p>
<p align="center">
  <a href="./README.md">简体中文</a>
  <span>&nbsp;·&nbsp;</span>
  <a href="https://docmee.cn/open-platform" target="_blank">Open API</a>
  <span>&nbsp;·&nbsp;</span>
  <a href="./docs/README_EN.md">Docs</a>
</p>

---

## Why ppt-master

ppt-master focuses on speed, stability, and visual polish. It condenses the full workflow—topic input → outline generation → template matching → live preview—
into three smooth steps so you can focus on the story, not the tooling.

### Premium Highlights

- **Mainstream model ecosystem**: Powered by Docmee Open API with access to leading enterprise model ecosystems for industry customization and private deployment.
- **Smart outlines**: Generate structured outlines from topics, text, or files and edit them visually.
- **Intelligent template matching**: Recommend presentation styles automatically to lock in a cohesive look.
- **Live generation preview**: Streamed generation updates with synchronized thumbnails and stage view.
- **High-quality export**: Generate PPTX assets ready for keynote, sales, training, and demo scenarios.
- **Easy integration**: Pure front-end UI with standard API calls for fast customization.

## Core Capabilities

- Topic/text/file → structured outline generation
- Visual outline editing at chapter/page/title levels
- Template recommendation and selection
- Live preview with PPTX download

## Tech Stack

- **Framework**: Vue 3 + Vite + TypeScript
- **Rendering**: SVG + Canvas dual renderers
- **Streaming**: SSE for real-time generation
- **Compression**: PPTX structure parsing + gzip
- **API**: Docmee Open API (mainstream enterprise model ecosystem access)

## Workflow

1. Choose an input type (topic / text / file)
2. Generate and edit the outline
3. Pick a template and generate PPTX

## Quick Start

```bash
npm install
npm run dev
```

## Configuration

- Set `apiKey` and `uid` in `src/components/AiPpt.vue`.
- In production, create tokens on the server and deliver them securely to the client.

## Docs

- Usage guide: `docs/README_EN.md`
- Docmee Open API: https://docmee.cn/open-platform

## License

MIT

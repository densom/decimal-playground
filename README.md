# Decimal Playground

**Live site: [https://apps.dennissomerville.net/decimals](https://apps.dennissomerville.net/decimals)**
Hosted as a static site on [Render](https://render.com).

An interactive tool that helps 5th graders visualize and understand decimals and fractions through hands-on grid exploration.

## Features

### Explore Decimals Tab
- **Six grid modes** — Fourths, Eighths, Sixteenths, Tenths, Hundredths, Thousandths
- Click or drag to fill and erase cells; the decimal value updates live
- Type any decimal or fraction (e.g. `0.5` or `3/4`) to auto-fill the grid
- Quick-fill buttons for ½ and ¼
- **Keep Together mode** — cells animate as a cluster; click to watch a cell fly into position
- Visual grouping indicators show place-value relationships between modes

### LCD Explorer Tab
- Enter two fractions and find their **Lowest Common Denominator**
- Visual fraction squares show each fraction as a filled grid
- Equivalent fraction transformation with step-by-step explanation
- Side-by-side comparison with `<`, `>`, or `=`
- Addition bonus showing the sum over the LCD
- Fully responsive and scrollable

## Tech Stack

- Vue 3 + Vite
- pnpm
- Single-page app, no router

## Getting Started

```bash
pnpm install
pnpm dev        # dev server at http://localhost:5173
pnpm build      # production build → dist/
pnpm preview    # preview the production build
```

Create an interactive tool to help 5th grade children visualize and understand decimals. Kids should be able to interact with the tool in a way that helps them learn. Make it fun.

## Tech

- Vue 3 + Vite, pnpm
- Single-page app, no router
- `src/App.vue` — main app (~1200 lines)
- `src/components/LcdExplorer.vue` — LCD Explorer tab (~400 lines)

## Current Features

**Explore Decimals tab** (`activeTab === 'explore'`):
- Interactive grid with six modes: fourths / eighths / sixteenths / tenths / hundredths / thousandths
- Click/drag to fill or erase cells; shows decimal value live
- Type a decimal value to auto-fill the grid
- Quick-fill buttons (½, ¼)
- Cluster mode: groups cells into visual clusters that "fly" into a regrouping animation
- Animated cell-to-cell transitions (flyAnim)

**LCD Explorer tab** (`activeTab === 'lcd'`):
- Fraction square visualizer (FracSquare component, inline) — responsive, fills container
- Finds LCM of two denominators
- Grid-based fraction comparisons
- Fully responsive; scrollable via `.lcd-scroll` wrapper in App.vue

## Run

```bash
pnpm dev      # dev server
pnpm build    # production build → dist/
```

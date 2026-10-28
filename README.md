# TheTerminal

A live 3D container-terminal operations game for the fictional **Port Mumbai**, built as a single static page with [three.js](https://threejs.org/) (r128, loaded from cdnjs).

Run Terminal 2: berth ships, discharge imports, receive and load exports, plan import/export yard blocks, and manage the tide, wind, gate traffic and carrier trust.

Vibe-coded with Claude Code + Opus 5.5

Enjoy the Game 🎮 https://datoway.github.io/TheTerminal/

## Run locally

No build step. Open `index.html` in a browser, or serve the folder:

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

## Controls

| Input | Action |
|---|---|
| Drag | Pan |
| Right-drag / Shift-drag | Rotate |
| Scroll / pinch | Zoom |
| W A S D, Q E | Pan, rotate |
| Space | Pause |
| 1 / 2 / 3 | 1×, 2×, 4× speed |
| / | Search vessels, cranes, blocks, tractors |

## Deploy (Render)

`render.yaml` defines a static site that publishes the repo root with no build command. In Render, use **New → Blueprint** (or **New → Static Site** with publish directory `.`) and select this repository.

## Notes

- Light and dark themes follow the system setting; the toggle in the top bar overrides it and is remembered in `localStorage`.
- Nothing else is persisted: reloading starts a new shift.

# 🎬 GIF Maker

A simple, privacy-friendly **video-to-GIF converter** that runs entirely in the
browser using [ffmpeg.wasm](https://github.com/ffmpegwasm/ffmpeg.wasm).
No server, no uploads — your video never leaves your device.

## Features

- Drag & drop or click to select a video (max 200 MB)
- Preview the first frame, file size, dimensions, and duration
- Adjustable **frame rate** (smoothness), **speed** (shortens duration), and **size**
- Real conversion progress bar
- Before / after size & dimension comparison
- In-page GIF preview
- One-click download (folder picker on Chrome/Edge, standard download elsewhere)

## How it works

Everything runs client-side. The page loads `ffmpeg.wasm` (~25 MB, cached after
first load) and runs FFmpeg directly in the browser. The conversion uses a
two-pass palette filter for better GIF quality:

```
setpts={1/speed}*PTS, fps={fps}, scale={width}:-1:flags=lanczos,
split[s0][s1];[s0]palettegen[p];[s1][p]paletteuse
```

`speed` shortens the clip via `setpts`; `fps` controls smoothness — they work
independently.

## Run locally

```bash
python3 -m http.server 8000
# open http://localhost:8000
```

> Use a local HTTP server — opening `index.html` via `file://` may fail because
> of ES module / CORS restrictions.

## Deploy to GitHub Pages

1. Push this repo to GitHub.
2. Go to **Settings → Pages**.
3. Under **Source**, choose the `main` branch and `/ (root)` folder.
4. Save. Your site goes live at `https://<username>.github.io/<repo>/`.

No build step required — it's a single static `index.html`.

## Notes & limitations

- **GIFs are often larger than the source video** — that's just how the GIF
  format works (no inter-frame compression like MP4). The UI states this
  honestly and suggests lowering fps/size to shrink output.
- Very large or long videos may run out of browser memory; the app catches this
  and suggests reducing fps or dimensions.
- The folder picker uses the File System Access API (Chrome/Edge). Other
  browsers fall back to a normal download.

## Project structure

```
.
├── index.html      # the entire app (HTML + CSS + JS)
├── CLAUDE.md        # project context & editing rules for Claude Code
├── README.md
├── LICENSE
└── .gitignore
```

## License

MIT

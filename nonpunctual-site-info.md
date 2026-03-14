# nonpunctual-site-info

## Colors

Light mode (theme defaults, except tip which is overridden in custom.css):

| Type      | Icon color | Accent  |
|-----------|------------|---------|
| note      | `#5e35b1`  | Purple  |
| tip       | `#ff8000`  | Orange  |
| info      | `#1e88e5`  | Blue    |
| warning   | `#ffb300`  | Amber   |
| error     | `#e53935`  | Red     |
| example   | `#6d4c41`  | Brown   |
| question  | `#7cb342`  | Green   |

Dark mode (overridden in custom.css for note, tip, info, question; warning/error/example fall back to light mode colors):

| Type      | Icon color | Accent     |
|-----------|------------|------------|
| note      | `#8c8c8c`  | Gray       |
| tip       | `#3d6b35`  | Dark green |
| info      | `#76d6ff`  | Light blue |
| question  | `#ffffff`  | White      |

---

## Shortcodes (added)

### apple-music
Embeds an Apple Music player iframe. Never use Spotify.

```
{{< apple-music "https://embed.music.apple.com/us/album/example/000000000" >}}
```

- URL must be an Apple Music embed URL (not the standard share link).

---

### audio-player
Embeds an HTML5 audio player. Audio files are served from `static/audio/`.

```
{{< audio-player "filename.mp3" >}}
```

---

### gear-photos
Displays a responsive photo grid. Images are served from `static/images/gear/`.

```
{{< gear-photos "photo1.jpg" "photo2.jpg" "photo3.jpg" >}}
```

- Accepts 1–3 image filenames as positional parameters.

---

### pdf
Embeds a PDF in a collapsible block. PDFs are served from `static/docs/`.

```
{{< pdf "filename.pdf" "Display Title" >}}
```

- First parameter: filename
- Second parameter: title shown on the collapse toggle and as the iframe title

---

### pullquote
Displays a large centered pull quote with optional attribution.

```
{{< pullquote "Quote text goes here." "Attribution Name" >}}
```

- Attribution is optional. Omit the second parameter for a quote-only block.

---

### social
Renders the social icon row from `hugo.toml` params. Used on the contact page.

```
{{< social >}}
```

---

## Shortcodes (Hugo built-in)

### figure
Image with optional caption.

```
{{< figure src="/images/path/to/image.jpg" alt="Alt text" caption="Optional caption" >}}
```

---

### mermaid (hugo-coder theme)
Renders a Mermaid diagram. Gantt charts are styled monochrome automatically.

```
{{< mermaid >}}
gantt
    dateFormat  YYYY-MM-DD
    section Phase One
    Task name   :2025-01-01, 14d
{{< /mermaid >}}
```

---

### notice (hugo-coder theme)
Alert block. Available types: `note`, `tip`, `info`, `warning`, `error`, `example`, `question`.

```
{{< notice info >}}
Your message here.
{{< /notice >}}
```

---

### youtube
Embeds a YouTube video.

```
{{< youtube "video-id" >}}
```

- The video ID is the part after `v=` in a YouTube URL.

---


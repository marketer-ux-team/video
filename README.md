# video

Statisches Video-Hosting für marketer-ux.com. Das Repo ist die Quelle des Vercel-Projekts `video`
(Team `marketer-uxs-projects`), das unter `https://video-lyart-one.vercel.app` liegt und bei jedem
Push auf `main` automatisch neu deployt. Dateien im Repo werden 1:1 unter ihrem Pfad ausgeliefert.

Zweck ist, die Videos der Website ohne fremden Player auszuliefern. Der Ordner `hero/` löst das
Wistia-Abo für die Startseite ab: statt eines iframes mit rund 360 KB Player-JavaScript lädt die
Seite eine MP4-Datei direkt. Vercel liefert statische Dateien mit `accept-ranges: bytes` aus, also
funktioniert Spulen über Range-Requests, und weil kein Drittanbieter mehr im Spiel ist, läuft das
Video auch ohne Cookie-Zustimmung. Die vier `animation-*`-Clips im Wurzelverzeichnis sind älter und
werden als Scroll-Animationen auf der Website genutzt; sie bleiben unberührt.

## Was wo eingebunden ist

| Datei | Größe | Eingebunden auf marketer-ux.com |
|---|---|---|
| `hero/hero-1080-v1.mp4` | 47,7 MiB | Hero-Embed der Startseite, ab Viewport 768 px. Wird per Klick auf das Vorschaubild geladen. |
| `hero/hero-720-v1.mp4` | 22,1 MiB | Derselbe Hero-Embed unter 768 px Viewport-Breite. |
| `hero/hero-poster-v1.webp` | 18,3 KB | Vorschaubild des Hero-Embeds (`<img>`) und `poster` des `<video>`. |
| `hero/hero-de-v1.vtt` | 6,9 KB | Deutsche Untertitel des Hero-Videos, als `<track>` am `<video>`. 90 Cues. |
| `hero/esc-loop-720-v1.mp4` | 1,8 MiB | ESC-Karte in der Referenzen-Section der Startseite. Stummer 10-Sekunden-Loop, startet beim Scrollen. |
| `animation-*.webm` / `animation-*.mov` | | Ältere Scroll-Animationen, unverändert. |

Die HTML-Schnipsel für die beiden Webflow-Embeds sowie für Page-Head und Page-Footer der
Startseite liegen unter `webflow/`. Sie werden im Webflow-Designer per Copy-Paste eingesetzt, weil
sich Embed-Inhalte über die Data-API nicht schreiben lassen. `webflow/hero-css-patch.md`
beschreibt die wenigen Zeilen, die im bestehenden Hero-Style-Embed anzupassen sind.

## Versionierung: niemals eine Datei überschreiben

`vercel.json` setzt für alles unter `/hero/` den Header
`Cache-Control: public, max-age=31536000, immutable`. Browser und CDN dürfen diese Dateien damit ein
Jahr lang halten, ohne je nachzufragen. Eine Datei unter demselben Namen zu ersetzen bringt deshalb
nichts — Besucher sehen weiter die alte Fassung.

Neue Fassungen bekommen darum eine neue Nummer im Dateinamen: aus `hero-1080-v1.mp4` wird
`hero-1080-v2.mp4`. Die alte Datei kann im Repo bleiben, bis die Website umgestellt und veröffentlicht
ist. Danach lässt sie sich gefahrlos löschen. Dieselbe Regel gilt für Poster und Untertitel.

## Wie die Dateien entstanden sind

Quelle ist das 4K-Original aus Wistia (`n58wkinar9`, 3:56 min, 3840×2160, H.264 High, 30 fps). Von dort
wird herunterskaliert, weil das die bessere Qualität ergibt als ein erneutes Encoding der bereits
komprimierten 1080p-Fassung. Werkzeug ist ffmpeg 9.0.1.

```bash
# 1080p — Ziel: unter 50 MiB bleiben (Warnschwelle von GitHub)
ffmpeg -i src-4k-original.mp4 -vf scale=1920:1080 \
  -c:v libx264 -profile:v high -pix_fmt yuv420p -preset slow -crf 25 \
  -c:a aac -b:a 96k -movflags +faststart hero-1080-v1.mp4

# 720p für schmale Viewports
ffmpeg -i src-4k-original.mp4 -vf scale=1280:720 \
  -c:v libx264 -profile:v high -pix_fmt yuv420p -preset slow -crf 26 \
  -c:a aac -b:a 96k -movflags +faststart hero-720-v1.mp4

# ESC-Loop: nur neu muxen, kein Re-Encode, Tonspur raus
ffmpeg -i esc-720.mp4 -an -c:v copy -movflags +faststart esc-loop-720-v1.mp4

# Poster aus dem 4K-Standbild
cwebp -q 80 -resize 960 540 still.png -o hero-poster-v1.webp
```

`-movflags +faststart` schiebt den `moov`-Atom an den Dateianfang. Ohne das müsste der Browser erst
die ganze Datei laden, bevor er abspielen kann. Nach jedem Encoding lohnt eine Kontrolle:

```bash
ffprobe -v trace hero-1080-v1.mp4 2>&1 | grep -m2 -oE "type:'(moov|mdat)'"
# erwartet: moov vor mdat
```

Die Untertitel stammen aus `https://fast.wistia.com/embed/captions/n58wkinar9.json`. Das JSON liefert
unter `captions[].hash.lines[]` Cues mit `start`/`end` in Sekunden und `text` als Array von Zeilen;
daraus entsteht die WebVTT-Datei. Wenn Wistia irgendwann abgeschaltet ist, lässt sich stattdessen ein
SRT-Export mit `ffmpeg -i hero.srt hero-de-v1.vtt` umwandeln.

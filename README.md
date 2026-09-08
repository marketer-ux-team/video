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
| `hero/wel-720-v1.mp4` | 12,9 MiB | Video auf `/website-erstellen-lassen`, ersetzt Wistia `zpebt984kw`. 1280×720, 2:54 min. Beide `data-video-src-*` zeigen darauf, siehe unten. |
| `hero/wel-poster-v1.webp` | 35,9 KB | Vorschaubild dieses Embeds (`<img>`) und `poster` des `<video>`. 960×540. |
| `hero/wel-de-v1.vtt` | 5,7 KB | Deutsche Untertitel dazu, als `<track>` am `<video>`. 73 Cues. |
| `animation-*.webm` / `animation-*.mov` | | Ältere Scroll-Animationen, unverändert. |

Trotz des Präfixes liegen die `wel-*`-Dateien bewusst in `hero/`: nur für diesen Pfad setzt
`vercel.json` den Immutable-Header, und die Versionsregel weiter unten gilt für sie genauso.

## Der Ordner `webflow/`

Hier liegt der Webflow-Teil der Umstellung (Site `64889a266012bdd6373f2952`, Page
`69e355534b2b266220042310`). Stand 2026-09-08 ist die Startseite umgestellt und veröffentlicht.

| Datei | Was |
|---|---|
| `hero-embed-full.html` | **Maßgeblich.** Kompletter Inhalt des Hero-Embeds (Element `adfa4dd4-1781-1c9b-e2db-8832ca189328`): Markup, CSS und das Click-to-play-Script in einem Embed. Genau so per API in Webflow geschrieben. Verhalten siehe „Der Player ohne Bedienleiste“. |
| `wel-embed-full.html` | **Maßgeblich** für `/website-erstellen-lassen`. Inhaltlich dieselbe Fassung wie `hero-embed-full.html`, aber mit dem Präfix `wel_video-wrapper` / `wel-video…` / `wel-loader`, damit nichts mit den Webflow-Klassen dieser Seite kollidiert. Die Legacy-Zweige (`dd-wistia-*`, Konstanten im Script) sind hier raus. **Noch nicht in Webflow eingespielt.** |
| `esc-embed.html` | Inhalt der ESC-Karte (Element `adfa4dd4-1781-1c9b-e2db-8832ca18955e`): stummer Loop als natives `<video>`. Per API geschrieben. |
| `hero-embed.html`, `hero-css-patch.md` | Bausteine von `hero-embed-full.html`, nur noch als Referenz. |
| `home-head.html`, `home-footer.html` | Bereinigter Page-Head- und Page-Footer-Code (ohne Wistia-`preconnect`, ohne Wistia-Click-Script, `[data-video-src]`-Script nur einmal). **Noch nicht eingespielt**, siehe unten. |
| `backup/home-head-original.html`, `backup/home-footer-original.html` | Page-Head- und Page-Footer-Code, wie er vor der Umstellung live war. Byte-genau über die Webflow-API gelesen. |
| `home-head-snippet.html`, `home-footer-snippet.html` | Erste Entwürfe, nur noch als Referenz. |

## Der Player ohne Bedienleiste

Der Kunde will das Verhalten des alten Wistia-Players mit `playbar:false`: Video inline im Hero,
keine native Bedienleiste, kein Popup. Das `<video>` läuft deshalb mit `controls = false`; gesteuert
wird ausschließlich über den mittigen Play-Button und einen Klick aufs Video.

- **Start:** Klick auf den Button lädt das Video und spielt es **mit Ton** (kein `muted`) — erlaubt,
  weil der Klick eine echte Nutzergeste ist.
- **Pause:** Klick aufs laufende Video pausiert. Der Wrapper bekommt `is-paused`; darüber wird der
  Play-Button wieder eingeblendet, während das Vorschaubild auf `opacity: 0` bleibt. Sichtbar ist
  also das **stehende Videobild**, nicht das Poster. Klick auf den Button spielt weiter.
- **Ende:** `is-loaded` fällt weg, `currentTime` geht auf 0 — Vorschaubild und Play-Button stehen
  wieder wie am Anfang, der nächste Klick spielt von vorn.
- **Untertitel** bleiben über `<track … default>` eingeblendet. Der Track ist Cross-Origin, deshalb
  ist `crossOrigin = "anonymous"` Pflicht (sonst bleiben die Cues leer). Zur Größe siehe den
  folgenden Absatz.
- **Bedienung ohne Maus:** das erzeugte `<video>` bekommt `tabindex="0"` und einen sichtbaren
  Fokusring (`video:focus-visible`). Leertaste, Enter und `k` schalten auf Video **und** Button
  zwischen Wiedergabe und Pause. Der Button ist nativ mit Leertaste/Enter auslösbar; weil
  `preventDefault` (gegen das Scrollen) diesen synthetischen Klick unterdrückt, schaltet der
  `keydown`-Handler selbst um — sonst käme es zum Doppel-Toggle.
- **Kein Kontextmenü auf dem Video** (`contextmenu` → `preventDefault`): Safari und Firefox bieten
  dort sonst „Bild-in-Bild", „Video sichern" und „Vollbild" an. Im Pausenzustand liegt das
  Button-Overlay über dem Video, dort greift der Handler nicht — das native Menü gehört dann zum
  Button, nicht zum Video.

**Untertitelgröße — keine Prozentangabe in `::cue`.** Bis V3 stand `video::cue { font-size: 150% }`
statisch im Embed. In Chrome ergibt das das Erwartete (1,5-fache der Standardgröße), auf iOS Safari
dagegen riesigen Text: WebKit bezieht die Prozentangabe auf eine andere Basis. Auf einem iPhone 17 Pro
(Simulator, Video 402 × 226 CSS-px) belegte derselbe einzeilige Cue mit `150 %` **40,6 % der
Videohöhe** über zwei Zeilen und lief bis über den Play-Button — Kundenmeldung bestätigt.

Seit V4 steht in `::cue` deshalb **kein `%` und kein `em` mehr**. Stattdessen rechnet das
Embed-Script aus der gerenderten Wrapperhöhe eine feste px-Größe und injiziert sie als eigene Regel
in ein `<style>`-Element im Wrapper:

```js
const px = Math.max(12, Math.round(wrapper.getBoundingClientRect().height * 0.055));
cueStyle.textContent = ".hero_video-wrapper video::cue { font-size: " + px + "px; }";
```

`applyCueSize()` läuft beim Aufbau und danach an einem `ResizeObserver` auf dem Wrapper
(`window.resize` als Rückfall), der Wert folgt also der Spaltenbreite. Gemessen: Desktop 1280 px →
Wrapper 529 px → **29 px**, Mobile 390 px → Wrapper 219 px → **12 px** (Untergrenze), iPhone 17 Pro →
Wrapper 226 px → **12 px**. Beim Viewport-Wechsel auf derselben Seite rechnet die Regel neu
(29 → 12 → 29).

Der Faktor 0,055 ist bewusst dicht an der Standardgröße: **Chrome und Safari setzen Cues von Haus aus
auf 5 % der Videohöhe** (in Chrome 152 kalibriert — Cue-Box-Breite wächst linear mit
13,0 px je Schrift-px, die Standardgröße rechnet sich damit zu 26,5 px auf einem 529 px hohen Video).
0,055 liegt also rund **1,1-fach** darüber. Wer die Untertitel deutlicher größer will, dreht allein an
`CUE_FACTOR` — 0,065 wären 1,3-fach. Auf iOS belegt eine Zeile damit 8,2 % der Videohöhe statt 40,6 %.

**Untertitelposition:** dafür ist bewusst *nichts* eingestellt. Es lag der Verdacht nahe, dass die
Cue-Box mit 150 % unten aus dem Video läuft und über WebVTT-Cue-Settings (`line:-3`) angehoben werden
muss — `::cue` kennt keine Positionierung, das ginge nur über die VTT-Datei. In Chrome 152 nachgemessen
stimmt der Verdacht nicht: bei zweizeiligen Cues bleiben 8 px, bei einzeiligen 4 px Abstand zum unteren
Videorand, auf 1280 px wie auf 390 px. `line:-3` würde die Untertitel stattdessen um 50 px (Desktop,
zweizeilig) bis 104 px (Desktop, einzeilig) nach oben schieben und die Unterkante zwischen ein- und
zweizeiligen Cues springen lassen, weil `line` die **erste** Zeile verankert. Automatische Positionierung
ist hier also die bessere. Falls doch einmal etwas anzuheben ist: Setting an jede Zeitzeile der VTT
anhängen (`00:00:01.000 --> 00:00:03.500 line:-3`), eine neue Dateiversion anlegen — `/hero/` ist
immutable gecacht.

Das Vorschaubild darf ab dem Wiedergabestart **nie wieder sichtbar werden**. Dafür sorgt
`.hero_video-wrapper.is-loaded .hero-video_thumbnail { opacity: 0 }`, und `.hero-video_thumbnail`
hat bewusst **keine** `transition`. Beides gehört zusammen: hing die Deckkraft nur an `.is-paused`
und fadete über 0,35 s, blitzte beim Pausieren für einen Moment das Poster auf, während der Button
einblendete (Kundenmeldung zu V2, in Chrome gemessen: `opacity` 1 → 0 über rund 350 ms). Erst am
Videoende fällt `is-loaded` weg, dann kommt das Vorschaubild absichtlich zurück, während das Video
ausblendet.

`controls = false` allein reicht auf dem iPhone nicht: iOS blendet sonst weiterhin Vollbild, PiP,
±10 s und AirPlay ein. Nötig sind zusätzlich `controlslist="nodownload nofullscreen noremoteplayback"`,
`disableRemotePlayback` + `disableremoteplayback`, `x-webkit-airplay="deny"`, `disablepictureinpicture`
sowie `playsinline` **und** `webkit-playsinline` als Attribute (ältere iOS-Safari lesen die Property
nicht). Ein `dblclick`-Listener mit `preventDefault()` unterbindet den Vollbild-Doppelklick.

Was die Webflow-API kann und was nicht (mit dem Webflow-MCP 2.0.1 verifiziert):

- HtmlEmbed-Inhalte lassen sich lesen und schreiben (`data_element_settings_tool`, Setting `code`),
  auch mit `<script>` darin. Deshalb sitzt das Click-to-play-Script direkt im Hero-Embed.
- Freeform-Custom-Code (Page-Head/-Footer) lässt sich lesen, beim Schreiben antwortet die API mit
  `HTTP 406`, sobald der Inhalt `<script>` oder `<link>` enthält (`<style>` und Kommentare gehen).
  Head und Footer der Startseite sind darum unverändert geblieben. Übrig sind dort vier harmlose
  Wistia-`preconnect`/`dns-prefetch`-Zeilen und das alte Wistia-Click-Script, das nichts mehr findet.
  Wer das aufräumen will, ersetzt in den Page-Settings der Startseite den Head durch
  `home-head.html` und den Footer durch `home-footer.html` (beide vollständig).
- Publish geht per REST: `POST /v2/sites/{site_id}/publish` mit `customDomains`.

Seite `/website-erstellen-lassen` (Page `653a9ed8ced44f4759b93170`): dort war Wistia ein natives
Webflow-Video-Element (Embedly). Per API wurde daneben ein HtmlEmbed angelegt (Element
`d8a83bc9-2441-f79c-b067-0cc7c62972fb`, Inhalt `webflow/wel-embed-full.html`), das alte Video-Element
entfernt. Wichtig: der Embed braucht die Webflow-Klasse `wistia-video_component` (`width:100%;
aspect-ratio:16/9; border-radius:1.75rem`), weil der umgebende `.video-preview_wrapper` ein
Flex-Container ist und ein Embed ohne Klasse dort 0×0 px groß bleibt. Der Klassenname ist nur noch
historisch, die Klasse enthält kein Wistia mehr.

### Das Click-to-play-Script

Es reagiert auf `[data-video-trigger]` (neues Embed) und zusätzlich auf `[dd-wistia-video-trigger]`
(altes Embed, falls es je zurückgesetzt wird). Im Legacy-Fall kommen die URLs aus
Konstanten im Script, und das Vorschaubild `img.hero-video_thumbnail` wird beim Laden auf das
selbst gehostete WebP umgeschrieben — so wird auch ohne Embed-Tausch kein Bild mehr von Wistia
geladen. Das erzeugte `<video>` bekommt Inline-Styles (`position:absolute; inset:0; …
object-fit:cover`), liegt also auch dann richtig, wenn der CSS-Patch aus `hero-css-patch.md` noch
nicht eingespielt ist.

Eine Falle beim Lesen der Quellen: `data-video-src-1080` landet **nicht** unter
`dataset.videoSrc1080`. Die Umwandlung in camelCase greift nur, wenn auf den Bindestrich ein
Kleinbuchstabe folgt — bei einer Ziffer bleibt der Bindestrich stehen, der Schlüssel heißt also
`dataset["videoSrc-1080"]`. Das Script liest diese Attribute deshalb mit `getAttribute()`.

Das `<video>` bekommt `crossOrigin = "anonymous"`. Ohne CORS-Modus lädt der Browser die
Untertitel-Datei vom fremden Origin nicht (die `<track>`-Cues bleiben leer); Vercel sendet
`access-control-allow-origin: *`, mehr ist nicht nötig. Live in Chrome geprüft: 90 Cues.

Nichts im Script ist Chrome-only: `matchMedia`, `closest`, `textTracks`, `ResizeObserver`
(mit `window.resize` als Rückfall) und Pfeilfunktionen sind überall Baseline — kein `?.`, kein
`??`, keine privaten Felder. Verifiziert in echten Engines über Playwright (Chromium, Firefox,
WebKit, je Desktop 1280 × 800 und Mobile 390 × 844, beide Embeds): 264 Checks grün, darunter
Wiedergabe mit Ton, geladene Cues, injizierte px-Regel, Klick- und Tastatur-Bedienung sowie der
unterdrückte Rechtsklick. In Firefox erscheint beim Hover über dem laufenden Video **kein**
Bild-in-Bild-Umschalter — `disablePictureInPicture` wird respektiert.

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

### Das Video von `/website-erstellen-lassen`

Quelle ist Wistia `zpebt984kw` („chris-webseite erstellen lassen", 2:54 min). Die Metadaten liegen
unter `https://fast.wistia.com/embed/medias/zpebt984kw.json`, die beste Fassung in `assets[]` ist
`original` mit **nur 1280×720** (H.264 Main, 25 fps, `.bin`-URL). Höher geht es nicht — deshalb
**kein Upscaling** und nur eine Datei; im Embed zeigen `data-video-src-1080` und `-720` beide auf
`wel-720-v1.mp4`.

```bash
# 720p, aus dem 720p-Original (kein Upscaling — es gibt keine groessere Quelle)
ffmpeg -i src-original.mp4 -vf scale=1280:720 \
  -c:v libx264 -profile:v high -pix_fmt yuv420p -preset slow -crf 26 \
  -c:a aac -b:a 96k -movflags +faststart wel-720-v1.mp4

# Poster aus dem still_image der Media-JSON (kommt als PNG, trotz .jpg-Endung)
cwebp -q 80 -resize 960 540 still.jpg -o wel-poster-v1.webp
```

Untertitel wie beim Hero aus `https://fast.wistia.com/embed/captions/zpebt984kw.json`, 73 Cues.
Ohne Cue-Settings (`line:` & Co.) — Chrome und Safari setzen die Untertitel von selbst korrekt
unten ins Video; siehe den Absatz zur Untertitelposition. Die Größe kommt wie beim Hero aus dem
Script (`CUE_FACTOR`, Selektor `.wel_video-wrapper video::cue`), nicht aus einer statischen Regel.

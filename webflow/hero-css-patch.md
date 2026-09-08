# CSS-Patch für den Hero-Style-Embed (Startseite)

Der Hero hat einen eigenen `<style>`-Embed weiter oben auf der Seite. Der bleibt bis auf die
folgenden Stellen unverändert — `.hero_video-wrapper`, `.hero-video`, `.hero-video_thumbnail`,
`.hero-video_play` und die `.is-loading`-Regeln werden nicht angefasst.

## 1. Die beiden `iframe`-Selektoren auf `video` umstellen

Das Wistia-`<iframe>` wird durch ein natives `<video>` ersetzt, das das Footer-Script in den
Wrapper hängt. Damit die vorhandene Einblend-Logik (opacity 0 → 1 bei `.is-loaded`) weiter
greift, müssen beide Selektoren auf `video` zeigen.

Vorher:

```css
.hero_video-wrapper iframe { … }
.hero_video-wrapper.is-loaded iframe { … }
```

Nachher:

```css
.hero_video-wrapper video { … }
.hero_video-wrapper.is-loaded video { … }
```

## 2. Im ersten der beiden Blöcke zwei Deklarationen ergänzen

Ein `<video>` skaliert anders als ein `<iframe>`. Damit das Bild das 16:9-Feld formatfüllend
belegt und beim Puffern kein heller Rand blitzt, kommen in `.hero_video-wrapper video` dazu:

```css
object-fit: cover;
background: #000;
```

Die bestehenden Deklarationen (Positionierung, Größe, `opacity`, Transition) bleiben, wie sie sind.

## 3. Keyframes umbenennen

Der Spinner heißt heute nach dem alten Anbieter. Beide Vorkommen umbenennen — die Definition und
die `animation`-Kurzschreibweise in der `.is-loading`-Regel:

```css
@keyframes wistia-loader { … }   →   @keyframes hero-loader { … }
animation: wistia-loader …       →   animation: hero-loader …
```

Nach dem Patch enthält der Style-Embed keinen Verweis mehr auf Wistia. Ein `Cmd+F` auf „wistia"
im Embed muss leer bleiben.

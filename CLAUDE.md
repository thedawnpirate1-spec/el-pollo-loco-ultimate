# EL POLLO LOCO ULTIMATE — PRODUCTION BUG FIX
# Datei: el-pollo-loco-ultimate.html (single file, ~2267 Zeilen)
# Ziel: Kein einziger spürbarer Fehler. Release-ready.

## OUTPUT-REGEL
Kein Kommentar, kein Preamble, kein Fortschrittsbericht.
Nur Fixes direkt im Code. Inline-Kommentar nur bei nicht-offensichtlicher Logik: `// FIX: [Grund]`
Am Ende NUR die Report-Tabelle (Format unten).

---

## BEKANNTE BUGS — DIREKT FIXEN (mit Fundstelle)

### 🔴 BUG 1 — Globale Variable `dt0` in `Coin.collide()` [Zeile ~381]
```js
// PROBLEM: collide() nutzt dt0 (globale Variable) statt dem dt-Parameter
this.vy += GRAVITY * dt0 * 1;
this.y += this.vy * dt0;
```
`dt0` wird in `Game.update()` gesetzt, BEVOR `coin.update()` läuft — bei hoher Last
oder Tab-Wechsel stimmt der Wert nicht mehr. Außerdem: `Coin.update(dt, game)` empfängt
dt korrekt, übergibt es aber NICHT an `collide()`.
**Fix:** `collide(platforms, dt)` als Signatur, Aufruf: `this.collide(game.platforms, dt)`,
intern `dt0` → `dt`.

---

### 🔴 BUG 2 — Hazard-Kollision LavaBubble/Meteor: falscher Guard [Zeile ~1641]
```js
if (hz.active !== false && !hz.warn > 0) {}   // <— leerer Block, Bedingung falsch
const hb = { x: hz.bx, y: hz.by, w: hz.w, h: hz.h };
if ((hz.warn === undefined || hz.warn <= 0) && aabb(hb, pl) && pl.invuln <= 0) pl.takeHit(1);
```
`!hz.warn > 0` → `(!hz.warn) > 0` → `false > 0` → immer false.
Damit wird die Hitbox-Prüfung für Meteor immer ausgeführt, auch während der Warn-Phase
(Meteor noch nicht sichtbar), sodass der Spieler Schaden nimmt bevor der Meteor landet.
**Fix:** Leeren Block und kaputten Guard entfernen. Einheitliche Bedingung:
```js
if (hz.warn <= 0 && !hz.exploded && aabb({ x: hz.bx, y: hz.by, w: hz.w, h: hz.h }, pl) && pl.invuln <= 0)
  pl.takeHit(1);
```

---

### 🔴 BUG 3 — `_deathTimer` wird bei `startLevel()` nicht zurückgesetzt [Zeile ~1713-1716]
```js
if (pl.dead && this.elapsed > 0 && pl.hearts <= 0) {
  this._deathTimer = (this._deathTimer || 0) + dt;
  if (this._deathTimer > 1200) this.lose();
}
```
`_deathTimer` ist nie in `resetWorld()` oder `startLevel()` deklariert, nur lazy-initialisiert.
Nach Neustart mit "NOCHMAL" ist der Timer vom vorherigen Run noch im Objekt gespeichert.
Direkt nach dem Start wenn `pl.dead === false` greift er nicht — ABER bei schnellem
Neustart nach Tod kann der State remnant das nächste `lose()` früher auslösen.
**Fix:** `this._deathTimer = 0;` in `resetWorld()` hinzufügen.

---

### 🔴 BUG 4 — Boss Phase-Transition: `Audio.bpmMul` direkt gesetzt statt über AudioEngine [Zeile ~778, 786]
```js
safe(() => Audio.bpmMul = 1.4, "boss.bpm2");
safe(() => Audio.bpmMul = 1.8, "boss.bpm3");
```
`Audio.bpmMul` ist eine Property der `AudioEngine`-Instanz und wird in `updateMusic()` gelesen.
Das funktioniert, ABER: nach `Audio.stopMusic()` und `Audio.setMusic(level)` (bei `startLevel`)
wird `bpmMul` wieder auf `1` zurückgesetzt. Die setze-Aufrufe in `enterPhase()` passieren
aber schon korrekt. Echter Bug: `bpmMul` wird in `setMusic()` auf `1` reset, was gut ist,
aber `enterPhase()` setzt es via `Audio.bpmMul = 1.4` ohne `safe`-Guard auf die richtige
Property — der `safe`-Aufruf wrapped eine Arrow-Function, die eine Property setzt, was
immer gelingt. Eigentliches Problem: in Phase-3 wird `applyScale` ZWEIMAL aufgerufen:
```js
this.applyScale(3.5 / 3);  // → scale = 1.166...
this.applyScale(1.17);      // → zweimal skaliert! boss wird RIESIG (1.37x statt 1.17x)
```
**Fix:** Ersten `applyScale`-Aufruf entfernen, nur `this.applyScale(1.17)` behalten.

---

### 🟠 BUG 5 — Coin `collectCoin()`: `c.dead = true` sofort gesetzt, `c.flying` aber auf false [Zeile ~1516]
```js
c.collected = true; c.flying = false; c.dead = true;
// flying tween visual
this.flyingCoins.push({ x: c.x, y: c.y, t: 0, type: c.type, sx: c.x, sy: c.y });
```
Die Münze wird sofort `dead = true` gesetzt, was sie beim nächsten Frame-Cleanup entfernt
(`this.coins = this.coins.filter(c => c.isAlive())`). Der Flying-Tween in `flyingCoins`
ist davon unabhängig (separates Array, reine Visualisierung) — korrekt.
ABER `c.flying = false` macht den `Coin.update()` Guard `if (this.flying) return` wirkungslos.
In `Coin.update()` Zeile ~407-411 wird `flying` gesetzt wenn der Coin eine Tween-Animation
machen soll — aber `collectCoin` setzt es sofort auf false. Das ist inkonsistent.
**Fix:** `c.flying = false` aus `collectCoin()` entfernen (war historischer Rest).

---

### 🟠 BUG 6 — `dangerZone` nicht in `resetWorld()` [Zeile ~1373]
```js
resetWorld() {
  // ...
  this.dangerZone = null;  // ← existiert NICHT in resetWorld()
```
`this.dangerZone` wird in `update()` (Zeile ~1669) lazy erstellt:
```js
if (!this.dangerZone) this.dangerZone = { x: WORLD_W * 0.35, w: WORLD_W * 0.3 };
```
Nach Level-3-Durchlauf und Neustart bleibt `dangerZone` im `game`-Objekt,
weil `resetWorld()` es NICHT auf `null` setzt. In Level 1 und 2 gibt es zwar keine
Aktivierungsbedingung (Level-3-Guard), aber sauber ist es nicht.
**Fix:** `this.dangerZone = null;` in `resetWorld()` hinzufügen.

---

### 🟠 BUG 7 — `Boss.hurt()`: Phase-Transition während TRANSITIONING-State nicht vollständig geblockt [Zeile ~746]
```js
hurt(dmg) {
  if (this.state === BOSS_STATE.TRANSITIONING) return;
  // ...
  this.checkPhase();
  if (this.hp <= 0) { this.hp = 0; this.die(); }
}
```
`checkPhase()` ruft `enterPhase()` auf, das `this.state = BOSS_STATE.TRANSITIONING` setzt.
Wenn zwei Hits in einem Frame ankommen (Dash + Stomp gleichzeitig via `aabb`), kann
`checkPhase()` doppelt feuern. Zweiter Hit → `state` ist schon TRANSITIONING → Guard greift.
Jedoch: Bei Boss-Death (`hp <= 0`) kann `die()` doppelt aufgerufen werden, wenn
gleichzeitig ein Projektil und ein Dash treffen.
**Fix:** Guard in `die()` hinzufügen (analog zu `Player.die()`):
```js
die() {
  if (this.dead) return;  // FIX: prevent double-die
  this.dead = true;
  // ... rest
}
```

---

### 🟠 BUG 8 — `completeLevel()` setzt `_deathTimer` zurück, aber `lose()` nicht [Zeile ~1731]
```js
completeLevel() {
  // ...
  this.state = STATE.SHOP;
  this._deathTimer = 0;   // ← gut, aber inkonsistent
}
lose() {
  if (this.state === STATE.GAMEOVER) return;
  this.state = STATE.GAMEOVER;
  this._deathTimer = 0;  // ← FEHLT hier!
```
**Fix:** `this._deathTimer = 0;` in `lose()` hinzufügen (bereits fast vorhanden durch Zeile 1747, check ob vorhanden — wenn ja, überspringen).

---

### 🟡 BUG 9 — Coyote-Time wird DOPPELT gesetzt [Zeile ~1229-1231]
```js
if (this.onGround) {
  this.coyote = 150;      // ← gesetzt wenn am Boden
} else if (wasGround) {
  this.coyote = 150;      // ← gesetzt wenn gerade vom Boden weg
}
```
Wenn der Spieler auf dem Boden ist, wird `coyote = 150` gesetzt, dann wird im selben
Frame `coyote -= dt` subtrahiert (Zeile ~1175). Das ist korrekt. ABER: `coyote` wird
auch gesetzt wenn `this.onGround === true` — Coyote-Time soll nur greifen wenn man
GERADE die Plattform verlassen hat, nicht während man drauf steht.
Das ist eigentlich ein Design-Bug aber kein Crash. Das `else if (wasGround)` ist der
einzig sinnvolle Coyote-Auslöser. Der `if (this.onGround)` Block setzt coyote unnötig,
hat aber keinen negativen Effekt da `canJumpGround = this.onGround || this.coyote > 0`
sowieso durch `this.onGround` abgedeckt ist.
**Status:** Kein kritischer Bug, kein Fix notwendig — dokumentieren als akzeptabel.

---

### 🟡 BUG 10 — Background-Stalaktiten in Level 2 nutzen `rand()` (live, non-deterministic) [Zeile ~1810]
```js
ctx.lineTo(x + 40, rand(60, 120));
```
`rand()` wird direkt in `drawBackground()` aufgerufen — jedes Frame eine neue Zufallszahl.
Stalaktiten flackern mit 60fps zwischen verschiedenen Höhen.
**Fix:** Stalaktit-Höhen beim `buildLevel2()` vorab berechnen und in `this.stalactites` speichern,
im Draw nur noch aus dem Array lesen.

---

### 🟡 BUG 11 — `Audio.buy()` spielt bei JEDEM Button-Klick, auch bei nicht-kaufbaren Items [Zeile ~2157]
```js
onButton(id) {
  safe(() => Audio.buy(), "btn");  // ← immer, auch wenn kauf fehlschlägt
```
Wenn ein Upgrade nicht leistbar ist, spielt trotzdem der Kauf-Sound.
**Fix:** `Audio.buy()` nur aufrufen wenn die Aktion tatsächlich etwas tut:
- Beim `buy:`-Zweig nur wenn Kauf erfolgreich
- Für Navigation-Buttons einen neutralen Click-Sound oder keinen

---

### ⚪ BUG 12 — `Coin.collide()` prüft Platform-Collision nur bis `p.y + p.h + 30` [Zeile ~385]
```js
this.y + this.r > p.y && this.y + this.r < p.y + p.h + 30
```
`+ 30` ist ein Magic-Number-Workaround um schnell fallende Münzen zu fangen.
Bei sehr schnell fallenden Münzen (hohe dt) kann der Check fehlschlagen.
**Fix:** Statt `< p.y + p.h + 30` besser: Prüfen ob Münze in letztem Frame über Platform war
(prevY-basiert) oder `+ 30` auf `+ this.vy * dt * 2` anpassen.
Akzeptable Vereinfachung: Wert auf `+ 50` erhöhen für mehr Toleranz.

---

## BROWSER-TEST PROTOKOLL (Claude in Chrome)

Führe nach allen Code-Fixes diese Sequenz aus:

```
1. navigate("file:///[ABSOLUTER-PFAD]/el-pollo-loco-ultimate.html")
2. Öffne DevTools Console → Errors notieren
3. Klick START → Spiel beginnt
4. LEVEL 1:
   - Laufe von links nach rechts (A/D oder Pfeiltasten)
   - Springe auf alle 5 Plattformen
   - Sammle mindestens 10 Münzen
   - Besiege 2 Hühner (Stomp von oben)
   - Besiege 1 Huhn (Dash)
   - Laufe zur Zielflagge rechts → Shop öffnet
5. SHOP:
   - Kaufe "Doppelsprung" wenn Münzen reichen
   - Klick "NÄCHSTES LEVEL"
6. LEVEL 2:
   - Überquere die Lavapools über die Plattformen
   - Weiche Lavablasen aus
   - Falle einmal in Lava → Spieler stirbt
   - Nach Tod: warte auf GAMEOVER-Screen
7. GAMEOVER:
   - Klick "NOCHMAL" → Level 1 startet neu
   - Prüfe: kein State-Bleed (Score = 0, Leben = 3)
8. Zweiten Durchlauf bis Level 3:
   - Endboss erscheint
   - Springe auf Endboss (Stomp) → HP-Bar sinkt
   - Phase 2 Transition: Boss wird rot, "¡ESTOY ENOJADO!"
   - Phase 3 Transition: Boss wächst, "¡AHORA MUERES!"
   - Besiege Endboss → VICTORY-Screen
9. VICTORY:
   - Klick "MENÜ" → Hauptmenü
   - Prüfe Highscore angezeigt
10. Console: ZERO Errors, ZERO Warnings
```

Screenshot nach jedem State-Wechsel. Prüfe DevTools Memory-Tab auf Leaks.

---

## NACH DEM FIX — NUR DIESES FORMAT AUSGEBEN

```
══════════════════════════════════════════════
 EL POLLO LOCO ULTIMATE — BUG FIX ABGESCHLOSSEN
══════════════════════════════════════════════

FIXES [N]
─────────
Zeile | Kategorie | Problem → Fix

BROWSER-TEST
────────────
States getestet: MENU / PLAYING / PAUSED / SHOP / GAMEOVER / VICTORY
Console-Errors: 0
Console-Warnings: 0
Boss-Phasen: 1→2→3→TOD ✓
Neustart sauber: ✓

STATUS: RELEASE-READY ✓
```

Nichts anderes. Kein anderer Output.

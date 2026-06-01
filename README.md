# El Pollo Loco ULTIMATE

Ein eigenständiges HTML5-Canvas-Jump-’n’-Run in **einer einzigen Datei** – kein Framework, kein Build, keine externen Assets. Grafik und Sound werden vollständig prozedural erzeugt.

▶️ **Spielen:** Einfach `index.html` (bzw. `el-pollo-loco-ultimate.html`) im Browser öffnen.

## Steuerung
- **Bewegen:** `A` / `D` oder Pfeiltasten ← →
- **Springen:** `W` / `↑` / `Leertaste`
- **Dash:** `Shift`
- **Pause:** `P`
- **Touch:** On-Screen-Buttons (Multi-Touch), Maus-Klick = Springen

## Features
- `requestAnimationFrame`-GameLoop mit deltaTime-Normalisierung (Cap 50 ms)
- Zentraler GameState-Manager (MENU / PLAYING / PAUSED / SHOP / GAMEOVER / VICTORY)
- Responsive 16:9-Welt mit Letterboxing + `devicePixelRatio` (scharf auf Retina)
- Münz-System: 5 Münztypen, Combo-Streaks, Magnet, Gravitation/Bounce
- 3 Level (El Rancho, La Cueva del Fuego, Arena del Diablo) + Upgrade-Shop
- 3-Phasen-Endboss „El Diablo Pollo" mit State-Machine und Wut-Meter
- Prozedurales Web-Audio, Object-Pool-Partikelsystem
- Highscore & Gesamtmünzen via `localStorage`

## Technik
Reines JavaScript / HTML5 Canvas. Die gesamte Logik nutzt eine feste virtuelle
Welt (1280×720) mit globalem Scale+Offset pro Frame – das sorgt für stabile
Physik bei voller Responsiveness.

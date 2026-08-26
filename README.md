# AYTO Tracker

Kleine Web-Anwendung, um eine Staffel "Are You The One?" mitzuverfolgen und zu sehen, welches Paar ein Perfect Match sein könnte. Gebaut für Nicos Schwester.

Spec: [`docs/superpowers/specs/2026-08-26-ayto-tracker-design.md`](../../docs/superpowers/specs/2026-08-26-ayto-tracker-design.md)

## Aufbau

```
index.html            die komplette Anwendung, eine Datei, kein Build
test/solver.test.mjs  Tests für die Rechenlogik
```

Der Rechenteil steht in `index.html` zwischen den Markern `// solver:start` und `// solver:end` in einem `<script type="text/plain">`. Zur Laufzeit wird daraus ein Web Worker gebaut, im Test wird derselbe Abschnitt herausgeschnitten und direkt ausgeführt. So bleibt es eine einzige Datei und ist trotzdem testbar.

## Benutzung

Datei im Browser öffnen oder statisch hosten. Alle Daten liegen im `localStorage` des Geräts, es gibt kein Backend.

1. **Kandidaten** anlegen, Männer und Frauen. Die Gruppen dürfen sich um höchstens eine Person unterscheiden, dann gibt es genau einen Doppelmatch.
2. Nach jeder Folge **Matching Night** eintragen: wer saß bei wem, wie viele Lichter gingen an.
3. **Match Box** eintragen, wenn ein Paar geprüft wurde.
4. Die Tabelle zeigt pro Paar, in wie viel Prozent der noch möglichen Konstellationen es ein Perfect Match ist. 100 heißt sicher, 0 heißt ausgeschlossen.

Unter **Daten** kann der Stand als Datei gesichert und wieder eingelesen werden.

## Entwicklung

```bash
node test/solver.test.mjs        # Tests
python3 -m http.server 8765      # lokal ausliefern
```

Die Anwendung muss über HTTP laufen, nicht über `file://`, sonst blockiert der Browser den Worker.

## Rechenlogik

Eine Lösung ordnet jede Person der größeren Gruppe genau einer Person der kleineren zu, wobei bei ungleicher Gruppengröße genau eine Person doppelt vorkommt. Gültig ist sie, wenn sie zu jeder Match Box passt und für jede Matching Night exakt die angegebene Lichterzahl trifft. Der Prozentwert einer Zelle ist der Anteil der gültigen Lösungen, die dieses Paar enthalten.

Bei 10 gegen 11 gibt es rund 200 Millionen Konstellationen. Exaktes Zählen und Stichprobe versagen an entgegengesetzten Stellen, deshalb laufen sie nacheinander: erst ein kurzer Zähllauf, bei Bedarf eine Stichprobe, und wenn die zu wenige Treffer liefert ein langer Zähllauf. Über simulierte Staffeln von Night 1 bis 8 ist das Ergebnis immer entweder exakt oder mit mindestens 20.000 Stichproben belegt.

## Failure Log

_Format: Datum | Kontext | Fehler | Learning_

- 2026-08-26 | Solver-Budget | Ein festes Knotenbudget von 40 Millionen ließ die Rechnung ausgerechnet mitten in der Staffel (Night 5 bis 7) auf Stichprobe zurückfallen, wo diese nur noch 1 bis 75 Treffer landete. Die Prozente wären Rauschen gewesen, ohne dass es jemand gemerkt hätte. | Nicht auf ein Verfahren verlassen, wenn zwei sich ergänzen. Gemessen statt geschätzt: erst eine simulierte Staffel über alle Phasen durchrechnen, dann die Schwellen setzen. Der Test prüft jetzt die Zusicherung selbst (exakt oder mindestens 20.000 Stichproben), nicht mehr nur die Laufzeit.

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

1. **Kandidaten** anlegen, Männer und Frauen. Die Gruppen dürfen sich um höchstens eine Person unterscheiden. Tun sie das, erscheint der Format-Umschalter:
   - **Eine geht leer aus:** die überzählige Person hat gar kein Perfect Match und sitzt in jeder Matching Night allein. Standard, so läuft die aktuelle Staffel mit 21 Kandidaten.
   - **Ein Doppelmatch:** jemand aus der kleineren Gruppe hat zwei Matches und sitzt doppelt.
2. Nach jeder Folge **Matching Night** eintragen: wer saß bei wem, wie viele Lichter gingen an.
3. **Match Box** eintragen, wenn ein Paar geprüft wurde.
4. Die Tabelle zeigt pro Paar, in wie viel Prozent der noch möglichen Konstellationen es ein Perfect Match ist. 100 heißt sicher, 0 heißt ausgeschlossen. Im Format ohne Match steht rechts angeheftet die Spalte **kein Match**: wie wahrscheinlich diese Frau diejenige ohne Perfect Match ist.

Unter **Daten** kann der Stand als Datei gesichert und wieder eingelesen werden.

## Entwicklung

```bash
node test/solver.test.mjs        # Tests der Rechenlogik
python3 -m http.server 8765      # lokal ausliefern
./deploy.sh "was geändert wurde" # testen, dann live
```

Bei jeder Änderung an Dialogen oder am Layout zusätzlich gegen eine zweite Browser-Engine prüfen, nicht nur gegen Chrome. Chrome verzeiht Layout-Fehler, die auf anderen Geräten den halben Dialog verschwinden lassen.

Die Anwendung muss über HTTP laufen, nicht über `file://`, sonst blockiert der Browser den Worker.

## Rechenlogik

Eine Lösung ordnet jede Person der größeren Gruppe genau einer Person der kleineren zu, wobei bei ungleicher Gruppengröße genau eine Person doppelt vorkommt. Gültig ist sie, wenn sie zu jeder Match Box passt und für jede Matching Night exakt die angegebene Lichterzahl trifft. Der Prozentwert einer Zelle ist der Anteil der gültigen Lösungen, die dieses Paar enthalten.

Beide Formate laufen über denselben Suchcode: jeder Platz nimmt entweder eine noch freie Person der kleineren Gruppe oder verbraucht den einen Sonderplatz, der je nach Format "niemand" oder "jemand ein zweites Mal" bedeutet. Bei 10 gegen 11 sind das rund 40 Millionen Konstellationen im Format ohne Match und rund 200 Millionen mit Doppelmatch. Exaktes Zählen und Stichprobe versagen an entgegengesetzten Stellen, deshalb laufen sie nacheinander: erst ein kurzer Zähllauf, bei Bedarf eine Stichprobe, und wenn die zu wenige Treffer liefert ein langer Zähllauf. Über simulierte Staffeln von Night 1 bis 8 ist das Ergebnis immer entweder exakt oder mit mindestens 20.000 Stichproben belegt.

## Failure Log

_Format: Datum | Kontext | Fehler | Learning_

- 2026-08-26 | Falsches Spielformat angenommen | Die erste Fassung kannte nur den Doppelmatch: bei 11 Frauen und 10 Männern sitzt ein Mann mit zwei Frauen, alle elf sitzen. Die laufende Staffel funktioniert anders. Von 21 Kandidaten geht **eine Person komplett leer aus**, sie hat gar kein Perfect Match und sitzt in jeder Matching Night allein. Die Rechnung war damit vom ersten Tag an sauber falsch, ohne dass irgendetwas kaputt aussah: die Tabelle zeigte plausible Prozente für ein Spiel, das so nicht gespielt wird. | Bei einer Anwendung, die eine reale Sendung abbildet, sind die Spielregeln die Anforderung, nicht die Kulisse. Vor dem Bauen die Regeln der konkreten Staffel belegen, nicht die des Formats im Allgemeinen, und im Zweifel beide Varianten als Einstellung anbieten statt eine zu raten. Ein falsches Modell fällt in keinem Test auf, der nur gegen dasselbe falsche Modell prüft. Die Brute-Force-Gegenprobe lief die ganze Zeit grün, weil sie dieselbe Annahme teilte.
- 2026-08-26 | Dialoge klappten auf Nicos Handy zusammen | `.dlg-body { flex: 1 }` in einem Dialog, dessen Höhe aus dem Inhalt kommt. `flex: 1` heißt Basisgröße 0, also trägt der Inhalt nichts zur Höhe des Dialogs bei, und danach gibt es keinen freien Platz zum Hineinwachsen. Chrome auf dem Mac löste das gnädig auf und zeigte alles korrekt, auf Nicos Gerät blieben nur die beiden Überschriften übrig. Zweimal ausgeliefert, ohne dass es auffiel. | **Eine Engine ist keine Prüfung.** Chrome allein bestätigt nur, wie Chrome rät. Seitdem läuft `webkit-check.mjs` gegen eine zweite Engine über alle Dialoge und Gerätegrößen, und zwar mit und ohne Daten, weil der leere Erststart ein eigener Zustand ist. Vorher am alten Stand nachweisen, dass der Test den Fehler überhaupt findet, sonst prüft er nichts. Inhaltlich: keine Höhe aus Flex-Basis 0 ableiten. Der Dialog ist jetzt schlichtes Blocklayout mit `max-height` und `overflow-y: auto`, Kopf und Fuß per `sticky`. Das kann nicht kollabieren, und wo `sticky` fehlt, scrollt es nur, statt zu verschwinden.
- 2026-08-26 | Ausliefern trotz roter Tests | `deploy.sh` stoppte, weil die 4-Nights-Rechnung 8,2 Sekunden statt der erlaubten 8 brauchte. Der Reflex wäre gewesen, die Grenze hochzusetzen und weiterzumachen. | Die Grenze lag zufällig genau auf dem echten Wert, das war kein Ausreißer, sondern ein Hinweis: auf einem Handy dauert dieselbe Rechnung noch länger. Grenze auf 15 Sekunden gesetzt und ausdrücklich als Rauchmelder gekennzeichnet, die harte Zusicherung bleibt "exakt oder mindestens 20.000 Stichproben". Zusätzlich zeigt die Oberfläche nach zwei Sekunden "Rechnet noch", damit eine lange Rechnung nicht wie ein Absturz aussieht.
- 2026-08-26 | Kandidaten-Dialog am Handy | Die Namenslisten standen auf schmalen Schirmen untereinander. Auf einem iPhone SE lagen dadurch 604 Pixel und damit die komplette Frauenliste unsichtbar unter der Falz, ohne jeden Hinweis, dass da noch etwas kommt. Technisch war alles korrekt, der Dialog war scrollbar. | "Es scrollt ja" ist kein Beleg. Geometrie an echten Geraetegroessen messen, nicht nur einen Screenshot ansehen: Dialoghoehe, verdeckte Pixel und Sichtbarkeit des Fusses. Zwei Spalten nebeneinander halbieren die Hoehe, das Eingabefeld gehoert ueber die Liste, nicht darunter. Zusaetzlich `dvh` statt `vh`, weil Safari `vh` gegen die Flaeche ohne Adressleiste misst.
- 2026-08-26 | Verifikation mit Chrome | Zwei Messlaeufe hintereinander lieferten identische Zahlen, obwohl die Datei dazwischen geaendert wurde. Chrome hatte die alte Fassung aus dem Cache bedient, die Messung bestaetigte damit den alten Stand. | In jedem Pruefskript `Network.setCacheDisabled` setzen und das Browserprofil vor dem Lauf loeschen. Gleiche Zahlen nach einer Aenderung sind ein Warnsignal, kein Erfolg.
- 2026-08-26 | Textersetzung per Skript | Ein Stapel Ersetzungen lief mit einem einzigen `assert s != before` durch. Eine davon traf nie, weil in der Zeichenkette noch "gross" statt "gross mit Eszett" stand. Der Lauf meldete Erfolg, der Text blieb falsch und fiel erst im Screenshot auf. | Jede Ersetzung einzeln pruefen (`assert s.count(old) == 1`), nicht den Stapel als Ganzes. Ein Sammel-Assert verdeckt genau die Ersetzung, die danebengeht.
- 2026-08-26 | Solver-Budget | Ein festes Knotenbudget von 40 Millionen ließ die Rechnung ausgerechnet mitten in der Staffel (Night 5 bis 7) auf Stichprobe zurückfallen, wo diese nur noch 1 bis 75 Treffer landete. Die Prozente wären Rauschen gewesen, ohne dass es jemand gemerkt hätte. | Nicht auf ein Verfahren verlassen, wenn zwei sich ergänzen. Gemessen statt geschätzt: erst eine simulierte Staffel über alle Phasen durchrechnen, dann die Schwellen setzen. Der Test prüft jetzt die Zusicherung selbst (exakt oder mindestens 20.000 Stichproben), nicht mehr nur die Laufzeit.

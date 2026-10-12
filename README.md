# Lückenfüllung in Meerestemperaturdaten mit einem CNN

In diesem Projekt wird untersucht, ob ein kleines Convolutional Neural Network (CNN) künstlich versteckte Werte eines Datensatzes besser rekonstruieren kann als drei weitere Methoden, die als Baselines herangezogen werden. Als Grundlage dient hierbei der Datensatz des NOAA NCEI, eingeschränkt auf die Nordsee und westliche Ostsee, im Zeitraum 2018–2022, welcher tägliche Meeresoberflächentemperaturen enthält. 

Das CNN, welches in diesem Projekt konstruiert wird, nutzt ein Nachbarschaftsmittel als Startwert und lernt, diesen Wert zu verbessern. Auf dem Testjahr 2022 erreicht es einen RMSE von 0,131 °C, während das Nachbarschaftsmittel ohne CNN 0,194 °C erzielt. Ein Ansatz aus der Klimatologie kommt auf einen RMSE von 1,056 °C. Ein einfacheres CNN ohne diesen Startwert schnitt mit 0,292 °C dagegen schlechter ab als das Nachbarschaftsmittel, aber immer noch besser als die Klimatologie. Dass die Daten selbst interpoliert und die Lücken künstlich und gleichverteilt sind, stellt eine Einschränkung dieses Projekts dar, sodass sich die Ergebnisse nur bedingt auf echte Satellitenlücken übertragen lassen. 

## Hintergrund

Erdbeobachtungsdaten sind selten vollständig. Mal fallen Sensoren aus, oder Wolken verdecken die Sicht. Ein sinnvoller Umgang mit fehlenden Daten ist daher unabdingbar. Mit dem Problem, fehlende Daten sinnvoll zu rekonstruieren, befassen wir uns daher in diesem Projekt.

## Daten

Grundlage dieses Projekts ist der Datensatz "SST, Daily Optimum Interpolation (OI), AVHRR Only, Version 2.1, Final, Global, 0.25°, 1981-present, Lon+/-180" des "NOAA NCEI". Dies ist ein vollständiger Datensatz mit täglichen Messungen von Meerestemperaturen, die zu einem lückenlosen Gitter interpoliert worden sind. Wir werden künstlich Daten löschen, um diese mithilfe eines neuronalen Netzes zu rekonstruieren. Bei den künstlichen Lücken handelt es sich also nicht um echte Satellitenlücken und die ursprünglichen Daten sind interpoliert. Da die korrekten Daten vorliegen, können wir diese nutzen, um den Erfolg unseres Netzes zu messen. 

## Methode

### Versuchsaufbau

Da die Daten vollständig vorliegen, konstruieren wir eine Maske, mit der wir die Werte künstlich verstecken, und messen den Erfolg unserer Rekonstruktionen mithilfe der wahren, versteckten Werte.

- **Maske:** Für jeden Tag werden etwa 30 % der Gitterpunkte im Meer zufällig versteckt (Seed 42, Lücken sind gleichverteilt). Diese Maske wird einmal für den ganzen Zeitraum erzeugt, sodass alle Methoden dieselben Lücken nutzen und deren Werte vorhersagen. Ihnen stehen also dieselben sichtbaren Punkte zur Verfügung.

- **Zeitlicher Split:**

| Zeitraum | Verwendung | Tage |
|---|---|---|
| 2018–2020 | Training (29.02.2020 entfernt) | 1095 |
| 2021 | Validierung | 363 |
| 2022 | Test (nur einmal am Ende ausgewertet) | 364 |

- Die Daten zu den Tagen 23.3.2021, 7.9.2021 und 19.10.2022 fehlen im ursprünglichen Datensatz und wurden nicht ersetzt.

- **Fehlermaß:** RMSE in °C, nur über die versteckten Meerespunkte ausgewertet.

### Baselines

1. **Konstanter Wert:** alle Lücken werden mit dem Mittelwert der Trainingsdaten aufgefüllt.
2. **Klimatologie:** für jeden Kalendertag und Gitterpunkt wird der Mittelwert über die drei Trainingsjahre verwendet. 
3. **Nachbarschaftsmittel:** Jeder versteckte Punkt wird durch den Mittelwert seiner direkten sichtbaren Nachbarn im am selben Tag geschätzt. Punkte ganz ohne sichtbaren Nachbarn werden mit der Klimatologie aufgefüllt.

### Netze

In diesem Projekt wurden zwei Netze erstellt, wobei das Netz v2 eine Weiterentwicklung des Netzes v1 ist. Beide Netze bestehen aus vier Convolutional Layers mit ReLU-Aktivierungsfunktionen dazwischen. Die letzte Schicht hat keine ReLU-Aktivierungsfunktion. Die Kanäle sind 3 (im Fall v1) oder 4 (im Fall v2), 32, 32, 32, 1. Daraus ergeben sich knapp 20.000 lernbare Parameter. Die Eingabe wird mithilfe des Mittelwerts (11,03 °C) und der Standardabweichung (4,72 °C) der Trainingsdaten normalisiert. Fehlende Werte (NaN) werden durch 0 ersetzt. Da wir mit normalisierten Werten arbeiten, entspricht dies dem Durchschnittswert.

- **CNN v1:** drei Eingangskanäle: (1) normalisierte Temperatur mit NaN-Werten, die mit 0 aufgefüllt wurden, (2) Angabe, ob Gitterpunkte sichtbar sind oder künstlich versteckt, (3) Angabe, ob ein Gitterpunkt im Meer oder an Land liegt. Mithilfe der Kanäle 2 und 3 kann das Netz einen Wert =0 korrekt einordnen. Das Netz umfasst 19.681 lernbare Parameter.
- **CNN v2:** zusätzlich als vierter Kanal wird das Nachbarschaftsmittel (normalisiert) an das Netz übergeben. Dadurch startet das Netz bereits mit der Baseline und lernt nur noch weitere Korrekturen, die schließlich ausgegeben werden. Das Netz umfasst 19.969 lernbare Parameter.

### Angaben zum Training

- Optimizer: Adam
- Lernrate: 0,001, konstant für v1, angepasst durch Cosine-Annealing-Scheduler bei v2
- Batchgröße: 32
- Anzahl der Epochen: 30 bei v1, 40 bei v2
- Der Loss wird nur an den versteckten Meerespunkten gemessen

## Ergebnisse

RMSE in °C auf den versteckten Meerespunkten:

| Methode | Validierung 2021 | **Test 2022** | Test gegenüber Nachbarschaftsmittel |
|---|---|---|---|
| **CNN v2** | 0,108 | **0,131** | −32,5 % |
| Nachbarschaftsmittel | 0,157 | 0,194 | |
| CNN v1 | 0,282 | 0,292 | +50,5 % |
| Klimatologie | 1,079 | 1,056 | — |
| Konstanter Wert | 4,901 | 4,309 | — |

v2 schlägt also alle drei Baselines, v1 schlägt nur den Ansatz der Klimatologie und des konstanten Werts, verliert jedoch gegen das Nachbarschaftsmittel.

## Grenzen

- **Interpolierte Daten:** Da die Daten aus OISST selbst eine interpolierte Analyse darstellen, sind benachbarte Punkte sehr ähnlich. Die Rekonstruktion ist dadurch leichter als bei echten Beobachtungen.
- **Künstliche Lücken:** Die Lücken sind zufällig verteilt. Echte Satellitenlücken sind zusammenhängend und größer und entstehen z. B. durch Wolken, die die Sicht verdecken. Es ist davon auszugehen, dass die Ergebnisse bei solchen Lücken schlechter ausfallen würden.
- **Kleines Modell und kleiner Datensatz:** etwa 20 000 Parameter, drei Trainingsjahre, ein Seed pro Netz.
- **Keine Auswertung auf Extremen:** Wie die Methoden bei Hitzewellen abschneiden, wurde aus Zeitgründen nicht getestet.

## Ausblick

- **Auswertung auf ungewöhnlich warmen Tagen:** Fehler getrennt für ungewöhnlich warme Tage auswerten, um zu prüfen, ob die Fehler bei Hitzewellen höher sind als über das gesamte Jahr hinweg betrachtet.
- **Realistischere Lücken:** Eine Maske erstellen, bei der die Lücken nicht gleichverteilt sind, sondern zusammenhängend, um zu prüfen, ob der Fehler dann höher ausfallen würde.
- **Andere Architekturen:** komplexere Datensätze und/ oder Netzwerkarchitekturen anwenden.

## Quellen

- Datensatz: https://catalog.data.gov/dataset/sst-daily-optimum-interpolation-oi-avhrr-only-version-2-1-final-global-0-25-1981-prese-180

## Hinweis 

Das Projekt entstand im Selbststudium. Bei der Aufgabenstellung, der Erstellung einiger Codeabschnitte, Erklärungen und der Fehlersuche habe ich KI genutzt. 

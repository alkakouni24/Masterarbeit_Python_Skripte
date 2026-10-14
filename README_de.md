# Defekterkennung an Turbinenschaufeln – Master_Arbeit_Majd

Code aus der Masterarbeit zur YOLO-basierten Defekterkennung an Turbinenschaufeln, trainiert auf synthetischen, aus 3D-Modellen gerenderten Datensätzen.

## Reihenfolge der Pipeline

1. `view_capture_render.py` – Bilder und YOLO-Labels aus 3D-Modellen rendern
2. `inject_cooling_holes.py` – Kühlbohrungs-Labels in die gerenderten Labels einfügen
3. `remove_occluded_cooling_holes.py` – von Schäden verdeckte Kühlbohrungs-Labels entfernen
4. `split_build_gui.py` – leckagefreier Train/Val/Test-Split und YOLO-Datensatz-Zip
5. (Training mit Ultralytics YOLOv8 auf der erzeugten `data.yaml`, nicht enthalten)
6. `blade_inspection_app.py` – trainiertes Modell und Reparatur-/Ausschussregeln anwenden

## Dateien

### `view_capture_render.py`
Rendert jedes `.obj`-Modell aus zwei festen orthografischen Ansichten (`Yneg`, `Ypos`) auf weißem Hintergrund, in mehreren Maßstäben (Standard: 10 logarithmisch verteilte Schaufellängen von 82 mm bis 2500 mm) und vier Beleuchtungsvarianten. Die Defekt-Bounding-Box aus einer nTop-BB-Datei wird in den Bildraum projiziert und als YOLO-Label geschrieben.
- Dateiname der Ausgabe: `<Typ>_ID_<n>__<Ansicht>__<Länge>mm__<Beleuchtung>.png/.txt`
- Klassen-IDs stammen aus `class_map.json`.
- Vorab-Bericht über die Zuordnung von OBJ- zu BB-Dateien vor dem Rendern.
- Pause / Abbruch / Fortsetzen über ein Job-Manifest (`render_job_<Zeitstempel>.json`), Fortschrittsbalken mit Restzeit.
- Start: `python view_capture_render.py` (GUI) oder per Kommandozeile mit `--obj`, `--bb`/`--bb-folder`, `--out`; Fortsetzen mit `--resume <manifest.json>`.
- Benötigt: `numpy`, `trimesh`, `pyrender`, `Pillow`, optional `tqdm`.

### `inject_cooling_holes.py`
Kühlbohrungen werden nur auf 10 Referenzbildern (10 Maßstäbe x 2 Ansichten) manuell annotiert. Das Skript kopiert diese Boxen in jede Label-Datei mit demselben Schlüssel (Ansicht, Maßstab), unabhängig von Defektklasse, ID und Beleuchtung. Der Klassenindex aus dem CVAT-Export (`0`) wird auf den Datensatzindex (`4`) umgeschrieben. Dateien, die bereits Kühlbohrungen enthalten, werden übersprungen.
- Start: `python inject_cooling_holes.py --labels-dir <Labels> --reference-dir <Referenz-Labels> [--dry-run]`

### `remove_occluded_cooling_holes.py`
Entfernt Kühlbohrungs-Boxen (Klasse `4`), die unter einer verdeckenden Defekt-Box liegen (Standard: Klasse `2`, Erosion). Eine Box wird entfernt, wenn ihr Mittelpunkt in der verdeckenden Box liegt oder mindestens der angegebene Anteil ihrer Fläche überlappt (Standard 0,3).
- Start: `python remove_occluded_cooling_holes.py --labels-dir <Labels> [--occluding-classes 2 3] [--overlap-threshold 0.3] [--dry-run]`

### `split_build_gui.py`
GUI-Werkzeug, das den Datensatz-Split erzeugt und die YOLO-Zip in einem Lauf baut.
- Split auf Modell-Ebene (Stem), sodass Bilder desselben Defekts nie in verschiedenen Splits landen (keine Leckage).
- Stratifiziert nach Schadensart x Größe x relative Höhe x relative Sehnenposition, exakte globale Train/Val/Test-Anteile (Largest-Remainder-Verfahren).
- Negative (defektfreie) Bilder werden je Bild und Maßstab geteilt.
- Berichtet Kolmogorov-Smirnov-Abstände Train gegen Val/Test und schreibt ECDF-, Abdeckungs- und Typ-Abbildungen, CSV-/JSON-Zuordnungen und einen Vorab-Bericht.
- Baut `dataset/images|labels/{train,val,test}` und `data.yaml` aus `class_map.json`; bricht ab, wenn Klassen-IDs der Labels nicht zur Map passen.
- Start: `python split_build_gui.py`
- Benötigt: `numpy`, `pandas`, `scipy`, `matplotlib`, `tkinter`, optional `openpyxl` (DoE-Datei).

### `blade_inspection_app.py`
Desktop-Entscheidungshilfe (deutsche Oberfläche). Lädt ein Schaufelbild und eine trainierte YOLO-Gewichtsdatei, erkennt Defekte, führt optional nahe beieinanderliegende Boxen zusammen und wendet die regelbasierte Reparatur-/Ausschusslogik auf jeden Defekt an. Zeigt ein Gesamturteil, Karten je Defekt und exportiert ein annotiertes Bild mit Ergebnisbox und Legende.

Entscheidungsregeln in dieser Reihenfolge:

1. Kein Defekt erkannt: die Schaufel ist intakt.
2. Lage: Ein Defekt, der nicht innerhalb der oberen 10 % der Schaufellänge liegt, ist Ausschuss. Ein Defekt, der in dieses Band hineinreicht, zählt als innerhalb.
3. Fläche: Ein Defekt mit mindestens 30 % der Schaufelfläche ist Ausschuss.
4. Zone: Der Defekt wird Zone B (Innenbereich), Zone C (Rand entlang der Oberkante und beider Seitenkanten) oder beiden zugeordnet. Ein Defekt in beiden Zonen wird gegen den strengeren Grenzwert von Zone B geprüft.
5. Breite: Die horizontale Breite der Bounding-Box wird mit dem Grenzwert der Zone verglichen. Ein breiterer Defekt ist Ausschuss, sonst Reparatur. Grenzwerte: Zone B 1/32 Zoll; Zone C 5/32 Zoll für Stufen 1 bis 4 und 1/8 Zoll ab Stufe 5, umgerechnet in mm.

Das Gesamturteil ist Ausschuss, wenn mindestens ein Defekt Ausschuss ist, sonst Reparatur, wenn mindestens ein Defekt Reparatur ist, sonst intakt. Die Stufe ergibt sich aus der Schaufellänge (nächstliegende der 10 Render-Größen).

- Benötigt `rules_config.json` im selben Ordner (Zonenanteile, Breitengrenzen, Flächengrenze, Stufengrenze).
- Ohne Gewichte läuft die App im Testmodus mit Beispieldetektionen.
- Start: `python blade_inspection_app.py`
- Benötigt: `numpy`, `scipy`, `Pillow`, `ultralytics`, optional `opencv-python`, `tkinter`.

## Nicht enthalten

Datensätze, gerenderte Bilder, 3D-Modelle, nTop-BB-Dateien, trainierte Gewichte (`*.pt`) und `rules_config.json`.

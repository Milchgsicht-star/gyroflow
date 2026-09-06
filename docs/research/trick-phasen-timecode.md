# Research: Trick-Phasen mit korrektem Timecode — der Weg zu «100 % der Fälle» (Issue resolve-coach #47)

Stand 06.09.2026 · Anschluss an #47 (Schnittpunkte), #49 (Research flaches Video), #52 (Pose-Prototyp,
571 Urteile), #53 (Sterne-Bogen), #59 (Fahrer-Zuordnung). Wendelins Frage (06.09.2026):

> «Was gäbe es für Möglichkeiten, dass es 100 % der Fälle funktioniert, mit korrektem Timecode für
> Absprung, Railphase, Landung oder vielleicht sogar Grab in der Luft? Ziel wäre, dass ich mein ganzes
> Cliparchiv durchgehen könnte und Clips generieren kann, die ich dann nur noch durchgehen und mit
> 1–5 Sternen bewerten muss.»

Quellenlage: Primärquellen (Repos, Model-Cards, API-Docs, Paper) wo erreichbar; arXiv, CVF, MDPI, PMC,
ai.google.dev und huggingface.co waren aus der Sandbox gesperrt — dort stammen Zahlen aus dem
Such-Auszug der zitierten Seite und sind mit **[Auszug]** markiert. Vor einer Budget- oder
Architektur-Entscheidung die markierten Stellen nachprüfen.

## Kurzantwort

**«100 % automatisch und bildgenau» ist weder erreichbar noch messbar** — auch Menschen setzen die
Marke nur auf ±1–2 Bilder (Abschnitt 1). Erreichbar und messbar ist etwas anderes, und das genügt für
das Ziel «Archiv durchgehen und Sterne vergeben»:

1. **100 % der Tricks kommen an den Bogen** (Recall), weil der Detektor bewusst zu viel meldet und
   die Restzeit jedes Clips als Rest-Fenster ebenfalls vorgelegt wird — ein Fehlalarm kostet einen
   Tastendruck, ein verpasster Trick wird nie gesehen (Abschnitt 9).
2. **Jede Marke sitzt innerhalb ±2 Bilder** (Landung, Rail-an: harte Aufschläge) bzw. **±4 Bilder**
   (Absprung, Rail-ab, Grab-Beginn: weiche Übergänge) an der Marke eines Menschen, der Bild für Bild
   schaltet — das ist die menschliche Untergrenze aus Fussball- und Gang-Studien (Abschnitt 1).
3. **Was daneben liegt, korrigiert Wendelin am Bogen mit einem Tastendruck je Bild**, und jede
   Korrektur wird Lernmaterial für den nächsten Lauf (Abschnitt 9).

Der Weg dahin hat vier technische Hebel, alle lokal und ohne neue Kamera oder Sensoren:

| Hebel | Was | Wirkung | Aufwand |
|---|---|---|---|
| **A Phasenmodell mit Grammatik** statt zwei unabhängiger Ereignis-Köpfe | je Bild ein Zustand (Anfahrt · Flug · Rail · Flug · Ausfahrt, dazu Grab an/aus), dekodiert mit Viterbi unter Reihenfolge- und Dauer-Regeln aus den 123 Hand-Tricks | liefert Rail-an/Rail-ab ohne Ton, beseitigt «zu kurz» (Rail-ab als Landung) und Ereignisse in falscher Reihenfolge | 1–2 Tage |
| **B Kurve als Kamera-Modell** statt Optical-Flow-Kompensation | die Reframe-Kurve (Pan/Tilt/FOV je Bild) ist die bekannte virtuelle Kamera auf der stabilisierten Kugel → Hüfte/Knöchel/Board analytisch in Welt-Winkel umrechnen | ersetzt die 5–20 % Bilder ohne Hintergrund-Schätzung; Flugparabel wird physikalisch sauber | ½–1 Tag |
| **C Bessere Geometrie je Bild** | yolo26-pose (+4 AP gegenüber yolo11s), Fuss-Keypoints (DWPose/RTMW, Apache) für die Board-Linie, Board/Rail-Boxen via YOLOE-26, 50 statt 20 Bilder/s in den Trick-Fenstern | Rail-Kontakt und Grab werden geometrisch definierbar (Fuss–Board–Rail, Hand–Board) | 2–3 Tage |
| **D Ton als Feinjustierung** | gelernter Onset-Kopf (EfficientAT/PANNs, 10-ms-Raster) auf den Hand-Marken, nur zum Nachrücken der Bild-Marke innerhalb ±3 Bilder, Laufzeit des Schalls abziehen | Landung/Rail-an auf ±1 Bild, wo hörbar | 1 Tag |

Dazu die **Schleife** (Sterne-Bogen als Prüf- und Korrektur-Werkzeug mit Rest-Fenstern, Rücklauf in
den Kopf) und für **künftige Drehtage** wahlweise ein IMU am Board (5–8 ms Genauigkeit, liefert
Wahrheit gratis). VLMs (Gemini, Claude, lokal Qwen3-VL/TimeLens) bleiben Sekunden-grob und dienen nur
für Trick-Name und Grab-Art (Abschnitt 7).

## 0. Stand und Lücke

Aus #47/#52/#53 (Terneuzen, 171 Clips, 530 Hand-Marken, 571 Urteile):

| Ereignis | heute | Quelle des Signals | Lücke |
|---|---|---|---|
| Absprung | Pose-Kopf: Recall 86 %, Präzision 91 %, Median 0,09 s (91 % ≤ 0,2 s) | Hüfte/Knöchel gegen affin geschätzten Hintergrund, 20 Bilder/s | flache Rail-Tricks ohne Hüft-Anstieg; 5–20 % Bilder ohne Hintergrund-Schätzung |
| Landung | Pose-Kopf 90 % / 61 %, Median 0,06 s; als Paar 94 % Präzision | wie oben | 16× «zu kurz»: Rail-ab wird als Landung genommen (Dauer 0,70 statt 1,70 s) |
| Rail-an | nur Ton: 82–88 % der Marken haben einen Impuls ±0,15 s | breitbandiger Impuls | Fremd-Impulse (Wendelins Brett, Stimmen); keine Bild-Definition |
| Rail-ab | nur Ton: 38–46 % | schwacher Impuls | im Bild nicht modelliert |
| Grab | — | — | nicht angefangen |
| Trick als Ganzes | Apex UND Pose 424/426 = 100 % stimmt; nur Pose 84 %; nur Apex 27 %; ohne «nur Apex» + Randregel 93 % behalten bei 99 % Präzision | | 7 % der bestätigten Tricks fallen durch den Filter; 6 Fehlurteile in 571 |

Die Zeitauflösung ist 50 ms (20 Bilder/s), die Hand-Marken sind bildgenau (20 ms). Die Trick-Dauern
aus den Hand-Marken (Median, 10–90 %): Absprung→Rail-an 0,47 s (0,34–0,69), Rail-an→Rail-ab 0,60 s
(0,38–1,07), Rail-ab→Landung 0,55 s (0,28–1,00), Absprung→Landung 1,71 s (1,39–2,27), reiner Sprung
1,39 s — das sind die Dauer-Regeln für Hebel A.

## 1. Was «100 % korrekt» heissen kann — die menschliche Untergrenze

- **Fussball, drei Expert:innen, 2 134 Ereignisse bei 25 Bilder/s** (ELASTIC, 2025 **[Auszug]**,
  https://arxiv.org/abs/2508.09238): über 90 % bildgenau gleich, mittlere Abweichung unter einem Bild
  (0,04 s); bei ~99 % liegen mindestens zwei innerhalb 2 Bildern (0,08 s) — erreicht, weil auf jedem
  Ereignis angehalten und Bild für Bild geschaltet wurde.
- **Gang-Ereignisse aus Video/Mocap** (SEM 0,4–1,9 Bilder, https://link.springer.com/article/10.1186/s12984-025-01544-9
  **[Auszug]**; Übereinstimmungsgrenzen Fersen-Aufsatz 24 ms, Zehen-Abstoss 16 ms, Fersen-Abheben 72 ms,
  Fuss flach 80 ms, https://www.ncbi.nlm.nih.gov/pmc/articles/PMC9773262/ **[Auszug]**): **Aufschläge
  werden auf ~1 Bild gleich gesetzt, weiche Übergänge streuen 70–80 ms** — das ist keine Modellfrage,
  sondern eine Definitionsfrage. Übersetzt: Landung und Rail-an (Aufschlag) ±1–2 Bilder; Absprung
  (Brett löst sich weich vom Kicker), Rail-ab und Grab-Beginn ±3–4 Bilder.
- **Eiskunstlauf-Sprünge mit 3D-Pose** (VIFSS auf FS-Jump3D, https://arxiv.org/abs/2508.10281
  **[Auszug]**): Phasen-Segmentierung Genauigkeit 89,9 %, F1@50 94,7 %, aber **F1@90 nur 51,7 %** —
  selbst mit 12 Kameras und 3D-Pose sind 1–2 Bilder Grenzfehler die Regel, nicht die Ausnahme.
- **IMU am Fahrer** (Halfpipe-U-Net, Sensors 2024, 626 Tricks, https://doi.org/10.3390/s24216773
  **[Auszug]**): Absprung Median 0,008 s, Landung 0,005 s — die physikalische Messlatte, ~10× besser als
  Video heute, aber nur mit Sensor am Körper (Abschnitt 10).
- **Konsequenz für die Messung:** Die Mess-Skripte (`--lernen`, `pruefbogen.py --auswerten`) sollten
  wie «Precise Event Spotting» (E2E-Spot, ECCV 2022, https://github.com/jhong93/spot; T-DEED,
  https://github.com/arturxe2/t-deed) je Ereignis-Art **mAP bei Toleranz 1, 2 und 4 Bildern** ausgeben,
  getrennt nach Aufschlag- und Übergangs-Ereignissen, statt eines gemeinsamen ±0,5-s-Treffers. Und der
  Bogen sollte protokollieren, wie oft Wendelin um wie viele Bilder nachrückt — das ist die einzige
  Präzisionsmessung, die zählt.

## 2. Hebel A — ein Phasenmodell mit Grammatik statt zwei Ereignis-Köpfen

**Befund:** Die heutigen Köpfe (Absprung, Landung) sind unabhängig; Rail-an/-ab kommen aus dem Ton;
die Paarung ist eine Nachregel. Das erzeugt die bekannten Fehler: Rail-ab als Landung («zu kurz»),
Landung ohne Absprung, Absprung in der Anfahrt. Die Literatur zur zeitlichen Aktions-Segmentierung
löst genau das mit **Zuständen je Bild plus eingeschränkter Dekodierung**:

- **Constraint-Aware Decoding** (Mai 2026, https://arxiv.org/html/2605.10149 **[Auszug]**): ein
  angepasster Viterbi, der Übergangs-Sicherheiten, erlaubte Grenzen und Dauer-Prioren je Klasse aus den
  Annotationen einbaut — **zur Inferenz, ohne Neutraining**, mit den grössten Gewinnen in Edit/F1 (also
  Reihenfolge und Kohärenz). Vorläufer: Set-Constrained Viterbi (CVPR 2020,
  https://arxiv.org/abs/2002.11925), Anchor-Constrained Viterbi (CVPR 2021, https://arxiv.org/abs/2104.02113).
- **Skelett-basierte Segmentierung mit wenig Daten:** Punkt-Supervision (ein markiertes Bild je Segment,
  https://arxiv.org/html/2603.06201v2 **[Auszug]**) erreicht auf PKU-MMD/MCFS voll-überwachte Werte — die
  530 Hand-Marken *sind* Punkt-Labels. DeST (IJCV 2026, https://github.com/lyhisme/DeST, CC BY-NC) als
  Referenz-Architektur für Skelett-Sequenzen; Few-Shot-Skelett-TAS mit synthetischer Augmentierung
  (https://arxiv.org/pdf/2207.09925).
- **UMEG-Net** (AAAI 2026, https://arxiv.org/abs/2511.14186): ein Graph über Skelett **und
  Sportgerät-Keypoints** (bei uns: Board-Nase/-Tail), GCN plus Multi-Scale Temporal Shift, für
  wenig Labels gebaut — die passende Architektur, falls der Gradient-Boosting-Kopf an seine Grenze kommt.
- **Label-Rauschen:** «Towards Precise Action Spotting» (https://arxiv.org/pdf/2504.00149) ordnet Labels
  dynamisch zu, um ±1 Bild Streuung in Hand-Marken zu absorbieren.

**Bauplan:**

1. Zustände je Bild: `Anfahrt`, `Flug1`, `Rail`, `Flug2`, `Ausfahrt`; parallel ein Bit `Grab`.
   Ereignisse sind die Übergänge: Absprung = Anfahrt→Flug1, Rail-an = Flug1→Rail, Rail-ab = Rail→Flug2,
   Landung = Flug→Ausfahrt. Ein reiner Sprung überspringt `Rail`.
2. Kopf: der bestehende HistGradientBoosting (oder ein kleines 1D-Conv/BiGRU) gibt je Bild
   Wahrscheinlichkeiten je Zustand aus; Labels aus den Hand-Marken (zwischen zwei Marken ist der Zustand
   bekannt) — das sind dichte Labels für alle 123 Tricks, nicht nur ±0,1-s-Fenster.
3. Dekodierung: Viterbi mit erlaubten Übergängen (Grammatik oben), Mindest-/Höchstdauern aus den
   10–90-%-Spannen (Abschnitt 0) und einem Prior gegen Tricks in den ersten 3 s / letzten 2 s
   (Randregel aus #52). Ton-Impulse gehen als zusätzliche Beobachtung in die Übergangs-Wahrscheinlichkeit
   Flug1→Rail und Flug→Ausfahrt ein (Abschnitt 6).
4. Messung: mAP je Ereignis bei 1/2/4 Bildern (Abschnitt 1) gegen die 530 Marken, leave-one-day-out;
   erwartete Wirkung: die 16 «zu kurz» verschwinden, Rail-an/-ab bekommen erstmals eine Bild-Definition.

## 3. Hebel B und C — bessere Geometrie je Bild

### B: Die Reframe-Kurve ist die Kamera

Die fertigen 9:16-Clips sind Insta360-Studio-Reframes aus dem **stabilisierten** 360-Export. Studio
stabilisiert per Gyro (FlowState); die Kugel ist damit rotationsfest zur Welt, und der 9:16-Ausschnitt ist
eine **virtuelle Kamera, deren Pan/Tilt/FOV je Bild bekannt ist** — es sind die Keyframes des
Fliessbands (`_fliessband/spur/<NNN>*/kurve_*_v4_fov45_d50.json`). Folgerung: statt die
Kamerabewegung per Optical Flow aus dem Hintergrund zu *schätzen* (fällt in 5–20 % der Bilder aus,
Boden/Decke ohne Merkmale), lässt sie sich **analytisch herausrechnen**: Pixel → Richtung im
Ausschnitt (FOV) → Rotation um Pan/Tilt → Azimut/Elevation auf der Kugel. Die Hüft-Elevation über der
Zeit ist dann die Flugparabel der Fahrer:in im Weltsystem, nur noch überlagert von Wendelins langsamer
Translation. Das ist genau das, was der Apex der Kurve *indirekt* misst (mit Nachlauf der Glättung), nur
bildgenau und ohne Glättung.

- Voraussetzung prüfen: liegt der Horizont in den Studio-Exporten fest (Horizon Lock / FlowState mit
  Horizont)? Gyroflow-Nutzer berichten, dass **Studio-Reframes keine Gyro-Daten mehr tragen**
  (Insta360 speichert Gyro als Datei-Trailer, den Studio beim Reframe verliert; Gyroflow kopiert ihn
  nur bei nativen, getrimmten Dateien: `copy_insta360_metadata` in `src/util.rs`) — das ist hier egal,
  weil die Kurve die Kamera *ist*.
- Messbar sofort: Hüft-Elevation aus Kurve gegen die kumulierte Flow-Kompensation an den 40 markierten
  Clips; Anteil Bilder mit gültiger Kompensation steigt von 80–95 % auf 100 %.
- Falls Flow trotzdem gebraucht wird (Handcam ohne Kurve): dichter Fluss statt LK auf spärlichen Punkten —
  NeuFlow v2 (Apache, https://github.com/neufieldrobotics/NeuFlow_v2) oder SEA-RAFT (BSD-3, liefert ein
  Checkpoint für genau 540×960: https://github.com/princeton-vl/SEA-RAFT), robust gefittet mit
  `USAC_MAGSAC`; 4-DoF-Ähnlichkeit beibehalten, Homographie nur bei echter Perspektivänderung.

### C: Pose, Fuss, Board, Rail

| Baustein | Empfehlung | Belege |
|---|---|---|
| Körper-Pose | **yolo26s-pose** statt yolo11s-pose: 63,0 vs 58,9 mAP-pose, NMS-frei, RLE-Keypoints, gleiches Paket, MPS; yolo26m-pose 68,8, falls das Zeitbudget reicht | Ultralytics README/YOLO26-Doku, Release v8.4.0 (https://github.com/ultralytics/ultralytics/releases/tag/v8.4.0); AGPL wie bisher |
| **Fuss-Keypoints** (Ferse, Gross-/Kleinzeh) | **DWPose-l oder RTMW-l 384×288 (133 Keypoints, Apache-2.0) via rtmlib** auf dem Personen-Ausschnitt; rtmlib `device='mps'` = onnxruntime CoreML-Provider mit CPU-Rückfall; Fuss-AP DWPose-l 0,704 | https://github.com/Tau-J/rtmlib, https://raw.githubusercontent.com/IDEA-Research/DWPose/onnx/README.md, https://raw.githubusercontent.com/open-mmlab/mmpose/main/projects/rtmpose/README.md |
| Board | **nicht** auf die COCO-Klasse «snowboard» verlassen (einzige Zahl: AP 0,164, stark kontextabhängig, https://arxiv.org/pdf/1809.08132 **[Auszug]**); Board-Linie als Ferse–Zeh-Achse beider Füsse (aus den Fuss-Keypoints), oder **YOLOE-26** (Ultralytics, Text-Prompt «snowboard», «rail», «box», LVIS 30,8–40,6 mAP) für Boxen/Masken | https://raw.githubusercontent.com/ultralytics/ultralytics/main/docs/en/models/yoloe.md |
| Board-Maske je Bild (optional) | **sam2-mlx** (`pip install mlx-sam`): SAM 2.1 hiera-tiny 71 ms / small 85 ms je Bild auf M2 Max 720p → ~2 h für 85 k Bilder, offline tragbar; Seed über YOLOE-Box | https://github.com/avbiswas/sam2-mlx; SAM 3 fällt auf dem Mac weiter aus (Triton, offene MPS-PRs, https://github.com/facebookresearch/sam3/issues?q=mps) |
| Rail/Box | Features stehen in der Halle fest: einmal je Clip (oder je Tag) mit YOLOE «rail»/«box» erkennen und als Linie/Maske im Kugel-Koordinatensystem (Hebel B) halten; Rail-Kontakt = Fuss-/Board-Linie schneidet die Rail-Oberkante | eigener Schritt, kein Paper nötig |
| Bildrate | 50 Bilder/s **nur in den Trick-Fenstern** (Kandidat ±1,5 s), 20 Bilder/s im Rest → Zeitraster 20 ms bei ~1,5× Rechenzeit | — |
| Kopfstehende/rotierende Posen | kein Modell wirbt damit; Rotation über ±30° kostet auf COCO ~5 % (https://arxiv.org/html/2310.06068 **[Auszug]**) → Ausschnitt anhand des Torso-Vektors des Vorbilds aufrichten, Keypoints zurückdrehen; alternativ yolo26-pose mit `degrees=180` auf eigenen Bildern nachtrainieren | — |
| 3D-Welt-Trajektorie (WHAM, TRAM, GVHMR, PromptHMR, OnlineHMR) | **nicht** für die Produktion: CUDA-SLAM-Abhängigkeiten, restriktive Lizenzen, SLAM scheitert auf merkmalsarmen Hallenböden; höchstens als Referenz-Oracle auf einer Workstation. CoMotion (Apple) läuft nativ, liefert aber nur Kamera-relative Posen | https://github.com/yohanshin/WHAM, https://github.com/zju3dv/GVHMR, https://github.com/apple/ml-comotion |
| Sapiens2 (Meta, Apr 2026, 308 Keypoints) | zu schwer für 85 k Bilder auf M1 Max (1B–5B Parameter, CUDA-Quickstart); allenfalls als Gold-Labeler auf einer Teilmenge | https://github.com/facebookresearch/sapiens2 |

## 4. Rail-an / Rail-ab — Kontakt aus dem Bild

**Befund:** Es gibt keine Arbeit, die Rail-Kontakt im Snowboard aus Video bestimmt. Es gibt aber eine
etablierte, mit wenigen hundert Labels trainierbare Vorlage: **Fuss-Kontakt-Netze auf 2D-Keypoints.**

- **Contact and Human Dynamics from Monocular Video** (Rempe et al., ECCV 2020,
  https://github.com/davrempe/contact-human-dynamics): zeitliches Fenster von 2D-Unterkörper-Keypoints →
  je Bild vier Bits (Ferse/Zeh links/rechts); Kontakt-Genauigkeit 0,94 gegenüber 0,85–0,87 für eine
  Geschwindigkeits-Schwelle. **PhysCap** (SIGGRAPH Asia 2020, nicht-kommerziell): 7-Bild-Fenster, gleiche
  Idee. **ContactVision** (Eurographics 2026, MIT, https://github.com/DaeeYong/ContactVision): BODY_25 →
  13 Unterkörper-Gelenke beckenrelativ → Ferse/Zeh-Kontakt je Bild, Checkpoint mitgeliefert.
- **Definition ohne Bodenebene:** WHAM (CVPR 2024, MIT) leitet Kontakt-Wahrheit aus der
  **Fuss-Geschwindigkeit**, nicht aus der Höhe ab — deshalb funktioniert es auf Treppen. Für Rails/Boxen
  heisst das: **Rail = Board-Linie (Ferse–Zeh-Achse, Hebel C) ruht relativ zur Rail-Linie** (Hebel B liefert
  beides im Weltsystem), Flug = Board-Linie bewegt sich frei, Anfahrt/Ausfahrt = Board auf der
  Schnee-Ebene. Das ist eine Geometrie-Regel, die der Phasen-Kopf (Hebel A) als Merkmal bekommt.
- UnderPressure (SCA 2022, https://github.com/InterDigitalInc/UnderPressure): gelernter Kontakt F1 0,947
  gegen 0,909 für die beste Höhe+Geschwindigkeit-Schwelle — gelernt schlägt Schwelle um ~4 Punkte,
  Schwelle bleibt aber eine brauchbare Baseline.
- Objekt-Kontakt allgemein (DECO, InteractVLM, PICO, UniCon3R): Einzelbild, 0,5–3,5 s je Bild,
  nicht-kommerziell, Snowboard/Rail nicht in den Trainingsklassen — **nicht** für uns.
- Ton bestätigt Rail-an (82–88 % der Marken haben einen Impuls ±0,15 s): als Beobachtung in den Übergang
  Flug1→Rail (Abschnitt 6).

**Messkriterium:** Rail-an mAP@2 Bilder, Rail-ab mAP@4 Bilder an den 132/111 Hand-Marken,
leave-one-day-out; heute (Ton) 82 % / 46 % bei ±0,15 s.

## 5. Grab in der Luft

**Befund:** Kein Paper erkennt Snowboard-Grabs aus Video-Keypoints. Die Bausteine sind trotzdem da:

- **Definition und Wahrheit:** ISBS 2024 (Friedl/Gorges/Merz, https://commons.nmu.edu/isbs/vol42/iss1/173/
  **[Auszug]**): Grab-Beginn/-Ende = erste/letzte Hand-Board-Berührung; IMU am Board findet 95/117
  Grabs mit Median 0,01 s Fehler (Trampolin). Für den Markier-Bogen: zwei neue Tasten «Grab an» / «Grab ab».
- **Geometrie:** kleinster Abstand Hand-Keypoints (Handgelenk, mit DWPose die Fingerspitzen) zur
  Board-Linie (Ferse–Zeh-Achsen beider Füsse verbunden), normiert auf die Torso-Länge, nur im Zustand
  Flug; Schwelle plus Mindestdauer (≥ 4 Bilder). Vorbild: «The Way Up» (CVPR-W 2025) erkennt
  Kletter-Griff-Nutzung aus Keypoint-Überlappung mit Griff-Boxen über die Zeit, 50 Bilder/s, mit
  Start/End-Labels (https://openaccess.thecvf.com/content/CVPR2025W/CVSPORTS/papers/Maschek_The_Way_Up_A_Dataset_for_Hold_Usage_Detection_in_CVPRW_2025_paper.pdf).
- **Hand-in-Kontakt-Detektor** als zweites Kriterium: 100DOH (CVPR 2020,
  https://github.com/ddshan/hand_detector.d2) gibt je Hand Kontaktzustand und Objekt-Box aus einem Bild.
- **Grab-Art** (Indy, Melon, Mute, Stalefish, Nose, Tail …): aus welcher Hand (vorne/hinten, Stance aus
  #59 ist unzuverlässig → aus der Fahrtrichtung), Nose/Tail-Seite und Zehen-/Fersenkante
  (Vorzeichen des Kreuzprodukts zur Körperachse) — regelbasiert, oder als **VLM-Frage am Apex-Bild**
  (Abschnitt 7: ein Bild je Trick, 571 Tricks, Batch ≈ 2–4 $).
- Grenzen: Handgelenk-Keypoints sind bei Rotation und Verdeckung durch den Körper unsicher; ein Grab
  hinter dem Körper (Stalefish von hinten gefilmt) ist im Followcam-Bild teils unsichtbar. Das ist ein
  Fall für «Grab?» am Bogen, nicht für 100 %.

## 6. Ton — Feinjustierung, nicht Finder (bestätigt #47)

- **Raster:** Musik-Onset-Erkennung arbeitet mit ±50 ms Toleranz, perkussive Onsets sind «präzise
  markierbar» **[Auszug]**; DCASE bewertet Ereignisse mit 200-ms-Kragen — die Literatur belegt also
  ±50 ms für Aufschläge, nicht ±20 ms. Spektraler Fluss mit 10-ms-Hop (heute schon) plus gelernter Kopf
  bringt die Bild-Marke um ±1–2 Bilder nach.
- **Modelle mit 10-ms-Hop, MIT, mit wenigen hundert Labels feinjustierbar:** EfficientAT mn10
  (4,9 M Parameter, AudioSet mAP 47,1, https://github.com/fschmid56/EfficientAT), PANNs
  Cnn14_DecisionLevelMax (bildweise SED-Ausgabe, https://github.com/qiuqiangkong/audioset_tagging_cnn),
  BEATs (gröberes Token-Raster). CLAP nur Clip-Ebene (Kandidaten-Scoring, keine Zeit). Whisper-AT
  0,4-s-Raster — ungeeignet.
- **Wendelins Brett:** keine Quell-Trennung per Text möglich (gleiche Klasse; AudioSep/TQ-SED trennen
  Klassen, nicht Instanzen). Stattdessen Nah/Fern-Merkmale: das eigene Brett ist lauter, heller, ohne
  Laufzeit; der Aufschlag der Fahrer:in kommt **~3 ms je Meter später** (343 m/s: 29 ms bei 10 m,
  58 ms bei 20 m) — den Abstand aus der Box-Höhe schätzen und **abziehen**, sonst ist der Ton-Versatz
  grösser als der Detektorfehler. Dazu ein 3-Klassen-Kopf (Fahrer-Aufschlag / eigenes Brett / anderes)
  auf den Hand-Marken.
- **X6-Mikrofone:** vier Mikros im 360°-Array plus «Engine»-Mikro laut Produktseite **[Auszug]**; ob die
  `.insv` vier diskrete Kanäle trägt oder nur einen Stereo-Mix, ist für die X6 **nicht belegt** (X5-Nutzer:
  Studio exportiert Stereo). `ffprobe` auf einer `.insv` klärt es in einer Minute; bei ≥ 4 Kanälen liesse
  sich per Laufzeitdifferenz (TDOA) vorne (Fahrer:in) von unten (eigenes Brett) trennen.
- **A/V-Versatz der Kamera** einmal mit einem Klatsch-Test messen — keine Spezifikation auffindbar.
- **Kamera-Gyro als Negativ-Kanal:** Wendelins eigene Ollies, Landungen und Brett-Vibrationen stehen im
  Gyro/Accel der `.insv` (Abschnitt 8) und könnten seine Impulse aus dem Ton-Kanal streichen.

## 7. VLMs — Sekunden-grob, gut für Semantik

**Zeitmarken:** Gemini gibt Zeitmarken im MM:SS-Raster aus (Forum: «timestamp accuracy … on second
level» **[Auszug]**, https://discuss.ai.google.dev/t/improve-timestamp-accuracy-on-video-understanding/95356);
`VideoMetadata.fps` erlaubt (0, 24] Bilder/s und `start_offset`/`end_offset` (verifiziert im SDK,
https://github.com/googleapis/python-genai/blob/main/google/genai/types.py), aber die Ausgabe bleibt
grob. Charades-STA zero-shot: Gemini 2.5 Pro R1@0.5 61 % / mIoU 35 % **[Auszug]**; lokal TimeLens-8B
63,0 / 55,2 (https://github.com/TencentARC/TimeLens), Time-R1-7B 60,8 zero-shot, 72,2 feinjustiert
(Apache, https://github.com/xiaomi-research/time-r1). R1@0.5 heisst 50 % Überlappung auf Momenten von
Sekunden — **zwei Grössenordnungen gröber als ±0,1 s**. Als Finder oder Marken-Setzer ausgeschlossen.

**Kosten für Semantik (Trick-Name, Grab-Art, Rail vs. Sprung, Bail):**

| Weg | Menge | Kosten | Quelle |
|---|---|---|---|
| Claude, 8 Bilder à 540×960 je Trick (700 Token je Bild), 571 Tricks | 3,2 M Bild-Token | Haiku 4.5 ≈ 3,7 $ (Batch 1,9), Sonnet 5 ≈ 7,3 $ (Batch 3,7), Opus 5 ≈ 18 $ (Batch 9) | https://platform.claude.com/docs/en/build-with-claude/vision, https://platform.claude.com/docs/en/about-claude/pricing (verifiziert) |
| Gemini 3.8 Flash, statischer Modus, 5 Bilder/s low, ganzes Archiv 70 min | 1,5 M Token | ≈ 1,1 $ (Flash-Lite 3.5: 0,5 $) | Preise aus LiteLLM-Tabelle (sekundär); Docs gesperrt |
| lokal Qwen3-VL-8B / TimeLens-8B über mlx-vlm oder vllm-mlx | gratis | M4 Max 73 Token/s bei 4-bit; **M1 Max nicht belegt**, Prefill der Bilder dominiert | https://github.com/Blaizzy/mlx-vlm, https://github.com/waybarrios/vllm-mlx/releases |

Claude nimmt weiterhin **kein Video**, nur Bilder (100 je Request bei 200k-Modellen, 600 bei 1M;
> 20 Bilder → jede Seite ≤ 2000 px). Ollama kann bei Qwen3-VL kein Video (Issue offen), llama.cpp hat
`--video-fps` und Zeitstempel-Einfügung. Rolle im Loop: **ein Bild am Apex plus zwei am Rail-an/Landung
je Trick an Haiku/Sonnet im Batch → Trick-Name und Grab-Art als Vorschlag für den Bogen**; Gemini nur,
falls ein Fahrer-übergreifender Grob-Scan neuer Archiv-Ordner («wo sind überhaupt Tricks?») gewünscht ist.

## 8. Gyro und 360-Original

- **telemetry-parser kennt X4 und X5** (Orientierung `yzX`, `cx_fix = 2.0`, PRs #43/#48), **die X6 nicht**
  — kein Eintrag, kein Issue, kein Commit (https://raw.githubusercontent.com/AdrianEddy/telemetry-parser/master/src/insta360/mod.rs).
  Eine unbekannte `camera_type` fällt auf `Xyz`/`cx_fix 1.0` zurück: die `.insv` parst vermutlich, aber
  mit falscher Achsen-Zuordnung, bis ein Vier-Zeilen-PR nach Vorbild #48 sie einträgt.
- **Gyroflow (dieses Repo) exportiert Orientierung je Bild:** `export_gyro_data` in
  `src/core/gyro_export.rs` schreibt CSV/JSON mit `org_quat_w/x/y/z`, `stab_quat_*`, Euler-Winkeln, Accel und
  Gyro je Video-Bild (oder je Sample). Der lokale Checkout pinnt telemetry-parser noch vor dem X5-PR.
  Gyroflow-Doku: 360-Kameras «nicht unterstützt» (fürs Stabilisieren), Gyro-Kurven aus X3-`.insv`
  importieren aber (Issue #848).
- **Wozu die Kamera-IMU taugt:** sie misst **Wendelin**, nicht die Fahrer:in. Nutzen: (a) eigene
  Aufschläge im Ton streichen (Abschnitt 6), (b) Pump-Bewegungen um Features als Kontext, (c) bei
  künftigem Material ohne Studio-Reframe (native `.insv` in Gyroflow) die Welt-Vertikale je Bild — dort ist
  Hebel B über Quaternionen statt über die Kurve zu rechnen.
- **Nicht nötig für Terneuzen:** die stabilisierte Kugel plus Kurve (Hebel B) ist bereits das
  Weltsystem. Alternative auf gleicher Basis: die Fahrer-Spur des Fliessbands direkt auf dem
  stabilisierten Equirect bei 25–50 Bilder/s im Ausschnitt um die Fahrer:in rechnen (Bild-y = Elevation,
  ohne Umrechnung) — ein Schritt weniger, aber die flachen Clips liegen schon da.

## 9. Die Schleife: Sterne-Bogen als Prüf-, Korrektur- und Lern-Werkzeug

Das ist der Teil, der «100 %» tatsächlich liefert. Belegte Muster:

- **Zweistufig, Recall zuerst:** MegaDetector (Kamerafallen) arbeitet mit Schwelle 0,15–0,3 und
  menschlicher Sichtung, «Schwelle so wählen, dass verpasste Tiere gegen Sichtungsaufwand abgewogen
  sind» (https://github.com/microsoft/MegaDetector/blob/main/README.md). Zweistufige Screening-Systeme
  erreichen 0,92 Recall mit menschlicher Triage gegenüber 0,81 vollautomatisch **[Auszug]**.
- **Ein Tastendruck je Kandidat:** Sport-Tracking-Anbieter zeigen der Bedienperson nur die ±2 s um
  den Vorschlag und fragen Ja/Nein/Vielleicht (US-Patent 12190585 **[Auszug]**); EventAnchor (CHI 2021,
  https://dl.acm.org/doi/10.1145/3411764.3445431) und Video2Action (UIST 2023) korrigieren
  CV-Vorschläge interaktiv. Genau das tut der Prüfbogen aus #52 schon (J/D/K, 1–5, B, N).
- **Rest-Fenster** (die eigene Idee): kein benanntes Muster, aber gängige Praxis, unmarkierte Restzeit
  in Segmenten zu prüfen oder eine Zufalls-Stichprobe davon zu ziehen, um die Fehlrate zu *messen*
  (Moderations-Pipelines, Überwachungs-Patent 9589190 **[Auszug]**).
- **Rücklauf ins Modell:** Label Studio ML-Backend (`predict()` schlägt vor, `fit()` läuft per Webhook
  bei jeder Korrektur, https://github.com/HumanSignal/label-studio-ml-backend); CVAT Auto-Annotation.
  Für uns reicht: Urteile-CSV → Neutraining → neuer Bogen.

**Bauplan für den Bogen (Erweiterung von `pruefbogen.py`/`sterne-bogen.html`):**

1. **Kandidaten grosszügig:** Phasen-Kopf mit niedriger Schwelle, NMS-Fenster 1–3 s (SoccerNet-Praxis),
   Quelle und Güte je Trick mitschreiben; «nur Apex» bleibt draussen (27 % stimmt), Randregel bleibt.
2. **Rest-Fenster:** alles, was nicht in einem Trick-Fenster ±1 s liegt, in 4-s-Stücke schneiden und in
   4× Geschwindigkeit als Schleife zeigen; Taste «nichts» oder «Trick hier» (setzt ein Fenster um die
   aktuelle Zeit, das der Kopf danach feinjustiert). Rechnung Terneuzen: 70 min Material, davon ≈ 35 min
   in Trick-Fenstern → 35 min Rest bei 4× ≈ **9 min Sichtung für 100 % Abdeckung**. Bei grösseren
   Archiven zuerst eine Zufalls-Stichprobe der Rest-Fenster sichten; nur wo sie Treffer zeigt, alles.
3. **Nachrücken je Marke:** die vier (sechs mit Grab) Marken als Ticks auf der Zeitleiste, Tab wählt die
   Marke, ←/→ verschiebt sie um ein Bild, das Vorschaubild springt mit; die Verschiebung wird protokolliert
   (`Delta_Bilder` je Marke) — das ist die Präzisionsmessung aus Abschnitt 1.
4. **Sterne und Bail** wie heute; dazu Vorschlag «Trick-Name/Grab-Art» aus dem VLM (Abschnitt 7) als
   editierbares Feld.
5. **Rücklauf:** jede bestätigte oder nachgerückte Marke geht in `ton-markiert.csv` (Quelle «Bogen»),
   Neutraining leave-one-day-out, Kennzahlen je Lauf in `trick-pose-messung.md`. Zielgrössen: Recall
   der Hand-Tricks 100 % (Kandidat oder Rest-Fenster), Nachrück-Anteil > 2 Bilder unter 10 % bei
   Landung/Rail-an, unter 20 % bei Absprung/Rail-ab.

## 10. Sensoren für künftige Drehtage (nicht fürs Archiv)

- **IMU am Board oder Unterschenkel** ist der einzige Weg unter ein Bild: Halfpipe-U-Net Absprung 8 ms,
  Landung 5 ms (Sensors 2024 **[Auszug]**); Board-IMU zählt Rotationen (CCC 0,998, ±8° **[Auszug]**);
  Grab-Beginn/-Ende 0,01 s (Trampolin). Gegenbeispiel: Boot-IMU-Schwelle findet 100 % der Big-Air-Sprünge,
  aber nur 44 % der Sprünge unter 0,5 s Flugzeit (PLOS One 2024 **[Auszug]**) — kleine Rail-Pops bleiben
  auch für Sensoren schwer.
- **Hardware:** Movella DOT (120 Hz Logging, µs-Zeitstempel, Sync 1–3 ppm), Movesense (bis 833 Hz),
  iPhone/Watch (`CMBatchedSensorManager` 200/800 Hz auf Watch Series 8/Ultra; iPhone 100 Hz).
  Konsumenten-Apps (Trace Snow, XON Snow-1) sind eingestellt; Apple Watch/Slopes/Carv messen keine Airtime.
- **Sync mit der Kamera:** ein **Brett-Stomp vor der Linse** zu Beginn jeder Fahrt gibt einen Impuls in
  Accel und Kamera-Ton (Restfehler = Sample-Abstand + Drift, darum je Fahrt neu). Kreuzkorrelation ohne
  Impuls (SyncWISE) erreicht nur 700 ms — ungeeignet. Die X5 nimmt LTC-Timecode über USB-C
  (Ambient Lockit **[Auszug]**), für die X6 nicht belegt.
- **Nutzen:** nicht als Produktionsweg (Fahrer:innen müssen mitmachen), sondern als **gratis Wahrheit**:
  ein Drehtag mit Board-IMU an zwei Fahrer:innen liefert hunderte bildgenaue Marken, ohne dass Wendelin
  markiert — für Halle *und* Outdoor (Abschnitt 11).

## 11. Das ganze Archiv: andere Hallen, Outdoor, andere Kameras

- **Domain-Shift ist real:** Pose-Modelle brechen bei Gegenlicht/Dunkel ein (UDAPose: 3,4 AP bei
  Schwachlicht **[Auszug]**), Outdoor-Leichtathletik zeigt eine «signifikante Erscheinungs-Lücke»
  (AthleticsPose **[Auszug]**). Der Halfpipe-U-Net gewann durch **Feinjustierung je Athlet:in**
  67,6 → 78,7 % gegenüber der Schwelle.
- **Billige Gegenmittel:** (a) je Ort/Tag 10–20 bestätigte Clips aus dem Bogen ins Training (der Loop
  aus Abschnitt 9 liefert sie ohnehin); (b) Pseudo-Labels aus dem unmarkierten Archiv mit Plausibilitäts-
  Filter (Flugdauer 0,5–2,8 s, Parabel) im Mean-Teacher-Schema — reduziert Pose-Domain-Lücken um
  65–82 % **[Auszug]**, bei TAL typisch nur einstellige mAP-Punkte; (c) Test-Zeit-Augmentierung bringt
  0,2–0,5 % **[Auszug]** — nicht der Hebel.
- **Ohne Reframe-Kurve** (Handcam, Drohne, fremdes Material): Hebel B entfällt, dichter Fluss (Abschnitt 3)
  und ggf. Gyro-Quaternionen aus Gyroflow (Abschnitt 8) ersetzen ihn; alles andere bleibt. Die Messung
  dafür braucht den Drehtag aus #49 — mit Board-IMU (Abschnitt 10) kommen die Marken gratis.
- **Umfang:** der Terneuzen-Lauf kostet einmalig ~30 min Pose für 171 Clips; mit yolo26s, Fuss-Keypoints
  auf dem Ausschnitt und 50 Bilder/s nur in Fenstern grob 1–1,5 h je 170 Clips — ein Archiv von 2 000 Clips
  läuft über Nacht.

## Empfehlung: Prototyp-Fahrplan

| Schritt | Inhalt | Messkriterium | Aufwand |
|---|---|---|---|
| 1 Kurve als Kamera (Hebel B) | Pixel → Kugel-Winkel aus `kurve_*.json`, Hüft-/Knöchel-Elevation je Bild; Vergleich mit Flow-Kompensation an 40 Clips | 100 % Bilder kompensiert; Absprung/Landung-Fehler nicht schlechter als heute | ½–1 Tag |
| 2 Phasenmodell (Hebel A) | Zustände je Bild, dichte Labels aus den 530 Marken, Viterbi mit Grammatik und Dauern; Ton als Übergangs-Beobachtung | Rail-an mAP@2 ≥ 85 %, Rail-ab mAP@4 ≥ 70 %, «zu kurz» = 0, Landung/Absprung nicht schlechter | 1–2 Tage |
| 3 Geometrie (Hebel C) | yolo26s-pose; DWPose-l/RTMW-l Fuss- und Hand-Keypoints auf dem Ausschnitt (rtmlib, CoreML); 50 Bilder/s in Fenstern; Board-Linie; YOLOE-26 Rail/Box je Clip | Rail-an/-ab aus Bild allein ≥ Ton; Landung mAP@1 ≥ 80 % | 2–3 Tage |
| 4 Grab | Hand–Board-Abstand im Flug, Mindestdauer; Tasten «Grab an/ab» im Markier-Bogen für ~50 Marken; Grab-Art regelbasiert oder VLM-Vorschlag | Grab-Fenster Recall ≥ 80 % an den neuen Marken; Grab-Art-Vorschlag ≥ 70 % | 1–2 Tage + Markieren |
| 5 Ton-Feinjustierung (Hebel D) | EfficientAT/PANNs-Kopf auf Hand-Marken, 3 Klassen, Laufzeit abziehen, Nachrücken ±3 Bilder | Landung/Rail-an mAP@1 steigt messbar; sonst weglassen | 1 Tag |
| 6 Bogen v2 (Abschnitt 9) | Rest-Fenster, Nachrücken je Marke mit Protokoll, VLM-Felder, Rücklauf-Skript | Recall 100 % am Terneuzen-Material (Kandidat oder Rest-Fenster); Nachrück-Statistik je Art | 1–2 Tage |
| 7 Stufe 2 Sterne (#53) | alle bestätigten Tricks aller Fahrer:innen (braucht Zuordnung #59) als Trick-Clips, Sterne, Ablage je Fahrer-Ordner | Wendelin bewertet das ganze Terneuzen-Material in einer Sitzung | ½ Tag nach #59 |

Reihenfolge nach Ertrag je Aufwand: **1 → 2 → 6 → 3 → 4 → 5**. Schritt 1 und 2 brauchen kein neues
Modell und keine neuen Labels; sie beantworten Rail-an/-ab und «zu kurz» aus dem, was da ist. Schritt 6
macht die Abdeckung zu 100 %, unabhängig davon, wie gut der Kopf wird. Schritte 3–5 heben die
Bild-Genauigkeit gegen die menschliche Untergrenze.

## Offen / nicht belegt

- Ob die Studio-Exporte den Horizont fest halten (Horizon Lock) — Voraussetzung für Hebel B ohne
  Gyro; an drei Clips prüfen (Rail-Oberkante über die Zeit).
- X6: Audio-Kanal-Layout der `.insv` (Stereo vs. 4 Kanäle), Timecode-Eingang, GPS, Eintrag in
  telemetry-parser — alles unbelegt; `ffprobe` und ein Vier-Zeilen-PR klären es.
- Apple-Silicon-Laufzeiten für DWPose/RTMW über CoreML-EP, yolo26-pose, NeuFlow v2, SEA-RAFT,
  V-JEPA 2.1, lokale VLMs auf M1 Max — nirgends publiziert, nur selbst messen.
- Kein Video-Datensatz mit Snowboard-Ereignis-Marken, kein Grab-Paper aus Video, kein Snowboard-
  Pose-Datensatz mit Board-Keypoints (nur Ski-Spitze/-Ende im YouTube-SkiJump-Set).
- Gemini-Free-Tier und Token je Bild für 3.x aus Primärquelle; Charades-STA-Werte für Gemini 3.x,
  GPT-5.x, Claude — nicht auffindbar.
- Modernes AP der COCO-Klasse «snowboard» (nur eine 2018er Zahl); Lizenz von sam2-mlx; Lizenz der
  Rempe-2020-Gewichte.
- Menschliche Grenze «±1–2 Bilder für eine Landung» steht in keiner Quelle wörtlich; sie folgt aus der
  Fussball-Studie (25 Bilder/s) und den Gang-Ereignis-Grenzen.
- Ob der Gradient-Boosting-Kopf mit Zuständen genügt oder UMEG-Net/1D-Conv nötig wird — erst nach
  Schritt 2 messbar.

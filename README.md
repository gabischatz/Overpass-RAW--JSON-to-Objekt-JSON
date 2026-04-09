# 🚲 OSM overpass-raw → OO Konverter v1.0.0

Wandelt eine **Overpass-RAW-JSON** oder **GeoJSON-FeatureCollection** mit OSM-Wegedaten
in ein **objektorientiertes JSON-Modell** um.  
Straßen werden zu Basisobjekten; Radwege, Schutzstreifen und Routen werden als
positionsbezogene `references` darauf verknüpft.

---

## ℹ️ Woher kommt die Eingabedatei?

Die Datei `tour-analyzer-pro-v17-overpass-raw.json` (oder ähnlich benannt) wird vom
**Tour Analyzer Pro** erzeugt und kann dort direkt heruntergeladen werden.

### Wie der Tour Analyzer die Datei erstellt

1. **GPX-Datei laden** – Der Benutzer lädt eine GPX-Touraufzeichnung in den Tour Analyzer.
   Das Werkzeug liest alle Trackpunkte und berechnet daraus eine Bounding Box
   (Ausdehnung der Tour plus ca. 315 m Puffer auf jeder Seite).

2. **Overpass-Abfrage aufbauen** – Aus der Bounding Box wird automatisch eine
   Overpass-QL-Abfrage zusammengestellt. Sie fragt in einem einzigen Aufruf ab:
   - alle Fahrradrouten-Relationen (`type=route`, `route=bicycle`)
   - Relationen mit Netzwerk-Tag `lcn`, `rcn` oder `ncn`
   - alle relevanten Wege (`highway=cycleway/path/track/service/footway/
     living_street/residential/unclassified/tertiary/secondary/primary`)
   - Wege mit `bicycle_road=yes` oder `cyclestreet=yes`
   - Wege mit einem beliebigen `cycleway`-Tag

   Die fertige Abfrage sieht so aus:
   ```
   [out:json][timeout:120];
   (
     relation["type"="route"]["route"="bicycle"](süd,west,nord,ost);
     relation["route"="bicycle"]["network"~"^(lcn|rcn|ncn)$"](süd,west,nord,ost);
     way["highway"~"cycleway|path|track|..."](süd,west,nord,ost);
     way["bicycle_road"="yes"](süd,west,nord,ost);
     way["cyclestreet"="yes"](süd,west,nord,ost);
     way["cycleway"](süd,west,nord,ost);
   );
   out body;>;out geom qt;
   ```
   Alternativ kann auch direkt eine OSM-Relationskennung (z.B. `12345`) eingegeben
   werden; dann lautet die Abfrage vereinfacht:
   `[out:json][timeout:90];relation(12345);(._;>;);out geom;`

3. **Abfrage abschicken** – Die Abfrage wird als HTTP-POST an die Overpass-API
   geschickt (`overpass-api.de`, mit automatischem Fallback auf
   `overpass.kumi.systems` und `overpass.openstreetmap.ru` bei Fehler oder
   Rate-Limit). Timeout je Versuch: 35 Sekunden, bis zu 3 Versuche pro Endpunkt.

4. **Rohantwort speichern & exportieren** – Die API antwortet mit einem
   JSON-Objekt im Overpass-RAW-Format (`{ "elements": [...] }`).
   Das Werkzeug speichert diese Antwort intern und stellt sie über den Button
   **„⬇️ Rohantwort"** als `tour-analyzer-pro-v17-overpass-raw.json` zum
   Download bereit.

5. **Diese Datei ist der Eingang für den OO-Konverter.** Sie enthält alle
   `node`-, `way`- und `relation`-Elemente des abgefragten Gebiets im
   Overpass-RAW-Format und kann direkt in den OO-Konverter geladen werden.

---

## Inhaltsverzeichnis

1. [Überblick](#1-überblick)
2. [Eingabeformate](#2-eingabeformate)
3. [Ablaufplan – Schritt für Schritt](#3-ablaufplan--schritt-für-schritt)
   - [Phase 0 – Datei laden & parsen](#phase-0--datei-laden--parsen)
   - [Phase 1 – Normalisierung zu Segmenten](#phase-1--normalisierung-zu-segmenten)
   - [Phase 2 – Klassifizierung](#phase-2--klassifizierung)
   - [Phase 3 – Tag-Referenzen erkennen](#phase-3--tag-referenzen-erkennen)
   - [Phase 4 – Straßenbasisobjekte aufbauen](#phase-4--straßenbasisobjekte-aufbauen)
   - [Phase 5 – Direkte Tag-Referenzen einbauen](#phase-5--direkte-tag-referenzen-einbauen)
   - [Phase 6 – Overlay-Objekte aufbauen](#phase-6--overlay-objekte-aufbauen)
   - [Phase 7 – Geometrische Zuordnung (Heuristik)](#phase-7--geometrische-zuordnung-heuristik)
   - [Phase 8 – Ausgabe zusammenstellen](#phase-8--ausgabe-zusammenstellen)
4. [Datenfluss-Diagramm](#4-datenfluss-diagramm)
5. [Klassenlogik (A–H)](#5-klassenlogik-a-h)
6. [Optionen](#6-optionen)
7. [Ausgabe-Struktur](#7-ausgabe-struktur)
8. [Export-Varianten](#8-export-varianten)
9. [Hilfsfunktionen & Geometrie](#9-hilfsfunktionen--geometrie)

---

## 1 Überblick

```
Overpass-RAW-JSON  ──┐
                     ├──► parseInputData() ──► FeatureCollection ──► OO-JSON
GeoJSON            ──┘
```

Das Werkzeug läuft vollständig im Browser (eine einzelne HTML-Datei, kein Server
nötig). Die gesamte Logik steckt in einem IIFE (`(function(){ ... })()`).

---

## 2 Eingabeformate

| Format | Erkennung | Besonderheit |
|--------|-----------|--------------|
| **GeoJSON FeatureCollection** | `data.type === 'FeatureCollection'` | Wird direkt verwendet |
| **Overpass-RAW-JSON** | `Array.isArray(data.elements)` | Wird zuerst in FeatureCollection konvertiert |

### Overpass-RAW → FeatureCollection (`convertOverpassRawToFeatureCollection`)

1. Alle Elemente vom Typ `node` werden in einer `Map(nodeId → [lon, lat])` gespeichert.
2. Alle Elemente vom Typ `way` mit mindestens 2 Knoten werden durchlaufen.
3. Für jeden Way werden die `nodes`-IDs über die Map in Koordinaten aufgelöst.
4. Daraus entsteht ein GeoJSON-`Feature` mit `geometry.type = "LineString"` und
   den OSM-Tags als `properties` (plus `sourceType`, `sourceWayId`, `sourceNodeCount`).

---

## 3 Ablaufplan – Schritt für Schritt

### Phase 0 – Datei laden & parsen

```
FileReader.readAsText()
    └─► JSON.parse(text)
         └─► parseInputData()
              ├─ FeatureCollection?  → direkt verwenden
              └─ Overpass-RAW?       → convertOverpassRawToFeatureCollection()
                                          → nur LineStrings & MultiLineStrings behalten
```

**Funktion:** `handleFile(file)` → `parseInputData(text)`

Nach dem Laden:
- `state.rawData` enthält die FeatureCollection.
- `state.sourceFeatures` enthält nur die LineString/MultiLineString-Features (Punkte
  und Polygone werden verworfen).
- Der „OO erzeugen"-Button wird aktiviert.

---

### Phase 1 – Normalisierung zu Segmenten

**Funktion:** `featureToSegments(feature, featureIndex)`

Jedes Feature wird in ein oder mehrere **normalisierte Segmente** umgewandelt:

```
Feature (LineString oder MultiLineString)
    └─► flattenProperties()         – verschachtelte tags-Objekte werden aufgeklappt
    └─► Koordinaten bereinigen
         ├─ NaN-Koordinaten filtern
         ├─ dedupeCoords()           – direkt aufeinanderfolgende Duplikate entfernen
         └─ mindestens 2 Punkte nötig
    └─► Segment-Objekt erzeugen:
         segmentId         = "seg_{featureIndex}_{geomIndex}"
         name              = normalizeName(name | ref | addr:street | "unnamed")
         baseClassId       = inferBaseClass(props)      → Phase 2
         referencesFromTags = detectTagReferences(props) → Phase 3
         coords            = [[lon,lat], ...]
         startNodeKey      = "lon,lat" (8 Dezimalstellen)
         endNodeKey        = "lon,lat"
         lengthM           = Haversine-Summe aller Segmentabschnitte
         overallBearing    = Azimut Startpunkt → Endpunkt
```

---

### Phase 2 – Klassifizierung

**Funktion:** `inferBaseClass(props)`

Jedes Segment bekommt eine **Basis-Klassen-ID** (A–H):

```
Regelwerk zur Klassifizierung von Wegtypen (Zielattribute G, B, D, E, F, H)

Privatstraße (G)

access=private ODER access=no

ODER service=driveway

Radweg (B)

highway=cycleway

ODER highway=path + (bicycle=designated ODER bicycle=official)

Fußweg mit Rad frei (D)

(highway=footway ODER highway=pedestrian) + bicycle=yes

ODER highway=path + foot=yes + bicycle=yes

Straße (E)

isStreetHighway(highway) = true
(Annahme: erfasst alle highway-Werte, die übliche Straßentypen darstellen, z. B. residential, unclassified, tertiary, etc.)

Land-/Forstweg (F)

highway=track + bicycle ≠ no

ODER highway=path + bicycle ≠ no

Sonstiges (H)

Alle Fälle, die nicht in 1–5 fallen
access=private/no  ODER  service=driveway   →  G  (Privatstraße)
highway=cycleway                            →  B  (Radweg)
highway=path + bicycle=designated/official  →  B  (Radweg)
highway=footway/pedestrian + bicycle=yes    →  D  (Fußweg mit Rad frei)
highway=path + foot=yes + bicycle=yes       →  D
isStreetHighway(highway)                    →  E  (Straße)
highway=track + bicycle≠no                  →  F  (Land-/Forstweg)
highway=path + bicycle≠no                   →  F
sonst                                       →  H  (Sonstiges)
```

`isStreetHighway` erkennt: `residential`, `living_street`, `unclassified`, `service`,
`tertiary`, `secondary`, `primary`, `trunk`, `motorway`.

---

### Phase 3 – Tag-Referenzen erkennen

**Funktion:** `detectTagReferences(props, baseClassId)`

Sucht in den OSM-Tags nach Hinweisen auf **andere Kategorien**, die auf demselben
Weg kodiert sind:

| Tag-Kombination | Erzeugt Referenz |
|----------------|-----------------|
| `route=bicycle` oder `network=lcn/rcn/ncn/icn` oder Relationsname mit „Radweg/cycle" | Klasse **A** (Radroute) |
| `cycleway:both/left/right = track` | Klasse **B** (Radweg), Seite |
| `cycleway:both/left/right = lane` | Klasse **C** (Schutzstreifen), Seite |
| `cycleway:...:lane = advisory/exclusive` | Klasse **C** (Schutzstreifen) |
| `cycleway = shared_lane` | Klasse **C** (Shared Lane) |

Duplikate (gleiche classId + kind + side + direction) werden mit `uniqueBy()`
entfernt.  
Jede erkannte Referenz ist ein Objekt mit:
`{ classId, kind, side, direction, confidence, origin: "tag", startPosM: 0, endPosM: null }`

---

### Phase 4 – Straßenbasisobjekte aufbauen

**Funktion:** `buildRoadBases(normalizedSegments, options)`

Nur Segmente mit `baseClassId === 'E'` (Straßen) kommen hier rein.

```
Schritt 4.1 – Gruppieren nach Name
    Map("strassenname" → [Segmente])

Schritt 4.2 – Ketten bilden: joinConnectedSegments(group, endpointTol)
    ┌─ Startpunkt & Endpunkt jedes Segments in keyIndex (räumlicher Hash)
    ├─ Greedy-Algorithmus:
    │   1. Erstes freies Segment als Ketten-Anfang nehmen
    │   2. Vom Ende der Kette: passenden Anschlusspunkt suchen (≤ endpointTol m)
    │      → nach Abstand und Winkeldifferenz sortiert
    │   3. Passendes Segment anhängen (ggf. Koordinaten umkehren)
    │   4. Vom Anfang der Kette dasselbe nach vorne
    │   5. Wiederholen bis keine Verbindung mehr gefunden
    └─ Eine Kette = zusammenhängender Linienzug

Schritt 4.3 – Road-Objekt erzeugen
    roadId        = "road_0001", "road_0002", ...
    coords        = dedupeCoords(alle Kettenkoordinaten)
    lengthM       = Haversine-Gesamtlänge
    tags          = gemergede Tags aller Mitgliedssegmente (first-wins)
    references    = []   (wird in Phase 5 & 7 befüllt)
    _helper       = { cumulative[], memberSegments[] }   (intern, wird in Phase 8 gelöscht)
```

`cumulativeMeasures(coords)` baut ein Array der kumulierten Meter-Positionen entlang
der Linie (wird für Positions-Projektion benötigt).

---

### Phase 5 – Direkte Tag-Referenzen einbauen

**Funktion:** `addDirectReferencesToRoads(roads)`

Für jede Straße werden die eigenen Mitgliedssegmente durchsucht.  
Hat ein Segment `referencesFromTags` (aus Phase 3), dann:

```
1. Startpunkt des Segments → projectPointToPolyline() → Meter-Position auf der Straße
2. Endpunkt des Segments   → projectPointToPolyline() → Meter-Position auf der Straße
3. Referenz-Objekt erzeugen:
   referenceId  = "ref_00001", ...
   source       = "same_segment"
   startPosM    = Min(projStart, projEnd)  [gerundet auf Meter]
   endPosM      = Max(projStart, projEnd)
   startCoord / endCoord
   + alle Felder aus detectTagReferences
4. collapseRoadReferences() – benachbarte gleiche Referenzen zusammenführen
   (gleiche classId + kind + side + direction UND Abstand ≤ 8 m)
```

**`projectPointToPolyline`** läuft über alle Liniensegmente der Straße und findet
den nächsten Fußpunkt (orthogonale Projektion, ebene Näherung mit `cosLat`-Korrektur).

---

### Phase 6 – Overlay-Objekte aufbauen

**Funktion:** `buildOverlayObjects(normalizedSegments, options)`

Alle Segmente mit `baseClassId !== 'E'` (also A, B, C, D, F, G, H) werden zu
Overlay-Objekten:

```
Gruppieren nach "classId|name"
→ joinConnectedSegments() (gleiche Ketten-Logik wie Phase 4)
→ Overlay-Objekt:
   objectId       = "obj_0001", ...
   baseClassId    = A/B/C/D/F/G/H
   coords, lengthM, startNodeKey, endNodeKey
   memberSegmentIds
   tags
   linkedRoadId   = null   (wird in Phase 7 gesetzt)
```

---

### Phase 7 – Geometrische Zuordnung (Heuristik)

**Funktion:** `mapExternalOverlaysToRoads(overlayObjects, roads, options, startingRefCounter)`

Jedes Overlay-Objekt (außer Klasse H) wird versucht, einer Straße zuzuordnen:

```
Für jedes Overlay-Objekt:
  Für jede Straße:
    1. Projektion aller Overlay-Koordinaten auf die Straßenlinie
    2. Filtern: nur Punkte ≤ proximityMeters (Standard: 22 m)
    3. Mindestanzahl naher Punkte: max(2, ceil(n * coverageRatio))
    4. Parallel-Check: angleDiff(overlayBearing, roadBearing) ≤ parallelDeg
       → Klasse A (Radrouten) sind vom Parallel-Check ausgenommen
    5. Mindestabdeckung: coveredLength ≥ min(12, lengthM * 0.25)
    6. Score = avgDist + parallelDiff * 0.8 - sameNameBonus(6 Punkte wenn Namen gleich)
    7. Kandidat mit bestem (niedrigstem) Score gewinnt

  Beste Straße gefunden?
    → obj.linkedRoadId = road.roadId
    → Referenz in road.references eintragen:
         source         = "parallel_object"
         side           = inferSide()      (Kreuzprodukt links/rechts)
         direction      = inferDirection() (Azimut-Differenz → forward/backward/both)
         confidence     = 1 − avgDist/maxDist − parallelDiff/180 + sameNameBonus/100
    → linkDiagnostics eintragen

  Keine Straße gefunden?
    → Objekt bleibt ohne linkedRoadId → wird zu Standalone (Phase 8)

Nach allen Zuordnungen:
  collapseAndSortRefs() für jede Straße
    – sortieren nach startPosM, dann classId, side
    – benachbarte gleiche Referenzen verschmelzen (Abstand ≤ 8 m)
```

---

### Phase 8 – Ausgabe zusammenstellen

**Funktion:** `convertNow()` (letzter Teil)

```
finalizeRoads(roads)
  → _helper-Feld entfernen (interne Hilfsdaten)

buildStandaloneOutput(overlayObjects)
  → nur Objekte ohne linkedRoadId
  → schlankes Format (objectId, name, classId, coords, lengthM, tags, ...)

buildChainsOutput(finalizedRoads, standalone)
  → Straßen + Standalone in einer gemeinsamen chains[]-Liste
  → einheitliches chainId/id-Feld

Ausgabe-Objekt:
{
  metadata: {
    generated, sourceFile, version, options,
    counts: { sourceFeatures, normalizedSegments, roads, standalone, chains, totalReferences }
  },
  roads:      [ ...finalizedRoads ],
  standalone: [ ...standaloneObjects ],
  chains:     [ ...roadChains, ...standaloneChains ]
}
```

---

## 4 Datenfluss-Diagramm

```
┌─────────────────────────┐
│   JSON-Datei (Eingabe)  │
│  Overpass-RAW oder      │
│  GeoJSON FeatureColl.   │
└────────────┬────────────┘
             │ parseInputData()
             ▼
┌─────────────────────────┐
│    FeatureCollection    │
│  nur LineString /       │
│  MultiLineString        │
└────────────┬────────────┘
             │ featureToSegments()
             ▼
┌─────────────────────────┐
│  Normalisierte          │◄── inferBaseClass()
│  Segmente[]             │◄── detectTagReferences()
│  seg_0_0, seg_1_0, ...  │
└──────────┬──────────────┘
           │
     ┌─────┴──────────────────────────────┐
     │                                    │
     ▼  baseClassId === 'E'               ▼  baseClassId ≠ 'E'
┌─────────────────────┐        ┌──────────────────────┐
│  Straßen-Segmente   │        │  Overlay-Segmente    │
│  buildRoadBases()   │        │  buildOverlayObjects()│
│  joinConnected...() │        │  joinConnected...()  │
└─────────┬───────────┘        └──────────┬───────────┘
          │                               │
          │ addDirectReferences...()      │ mapExternalOverlays...()
          │ (Tag-Referenzen auf Pos.)     │ (Geometrie-Heuristik)
          ▼                               ▼
┌─────────────────────┐        ┌──────────────────────┐
│  Road-Objekte       │◄───────│  Verknüpfte Overlays │
│  mit references[]   │        │  als references[]    │
└─────────┬───────────┘        └──────────┬───────────┘
          │                               │ linkedRoadId == null
          │                               ▼
          │                    ┌──────────────────────┐
          │                    │  Standalone-Objekte  │
          │                    └──────────┬───────────┘
          │                               │
          └──────────────┬────────────────┘
                         ▼
               ┌──────────────────┐
               │   OO-JSON        │
               │  roads[]         │
               │  standalone[]    │
               │  chains[]        │
               │  metadata{}      │
               └──────────────────┘
```

---

## 5 Klassenlogik (A–H)

| ID | Name | Farbe | Beschreibung |
|----|------|-------|--------------|
| A | Radroute | 🟢 `#22c55e` | Radroute aus Relationen oder `route=bicycle` |
| B | Radweg | 🔵 `#3b82f6` | Eigenständiger oder via `cycleway=track` |
| C | Schutzstreifen | 🟣 `#a855f7` | `cycleway=lane`, `shared_lane`, `advisory` |
| D | Fußweg mit Rad frei | 🟡 `#facc15` | `highway=footway` + `bicycle=yes` |
| E | Straße | 🟠 `#f97316` | Alle `highway=residential/primary/...` |
| F | Land-/Forstweg | 🟩 `#84cc16` | `highway=track/path` ohne Radverbot |
| G | Privatstraße | 🩷 `#fb7185` | `access=private` oder `service=driveway` |
| H | Sonstiges | ⬜ `#64748b` | Alles andere |

---

## 6 Optionen

| Parameter | Standard | Bereich | Bedeutung |
|-----------|---------|---------|-----------|
| `proximityMeters` | 22 | 5–80 | Max. Abstand Overlay → Straße für Zuordnung |
| `parallelDeg` | 35 | 5–90 | Max. erlaubte Winkelabweichung (Parallelität) |
| `coverageRatio` | 0.55 | 0.1–1 | Anteil der Overlay-Punkte die nahe genug sein müssen |
| `endpointTol` | 7 | 1–20 | Toleranz in Metern beim Zusammenfügen von Segmenten |

---

## 7 Ausgabe-Struktur

### Road-Objekt (in `roads[]`)

```json
{
  "roadId": "road_0001",
  "chainId": "road_0001",
  "name": "Hauptstraße",
  "baseClassId": "E",
  "baseClassName": "Straße",
  "coords": [[lon, lat], ...],
  "lengthM": 842,
  "startNodeKey": "11.12345678,51.12345678",
  "endNodeKey": "11.13456789,51.13456789",
  "memberSegmentIds": ["seg_3_0", "seg_7_0"],
  "tags": { "highway": "residential", "name": "Hauptstraße" },
  "references": [
    {
      "referenceId": "ref_00001",
      "source": "same_segment",
      "classId": "B",
      "kind": "radweg",
      "side": "right",
      "direction": "both",
      "startPosM": 0,
      "endPosM": 842,
      "confidence": 1,
      "origin": "tag"
    }
  ]
}
```

### Standalone-Objekt (in `standalone[]`)

```json
{
  "objectId": "obj_0003",
  "chainId": "obj_0003",
  "name": "Elsterradweg",
  "classId": "A",
  "className": "Radroute",
  "coords": [[lon, lat], ...],
  "lengthM": 1240,
  "startNodeKey": "...",
  "endNodeKey": "...",
  "memberSegmentIds": ["seg_12_0"],
  "tags": { "route": "bicycle", "name": "Elsterradweg" }
}
```

### Reference-Objekt

| Feld | Typ | Bedeutung |
|------|-----|-----------|
| `referenceId` | string | Eindeutige ID `ref_00001` |
| `source` | string | `"same_segment"` (Tag) oder `"parallel_object"` (Geometrie) |
| `classId` | A–H | Klasse des referenzierten Elements |
| `kind` | string | z.B. `"radweg"`, `"schutzstreifen"`, `"radroute"` |
| `side` | string | `"left"`, `"right"`, `"center"`, `"both"`, `"unknown"` |
| `direction` | string | `"forward"`, `"backward"`, `"both"` |
| `startPosM` | number | Startposition in Meter entlang der Straße |
| `endPosM` | number | Endposition in Meter entlang der Straße |
| `confidence` | number | 0.05–0.99 (nur bei Geometrie-Zuordnung) |
| `origin` | string | `"tag"` oder `"geometry"` |

---

## 8 Export-Varianten

| Schaltfläche | Dateiname | Inhalt |
|-------------|-----------|--------|
| Vollständiges OO JSON | `*_oo_v2.json` | `metadata` + `roads` + `standalone` + `chains` |
| Nur Straßenbasis | `*_roads_only.json` | `metadata` + `roads` |
| Nur Standalone | `*_standalone_only.json` | `metadata` + `standalone` |
| Debug JSON | `*_debug_v2.json` | `metadata` + `directReferenceStats` + `externalLinks` + `normalizedSegmentsSample` |

---

## 9 Hilfsfunktionen & Geometrie

| Funktion | Beschreibung |
|----------|-------------|
| `haversine(lon1,lat1,lon2,lat2)` | Entfernung in Metern zwischen zwei WGS-84-Punkten |
| `bearingDeg(lon1,lat1,lon2,lat2)` | Azimut in Grad (0–360) |
| `angleDiff(a,b)` | Kleinste Winkeldifferenz (0–90°, berücksichtigt Gegenrichtung) |
| `segmentPointProjection(p, a, b)` | Orthogonale Projektion eines Punktes auf ein Liniensegment (mit cosLat-Korrektur) |
| `projectPointToPolyline(point, coords, cumulative)` | Beste Projektion auf eine gesamte Polylinie + Meter-Position |
| `cumulativeMeasures(coords)` | Array der kumulierten Meter-Positionen entlang einer Linie |
| `dedupeCoords(coords)` | Entfernt direkt aufeinanderfolgende identische Koordinaten |
| `inferSide(a, b, p)` | Kreuzprodukt → links / rechts / center |
| `inferDirection(overlayBearing, roadBearing)` | forward / backward / both anhand Azimut-Differenz |
| `collapseAndSortRefs(refs, roadLengthM)` | Benachbarte gleiche Referenzen verschmelzen + nach Position sortieren |
| `joinConnectedSegments(segments, tol)` | Greedy-Ketten-Algorithmus mit Winkel-Priorisierung |

---

## Lizenz

Dieses Werkzeug ist ein reines Client-Side-Tool.  
OSM-Daten unterliegen der [ODbL-Lizenz](https://www.openstreetmap.org/copyright).

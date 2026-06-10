# pgrouting Hundesitter Tutorial

## Einleitung

In dieser Übung löst ihr ein **Traveling Salesman Problem (TSP)** mit pgrouting: Wie fahrt ihr als Hundesitter optimal zu mehreren Kunden in Bochum, um Hunde abzuholen und wieder abzuliefern?

**Was ihr lernt:**
- PostGIS und pgrouting in pgAdmin verwenden
- Shortest-Path-Algorithmen (Dijkstra)
- TSP-Algorithmen zur Routenoptimierung
- Ergebnisse im Geometry Viewer visualisieren
- Fahrtzeiten und Kosten berechnen

**Voraussetzungen:**
- Zugang zu pgAdmin auf `edu-map25.geomatik.ruhr-uni-bochum.de`
- Eine eigene Datenbank `gisII_[nachname]`
- Der Dump `pgr_dog.dump` zur Hand

---

## 1. pgAdmin verbinden

Öffnet pgAdmin und verbindet euch mit dem Server:

- **Server:** `edu-map25.geomatik.ruhr-uni-bochum.de`
- **Port:** 5432 (standard)
- **Username:** [euer Nutzername]
- **Password:** [euer Passwort]

Im linken Panel unter **Servers** seht ihr dann den verbundenen Server. Klickt auf ihn und navigiert zu eurer Datenbank.

<img width="1154" height="573" alt="image" src="https://github.com/user-attachments/assets/e7d1d073-62be-49e1-94d7-5a8d8a47d0d0" />

---

## 2. Datenbank einspielen

Die Übungsdaten liegen im Dump `pgr_dog.dump`. So spielt ihr sie in eure Datenbank ein:

- Im **Object Explorer:** Rechtsklick auf eure DB → **Restore**
- **Format:** Custom or tar
- **Filename:** Pfad zur `pgr_dog.dump`
- **Data Options:** "Do not save Owner" ankreuzen
- **Restore** ausführen

Der Status "Failed" ist ok – wichtig ist, dass die Tabellen nachher vorhanden sind.

**Kontrolle:** Schemas → public → Tables (evtl. Refresh):
- `halter`
- `ways`
- `ways_vertices_pgr`

---

## 3. Die Datenbank

Das Straßennetz besteht aus drei Komponenten:

| Tabelle | Inhalt |
|---------|--------|
| **ways_vertices_pgr** | Knoten mit Koordinaten (Straßenkreuzungen) |
| **ways** | Kanten mit Knoten-IDs, Längen und Geschwindigkeiten |
| **halter** | Hundebesitzer mit Koordinaten und Preisen |

### Das Szenario

Ihr habt mehrere Kunden (halter) in Bochum. Jeder Kunde hat:
- Koordinaten (lon, lat)
- Eine Gebühr pro Stunde

Die Kosten einer Strecke berechnen sich aus **Fahrzeit**:

$$\text{cost} = \frac{\text{ways.length\_m}}{1000 \times \text{ways.maxspeed\_forward}} \times 60 \quad [\text{min}]$$

### halter-Tabelle erweitern

Um Studis hinzuzufügen: Schemas → public → Tables → Rechtsklick auf `halter` → View/Edit Data → First 100 Rows → **Add Row** → **Save Changes**

---

## 4. node_id bestimmen

pgRouting arbeitet mit **Knoten-IDs** aus `ways_vertices_pgr`, nicht mit Koordinaten. Wir müssen für jeden Halter den nächstgelegenen Knoten finden und seine ID speichern.

```sql
UPDATE halter h
SET node_id = (
  SELECT v.id
  FROM ways_vertices_pgr v
  ORDER BY v.the_geom <-> ST_SetSRID(
    ST_MakePoint(▯.lon, ▯.lat), 4326)
  LIMIT 1
);
```

---

## 5. Kostenmatrix berechnen

`pgr_dijkstraCostMatrix` berechnet für jedes Knotenpaar die kürzeste Fahrzeit. Das Ergebnis ist eine vollständige (Zeit)Kostenmatrix.

```sql
SELECT * FROM pgr_dijkstraCostMatrix(
  'SELECT gid AS id, source, target,
   ways.length_m / 1000.0 / ▯ * 60.0 AS cost,
   CASE WHEN reverse_cost > 0
     THEN ways.length_m / 1000.0 / ▯ * 60.0
     ELSE -1
   END AS reverse_cost
   FROM ways',
  ARRAY(SELECT node_id FROM halter
    WHERE id = ANY(ARRAY[▯, ▯, ▯, ▯, ▯, ▯])),
  directed := true
);
```

---

## 6. TSP für ideale Route

`pgr_TSP` findet die optimale Besuchsreihenfolge der Knoten – also in welcher Reihenfolge fahrt ihr die Halter an, um die Gesamtfahrzeit zu minimieren.

**Wichtig:** TSP berechnet geschlossene Rundtouren – die letzte Station ist immer der Ausgangspunkt, den ignorieren wir.

```sql
SELECT seq, node, cost, agg_cost
FROM pgr_TSP(
  $$SELECT * FROM pgr_dijkstraCostMatrix(
    'SELECT gid AS id, source, target,
     ways.length_m / 1000.0 / ▯ * 60.0 AS cost,
     CASE WHEN ways.reverse_cost > 0
       THEN ways.length_m / 1000.0 / ▯ * 60.0
       ELSE -1
     END AS reverse_cost
     FROM ways',
    ARRAY(SELECT node_id FROM halter
      WHERE id = ANY(ARRAY[▯, ▯, ▯, ▯, ▯, ▯])),
    directed := true
  )$$,
  start_id := (SELECT node_id FROM halter WHERE id = ▯),
  end_id := (SELECT node_id FROM halter WHERE id = ▯)
);
```

---

## 7. tsp() als Funktion

Da wir TSP noch mehrfach berechnen (Hinweg, Rückweg, Visualisierung), speichern wir es in einer **Funktion**. Funktionen werden einmalig erstellt und sind dann dauerhaft in der DB unter Schemas → public → Functions gespeichert.

```sql
CREATE OR REPLACE FUNCTION tsp(halter_ids int[], start_id int, end_id int)
RETURNS TABLE(seq int, node bigint, cost float, agg_cost float) AS $func$
DECLARE
  knoten bigint[];
  matrix text;
BEGIN
  SELECT ARRAY(SELECT node_id FROM halter
    WHERE id = ANY(halter_ids)) INTO knoten;
  
  matrix := 'SELECT * FROM pgr_dijkstraCostMatrix(
    ''SELECT gid AS id, source, target,
     ways.length_m / 1000.0 / ways.maxspeed_forward * 60.0 AS cost,
     CASE WHEN ways.reverse_cost > 0
       THEN ways.length_m / 1000.0 / ways.maxspeed_forward * 60.0
       ELSE -1
     END AS reverse_cost
     FROM ways'',
    ARRAY[' || array_to_string(knoten, ',') || '],
    directed := true)';
  
  RETURN QUERY
  SELECT t.seq, t.node, t.cost, t.agg_cost
  FROM pgr_TSP(
    matrix,
    start_id := (SELECT node_id FROM halter WHERE id = start_id),
    end_id := (SELECT node_id FROM halter WHERE id = end_id)
  ) t;
END;
$func$ LANGUAGE plpgsql;
```

Nach Ausführung findet ihr die Funktion unter Schemas → public → Functions → tsp().

---

## 8. Route visualisieren

Jetzt visualisieren wir die Route mit `pgr_dijkstraVia`, das die exakte Strecke entlang des Straßennetzes durch alle Knoten in TSP-Reihenfolge berechnet. `ST_Collect` fasst alle Straßenabschnitte zu einer Linie zusammen – sichtbar im Geometry Viewer.

```sql
WITH tsp_hinweg AS (
  SELECT node, seq FROM tsp(ARRAY[0, 1, 3, 7, 10, 99], 0, 99)
  ORDER BY seq
)
SELECT ST_Collect(ways.the_geom) AS route
FROM pgr_dijkstraVia(
  'SELECT gid AS id, source, target,
   ways.length_m / 1000.0 / ways.maxspeed_forward * 60.0 AS cost,
   CASE WHEN ways.reverse_cost > 0
     THEN ways.length_m / 1000.0 / ways.maxspeed_forward * 60.0
     ELSE -1
   END AS reverse_cost
   FROM ways',
  ARRAY(SELECT node FROM tsp_hinweg
    WHERE seq < (SELECT MAX(seq) FROM tsp_hinweg)),
  directed := true
) AS route
JOIN ways ON ways.gid = route.edge
WHERE route.edge != -1;
```

**So seht ihr die Route:**
1. Query ausführen
2. Im Ergebnis-Fenster: Rechtsklick auf die `route`-Spalte → **View Geometry** (oder Geometry Viewer-Symbol)
3. Die Route wird auf der Karte angezeigt!

---

## 9. TSP für Hin- & Rückweg

Um die Hunde auch wieder abzuliefern, führen wir `pgr_TSP` zweimal aus – einmal für den Hinweg (Zuhause → Hundewiese), einmal für den Rückweg (Hundewiese → Zuhause). Mit `UNION ALL` werden beide Ergebnisse in einer Tabelle zusammengefasst.

```sql
SELECT 'Hinweg' AS richtung, seq, node, cost, agg_cost
FROM tsp(ARRAY[▯, ▯, ▯, ▯, ▯, ▯], ▯, ▯)

UNION ALL

SELECT 'Rückweg' AS richtung, seq, node, cost, agg_cost
FROM tsp(ARRAY[▯, ▯, ▯, ▯, ▯, ▯], ▯, ▯)

ORDER BY richtung, seq;
```

Im Rückweg tauscht ihr `start_id` und `end_id` – fahrt also von der Hundewiese zurück nach Zuhause.

---

## 10. Fahrzeit & €/h berechnen

*(Folgt in Kürze)*

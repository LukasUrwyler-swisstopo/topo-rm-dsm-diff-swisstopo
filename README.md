# DSM-DIFF

> [!NOTE]
> Dieses Repository ist ein Wegweiser. Der Quellcode wird in der swisstopo-Organisation weiterentwickelt:
>
> **https://github.com/swisstopo/topo-rm-dsm-diff**
>
> Das Repository ist privat. Zugriff haben Mitglieder der swisstopo GitHub-Organisation.


<img width="1110" height="1127" alt="image" src="https://github.com/user-attachments/assets/a433e020-8ff2-47b2-a99e-2d4d4dc4805f" />


## Worum es geht

Desktop-Tool mit grafischer Oberfläche, das aus zwei digitalen Oberflächenmodellen ein **Differenzmodell** rechnet: **DSM aktuell minus DSM alt**. Das Ergebnis ist ein Cloud Optimized GeoTIFF, das auf Wunsch direkt als Asset ins bestehende STAC-Item der Befliegung publiziert wird (Collection `ch.swisstopo.spezialbefliegungen`).

Typische Anwendung sind Rapid Mapping und Monitoring: Wo hat sich die Oberfläche nach einem Murgang, Felssturz oder über einen Gletschersommer verändert?

Als Referenz dient standardmässig **swissSURFACE3D Raster**. Das Tool sucht die passenden Kacheln selbst im swisstopo-STAC, lädt sie und setzt sie zu einem Mosaik zusammen. Alternativ lässt sich ein eigenes DSM als Referenz angeben.

Eine Volumenberechnung gibt es bewusst nicht. Ein Volumen hängt am Perimeter und kann leicht falsch definiert werden. Wer eines braucht, rechnet es mit dem Diff und dem eigenen Polygon.

## Funktionen

- **Eingabe lokal oder per STAC-Link:** DSM aktuell und DSM alt können als Datei oder als Link auf ein STAC-Asset angegeben werden. Bei einem Link werden Metadaten wie Gebiet, Kamerasystem und Befliegungslinien automatisch übernommen.
- **swissSURFACE3D automatisch:** Kachelsuche über die Ausdehnung von DSM aktuell, Wahl des jüngsten Jahrgangs **vor** dem Aufnahmejahr, Download mit Prüfsumme und Cache, Mosaik als COG.
- **Konsequente Prüfungen vor dem ersten Schreiben:** ein Band, LV95, nicht rotiert, Einheit Meter, gleicher Höhenbezug, Überlappung. Ein DTM statt DSM wird abgelehnt.
- **Wählbare Auflösung** des Diff: 1 m (Standard) oder 0.5 m.
- **Publikation via STAC** in die Umgebung INT oder PROD, mit Multipart-Upload und Prüfsummen-Kontrolle. Ein bestehendes Item wird nie verändert, nur das Diff-Asset kommt dazu.
- **Nur-Import** eines bereits gerechneten Diff, falls der Upload separat nachgeholt werden muss.

## Ablauf

```
DSM aktuell ─────────────┐
                         │   Prüfungen: LV95, 1 Band, Meter, Höhenbezug, Überlappung
DSM alt                  │
(swissSURFACE3D via STAC │
 oder eigenes DSM) ──────┤
                         ▼
          gleiches Gitter? sonst Resampling (bilinear / average, mit Warnung)
                         ▼
          dh = DSM aktuell − DSM alt, blockweise
                         ▼
          optional auf 1 m mitteln (flächengewichtet)
                         ▼
          Metadaten + Statistik → COG → Prüfung → Ablage
                         ▼
          optional: STAC-Asset anlegen → Upload → Kontrolle
```

## Fachliche Konventionen

| Thema | Regel |
|---|---|
| Vorzeichen | **dh = DSM aktuell − DSM alt**, negativ = Abtrag bzw. Verlust |
| Gitter | Gerechnet wird im Gitter von DSM aktuell. Bei gleicher Auflösung liegt das Diff pixelgenau auf dem DSM. |
| Mitteln | Zuerst die Differenz, dann mitteln. So stammt jede Zelle nur aus Pixeln, die in beiden DSM gültig sind. |
| Höhenbezug | Beide DSM müssen denselben Höhenbezug haben (LN02 bzw. LHN95). Sonst Abbruch, denn der Unterschied liegt bei 6 bis 30 cm. |
| Raumbezug | Diff in **EPSG:2056** ohne Höhenbezug, weil dh eine Höhendifferenz ist und keine Höhe. Mosaik der Referenz in EPSG:2056+5728 (LV95 + LN02). |
| Umprojektion | nie stillschweigend. Ein horizontaler Bezug ausser LV95 führt zum Abbruch. |
| COG | DEFLATE mit Floating-Point-Predictor, Overviews, NoData bitgenau, alle Metadaten im TIFF selbst |

### Höhengenauigkeit und Nachweisgrenze

Das Tool gibt zu jedem Diff die Genauigkeit an. Aus der Höhengenauigkeit beider DSM (1σ) folgt bei unabhängigen Fehlern:

```
σ_dh  = √(σ_aktuell² + σ_alt²)
LoD95 = 1.96 · σ_dh
```

Änderungen kleiner als LoD95 lassen sich statistisch nicht vom Rauschen unterscheiden. Beim Mitteln auf 1 m bleibt die Angabe bewusst konservativ: Die Fehler photogrammetrischer DSM sind räumlich stark korreliert, systematische Anteile wie ein Höhenversatz zwischen zwei Befliegungen verschwinden durch Mitteln nicht.

## Technik

- **Python** mit **tkinter**-GUI. Die Verarbeitung läuft getrennt davon in der OSGeo4W- bzw. QGIS-Python-Umgebung.
- **GDAL** und **NumPy** für die Rasterrechnung, **requests** für STAC-Suche, Download und Upload.
- STAC-API von swisstopo (data.geo.admin.ch).
- Raumbezug **EPSG:2056** (CH1903+ / LV95), Höhen **LN02** (EPSG:5728) bzw. **LHN95** (EPSG:5729).
- Tests mit pytest, synthetisch und ohne Netz: Gitter, NoData, Versatz, Höhenbezug-Konflikte, COG-Eigenschaften, gemockte STAC-Abläufe.

## Autor

Lukas Urwyler, swisstopo (Bundesamt für Landestopografie)

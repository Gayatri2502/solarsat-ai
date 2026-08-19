# Floating Solar Site Detection — NDWI Methodology
### Google Earth Engine pipeline for waterbody suitability scoring
SolarSat AI · Technical Methodology Doc v1.0

---

## 1. Objective

Identify waterbodies (reservoirs, dams, large tanks) that stay **wet year-round**, sit
**close to grid infrastructure**, and have **low ecological/navigation conflict** — the
three preconditions for a floating solar (FPV) feasibility study. Everything below runs
on free Sentinel-2 imagery and public GIS layers; no site visit or sensor is required
for this screening stage.

The output of this pipeline is exactly what feeds the `waterbodies[]` suitability
scores shown in the dashboard (`permanence`, `gridProx`, `eco`, `conflict`, `suit`).

---

## 2. Core index: NDWI (McFeeters, 1996)

```
NDWI = (Green − NIR) / (Green + NIR)
```

For Sentinel-2 Level-2A surface reflectance:
- Green = **B3** (560 nm, 10 m)
- NIR = **B8** (842 nm, 10 m)

Water reflects strongly in green and absorbs in NIR, so open water pixels return
**NDWI > 0** (commonly thresholded at **0.2–0.3** to reject wet soil / turbid edges),
while vegetation and dry land return negative values.

> Note: MNDWI (using SWIR band B11 instead of NIR) is more robust against
> built-up/shadow false positives and is used as a cross-check in §5.

---

## 3. Step 1 — Cloud-masked Sentinel-2 collection

```javascript
// Google Earth Engine JavaScript API

function maskS2clouds(image) {
  var scl = image.select('SCL');
  // SCL: 3=cloud shadow, 8/9=cloud medium/high prob, 10=thin cirrus, 11=snow
  var mask = scl.neq(3).and(scl.neq(8)).and(scl.neq(9))
                .and(scl.neq(10)).and(scl.neq(11));
  return image.updateMask(mask).divide(10000)
              .copyProperties(image, ['system:time_start']);
}

var s2 = ee.ImageCollection('COPERNICUS/S2_SR_HARMONIZED')
  .filterBounds(waterbodyAOI)
  .filterDate('2023-01-01', '2024-12-31')
  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 20))
  .map(maskS2clouds);
```

`waterbodyAOI` is an `ee.Geometry` (or `ee.FeatureCollection` buffer) drawn around
each candidate reservoir — sourced from the **India-WRIS** reservoir boundary layer
or the **JRC Global Surface Water** max-extent polygon (both free) so the AOI is not
hand-drawn per site.

---

## 4. Step 2 — Per-image NDWI and monthly water mask

```javascript
function addNDWI(image) {
  var ndwi = image.normalizedDifference(['B3', 'B8']).rename('NDWI');
  return image.addBands(ndwi);
}

var s2ndwi = s2.map(addNDWI);

// Monthly composite (median) to smooth residual cloud noise
var months = ee.List.sequence(0, 23); // 24 months
var monthlyWaterMasks = ee.ImageCollection.fromImages(
  months.map(function(m) {
    var start = ee.Date('2023-01-01').advance(m, 'month');
    var end = start.advance(1, 'month');
    var comp = s2ndwi.filterDate(start, end).select('NDWI').median();
    var waterMask = comp.gt(0.25).rename('water').set('month', start);
    return waterMask;
  })
);
```

---

## 5. Step 3 — Water Permanence Score

For each pixel, permanence is simply *how often it was classified as water* across
the 24 monthly composites:

```javascript
var permanenceRaster = monthlyWaterMasks.sum()
  .divide(monthlyWaterMasks.size())   // fraction 0–1
  .rename('permanence');

// Site-level permanence = mean permanence over the reservoir's max-extent polygon
var permanenceStat = permanenceRaster.reduceRegion({
  reducer: ee.Reducer.mean(),
  geometry: waterbodyAOI,
  scale: 10,
  maxPixels: 1e9
});

var permanenceScore = ee.Number(permanenceStat.get('permanence')).multiply(100);
```

- **`permanence = 97%`** (e.g. Indira Sagar) → the reservoir footprint is water in
  97% of monthly composites → effectively a perennial, stable waterbody → strong FPV
  candidate.
- A reservoir that halves in area every dry season will show permanence in the
  60–75% range on the *max-extent* polygon — flagged **Under Review**, not
  disqualified outright, since anchoring can be designed for the *stable-core* zone
  only (see §7).

Cross-check with **MNDWI** (`(Green−SWIR)/(Green+SWIR)`, using B3/B11) on the same
composites; if the two indices disagree by more than 15 percentage points on a tile,
that tile is flagged for manual QA (usually shadow from terrain or a cloud-mask miss).

---

## 6. Step 4 — Stable-core polygon (where panels can actually float)

```javascript
var stableCore = permanenceRaster.gte(0.90).selfMask();

var stableCoreVector = stableCore.reduceToVectors({
  geometry: waterbodyAOI,
  scale: 10,
  geometryType: 'polygon',
  eightConnected: true,
  maxPixels: 1e9
});

var stableCoreAreaKm2 = stableCoreVector.geometry().area().divide(1e6);
```

This is the polygon actually used for the **potential MW** estimate — not the full
max-extent area — because floating platforms need to sit on water for 12 months,
not just during monsoon.

```
potential_MW = stable_core_area_km² × 100 (ha/km²) × coverage_fraction / hectares_per_MW
```
with `coverage_fraction = 0.015` (1.5%, a conservative non-navigational cap used
across Indian FPV pilots) and `hectares_per_MW ≈ 1.7` for crystalline-silicon FPV.

---

## 7. Step 5 — Grid proximity, ecological, and conflict layers

These three factors are vector overlay operations, not raster classification:

```javascript
// Grid proximity — distance from AOI centroid to nearest substation node
var substations = ee.FeatureCollection('projects/your-project/assets/osm_substations_IN');
var nearest = substations.map(function(f) {
  return f.set('dist', f.geometry().distance(waterbodyAOI.centroid()));
}).sort('dist').first();
var gridDistKm = ee.Number(nearest.get('dist')).divide(1000);
var gridProxScore = ee.Number(100).subtract(gridDistKm.multiply(2)).max(0).min(100);

// Ecological sensitivity — overlap with protected area / wetland buffers
var protectedAreas = ee.FeatureCollection('WCMC/WDPA/current/polygons')
  .filterBounds(waterbodyAOI.buffer(5000));
var ecoOverlapArea = protectedAreas.geometry().intersection(waterbodyAOI, 1)
  .area().divide(waterbodyAOI.area());
var ecoScore = ee.Number(100).subtract(ee.Number(ecoOverlapArea).multiply(100));

// Navigation / irrigation conflict — overlap with command-area polygons
var commandAreas = ee.FeatureCollection('projects/your-project/assets/irrigation_command_areas');
var conflictOverlap = commandAreas.filterBounds(waterbodyAOI)
  .geometry().intersection(waterbodyAOI, 1).area().divide(waterbodyAOI.area());
var conflictScore = ee.Number(100).subtract(ee.Number(conflictOverlap).multiply(100));
```

`WCMC/WDPA` (World Database on Protected Areas) is a public GEE-hosted dataset.
Substation nodes and irrigation command-area polygons are ingested once from
OpenStreetMap (Overpass API export) and state irrigation department open-data
portals, then uploaded as a GEE asset — this is the only one-time manual ingestion
step in the whole pipeline.

---

## 8. Step 6 — Composite Suitability Score

```javascript
var suitability = permanenceScore.multiply(0.40)
  .add(gridProxScore.multiply(0.25))
  .add(ecoScore.multiply(0.20))
  .add(conflictScore.multiply(0.15));
```

| Factor | Weight | Rationale |
|---|---|---|
| Water Permanence | 40% | Hard constraint — unstable water defeats the purpose of FPV |
| Grid Proximity | 25% | Directly drives interconnection cost (largest single line-item in an FPV DPR) |
| Ecological Suitability | 20% | Protected-area overlap can block clearance entirely |
| Navigation/Irrigation Conflict | 15% | Determines usable surface fraction, not a hard blocker |

Bands used by the dashboard: **≥75 → Recommended**, **50–74 → Under Review**,
**<50 → Not Suitable**.

---

## 9. Step 7 — Batch export across all candidate waterbodies

```javascript
var reservoirList = ee.FeatureCollection('projects/your-project/assets/india_wris_reservoirs');

var results = reservoirList.map(function(feature) {
  var aoi = feature.geometry();
  // ... repeat steps 3–8 with aoi in place of waterbodyAOI ...
  return feature.set({
    permanence: permanenceScore,
    stable_core_km2: stableCoreAreaKm2,
    potential_mw: potentialMW,
    grid_proximity: gridProxScore,
    eco_score: ecoScore,
    conflict_score: conflictScore,
    suitability: suitability
  });
});

Export.table.toDrive({
  collection: results,
  description: 'floating_solar_national_scan',
  fileFormat: 'GeoJSON'
});
```

This GeoJSON is what the FastAPI ingestion job (`/pipelines/floating-solar-scan`)
picks up, loads into `PostGIS` (`waterbody_candidates` table), and serves to the
dashboard's Floating Solar view.

---

## 10. Python equivalent (for the Airflow/FastAPI batch job)

```python
import ee
ee.Initialize(project="your-project")

def water_permanence(aoi: ee.Geometry, start="2023-01-01", end="2024-12-31") -> float:
    def mask_clouds(img):
        scl = img.select("SCL")
        mask = scl.remap([3,8,9,10,11],[0,0,0,0,0], 1)
        return img.updateMask(mask).divide(10000)

    s2 = (ee.ImageCollection("COPERNICUS/S2_SR_HARMONIZED")
          .filterBounds(aoi).filterDate(start, end)
          .filter(ee.Filter.lt("CLOUDY_PIXEL_PERCENTAGE", 20))
          .map(mask_clouds))

    def ndwi_water(img):
        ndwi = img.normalizedDifference(["B3", "B8"])
        return ndwi.gt(0.25).rename("water").set("system:time_start", img.get("system:time_start"))

    monthly = (s2.map(ndwi_water)
               .map(lambda i: i.set("month", ee.Date(i.get("system:time_start")).format("YYYY-MM")))
               .reduceColumns  # grouped monthly median composited upstream in practice
               )
    permanence_img = s2.map(ndwi_water).mean()  # simplified: fraction of water-classified passes
    stat = permanence_img.reduceRegion(ee.Reducer.mean(), aoi, 10, maxPixels=1e9)
    return stat.getInfo().get("water", 0) * 100
```

*(Production job groups by calendar month before averaging, exactly as in the
JS version — the Python snippet above is the readable core of that logic.)*

---

## 11. Validation approach

- **Ground truth**: cross-check `stable_core_km2` against India-WRIS live storage
  bulletins (published reservoir water-spread area) for a 20-reservoir sample —
  target agreement within ±8%.
- **Temporal robustness**: re-run on a held-out year (2021–2022) and confirm
  permanence scores are stable within ±5 points for reservoirs not affected by
  drought declarations.
- **False-positive check**: MNDWI cross-validation (§5) catches terrain-shadow and
  cloud-mask leakage before a site is surfaced as "Recommended."

---

## 12. Refresh cadence

- Sentinel-2 revisit: **5 days** (2-satellite constellation) → new monthly composite
  as soon as ≥3 clear passes are available for a given AOI/month.
- Full national re-scan (all reservoirs): **monthly**, batched overnight.
- Grid/ecological/conflict layers: refreshed **quarterly** (these change far slower
  than water extent) or on-demand when a new substation asset is commissioned.

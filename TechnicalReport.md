# Flood Risk Index (FRI) Mapping — Lagos, Nigeria (April–June 2025)

## 1. Introduction
This report outlines the technical workflow, data sources, and analytical methods used to generate a Flood Risk Index (FRI) map for Lagos, Nigeria. The study integrates satellite rainfall estimates, elevation, and terrain slope data between **April and June 2025**, producing a composite flood risk surface designed for environmental and defense applications.

---

## 2. Data Sources

| Dataset | Source | Description | Spatial Resolution | Projection |
|----------|---------|--------------|--------------------|-------------|
| CHIRPS Daily Rainfall | `UCSB-CHG/CHIRPS/DAILY` | Daily precipitation estimates (mm/day) | ~5 km | EPSG:4326 |
| SRTM DEM | `USGS/SRTMGL1_003` | Elevation model from NASA Shuttle Radar Topography Mission | 30 m | EPSG:4326 |
| Derived Slope | Derived from SRTM DEM | Slope in degrees | 30 m | EPSG:4326 |
| Hillshade | Derived from SRTM DEM | Relief visualization for cartographic enhancement | 30 m | EPSG:4326 |
| HydroRivers | `WWF/HydroSHEDS/v1/FreeFlowingRivers` | Vector river network | Vector | EPSG:4326 |

All layers were projected to **UTM Zone 31N (EPSG:32631)** and clipped to the Lagos administrative boundary.

---

## 3. GEE Export Parameters

**Drive folder:** `GEE_Floodgate`  
**Max pixels:** `1e13`  

### Export Commands (Google Earth Engine)

```javascript
// --- Export CHIRPS (≈5 km) ---
Export.image.toDrive({
  image: chirps_total,
  description: 'CHIRPS_total_AprJun2025_Lagos',
  folder: 'GEE_Floodgate',
  fileNamePrefix: 'chirps_total_20250401_20250630_lagos',
  region: roi,
  scale: 5000,
  crs: 'EPSG:4326',
  fileFormat: 'GeoTIFF',
  maxPixels: 1e13
});

// --- Export DEM (30 m) ---
Export.image.toDrive({
  image: dem,
  description: 'SRTM_DEM_Lagos',
  folder: 'GEE_Floodgate',
  fileNamePrefix: 'srtm_dem_lagos',
  region: roi,
  scale: 30,
  crs: 'EPSG:4326',
  fileFormat: 'GeoTIFF',
  maxPixels: 1e13
});

// --- Export Slope (30 m) ---
Export.image.toDrive({
  image: slope,
  description: 'SRTM_Slope_Lagos',
  folder: 'GEE_Floodgate',
  fileNamePrefix: 'srtm_slope_lagos',
  region: roi,
  scale: 30,
  crs: 'EPSG:4326',
  fileFormat: 'GeoTIFF',
  maxPixels: 1e13
});

// --- Export Hillshade (30 m) ---
Export.image.toDrive({
  image: hillshade,
  description: 'Hillshade_Lagos',
  folder: 'GEE_Floodgate',
  fileNamePrefix: 'hillshade_lagos',
  region: roi,
  scale: 30,
  crs: 'EPSG:4326',
  fileFormat: 'GeoTIFF'
});

// --- Export river features (SHP) ---
Export.table.toDrive({
  collection: rivers,
  description: 'Lagos_Rivers_Export',
  folder: 'GEE_Floodgate',
  fileNamePrefix: 'lagos_hydrorivers',
  fileFormat: 'SHP'
});

// --- Export bounding box (GeoJSON) ---
var bbox_fc = ee.FeatureCollection([ee.Feature(roi_bbox)]);
Export.table.toDrive({
  collection: bbox_fc,
  description: 'Lagos_BBox_Export',
  folder: 'GEE_Floodgate',
  fileNamePrefix: 'lagos_bbox',
  fileFormat: 'GeoJSON'
});
```

## 4. Methodology

### 4.1 Data Preprocessing
All raster layers were:
- Clipped to the Lagos Region of Interest (ROI)
- Reprojected to EPSG:32631 (UTM Zone 31N)
- Aligned to ensure consistent grid and cell size
- Converted to raster datasets in GeoTIFF format for subsequent analysis
### 4.2 Normalization
Each input variable was normalized to a 0–1 scale to ensure comparability:
```python
def normalize(arr):
    valid = arr.where(~np.isnan(arr), drop=True)
    vmin = float(valid.min())
    vmax = float(valid.max())
    norm = (arr - vmin) / (vmax - vmin)
    return norm.clip(0, 1)
```
This produced:
- rain_n → normalized rainfall intensity
- elev_n → normalized elevation
- slope_n → normalized slope gradient
### 4.3 FRI Calculation
The Flood Risk Index (FRI) integrates the normalized layers through weighted aggregation:

$$FRI = (0.5×Rain_n​) + (0.35×(1−Elev_n​)) + (0.15×Slope_factor)$$

Where:
- Rainfall contributes most to flood potential (50%)
- Low elevation increases risk (hence, 1 - Elev_n)
- Slope accounts for surface runoff dynamics (15%)
The resulting raster was saved as:
```
/content/drive/MyDrive/GEE_Floodgate/processed/fri_utm_repaired.tif
```
### 4.4 Corridor Thresholds
Two buffer zones were established to assess potential impact corridors:
- ≤500 m — high-risk influence zone
- ≤1000 m — moderate influence zone

These serve as planning thresholds for logistics and emergency response assessments.

## 5. Assumptions
Although no explicit modeling assumptions were stated, several were implicit:
1. **Static Terrain:** Terrain and slope from SRTM (2000) are assumed to accurately represent current conditions.
2. **Uniform Weighting Validity:** Weights (0.5, 0.35, 0.15) are assumed representative of Lagos’ hydro-geomorphic context.
3. **Temporal Aggregation:** Rainfall was aggregated over April–June 2025, assuming spatial uniformity of cumulative precipitation.
4. **Hydrological Linearity:** Relationships among rainfall, slope, and elevation are assumed additive and linearly scalable.

## 6. Limitations
1. **Temporal Resolution:** The analysis uses a single 3-month window, not accounting for intra-seasonal rainfall variation or antecedent soil moisture.
2. **DEM Age and Artifacts:** SRTM data (30 m, 2000) may not reflect updated urban morphology or infrastructure changes affecting runoff.
3. **Hydrological Simplification:** The FRI model omits drainage capacity, land cover, and impermeable surfaces — critical urban flood determinants.
4. **Weight Subjectivity:** Chosen weights lack empirical calibration or validation against observed flood events.
5. **Data Knowledge Gap:** Limited subject expertise restricted advanced geostatistical or hydrological modeling (e.g., HEC-RAS, SWAT).

## 7. Outputs
| Layer | Description | File Path |
|-------|-------------|-----------|
| `fri_utm_repaired.tif` | Final normalized flood risk index | `/processed/fri_utm_repaired.tif`|
| `srtm_dem_lagos_utm.tif` |	Elevation layer (DEM)	| `/processed/srtm_dem_lagos_utm.tif` |
| `srtm_slope_lagos_utm.tif`	| Slope layer |	`/processed/srtm_slope_lagos_utm.tif` |
| `hillshade_lagos_utm.tif`	| Hillshade for visualization	| `/processed/hillshade_lagos_utm.tif` |
| `lagos_hydrorivers_rprjctd.shp` |	Reprojected river network	| `/processed/lagos_hydrorivers_rprjctd.shp` |

## 8. Summary
This workflow demonstrates an applied flood risk mapping approach for Lagos using **GEE, Python (rioxarray, rasterio, geopandas, matplotlib)**, and open data. The output FRI serves as an initial screening tool for identifying areas of elevated flood susceptibility. Future iterations can incorporate **land use, drainage density, soil type, and temporal flood validation** for improved accuracy.

---
**Author:** Favour Adebayo

**Date:** October 2025

**Projection:** UTM Zone 31N (EPSG:32631)

**Data Source:** GEE & SRTM (2025)

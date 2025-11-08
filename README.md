# 🌍 Project Floodgate: Mapping Flood Risk Zones in Lagos, Nigeria Using Open Data

This repository contains the workflow, datasets, and scripts used to build a Flood Risk Index (FRI) for Lagos State, Nigeria using Google Earth Engine (GEE) and Python (Colab). The project integrates satellite rainfall (CHIRPS) with terrain (SRTM) and hydrography (HydroSHEDS) to identify flood-prone corridors for the April–June 2025 rainy season.

## Project Structure
```
GEE_Floodgate/
├── raw/                # Raw exports from GEE (original GeoTIFFs, SHP, GeoJSON)
├── processed/          # Reprojected and cleaned rasters and vectors
├── maps/               # Final map exports (PNG/PDF)
├── flood_corridors/    # Corridor rasters and vector outputs (500m, 1000m)
├── notebooks/          # Colab notebooks (data_collection.ipynb, analysis.ipynb, cartography_and_QA.ipynb)
├── README.md           # This file
└── TechnicalReport.md  # Full report on flood risk
```
## Summary
- Study area: Lagos State, Nigeria
- Date window: 2025-04-01 → 2025-06-30
- Projection used for analysis & outputs: EPSG:32631 (UTM Zone 31N)
- Key deliverables: `fri_utm_repaired.tif`, corridor rasters (500 m, 1000 m), `lagos_hydrorivers_rprjctd.shp`, maps (PDF/PNG), notebooks, and a short analytic brief.

## Data Sources & GEE Assets
| Layer                                                 | GEE asset / collection                |
| ----------------------------------------------------- | ------------------------------------- |
| CHIRPS daily precipitation (raw)                      | `UCSB-CHG/CHIRPS/DAILY`               |
| SRTM DEM                                              | `USGS/SRTMGL1_003`                    |
| Hydrography (HydroSHEDS freeflowing rivers)           | `WWF/HydroSHEDS/v1/FreeFlowingRivers` |
| Note: slope & hillshade are derived from SRTM in GEE. | —                                     |

### Exact GEE export commands used

All exports were executed from the Earth Engine Code Editor and exported to Google Drive in folder `GEE_Floodgate`. Replace `roi` with your Lagos geometry variable.
```javascript
var driveFolder = 'GEE_Floodgate';
var maxPixels = 1e13;

// CHIRPS seasonal total (Apr–Jun 2025)
Export.image.toDrive({
  image: chirps_total,
  description: 'CHIRPS_total_AprJun2025_Lagos',
  folder: driveFolder,
  fileNamePrefix: 'chirps_total_20250401_20250630_lagos',
  region: roi,
  scale: 5000,
  crs: 'EPSG:4326',
  fileFormat: 'GeoTIFF',
  maxPixels: maxPixels
});

// SRTM DEM (30 m)
Export.image.toDrive({
  image: dem,
  description: 'SRTM_DEM_Lagos',
  folder: driveFolder,
  fileNamePrefix: 'srtm_dem_lagos',
  region: roi,
  scale: 30,
  crs: 'EPSG:4326',
  fileFormat: 'GeoTIFF',
  maxPixels: maxPixels
});

// Slope (derived)
Export.image.toDrive({
  image: slope,
  description: 'SRTM_Slope_Lagos',
  folder: driveFolder,
  fileNamePrefix: 'srtm_slope_lagos',
  region: roi,
  scale: 30,
  crs: 'EPSG:4326',
  fileFormat: 'GeoTIFF',
  maxPixels: maxPixels
});

// Hillshade (derived)
Export.image.toDrive({
  image: hillshade,
  description: 'Hillshade_Lagos',
  folder: driveFolder,
  fileNamePrefix: 'hillshade_lagos',
  region: roi,
  scale: 30,
  crs: 'EPSG:4326',
  fileFormat: 'GeoTIFF',
  maxPixels: maxPixels
});

// Rivers (vector)
Export.table.toDrive({
  collection: rivers,
  description: 'Lagos_Rivers_Export',
  folder: driveFolder,
  fileNamePrefix: 'lagos_hydrorivers',
  fileFormat: 'SHP'
});

// Bounding box export (GeoJSON)
var bbox_fc = ee.FeatureCollection([ee.Feature(roi_bbox)]);
Export.table.toDrive({
  collection: bbox_fc,
  description: 'Lagos_BBox_Export',
  folder: driveFolder,
  fileNamePrefix: 'lagos_bbox',
  fileFormat: 'GeoJSON'
});
```
### Local environment (Colab) & dependencies
Work was done in Google Colab (Python 3.10). Install the required libraries with:
```
# Run in a Colab cell (or in your local environment)
!pip install geopandas rasterio rioxarray matplotlib shapely numpy scipy contextily
```
Key Python libraries used (versions may vary):
- `rioxarray`, `rasterio` — raster I/O, reprojection
- `geopandas` — vector I/O and reprojection
- `numpy`, `scipy` — array math and distance transforms
- `matplotlib` — plotting and map export
- `shapely` — geometry operations

## Processing Steps (Colab / Python)
### 1. Mount Drive
```python
from google.colab import drive
drive.mount('/content/drive')
```
### 2. Paths
Put GEE exports under:
```python
/content/drive/MyDrive/GEE_Floodgate/raw/
```
After processing, save cleaned outputs to:
```python
/content/drive/MyDrive/GEE_Floodgate/processed/
```
### 3. Reproject DEM → UTM and use DEM as reference grid
- Auto-select UTM zone from DEM centroid (Lagos → EPSG:32631).
- Reproject CHIRPS, slope, hillshade to match DEM grid via bilinear resampling.
### 4. Normalize layers to 0–1 (min–max normalization)
```python
def normalize(arr):
    valid = arr.where(~np.isnan(arr), drop=True)
    vmin = float(valid.min())
    vmax = float(valid.max())
    norm = (arr - vmin) / (vmax - vmin)
    return norm.clip(0, 1)
```
### 5. FRI formula
```python
w1, w2, w3 = 0.5, 0.35, 0.15
fri = (w1 * rain_n) + (w2 * (1 - elev_n)) + (w3 * slope_factor)
fri = fri.clip(min=0, max=1)
fri.rio.to_raster('/content/drive/MyDrive/GEE_Floodgate/processed/fri_utm_repaired.tif')
```
### 6. River reprojection and raserization
```python
# Reproject if needed
if rivers.crs != fri_crs:
    rivers = rivers.to_crs(fri_crs)

# Rasterize to match FRI transform and shape

from rasterio import features
river_raster = features.rasterize(((geom,1) for geom in rivers.geometry),
                                  out_shape=(height, width),
                                  transform=fri_transform, dtype='uint8')
```
### 7. Distance transform and corridors
```python
from scipy.ndimage import distance_transform_edt
dist_pixels = distance_transform_edt(1 - river_raster)
dist_meters = dist_pixels * pixel_size_m
corridor_500 = dist_meters <= 500
corridor_1000 = dist_meters <= 1000
```
### 8. Combine corridors with high FRI mask
- High FRI threshold: 75th percentile (example: ~0.631)
`high_corridor_1000 = (fri >= fri_thresh) & corridor_1000`
### 9. Export corridor rasters & vectorize
Save rasters as GeoTIFF and polygons as GeoJSON for mapping.

## Data Cleaning and repair snippets
Reproject rivers to match the FRI CRS:
```python
if rivers.crs != fri_crs:
    rivers = rivers.to_crs(fri_crs)
    print("✅ Rivers reprojected to:", rivers.crs)
```
Repair small NaN gaps in the FRI raster using array mean (conservative gap-fill):
```python
fri_filled = fri.copy()
mask_nan = fri.isnull()
fri_filled.values[mask_nan.values] = np.nanmean(fri.values)
fri_repaired = fri_filled.fillna(0).rio.write_crs(dem.rio.crs)
fri_repaired_fp = '/content/drive/MyDrive/GEE_Floodgate/processed/fri_utm_repaired.tif'
fri_repaired.rio.to_raster(fri_repaired_fp)
```

## Key outputs (paths)
- `/content/drive/MyDrive/GEE_Floodgate/raw/chirps_total_20250401_20250630_lagos.tif`
- `/content/drive/MyDrive/GEE_Floodgate/raw/srtm_dem_lagos.tif`
- `/content/drive/MyDrive/GEE_Floodgate/raw/srtm_slope_lagos.tif`
- `/content/drive/MyDrive/GEE_Floodgate/raw/hillshade_lagos.tif`
- `/content/drive/MyDrive/GEE_Floodgate/raw/lagos_hydrorivers.* (shp set)`
- `/content/drive/MyDrive/GEE_Floodgate/processed/fri_utm_repaired.tif`
- `/content/drive/MyDrive/GEE_Floodgate/flood_corridors/fri_highrisk_corridor_500m.tif`
- `/content/drive/MyDrive/GEE_Floodgate/flood_corridors/fri_highrisk_corridor_1000m.tif`
- `/content/drive/MyDrive/GEE_Floodgate/maps/fri_map_lagos.png`

## Reproducibility Tips & Caveats
- **CRS consistency:* Always reproject vectors to the raster CRS before rasterization. For Lagos, use EPSG:32631.
- **CHIRPS scale:** CHIRPS is coarse (~5 km). Resampling to 30 m increases visual detail but not underlying native rainfall resolution. Interpret rainfall-driven signals cautiously at fine scales.
- **Large exports:** GEE may reject very large single exports; tile exports if necessary (2×2 or 3×3 tiling).-
- **Memory considerations:** Avoid loading extremely large rasters fully into memory; use windowed reads or downsample for exploratory visualization.

## Notes on Assumptions & Limitations
**Assumptions inferred from the workflow**
- The weighted linear combination (FRI) approximates relative flood susceptibility, not physical inundation depth.
- HydroRIVERS is treated as a reliable representation of perennial channels; small urban drainage networks and culverts are not captured.
- CHIRPS provides reasonable spatial rainfall patterns at the scale of analysis but not micro-urban storms.

**Main limitations**
- No hydrodynamic (2D/1D) modeling; this is a screening-level risk surface.
- No local calibration with observed flood events or gauge data.
- Urban drainage and land-cover imperviousness are not explicitly modelled.
- DEM may contain urban artifacts and is dated (SRTM ~2000).

## Suggested Next Steps (for higher fidelity)
- **Hydrodynamic modeling** (HEC-RAS, LISFLOOD-FP) for depth and inundation timing.
- **Incorporate land cover / imperviousness** (e.g., GHSL, Sentinel) to model urban runoff.
- **Use Sentinel-1 SAR** to validate recent flood extents and tune thresholds.
- **Calibrate** weights and thresholds with historical flood observations or insurance/NGO datasets.
- **Produce LGA-level summaries** for targeted planning.

## Contact & Authorship

**Author:** Favour Adebayo

**LinkedIn:** https://www.linkedin.com/in/kayeneii

**Date:** October 2025

## License
This repository and the derived products are released for research and educational use. Cite data sources (CHIRPS, SRTM, HydroSHEDS) when reusing results.

# Batch-Chlorophyll-Gapfiller
This Python script automates the batch processing of satellite chlorophyll-a (chl-a) scenes to create "seamless" gap-filled maps. It is designed for researchers who need continuous chlorophyll data, even in areas with data gaps from clouds, glint, or other artifacts.

The script reads a summary CSV of L2 file paths. For each scene, it loads the chlor_a and l2_flags data, masks land, and then interpolates missing water pixels using **Ordinary Kriging (OK)**. If OK fails or is not available, it falls back to **Inverse Distance Weighting (IDW)**.

Finally, it saves multiple outputs for each scene—including PNGs, GeoTIFFs, and a new NetCDF file—and compiles a diagnostics CSV for the entire batch.

## Overview
The main workflow is as follows:

1. Read a master list of L2 NetCDF file paths from a user-provided ```SUMMARY_CSV```.

2. For each file in the list:

  - Load the ```chlor_a``` variable and ```l2_flags``` (to create a land mask).

  - Compute a seamless, gap-filled ```chlor_a``` map by applying Ordinary Kriging (OK) in log-space to all missing water pixels.

  - If ```pykrige``` is not installed or fails, use a ```scipy.cKDTree``` IDW fallback.

  - Extract diagnostic statistics (e.g., 3x3, 5x5, 10x10 mean/median) for the filled map around a central ```TARGET_LAT/TARGET_LON```.

3. Save all image and data outputs to a single ```OUTPUT_DIR```.

4. Append the diagnostic stats for the processed scene to a new summary CSV (e.g., ```LISCO_krige_stats_v3.csv```) in the ```OUTPUT_DIR```.

## Key Settings
```
# ============================== SETTINGS ==============================
SITE        = "LISCO"
BASE_DIR    = Path("/Volumes/purkislab2a/") / "Mingyue" / SITE
SUMMARY_CSV = BASE_DIR / f"{SITE}_l2_summary_v2.csv"

# All outputs go here
OUTPUT_DIR  = BASE_DIR / "krige_outputs_3"   # <— change if you like

# Map Windows drive(s) to macOS mount(s)
# Use this if CSV paths were made on Windows (e.g., "Y:\\...")
# but you are running on macOS/Linux (e.g., "/Volumes/purkislab2a")
DRIVE_MAP   = {"Y:": Path("/Volumes/purkislab2a")}

# L2 variable
GROUP       = "geophysical_data"
VARNAME     = "chlor_a"

# Diagnostics target (used if CSV lacks nearest_row/col)
TARGET_LAT  = 40.954517
TARGET_LON  = -73.341767

# Seamless/global-OK knobs (same as old script)
N_CLOSEST_POINTS   = 50
VARIOGRAM_MODEL    = "exponential"
MAX_GLOBAL_OBS     = 120_000
```
## Outputs
All outputs for the entire batch run are saved in the single ```OUTPUT_DIR```.

- **PNG Images:** For quick visualization of each scene.

  - ```{base}_orig_masked.png```: The original ```chlor_a``` data with land masked.

  - ```{base}_filled.png```: The final gap-filled ```chlor_a``` map.

  - ```{base}_where_filled.png```: A black-and-white mask showing which pixels were interpolated (white = filled).

  - ```{base}_confidence.png```: The interpolation confidence (inverted kriging variance).

- **GeoTIFFs** (Optional):

  -If the scene is on a rectilinear grid and ```rasterio``` is installed, the script will save a GeoTIFF version of all four PNGs (e.g., ```{base}_filled.tif```).

- **NetCDF Data** (Optional):

  - ```{base}_seamless.nc```: This is the main data product. It is a full copy of the original L2 NetCDF file, but with new variables added to the ```geophysical_data``` group:

    - ```chlor_a_filled```: The gap-filled data array.

    - ```where_filled```: The 0/1 mask of interpolated pixels.

    - ```confidence```: The confidence/variance map.

  - Note: This file is only created if the ```netCDF4``` library is installed.

- **Final Summary CSV**:

  - ```{SITE}_krige_stats_v3.csv```: A single CSV summarizing the entire batch. It contains one row for each processed scene, with columns for:

    - ```file_raw```, ```file_local```

    - ```center_value_mg_m3``` (at the target lat/lon)

    - ```center_confidence```

    - ```krige3x3_mean```, ```krige3x3_median```

    - ```krige5x5_mean```, ```krige5x5_median```

    - ```nearest_obs_dist_m``` (distance from target to nearest original pixel)

    - ```insitu_value``` (if present in the input CSV)

    - ```abs_k_vs_is``` (absolute difference: kriged 3x3 mean vs. in-situ value)

# Landsat 8 Surface Reflectance & NDVI Pipeline

A small end-to-end pipeline that takes a raw Landsat 8 Level-1 scene and processes it to surface reflectance, then computes NDVI. I built it to get hands-on with the parts of the chain that usually sit hidden behind analysis-ready products: radiometric rescaling, the sun-angle correction, and atmospheric correction with 6S.
Scene used: `LC08_L1TP_141045`, path/row 141/045 over Odisha, April 2026, projected in UTM zone 44N.

# What it does

The flow is DN → TOA reflectance → surface reflectance → NDVI.
-Load the bands. All nine OLI bands are read with rasterio and fill pixels (DN = 0) are masked to NaN so they don't pollute the statistics.
-Read the MTL. The rescaling coefficients and sun elevation are parsed out of the MTL XML rather than hardcoded, so the same code runs on any Landsat 8/9 scene    without editing numbers.
-Atmospheric parameters. 6S is run per band (via Py6S) for the scene's actual solar geometry to get path reflectance, gas transmittance and spherical albedo. If 6S isn't installed, the code falls back to tabulated standard-atmosphere values and says so.
-Correction. DN is rescaled to TOA reflectance and divided by sin(sun elevation), then inverted to surface reflectance using the 6S parameters.
-NDVI from corrected NIR and red, plus a three-panel figure (raw DN, surface reflectance, NDVI).

# Atmospheric correction — the honest version

The 6S run uses a mid-latitude summer atmosphere, a continental aerosol model, and AOT 0.2 at 550 nm. Those are reasonable defaults but they're a predefined profile, not measurements for this date and place. A more rigorous correction would pull the real aerosol optical depth and water vapour (AERONET or a MODIS/VIIRS aerosol product) and feed those into 6S. I've kept the assumptions explicit in the notebook rather than hiding them.
The output isn't validated against a reference product. The notebook includes a magnitude spot-check, but the proper test is a pixel-wise comparison against the USGS Collection 2 Level-2 surface reflectance for the same scene. That's the next thing I'd add.

# Running it

The notebook is built for Google Colab. 6S needs a compiled Fortran binary plus the Py6S wrapper, which on Colab is easiest through conda:
```python
!pip install -q condacolab
import condacolab; condacolab.install()      # restarts the runtime
```
After the restart:
```python
!conda install -c conda-forge sixs py6s -y
import sys
!{sys.executable} -m pip install Py6S          # install into the kernel's Python
```
Then upload the `Data` folder (the nine band TIFs and the MTL XML) into `/content/Data` and run the pipeline cells top to bottom. Without 6S installed it still runs, just on the fallback atmosphere.
One gotcha worth noting: after condacolab swaps the environment, a plain `pip install` can land in a different Python than the kernel, so the import fails even though pip reported success. Pointing pip at `sys.executable` fixes it.

# Results

For this scene the NDVI mean is about 0.29, consistent with a mix of vegetated and bare/built surfaces rather than dense crop canopy. Surface reflectance sits sensibly below TOA across all bands, with the biggest drop in the blue bands where atmospheric scattering is strongest.

# Stack

Python, rasterio, NumPy, Py6S / 6S, Matplotlib.

Notes / limitations

Predefined atmosphere, not scene-measured AOD/water vapour.
No geometric/terrain correction beyond what's already in the L1TP product.
Surface reflectance not yet validated against the official L2 product.
The fallback lookup table is an approximation for standard conditions and is labelled as such in the code.

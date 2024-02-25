# Lidar-with-pdal

In this tutorial, we explore PDAL package to process lidar point clouds. The goal is to create a pseudo NDVI image for a region using 1) a lidar las file, 2) an RGB imagery tiff file. The two input files have been provided for an optional exercise of "Advanced Remote Sensing Using Lidar" course at University of Maryland. 

The lidar intensity values represents a "pseudo" Near-Infrared reflectance (NIR) value. It is not a true NIR reflectance since it is affected by factors like, reflectance of the object, scan angle, return number, and pulse wavelength. However, as a relative measure of NIR reflectance, we can still utilize lidar NIR in conjunction with imagery data to generate a "pseudo" NDVI image. The NDVI is defined as:  
NDVI = (NIR-RED) /(NIR+ RED).   

To solve this, we develop two raster files representing NIR and RED for the study area, and finally we can performa raster calculation to compute NDVI. 

### Spatial reference of lidar dataset:
We can get the crs of a las file using [laspy](https://laspy.readthedocs.io) package, which is a powerful python package for data processing of lidar dataset. 
```python
import laspy

las_file = "Data/MD_Baltimore_2008_000030.las"
las = laspy.read(las_file)
crs = las.header.parse_crs()
print(crs)

```
EPSG : 26985

### PDAL package
[PDAL](https://pdal.io/en/) is an open source package for processing point cloud data. [Here](https://pdal.io/en/2.6.0/workshop/index.html) presents a series of PDAL examples. Here, we use PDAL for most of our lidar data analysis. 

### Spatial resolution of rasters
To create a raster file from lidar point clouds, it is recommneded to set raster resolution to be roughly 4 times of the average point spacing of lidar dataset. To get average point spacing of our las file, we can use below PDAL command through a python code:
```python
import json
import subprocess

las_file = "Data/MD_Baltimore_2008_000030.las"
pdal_command = ["pdal", "info", "%s" % las_file, "--boundary"]
result = subprocess.run(pdal_command, capture_output=True, text=True)
results_dict = json.loads(result.stdout)
print(f"avg_pt_spacing:{round(results_dict['boundary']['avg_pt_spacing'],3)} m")
```
And the results is avg_pt_spacing: 2.869 m. So, we use cell size of 10m, which is almost 4 times of average point spacing.



### Step 1: Develop NIR (Intensity) raster
To create a more reliable intensity image, it's best to filter the lidar points to select only first returns and no overlap. We used below PDAL pipeline within a python code to:
-  Read the las file
-  Select only points with return number one, and only classifications of 1 and 2 (This las file has only classification 1, 2 and 12, which represents overlaps)
-  Make a raster file with cell size of 10 using mean values of intensity in each cell
```python
import json

import pdal

pdal_pipeline = {
    "pipeline": [
        {  # read las file
            "type": "readers.las",
            "filename": "Data/MD_Baltimore_2008_000030.las",
            "spatialreference": "EPSG:26985",
        },
        {  # filter data
            "type": "filters.range",
            "limits": "returnNumber[1:1],Classification[1:2]",
        },
        {  # create raster from filter points using intensity mean values
            "type": "writers.gdal",
            "filename": "intensity_img_pdal.tiff",
            "dimension": "intensity",
            "output_type": "mean",
            "resolution": 10,
            "nodata": -999,
            "data_type": "float32",
        },
    ]
}
# Create a PDAL pipeline object
pipeline = pdal.Pipeline(json.dumps(pdal_pipeline))
# Execute the pipeline
pipeline.execute()

```

We can use below code to plot the intensity raster created above by PDAL:
```python
import matplotlib.pyplot as plt
import numpy as np
import rasterio

intensity_dataset = rasterio.open("intensity_img_pdal.tiff")
no_data = intensity_dataset.nodata
intensity_array = intensity_dataset.read(1)
intensity_array = np.where(intensity_array == no_data, np.nan, intensity_array)
plt.imshow(intensity_array, cmap="gray")
plt.colorbar(label="Intensity")
plt.title("Intensity map")
plt.axis("off")
plt.show()
```
![image](https://github.com/AliForghani/Lidar-with-pdal/assets/22843733/d68c4cd1-080f-442b-8a06-e973828b87e0)

### Step 2: Develop Red raster
Currently, our lidar las file does not have Red band data needed for the calculation of NDVI. Here we aim to get Red data from the imagery dataset by adding RGB values from the image tiff file to the lidar point records.
Below is a PDAL pipeline that reads las file, filter data (for desired return number and classification), add RGB into the las file, and make a raster file with cell size of 10 using mean values of Red in each cell.

```python
import json

import pdal

pdal_pipeline = {
    "pipeline": [
        {  # read las file
            "type": "readers.las",
            "filename": "Data/MD_Baltimore_2008_000030.las",
            "spatialreference": "EPSG:26985",
        },
        {  # filter data
            "type": "filters.range",
            "limits": "returnNumber[1:1],Classification[1:2]",
        },
        {  # add RGB from tif file
            "type": "filters.colorization",
            "raster": "Data/2s-1w.tif",
            # The format of each dimension is <name>:<band_number>:<scale_factor>
            "dimensions": "Red:1:1.0, Green:2:1.0, Blue:3:1.0",
        },
        {  # create raster from filter points using Red mean values
            "type": "writers.gdal",
            "filename": "red_img_pdal.tiff",
            "dimension": "Red",
            "output_type": "mean",
            "resolution": 10,
            "nodata": -999,
            "data_type": "float32",
        },
    ]
}
# Create a PDAL pipeline object
pipeline = pdal.Pipeline(json.dumps(pdal_pipeline))
# Execute the pipeline
pipeline.execute()
```

Similarly, we use below code to plot the Red raster:

```python
import matplotlib.pyplot as plt
import numpy as np
import rasterio

red_dataset = rasterio.open("red_img_pdal.tiff")
no_data = red_dataset.nodata
red_array = red_dataset.read(1)
red_array = np.where(red_array == no_data, np.nan, red_array)
plt.imshow(red_array, cmap="Reds")
plt.colorbar(label="Red")
plt.title("Red map")
plt.axis("off")
plt.show()
```

![image](https://github.com/AliForghani/Lidar-with-pdal/assets/22843733/253f1be4-e92f-4728-a1a2-3ef421bd039b)


### Step 3: Calculate the "pseudo" NDVI
Finally, we use below python code to calculate NDVI as NDVI = (NIR-RED) /(NIR+ RED), plot it and save the results into a new raster file. 
```python
import matplotlib.pyplot as plt
import numpy as np
import rasterio
from matplotlib.colors import LinearSegmentedColormap


# read intensity and Red data...record nodata
NIR_dataset = rasterio.open("intensity_img_pdal.tiff")
no_data = NIR_dataset.nodata
NIR = NIR_dataset.read(1)
Red = rasterio.open("red_img_pdal.tiff").read(1)
# build NDVI array considering nodata
NDVI = np.where(
    ((NIR == no_data) | (Red == no_data)), np.nan, (NIR - Red) / (NIR + Red)
)
# Define a colormap for -1 as red and 1 as green
color_points = {
    "red": [(0.0, 1.0, 1.0), (0.5, 1.0, 1.0), (1.0, 0.0, 0.0)],
    "green": [(0.0, 0.0, 0.0), (0.5, 1.0, 1.0), (1.0, 1.0, 1.0)],
    "blue": [(0.0, 0.0, 0.0), (0.5, 0.0, 0.0), (1.0, 0.0, 0.0)],
}
# Create colormap
custom_cmap = LinearSegmentedColormap("CustomColormap", color_points)
plt.imshow(NDVI, cmap=custom_cmap)
plt.colorbar(label="NDVI Value")
plt.title("Pseudo NDVI")
plt.axis("off")
plt.show()
# save the NDVI in a new raster file
with rasterio.open(
    "NDVI.tiff",
    "w",
    driver="GTiff",
    height=NIR_dataset.height,
    width=NIR_dataset.width,
    dtype=NIR_dataset.dtypes[0],
    count=1,  # Number of bands
    crs=NIR_dataset.crs,
    transform=NIR_dataset.transform,
    nodata=NIR_dataset.nodata,
) as dst:
    dst.write(NDVI, 1)
```
![photo_2024-02-25_00-15-44](https://github.com/AliForghani/Lidar-with-pdal/assets/22843733/b79d8906-a3b0-477a-afcf-2a6835c54757)







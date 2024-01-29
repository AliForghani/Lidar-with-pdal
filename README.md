# Lidar-with-pdal

In this tutorial, we explore PDAL package to process lidar point clouds. The goal is to create an NDVI image for a region using 1) a lidar las file, 2) an RGB imagery tif file.  
The two input files have been provided for an optional exercise of "Advanced Remote Sensing Using Lidar" course at University of Maryland. 

The lidar intensity values represents a "pseudo" NIR reflectance value. It is not a true NIR reflectance since it is affected by factors like,the reflectance of the object, scan angle, the point return number, and pulse wavelength. However, we can still use it as a relative measure of NIR reflectance and use it to create a "pseudo" NDVI image using imagery data. NDVI = NIR-RED /(NIR+ RED). Here, we need to develop two raster files representing NIR and RED. 

### PDAL package
PDAL is an open source package for processing point cloud data. More info: https://pdal.io/en/. See the link below for a series of PDAL examples:
https://pdal.io/en/2.6.0/workshop/index.html

### Get Spatial Reference of lidar ataset:
We can get the crs of a las file using laspy package as shown below: 
```python
import laspy

las_path = ("Data/MD_Baltimore_2008_000030.las",)
las = laspy.read(las_path)
crs = las.header.parse_crs()
print(crs)

```
EPSG:26985

### Spatial resolution of rasters
It is recommneded to set raster resolution to be roughly 4 times of the average point spacing. To get point spacing of our las file, we can use below pdal command through a python code:
```python
import json
import subprocess

las_file = "Results/lasclassify_output.las"
pdal_command = ["pdal", "info", "%s" % las_file, "--boundary"]
result = subprocess.run(pdal_command, capture_output=True, text=True)
results_dict = json.loads(result.stdout)
print(
    f"{las_file} file: avg_pt_spacing:{round(results_dict['boundary']['avg_pt_spacing'],3)}m"
```
And the results is “avg_pt_spacing": 2.869m”. So we use cell size of 10ft, which is almost 4 times of average point spacing.



### Step 1: Develop NIR (Intensity) raster
To get a good intensity image, it's best to filter the lidar points to select only first returns and no overlap. We used below PDAL pipeline within a python code to achieve this. 
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

Finally, we use below code to plot the intensity raster:
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
Currently, our lidar dataset does not have Red band data to be used for calculation of NDVI. Here we aim to get Red data from the imagery dataset. To do that we need to add RGB values from the image to the lidar point records.
Below is a PDALpipeline that reads las file, filter data (for desired return number and classification), add RGB, and save into a new las file.

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
        {  # save into a new las file
            "type": "writers.las",
            "filename": "MD_Baltimore_2008_000030_withRGB_pdal.las",
        },
    ]
}
# Create a PDAL pipeline object
pipeline = pdal.Pipeline(json.dumps(pdal_pipeline))
# Execute the pipeline
pipeline.execute()
```
Then, we use PDAL as below to make a new raster file with cell size of 10ft for the Red dimension:

```python
import json

import pdal


def make_red_raster():
    pdal_pipeline = {
        "pipeline": [
            {  # read las file
                "type": "readers.las",
                "filename": "MD_Baltimore_2008_000030_withRGB_pdal.las",
                "spatialreference": "EPSG:26985",
            },
            {  # create raster from filter points using intensity mean values
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
Finally, use below python code to calculate NDVI as NDVI = NIR-RED /(NIR+ RED), plot it and save the results into a new raster file. 
```python
import matplotlib.pyplot as plt
import numpy as np
import rasterio
from matplotlib.colors import LinearSegmentedColormap


def calculate_ndvi():
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
    plt.title("NDVI Using open-source")
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
![image](https://github.com/AliForghani/Lidar-with-pdal/assets/22843733/467055ac-8213-49f1-8888-4a5a52a7f71a)


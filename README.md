# Land-Use-Change-Detection-in-Brandenburg

This project analyzes land use changes, particularly water body changes, in the Brandenburg region using remote sensing data from Landsat 8 and Sentinel-2. The analysis is conducted using Google Earth Engine (GEE).

## Project Structure
- `data/`: Directory for input and output data files.
- `scripts/`: Contains the GEE scripts for analysis and visualization.
- `results/`: Stores the output visualizations and summary report.

## Usage
1. Clone the repository.
2. Add your GEE credentials.
3. Run the scripts in the `scripts/` directory using GEE.

```
// Define the study area: Brandenburg state boundary
var brandenburg = ee.FeatureCollection('FAO/GAUL_SIMPLIFIED_500m/2015/level2')
                   .filter(ee.Filter.eq('ADM1_NAME', 'Brandenburg'))
                   .geometry();

// Function to mask clouds in Landsat 8 Collection 2
function maskL8sr(image) {
  var cloudShadowBitMask = (1 << 4);
  var cloudsBitMask = (1 << 3);
  var qa = image.select('QA_PIXEL');
  var mask = qa.bitwiseAnd(cloudShadowBitMask).eq(0)
                   .and(qa.bitwiseAnd(cloudsBitMask).eq(0));
  return image.updateMask(mask).multiply(0.0000275).add(-0.2);
}

// Load Landsat 8 Collection 2 surface reflectance data for 2015
var landsat2015 = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')
                    .filterDate('2015-01-01', '2015-12-31')
                    .filterBounds(brandenburg)
                    .filter(ee.Filter.lt('CLOUD_COVER', 20))
                    .map(maskL8sr)
                    .median()
                    .clip(brandenburg);

// Print available bands for Landsat 8 Collection 2
print('Landsat 8 bands:', landsat2015.bandNames());

// Function to mask clouds using the quality band of Sentinel-2 data
function maskS2clouds(image) {
  var qa = image.select('QA60');
  var cloudBitMask = 1 << 10;
  var cirrusBitMask = 1 << 11;
  var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
                  .and(qa.bitwiseAnd(cirrusBitMask).eq(0));
  return image.updateMask(mask).divide(10000);
}

// Load Sentinel-2 surface reflectance data for 2023
var sentinel2023 = ee.ImageCollection('COPERNICUS/S2_SR')
                     .filterDate('2023-01-01', '2023-12-31')
                     .filterBounds(brandenburg)
                     .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 20))
                     .map(maskS2clouds)
                     .median()
                     .clip(brandenburg);

// Print available bands for Sentinel-2
print('Sentinel-2 bands:', sentinel2023.bandNames());

// Define a function to calculate the Normalized Difference Water Index (NDWI) for Landsat 8
function addNDWI_L8(image) {
  return image.addBands(image.normalizedDifference(['SR_B3', 'SR_B5']).rename('NDWI'));
}

// Define a function to calculate the Normalized Difference Water Index (NDWI) for Sentinel-2
function addNDWI_S2(image) {
  return image.addBands(image.normalizedDifference(['B3', 'B8']).rename('NDWI'));
}

// Add NDWI to the images
var ndwi2015 = addNDWI_L8(landsat2015).select('NDWI');
var ndwi2023 = addNDWI_S2(sentinel2023).select('NDWI');

// Define a threshold to classify water and non-water
var waterThreshold = 0.3;

// Create water/non-water masks for both years
var water2015 = ndwi2015.gt(0.05);
var water2023 = ndwi2023.gt(0.05);

// Define pixelArea
var pixelArea = ee.Image.pixelArea().divide(10000); // Pixel area in hectares

// Calculate the area of water in 2015
var waterArea2015 = water2015.multiply(pixelArea).reduceRegion({
  reducer: ee.Reducer.sum(),
  geometry: brandenburg,
  scale: 30,
  maxPixels: 1e9
}).get('NDWI');

// Calculate the area of water in 2023
var waterArea2023 = water2023.multiply(pixelArea).reduceRegion({
  reducer: ee.Reducer.sum(),
  geometry: brandenburg,
  scale: 30,
  maxPixels: 1e9
}).get('NDWI');

// Print the water area for both years
print('Water area in 2015 (hectares):', waterArea2015);
print('Water area in 2023 (hectares):', waterArea2023);

// Calculate land use change from non-water to water
var landUseChangeToWater = water2023.and(water2015.not());

// Calculate the area of land use change to water in hectares
var landUseChangeToWaterArea = landUseChangeToWater.multiply(pixelArea).reduceRegion({
  reducer: ee.Reducer.sum(),
  geometry: brandenburg,
  scale: 30,
  maxPixels: 1e9
}).get('NDWI');

// Print the area of land use change to water
print('Area of land use change to water (hectares):', landUseChangeToWaterArea);

// Visualization
Map.centerObject(brandenburg, 8);
var water2015Vis = water2015.updateMask(water2015).visualize({palette: ['blue'], min: 0, max: 1});
var water2023Vis = water2023.updateMask(water2023).visualize({palette: ['cyan'], min: 0, max: 1});
var landUseChangeToWaterVis = landUseChangeToWater.updateMask(landUseChangeToWater)
                                                  .visualize({palette: ['red'], min: 0, max: 1});
var ndwi2015Vis = ndwi2015.visualize({min: -1, max: 1, palette: ['00FFFF', '0000FF']});
var ndwi2023Vis = ndwi2023.visualize({min: -1, max: 1, palette: ['00FFFF', '0000FF']});

Map.addLayer(brandenburg, {}, 'Brandenburg Boundary');
Map.addLayer(ndwi2015Vis, {}, 'NDWI 2015');
Map.addLayer(ndwi2023Vis, {}, 'NDWI 2023');
Map.addLayer(water2015Vis, {}, 'Water 2015');
Map.addLayer(water2023Vis, {}, 'Water 2023');
Map.addLayer(landUseChangeToWaterVis, {}, 'Land Use Change to Water 2015-2023');

// Export the results to Google Drive
Export.image.toDrive({
  image: ee.Image().paint(brandenburg, 0, 2),
  description: 'Brandenburg_Boundary',
  folder: 'EarthEngineExports',
  fileNamePrefix: 'Brandenburg_Boundary',
  region: brandenburg,
  scale: 30,
  crs: 'EPSG:4326'
});

Export.image.toDrive({
  image: ndwi2015,
  description: 'NDWI_2015',
  folder: 'EarthEngineExports',
  fileNamePrefix: 'NDWI_2015',
  region: brandenburg,
  scale: 30,
  crs: 'EPSG:4326'
});

Export.image.toDrive({
  image: ndwi2023,
  description: 'NDWI_2023',
  folder: 'EarthEngineExports',
  fileNamePrefix: 'NDWI_2023',
  region: brandenburg,
  scale: 30,
  crs: 'EPSG:4326'
});

Export.image.toDrive({
  image: water2015,
  description: 'Water_2015',
  folder: 'EarthEngineExports',
  fileNamePrefix: 'Water_2015',
  region: brandenburg,
  scale: 30,
  crs: 'EPSG:4326'
});

Export.image.toDrive({
  image: water2023,
  description: 'Water_2023',
  folder: 'EarthEngineExports',
  fileNamePrefix: 'Water_2023',
  region: brandenburg,
  scale: 30,
  crs: 'EPSG:4326'
});

Export.image.toDrive({
  image: landUseChangeToWater,
  description: 'Land_Use_Change_to_Water',
  folder: 'EarthEngineExports',
  fileNamePrefix: 'Land_Use_Change_to_Water',
  region: brandenburg,
  scale: 30,
  crs: 'EPSG:4326'
});
```

![image](https://github.com/KonlavachMengsuwan/-Land-Use-Change-Detection-in-Brandenburg/assets/52453368/87837910-b101-4600-be0a-4319cb5e632d)

Water 2015
![image](https://github.com/KonlavachMengsuwan/-Land-Use-Change-Detection-in-Brandenburg/assets/52453368/9bf46660-afb9-44a2-98ed-f7b41cfc7a94)

Water 2023
![image](https://github.com/KonlavachMengsuwan/-Land-Use-Change-Detection-in-Brandenburg/assets/52453368/27be27d1-49e0-45dd-849e-f32a04816038)

Land Use Change Detection in Brandenburg
![image](https://github.com/KonlavachMengsuwan/-Land-Use-Change-Detection-in-Brandenburg/assets/52453368/9277d927-a332-4ffb-b5f7-568e0863e4bf)

https://code.earthengine.google.com/39543b84e71e1e3f1018e1a412f3458e

## License
This project is licensed under the MIT License.

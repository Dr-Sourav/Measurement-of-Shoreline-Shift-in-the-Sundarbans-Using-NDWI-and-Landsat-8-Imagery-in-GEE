// Shoreline Change Detection using Google Earth Engine

// Load your study area asset
var studyArea = ee.FeatureCollection('projects/ee-sourav16787/assets/sundarban1');

// Cloud mask for Landsat 8 SR
function maskL8sr(image) {
  var cloudShadowBitMask = 1 << 3;
  var cloudsBitMask = 1 << 5;
  var qa = image.select('QA_PIXEL');
  var mask = qa.bitwiseAnd(cloudShadowBitMask).eq(0)
               .and(qa.bitwiseAnd(cloudsBitMask).eq(0));
  return image.updateMask(mask);
}

// NDWI calculation using Green (SR_B3) and SWIR1 (SR_B6)
function calculateNDWI(image) {
  return image.normalizedDifference(['SR_B3', 'SR_B6']).rename('NDWI');
}

// Create shoreline mask from NDWI
function getShorelineMask(image, threshold) {
  var ndwi = calculateNDWI(image).clip(studyArea);
  return ndwi.gt(threshold).rename('water');
}

// Convert water mask to vector shoreline
function vectorizeShoreline(mask) {
  var withDummy = mask.addBands(ee.Image.constant(1));
  return withDummy.reduceToVectors({
    geometry: studyArea.geometry(),
    scale: 30,
    geometryType: 'polygon',
    eightConnected: false,
    labelProperty: 'label',
    reducer: ee.Reducer.first(),
    maxPixels: 1e8,
    bestEffort: true
  });
}

// Threshold for NDWI
var ndwiThreshold = 0.0;

// Load Landsat 8 collections for 2015 and 2020
var collection2015 = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')
  .filterDate('2015-01-01', '2015-12-31')
  .filterBounds(studyArea)
  .filter(ee.Filter.lt('CLOUD_COVER', 20))
  .map(maskL8sr);

var collection2020 = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')
  .filterDate('2020-01-01', '2020-12-31')
  .filterBounds(studyArea)
  .filter(ee.Filter.lt('CLOUD_COVER', 20))
  .map(maskL8sr);

// Median composites
var composite2015 = collection2015.median().clip(studyArea);
var composite2020 = collection2020.median().clip(studyArea);

// NDWI-based binary water mask
var shorelineMask2015 = getShorelineMask(composite2015, ndwiThreshold);
var shorelineMask2020 = getShorelineMask(composite2020, ndwiThreshold);

// Convert masks to vector polygons
var shoreline2015 = vectorizeShoreline(shorelineMask2015);
var shoreline2020 = vectorizeShoreline(shorelineMask2020);

// Define your measurement point (update coordinates if needed)
var point = ee.Geometry.Point([88.635, 21.945]);

// Find the closest polygon from 2015 shoreline
var nearest2015 = shoreline2015
  .map(function(f) {
    return f.set('distance', f.geometry().distance(point, 1));  // with maxError
  })
  .sort('distance')
  .first();

// Find the closest polygon from 2020 shoreline
var nearest2020 = shoreline2020
  .map(function(f) {
    return f.set('distance', f.geometry().distance(point, 1));  // with maxError
  })
  .sort('distance')
  .first();

// Use centroids of nearest shoreline polygons as approximation of shoreline positions
var point2015 = ee.Feature(nearest2015).geometry().centroid(1);
var point2020 = ee.Feature(nearest2020).geometry().centroid(1);

// Create a line showing the shift
var shiftLine = ee.Geometry.LineString([point2015.coordinates(), point2020.coordinates()]);

// Measure the distance between centroids (shoreline shift)
var shiftDistance = point2015.distance(point2020, 1);  // in meters with maxError

// ---------------------------------
// 🗺️ Map Display
// ---------------------------------
Map.centerObject(point, 14);
Map.addLayer(point, {color: 'green'}, 'Reference Point');
Map.addLayer(shoreline2015.style({color: 'blue', width: 2}), {}, 'Shoreline 2015');
Map.addLayer(shoreline2020.style({color: 'red', width: 2}), {}, 'Shoreline 2020');
Map.addLayer(point2015, {color: 'blue'}, 'Centroid 2015');
Map.addLayer(point2020, {color: 'red'}, 'Centroid 2020');
Map.addLayer(shiftLine, {color: 'orange'}, 'Shoreline Shift');

// 📏 Print result
print('Shoreline Shift Distance at Point (meters):', shiftDistance);

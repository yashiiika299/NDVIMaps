var india = ee.FeatureCollection("FAO/GAUL/2015/level0")
  .filter(ee.Filter.eq('ADM0_NAME', 'India'));

Map.centerObject(india, 5);

var s2 = ee.ImageCollection('COPERNICUS/S2_SR_HARMONIZED')
  .filterBounds(india)
  .filterDate('2024-01-01', '2024-12-31')
  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 10));

function maskS2clouds(image) {
  var qa = image.select('QA60');
  var cloudBitMask = 1 << 10;
  var cirrusBitMask = 1 << 11;

  var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
    .and(qa.bitwiseAnd(cirrusBitMask).eq(0));

  return image.updateMask(mask).divide(10000);
}

var cleanCollection = s2.map(maskS2clouds);

var composite = cleanCollection.median().clip(india);

// NDVI = (B8 - B4) / (B8 + B4)
var ndvi = composite.normalizedDifference(['B8', 'B4']).rename('NDVI');

var ndviVis = {
  min: 0,
  max: 1,
  palette: ['red', 'yellow', 'green']
};

Map.addLayer(ndvi, ndviVis, 'NDVI');

var degraded = ndvi.expression(
  "(b('NDVI') < 0.1) ? 1" +
  ": (b('NDVI') < 0.3) ? 2" +
  ": 3"
).rename('Degraded_Land');

var degradedVis = {
  min: 1,
  max: 3,
  palette: ['red', 'yellow', 'green']
};

Map.addLayer(degraded, degradedVis, 'Degraded Land');

var lonGrid = ee.List.sequence(68, 97, 5);
var latGrid = ee.List.sequence(6, 37, 5);

var tiles = lonGrid.map(function(lon) {
  return latGrid.map(function(lat) {
    var tile = ee.Geometry.Rectangle([
      lon,
      lat,
      ee.Number(lon).add(5),
      ee.Number(lat).add(5)
    ]);
    return ee.Feature(tile);
  });
}).flatten();

var tileFC = ee.FeatureCollection(tiles);

Map.addLayer(tileFC, {}, 'Tiles');

var histogram = ui.Chart.image.histogram({
  image: ndvi,
  region: india.geometry(),
  scale: 1000,
  maxPixels: 1e13
});

print(histogram);

var areaImage = ee.Image.pixelArea().divide(1000000);

var degradedArea = areaImage.updateMask(ndvi.lt(0.1));

var degradedStats = degradedArea.reduceRegion({
  reducer: ee.Reducer.sum(),
  geometry: india.geometry(),
  scale: 1000,
  maxPixels: 1e13
});

print('Degraded Area (sq km):', degradedStats);

var healthyArea = areaImage.updateMask(ndvi.gt(0.3));

var healthyStats = healthyArea.reduceRegion({
  reducer: ee.Reducer.sum(),
  geometry: india.geometry(),
  scale: 1000,
  maxPixels: 1e13
});

print('Healthy Area (sq km):', healthyStats);

var legend = ui.Panel({
  style: {position: 'bottom-left', padding: '8px'}
});

legend.add(ui.Label('NDVI'));

function addLegend(color, name) {
  var row = ui.Panel({layout: ui.Panel.Layout.Flow('horizontal')});

  var box = ui.Label({
    style: {backgroundColor: color, padding: '8px'}
  });

  var label = ui.Label(name);

  row.add(box);
  row.add(label);

  legend.add(row);
}

addLegend('red', 'Low');
addLegend('yellow', 'Medium');
addLegend('green', 'High');

Map.add(legend);

Map.onClick(function(coords) {
  var point = ee.Geometry.Point(coords.lon, coords.lat);

  var value = ndvi.reduceRegion({
    reducer: ee.Reducer.first(),
    geometry: point,
    scale: 10
  });

  value.evaluate(function(val) {
    print('NDVI:', val);
  });
});

Export.image.toDrive({
  image: ndvi,
  description: 'India_NDVI',
  folder: 'GEE_Output',
  region: india.geometry(),
  scale: 10,
  maxPixels: 1e13
});

Export.image.toDrive({
  image: degraded,
  description: 'India_Degraded',
  folder: 'GEE_Output',
  region: india.geometry(),
  scale: 10,
  maxPixels: 1e13
});

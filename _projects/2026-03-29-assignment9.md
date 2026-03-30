---
title: "Assignment 9"
layout: single
author_profile: true
excerpt: "See details of the assignment 9"
---

### 1. Code link of Assessment of Post-Flood Recovery in Sylhet district of Bangladesh Using VIIRS Nighttime Lights and Socio-Economic Data

- Code link [this is the link of the post flood recovery of Sylhet district](https://code.earthengine.google.com/9236463d55d481023ac0084abb442b5e)

### 2. Map of flooded areas in Sylhet in 2017
![flooded areas](/images/flood.png)

### 3. Night time light data comaparison map
![ntl](/images/ntl.png)

### 4. The report on "Flood Detection and Post-Flood Recovery Assessment in Sylhet District, Bangladesh Using Sentinel-1 SAR and Night-Time Light Data"

[pdf](https://oishi-12.github.io/files/Flood Detection and Post.pdf)

### 5. Manual code of Flood Detection and Post-Flood Recovery Assessment in Sylhet District, Bangladesh Using Sentinel-1 SAR and Night-Time Light Data
-----
```Go
var countries = ee.FeatureCollection("FAO/GAUL/2015/level2");
var studyArea = countries.filter(ee.Filter.and(
  ee.Filter.eq('ADM2_NAME', 'Sylhet')));

Map.centerObject(studyArea, 9);
Map.addLayer(studyArea, {color: 'grey'}, 'Study Area (Sylhet)');


var collectionVV = ee.ImageCollection('COPERNICUS/S1_GRD')
  .filter(ee.Filter.eq('instrumentMode', 'IW'))
  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))
  .filter(ee.Filter.eq('orbitProperties_pass', 'DESCENDING'))
  .filterMetadata('resolution_meters', 'equals', 10)
  .filterBounds(studyArea)
  .select('VV');

var collectionVH = ee.ImageCollection('COPERNICUS/S1_GRD')
  .filter(ee.Filter.eq('instrumentMode', 'IW'))
  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VH'))
  .filter(ee.Filter.eq('orbitProperties_pass', 'DESCENDING'))
  .filterMetadata('resolution_meters', 'equals', 10)
  .filterBounds(studyArea)
  .select('VH');

print(collectionVV, 'VV Collection');
print(collectionVH, 'VH Collection');

var beforeVV = collectionVV.filterDate('2017-05-01', '2017-05-31').max().clip(studyArea);
var afterVV = collectionVV.filterDate('2017-07-01', '2017-07-31').max().clip(studyArea);
var beforeVH = collectionVH.filterDate('2017-05-01', '2017-05-31').max().clip(studyArea);
var afterVH = collectionVH.filterDate('2017-07-01', '2017-07-31').max().clip(studyArea);

var SMOOTHING_RADIUS = 50; // meters
beforeVV = beforeVV.focal_mean(SMOOTHING_RADIUS, 'circle', 'meters');
beforeVH = beforeVH.focal_mean(SMOOTHING_RADIUS, 'circle', 'meters');
afterVV = afterVV.focal_mean(SMOOTHING_RADIUS, 'circle', 'meters');
afterVH = afterVH.focal_mean(SMOOTHING_RADIUS, 'circle', 'meters');

var ratioVH = afterVH.divide(beforeVH);
var ratioVV = afterVV.divide(beforeVV);

var floodedCombined = ratioVH.gt(1.25).and(ratioVV.gt(1.2));
var floodedMasked = floodedCombined.updateMask(floodedCombined);

Map.addLayer(floodedMasked, {palette: "0000FF"}, 'Flooded Areas VV+VH', 1);
Map.addLayer(floodedCombined, {}, 'combined Flooded Areas VV+VH');
Map.addLayer(ratioVH, {min:0, max:2}, 'VH Ratio', 0);
Map.addLayer(ratioVV, {min:0, max:2}, 'VV Ratio', 0);

var pixelArea = ee.Image.pixelArea();
var floodedPixelArea = pixelArea.updateMask(floodedMasked);
var floodedArea = floodedPixelArea.reduceRegion({
  reducer: ee.Reducer.sum(),
  geometry: studyArea,
  scale: 10,
  maxPixels: 1e13
});

var floodedArea_m2 = floodedArea.get('area');
var floodedArea_km2 = ee.Number(floodedArea_m2).divide(1e6);
print('Total Flooded Area (m²):', floodedArea_m2);
print('Total Flooded Area (km²):', floodedArea_km2);

var ntlCol = ee.ImageCollection("NASA/VIIRS/002/VNP46A2")
  .select('Gap_Filled_DNB_BRDF_Corrected_NTL');

function getMeanNTL(start, end) {
  return ntlCol.filterDate(start, end).mean().clip(studyArea);
}

var pre  = getMeanNTL('2017-05-01', '2017-05-31'); 
var post = getMeanNTL('2017-10-01', '2017-10-31'); 
var rec  = getMeanNTL('2017-11-01', '2017-12-30'); 

Map.addLayer(pre, {}, "pre flood ntl")
Map.addLayer(post, {}, "post flood ntl")
Map.addLayer(rec, {}, "rec flood ntl")

var ntlRecovery = rec.subtract(post)
  .divide(pre.add(0.1)) 
  .rename('NTL_Recovery');



var pop = ee.ImageCollection("WorldPop/GP/100m/pop").filterDate('2017-01-01', '2017-12-31').mean();
var popDensity = pop.rename('PopDensity').clip(studyArea);

var rwi = ee.FeatureCollection("projects/sat-io/open-datasets/facebook/relative_wealth_index")
  .filterBounds(studyArea);
  
Map.addLayer(rwi, {}, "rwi")
var ghsl = ee.ImageCollection("JRC/GHSL/P2023A/GHS_BUILT_C").first();
var roads = ghsl.select('built_characteristics').eq(5);
var roadDensity = roads.reduceNeighborhood({
  reducer: ee.Reducer.mean(),
  kernel: ee.Kernel.circle(500, 'meters'),
}).rename('RoadDensity').clip(studyArea);


var stack = ntlRecovery
  .addBands(popDensity)
  .addBands(roadDensity);

var regressionData = stack.sampleRegions({
  collection: rwi,       
  properties: ['rwi'],    
  scale: 100,            
  geometries: true      
}).map(function(f) {
  return f.set('constant', 1);
}).filter(ee.Filter.notNull(['NTL_Recovery', 'PopDensity', 'rwi', 'RoadDensity'])); 

print(regressionData)

var result = regressionData.reduceColumns({
  reducer: ee.Reducer.linearRegression({numX: 4, numY: 1}),
  selectors: ['constant', 'PopDensity', 'rwi', 'RoadDensity', 'NTL_Recovery']
});


var coefficients = ee.Array(result.get('coefficients')).transpose();

print('--- Regression Results ---');
print('Intercept (β0):', coefficients.get([0,0]));
print('PopDensity Coeff (β1):', coefficients.get([0,1]));
print('Wealth (RWI) Coeff (β2):', coefficients.get([0,2])); 
print('RoadDensity Coeff (β3):', coefficients.get([0,3]));

Map.addLayer(ntlRecovery, {min: -0.5, max: 0.5, palette: ['red', 'white', 'green']}, 'NTL Recovery Index');
Map.addLayer(popDensity, {min: 0, max: 50, palette: ['white', 'purple']}, 'Population Density');
Export.image.toDrive({
  image: floodedMasked,
  description: 'Sylhet_Flood_VV_VH_2017',
  scale: 10,
  region: studyArea,
  maxPixels: 1e13
});

Export.table.toDrive({
  collection: regressionData,
  description: 'Sylhet_SocioEconomic_Recovery_Table',
  fileFormat: 'CSV'
});

Export.table.toDrive({
  collection: studyArea,
  description: 'Sylhet_StudyArea',
  fileFormat: 'SHP'
});

Export.image.toDrive({
  image: pre,
  description: 'Pre_Flood_NTL',
  folder: 'GEE',
  fileNamePrefix: 'pre_ntl',
  region: studyArea,
  scale: 500,
  maxPixels: 1e13
});


Export.image.toDrive({
  image: post,
  description: 'Post_Flood_NTL',
  folder: 'GEE',
  fileNamePrefix: 'post_ntl',
  region: studyArea,
  scale: 500,
  maxPixels: 1e13
});

Export.image.toDrive({
  image: rec,
  description: 'Recovery_NTL',
  folder: 'GEE',
  fileNamePrefix: 'rec_ntl',
  region: studyArea,
  scale: 500,
  maxPixels: 1e13
});
```





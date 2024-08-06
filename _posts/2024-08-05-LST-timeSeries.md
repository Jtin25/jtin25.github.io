---
layout: post
title: Land surface temperature time series analysis
date: 2024-08-05 15:09:00
description: an example of a blog post with some code
tags: formatting code
categories: sample-posts
thumbnail: assets/img/timeSeries.jpg
featured: true
---

Time series analysis is a key technique in remote sensing for understanding temporal changes in the Earth's surface and atmosphere. In this tutorial, I will demonstrate its application in monitoring surface heat to analyze oscillations and seasonal variations in daytime and nighttime land surface temperature using MODIS satellite products.

### Setting Up Context
To begin, we define our area of interest and the timeframe for analysis. For this example, we focus on the city limits of Stockton over a 10-year period, from 2013 to 2023.
```javascript
//import aoi from assets
var cities = ee.FeatureCollection('projects/ee-tsejustin-geo4dev/assets/UScities');
var aoi = cities.filterMetadata('NAME', 'equals', 'Stockton');
var YEAR_RANGE = ee.Filter.calendarRange(2013, 2023, 'year');

//add the aoi to your map
Map.addLayer(aoi,'', 'aoi');
Map.centerObject(aoi, 10);
```
### Loading and Rescaling Collection
We use the MODIS Terra Land Surface Temperature and Emissivity product, which offers high temporal resolution (daily data) and includes both daytime and nighttime surface temperature measurements. Before conducting any analysis, it is crucial to apply the appropriate scale factor to the data, making the values comprehensible and comparable. In this case, the scale factor is 0.02, which converts the temperature data from its original unit to Kelvin. Subsequently, we subtract a constant to convert the temperature from Kelvin to Celsius. Detailed information about the band descriptions can be found in the GGE data catalog.
```javascript
//Import the LST image collection
var LST = ee.ImageCollection("MODIS/061/MOD11A1")
                  .filter(YEAR_RANGE)
                  .select('LST_Day_1km','LST_Night_1km','QC_Day','QC_Night')
                  .filterBounds(aoi);

//kelvin to celcius
function LSTScaleFactors(image) {
  var thermalBands = image.select('LST_Day_1km','LST_Night_1km').multiply(0.02)
  .subtract(273.15); 
  return image.addBands(thermalBands, null, true);

var LST_scaled = LST.map(LSTScaleFactors);
}
```
### Applying Bitmask
To enhance the accuracy and robustness of the analysis, we employ a bitmask function provided by Spatial Thoughts to filter out low-quality pixels. This step helps in mitigating the impact of data anomalies. However, since we will be using the mean value of each image for plotting the time series, this step may be optional as averaging can naturally reduce the influence of outliers. The original blog can be found here: <a href="https://spatialthoughts.com/2021/08/19/qa-bands-bitmasks-gee/">https://spatialthoughts.com/2021/08/19/qa-bands-bitmasks-gee/</a>.
```javascript
var bitwiseExtract = function(input, fromBit, toBit) {
  var maskSize = ee.Number(1).add(toBit).subtract(fromBit)
  var mask = ee.Number(1).leftShift(maskSize).subtract(1)
  return input.rightShift(fromBit).bitwiseAnd(mask)
}

// Apply QA mask to both LST Day and LST Night
var applyQaMask = function(image) {
  var lstDay = image.select('LST_Day_1km');
  var qcDay = image.select('QC_Day');
  var lstNight = image.select('LST_Night_1km');
  var qcNight = image.select('QC_Night');

  // Apply masks for LST Day
  var qaMaskDay = bitwiseExtract(qcDay, 0, 1).lte(1);
  var dataQualityMaskDay = bitwiseExtract(qcDay, 2, 3).eq(0);
  var lstErrorMaskDay = bitwiseExtract(qcDay, 6, 7).eq(0);
  var maskDay = qaMaskDay.and(dataQualityMaskDay).and(lstErrorMaskDay);
  var lstDayMasked = lstDay.updateMask(maskDay);

  // Apply masks for LST Night
  var qaMaskNight = bitwiseExtract(qcNight, 0, 1).lte(1);
  var dataQualityMaskNight = bitwiseExtract(qcNight, 2, 3).eq(0);
  var lstErrorMaskNight = bitwiseExtract(qcNight, 6, 7).eq(0);
  var maskNight = qaMaskNight.and(dataQualityMaskNight).and(lstErrorMaskNight);
  var lstNightMasked = lstNight.updateMask(maskNight);

  // Add the masked bands back to the image
  return image.addBands([lstDayMasked.rename('LST_Day_Masked'), lstNightMasked.rename('LST_Night_Masked')], null, true);
}

var LST_Masked = LST_scaled.map(applyQaMask);
```

### Plotting the Chart
Finally, we run the following code to create a time series chart. You can modify the reducer to use other descriptive statistics such as max, min, median, etc.

```javascript
var chart = ui.Chart.image.series({
imageCollection: LST_Masked.select('LST_Day_Masked','LST_Night_Masked'),
region: aoi,
reducer: ee.Reducer.mean(),
scale: 1000,
xProperty: 'system:time_start'
})
.setSeriesNames(['LST_Day_1km', 'LST_Night_1km'])
.setOptions({
title: 'Stockton Mean Surface Temperature in 2013-2023',
hAxis: {title: 'Date', titleTextStyle: {italic: false, bold: true}},
vAxis: {title: 'Surface Temperature', titleTextStyle: {italic: false, bold: true}},
lineWidth: 1.5,
colors: ['ED254E', '465362'], //change the color as you like
curveType: 'function',
interpolateNulls: true
});

print(chart)
```
### Final Result
<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/timeSeries.jpg" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    A simple, elegant caption looks good between image rows, after each row, or doesn't have to be there at all.
</div>

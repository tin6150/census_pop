
2020.0627 Census 2010 survey - population by Block Groups.  using LngLat coord.
based on 2020.0608 README.lngLat.rst

Essentially same steps as before.
but instead of only having the calculated pop density in the final geojson,  going to keep population, aland fields as well for easier verification.


ref:
https://medium.com/@mbostock/command-line-cartography-part-4-82d0d26df0cf
https://blog.mapbox.com/dive-into-large-datasets-with-3d-shapes-in-mapbox-gl-c89023ef291

~~~~

part 1 - census shape to svg
======

https://medium.com/@mbostock/command-line-cartography-part-1-897aa8f8ca2c

CA FIPS code = 06.

mkdir TMP_DATA_2010_BG
cd    TMP_DATA_2010_BG


2010 survey has diff filename convention than the 2018 estimate data.
see https://www2.census.gov/geo/tiger/GENZ2010/ReadMe.pdf
	The 2010 cartographic boundary files are named gz_2010_ss_lll_vv_rr.zip where:
	ss = state fips code or 'us' for a national level file # 06 = CA
	lll = summary level code   	# 150 is Block Group level data.  
					# 140 is Tract level data.  
					# 050 is county level data.
	vv = 2-digit version code
	rr = resolution level      # 500k is good enough for cholopleth, keep geojson size small
	o 500k = 1:500,000
	o 5m = 1:5,000,000
	o 20m = 1:20,000,000 

curl  https://www2.census.gov/geo/tiger/GENZ2010/gz_2010_06_150_00_500k.zip -o cb_2010_06_bg_500k.zip 
unzip ...

shp2json gz_2010_06_150_00_500k.shp -o ca2010bg.json
# above json is in lng/lat, truely geojson it seems)

part 2 - join shape with pop data by id
======

https://medium.com/@mbostock/command-line-cartography-part-2-c3a82c5c0f3

# xref: inet-dev-class/mapbox/eg_data_ndjson/README.txt.rst , which likely also in covid19_care_capacity_map/
# **step 2a**  geojson to ndjson
ndjson-split 'd.features' < ca2010bg.json  > ca2010bg.ndjson

# prev census 2018 estimate data in  lng/lat : 
{"type":"Feature","properties":{"STATEFP":"06","COUNTYFP":"075","TRACTCE":"980401","BLKGRPCE":"1","AFFGEOID":"1500000US060759804011","GEOID":"060759804011","NAME":"1","LSAD":"BG","ALAND":419323,"AWATER":247501289},
"geometry":{"type":"Polygon","coordinates":[[[-123.013916,37.700355],[-123.007786,37.698943],[-123.007548,37.70214],[-123.003507,37.704395999999996],[-123.00089299999999,37.701011],[-122.99875399999999,37.697438],[-123.002794,37.692736],[-123.005884,37.693489],[-123.007548,37.695934],[-123.012777,37.696498],[-123.013916,37.700355]]]}}

# round 4, using census 2010 data, example record:
{"type":"Feature","properties":{"GEO_ID":"1500000US060759804011","STATE":"06","COUNTY":"075","TRACT":"980401","BLKGRP":"1","NAME":"1","LSAD":"BG","CENSUSAREA":0.162},
"geometry":{"type":"Polygon","coordinates":[[[-123.013915661213,37.7003546200356],[-123.013897208114,37.7044781079737],[-123.01219398246,37.7067490723693],[-123.004488914922,37.7062624378153],[-123.000190295549,37.7029370922656],[-122.997189374607,37.6979085210616],[-123.00067693324601,37.6902034527376],[-123.00554329136101,37.6893923927899],[-123.01146402905701,37.6919066776061],[-123.014302738481,37.696205295407296],[-123.013915661213,37.7003546200356]]]}}


Census 2010 don't have GEOID.  it has GEO_ID, which correspond to AFFGEOID.  It is like GEOID but with extra text prefixed to it.  
eg: 060759804011 vs 1500000US060759804011
    01              0123456789|
"06" was likely FIPS of CA

No ALAND, AWATER, has a thing called CNESUSAREA, which maybe land area, but likely in diff unit.
(2014/2018 ACS (estimate) data ALAND was in m^2, this 2010 survey data CENSUSAREA is in sq mile  )


# **2b** add id field

#was# ndjson-map 'd.id = d.properties.GEOID.slice(2), d'  < ca2018bg.ndjson  > ca2018bg-id.ndjson
slice(2) means start at index=2 ?  so now use index=11
ndjson-map 'd.id = d.properties.GEO_ID.slice(11), d'  < ca2010bg.ndjson  > ca2010bg-id.ndjson


#2018 estimate data eg -- with extra id field at the end:
{"type":"Feature","properties":{"STATEFP":"06","COUNTYFP":"075","TRACTCE":"980401","BLKGRPCE":"1","AFFGEOID":"1500000US060759804011","GEOID":"060759804011","NAME":"1","LSAD":"BG","ALAND":419323,"AWATER":247501289},"geometry":{"type":"Polygon","coordinates":[[[-123.013916,37.700355],[-123.007786,37.698943],[-123.007548,37.70214],[-123.003507,37.704395999999996],[-123.00089299999999,37.701011],[-122.99875399999999,37.697438],[-123.002794,37.692736],[-123.005884,37.693489],[-123.007548,37.695934],[-123.012777,37.696498],[-123.013916,37.700355]]]},"id":"0759804011"}

#2010 survey data:
{"type":"Feature","properties":{"GEO_ID":"1500000US060759804011","STATE":"06","COUNTY":"075","TRACT":"980401","BLKGRP":"1","NAME":"1","LSAD":"BG","CENSUSAREA":0.162},"geometry":{"type":"Polygon","coordinates":[[[-123.013915661213,37.7003546200356],[-123.013897208114,37.7044781079737],[-123.01219398246,37.7067490723693],[-123.004488914922,37.7062624378153],[-123.000190295549,37.7029370922656],[-122.997189374607,37.6979085210616],[-123.00067693324601,37.6902034527376],[-123.00554329136101,37.6893923927899],[-123.01146402905701,37.6919066776061],[-123.014302738481,37.696205295407296],[-123.013915661213,37.7003546200356]]]},"id":"0759804011"}


# **2c** get data via census api

# census api to get pop 
source ~/.ssh/.env
echo $ApiKey

for ACS 5 year estimate API, refer to README.censusBlock.rst
For list of Census data API, see https://www.census.gov/data/developers/data-sets.html
Decenial census 2010 data API: https://www.census.gov/data/developers/data-sets/decennial-census.html

URL/VAR:: The B01003_001E in the URL specifies the total population estimate,
https://api.census.gov/data/2010/dec/sf1/variables.html ::
	P001001 is Total Population, but there are diff var for urban, rural and some stange combinations.
	H010001 : TOTAL POPULATION IN OCCUPIED HOUSING UNITS

eg: https://api.census.gov/data/2010/dec/sf1?get=H001001,NAME&for=state:*&key=[user key]

Example call for white population of 12 year olds in Alabama: 
api.census.gov/data/2010/dec/sf1?get=PCT012A015,PCT012A119&for=state:01&key=[user key]
	PCT012A015	Total!!Male!!12 years	SEX BY AGE (WHITE ALONE)
	PCT012A119	Total!!Female!!12 years	SEX BY AGE (WHITE ALONE)

#curl "https://api.census.gov/data/2010/dec/sf1?get=P001001,NAME&for=state:06&key=$ApiKey" -o caliPop2010.json

ref for more examples: https://api.census.gov/data/2010/dec/sf1/examples.html


150 = block group, 23212 rows
curl "https://api.census.gov/data/2010/dec/sf1?get=P001001,NAME&for=block%20group:*&in=state:06&in=county:*&in=tract:*&key=$ApiKey" -o CaAllBG.json

140 = tract, 8057 rows
curl "https://api.census.gov/data/2010/dec/sf1?get=P001001,NAME&for=tract:*&in=state:06&in=county:*&key=$ApiKey" -o CaAllTract.json


CaAllBG.json has 23212 rows, which match round 3 ACS 2018 estimate data downloaded as: 
for FIPS in $(seq -w 001 2 115); do
        echo curl "https://api.census.gov/data/2018/acs/acs5?get=NAME,B01003_001E&for=block%20group:*&in=state:06%20county:$FIPS&key=$ApiKey" -o cb_2018_06_bg_B01003.$FIPS.json
done


round 4 census 2010 eg result,

[["P001001","NAME"                                                 ,"state","county","tract","block group"],
["1703","Block Group 3, Census Tract 4441, Alameda County, California","06","001","444100","3"],
["1531","Block Group 2, Census Tract 4441, Alameda County, California","06","001","444100","2"],
 ["902","Block Group 1, Census Tract 4445, Alameda County, California","06","001","444500","1"],
  ^#0^  ^#1^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^  ^#2^ ^#3^   ^^#4^^  ^5^

# **fiexed2d**

#   field f1 is "id" field, combination of 3 columns: 3 and 4, 5, merged, no space.  0-idx
#   field f2 is pop,  prev used the var name "b01003" (pop estimate) , this round 4 change to just simply say "popCount"
#   dont have State FIPS in it cuz always CA (06)


ndjson-cat CaAllBG.json \
  | ndjson-split 'd.slice(1)' \
  | ndjson-map '{id: d[3] + d[4] + d[5], popCount:  d[0]}'  >  c_2010_06_bg_popCount.CA.ndjson
#                    ^^^^f1^^^^^^^^^^^^            ^^f2^^


#   ndjson has key: value pair, field f1 key is id,  field f2 key is popCount
# prev census 2018 est result look like this, which was that bostock expected
{"id":"0014441003","B01003":1755}
{"id":"0014441002","B01003":1320}
{"id":"0014445001","B01003":1199}

# this round 4 census 2010 survey data look like this:
{"id":"0014016001","popCount":"1205"}
{"id":"0014441002","popCount":"1531"}
{"id":"0014441003","popCount":"1703"}


# **eg 2e**  magic! join

ndjson-join 'd.id' \
  ca2010bg-id.ndjson \
  c_2010_06_bg_popCount.CA.ndjson \
  > ca2010bg-join.ndjson



# 2018 lng/lat:
[{"type":"Feature","properties":{"STATEFP":"06","COUNTYFP":"001","TRACTCE":"400400","BLKGRPCE":"3","AFFGEOID":"1500000US060014004003","GEOID":"060014004003","NAME":"3","LSAD":"BG","ALAND":201094,"AWATER":0},"geometry":{"type":"Polygon","coordinates":[[[-122.260223,37.852793],[-122.25836699999999,37.853196],[-122.257251,37.853176],[-122.25657799999999,37.847773],[-122.25721300000001,37.847712],[-122.261019,37.847232999999996],[-122.260223,37.852793]]]},"id":"0014004003"},{"id":"0014004003","B01003":1240}]

# 2010 block group data with lng/lat:
[{"type":"Feature","properties":{"GEO_ID":"1500000US060014004003","STATE":"06","COUNTY":"001","TRACT":"400400","BLKGRP":"3","NAME":"3","LSAD":"BG","CENSUSAREA":0.076},"geometry":{"type":"Polygon","coordinates":[[[-122.256689,37.848518999999996],[-122.25657799999999,37.847773],[-122.261019,37.847232999999996],[-122.260805,37.848694],[-122.260232,37.852742],[-122.257249,37.853164],[-122.256689,37.848518999999996]]]},"id":"0014004003"},{"id":"0014004003","popCount":"1110"}]




# **2f** calc pop density
# for round 4, also keep originally reported pop count and area.
# for 2018 ACS estimate data, area was under field ALAND, and unit was sq meter. eg: 201094
# for 2010 decenial census survey, area is under CENSUSAREA, and area seems to be in sq mile eg 0.076
# ratio seems to be conversion factor of sq meter to sq mile: 3.86102e-7
# let's assume that's the number and s
# https://www.census.gov/quickfacts/fact/note/US/LND110210 says
# Land area measurements are originally recorded as whole square meters (to convert square meters to square kilometers, divide by 1,000,000; to convert square kilometers to square miles, divide by 2.58999; to convert square meters to square miles, divide by 2,589,988).
# but the data for 2010 cant be in sq meter, must be square mile.  
# calculation of ratio and density match my guest that it is in sq mile also (which incidentally seems less accurate cuz of the unit is now order of factor larger and they don't have more significant figures)

#2018: ndjson-map 'd[0].properties = {density: Math.floor(d[1].B01003 / d[0].properties.ALAND * 2589975.2356)}, d[0]' \
  < ca2018bg-albers-join.ndjson \
  > ca2018bg-albers-density.ndjson

# 2010 decenial census api (area in sq mile, no longer need multiply by constant of 2589975.2356
# added 2 extra properties to be in the geojson
ndjson-map 'd[0].properties = {density: Math.floor(d[1].popCount / d[0].properties.CENSUSAREA), CENSUSAREA: d[0].properties.CENSUSAREA, popCount: d[1].popCount}, d[0]' \
  < ca2010bg-join.ndjson \
  > ca2010bg-density.ndjson

# eg result:
# 2018 lng/lat: seems fine, first property is density, rid of rest of the fields.
{"type":"Feature","properties":{"density":5440},"geometry":{"type":"Polygon","coordinates":[[[-117.878044124759,33.592764990129794],[-117.87591499999999,33.594837],[-117.87243,33.593393],[-117.870139,33.595701999999996],[-117.869425,33.595130999999995],[-117.866922,33.593781],[-117.86188899999999,33.591141],[-117.864058,33.589897],[-117.865574,33.587582],[-117.86614764088401,33.5873392496233],[-117.870749,33.59093],[-117.873352,33.592932999999995],[-117.87679,33.592321999999996],[-117.878044124759,33.592764990129794]]]},"id":"0590627021"}


# 2010
{"type":"Feature","properties":{"density":4527,"CENSUSAREA":0.197,"popCount":"892"},"geometry":{"type":"Polygon","coordinates":[[[-117.877655173756,33.5925965986269],[-117.877894,33.59319],[-117.87729,33.593821999999996],[-117.87649,33.593621999999996],[-117.87591499999999,33.594837],[-117.87243,33.593393],[-117.870139,33.595701999999996],[-117.869574,33.595236],[-117.869425,33.595130999999995],[-117.868622,33.59465],[-117.866922,33.593781],[-117.86188899999999,33.591141],[-117.866132,33.587362],[-117.86639961145201,33.5869706695073],[-117.87679,33.592321999999996],[-117.877655173756,33.5925965986269]]]},"id":"0590627021"}


# **2g** (prev 2h)- this should produce a proper geojson file.  

ndjson-reduce \
  < ca2010bg-density.ndjson \
  | ndjson-map '{type: "FeatureCollection", features: d}' \
  > ca2010bg-density.geojson

cp -p ca2010bg-density.geojson ../data/

# **QC**


Density for id: 0590627021 
per 2010 decenial census result:
{"density":4527,"CENSUSAREA":0.197,"popCount":"892"}

per 2018 estimate data:
{"density":5440}

seems about right given the increased population.
So guest that CENSUSAREA is in sq mile should also be accurate.

Further check by looking at mapbox result.

~~~~

Check:
why no data near East Palo Alto? BG id 
19900000
819901000
marsh area, thus no population?


~~~~~~~~


.. # use 8-space tab as that's how github render the rst
.. # vim: shiftwidth=8 tabstop=8 noexpandtab paste

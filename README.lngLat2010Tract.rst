
2020.0627 Census 2010 survey - population by Census Tract.  using LngLat coord.
based on 2020.0608 README.lngLat2010BG.rst

Essentially same steps as before.
But this one use Census Tract instead of Block Group level data.


ref:
https://medium.com/@mbostock/command-line-cartography-part-4-82d0d26df0cf
https://blog.mapbox.com/dive-into-large-datasets-with-3d-shapes-in-mapbox-gl-c89023ef291

~~~~

part 1 - census shape to svg
======

https://medium.com/@mbostock/command-line-cartography-part-1-897aa8f8ca2c

CA FIPS code = 06.

mkdir TMP_DATA_2010_TRACT
cd    TMP_DATA_2010_TRACT


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

XX curl  https://www2.census.gov/geo/tiger/GENZ2010/gz_2010_06_150_00_500k.zip -o cb_2010_06_bg_500k.zip 
curl  https://www2.census.gov/geo/tiger/GENZ2010/gz_2010_06_140_00_500k.zip -o cb_2010_06_tr_500k.zip 
unzip ...

shp2json gz_2010_06_140_00_500k.shp -o ca2010tr.json
# above json is in lng/lat, truely geojson 

part 2 - join shape with pop data by id
======

https://medium.com/@mbostock/command-line-cartography-part-2-c3a82c5c0f3

# xref: inet-dev-class/mapbox/eg_data_ndjson/README.txt.rst , which likely also in covid19_care_capacity_map/
# **step 2a**  geojson to ndjson
ndjson-split 'd.features' < ca2010tr.json  > ca2010tr.ndjson


# round 4, using census 2010 data, example record (block group level data): 
{"type":"Feature","properties":{"GEO_ID":"1500000US060759804011","STATE":"06","COUNTY":"075","TRACT":"980401","BLKGRP":"1","NAME":"1","LSAD":"BG","CENSUSAREA":0.162},
"geometry":{"type":"Polygon","coordinates":[[[-123.013915661213,37.7003546200356],[-123.013897208114,37.7044781079737],[-123.01219398246,37.7067490723693],[-123.004488914922,37.7062624378153],[-123.000190295549,37.7029370922656],[-122.997189374607,37.6979085210616],[-123.00067693324601,37.6902034527376],[-123.00554329136101,37.6893923927899],[-123.01146402905701,37.6919066776061],[-123.014302738481,37.696205295407296],[-123.013915661213,37.7003546200356]]]}}

# round 5, using census 2010 tract level data, example record:
{"type":"Feature","properties":{"GEO_ID":"1400000US06075980401","STATE":"06","COUNTY":"075","TRACT":"980401","NAME":"9804.01","LSAD":"Tract","CENSUSAREA":0.162},"geometry":{"type":"Polygon","coordinates":[[[-123.013915661213,37.7003546200356],[-123.013897208114,37.7044781079737],[-123.01219398246,37.7067490723693],[-123.004488914922,37.7062624378153],[-123.000190295549,37.7029370922656],[-122.997189374607,37.6979085210616],[-123.00067693324601,37.6902034527376],[-123.00554329136101,37.6893923927899],[-123.01146402905701,37.6919066776061],[-123.014302738481,37.696205295407296],[-123.013915661213,37.7003546200356]]]}}


Hmm... that tract and block group were the same size... city=county kind of ovelap situation?
comments below apply to 2010 tract-level data as well (as opposed to 2018 estimate data)

Census 2010 don't have GEOID.  it has GEO_ID, which correspond to AFFGEOID.  It is like GEOID but with extra text prefixed to it.  
eg: 060759804011 vs 1500000US060759804011
    01              0123456789|
"06" was likely FIPS of CA
No ALAND, AWATER, has a thing called CNESUSAREA, which maybe land area, but likely in diff unit.
(2014/2018 ACS (estimate) data ALAND was in m^2, this 2010 survey data CENSUSAREA is in sq mile.   )




# **2b** add id field

slice(2) means start at index=2 ?  so now use index=11
ndjson-map 'd.id = d.properties.GEO_ID.slice(11), d'  < ca2010tr.ndjson  > ca2010tr-id.ndjson


#2010 survey data:
{"type":"Feature","properties":{"GEO_ID":"1500000US060759804011","STATE":"06","COUNTY":"075","TRACT":"980401","BLKGRP":"1","NAME":"1","LSAD":"BG","CENSUSAREA":0.162},"geometry":{"type":"Polygon","coordinates":[[[-123.013915661213,37.7003546200356],[-123.013897208114,37.7044781079737],[-123.01219398246,37.7067490723693],[-123.004488914922,37.7062624378153],[-123.000190295549,37.7029370922656],[-122.997189374607,37.6979085210616],[-123.00067693324601,37.6902034527376],[-123.00554329136101,37.6893923927899],[-123.01146402905701,37.6919066776061],[-123.014302738481,37.696205295407296],[-123.013915661213,37.7003546200356]]]},"id":"0759804011"}


#2010 tract level data:
{"type":"Feature","properties":{"GEO_ID":"1400000US06075980401","STATE":"06","COUNTY":"075","TRACT":"980401","NAME":"9804.01","LSAD":"Tract","CENSUSAREA":0.162},"geometry":{"type":"Polygon","coordinates":[[[-123.013915661213,37.7003546200356],[-123.013897208114,37.7044781079737],[-123.01219398246,37.7067490723693],[-123.004488914922,37.7062624378153],[-123.000190295549,37.7029370922656],[-122.997189374607,37.6979085210616],[-123.00067693324601,37.6902034527376],[-123.00554329136101,37.6893923927899],[-123.01146402905701,37.6919066776061],[-123.014302738481,37.696205295407296],[-123.013915661213,37.7003546200356]]]},"id":"075980401"}


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

150 = block group, 23212 rows
curl "https://api.census.gov/data/2010/dec/sf1?get=P001001,NAME&for=block%20group:*&in=state:06&in=county:*&in=tract:*&key=$ApiKey" -o CaAllBG.json

140 = tract, 8057 rows
curl "https://api.census.gov/data/2010/dec/sf1?get=P001001,NAME&for=tract:*&in=state:06&in=county:*&key=$ApiKey" -o CaAllTract.json


round 4 census 2010 eg result,

[["P001001","NAME"                                                 ,"state","county","tract","block group"],
["1703","Block Group 3, Census Tract 4441, Alameda County, California","06","001","444100","3"],
["1531","Block Group 2, Census Tract 4441, Alameda County, California","06","001","444100","2"],
 ["902","Block Group 1, Census Tract 4445, Alameda County, California","06","001","444500","1"],
  ^#0^  ^#1^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^  ^#2^ ^#3^   ^^#4^^  ^5^

round 5 census 2010 tract level data eg result:
[["P001001","NAME"                                  ,"state","county","tract"],
["7280","Census Tract 4441, Alameda County, California","06","001","444100"],
["6696","Census Tract 4445, Alameda County, California","06","001","444500"],
  ^#0^  ^#1^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^  ^#2^ ^#3^   ^^#4^^  



# **fiexed2d**

#   field f1 is "id" field, combination of 2 columns: 3 and 4, merged, no space.  0-idx
#   field f2 is pop,  prev used the var name "b01003" (pop estimate) , this round 4 change to just simply say "popCount"
#   dont have State FIPS in it cuz always CA (06)


ndjson-cat CaAllTract.json \
  | ndjson-split 'd.slice(1)' \
  | ndjson-map '{id: d[3] + d[4], popCount:  d[0]}'  >  c_2010_06_tr_popCount.CA.ndjson
#                    ^^^^f1^^^^^            ^^f2^^


#   ndjson has key: value pair, field f1 key is id,  field f2 key is popCount

# prev round 4 census 2010 survey data look like this:
{"id":"0014016001","popCount":"1205"}
{"id":"0014441002","popCount":"1531"}
{"id":"0014441003","popCount":"1703"}

# this round 5 census 2010 survey data at tract level look like this:
# the id field has 2 fewer digits
{"id":"001444100","popCount":"7280"}
{"id":"001401600","popCount":"2163"}


# **eg 2e**  magic! join

ndjson-join 'd.id' \
  ca2010tr-id.ndjson \
  c_2010_06_tr_popCount.CA.ndjson \
  > ca2010tr-join.ndjson

# prev:
# 2010 block group data with lng/lat:
[{"type":"Feature","properties":{"GEO_ID":"1500000US060014004003","STATE":"06","COUNTY":"001","TRACT":"400400","BLKGRP":"3","NAME":"3","LSAD":"BG","CENSUSAREA":0.076},"geometry":{"type":"Polygon","coordinates":[[[-122.256689,37.848518999999996],[-122.25657799999999,37.847773],[-122.261019,37.847232999999996],[-122.260805,37.848694],[-122.260232,37.852742],[-122.257249,37.853164],[-122.256689,37.848518999999996]]]},"id":"0014004003"},{"id":"0014004003","popCount":"1110"}]

# round 5 2010 tract level eg:
[{"type":"Feature","properties":{"GEO_ID":"1400000US06001400400","STATE":"06","COUNTY":"001","TRACT":"400400","NAME":"4004","LSAD":"Tract","CENSUSAREA":0.272},"geometry":{"type":"Polygon","coordinates":[[[-122.260805,37.848694],[-122.260232,37.852742],[-122.257249,37.853164],[-122.256146,37.85321],[-122.253354,37.853581999999996],[-122.252534,37.851104],[-122.25245,37.849385999999996],[-122.255084,37.846069],[-122.256205,37.844685999999996],[-122.257524,37.843058],[-122.257923,37.842605999999996],[-122.26186,37.841353],[-122.261805,37.841789999999996],[-122.261296,37.845027],[-122.261019,37.847232999999996],[-122.260805,37.848694]]]},"id":"001400400"},{"id":"001400400","popCount":"3703"}]



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

# 2010 decenial census api (area in sq mile, no longer need multiply by constant of 2589975.2356
# added 2 extra properties to be in the geojson
ndjson-map 'd[0].properties = {density: Math.floor(d[1].popCount / d[0].properties.CENSUSAREA), CENSUSAREA: d[0].properties.CENSUSAREA, popCount: d[1].popCount}, d[0]' \
  < ca2010tr-join.ndjson \
  > ca2010tr-density.ndjson

# round 4 block group eg (with 23203 rows, vs downloaded json had 23212 rows):
# 2010
{"type":"Feature","properties":{"density":4527,"CENSUSAREA":0.197,"popCount":"892"},"geometry":{"type":"Polygon","coordinates":[[[-117.877655173756,33.5925965986269],[-117.877894,33.59319],[-117.87729,33.593821999999996],[-117.87649,33.593621999999996],[-117.87591499999999,33.594837],[-117.87243,33.593393],[-117.870139,33.595701999999996],[-117.869574,33.595236],[-117.869425,33.595130999999995],[-117.868622,33.59465],[-117.866922,33.593781],[-117.86188899999999,33.591141],[-117.866132,33.587362],[-117.86639961145201,33.5869706695073],[-117.87679,33.592321999999996],[-117.877655173756,33.5925965986269]]]},"id":"0590627021"}


# round 5 tract level eg (2010 data) 8048 rows, so the join like dropped some row, as downloaded json had 8057 rows:

{"type":"Feature","properties":{"density":8196,"CENSUSAREA":0.575,"popCount":"4713"},"geometry":{"type":"Polygon","coordinates":[[[-117.877655173756,33.5925965986269],[-117.87886893225802,33.5929818350211],[-117.879128,33.59377],[-117.878556,33.594151],[-117.877672,33.595940999999996],[-117.87260599999999,33.600871999999995],[-117.872704,33.601169],[-117.87242,33.601234],[-117.871272,33.602391],[-117.867454,33.599741],[-117.865913,33.598672],[-117.863636,33.597093],[-117.862991,33.59681],[-117.862926,33.596847],[-117.862797,33.596274],[-117.862165,33.596156],[-117.861672,33.59644],[-117.861009,33.596286],[-117.85927699999999,33.593005],[-117.86188899999999,33.591141],[-117.866132,33.587362],[-117.86639961145201,33.5869706695073],[-117.87679,33.592321999999996],[-117.877655173756,33.5925965986269]]]},"id":"059062702"}
    # ^^ 059062702 has 1 digit fewer than the round 4 version with block group id, expected.
{"type":"Feature","properties":{"density":13613,"CENSUSAREA":0.272,"popCount":"3703"},"geometry":{"type":"Polygon","coordinates":[[[-122.260805,37.848694],[-122.260232,37.852742],[-122.257249,37.853164],[-122.256146,37.85321],[-122.253354,37.853581999999996],[-122.252534,37.851104],[-122.25245,37.849385999999996],[-122.255084,37.846069],[-122.256205,37.844685999999996],[-122.257524,37.843058],[-122.257923,37.842605999999996],[-122.26186,37.841353],[-122.261805,37.841789999999996],[-122.261296,37.845027],[-122.261019,37.847232999999996],[-122.260805,37.848694]]]},"id":"001400400"}


# **2g** (prev 2h)- this should produce a proper geojson file.  

ndjson-reduce \
  < ca2010tr-density.ndjson \
  | ndjson-map '{type: "FeatureCollection", features: d}' \
  > ca2010tr-density.geojson

cp -p ca2010tr-density.geojson ../data/

# **QC**


Density for id: 0590627021 
per 2010 decenial census result:
{"density":4527,"CENSUSAREA":0.197,"popCount":"892"}

Density for id: 059062702 , tract level data (id has 1 fewer digit, which combine several blk grp):
{"density":8196,"CENSUSAREA":0.575,"popCount":"4713"}


~~~~


.. # use 8-space tab as that's how github render the rst
.. # vim: shiftwidth=8 tabstop=8 noexpandtab paste

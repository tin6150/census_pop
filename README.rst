

2020.0605

playing with census data.

Ling is interested to see if can get pop at census tract vs block level.  
preferably density, so need aland 

ref:
https://medium.com/@mbostock/command-line-cartography-part-4-82d0d26df0cf
https://blog.mapbox.com/dive-into-large-datasets-with-3d-shapes-in-mapbox-gl-c89023ef291

maybe tidycensus in R
maybe mapbox
tbd


~~~~

part 1 - census shape to svg
======

https://medium.com/@mbostock/command-line-cartography-part-1-897aa8f8ca2c


TIGER = very detailed.
SHAPE file - much smaller, good for thematic maps (cartographically small scale)

need pop data matching shape: tract vs block

CA FIPS code = 06.
download data:

mkdir TMP_DATA
cd    TMP_DATA

curl 'https://www2.census.gov/geo/tiger/GENZ2014/shp/cb_2014_06_tract_500k.zip' -o cb_2014_06_tract_500k.zip  # shape + dbIII
unzip ...

shp2json cb_2014_06_tract_500k.shp -o ca.json
geoproject 'd3.geoConicEqualArea().parallels([34, 40.5]).rotate([120, 0]).fitSize([960, 960], d)' < ca.json > ca-albers.json
geo2svg -w 960 -h 960 < ca-albers.json > ca-albers.svg




part 2 - join shape with pop data by id
======

https://medium.com/@mbostock/command-line-cartography-part-2-c3a82c5c0f3

# xref: inet-dev-class/mapbox/eg_data_ndjson/README.txt.rst , which likely also in covid19_care_capacity_map/
# **eg 2a**  geojson to ndjson
ndjson-split 'd.features' < ca-albers.json  > ca-albers.ndjson

# **eg 2b**  add id field to the end
ndjson-map 'd.id = d.properties.GEOID.slice(2), d'  < ca-albers.ndjson  > ca-albers-id.ndjson


# **2c** get data via census api

# census api to get pop 
# need census api key, see bmail.


B01003_001E	Estimate!!Total	TOTAL POPULATION	not required	B01003_001EA, B01003_001M, B01003_001MA	0	int

XX curl 'https://api.census.gov/data/2014/acs5?get=B01003_001EA&for=tract:*&in=state:06' -o cb_2014_06_tract_B01003.json # track pop ?
slightly changed api

Example Call:    http://api.census.gov/data/2014/acs/acs5/profile?get=DP02_0001PE&for=state:*&key=... (See notes)
many examples : https://api.census.gov/data/2014/acs/acs5/examples.html
                https://api.census.gov/data/2014/acs/acs5?get=NAME,B00001_001E&for=tract:*&in=state:01&key=YOUR_KEY_GOES_HERE

			  export ApiKey=c9b728... see bmail or . .env , also in .ssh
curl "https://api.census.gov/data/2014/acs/acs5?get=NAME,B01003_001E&for=tract:*&in=state:06&key=$ApiKey" -o cb_2014_06_tract_B01003.json # track pop 
curl "https://api.census.gov/data/2018/acs/acs5?get=NAME,B01003_001E&for=tract:*&in=state:06&key=$ApiKey" -o cb_2018_06_tract_B01003.json # track pop 2018

works! but result now has extra descriptive fields in them...
cat cb_2014_06_tract_B01003.json | wc
cat cb_2014_06_tract_B01003.json | json2csv > cb_2014_06_tract_B01003.json.csv 

[["NAME","B01003_001E","state","county","tract"],
["Census Tract 4382.03, Alameda County, California","4384","06","001","438203"],
["Census Tract 4382.04, Alameda County, California","5338","06","001","438204"],
  ^^extra 1^^           ^^extra 2^^^^^  ^^extra 3^^  ^#0^  ^#1  ^#2^   ^^#3^^
 ^^^^^^single^^field^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
#1, ... #3 are the orig field bostock eg refers to.  so i am no off by +1 

# **eg 2d** 

ndjson-cat cb_2014_06_tract_B01003.json \
  | ndjson-split 'd.slice(1)' \
  | ndjson-map '{id: d[2] + d[3], B01003: +d[0]}'  >        cb_2014_06_tract_B01003.ndjson
#                    ^^^^^1^^^^^          ^^^2^^
#   field 1 is combination of 2 column, 2 and 3, merged, no space.  0-idx
#   field 2 is column 0
#   ndjson has key: value pair, field 1 key is "id: ', field 2 key is "B01003: '

# **fiexed 2d** 
ndjson-cat cb_2014_06_tract_B01003.json \
  | ndjson-split 'd.slice(1)' \
  | ndjson-map '{id: d[3] + d[4], B01003: +d[1]}'  >        cb_2014_06_tract_B01003.ndjson
#                    ^^^^^1^^^^^          ^^^2^^
# should have been off by +1 in the new json retrieved via new census api...

# result is this, which looks like what bostock expect
{"id":"001438203","B01003":4384}
{"id":"001438204","B01003":5338}
{"id":"001438300","B01003":4133}


# json2csv cannot handle ndjson
# use vscode data preview extension to help viz file, using head -4 or so...

# **eg 2e**  magic! join

ndjson-join 'd.id' \
  ca-albers-id.ndjson \
  cb_2014_06_tract_B01003.ndjson \
  > ca-albers-join.ndjson

# **$** previously  borked here.  fixed now
# [{"type":"Feature","properties":{"STATEFP":"06","COUNTYFP":"001","TRACTCE":"400300","AFFGEOID":"1400000US06001400300","GEOID":"06001400300","NAME":"4003","LSAD":"CT","ALAND":1105329,"AWATER":0},"geometry":{"type":"Polygon","coordinates":[[[224.3021507494117,425.1613296471837],[224.4889212459765,425.02853000146524],[224.8054892227229,424.90924473882023],[225.09157727394734,424.797926817982],[225.29373002719294,424.7042420166931],[225.65996339344974,424.52901179192713],[225.95108431320563,424.3385241647384],[225.912059937863,424.3983338513344],[225.81079279254033,424.6100213459463],[225.58249395352414,425.05059707011105],[225.35882837057437,425.47619464326226],[225.22516372508392,425.73538936106115],[224.86658222608307,425.5294755512],[224.63434603931907,425.4732297669584],[224.43926884491924,425.4361850983005],[224.44504485979195,425.3811563562076],[224.37116077415172,425.3749388649712],[224.17960589902756,425.397389513148],[224.3021507494117,425.1613296471837]]]},"id":"001400300"},{"id":"001400300","B01003":5428}]


# **2f** - calc pop density

ndjson-map 'd[0].properties = {density: Math.floor(d[1].B01003 / d[0].properties.ALAND * 2589975.2356)}, d[0]' \
  < ca-albers-join.ndjson \
  > ca-albers-density.ndjson

# result of 2f seems good


# **2g**  (prev had an extra erroneous step and this was called 2h)

ndjson-reduce \
  < ca-albers-density.ndjson \
  | ndjson-map '{type: "FeatureCollection", features: d}' \
  > ca-albers-density.json

# **2g alt** (prev as 2h alt)
ndjson-reduce 'p.features.push(d), p' '{type: "FeatureCollection", features: []}' \
  < ca-albers-density.ndjson \
  > ca-albers-density.json
# this one worked.  result said to be viewable in mapshaper.org


npm install -g d3

# **2h** (prev 2i)

ndjson-map -r d3 \
  '(d.properties.fill = d3.scaleSequential(d3.interpolateViridis).domain([0, 4000])(d.properties.density), d)' \
  < ca-albers-density.ndjson \
  > ca-albers-color.ndjson



geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  < ca-albers-color.ndjson \
  > ca-albers-color.purple.svg

xviewer ca-albers-color.purple.svg  # work, but ugly purple map.



part 3 - shrink with TopoJSON
======

https://medium.com/@mbostock/command-line-cartography-part-3-1158e4c55a1e

npm install -g topojson

# **3a**
geo2topo -n \
  tracts=ca-albers-density.ndjson \
  > ca-tracts-topo.json

toposimplify -p 1 -f \
  < ca-tracts-topo.json \
  > ca-simple-topo.json

topoquantize 1e5 \
  < ca-simple-topo.json \
  > ca-quantized-topo.json

topomerge -k 'd.id.slice(0, 3)' counties=tracts \
  < ca-quantized-topo.json \
  > ca-merge-topo.json


topomerge --mesh -f 'a !== b' counties=counties \
  < ca-merge-topo.json \
  > ca-topo.json


# tried preview, but don't work.  
geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  < ca-topo.json \
  > ca-topo.svg

part 4 - improve color 
======

https://medium.com/@mbostock/command-line-cartography-part-4-82d0d26df0cf

# each version below are independent of one another
# they just need input ca-topo.svg, the result of part 3 above.

# **4a** linear transform

topo2geo tracts=- \
  < ca-topo.json \
  | ndjson-map -r d3 'z = d3.scaleSequential(d3.interpolateViridis).domain([0, 4000]), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' \
  | ndjson-split 'd.features' \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > ca-tracts-color.svg
# result visually very similar to ca-albers-color.purple.svg, but about 1/4 the file size.

# **4b** non-linear (sqrt) transform, still purple
# used sqrt, which was said hard to conceptualize, not lots of point to do it.

topo2geo tracts=- \
  < ca-topo.json \
  | ndjson-map -r d3 'z = d3.scaleSequential(d3.interpolateViridis).domain([0, 100]), d.features.forEach(f => f.properties.fill = z(Math.sqrt(f.properties.density))), d' \
  | ndjson-split 'd.features' \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > ca-tracts-sqrt.svg

# 4c = interesting looking
topo2geo tracts=- \
  < ca-topo.json \
  | ndjson-map -r d3 'z = d3.scaleLog().domain(d3.extent(d.features.filter(f => f.properties.density), f => f.properties.density)).interpolate(() => d3.interpolateViridis), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' \
  | ndjson-split 'd.features' \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > ca-tracts-log.svg

# 4d p-quantile
topo2geo tracts=- \
  < ca-topo.json \
  | ndjson-map -r d3 'z = d3.scaleQuantile().domain(d.features.map(f => f.properties.density)).range(d3.quantize(d3.interpolateViridis, 256)), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' \
  | ndjson-split 'd.features' \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > ca-tracts-quantile.svg

# result said to show diff even in densest area 
# (ie, map isn't just a bright blob in metro area, but there are some fine details)



npm install -g d3-scale-chromatic

# **4e** OrRd color scheme, decent looking result

topo2geo tracts=- \
  < ca-topo.json \
  | ndjson-map -r d3 -r d3=d3-scale-chromatic 'z = d3.scaleThreshold().domain([1, 10, 50, 200, 500, 1000, 2000, 4000]).range(d3.schemeOrRd[9]), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' \
  | ndjson-split 'd.features' \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > ca-tracts-threshold.svg

# borked again :/

# **4e fixing** actually just need to say -r d3-scale-chromatic (ie, just drop the prefix d3= )
# ref: https://medium.com/@v.brusylovets/hi-dario-yeah-after-two-years-something-is-changed-in-d3-1e4222744c93
topo2geo tracts=- \
  < ca-topo.json \
  | ndjson-map -r d3 -r d3-scale-chromatic 'z = d3.scaleThreshold().domain([1, 10, 50, 200, 500, 1000, 2000, 4000]).range(d3.schemeOrRd[9]), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' \
  | ndjson-split 'd.features' \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > ca-tracts-threshold.svg


# **4f** add county borders 
# instead of county borders, i think highway may better explain the density pattern.
# but county lines may still be needed to help orientation, especially San Joaquin valley?
# not if include some smaller state highway ?
(topo2geo tracts=- \
    < ca-topo.json \
    | ndjson-map -r d3 -r d3-scale-chromatic 'z = d3.scaleThreshold().domain([1, 10, 50, 200, 500, 1000, 2000, 4000]).range(d3.schemeOrRd[9]), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' \
    | ndjson-split 'd.features'; \
topo2geo counties=- \
    < ca-topo.json \
    | ndjson-map 'd.properties = {"stroke": "#000", "stroke-opacity": 0.3}, d')\
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > ca.svg

# ca.svg is final result presented on web page.
# all steps worked now, get ca map with pop density per census tracts, OrRd color scale
# need to add a color scale, which was not well explained.
# i dont think i want to deal with d3 graphics...

# cp ca.svg ca-popDensityByTract-OrRd.svg

# next step is try to do the same with census block level data

xviewer ca.svg


.. # use 8-space tab as that's how github render the rst
.. # vim: shiftwidth=8 tabstop=8 noexpandtab paste

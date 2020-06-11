

2020.0608

based on README.censusBlock.rst,
but stripping the CA Albers projection where possible
and want to create files that are true geojson with lng/lat coordinates.



it is still census data, at block level.  more specifically, at block group level.

need pop, preferably density, so need aland 

ref:
https://medium.com/@mbostock/command-line-cartography-part-4-82d0d26df0cf
https://blog.mapbox.com/dive-into-large-datasets-with-3d-shapes-in-mapbox-gl-c89023ef291

~~~~

part 1 - census shape to svg
======

https://medium.com/@mbostock/command-line-cartography-part-1-897aa8f8ca2c

CA FIPS code = 06.

mkdir TMP_DATA_LngLat
cd    TMP_DATA_LngLat


XX curl 'https://www2.census.gov/geo/tiger/GENZ2018/shp/cb_2018_06_bg_500k.zip' -o cb_2018_06_bg_500k.zip ;  unzip ...
shp2json cb_2018_06_bg_500k.shp -o ca2018bg.json
# above json is in lng/lat, truely geojson it seems)

ln ../TMP_DATA_BLOCK/ca2018bg.json .


XX omiting ca albers projection
XX geoproject 'd3.geoConicEqualArea().parallels([34, 40.5]).rotate([120, 0]).fitSize([960, 960], d)' < ca2018bg.json > ca2018bg-albers.json
# map project, changed coordinates to like [164.1468809671912,437.62295438355295]
# anyway, lost lng/lat by here, not good for modeling work downstream.

XX geo2svg -w 960 -h 960 < ca2018bg-albers.json > ca2018bg-albers.svg
# that svg file is huge, 11M, xviewer could not handle it.  (but prev handled 6.7M file)



part 2 - join shape with pop data by id
======

https://medium.com/@mbostock/command-line-cartography-part-2-c3a82c5c0f3

# xref: inet-dev-class/mapbox/eg_data_ndjson/README.txt.rst , which likely also in covid19_care_capacity_map/
# **eg 2a**  geojson to ndjson
XX ndjson-split 'd.features' < ca2018bg-albers.json  > ca2018bg-albers.ndjson
ndjson-split 'd.features' < ca2018bg.json  > ca2018bg.ndjson

# prev albers -  census block group (2018) TRACTCE: 980401
{"type":"Feature","properties":{"STATEFP":"06","COUNTYFP":"075","TRACTCE":"980401","BLKGRPCE":"1","AFFGEOID":"1500000US060759804011","GEOID":"060759804011","NAME":"1","LSAD":"BG","ALAND":419323,"AWATER":247501289},"geometry":{"type":"Polygon","coordinates":[[[164.1468809671912,437.62295438355295],[164.63136562909594,437.78130627883274],[164.66061334779198,437.4585472897443],[164.99020559033386,437.24058568718465],[165.18788475165627,437.5895682364189],[165.34708696199812,437.95636228142894],[165.00971718370104,438.4217441413507],[164.76560417638595,438.337767038262],[164.64069467117463,438.0862550961165],[164.22534302939806,438.0159600586103],[164.1468809671912,437.62295438355295]]]}}

# new lng/lat : 
{"type":"Feature","properties":{"STATEFP":"06","COUNTYFP":"075","TRACTCE":"980401","BLKGRPCE":"1","AFFGEOID":"1500000US060759804011","GEOID":"060759804011","NAME":"1","LSAD":"BG","ALAND":419323,"AWATER":247501289},"geometry":{"type":"Polygon","coordinates":[[[-123.013916,37.700355],[-123.007786,37.698943],[-123.007548,37.70214],[-123.003507,37.704395999999996],[-123.00089299999999,37.701011],[-122.99875399999999,37.697438],[-123.002794,37.692736],[-123.005884,37.693489],[-123.007548,37.695934],[-123.012777,37.696498],[-123.013916,37.700355]]]}}

# **2b** add id field

ndjson-map 'd.id = d.properties.GEOID.slice(2), d'  < ca2018bg.ndjson  > ca2018bg-id.ndjson

eg extra id field at the end:
{"type":"Feature","properties":{"STATEFP":"06","COUNTYFP":"075","TRACTCE":"980401","BLKGRPCE":"1","AFFGEOID":"1500000US060759804011","GEOID":"060759804011","NAME":"1","LSAD":"BG","ALAND":419323,"AWATER":247501289},"geometry":{"type":"Polygon","coordinates":[[[-123.013916,37.700355],[-123.007786,37.698943],[-123.007548,37.70214],[-123.003507,37.704395999999996],[-123.00089299999999,37.701011],[-122.99875399999999,37.697438],[-123.002794,37.692736],[-123.005884,37.693489],[-123.007548,37.695934],[-123.012777,37.696498],[-123.013916,37.700355]]]},"id":"0759804011"}

# **2c** get data via census api

# census api to get pop 
# 58 ndjson files, combine into single one.  (23212 lines, match prev wc sum for all counties)
#~ cat cb_2018_06_bg_B01003.???.ndjson > cb_2018_06_bg_B01003.CA.ndjson
# (skipped 2b to 2d, just reuse prev Block Grp data)
ln ../TMP_DATA_BLOCK/cb_2018_06_bg_B01003.CA.ndjson  .

# **2d** 
# result is this, which looks like what bostock expect.  could be piped to a csv
{"id":"0014441003","B01003":1755}
{"id":"0014441002","B01003":1320}
{"id":"0014445001","B01003":1199}



# **eg 2e**  magic! join

ndjson-join 'd.id' \
  ca2018bg-id.ndjson \
  cb_2018_06_bg_B01003.CA.ndjson \
  > ca2018bg-join.ndjson

# new eg for blocks group level data, seems like 4 rec for TRACTCE 400300, only showing first one
# new field BLKGRPCE added.  but not consequential, just need population data (B01003) and ALAND (land area) in the next step "2f"
# [{"type":"Feature","properties":{"STATEFP":"06","COUNTYFP":"001","TRACTCE":"400300","BLKGRPCE":"2","AFFGEOID":"1500000US060014003002","GEOID":"060014003002","NAME":"2","LSAD":"BG","ALAND":269347,"AWATER":0},"geometry":{"type":"Polygon","coordinates":[[[224.44636755999747,425.3691423744949],[224.5939145191771,425.21708244189904],[224.7830672777208,425.34538644862505],[225.07612515752876,425.52808473848654],[225.1067691336628,425.4894377365358],[225.31806810037276,425.1724692650978],[225.58249395352414,425.05059707011105],[225.35882837057437,425.47619464326226],[225.22516372508392,425.73538936106115],[224.86658222608307,425.5294755512],[224.6649471804986,425.47993141313145],[224.43926884491924,425.4361850983005],[224.44636755999747,425.3691423744949]]]},"id":"0014003002"},{"id":"0014003002","B01003":1404}]


# new lng/lat:
[{"type":"Feature","properties":{"STATEFP":"06","COUNTYFP":"001","TRACTCE":"400400","BLKGRPCE":"3","AFFGEOID":"1500000US060014004003","GEOID":"060014004003","NAME":"3","LSAD":"BG","ALAND":201094,"AWATER":0},"geometry":{"type":"Polygon","coordinates":[[[-122.260223,37.852793],[-122.25836699999999,37.853196],[-122.257251,37.853176],[-122.25657799999999,37.847773],[-122.25721300000001,37.847712],[-122.261019,37.847232999999996],[-122.260223,37.852793]]]},"id":"0014004003"},{"id":"0014004003","B01003":1240}]


# **2f** calc pop density

XX ndjson-map 'd[0].properties = {density: Math.floor(d[1].B01003 / d[0].properties.ALAND * 2589975.2356)}, d[0]' \
  < ca2018bg-albers-join.ndjson \
  > ca2018bg-albers-density.ndjson

ndjson-map 'd[0].properties = {density: Math.floor(d[1].B01003 / d[0].properties.ALAND * 2589975.2356)}, d[0]' \
  < ca2018bg-join.ndjson \
  > ca2018bg-density.ndjson

# eg result:
{"type":"Feature","properties":{"density":0},"geometry":{"type":"Polygon","coordinates":[[[164.1468809671912,437.62295438355295],[164.63136562909594,437.78130627883274],[164.66061334779198,437.4585472897443],[164.99020559033386,437.24058568718465],[165.18788475165627,437.5895682364189],[165.34708696199812,437.95636228142894],[165.00971718370104,438.4217441413507],[164.76560417638595,438.337767038262],[164.64069467117463,438.0862550961165],[164.22534302939806,438.0159600586103],[164.1468809671912,437.62295438355295]]]},"id":"0759804011"}
{"type":"Feature","properties":{"density":5440},"geometry":{"type":"Polygon","coordinates":[[[583.3067862409382,854.8717329876263],[583.4813178511808,854.6585101562487],[583.7779327272376,854.7977310974125],[583.9655380355614,854.5602629991972],[584.0269325681705,854.6165658408554],[584.240683477404,854.7481359034291],[584.670342625474,855.0051488986787],[584.4905937377472,855.1348697365938],[584.3682429572099,855.3714858612866],[584.3205053355547,855.3970830898411],[583.9250158061104,855.0432402918268],[583.701365900805,854.8459183147456],[583.4133542298542,854.9140974854508],[583.3067862409382,854.8717329876263]]]},"id":"0590627021"}

# new lng/lat: seems fine, first property is density, rid of rest of the fields.
{"type":"Feature","properties":{"density":0},"geometry":{"type":"Polygon","coordinates":[[[-123.013916,37.700355],[-123.007786,37.698943],[-123.007548,37.70214],[-123.003507,37.704395999999996],[-123.00089299999999,37.701011],[-122.99875399999999,37.697438],[-123.002794,37.692736],[-123.005884,37.693489],[-123.007548,37.695934],[-123.012777,37.696498],[-123.013916,37.700355]]]},"id":"0759804011"}
{"type":"Feature","properties":{"density":5440},"geometry":{"type":"Polygon","coordinates":[[[-117.878044124759,33.592764990129794],[-117.87591499999999,33.594837],[-117.87243,33.593393],[-117.870139,33.595701999999996],[-117.869425,33.595130999999995],[-117.866922,33.593781],[-117.86188899999999,33.591141],[-117.864058,33.589897],[-117.865574,33.587582],[-117.86614764088401,33.5873392496233],[-117.870749,33.59093],[-117.873352,33.592932999999995],[-117.87679,33.592321999999996],[-117.878044124759,33.592764990129794]]]},"id":"0590627021"}



# **2g** (prev 2h)- this should produce a proper geojson file.  

ndjson-reduce \
  < ca2018bg-density.ndjson \
  | ndjson-map '{type: "FeatureCollection", features: d}' \
  > ca2018bg-density.json

# vscode show this as upside down ca map.
# took a long ass time to render in vscode preview, and CA is upside down.
# long time cuz it has lots of details, not simplified with TopoJSON yet. 17M file.
# coordinate is in lng/lat.
# but that only display map data (census block group outlines?), and density is not colored in.

# this time this json should really be a geojson, so
ln ca2018bg-density.json ca2018bg-density.lngLat.geojson 

# **2g alt** (prev 2h alt)

ndjson-reduce 'p.features.push(d), p' '{type: "FeatureCollection", features: []}' \
  < ca2018bg-density.ndjson \
  > ca2018bg-density.2gAlt.json
# diff ca2018bg-density.2gAlt.json ca2018bg-density.json # results are identical


# **QC**

Ling asked to check density value between this and prev.
ie ca2018bg-density.json and ca2018bg-albers-density.json
maybe easier comaring 
ca2018bg-density.ndjson ca2018bg-albers-density.ndjson

	head -1  ca2018bg-albers-density.ndjson 
{"type":"Feature","properties":{"density":0},"geometry":{"type":"Polygon","coordinates":[[[164.1468809671912,437.62295438355295],[164.63136562909594,437.78130627883274],[164.66061334779198,437.4585472897443],[164.99020559033386,437.24058568718465],[165.18788475165627,437.5895682364189],[165.34708696199812,437.95636228142894],[165.00971718370104,438.4217441413507],[164.76560417638595,438.337767038262],[164.64069467117463,438.0862550961165],[164.22534302939806,438.0159600586103],[164.1468809671912,437.62295438355295]]]},"id":"0759804011"}

	# column 8 contains block group id, column 4 has pop density 
	head -2  ca2018bg-albers-density.ndjson | awk -F: '{print $8 "\t" $4   }'
"0759804011"}	0},"geometry"
"0590627021"}	5440},"geometry"

	cat ca2018bg-albers-density.ndjson | awk -F: '{print $8 "\t" $4   }' | sort -n > ca2018bg-albers-density.tsv 
	cat ca2018bg-density.ndjson        | awk -F: '{print $8 "\t" $4   }' | sort -n > ca2018bg-density.tsv 

vimdiff -o ca2018bg-density.tsv ../TMP_DATA_BLOCK/ca2018bg-albers-density.tsv 
diff ca2018bg-density.tsv ../TMP_DATA_BLOCK/ca2018bg-albers-density.tsv 
no diff shown, so they are the same.


# eg: 
{"type":"FeatureCollection","features":[{"type":"Feature","properties":{"density":0},"geometry":{"type":"Polygon","coordinates":[[[-123.013916,37.700355],[-123.007786,37.698943],[-123.007548,37.70214],[-123.003507,37.704395999999996],[-123.00089299999999,37.701011],[-122.99875399999999,37.697438],[-123.002794,37.692736],[-123.005884,37.693489],[-123.007548,37.695934],[-123.012777,37.696498],[-123.013916,37.700355]]]},"id":"0759804011"},{"type":"Feature","properties":{"density":5440},"geometry":{"type":"Polygon","coordinates":[[[-117.878044124759,33.592764990129794],[-117.87591499999999,33.594837]...

*up to here should be ok*

# **2h prep**
# *input is from result of 2f, not 2g*
XX geoproject 'd3.geoConicEqualArea().parallels([34, 40.5]).rotate([120, 0]).fitSize([960, 960], d)' < ca.json > ca-albers.json
geoproject 'd3.geoConicEqualArea().parallels([34, 40.5]).rotate([120, 0]).fitSize([960, 960], d)' < ca2018bg-density.ndjson > ca2018bg-density-albers.json

*didn't work.  so taking geojson from 2g and upload to mapbox as tilesets import and see.  name is ca2018bg-density-dbc449 -> sn50.4jkgze8k*  yeap, worked in mapbox.  probably want to avoid dealing with d3...



# **2h** (prev 2i)

# *input is from result of 2f, not 2g*
ndjson-map -r d3 \
  '(d.properties.fill = d3.scaleSequential(d3.interpolateViridis).domain([0, 4000])(d.properties.density), d)' \
  < ca2018bg-density.ndjson \
  > ca2018bg-color.ndjson

# eg
{"type":"Feature","properties":{"density":0,"fill":"#440154"},"geometry":{"type":"Polygon","coordinates":[[[-123.013916,37.700355],[-123.007786,37.698943],[-123.007548,37.70214],[-123.003507,37.704395999999996],[-123.00089299999999,37.701011],[-122.99875399999999,37.697438],[-123.002794,37.692736],[-123.005884,37.693489],[-123.007548,37.695934],[-123.012777,37.696498],[-123.013916,37.700355]]]},"id":"0759804011"}
{"type":"Feature","properties":{"density":5440,"fill":"#fde725"},"geometry":{"type":"Polygon","coordinates":[[[-117.878044124759,33.592764990129794],[-117.87591499999999,33.594837],[-117.87243,33.593393],[-117.870139,33.595701999999996],[-117.869425,33.595130999999995],[-117.866922,33.593781],[-117.86188899999999,33.591141],[-117.864058,33.589897],[-117.865574,33.587582],[-117.86614764088401,33.5873392496233],[-117.870749,33.59093],[-117.873352,33.592932999999995],[-117.87679,33.592321999999996],[-117.878044124759,33.592764990129794]]]},"id":"0590627021"}



*bad result in svg gen 2020.0608 2147*
# need to add a projection?  from skipped step 1...



geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  < ca2018bg-color.ndjson \
  > ca2018bg-color.purple.svg

xviewer ca2018bg-color.purple.svg
# seems borked for lng/lat :/



part 3 - shrink with TopoJSON
======

https://medium.com/@mbostock/command-line-cartography-part-3-1158e4c55a1e

# essentially same process as work with census block before, just changed file name
# but maybe not needed if end result was to get the county borders to aid visualization

npm install -g topojson

# **3a**
# *input is from result of 2f*
geo2topo -n \
  tracts=ca2018bg-density.ndjson \
  > ca2018bg-topo.json

toposimplify -p 1 -f \
  < ca2018bg-topo.json \
  > ca2018bg-simple-topo.json

topoquantize 1e5 \
  < ca2018bg-simple-topo.json \
  > ca2018bg-quantized-topo.json

*what was counties=tracts here means?*
topomerge -k 'd.id.slice(0, 3)' counties=tracts \
  < ca2018bg-quantized-topo.json \
  > ca2018bg-merge-topo.json


topomerge --mesh -f 'a !== b' counties=counties \
  < ca2018bg-merge-topo.json \
  > ca2018bg-topo.json

 4278961 Jun  7 09:38 ca2018bg-topo.json        # ok, this is bigger (cuz block group) # prev albers projection
 1526619 Jun  6 16:52 ../TMP_DATA/ca-topo.json  # truly map by track, which has less data.
 4182080 Jun  8 20:41 ca2018bg-topo.json        # topo json, with lng/lat 

# tried preview, but don't work.   ditto, didn't work in lng/lat view
geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  < ca2018bg-topo.json \
  > ca2018bg-topo.svg

part 4 - improve color, generate svg
======

https://medium.com/@mbostock/command-line-cartography-part-4-82d0d26df0cf

# each version below are independent of one another
# they just need input ca-topo.svg, the result of part 3 above.
# for block group level data, skippig to the last step "4e"

npm install -g d3-scale-chromatic

# **4e** OrRd color scheme, decent looking result

# borked again :/ , remove "d3=" in the -r option

# **4e fixing** actually just need to say -r d3-scale-chromatic (ie, just drop the prefix d3= )
# ref: https://medium.com/@v.brusylovets/hi-dario-yeah-after-two-years-something-is-changed-in-d3-1e4222744c93
topo2geo tracts=- \
  < ca2018bg-topo.json \
  | ndjson-map -r d3 -r d3-scale-chromatic 'z = d3.scaleThreshold().domain([1, 10, 50, 200, 500, 1000, 2000, 4000]).range(d3.schemeOrRd[9]), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' \
  | ndjson-split 'd.features' \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > ca-2018bg-threshold.svg
# no data for lng/lat version :/


# **4f** add county borders 
# instead of county borders, i think highway may better explain the density pattern.
# but county lines may still be needed to help orientation, especially San Joaquin valley?
# not if include some smaller state highway ?
(topo2geo tracts=- \
    < ca2018bg-topo.json \
    | ndjson-map -r d3 -r d3-scale-chromatic 'z = d3.scaleThreshold().domain([1, 10, 50, 200, 500, 1000, 2000, 4000]).range(d3.schemeOrRd[9]), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' \
    | ndjson-split 'd.features'; \
topo2geo counties=- \
    < ca2018bg-topo.json \
    | ndjson-map 'd.properties = {"stroke": "#000", "stroke-opacity": 0.3}, d')\
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > ca2018bg.svg
# nope :/

# ca.svg/ca2018bg.svg is final result presented on web page.
# all steps worked now, get ca map with pop density per census tracts, OrRd color scale
# need to add a color scale, which was not well explained.
# i dont think i want to deal with d3 graphics...

# PREV: cp ca.svg ca-popDensityByTract-OrRd.svg
# now:  ln ca2018bg.svg ca-popDensityByBlockGrp-OrRd.svg

xviewer ***.svg



# xref: https://mail.google.com/mail/u/2/#sent/QgrcJHrhwLQnRRMmGSkszxNZBkpDbDfHbPg
# Bkly Gdrv 

.. # use 8-space tab as that's how github render the rst
.. # vim: shiftwidth=8 tabstop=8 noexpandtab paste

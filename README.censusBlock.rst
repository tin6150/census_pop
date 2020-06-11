

2020.0606


playing with census data., this version tries to get population density at the census block level
(prev was at tracts level)

Ling is interested to see if can get pop at census tract vs block level.  
preferably density, so need aland 

ref:
https://medium.com/@mbostock/command-line-cartography-part-4-82d0d26df0cf
https://blog.mapbox.com/dive-into-large-datasets-with-3d-shapes-in-mapbox-gl-c89023ef291

maybe tidycensus in R
maybe mapbox
tbd
(prev was just reproducing blocstock tracts level map)


~~~~

part 1 - census shape to svg
======

https://medium.com/@mbostock/command-line-cartography-part-1-897aa8f8ca2c


TIGER = very detailed.
SHAPE file - much smaller, good for thematic maps (cartographically small scale)

need pop data matching shape: tract vs block

CA FIPS code = 06.
download data:

mkdir TMP_DATA_BLOCK
cd    TMP_DATA_BLOCK

curl 'https://www2.census.gov/geo/tiger/GENZ2018/shp/cb_2018_06_tract_500k.zip' -o cb_2018_06_tract_500k.zip  # shape + dbIII
# 500k is the map scale: 1:500,000 
# there is a kml version at https://www2.census.gov/geo/tiger/GENZ2018/kml/
# BG is block group
# ref: https://www2.census.gov/geo/tiger/GENZ2018/2018_file_name_def.pdf

curl 'https://www2.census.gov/geo/tiger/GENZ2018/shp/cb_2018_06_bg_500k.zip' -o cb_2018_06_bg_500k.zip  # block group shape + dbIII
unzip ...

shp2json cb_2018_06_bg_500k.shp -o ca2018bg.json
# above json is in lng/lat, truely geojson it seems)

geoproject 'd3.geoConicEqualArea().parallels([34, 40.5]).rotate([120, 0]).fitSize([960, 960], d)' < ca2018bg.json > ca2018bg-albers.json
# map project, changed coordinates to like [164.1468809671912,437.62295438355295]
# see medium post for screenshot.
# anyway, lost lng/lat by here, not good for modeling work downstream.

geo2svg -w 960 -h 960 < ca2018bg-albers.json > ca2018bg-albers.svg
# that svg file is huge, 11M, xviewer could not handle it.  (but prev handled 6.7M file)
# huge input lookup...   complain by xviewer and gimp
# firefox handled it, but sometime crash
# simplification by TopoJSON may be needed




part 2 - join shape with pop data by id
======

https://medium.com/@mbostock/command-line-cartography-part-2-c3a82c5c0f3

# xref: inet-dev-class/mapbox/eg_data_ndjson/README.txt.rst , which likely also in covid19_care_capacity_map/
# **eg 2a**  geojson to ndjson
ndjson-split 'd.features' < ca2018bg-albers.json  > ca2018bg-albers.ndjson

# prev census tract (2014?) :
{"type":"Feature","properties":{"STATEFP":"06","COUNTYFP":"075","TRACTCE":"980401","AFFGEOID":"1400000US06075980401","GEOID":"06075980401","NAME":"9804.01","LSAD":"CT","ALAND":419323,"AWATER":247501271},"geometry":{"type":"Polygon","coordinates":[[[164.1468809671912,437.62295438355295],[164.63136562909594,437.78130627883274],[164.66061334779198,437.4585472897443],[164.99020559033386,437.24058568718465],[165.18788475165627,437.5895682364189],[165.34708696199812,437.95636228142894],[165.00971718370104,438.4217441413507],[164.76560417638595,438.337767038262],[164.64069467117463,438.0862550961165],[164.22534302939806,438.0159600586103],[164.1468809671912,437.62295438355295]]]}}



# now census block group (2018) TRACTCE: 980401
{"type":"Feature","properties":{"STATEFP":"06","COUNTYFP":"075","TRACTCE":"980401","BLKGRPCE":"1","AFFGEOID":"1500000US060759804011","GEOID":"060759804011","NAME":"1","LSAD":"BG","ALAND":419323,"AWATER":247501289},"geometry":{"type":"Polygon","coordinates":[[[164.1468809671912,437.62295438355295],[164.63136562909594,437.78130627883274],[164.66061334779198,437.4585472897443],[164.99020559033386,437.24058568718465],[165.18788475165627,437.5895682364189],[165.34708696199812,437.95636228142894],[165.00971718370104,438.4217441413507],[164.76560417638595,438.337767038262],[164.64069467117463,438.0862550961165],[164.22534302939806,438.0159600586103],[164.1468809671912,437.62295438355295]]]}}


ndjson-map 'd.id = d.properties.GEOID.slice(2), d'  < ca2018bg-albers.ndjson  > ca2018bg-albers-id.ndjson


# census api to get pop 
# need census api key, see bmail.

ref: https://www.census.gov/data/developers/data-sets/acs-5year.html
B01003_001E is total population estimate 
Example Call:
many examples : https://api.census.gov/data/2014/acs/acs5/examples.html
      tract eg: https://api.census.gov/data/2014/acs/acs5?get=NAME,B00001_001E&for=tract:*&in=state:01&key=YOUR_KEY_GOES_HERE
    blk grp eg:	https://api.census.gov/data/2014/acs/acs5?get=NAME,B00001_001E&for=block%20group:*&in=state:01%20county:025&key=YOUR_KEY_GOES_HERE

			  export ApiKey=c9b728... see bmail or . .env , also in .ssh
curl "https://api.census.gov/data/2018/acs/acs5?get=NAME,B01003_001E&for=tract:*&in=state:06&key=$ApiKey" -o cb_2018_06_tract_B01003.json # track pop 2018

curl "https://api.census.gov/data/2018/acs/acs5?get=NAME,B01003_001E&for=block%20group:*&in=state:06%20county:*&key=$ApiKey" -o cb_2018_06_bg_B01003.json # pop 2018 by block group?
	# arrggg!!
	# error: wildcard not allowed for 'county' in geography heirarchy 
	

# so would need to get 1 json per county
# convert them to ndjson(s)
# and merge
# san joaquin valley counties... by census county number!!  not sure where to find such list :/

#xx nope, not by county name (need number) curl "https://api.census.gov/data/2018/acs/acs5?get=NAME,B01003_001E&for=block%20group:*&in=state:06%20county:fresno&key=$ApiKey" -o cb_2018_06_bg_B01003.fresno.json
# county # 025 is imperial county
curl "https://api.census.gov/data/2018/acs/acs5?get=NAME,B01003_001E&for=block%20group:*&in=state:06%20county:025&key=$ApiKey" -o cb_2018_06_bg_B01003.025.json

# county numbers are sequential, so not trivial to seq get countries in central valley.
# get em all then.

.. code:: bash 

for FIPS in $(seq -w 19 2 115); do
	echo curl "https://api.census.gov/data/2018/acs/acs5?get=NAME,B01003_001E&for=block%20group:*&in=state:06%20county:$FIPS&key=$ApiKey" -o cb_2018_06_bg_B01003.$FIPS.json
done

# pfff... county FIPS are  only odd numbers, 001, 003, ... 115.  58 counties total in CA.


works! but result now has extra descriptive fields in them...
cat cb_2014_06_tract_B01003.json | wc
cat cb_2014_06_tract_B01003.json | json2csv > cb_2014_06_tract_B01003.json.csv 

prev census tract 
[["NAME","B01003_001E","state","county","tract"],
["Census Tract 4382.03, Alameda County, California","4384","06","001","438203"],
["Census Tract 4382.04, Alameda County, California","5338","06","001","438204"],
  ^^extra 1^^           ^^extra 2^^^^^  ^^extra 3^^  ^#0^  ^#1  ^#2^   ^^#3^^
 ^^^^^^single^^field^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
#1, ... #3 are the orig field bostock eg refers to.  so i am no off by +1 

new census block group 1 file per county - cb_2018_06_bg_B01003.001.json:
#  offset, ----#0-----, #1----,#2------,#3,----,#4-----------
[["NAME","B01003_001E","state","county","tract","block group"],
["Block Group 2, Census Tract 4384, Alameda County, California","1623","06","001","438400","2"],
["Block Group 1, Census Tract 4384, Alameda County, California","945" ,"06","001","438400","1"],
 ^^^^^^single^^field^^^add^+1^as^offset^^^^^^^^^^^^^^^^^^^^^^^^ ^#0^   ^#1  ^#2^   ^^#3^^  +++--one more tailing column for BG
 NAME                                                          ,pop, state,county,tract---,BG
 #0 new numbering after +1 offset for NAME column              , #1,    #2,  #3  ,   #4   ,#5

now now could have multple record per tract for population.  
maybe combine #2, #3 and #4 in next step "2d"

# **eg 2d** 

#   field f1 is "id" field, combination of 3 columns: 2 and 3, 4, merged, no space.  0-idx, add +1 as offset cuz NAME field from new census api
#   field f2 is "B01003" (pop estimate, name from census var name): use column 0  [add +1 offset] [same as prev tract data]
#   dont have State FIPS in it cuz always CA (06)

# **fiexed 2d** 
ndjson-cat cb_2018_06_bg_B01003.001.json \
  | ndjson-split 'd.slice(1)' \
  | ndjson-map '{id: d[3] + d[4] +d[5], B01003: +d[1]}'  >        cb_2018_06_bg_B01003.001.ndjson
#                    ^^^^f1^^^^^                ^^f2^^

#   ndjson has key: value pair, field 1 key is "id: ', field 2 key is "B01003: '
# result is this, which looks like what bostock expect
# prev (census tract):
{"id":"001438203","B01003":4384}
{"id":"001438204","B01003":5338}
{"id":"001438300","B01003":4133}

# new (census block group):
{"id":"0014441003","B01003":1755}
{"id":"0014441002","B01003":1320}

# **2d addition** combine multiple counties ndjson into a single one

# first repeat 2d for all counties

for FIPS in $(seq -w 001 2 115); do
  ndjson-cat cb_2018_06_bg_B01003.$FIPS.json  | ndjson-split 'd.slice(1)' | ndjson-map '{id: d[3] + d[4] +d[5], B01003: +d[1]}'  >  cb_2018_06_bg_B01003.$FIPS.ndjson
done

# 58 ndjson files, combine into single one.  (23212 lines, match prev wc sum for all counties)
cat cb_2018_06_bg_B01003.???.ndjson > cb_2018_06_bg_B01003.CA.ndjson


# **eg 2e** 

ndjson-join 'd.id' \
  ca2018bg-albers-id.ndjson \
  cb_2018_06_bg_B01003.CA.ndjson \
  > ca2018bg-albers-join.ndjson

# previously  borked here.  fixed now.  prev eg for tract level data (should be only 1 rec for TRACTCE 400300):
# [{"type":"Feature","properties":{"STATEFP":"06","COUNTYFP":"001","TRACTCE":"400300","AFFGEOID":"1400000US06001400300","GEOID":"06001400300","NAME":"4003","LSAD":"CT","ALAND":1105329,"AWATER":0},"geometry":{"type":"Polygon","coordinates":[[[224.3021507494117,425.1613296471837],[224.4889212459765,425.02853000146524],[224.8054892227229,424.90924473882023],[225.09157727394734,424.797926817982],[225.29373002719294,424.7042420166931],[225.65996339344974,424.52901179192713],[225.95108431320563,424.3385241647384],[225.912059937863,424.3983338513344],[225.81079279254033,424.6100213459463],[225.58249395352414,425.05059707011105],[225.35882837057437,425.47619464326226],[225.22516372508392,425.73538936106115],[224.86658222608307,425.5294755512],[224.63434603931907,425.4732297669584],[224.43926884491924,425.4361850983005],[224.44504485979195,425.3811563562076],[224.37116077415172,425.3749388649712],[224.17960589902756,425.397389513148],[224.3021507494117,425.1613296471837]]]},"id":"001400300"},{"id":"001400300","B01003":5428}]

# new eg for blocks group level data, seems like 4 rec for TRACTCE 400300, only showing first one
# new field BLKGRPCE added.  but not consequential, just need population data (B01003) and ALAND (land area) in the next step "2f"
# [{"type":"Feature","properties":{"STATEFP":"06","COUNTYFP":"001","TRACTCE":"400300","BLKGRPCE":"2","AFFGEOID":"1500000US060014003002","GEOID":"060014003002","NAME":"2","LSAD":"BG","ALAND":269347,"AWATER":0},"geometry":{"type":"Polygon","coordinates":[[[224.44636755999747,425.3691423744949],[224.5939145191771,425.21708244189904],[224.7830672777208,425.34538644862505],[225.07612515752876,425.52808473848654],[225.1067691336628,425.4894377365358],[225.31806810037276,425.1724692650978],[225.58249395352414,425.05059707011105],[225.35882837057437,425.47619464326226],[225.22516372508392,425.73538936106115],[224.86658222608307,425.5294755512],[224.6649471804986,425.47993141313145],[224.43926884491924,425.4361850983005],[224.44636755999747,425.3691423744949]]]},"id":"0014003002"},{"id":"0014003002","B01003":1404}]



# **2f**

ndjson-map 'd[0].properties = {density: Math.floor(d[1].B01003 / d[0].properties.ALAND * 2589975.2356)}, d[0]' \
  < ca2018bg-albers-join.ndjson \
  > ca2018bg-albers-density.ndjson

# **2g**

ndjson-reduce \
  < ca2018bg-albers-density.ndjson \
  | ndjson-map '{type: "FeatureCollection", features: d}' \
  > ca2018bg-albers-density.json

# eg result:
{"type":"Feature","properties":{"density":0},"geometry":{"type":"Polygon","coordinates":[[[164.1468809671912,437.62295438355295],[164.63136562909594,437.78130627883274],[164.66061334779198,437.4585472897443],[164.99020559033386,437.24058568718465],[165.18788475165627,437.5895682364189],[165.34708696199812,437.95636228142894],[165.00971718370104,438.4217441413507],[164.76560417638595,438.337767038262],[164.64069467117463,438.0862550961165],[164.22534302939806,438.0159600586103],[164.1468809671912,437.62295438355295]]]},"id":"0759804011"}
{"type":"Feature","properties":{"density":5440},"geometry":{"type":"Polygon","coordinates":[[[583.3067862409382,854.8717329876263],[583.4813178511808,854.6585101562487],[583.7779327272376,854.7977310974125],[583.9655380355614,854.5602629991972],[584.0269325681705,854.6165658408554],[584.240683477404,854.7481359034291],[584.670342625474,855.0051488986787],[584.4905937377472,855.1348697365938],[584.3682429572099,855.3714858612866],[584.3205053355547,855.3970830898411],[583.9250158061104,855.0432402918268],[583.701365900805,854.8459183147456],[583.4133542298542,854.9140974854508],[583.3067862409382,854.8717329876263]]]},"id":"0590627021"}


# **2h alt**
# using the alternate method as it works reliably for me
ndjson-reduce 'p.features.push(d), p' '{type: "FeatureCollection", features: []}' \
  < ca2018bg-albers-density.ndjson \
  > ca2018bg-albers-density.json
# ca2018bg-albers-density.json is a geojson, so
ln ca2018bg-albers-density.json ca2018bg-albers-density.geojson
# result is viewable in mapshaper.org, but file isn't truly a geojson?
# but that only display map data (census block group outlines?), and density is not colored in.


npm install -g d3

# **2i**

ndjson-map -r d3 \
  '(d.properties.fill = d3.scaleSequential(d3.interpolateViridis).domain([0, 4000])(d.properties.density), d)' \
  < ca2018bg-albers-density.ndjson \
  > ca2018bg-albers-color.ndjson



geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  < ca2018bg-albers-color.ndjson \
  > ca2018bg-albers-color.purple.svg

xviewer ca-albers-color.purple.svg  # work, but ugly purple map.



part 3 - shrink with TopoJSON
======

https://medium.com/@mbostock/command-line-cartography-part-3-1158e4c55a1e

# essentially same process as work with census block before, just changed file name
# but maybe not needed if end result was to get the county borders to aid visualization

npm install -g topojson

# **3a**
geo2topo -n \
  tracts=ca2018bg-albers-density.ndjson \
  > ca2018bg-tracts-topo.json

toposimplify -p 1 -f \
  < ca2018bg-tracts-topo.json \
  > ca2018bg-simple-topo.json

topoquantize 1e5 \
  < ca2018bg-simple-topo.json \
  > ca2018bg-quantized-topo.json

topomerge -k 'd.id.slice(0, 3)' counties=tracts \
  < ca2018bg-quantized-topo.json \
  > ca2018bg-merge-topo.json


topomerge --mesh -f 'a !== b' counties=counties \
  < ca2018bg-merge-topo.json \
  > ca2018bg-topo.json

 4278961 Jun  7 09:38 ca2018bg-topo.json        # ok, this is bigger...
 1526619 Jun  6 16:52 ../TMP_DATA/ca-topo.json

# tried preview, but don't work.  
geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  < ca-topo.json \
  > ca-topo.svg

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

# ca.svg/ca2018bg.svg is final result presented on web page.
# all steps worked now, get ca map with pop density per census tracts, OrRd color scale
# need to add a color scale, which was not well explained.
# i dont think i want to deal with d3 graphics...

# PREV: cp ca.svg ca-popDensityByTract-OrRd.svg
# now:  ln ca2018bg.svg ca-popDensityByBlockGrp-OrRd.svg

xviewer ***.svg



# xref: https://mail.google.com/mail/u/2/#sent/QgrcJHrhwLQnRRMmGSkszxNZBkpDbDfHbPg
# Bkly Gdrv prj/census  for some screenshots, svg and geojson.  slightly renamed compared to the TMP_DATA* folders on Zink 
# most content placed in Medium post.  
# Friends link (not metered): https://medium.com/@tin_ho/command-line-cartography-census-blockgroup-level-data-5aa222de8ab0?sk=07861cc1e0d1ef0e4e36be452302ffd5
# metered.  stripped ?sk=...  https://medium.com/@tin_ho/command-line-cartography-census-blockgroup-level-data-5aa222de8ab0
# old medium username tin_2041, new tin_ho.  (linked to ucb email, actually not connected to tweeter yet).

.. # use 8-space tab as that's how github render the rst
.. # vim: shiftwidth=8 tabstop=8 noexpandtab paste

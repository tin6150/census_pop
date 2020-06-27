Census Population Map
~~~~~~~~~~~~~~~~~~~~~

Population density from Census data
- Population density by Census Block Group 
- Population density by Census Tract

URL: https://tin6150.github.io/census_pop/census_map_medium.html#8/37.7535/-122.44


Write Up
========

Post in Medium:
https://medium.com/@tin_ho/command-line-cartography-census-blockgroup-level-data-5aa222de8ab0?sk=07861cc1e0d1ef0e4e36be452302ffd5


Previous README for various census data:

* README.rst             - this general ReadMe
* README.censusTract.rst - 1st round, replicating Bostock commands, pop density by Census Tract
* README.censusBlock.rst - 2nd round, pop density by Census Block Group (in CA Albers projection)
* README.lngLat.rst      - 3rd round, variant of 2, without Albers projection, just use plain Longitude,Latitude coordinate
* README.lngLat2010BG.rst - 4th round, variant of 3, use Census 2010 data.  have pop density, pop count and ALAND in final geojson
* README.devNote.rst     - commands to aid in development.


DATA SOURCES
============

- Census Estimate for 2018
- Mapbox



ABOUT
=====

(cc) copyleft BY-SA.
Tin Ho [tin (at) berkeley.edu]

Special thanks to Mapbox for generously providing free tier usage for their map platform!



.. # use 8-space tab as that's how github render the rst
.. # vim: shiftwidth=8 tabstop=8 noexpandtab paste 

Census Population Map
~~~~~~~~~~~~~~~~~~~~~

Population density from Census data
- Population density by Census Block Group 
- Population density by Census Tract


Dev Env
=======

To avoid CORS error (since html need to load a .geojson), run a simple web server from the d
irectory containing the files of the project ::

        python  -m SimpleHTTPServer 8000  # If Python version returned above is 2.X
        python3 -m http.server      8000  # If Python version returned above is 3.X

Then on browser, navigate to http://localhost:8000 




Ref
===

 
* Example geoJSON: https://www.mapbox.com/help/data/stations.geojson
* Additional ref: https://www.mapbox.com/help/define-geojson/

* html based on covidtracking_care_capacity_map.dev.html




.. # use 8-space tab as that's how github render the rst
.. # vim: shiftwidth=8 tabstop=8 noexpandtab paste 

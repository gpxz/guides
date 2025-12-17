# Python GPXZ elevation data guide

An overview of querying and analysing GPXZ elevation data using python.

## Authentication

You'll need an API key to use the GPXZ elevation API. This requirement cuts down on misuse and helps us make sure there's enough server capacity for everyone. But it's easy and free to make an account and it comes with lots of quota!

Register for a free GPXZ account here: [gpxz.io/app/accounts/signup/](https://www.gpxz.io/app/accounts/signup/)

Once you're signed up, you should see your API key displayed on the dashboard. It will be a bunch of random characters starting with `ak_`, for example: `ak_k2g47_gd9ni6818xj6`. We'll use the key `ak_demo_apikey` in this guide to make it clear it's not a live key.

The easiest way to authenticate queries is to use the key as a URL query argument. Basically, adding `&api-key=ak_demo_apikey` to the end of your URLs. This approach makes it easy to test queries in the browser. Try visiting the URL below, then replace `ak_demo_apikey` with your own API key:

[api.gpxz.io/v1/elevation/point?lat=44.841&lon=-0.569&api-key=ak_demo_apikey](https://api.gpxz.io/v1/elevation/point?lat=44.841&lon=-0.569&api-key=ak_demo_apikey)

You should see a successful response, showing the elevation of the Miroir d'eau in Bordeaux, France as about 5 metres.

```json
{
  "result": {
    "elevation": 5.134736,
    "lat": 44.841,
    "lon": -0.569,
    "data_source": "france_1m_dtm",
    "resolution": 1
  },
  "status": "OK"
}
```

Now you're ready to start exploring elevation data with python!

One final note: while the URL-based API key is convenient for debugging, it's recommended to eventually move to adding the key as an HTTP header instead. URLs have a tendency to be logged, displayed, and shared, which makes it harder to keep your API key a secret. HTTP headers don't have these problems. We'll cover header-based authentication with python shorly.


## Querying the elevation for a single point

To query the elevation of a single point with the GPXZ API you'll need a latitude and a longitude. If you're doing this manually, you can fit the coordinates of a point by right-clicking it on Google Maps. For example, right-clicking the Sydney Opera House in Google Maps gives the coordinates `-33.85777731669462, 151.2148714376462`.

Some notes about latitude and longitude coordinates:

* Most software will give coordinates in `latitude,longitude` order. But it's still fairly common to represent coordinates the other way around, with logitude first! Make sure you check which order your data is in. Checking a location in Australia or California is one way to do this, as a latitude can't be less than -90 or greater than 90.
* GPXZ uses the common convention of longitudes being between -180 and 180. However, some sofware allows longitudes to "wrap" outside of this range. We'll cover fixing this later in the guide.
* Some software will give lots of decimal points for lat/lon coordinates. But unless you're trying to locate individual atoms, this level of precision is uneccesary! The GPXZ dataset currently goes down to 50cm of resolution, where 7 decimal places of latitude and longitude is more than sufficient. 

Now, lets finally get python to query this for us. We can use the `/v1/elevation/point` endpoint for this (here's a [list of all the GPXZ endpoints](https://www.gpxz.io/docs/api-reference) for reference).

```python
# Import the libraries we need.
import requests

# Define the data.
#
# Remember to replace with your own API key!
gpxz_api_key = "ak_demo_apikey"
latitude = -33.857
longitude = 151.214
endpoint = "https://api.gpxz.io/v1/elevation/point"

# Set up the query.
#
# The data variables can be called whatever you like, 
# but the string keys (lat, lon, api_key) must match the API docs exactly.
params = {
    "lat": latitude,
    "lon": longitude,
    "api_key": gpxz_api_key,
}

# Make the request.
response = requests.get(
    endpoint,
    params=params,
)

# Convert the json text response into a python dictionary.
data = response.json()
print(data)

# {'result': {'elevation': 3.214107, 'lat': -33.857, 'lon': 151.214, 'data_source': 'australia_5m', 'resolution': 5}, 'status': 'OK'}
```

### Pretty printing the result

The example above spits out everything on one line. This will get tricky to view as we move to more complex queries.

For slightly nicer formatting, we can use the builtin `json` library to view the response more neatly.

```python
import json

print(json.dumps(data, indent=4))
# {
#     "result": {
#         "elevation": 3.214107,
#         "lat": -33.857,
#         "lon": 151.214,
#         "data_source": "australia_5m",
#         "resolution": 5
#     },
#     "status": "OK"
# }
```


### Using the builtin urllib library

The examples in this guide all use the `requests` library. You can install it with the command `pip install requests` and it's so common that it might already be installed in your environment!

But if you'd like to use the `urllib` library that comes preinstalled with python instead, you can modify the initial example like this

```python
from urllib.parse import urlencode
from urllib.request import urlopen
import json

# Define the data and build the query parameters as before.
gpxz_api_key = "ak_demo_apikey"
latitude = -33.857
longitude = 151.214
endpoint = "https://api.gpxz.io/v1/elevation/point"
params = {
    "lat": latitude,
    "lon": longitude,
    "api_key": gpxz_api_key,
}

# With urllib we have to maually build the url.
query_string = urlencode(params)
url = endpoint + "?" + query_string

# Make the request and parse the response.
with urlopen(url) as response:
    data = json.load(response)
```

You have to do a little more work than with the `requests` library, but all the examples in this guide can be rewritten to work with urllib.


### Validating the API response

It's good practice to validate the HTTP status code of the response.

This is a number returned with API responses that will be `200` when successful, and something else if unsuccessful. The `requests` library has the `raise_for_status()` method that will throw an arror for non-successful status codes. That prevents your code from continuing to operate on invalid data!

```python
# Make the request and validate the response.
response = requests.get(
    endpoint,
    params=params,
)
response.raise_for_status()
```



### Wrapping longitudes

GPXZ requires longitudes to be between -180 and 180. However you may come across different definitions: for example, the longitude `511.2` or `-208.8` can sometimes represent Sydney's longitude of `151.2`. To convert to the ±180 format, keep adding/removing 360 from the value until it falls into the correct range.

```python
def wrap_lon(lon: float) -> float:
    """Wrap longitude between -180 and 180."""
    while lon > 180:
        lon -= 360
    while lon < -180:
        lon += 360
    return lon

print(wrap_lon(511.2))
# 151.2
```

### Including the API key as a header

It's fine to place API keys in the URL when debugging, but if moving to production it's better to transmit them in a header. GPXZ uses the `x-api-key` header for authentication.

The `requests` library makes this simple:

```python
# Set up the query.
params = {
    "lat": latitude,
    "lon": longitude,
}
headers = {"x-api-key": gpxz_api_key}

# Make the request.
response = requests.get(
    endpoint,
    params=params,
    headers=headers,
)
```


## Requesting the elevation of multiple points

GPXZ lets you query multiple points in a single query. A multi-point query still only counts as a single request against your quota, and is a lot faster than querying single points in a loop!

A multi-point request is also a natural way of querying multi-point data, like a route made up of multile GPS location readings. And unlike the Google Maps Elevation API, GPXZ doesn't reduce the accuracy of data queried in batches.

We'll switch to the `/v1/elevation/points` endpoint for this. 
* GPXZ calls its coordinates parameter `latlons` to make it clear that the coordinates should be in latitude-then-longitude order.
* Points should be separated by the pipe character `|`.

Here's a browser example of a short segment along Abbey Road in London:

[api.gpxz.io/v1/elevation/points?latlons=51.5317,-0.1770|51.5378,-0.1846|51.5398,-0.1872&api-key=ak_demo_apikey
](51.5317,-0.1770|51.5378,-0.1846|51.5398,-0.1872
)


And here's that same example in python:

```python
import json
import requests

# Define the data.
gpxz_api_key = "ak_demo_apikey"
endpoint = "https://api.gpxz.io/v1/elevation/points"
latlon_coords = [
    (51.5317, -0.1770),
    (51.5378, -0.1846),
    (51.5398, -0.1872), 
]

# Build the query.
#
# Coords are separated by ',' points by '|'.
latlon_string = ""
for point in latlon_coords:
    latlon_string += str(point[0])
    latlon_string += ","
    latlon_string += str(point[1])
    latlon_string += "|"
latlon_string = latlon_string.rstrip("|")
params = {"latlons": latlon_string}
headers = {"x-api-key": gpxz_api_key}

# Make the request.
response = requests.get(
    endpoint,
    params=params,
    headers=headers,
)
response.raise_for_status()

# View the result.
data = response.json()
print(json.dumps(data, indent=4))

# {
#     "results": [
#         {
#             "elevation": 43.950146,
#             "lat": 51.5317,
#             "lon": -0.177,
#             "data_source": "england_2022_1m_dtm",
#             "resolution": 1
#         },
#         {
#             "elevation": 35.387238,
#             "lat": 51.5378,
#             "lon": -0.1846,
#             "data_source": "england_2022_1m_dtm",
#             "resolution": 1
#         },
#         {
#             "elevation": 37.162411,
#             "lat": 51.5398,
#             "lon": -0.1872,
#             "data_source": "england_2022_1m_dtm",
#             "resolution": 1
#         }
#     ],
#     "status": "OK"
# }
```

With the result as a dictionary, you can extract the information you need. For example, to pull out a list of elevations:

```python
elevations = [result["elevation"] for result in data["results"]]
```


### Requesting a single point with the multi-point endpoint


The `/v1/elevation/points` endpoint does accept a single point! Just use `latlons=51.5398,-0.1872` instead of `lat=51.5398&lon=-0.1872`, and keep in mind you'll still get a list of results (with a single result element!)


### Batching points

The points endpoint is limited to 512 points per request. If you have more points than that (or need to handle user-generated point collections that *may* be larger than that), you'll need to split your points into multiple batches of 512.


gpxz_batch_size = 512

```python
import random

import requests

# For this example, randomly generate coordinates.
n_points = 2000
gpxz_batch_size = 512
lats = [random.uniform(-90, 90) for _ in range(n_points)]
lons = [random.uniform(-180, 180) for _ in range(n_points)]
latlons = zip(lats, lons)

# Simple function to split a list into batches.
def batch_list(items: list, batch_size: int) -> list[list]:
    batches = []
    index = 0
    while index < len(items):
        batch = items[index: index + batch_size]
        batches.append(batch)
        index += batch_size

    return batches

# Make the batch requests.
# 
# Merge all the result objects into one list.
results = []
headers = {"x-api-key": gpxz_api_key}
for latlon_batch in batch_list(latlons):
    latlon_string = "|".join(",".join(map(str, x)) for x in latlon_batch)
    params = {"latlons": latlon_string}
    response = requests.get(
        endpoint,
        params=params,
        headers=headers,
    )
    response.raise_for_status()
    results += response.json()["results"]
```


If you have numpy installed, the `array_split` function saves you having to write your own:

```python
import numpy as np

...

latlon_batches = np.array_split(latlons, gpxz_batch_size)
```


### POST requests

Stuffing all these coordinates in a query argument can result in some rather large URLs, especially if you're adding lots of decimal places!

The GPXZ can accept rather large URLs (up to about 60,000 charaters long!) but not all sofware is so forgiving. Any browser, HTTP request library, or proxy you're using may impose a lower limit.

To avoid such issues, the `/v1/elevation/points` endpoint also supports POST requests. Simply convert any query arguments to POST data instead:

```python
import requests

gpxz_api_key = "ak_demo_apikey"
endpoint = "https://api.gpxz.io/v1/elevation/points"
latlon_string = "51.5317,-0.177|51.5378,-0.1846|51.5398,-0.1872"

params = {"latlons": latlon_string}
headers = {"x-api-key": gpxz_api_key}

response = requests.post(
    endpoint,
    data=params,
    headers=headers,
)
response.raise_for_status()
data = response.json()
```


### Polyline requests

You can also convert your coordinates to a [polyline](https://developers.google.com/maps/documentation/utilities/polylinealgorithm) string.

```python
import polyline
import requests

# Data setup.
gpxz_api_key = "ak_demo_apikey"
endpoint = "https://api.gpxz.io/v1/elevation/points"
latlon_coords = [
    (51.5317, -0.1770),
    (51.5378, -0.1846),
    (51.5398, -0.1872), 
]

# Build the polyline.
#
# Increase precision above the default 5 for hires elevation data.
polyline_string = polyline.encode(latlon_coords, precision=6)

# Make the request using the `polyline` arg rather than the `latlons` one.
params = {"polyline": polyline_string}
headers = {"x-api-key": gpxz_api_key}
response = requests.get(
    endpoint,
    params=params,
    headers=headers,
)
response.raise_for_status()
data = response.json()
```

## Migrating from the Google Maps Elevation API

Migrating from Google Maps to GPXZ is easy! Simply replace `https://maps.googleapis.com/maps/api/elevation` with `https://api.gpxz.io/v1/elevation/gmaps-compat` and update your API key.

Take this [example from Air Sciences Inc.](https://airsci.com/google-maps-elevation-api-python-example/), which is loading a shapefile and adding elevation data to it.

```python
import matplotlib.pyplot as plt
import geopandas as gpd
import requests
from shapely.geometry import Point
from os.path import join as pth_join
import json
import time

#Configure API request link
url_root = "https://maps.googleapis.com/maps/api/elevation"
key_str = "GFVerAfCoyBsdfPD7nnW8QQC7ZZt5ytKiCO4e3Zu"
output_fmt = "json"
request_str = "%s/%s?locations=%s&key=%s"

#Define data inputs and outputs
data_root = "/Users/airsci/git/wrap-cmfd/CM_data"
input_file = "wrap_oregon_odf_clean.shp"
output_file = input_file[:-4]+"_elev.shp"

#Read input file into a GeoDataFrame
gdf = gpd.read_file(pth_join(data_root,input_file))
gdf_wgs = gdf.to_crs("EPSG:4326")

#Extract x,y from geometry and convert to strings
build_str = gdf_wgs['geometry'].apply(lambda v: ("%s,%s|" % (str(v.y), str(v.x))))

#Build concatenated string for API request
locs = "".join([x for x in build_str]).rstrip('|')

#Make request, parse results
r = requests.get(request_str % (url_root, output_fmt, locs, key_str))
parsed = json.loads(r.text)

#Extract elevations from result
geom = [Point(
    i['location']['lng'], i['location']['lat'],
    i['elevation']) for i in parsed['results'],
]
```

Without changing any of the complex geospatial and data wrangling logic, you can migrate to GPXZ's high-quality elevation data by changing just the API key end endpoint:

```python
url_root = "https://api.gpxz.io/v1/elevation/gmaps-compat"
key_str = "ak_demo_apikey"  # GPXZ API key.
```

Everything else about the parsing will remain the same: the same input parameters, and the same output format.

## Migrating from opentopodata

It's also easy to migrate from opentopodata: perhape you're running into rate-limiting or CORS restrictions on opentopodata, or just need higher quality and resolution elevation data.

Again, just replace the endpoint (`/v1/elevation/otd-compat/<dataset>`) with the GPXZ equivalent `https://api.gpxz.io/v1/elevation/otd-compat`, and add you API key.

Because GPXZ uses the best sources available at all locations, there's no need to specify your dataset like with opentopodata. So for example, if you're using the mapzen dataset with `/v1/elevation/otd-compat/mapzen` you would still use  `https://api.gpxz.io/v1/elevation/otd-compat`.

## Querying source info

Every response from the GPXZ API for elevation data includes a `data_source` attribute, which indicates the provenance of the elevation data used. 

The `data_source` value is a short human-readable ID. To get the full information about that source, you can query the `/v1/elevation/sources` endpoint.

```python
import requests
import json

gpxz_api_key = "ak_demo_apikey"
headers = {"x-api-key": gpxz_api_key}

# Define some different points around the world.
latlon_coords = [
    (-43.508, 172.633),
    (22.395, 114.165),
    (51.915, 5.380), 
]
latlon_string = "|".join(",".join(map(str, x)) for x in latlon_coords)
params = {"latlons": latlon_string}

# Query elevation, saving the results as a list of results.
response_elevation = requests.get(
    "https://api.gpxz.io/v1/elevation/points",
    params=params,
    headers=headers,
)
response_elevation.raise_for_status()
data_elevation = response_elevation.json()
results = data_elevation["results"]

# Also query the source info.
#
# Convert the result list into a dict for easy lookups.
response_sources = requests.get(
    "https://api.gpxz.io/v1/elevation/sources",
    headers=headers,
)
response_sources.raise_for_status()
data_sources = response_sources.json()
source_infos = {x["data_source"]: x for x in data_sources["results"]}

# Modify the elevation results, adding the respective source information.
for result in results:
    result["source_info"] = source_infos[result["data_source"]]

# Pretty print the result.
print(json.dumps(results, indent=4))
# [
#     {
#         "elevation": 8.153094,
#         "lat": -43.508,
#         "lon": 172.633,
#         "data_source": "nz_1m_dtm",
#         "resolution": 1,
#         "source_info": {
#             "data_source": "nz_1m_dtm",
#             "name": "LINZ 1m Lidar",
#             "resolution": 1,
#             "url": "https://data.linz.govt.nz/layers/category/elevation/",
#             "organization": "Land Information New Zealand",
#             "attribution": "CC-BY-4.0: LINZ",
#             "licence": "Creative Commons Attribution 4.0 International",
#             "licence_url": "https://creativecommons.org/licenses/by/4.0/"
#         }
#     },
#     {
#         "elevation": 314.017609,
#         "lat": 22.395,
#         "lon": 114.165,
#         "data_source": "hong_kong_50cm_dtm",
#         "resolution": 0.5,
#         "source_info": {
#             "data_source": "hong_kong_50cm_dtm",
#             "name": "Hong Kong 50cm DTM 2020",
#             "resolution": 0.5,
#             "url": "https://geodata.gov.hk/gs/view-dataset?uuid=cb6c4d1e-fc38-46cc-974a-77253a773592&sidx=0",
#             "organization": "Hong Kong Civil Engineering and Development Department",
#             "attribution": "Hong Kong Geodata Store",
#             "licence": "Hong Kong Geodata Store Terms",
#             "licence_url": "https://geodata.gov.hk/gs/?p=terms_and_conditions"
#         }
#     },
#     {
#         "elevation": 3.762188,
#         "lat": 51.915,
#         "lon": 5.38,
#         "data_source": "netherlands_50cm_dtm",
#         "resolution": 2,
#         "source_info": {
#             "data_source": "netherlands_50cm_dtm",
#             "name": "Netherlands 50cm AHN4 DTM",
#             "resolution": 2,
#             "url": "https://www.ahn.nl/ahn-4",
#             "organization": "Actueel Hoogtebestand Nederland",
#             "attribution": "CC0",
#             "licence": "CC0 1.0 Universal",
#             "licence_url": "https://creativecommons.org/publicdomain/zero/1.0/"
#         }
#     }
# ]
```



## Querying rasters

As well as points, the GPXZ API also lets you query 2D rasters.

You'll need to define the rectangular bounding box of your raster, which must be less than 10km2.

```python
import requests

# Define the bounding box.
lat_min = -36.879
lat_max = -36.871
lon_min = 174.761
lon_max = 174.768
params = {
    "bbox_left": lon_min,
    "bbox_right": lon_max,
    "bbox_bottom": lat_min,
    "bbox_top": lat_max,
}

# And where we want to save the output file.
output_path = "/tmp/gpxz-demo-raster.tif"

# Otherwise make the request as usual.
gpxz_api_key = "ak_demo_apikey"
headers = {"x-api-key": gpxz_api_key}
response = requests.get(
    "https://api.gpxz.io/v1/elevation/hires-raster",
    params=params,
    headers=headers,
)
response.raise_for_status()

# Then save the raster.
with open(output_path, "wb") as f:
    f.write(response.content)
```

The resulting raster will be in a UTM projection, with a 1m resolution. Check out the [raster endpoint documentation](https://www.gpxz.io/docs/api-reference-raster) for the many different options to configure your output raster.


### Streaming rasters to disk

The above approach will load the entire raster into memory, then write it to disk once the HTTP request has completed. To reduce memory impact for large rasters, you can instead *stream* the response to disk (writing the file directly to disk):


```python
import requests
import shutil

...

 with requests.get(url, stream=True) as r:
    with open(output_path, "wb") as f:
        shutil.copyfileobj(r.raw, f)
```

### Loading rasters directly with rasterio

If using the amazingly-helpful rasterio library, it can download rasters directly. The only catch is you'll have to put all the request parameters (including your API key) into the URL. 

```python
from urllib.parse import urlencode

import raster

# Built the url.
endpoint = "https://api.gpxz.io/v1/elevation/hires-raster"
gpxz_api_key = "ak_demo_apikey"
params = {
    "bbox_left": lon_min,
    "bbox_right": lon_max,
    "bbox_bottom": lat_min,
    "bbox_top": lat_max,
    "api_key": gpxz_api_key,
}
query_string = urlencode(params)
url = endpoint + "?" + query_string

# Rasterio is smart enough to read URLs directly. GDAL may need prefixing
# with the "magic" file system token `/vsicurl/`
with rasterio.open(url) as f:
    z_array = f.read(1)
```


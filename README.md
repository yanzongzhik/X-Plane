Tools for working with X-Plane airport data
============================================================================================================================================================

[![CircleCI](https://circleci.com/gh/X-Plane/xplane_airports/tree/main.svg?style=svg)](https://circleci.com/gh/X-Plane/xplane_airports/tree/main)

`xplane_airports` is a Python package for interacting with [X-Plane](https://www.x-plane.com)'s airport data ([`apt.dat`](https://developer.x-plane.com/article/airport-data-apt-dat-file-format-specification/)) files.

This includes the following major components:

1. [The `AptDat` module](#the-aptdat-module): Used to parse & query raw `apt.dat` files (e.g., the files stored on disk in your X-Plane installation)
    - The [`AptDat`](#aptdataptdat) class itself: A parser for X-Plane's airport data files (which may contain more than 35,000 airports); a collection of [`Airport`](#aptdatairport) objects
    - The [`Airport`](#aptdatairport) class: Represents an individual airport from an `apt.dat` file.
2. [The `gateway` module](#the-gateway-module): Used to interact with [the X-Plane Scenery Gateway](https://gateway.x-plane.com) to get information about what airports are available, and to download individual scenery packs contributed by the community.
    - [`airports()`](#xplane_airportsgatewayairports---dict): Queries for metadata on all 35,000+ airports on the Gateway. 
    - [`airport()`](#xplane_airportsgatewayairportairport_id---dict): Queries the Gateway for information about the specified airport itself, as well as metadata on all scenery packs submitted for it. Unlike [`scenery_pack()`](#xplane_airportsgatewayscenery_packpack_to_download---gatewayapt), though, this does *not* include actual `apt.dat` or DSF data. 
    - [`scenery_pack()`](#xplane_airportsgatewayscenery_packpack_to_download---gatewayapt): Downloads either the recommended pack for the specified airport, or the scenery pack with the specified `int` ID. Includes both the `apt.dat` data and DSF, where applicable.
    - [`recommended_scenery_packs()`](#xplane_airportsgatewayrecommended_scenery_packsselective_apt_idsnone---collectionsiterablegatewayapt): A generator equivalent to calling [`scenery_pack()`](#xplane_airportsgatewayscenery_packpack_to_download---gatewayapt) to download the recommended scenery pack for every airport (or only a preselected list of airports, at your discretion).

## Table of Contents

* [Installation instructions](#installation-instructions)
* [Sample code](#sample-code)
  + [Parsing the default apt.dat file in your local X-Plane installation](#parsing-the-default-aptdat-file-in-your-local-x-plane-installation)
  + [Getting metadata on airports from the Gateway](#getting-metadata-on-airports-from-the-gateway)
  + [Downloaded the recommended scenery pack for an airport from the Gateway](#downloaded-the-recommended-scenery-pack-for-an-airport-from-the-gateway)
* [The `AptDat` module](#the-aptdat-module)
  + [AptDat.AptDat](#aptdataptdat)
  + [AptDat.Airport](#aptdatairport)
  + [AptDat.AptDatLine](#aptdataptdatline)
  + [AptDat.RunwayType](#aptdatrunwaytype)
* [The `gateway` module](#the-gateway-module)
  + [gateway.GatewayApt](#gatewaygatewayapt)
  + [gateway.GatewayFeature](#gatewaygatewayfeature)
  + [API wrapping functions](#api-wrapping-functions)
    - [`xplane_airports.gateway.airport`(_airport\_id_) -> dict](#xplane_airportsgatewayrecommended_scenery_packsselective_apt_idsnone---collectionsiterablegatewayapt)
    - [`xplane_airports.gateway.airports`() -> dict](#xplane_airportsgatewayairports---dict)
    - [`xplane_airports.gateway.recommended_scenery_packs`(_selective\_apt\_ids=None_) -> collections.Iterable\[[GatewayApt](#gatewaygatewayapt)\]](#xplane_airportsgatewayrecommended_scenery_packsselective_apt_idsnone---collectionsiterablegatewayapt)
    - [`xplane_airports.gateway.scenery_pack`(_pack\_to\_download_) -> [GatewayApt](#gatewaygatewayapt)](#xplane_airportsgatewayscenery_packpack_to_download---gatewayapt)
* [Migration notes](#migration-notes)
* [Running the tests (for maintainers)](#running-the-tests-for-maintainers)
* [Publishing package updates to PyPI (for maintainers)](#publishing-package-updates-to-pypi-for-maintainers)

## Installation instructions

`xplane_airports` requires Python 3.6 or newer.

Install via pip with:

`$ pip install xplane_airports`

If you're migrating from a pre-4.0 version of the library, see the [Migration notes](#migration-notes) section below.

## Sample code

### Parsing the default apt.dat file in your local X-Plane installation

```python
from xplane_airports.AptDat import AptDat, Airport
from pathlib import Path

xplane_installation = Path(input("Path to your X-Plane installation: "))
assert xplane_installation.is_dir(), f"{xplane_installation} does not exist or is not a directory"
print("Reading 35,000+ airports from disk (this will take awhile)")
default_xplane_apt_dat = AptDat(xplane_installation / 'Resources/default scenery/default apt dat/Earth nav data/apt.dat')
print(f"{len(default_xplane_apt_dat)} airports found in your default apt.dat\n")

ksea = default_xplane_apt_dat['KSEA']
""":type ksea: Airport"""
print("KSEA's airport data on disk begins:")
print(ksea.head())
```

### Getting metadata on airports from the Gateway

```python
from xplane_airports.gateway import airports
all_apts = airports()
print("There are %d airports on the X-Plane Scenery Gateway" % len(all_apts))
print("KSEA has the following metadata on the Gateway:")
for key, value in all_apts['KSEA'].items():
    print('\t' + key + ':', value)
```

### Downloaded the recommended scenery pack for an airport from the Gateway

```python
from xplane_airports.gateway import scenery_pack, GatewayApt
ksea_recommended_pack = scenery_pack('KSEA')
""":type ksea_recommended_pack: GatewayApt"""
print("KSEA downloaded from the Gateway begins:")
print(ksea_recommended_pack.apt.head())
```

More sample code is available in the doctests in [the `gateway` module](#the-gateway-module) docs below.

## The `AptDat` module

Tools for reading, inspecting, and manipulating X-Plane???s airport (apt.dat) files.

### AptDat.AptDat

_class_ `AptDat.AptDat`(_path\_to\_file=None_)

A container class for [`Airport`](#aptdatairport) objects. Parses X-Plane???s gigantic `apt.dat` files, which may have data on tens of thousands of airports.

**Fields**

- `airports` (List\[Airport\])
- `xplane_version` (int): The version of the apt.dat spec used by the airports in this collection

**Static method** `from_file_text`(_apt\_dat\_file\_text_, _from\_file_) -> [`AptDat`](#aptdataptdat)\
Parameters:

- **apt\_dat\_file\_text** (_str_): The contents of an apt.dat (or ICAO.dat) file
- **from\_file** (_os.PathLike_): Path to the file from which this was read

**Property** `ids`\
A generator containing the X-Plane IDs of all airports in the collection. Note that these IDs may or may not correspond to the airports??? ICAO identifiers.\
Type: collection.Iterable\[str\]

**Property** `names`\
A generator containing the names of all airports in the collection\
Type: collection.Iterable\[str\]

**Method** `search_by_id`(_apt\_id_)\
Parameter: **apt\_id** (_str_) ??? The X-Plane ID of the airport you want to query\
Returns: The airport with the specified ID, or `None` if no matching airport exists in this collection.\
Return type: Optional\[[Airport](#aptdatairport)\]

**Method** `search_by_name`(_apt\_name_)\
Parameter: **apt\_name** (_str_) ??? The name of the airport you want to query\
Returns: All airports that match the specified name, case-insensitive (an empty list if no airports match)
Return type: list\[[Airport](#aptdatairport)\]

**Method** `search_by_predicate`(_predicate\_fn_)\
Parameter: **predicate\_fn** (_(_[_Airport_](#aptdatairport)_)_ _\-> bool_) ??? We will collect all airports for which this function returns `True`\
Return type: list\[[Airport](#aptdatairport)\]

**Method** `sort`(_key='name'_)\
By default, we store the airport data in whatever order we read it from the apt.dat file. When you call sort, though, we???ll ensure that it???s in order (default to name order, just like it???s always been in the shipping versions of X-Plane).\
Parameter: **key** (_str_) ??? The [Airport](#aptdatairport) key to sort on

**Method** `write_to_disk`(_path_to_write_to_)\
Writes a complete apt.dat file containing this entire collection of airports.\
Parameter: **path_to_write_to** (_Optional_\[_os.PathLike_\]) ??? A complete file path (ending in .dat); if `None`, we'll use the path we read this apt.dat in from.

### AptDat.Airport

A single airport from an apt.dat file.

_class_ `xplane_airports.AptDat.Airport`(_name: str_, _id: str_, _from\_file: str = ''_, _has\_atc: bool = False_, _elevation\_ft\_amsl: float = 0_, _text: List\[[AptDat.AptDatLine](#aptdataptdatline)\]=\[\]_)\

Dataclass members:

- _from_file_ (_pathlib.Path_; default empty): Path to the `apt.dat` file from which this airport was read
- _raw_lines_ (List\[str\]; default empty): The complete text of the portion of the apt.dat file pertaining to this airport
- _xplane_version_ (int; default 1100): The version of the X-Plane apt.dat spec this airport uses (e.g., 1050, 1100, 1130)
- _tokenized_lines_ (List\[Tuple\]; default empty): An intermediate tokenization of the raw lines, used for speed of parsing

**Static method** `from_lines`(_apt\_dat\_lines_, _from\_file\_name_) -> [Airport](#aptdatairport)\
Parameters:\

- **from\_file\_name** (_str_) ??? The name of the apt.dat file you read this airport in from
- **apt\_dat\_lines** (_collections.Iterable\[[AptDatLine](#aptdataptdatline)|str\]_) ??? The lines of the apt.dat file (either strings or parsed AptDatLine objects)

**Static method** `from_str`(_file\_text_, _from\_file\_name_) -> [Airport](#aptdatairport)\
Parameters:

- **file\_text** (_str_) ??? The portion of the apt.dat file text that specifies this airport
- **from\_file\_name** (_str_) ??? The name of the apt.dat file you read this airport in from

**Method** `head(num_lines=10)` -> str\
Returns the first `num_lines` of the `apt.dat` text for this airport

**Property** `text` (List\[[AptDatLine](#aptdataptdatline)\])\
The raw `str` lines transformed into more usable `AptDatLine` objects. Note that this will be measurably slow if you need to work with millions of lines.

**Property** `name` (str)\
The name of the airport, like "Seattle-Tacoma Intl"

**Property** `id` (str)\
The X-Plane identifier for the airport, which may or may not correspond to its ICAO ID

**Property** `has_atc` (bool)\
True if the airport header indicates the airport has air traffic control

**Property** `elevation_ft_amsl` (float)\
The elevation, in feet above mean sea level, indicated in the airport header line

**Property** `metadata` (Dict[MetadataKey, str])\
Various key-value metadata pairs defined by X-Plane.

**Property** `has_comm_freq` (bool)\
True if this airport defines communication radio frequencies for interacting with ATC\

**Property** `has_ground_routes` (bool)\  
True if this airport defines any destinations for ground vehicles (like baggage cars, fuel trucks, etc.), ground truck parking locations, or taxi routes\

**Property** `has_taxi_route` (bool)\
True if this airport defines routing rules for ATC???s use of its taxiways.\

**Property** `has_taxiway` (bool)\
True if this airport defines any taxiway geometry\

**Property** `has_taxiway_sign` (bool)\
True if this airport defines any taxi signs\

**Property** `has_traffic_flow` (bool)\
True if this airport defines rules for when and under what conditions certain runways should be used by ATC\

**Property** `latitude` (float)\
The latitude of the airport, which X-Plane calculates as the latitude of the center of the first runway.\

**Property** `longitude` (float)\
The longitude of the airport, which X-Plane calculates as the longitude of the center of the first runway.\

**Method** `has_row_code`(_row\_code\_or\_codes_) -> bool\
True if the airport has any lines in its text that begin with the specified row code(s)\
Parameter: **row\_code\_or\_codes** (_Union__\[__int__,_ _str__,_ _collections.Iterable__\[__int__\]__\]_) ??? One or more ???row codes??? (the first token at the beginning of a line; almost always int)

**Method** `write_to_disk`(_path_to_write_to_)\
Writes a complete apt.dat file containing just this airport.\
Parameter: **path_to_write_to** (_os.PathLike_) ??? A complete file path (ending in .dat)

### AptDat.AptDatLine

A single line from an `apt.dat` file.

_class_ `xplane_airports.AptDat.AptDatLine`(_line\_text_)

**Method** `is_airport_header`() -> bool\
True if this line marks the beginning of an airport, seaport, or heliport

**Method** `is_file_header`() -> bool
True if this is part of an apt.dat file header

**Method** `is_ignorable`() -> bool\
True if this line carries no semantic value for any airport in the apt.dat file.

**Method** `is_runway`() -> bool\
True if this line represents a land runway, waterway, or helipad

**Method** `runway_type` -> [RunwayType](#aptdatrunwaytype)\
The type of runway represented by this line

**Property** `tokens` -> list\[str\]\
The tokens in this line

### AptDat.RunwayType

_class_ `xplane_airports.AptDat.RunwayType`

Enum for row codes used to identify different types of runways:

- LAND_RUNWAY
- WATER_RUNWAY
- HELIPAD

## The `gateway` module

Tools for interfacing with the X-Plane Scenery Gateway???s API.

Docs at: [https://gateway.x-plane.com/api](https://gateway.x-plane.com/api)

### gateway.GatewayApt

_class_ `xplane_airports.gateway.GatewayApt`(_apt: [AptDat.Airport](#aptdatairport), txt: Optional\[str\], readme: str, copying: str, pack\_metadata: dict, apt\_metadata: Optional\[dict\]_)

All the data we get back about an airport when we download a scenery pack via `scenery_pack()`

Dataclass members:

- _apt_ ([Airport](#aptdatairport)): Python object with the contents of the `apt.dat` file
- _txt_ (str or `None`): Contents of the DSF .txt file; airports with no 3D will not include this
- _readme_ (str): Contents of the README for this scenery pack
- _copying_ (str): Contents of the COPYING instructions for this scenery pack
- _pack_metadata_ (dict): The JSON object received from the Gateway with metadata about this particular scenery pack
- _apt_metadata_ (dict or `None`): The JSON object received from the Gateway with metadata about the airport this scenery pack represents; None if this hasn't been downloaded (yet)


### gateway.GatewayFeature

_class_ `xplane_airports.gateway.GatewayFeature`

Enum class representing the features that may be used to tag scenery packs on the Gateway. Note that these are subject to frequent addition/removal/change; only a few are guaranteed to be stable.

- HasATCFlow (guaranteed stable)
- HasTaxiRoute (guaranteed stable)
- HasNavaidConflict
- AlwaysFlatten
- HasLogTxtIssue
- LRInternalUse (guaranteed stable)
- ExcludeSubmissions (guaranteed stable)
- HasGroundRoutes (guaranteed stable)
- TerrainIncompatible
- RunwayNumberingOrLengthFix
- AlwaysFlattenIneffective
- MajorAirport
- TerrainIncompatibleAtPerimeter
- RunwayNumberingFix
- IconicAirport
- FloatingRunway
- GroundRoutesCertified
- FacadeInjection
- ScenicAirport
- MisusedGroundPolygons
- Top30
- Top50
- RunwayInWater
- RunwayUnusable
- TerrainMeshMissing
- LowResolutionTerrainPolygons

### API wrapping functions

#### `xplane_airports.gateway.airport`(_airport\_id_) -> dict

Queries the Scenery Gateway for metadata on a single airport, plus metadata on all the scenery packs uploaded for that airport.\
API endpoint documented at: [https://gateway.x-plane.com/api#get-a-single-airport](https://gateway.x-plane.com/api#get-a-single-airport)\

Returns: A dict with metadata about the airport\
Parameter: **airport\_id** (_str_) ??? The identifier of the airport on the Gateway (may or may not be an ICAO ID)

```python
>>> expected_keys = {'icao', 'airportName', 'airportClass', 'latitude', 'longitude', 'elevation', 'acceptedSceneryCount', 'approvedSceneryCount', 'recommendedSceneryId', 'scenery'}
>>> ksea = airport('KSEA')
>>> all(key in ksea for key in expected_keys)
True
```

Includes metadata of all scenery packs uploaded for this airport:

```python
>>> len(airport('KSEA')['scenery']) >= 9
True
```

```python
>>> all_scenery_metadata = airport('KSEA')['scenery']
>>> first_scenery_pack_metadata = all_scenery_metadata[0]
>>> expected_keys = {'sceneryId', 'parentId', 'userId', 'userName', 'dateUploaded', 'dateAccepted', 'dateApproved', 'dateDeclined', 'type', 'features', 'artistComments', 'moderatorComments', 'Status'}
>>> all(key in first_scenery_pack_metadata for key in expected_keys)
True
```

#### `xplane_airports.gateway.airports`() -> dict

Queries the Scenery Gateway for all the airports it knows about. Note that the download size is greater than 1 MB.\
API endpoint documented at: [https://gateway.x-plane.com/api#get-all-airports](https://gateway.x-plane.com/api#get-all-airports)\

Returns a dict with metadata on all 35,000+ airports; keys are X-Plane identifiers (which may or may not correspond to ICAO identifiers), and values are various airport metadata.

```python
>>> airports()['KSEA']
{'AirportCode': 'KSEA', 'AirportName': 'Seattle Tacoma Intl', 'AirportClass': None, 'Latitude': 47, 'Longitude': -122, 'Elevation': None, 'Deprecated': None, 'DeprecatedInFavorOf': None, 'AcceptedSceneryCount': 2, 'ApprovedSceneryCount': 2, 'ExcludeSubmissions': 0, 'RecommendedSceneryId': 45283, 'Status': 'Scenery Submitted', 'SceneryType': 0, 'SubmissionCount': 2}
```

```python
>>> len(airports()) > 35000
True
```

#### `xplane_airports.gateway.recommended_scenery_packs`(_selective\_apt\_ids=None_) -> collections.Iterable\[[GatewayApt](#gatewaygatewayapt)\] 

A generator to iterate over the recommended scenery packs for all (or just the selected) airports on the Gateway. Downloads and unzips all files into memory.

Parameter: **selective\_apt\_ids** (_Optional_\[_collections.Iterable_\[_str_\]\]) ??? If `None`, we will download scenery for all 35,000+ airports; if a list of airport IDs (as returned by `airports()`), the airports whose recommended packs we should download.\
Returns a generator of the recommended scenery packs; each pack contains the same data as a call to `scenery_pack()` directly

Easily request a subset of airports:

```python
>>> packs = recommended_scenery_packs(['KSEA', 'KLAX', 'KBOS'])
>>> len(list(packs)) == 3 and all(isinstance(pack, GatewayApt) for pack in packs)
True
```

Audit airports for specific features:

```python
>>> all_3d = True
>>> all_have_atc_flow = True
>>> all_have_taxi_route = True
>>> for pack in recommended_scenery_packs(['KATL', 'KORD', 'KDFW', 'KLAX']):
...     all_3d &= pack.pack_metadata['type'] == '3D' and pack.txt is not None
...     all_have_atc_flow &= GatewayFeature.HasATCFlow in pack.pack_metadata['features'] and pack.apt.has_traffic_flow()
...     all_have_taxi_route &= GatewayFeature.HasTaxiRoute in pack.pack_metadata['features'] and pack.apt.has_taxi_route()
>>> all_3d and all_have_atc_flow and all_have_taxi_route
True
```

#### `xplane_airports.gateway.scenery_pack`(_pack\_to\_download_) -> [GatewayApt](#gatewaygatewayapt)

Downloads a single scenery pack, including its apt.dat and any associated DSF from the Gateway, and unzips it into memory.

Parameter: **pack\_to\_download** (_str_ or _int_) ??? If `int`, the scenery ID of the pack to be downloaded; if `str`, the airport whose recommended pack we should download.\
Returns the downloaded files and the metadata about the scenery pack

```python
>>> expected_keys = {'sceneryId', 'parentId', 'icao', 'aptName', 'userId', 'userName', 'dateUploaded', 'dateAccepted', 'dateApproved', 'dateDeclined', 'type', 'features', 'artistComments', 'moderatorComments', 'additionalMetadata', 'masterZipBlob'}
>>> ksea_pack_metadata = scenery_pack('KSEA').pack_metadata
>>> all(key in ksea_pack_metadata for key in expected_keys)
True
>>> scenery_pack('KORD').pack_metadata['type'] in ('3D', '2D')
True
>>> all(isinstance(feature, GatewayFeature) for feature in scenery_pack('KMCI').pack_metadata['features'])
True
```

## Migration notes

Version 4.0 of the library introduces a handful of important breaking changes:

I've rewritten the `Airports` class with a few major changes for performance:
  
1. We no longer use `AptDatLine` objects by default. They're still available as a `@property` with the same name as their previous dataclass member (`text`), but we now use a tuple of tokenized lines as our primary data structure internally. This eliminated the single biggest bottleneck in parsing X-Plane's 35k+ airport default apt.dat, and cut the total parsing time down from > 140 seconds on my machine to under 32 seconds.
2. A number of other former dataclass members have been moved to `@property` methods. 
   
Neither change should affect basically any sane usage of the `Airport` class *except* for construction (but even that you should probably be getting from the `AptDat` class or one of the `Airport` class's static methods!).

Version 4 also utilizes `@functools.cached_property` to cache some potentially-expensive `@property` methods in the `Airport` class. Since `functools` introduced this in Python 3.8, if you're on an earlier Python version, you won't get this caching behavior (you'll get normal properties instead). This is less than ideal, but it works.

## Running the tests (for maintainers)

[We use CircleCI](https://app.circleci.com/pipelines/github/X-Plane/xplane_airports) to run the test suite on every commit.

You can run the same tests that CircleCI does as follows:

1. Doctest the Gateway module: `$ python -m doctest -v xplane_airports/gateway.py`
2. Unit test the AptDat moduel: `$ python -m unittest discover -s xplane_airports/ -t xplane_airports/ -p test_*.py`

## Publishing package updates to PyPI (for maintainers)

1. Bump the version in setup.py
2. Activate your virtual environment: `$ source env/bin/activate`
3. Package it: `$ python3 setup.py sdist bdist_wheel`
4. Upload
    - To production: `$ python3 -m twine upload --repository-url https://upload.pypi.org/legacy/ dist/*`
    - To the test server: `$ python3 -m twine upload --repository-url https://test.pypi.org/legacy/ dist/*`
5. Confirm all is well: https://pypi.org/project/xplane-airports/

You may also want to upgrade whatever project you have that depends on it: `$ pip3 install --upgrade xplane_airports`



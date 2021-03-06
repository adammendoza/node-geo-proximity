node-geo-proximity
=====================

[![Build Status](https://travis-ci.org/arjunmehta/node-geo-proximity.svg?branch=master)](https://travis-ci.org/arjunmehta/node-geo-proximity)

A node.js module that leverages the functionality of [node-geohash (ngeohash)](https://github.com/sunng87/node-geohash) and [node-redis](https://github.com/mranney/node_redis) to provide super fast proximity searches for geo coordinates.

It should be noted that the method used here is not the most precise, but the query is very fast, and should be appropriate for most consumer applications looking for this basic function. This module leverages a [process for spatial indexing as outlined by Yin Qiwen](https://github.com/yinqiwen/ardb/blob/master/doc/spatial-index.md).

Please leave feedback in the module's [GitHub issues tracker](https://github.com/arjunmehta/node-geo-proximity/issues).

## Installation

```bash
npm install geo-proximity
```


## Example Usage
This module requires a redis server in order to work and of course you need to have redis accessible to node. Visit [node-redis](https://github.com/mranney/node_redis) for more information on using redis into your node environment.

You should at the very least initialize the module with your redis client, but if you only have one set of coordinates, you can initialize the module with your client AND a zset name with which it will use for coordinate queries.

If you have more than one set of coordinates (ie. people and places) that you want to query separately, you can store and query them using optional parameters in the method calls. (see section on Multiple Sets)

```javascript
var redis = require('redis');
var client = redis.createClient();

var proximity = require('geo-proximity').initialize(client, "geo:locations");
```

### Add Coordinates
Generally you'll have some trigger to add new coordinates to your set (a user logs in, or a new point gets added in your application), or perhaps you'll want to load all the coordinates from a file of existing places. Whatever the case you should add them to redis as folows:

```javascript
proximity.addCoordinate(43.6667, -79.4167, "Toronto", function(err, reply){
  if(err) console.error(err);
  console.log("ADD successful:", reply)
});

// OR (much quicker for large sets)
var coordinates = [[43.6667, -79.4167,  "Toronto"],
                   [39.9523, -75.1638,  "Philadelphia"],
                   [37.4688, -122.1411, "Palo Alto"],
                   [37.7691, -122.4449, "San Francisco"],
                   [47.5500, -52.6667,  "St. John's"],
                   [40.7143, -74.0060,  "New York"],
                   [49.6500, -54.7500,  "Twillingate"],
                   [45.4167, -75.7000,  "Ottawa"],
                   [51.0833, -114.0833, "Calgary"],
                   [18.9750, 72.8258,   "Mumbai"]];

proximity.addCoordinates(coordinates, function(err, reply){
  if(err) console.error(err);
  console.log("ADD successful:", reply)
});
```

### Query for proximal points
Now you can look for points that exist within a certain range of any other coordinate in the system.

```javascript
// look for all points within 5000m of Toronto.
proximity.query(43.646838, -79.403723, 5000, function(err, replies){
  if(err) console.error(err);
  console.log(replies);
});
```

### Remove points
Of course you may need to remove some points from your set as users/temporary events/whatever no longer are part of the set.

```javascript
proximity.removeCoordinate("New York", function(err, reply){
  if(err) console.error(err);
  console.log("Removed Coordinate", reply);
});

// OR Quicker for Bulk Removals
proximity.removeCoordinates(["New York", "St. John's", "San Francisco"], function(err, reply){
  if(err) console.error(err);
  console.log("Removed Coordinates", reply);
});
```

### Multiple zSets
If you have different sets of coordinates, you can store and query them separately by passing options with a `zset` property that specifies the Redis ordered set to store/query them in. Removal and Performant Querying work the same way. Review the API to see where you can specify options.

```javascript
var people       = [[43.6667,-79.4167,  "John"],
                   [39.9523, -75.1638,  "Shankar"],
                   [37.4688, -122.1411, "Cynthia"],
                   [37.7691, -122.4449, "Chen"]];

var places       = [[43.6667,-79.4167,  "Toronto"],
                   [39.9523, -75.1638,  "Philadelphia"],
                   [37.4688, -122.1411, "Palo Alto"],
                   [37.7691, -122.4449, "San Francisco"],
                   [47.5500, -52.6667,  "St. John's"]];

proximity.addCoordinates(people, {zset: "geo:locations:people"}, function(err, reply){
  if(err) console.error(err);
  console.log("ADD successful:", reply)
});

proximity.addCoordinates(places, {zset: "geo:locations:places"}, function(err, reply){
  if(err) console.error(err);
  console.log("ADD successful:", reply)
});
```

```javascript
// will find all PEOPLE ~5000m from the passed in coordinate
proximity.query(43.646838, -79.403723, 5000, {zset: "geo:locations:people"}, function(err, people){
  if(err) console.error(err);
  console.log(people);
});

// will find all PLACES ~5000m from the passed in coordinate
proximity.query(43.646838, -79.403723, 5000, {zset: "geo:locations:places"}, function(err, places){
  if(err) console.error(err);
  console.log(places);
});
```


# API

## Initialization

### proximity.initialize(redisClient, redisZSetName);
Initialize the module with a redis client, and a ZSET name. This is not required, but will slightly improve efficiency and make your life easier. If you don't initialize, you will need to pass in a client and zset name as options for method calls.

## Adding/Removing Coordinates

### proximity.addCoordinate(lat, lon, coordinateName, {options}, callBack);
Add a new coordinate to your set. You can get quite technical here by specifying the geohash integer resolution at which to store (MUST BE CONSISTENT).

#### Options
- `bitDepth: {Number, default is 52}`: the bit depth you want to store your geohashes in, usually the highest possible (52 bits for javascript). MUST BE CONSISTENT. If you set this to another value other than 52, you will have to ensure you set bitDepth in options for querying methods.
- `client: {redisClient}`
- `zset: {String}`

### proximity.addCoordinates(coordinateArray, {options}, callBack);
Adds an array of new coordinates to your set. The `coordinateArray` must be in the form `[[lat, lon, name],[lat, lon, name],...,[lat, lon, name]]`. Again you can specify the geohash integer resolution at which to store (MUST BE CONSISTENT). Use this method for bulk additions, as it is much faster than individual adds.

#### Options
- `bitDepth: {Number, default is 52}`: the bit depth you want to store your geohashes in, usually the highest possible (52 bits for javascript). MUST BE CONSISTENT. If you set this to another value other than 52, you will have to ensure you set bitDepth in options for querying methods.
- `client: {redisClient}`
- `zset: {String}`

### proximity.removeCoordinate(coordinateName, {options}, callBack);
Remove the specified coordinate by name.

#### Options
- `client: {redisClient}`
- `zset: {String}`

### proximity.removeCoordinates(coordinateNameArray, {options}, callBack);
Remove a set of coordinates by name. `coordinateNameArray` must be of the form `[nameA,nameB,nameC,...,nameN]`.

#### Options
- `client: {redisClient}`
- `zset: {String}`


## Basic Querying

### proximity.query(lat, lon, radius, {options}, callBack);
Use this function for a basic search by proximity within the given latitude and longitude and radius (in meters). It is not ideal to use this method if you intend on making the same query multiple times. **If performance is important and you'll be making the same query over and over again, it is recommended you instead have a look at proximity.queryByRanges and promixity.getQueryRangesFromRadius.** Otherwise this is an easy method to use.

**Options:**
- `radiusBitDepth: {Number, default is 48}`: This is the bit depth that is associated with a search radius. It will override your radius setting, so if you use this option, for good form pass in `null` as your radius.
- `bitDepth: {Number, default is 52}`: the bit depth your geohashes are stored in if they are not in the default 52 bits.
- `client: {redisClient}`
- `zset: {String}`
- `values: {Boolean, default is false}`: Instead of returning a flat array of key names, it will instead return a full set of keynames with coordinates in the form of `[[name, lat, lon], [name, lat, lon]...]`.This will be a slower query compared to just returning the keynames because the coordinates need to be calculated from the stored geohashes.


## Performant Querying

If you intend on performing the same query over and over again with the same initial coordinate and the same distance, you should cache the **geohash ranges** that are used to search for nearby locations. The geohash ranges are what the methods ultimately search within to find nearby points. So keeping these stored in a variable some place and passing them into a more basic search function will save some cycles (at least 5ms on a basic machine). This will save you quite a bit of processing time if you expect to refresh your searches often, and especially if you expect to have empty results often. Your processor is probably best used for other things.

### proximity.getQueryCache(lat, lon, radius, {bitDepth=52});
Get the query ranges to use with **proximity.queryByRanges**. This returns an array of geohash ranges to search your set for. `bitDepth` is optional and defaults to 52, set it if you have chosen to store your coordinates at a different bit depth. Store the return value of this function for making the same query often.

### proximity.queryWithCache(ranges, {options}, callBack);
Pass in query ranges returned by **proximity.getQueryRangesFromRadius** to find points that fall within your range value.

**Options:**
- `client: {redisClient}`
- `zset: {String}`
- `values: {Boolean, default is false}`: Instead of returning a flat array of key names, it will instead return a full set of keynames with coordinates in the form of `[[name, lat, lon], [name, lat, lon]...]`.This will be a slower query compared to just returning the keynames because the coordinates need to be calculated from the stored geohashes.
- `bitDepth: {Number, default is 52}`: the bit depth your geohashes are stored in if they are not in the default 52 bits. Only needed if the `values` option is set.

## Example of Performant Method Usage
As mentioned, you may want to cache the ranges to search for in your data model. Perhaps if you have a connection or user that is logged in, you can associate these ranges with their object.

```javascript
// will hold the generated query ranges used for the query. This will
// save you some processing time for future queries using the same values
var cachedQuery = proximity.getQueryCache(37.4688, -122.1411, 5000);

proximity.queryWithCache(cachedQuery, function(err, replies){
  console.log("results to the query:", replies);
});
```

Get it? Got it?


## License

The MIT License (MIT)

Copyright (c) 2014 Arjun Mehta

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

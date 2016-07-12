# express-throttle
Request throttling middleware for Express framework

[![npm version](https://badge.fury.io/js/express-throttle.svg)](https://badge.fury.io/js/express-throttle)
[![Build Status](https://travis-ci.org/GlurG/express-throttle.svg?branch=master)](https://travis-ci.org/GlurG/express-throttle)

## Installation

```bash
$ npm install express-throttle
```

## Implementation

The throttling is done using the canonical [token bucket](https://en.wikipedia.org/wiki/Token_bucket) algorithm, where tokens are "refilled" in a sliding window manner by default (can be configured to fixed time windows). This means that if we a set maximum rate of 10 requests / minute, a client will not be able to send 10 requests 0:59, and 10 more 1:01. However, if the client sends 10 requests at 0:30, he will be able to send a new request at 0:36 (since tokens are refilled continuously 1 every 6 seconds).

## Limitations

By default, throttling data is stored in memory and is thus not shared between multiple processes. If your application is behind a load balancer which distributes traffic among several node processes, then throttling will be applied per process, which is generally not what you want (unless you can ensure that a client always hits the same process). It is possible to customize the storage so that the throttling data gets saved to a shared backend (e.g Redis). However, the current implementation contains a race-condition and will likely fail (erroneously allow/block certain requests) under high load. My plan is to address this shortcoming in future versions.

**TL;DR** - Use this package in production at your own risk, beware of the limitations.
* If you are running node as single process = you should be fine
* If you are running node as multiple processes = be aware that throttling is done per process
* If you have configured to use an external backend = some requests may erroneously be throttled (or passed through) under high load conditions

## Examples

```js
var express = require("express");
var throttle = require("express-throttle");
  
var app = express();
  
// Throttle to 5 reqs/s
app.post("/search", throttle("5/s"), function(req, res, next) {
  // ...
});

// ...using fixed time windows instead
app.post("/search", throttle("5/s:fixed"), function(req, res, next) {
  // ...
})
```
Combine it with a burst capacity of 10, meaning that the client can make 10 requests at any rate. The capacity is "refilled" with the specified rate (in this case 5/s).
```js
app.post("/search", throttle({ "rate": "5/s", "burst": 10 }), function(req, res, next) {
  // ...
});
  
// "Half" requests are supported as well (1 request every other second)
app.post("/search", throttle({ "rate": "1/2s", "burst": 5 }), function(req, res, next) {
  // ...
});
```
By default, throttling is done on a per ip-address basis (see [this link](http://expressjs.com/en/api.html#req.ip) about how the ip address is extracted from the request). This can be configured by providing a custom key-function:
```js
var options = {
  "rate": "5/s",
  "burst": 10,
  "key": function(req) {
    return req.session.username;
  }
};

app.post("/search", throttle(options), function(req, res, next) {
  // ...
});
```
The "cost" per request can also be customized, making it possible to, for example whitelist certain requests:
```js
var whitelist = ["ip-1", "ip-2", ...];
  
var options = {
  "rate": "5/s",
  "burst": 10,
  "cost": function(req) {
    var ip_address = req.connection.remoteAddress;
      
    if (whitelist.indexOf(ip_address) >= 0) {
      return 0;
    } else if (req.session.is_privileged_user) {
      return 0.5;
    } else {
      return 1;
    }
  }
};
  
app.post("/search", throttle(options), function(req, res, next) {
  // ...
});

var options = {
  "rate": "1/s",
  "burst": 10,
  "cost": 2.5 // fixed costs are also supported
};

app.post("/expensive", throttle(options), function(req, res, next) {
  // ...
});
```
Throttled requests will simply be responded with an empty 429 response. This can be overridden:
```js
var options = {
  "rate": "5/s",
  "on_throttled": function(req, res, next, bucket) {
    // Possible course of actions:
    // 1) Log request
    // 2) Add client ip address to a ban list
    // 3) Send back more information
    res.set("X-Rate-Limit-Limit", 5);
    res.set("X-Rate-Limit-Remaining", 0);
    res.status(503).send("System overloaded, try again at a later time.");
  }
};
```
You may also customize the response on requests that are passed through:
```js
var options = {
  "rate": "5/s",
  "on_allowed": function(req, res, next, bucket) {
    res.set("X-Rate-Limit-Limit", 5);
    res.set("X-Rate-Limit-Remaining", bucket.tokens);
    res.set("X-Rate-Limit-Reset", bucket.rtime);
  }
}
```
Throttling can be applied across multiple processes. This requires an external storage mechanism which can be configured as follows:
```js
function ExternalStorage(connection_settings) {
  // ...
}
  
// These methods must be implemented
ExternalStorage.prototype.get = function(key, callback) {
  fetch(key, function(bucket) {
    // First argument should be null if no errors occurred
    callback(null, bucket);
  });
}
  
ExternalStorage.prototype.set = function(key, bucket, callback) {
  save(key, bucket, function(err) {
    // err should be null if no errors occurred
    callback(err); 
  });
}
  
var options = {
  "rate": "5/s",
  "store": new ExternalStorage()
}
  
app.post("/search", throttle(options), function(req, res, next) {
  // ...
});
```

## Options

`rate`: Determines the number of requests allowed within the specified time unit before subsequent requests get throttled. Must be specified according to the following format: *X/Yt(:fixed)*

where *X* and *Y* are integers and *t* is the time unit which can be any of the following: `ms, s, sec, second, m, min, minute, h, hour, d, day`

If you prefer tokens to be refilled in fixed intervals, append `:fixed`. E.g `5/min:fixed`.

`burst`: The number of requests that can be made at any rate. Defaults to *X* as defined above.

`store`: Custom storage class. Must implement a `get` and `set` method with the following signatures:
```js
// callback in both methods must be called with an error (if any) as first argument

function get(key, callback) {
  fetch(key, function(err, bucket) {
    if (err) callback(err);
    else callback(null, bucket);
  });
}
function set(key, bucket, callback) {
  // 'bucket' will be an object with the following structure:
  /*
    {
      "tokens": Number, (current number of tokens)
      "mtime": Number, (last modification time)
      "rtime": Number (time until next reset)
    }
  */
  save(key, bucket, function(err) {
    callback(err);
  }
}
```
Defaults to an LRU cache with a maximum of 10000 entries.

`key`: Function used to identify clients. It will be called with an [express request object](http://expressjs.com/en/4x/api.html#req). Defaults to:
```js
function(req) {
	return req.ip; // http://expressjs.com/en/api.html#req.ip
}
```

`cost`: Number or function used to calculate the cost for a request with an [express request object](http://expressjs.com/en/4x/api.html#req). Defaults to 1.

`on_allowed`: A function called when the request is passed through with an [express request object](http://expressjs.com/en/4x/api.html#req), [express response object](http://expressjs.com/en/4x/api.html#res), `next` function and a `bucket` object. Defaults to:
```js
function(req, res, next, bucket) {
	next();
}
``` 

`on_throttled`: A function called when the request is throttled with an [express request object](http://expressjs.com/en/4x/api.html#req), [express response object](http://expressjs.com/en/4x/api.html#res), `next` function and a `bucket` object. Defaults to:
```js
function(req, res, next, bucket) {
	res.status(429).end();
};
```

## Benchmark

Refer to [this](https://github.com/GlurG/express-throttle/blob/master/Benchmark.md) document for ballparks / guidelines of the performance penalty this middleware will incur.
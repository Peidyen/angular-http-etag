# Angular HTTP ETag Module

[![npm version](https://badge.fury.io/js/angular-http-etag.svg)](http://badge.fury.io/js/angular-http-etag)
[![Build Status](https://travis-ci.org/shaungrady/angular-http-etag.svg?branch=master)](https://travis-ci.org/shaungrady/angular-http-etag)
[![Test Coverage](https://codeclimate.com/github/shaungrady/angular-http-etag/badges/coverage.svg)](https://codeclimate.com/github/shaungrady/angular-http-etag/coverage)
[![Code Climate](https://codeclimate.com/github/shaungrady/angular-http-etag/badges/gpa.svg)](https://codeclimate.com/github/shaungrady/angular-http-etag)  
[![Dependency Status](https://david-dm.org/shaungrady/angular-http-etag.svg)](https://david-dm.org/shaungrady/angular-http-etag)
[![devDependency Status](https://david-dm.org/shaungrady/angular-http-etag/dev-status.svg)](https://david-dm.org/shaungrady/angular-http-etag#info=devDependencies)

---

Easy ETag-based caching for `$http` service requests!

* Caches ETag headers and sends them back to the server in the `If-None-Match` header.
* Caches response data with flexible cache configuration.
* Support for `$cacheFactory`, `sessionStorage`, and `localStorage` caches out-of-the-box.
* Easily [adaptable][Cache Service Adapters] for other third-party cache services.

**Example Usage:**

``` javascript
angular
  .module('myApp', [
    'angular-http-etag'
  ])
  .config(function (httpEtagProvider) {
    httpEtagProvider
      .defineCache('persistentCache', {
        cacheService: 'localStorage'
      })
  })

  .controller('MyCtrl', function ($http) {
    var self = this

    $http
      .get('/my_data.json', {
        etagCache: 'persistentCache'
      })
      .success(function (data, status, headers, config, itemCache) {
        // Modify the data from the server
        data._fullName = data.first_name + ' ' data.last_name
        // Update the cache with the modified data
        itemCache.set(data)
        // Assign to controller property
        self.fullName = data._fullName
      })
      // Synchronous method called if request was previously cached
      .cached(function (data, status, headers, config, itemCache) {
        self.fullName = data._fullName
      })
      .error(function (data, status) {
        // 304: 'Not Modified'--Etag matched, cached data is fresh
        if (status != 304) alert('Request error')
      })
  })
```

_Need more information on how ETags work? [Stack Overflow](http://stackoverflow.com/questions/499966/etag-vs-header-expires) has your back._

## Detailed Documentation

- [Service Provider]
- [Service]
- [$http Decorator]
- [Cache Service Adapters]

[Service Provider]: /shaungrady/angular-http-etag/blob/master/docs/service_provider.md
[Service]: /shaungrady/angular-http-etag/blob/master/docs/service.md
[$http Decorator]: /shaungrady/angular-http-etag/blob/master/docs/http_decorator.md
[Cache Service Adapters]: /shaungrady/angular-http-etag/blob/master/docs/cache_service_adapters.md

## Quick Guide

### Install Module

``` bash
$ npm install angular-http-etag
```

### Include Dependency

Include `'http-etag'` in your module's dependencies.

``` javascript
// The node module exports the string 'http-etag'...
angular.module('myApp', [
  require('http-etag')
])
```

``` javascript
// ... otherwise, include the code first then the module name
angular.module('myApp', [
  'http-etag'
])
```

### Define Caches

``` javascript
.config(function ('httpEtagProvider') {
  httpEtagProvider
    .defineCache('persistentCache', {
      cacheService: 'localStorage'
    })
    .defineCache('sessionCache', {
      cacheService: 'sessionStorage'
    })
    .defineCache('memoryCache', {
      cacheService: '$cacheFactory',
      cacheOptions: {
        number: 50 // LRU cache
      },
      deepCopy: true // $cacheFactory copies by reference by default
    })
})
```

Define the caches you'd like to use by using `defineCache(cacheId[, config])`, passing a cache ID
followed by the cache configuration. The configuration you pass will extend the
default configuration, which can be set using the `setDefaultCacheConfig(config)` method. If you don't pass a config, a new cache will be defined using the default config.

When making `$http` requests, if you don't specify a `cacheId` in the request configuration, a default cache ID of `httpEtagCache` will be used.

 _See more in the [Service Provider] documentation._

### Cache Requests

Using the default cache with default configuration and an automatically generated cache itemKey based on the request:

``` javascript
$http.get('/data', {
    etagCache: true
  })
  .cached(requestHandler)
  .success(requestHandler)

function requestHandler(data, status, headers, config, itemCache) {
  // Differentiating between cached and successful request responses
  var isCached = status === 'cached'

  // itemCache is a cache object bound to the cache item associated with this request.
  itemCache.info()
  // { deepCopy: false,
  //   cacheResponseData: true,
  //   cacheService: '$cacheFactory',
  //   cacheOptions: { number: 25 },
  //   id: 'httpEtagCache',
  //   itemKey: '/data' }
}
```
Using a defined cache from the previous section and an automatically generated cache itemKey:

``` javascript
$http.get('/data', {
    etagCache: 'persistentCache'
  })
  .cached(requestHandler)
  .success(requestHandler)

function requestHandler(data, status, headers, config, itemCache) {
  itemCache.info()
  // { deepCopy: false,
  //   cacheResponseData: true,
  //   cacheService: 'localStorage',
  //   cacheOptions: { number: 25 },
  //   id: 'httpEtagCache',
  //   itemKey: '/data' }
}
```

#### etagCache Property

The `etagCache` `$http` config property accepts the following value types:

| Type | Details |
| :-- | :-- |
| `boolean` | If `true`, use default `cacheId` and `itemKey`. |
| `string` | String representing the `cacheId` to be used. |
| `object.<string>` | Object with `cacheId` property and optional `itemKey` property. See below. |
| `function` | A function that returns one of the above value types. The current `$http` config is passed an argument. |

Config object with optional `itemKey` property. If not specified, one will be generated based on the request URL/params.

``` javascript
{
  cacheId: 'sessionCache',
  itemKey: 'specifiedKey' // Optional property
}
```

#### itemCache Object

The `itemCache` object passed to `cached` and `success` callbacks has these methods:

| Method | Details |
| :-- | :-- |
| `info()` | Return info about this cacheItem |
| `set(value[, options])` | Update the cache with the passed value and optional options (if the cache service supports options) |
| `get([options])` | Get the cache contents with optional options (if the cache service supports options) |
| `unset()` | Remove cached data while preserving the cached ETag. |
| `expire()` | Remove cached ETag while preserving the cached data. |
| `remove()` | Clear the cached response data and ETag. |

 _See more in the [$http Decorator] and [Service] documentation._

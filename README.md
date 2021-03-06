rets-client
===========
A RETS (Real Estate Transaction Standard) client for Node.js.


## Changes

#### 4.0.0

Version 4.0!  This represents a substantial rewrite of the object-retrieval code, making it more robust and flexible,
adding support for streaming object results, and allowing for additional query options (like `Location` and
`ObjectData`).  See the simple photo query example at the end of the [Example RETS Session](#example-rets-session), and
the [photo streaming example](#photo-streaming-example).

Also new in version 4.0, just about every API call now gives access to header info from the RETS response.  For some
queries, in particular object queries, headers are often used as a vehicle for important metadata about the response,
so this is an important feature.  The exact mechanism varies depending on whether it a call that resolves to a stream
(which now emits a `headerInfo` event) or a call that resolves to an object (which now has a `headerInfo` field).  Also,
the RetsServerError and RetsReplyError objects now both contain a `headerInfo` field.

#### 3.3.0

Version 3.3 adds support for debugging via the [debug](https://github.com/visionmedia/debug) and
[request-debug](https://github.com/request/request-debug) modules. See the [debugging section](#debugging).

#### 3.2.2

Version 3.2.2 adds support for per-object errors when calling `client.objects.getPhotos()`.  The
[Example RETS Session](#example-rets-session) illustrates proper error checking.

#### 3.2.0

Version 3.2 passes through any multipart headers (except for content-disposition, which gets split up first;
content-type which is renamed to `mime`; and content-transfer-encoding which is used internally and not passed) onto
the objects resolved from `client.objects.getPhotos()`. It also fixes a race condition in `client.objects.getPhotos()`.

#### 3.1.0

Version 3.1 adds a `response` field to the object resolved from `client.objects.getObject()`, containing the full HTTP
response object.  It also fixes a major bug interfering with `client.objects.getPhotos()` and
`client.objects.getObject()` calls.

#### 3.0.0

Version 3.x is out!  This represents a substantial rewrite of the underlying code, which should improve performance
(both CPU and memory use) for almost all RETS calls by using node-expat instead of xml2js for xml parsing.  The changes
are mostly internal, however there is 1 small backward-incompatible change needed for correctness, described below.
The large internal refactor plus even a small breaking change warrants a major version bump.

Version 3.x has almost the same interface as 2.x, which is completely different from 1.x.  If you wish to continue to
use the 1.x version, you can use the [v1 branch](https://github.com/sbruno81/rets-client/tree/v1).

Many of the metadata methods are capable of returning multiple sets of data, including (but not limited to) the
getAll* methods.  Versions 1.x and 2.x did not handle this properly; ~~version 1.x returns the values from the last set
encountered~~, and version 2.x returns the values from the first set encountered.  (This has been corrected in version
1.2.0.)  Version 3.x always returns all values encountered, by returning an array of data sets rather than a single one.  

In addition to the methods available in 2.x, version 3.0 adds `client.search.stream.searchRets()`, which returns a
text stream of the raw XML result, and `client.search.stream.query()`, which returns a stream of low-level objects
parsed from the XML.  (See the [streaming example](#simple-streaming-example) below.)  These streams, if used properly,
should result in a much lower memory footprint than their corresponding non-streaming counterparts.


## Implementation Notes

This interface uses promises, and an optional stream-based interface for better performance with large search results.
Future development will include an optional stream-based interface for object downloads, and an improved API for the
non-streaming object methods.

This library is written primarily in CoffeeScript, but may be used just as easily in a Node app using Javascript or
CoffeeScript.  Promises in this module are provided by [Bluebird](https://github.com/petkaantonov/bluebird).

The original module was developed against a server running RETS v1.7.2, so there may be incompatibilities with other
versions.  However, we want this library to work against any RETS servers that are in current use, so issue tickets
describing problems or (even better) pull requests that fix interactions with servers running other versions of RETS
are welcomed.

For more information about what all the parameters and return values and such mean, you might want to look at the
[RETS Specifications](http://www.reso.org/specifications)


## Contributions
Issue tickets and pull requests are welcome.  Pull requests must be backward-compatible to be considered, and ideally
should match existing code style.

#### TODO
- create unit tests -- specifically ones that run off example RETS data rather than requiring access to a real RETS server


## Example Usage

##### Client Configuration
```javascript
    //create rets-client
    var clientSettings = {
        loginUrl:retsLoginUrl,
        username:retsUser,
        password:retsPassword,
        version:'RETS/1.7.2',
        userAgent:'RETS node-client/3.0'
    };
...
```    
##### Client Configuration with UA Authorization
```javascript
    //create rets-client
    var clientSettings = {
        version:'RETS/1.7.2',
        userAgent:userAgent,
        userAgentPassword:userAgentPassword,
        sessionId:sessionId
    };
...
```

#### Example RETS Session
```javascript
  var rets = require('rets-client');
  var fs = require('fs');
  var photoSourceId = '12345'; // <--- dummy example ID!  this will usually be a MLS number / listing id
  var outputFields = function(obj, opts) {
    if (!opts) opts = {};
    
    var excludeFields;
    var loopFields;
    if (opts.exclude) {
      excludeFields = opts.exclude;
      loopFields = Object.keys(obj);
    } else if (opts.fields) {
      loopFields = opts.fields;
      excludeFields = [];
    } else {
      loopFields = Object.keys(obj);
      excludeFields = [];
    }
    for (var i=0; i<loopFields.length; i++) {
      if (excludeFields.indexOf(loopFields[i]) == -1) {
      if (typeof(obj[field]) == 'object') {
        console.log("    "+loopFields[i]+": "+JSON.stringify(obj[loopFields[i]]));
      } else {
        console.log("    "+loopFields[i]+": "+obj[loopFields[i]]);
      }
    }
    console.log("");
  };
  
  // establish connection to RETS server which auto-logs out when we're done
  rets.getAutoLogoutClient(clientSettings, function (client) {
    console.log("===================================");
    console.log("========  System Metadata  ========");
    console.log("===================================");
    outputFields(client.systemData);
    
    //get resources metadata
    return client.metadata.getResources()
      .then(function (data) {
        console.log("======================================");
        console.log("========  Resources Metadata  ========");
        console.log("======================================");
        outputFields(data.results[0].info);
        for (var dataItem = 0; dataItem < data.results[0].metadata.length; dataItem++) {
          console.log("-------- Resource " + dataItem + " --------");
          outputFields(data.results[0].metadata[dataItem], {fields: ['ResourceID', 'StandardName', 'VisibleName', 'ObjectVersion']});
        }
      }).then(function () {
      
        //get class metadata
        return client.metadata.getClass("Property");
      }).then(function (data) {
        console.log("===========================================================");
        console.log("========  Class Metadata (from Property Resource)  ========");
        console.log("===========================================================");
        outputFields(data.results[0].info);
        for (var classItem = 0; classItem < data.results[0].metadata.length; classItem++) {
          console.log("-------- Table " + classItem + " --------");
          outputFields(data.results[0].metadata[classItem], {fields: ['ClassName', 'StandardName', 'VisibleName', 'TableVersion']});
        }
      }).then(function () {
      
        //get field data for open houses
        return client.metadata.getTable("OpenHouse", "OPENHOUSE");
      }).then(function (data) {
        console.log("=============================================");
        console.log("========  OpenHouse Table Metadata  ========");
        console.log("=============================================");
        outputFields(data.results[0].info);
        for (var tableItem = 0; tableItem < data.results[0].metadata.length; tableItem++) {
          console.log("-------- Field " + tableItem + " --------");
          outputFields(data.results[0].metadata[tableItem], {fields: ['MetadataEntryID', 'SystemName', 'ShortName', 'LongName', 'DataType']});
        }
        return data.results[0].metadata
      }).then(function (fieldsData) {
        var plucked = [];
        for (var fieldItem = 0; fieldItem < fieldsData.length; fieldItem++) {
          plucked.push(fieldsData[fieldItem].SystemName);
        }
        return plucked;
      }).then(function (fields) {
      
        //perform a query using DQML2 -- pass resource, class, and query, and options
        return client.search.query("OpenHouse", "OPENHOUSE", "(OpenHouseType=PUBLIC),(ActiveYN=1)", {limit:100, offset:10})
        .then(function (searchData) {
          console.log("===========================================");
          console.log("========  OpenHouse Query Results  ========");
          console.log("===========================================");
          outputFields(searchData, {exclude: ['results']});
          //iterate through search results
          for (var dataItem = 0; dataItem < searchData.results.length; dataItem++) {
            console.log("-------- Result " + dataItem + " --------");
            outputFields(searchData.results[dataItem], {fields: fields});
          }
          if (searchData.maxRowsExceeded) {
            console.log("-------- More rows available!");
          }
        });
      }).then(function () {
      
        // get photos
        return client.objects.getAllObjects("Property", "LargePhoto", photoSourceId, {alwaysGroupObjects: true, ObjectData: '*'})
      }).then(function (photoResults) {
        console.log("=================================");
        console.log("========  Photo Results  ========");
        console.log("=================================");
        for (var i = 0; i < photoResults.objects.length; i++) {
          if (photoResults.objects[i].error) {
            console.log("Photo " + (i + 1) + " had an error: " + photoResults.objects[i].error);
          } else {
            console.log("Photo " + (i + 1) + ":");
            outputFields(photoResults.objects[i].headerInfo);
            fs.writeFileSync(
              "/tmp/photo" + (i + 1) + "." + photoResults.objects[i].headerInfo.contentType.match(/\w+\/(\w+)/i)[1],
              photoResults.objects[i].data
            );
          }
          console.log("---------------------------------");
        }
      });
      
  }).catch(function (errorInfo) {
    var error = errorInfo.error || errorInfo;
    console.log("ERROR: issue encountered: "+(error.stack||error));
  });
```

#### Simple streaming example
```javascript
  var rets = require('rets-client');
  var through2 = require('through2');
  var Promise = require('bluebird');
  // establish connection to RETS server which auto-logs out when we're done
  rets.getAutoLogoutClient(clientSettings, function (client) {
    // in order to have the auto-logout function work properly, we need to make a promise that either rejects or
    // resolves only once we're done processing the stream
    return new Promise(function (reject, resolve) {
      var retsStream = client.search.stream.query("OpenHouse", "OPENHOUSE", "(OpenHouseType=PUBLIC),(ActiveYN=1)", {limit:100, offset:10});
      var processorStream = through2.obj(function (event, encoding, callback) {
        switch (event.type) {
          case 'data':
            // event.payload is an object representing a single row of results
            // make sure callback is called only when all processing is complete
            doAsyncProcessing(event.payload, callback);
            break;
          case 'done':
            // event.payload is an object containing a count of rows actually received, plus some other things
            // now we can resolve the auto-logout promise
            resolve(event.payload.rowsReceived);
            callback();
            break;
          case 'error':
            // event.payload is an Error object
            console.log('Error streaming RETS results: '+event.payload);
            retsStream.unpipe(processorStream);
            processorStream.end();
            // we need to reject the auto-logout promise
            reject(event.payload);
            callback();
            break;
          default:
            // ignore other events
            callback();
        }
      });
      retsStream.pipe(processorStream);
    });
  });
```

#### Photo streaming example
```javascript
  var rets = require('rets-client');
  // establish connection to RETS server which auto-logs out when we're done
  rets.getAutoLogoutClient(clientSettings, function (client) {
    // getObjects will accept a single string, an array of strings, or an object as shown below
    var photoIds = {
      '11111': [1,3],  // get photos #1 and #3 for listingId 11111
      '22222': '*',    // get all photos for listingId 22222
      '33333': '0'     // get 'preferred' photo for listingId 33333
    };
    return client.objects.stream.getObjects('Property', 'Photo', photoIds, {alwaysGroupObjects: true, ObjectData: '*'})
    .then(function (photoStream) {
      return new Promise(function (resolve, reject) {
        var i=0;
        photoStream.on('data', function (photoEvent) {
          i++;
          if (photoEvent.error) {
            console.log("Photo " + i + " had an error: " + photoEvent.error);
          } else {
            console.log("Photo " + (i + 1) + ":");
            outputFields(photoResults.objects[i].headerInfo);
            fileStream = fs.createWriteStream(
              "/tmp/photo" + i + "." + photoEvent.headerInfo.contentType.match(/\w+\/(\w+)/i)[1]
            );
            photoEvent.dataStream.pipe(fileStream);
          }
        });
        photoStream.on('end', function () {
          resolve();
        });
      });
    });
  });
```

##### Debugging
You can turn on all debug logging by adding `rets-client:*` to your `DEBUG` environment variable, as per the
[debug module](https://github.com/visionmedia/debug).  Sub-loggers available:
* `rets-client:main`: basic logging of RETS call options and errors
* `rets-client:request`: logging of HTTP request/response headers and other related info, with output almost identical
to that provided by the [request-debug module](https://github.com/request/request-debug).

If you want access to the request debugging data directly, you can use the `requestDebugFunction` client setting.  This
function will be set up as a debug handler as per the [request-debug module](https://github.com/request/request-debug).

In order to get either `rets-client:request` logging, or to use `requestDebugFunction`, you will need to ensure
dev dependencies (in particular, request-debug) are installed for rets-client.  The easiest way to do this is to first
change directory to the location of rets-client (e.g. `cd ./node_modules/rets-client`), and then run `npm install`.
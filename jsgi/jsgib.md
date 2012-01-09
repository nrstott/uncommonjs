# JSGI 0.3 B

## Philosophy

* Straightforward mapping to HTTP with cosmetic modifications to match JavaScript conventions and idioms.
* Suitable for extension.

### Principles

* Discourage duplicate data in request and response objects.
* Optimize for the Application and Middleware Developer, not the Server developer.
* Adhere to HTTP semantics where possible.

## Specification

### Middleware

JSGI Middleware are JavaScript Functions that takes a variadic number of arguments. The first argument is a 
JavaScript Object representing the Request. If the Middleware is capable of calling other Middleware then 
it should be passed a JavaScript Function as its last arguemnt that is the next Middleware to call.

### Application

A JSGI Application is one or more JSGI Middleware that compose a Response as a JavaScript Object
containing three required attributes: status, headers, and body.

### Server

A JSGI Server is the glue that connects JSGI Applications to HTTP.

## Request

The request environment must be a JavaScript Object representing a HTTP request. Applications and Middleware are free to modify the request.

### Required Keys

The request is required to include the following keys:

* _method_: The HTTP method e.g. "GET", "POST", "PUT", "DELETE", etc. The value must be an uppercase string.
* _auth_: The authentication information portion of the URL.
* _pathname_: The remainder of the request URL's path. This may be an empty string if the request URL targets the Application root.
* _search_:  The query string portion of the URL, including the '?'.
* _protocol_: The URL scheme per RFC 1738 e.g. "http", "https", etc.
* _port_: The port the HTTP request was made to, if not present in the URL may be dervied from the protocol. Must a number.
* _hostname_: The host name without hte port number.
* _input_: The request body. Must be a ForEachable.
* _env_: An Object defined in the `prototype` of Request making it a data store shared by all requests. Servers and Middleware may add arbitrary keys to `env`.
  * version: The string "0.3b"

### Optional Keys

* _authType_: The authorization type of the request e.g. Basic.
* _remoteAddr_: The IP Address from which the request originates.
* _remoteHost_: The host name from which the request originates. The reverse DNS lookup is based off of the remoteAddr of the request.
* _remoteIdent_: Name of the remote user dervied from the ident information of the request.
* _remoteUser_: Name of the remote user.
* _serverSoftware_: The name and version of the server.

## Response

A JSGI Application must return a JavaScript Object representative of a HTTP Response.

### Status

The status must be a three-digit integer. [RFC 2616 Section 6](http://www.w3.org/Protocols/rfc2616/rfc2616-sec6.html#sec6.1.1)

### Headers

Must be a JavaScript Object containing key/value pairs of Strings or Arrays. Servers should output
multiple headers for keys that have values specified as Arrays. The headers object must not contain a `status`
key and must not contain key names that end in '-' or '_'. It must contain keys that consist of letters, digits,
underscore, or dash and start with a letter. Header values must not contain characters below 037.

* content-type: There must be a `content-type` key except when the `status` is 100-199, 204, or 304 in which case the `content-type` must not be present.
* content-length: There must not be a `content-length` key when the status is 100-199, 204, or 304.

### Body

Must be a ForEachable.

## ForEachable

A ForEachable is a JavaScript Object that has a `forEach` method. The `forEach` method must take a callback that
will be invoked a single argument representing a value.

If the ForEachable is asynchronous, the `forEach` method must return a Promise. The Promise will be resolved when
the iteration is completed.

### Example

    function SimpleAsyncForEachable() {
      var values = [1,2,3,4]
        , deferred = q.defer();

      this.forEach = function(callback) {
        function next() {
          if (values.length > 0) {
            setTimeout(function() {
              // Simulate Async Stream
              callback(values.shift());
              next();
            }, 0);
          } else {
            deferred.resolve();
          }
        }

        return deferred.promise;
      }
    }

### Reasons for JSGI B

JSGI B is intended to unify the concepts of JSGI A Applications and Middleware and redefine Application as a term
for a group of Middleware that work together to produce a response. By passing the next application to call as an
optional parameter to a Middleware, the composition of Middleware becomes more transient and composable. There is also
one less closure which should aid in understanding Middleware and making it more straightforward to learn.

URL information on the Request match the naming scheme of the `location` property of `window` in the browser 
in an effort to make JSGI more consistent with existing JavaScript means of reprenseting a URL.

## Examples

### Request

    {
      version: "1.1", // parsed HTTP version
      method: "GET",
      headers: { // HTTP headers, lowercased
        host: "github.com",
        "user-agent": "Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.9.1.3) Gecko/20090824 Firefox/3.5.3",
        accept: "text/html",
        "accept-encoding": "gzip,deflate"
      },
      input: (new ForEachable), // request body stream
      hostname: "github.com",
      host: "github.com",
      protocol: "https",
      search: "",
      pathname: "/nrstott/bogart"
      env: { 
        // env is reserved for Server and Middleware to add keys and is shared amongst Requests
      }
    }

### Response

    {
      status: 200,
      headers: { "content-type": "text/html" },
      body: [ "Hello World" ]
    }

### Middleware Example

    function LogRequest(req, next) {
      writeToLog(new Date(), JSON.strigify(req)); // assume writeToLog is defined elsewhere
      return next(req);
    }

    function NotFound(req, next) {
      var deferred = q.defer();
      q.when(next(req), deferred.resolve, function onRejection() {
        return {
          status: 404,
          body: [ 'Not Found' ],
          headers: { 'content-type': 'text/plain' }
        }
      });

      return deferred.promise;
    }

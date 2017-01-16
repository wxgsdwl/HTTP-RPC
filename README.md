# Introduction
HTTP-RPC is an open-source framework for simplifying development of REST applications. It allows developers to access REST-based web services using a convenient, RPC-like metaphor while preserving fundamental REST principles such as statelessness and uniform resource access.

The project currently includes support for consuming web services in Objective-C/Swift and Java (including Android). It provides a consistent, callback-based API that makes it easy to interact with services regardless of target device or operating system.

For example, the following code snippet shows how a Swift client might access a simple web service that returns a friendly greeting:

_Swift_

    serviceProxy.invoke("GET", path: "/hello") { result, error in
        print(result) // Prints "Hello, World!"
    }

In Java, the code might look like this:

_Java_

    serviceProxy.invoke("GET", "/hello", (result, exception) -> {
        System.out.println(result); // Prints "Hello, World!"
    });

In either case, the operation would return the string "Hello, World!".

This guide introduces the HTTP-RPC framework and provides an overview of its key features.

# Contents
* [Objective-C/Swift Client](#objective-cswift-client)
* [Java Client](#java-client)
* [Additional Information](#additional-information)

# Objective-C/Swift Client
The Objective-C/Swift client enables iOS and tvOS applications to consume REST-based web services. It is distributed as a universal framework that contains a single `WSWebServiceProxy` class, discussed in more detail below. 

The iOS and tvOS frameworks can be downloaded [here](https://github.com/gk-brown/HTTP-RPC/releases). They are also available via [CocoaPods](https://cocoapods.org/pods/HTTPRPC). iOS 8 or later or tvOS 10 or later is required.

## WSWebServiceProxy Class
The `WSWebServiceProxy` class serves as a client-side invocation proxy for web services. Internally, it uses an instance of `NSURLSession` to issue HTTP requests. 

Service proxies are initialized via the `initWithSession:serverURL:` method, which takes an `NSURLSession` instance and the service's base URL as arguments. Service operations are initiated by calling the `invoke:path:arguments:resultHandler:` method, which takes the following arguments:

* `method` - the HTTP method to execute
* `path` - the path to the requested resource
* `arguments` - a dictionary containing the request arguments as key/value pairs
* `resultHandler` - a callback that will be invoked upon completion of the method

A convenience method is also provided for invoking operations that don't take any arguments. Both variants return an instance of `NSURLSessionTask` representing the invocation request. This allows an application to cancel a task, if necessary.

### Arguments
As with HTML forms, arguments are submitted either via the query string or in the request body. Arguments for `GET` and `DELETE` requests are always sent in the query string. `POST` arguments are always sent in the request body, and may be submitted using either the standard W3C [URL-encoded form](https://www.w3.org/TR/html401/interact/forms.html#h-17.13.4.1) or [multi-part form data](https://www.w3.org/TR/html401/interact/forms.html#h-17.13.4.2) encodings or as JSON. `PUT` arguments may be submitted either as JSON (sent in the request body) or via the query string.

The request encoding is set via the `encoding` property of the service proxy instance. HTTP-RPC provides the following constants representing the supported encoding types:

* `WSApplicationXWWWFormURLEncoded`
* `WSMultipartFormData`
* `WSApplicationJSON`

The default value is `WSMultipartFormData`.

#### Query String Encoding
Arguments may be of any type, and are converted to parameter values via the `description` method. Arrays represent multi-value parameters and behave similarly to `<select multiple>` tags in HTML forms. 

#### URL-Encoded Form Encoding
Identical to query string, but values are submitted in request body.

#### Multi-Part Form Data Encoding
Similar to URL-encoded form encoding, with additional support for uploading binary content. Arguments are generally converted to parameter values via the `description` method. However, instances of `NSURL` represent "files" and behave similarly to `<input type="file">` tags in HTML forms. Arrays of URL values behave similarly to `<input type="file" multiple>` tags.

#### JSON Encoding
Arguments are converted to JSON using `NSJSONSerialization`. A root JSON object containing the arguments as top-level elements is sent in the request body.

### Return Values
The result handler is called upon completion of the operation. If successful, the first argument will contain a deserialized representation of the content returned by the server. Otherwise, the first argument will be `nil`, and the second argument will be populated with an `NSError` instance describing the problem that occurred.

`WSWebServiceProxy` accepts the following response types:

* _application/json_
* _image/*_
* _text/*_

`NSJSONSerialization` is used to decode JSON response data, and `UIImage` is used to decode image content. Text content is returned as a string.

Note that, while requests are actually processed on a background thread, result handlers are called on the same operation queue that initially invoked the service method. This is typically the application's main queue, which allows result handlers to update the application's user interface directly, rather than posting a separate update operation to the main queue.

### Authentication
Although it is possible to use the `URLSession:didReceiveChallenge:completionHandler:` method of the `NSURLSessionDelegate` protocol to authenticate service requests, this method requires an unnecessary round trip to the server if a user's credentials are already known up front, as is often the case.

HTTP-RPC provides an additional authentication mechanism that can be specified on a per-proxy basis. The `authorization` property can be used to associate a set of user credentials with a proxy instance. This property accepts an instance of `NSURLCredential` identifying the user. When specified, the credentials are submitted with each service request using basic HTTP authentication.

**IMPORTANT** Since basic authentication transmits the encoded username and password in clear text, it should only be used with secure (i.e. HTTPS) connections.

## Examples
The following code snippet demonstrates how `WSWebServiceProxy` can be used to access a hypothetical math service:

    // Create service proxy
    let serviceProxy = WSWebServiceProxy(session: URLSession.shared, serverURL: URL(string: "https://localhost:8443")!)
    
    // Get sum of "a" and "b"
    serviceProxy.invoke("GET", path: "/math/sum", arguments: ["a": 2, "b": 4]) { result, error in
        // result is 6
    }

    // Get sum of all values
    serviceProxy.invoke("GET", path: "/math/sum", arguments: ["values": [1, 2, 3, 4]]) { result, error in
        // result is 6
    }

# Java Client
The Java client enables Java applications (including Android) to consume REST-based web services. It is distributed as a JAR file that contains the following types, discussed in more detail below:

* `WebServiceProxy` - web service invocation proxy
* `WebServiceException` - exception thrown when a service operation returns an error
* `ResultHandler` - callback interface for handling service results

The Java client library can be downloaded [here](https://github.com/gk-brown/HTTP-RPC/releases). Java 8 or later is required.

## WebServiceProxy Class
The `WebServiceProxy` class serves as a client-side invocation proxy for web services. Internally, it uses an instance of `HttpURLConnection` to send and receive data. 

Service proxies are initialized via a constructor that takes the following arguments:

* `serverURL` - an instance of `java.net.URL` representing the base URL of the service
* `executorService` - an instance of `java.util.concurrent.ExecutorService` that is used to  dispatch service requests

Service operations are initiated by calling the `invoke()` method, which takes the following arguments:

* `method` - the HTTP method to execute
* `path` - the path to the requested resource
* `arguments` - a map containing the request arguments as key/value pairs
* `resultHandler` - an instance of `ResultHandler` that will be invoked upon completion of the service operation

A convenience method is also provided for invoking operations that don't take any arguments. Both variants return an instance of `java.util.concurrent.Future` representing the invocation request. This object allows a caller to cancel an outstanding request, obtain information about a request that has completed, or block the current thread while waiting for an operation to complete.

### Arguments and Return Values
TODO Encodings (request/response); note that JSON does not support date types

`WebServiceProxy` deserializes JSON and plain text content automatically, and can be extended to support additional content types. Custom deserialization is discussed in more detail later.

Arguments may be of any type, and are generally converted to parameter values via the `toString()` method. However, the following types are given special consideration:

* Instances of `java.net.URL` represent binary content. They behave similarly to `<input type="file">` tags in HTML and can only be used with `POST` requests. 
* Instances of `java.util.List` represent multi-value parameters and generally behave similarly to `<select multiple>` tags in HTML forms. However, lists containing URL values are handled like `<input type="file" multiple>` tags in HTML and and can only be used with `POST` requests. 

The result handler is called upon completion of the operation. `ResultHandler` is a functional interface whose single method, `execute()`, is defined as follows:

    public void execute(V result, Exception exception);

On successful completion, the first argument will contain the value returned by the server. This will typically be an instance of one of the following types or `null`:

* string: `String`
* number: `Number`
* true/false: `Boolean`
* array: `java.util.List`
* object: `java.util.Map`

If an error occurs, the first argument will be `null`, and the second will contain an exception representing the error that occurred.

### Argument Map Creation
Since explicit creation and population of the argument map can be cumbersome, `WebServiceProxy` provides the following static convenience methods to help simplify map creation:

    public static <K> Map<K, ?> mapOf(Map.Entry<K, ?>... entries) { ... }
    public static <K> Map.Entry<K, ?> entry(K key, Object value) { ... }
    
Using these methods, argument map creation can be reduced from this:

    HashMap<String, Object> arguments = new HashMap<>();
    arguments.put("a", 2);
    arguments.put("b", 4);
    
to this:

    Map<String, Object> arguments = mapOf(entry("a", 2), entry("b", 4));
    
A complete example is provided later.

### Nested Structures
`WebServiceProxy` also provides the following static method for accessing nested map values by key path:

    public static <V> V getValue(Map<String, ?> root, String path) { ... }
    
For example, given the following JSON response, a call to `getValue(result, "foo.bar")` would return 123:

    {
        "foo": {
            "bar": 123
        }
    }

### Custom Deserialization
TODO decodeImageResponse(), decodeTextResponse()

Subclasses of `WebServiceProxy` can override the `decodeResponse()` method to provide custom deserialization behavior. For example, an Android client could override this method to support `Bitmap` data: 

    @Override
    protected Object decodeResponse(InputStream inputStream, String contentType) throws IOException {
        Object value;
        if (contentType.toLowerCase().startsWith("image/")) {
            value = BitmapFactory.decodeStream(inputStream);
        } else {
            value = super.decodeResponse(inputStream, contentType);
        }

        return value;
    }

### Multi-Threading Considerations
By default, a result handler is called on the thread that executed the remote request. In most cases, this will be a background thread. However, user interface toolkits generally require updates to be performed on the main thread. As a result, handlers typically need to "post" a message back to the UI thread in order to update the application's state. For example, a Swing application might call `SwingUtilities#invokeAndWait()`, whereas an Android application might call `Activity#runOnUiThread()` or `Handler#post()`.

While this can be done in the result handler itself, `WebServiceProxy` provides a more convenient alternative. The protected `dispatchResult()` method can be overridden to process all result handler notifications. For example, the following Android-specific code ensures that all result handlers will be executed on the main UI thread:

    WebServiceProxy serviceProxy = new WebServiceProxy(serverURL, executorService) {
        private Handler handler = new Handler(Looper.getMainLooper());

        @Override
        protected void dispatchResult(Runnable command) {
            handler.post(command);
        }
    };

Similar dispatchers can be configured for other Java UI toolkits such as Swing, JavaFX, and SWT. Command line applications can generally use the default dispatcher, which simply performs result handler notifications on the current thread.

### Authentication
Although it is possible to use the `java.net.Authenticator` class to authenticate service requests, this class can be difficult to work with, especially when dealing with multiple concurrent requests or authenticating to multiple services with different credentials. It also requires an unnecessary round trip to the server if a user's credentials are already known up front, as is often the case.

HTTP-RPC provides an additional authentication mechanism that can be specified on a per-proxy basis. The `setAuthorization()` method can be used to associate a set of user credentials with a proxy instance. This method takes an instance of `java.net.PasswordAuthentication` identifying the user. When specified, the credentials are submitted with each service request using basic HTTP authentication.

**IMPORTANT** Since basic authentication transmits the encoded username and password in clear text, it should only be used with secure (i.e. HTTPS) connections.

## Examples
The following code snippet demonstrates how `WebServiceProxy` can be used to access a hypothetical math service:

    // Create service proxy
    WebServiceProxy serviceProxy = new WebServiceProxy(new URL("https://localhost:8443"), Executors.newSingleThreadExecutor());

    // Get sum of "a" and "b"
    serviceProxy.invoke("GET", "/math/sum", mapOf(entry("a", 2), entry("b", 4)), new ResultHandler<Number>() {
        @Override
        public void execute(Number result, Exception exception) {
            // result is 6
        }
    });
    
    // Get sum of all values
    serviceProxy.invoke("GET", "/math/sum", mapOf(entry("values", listOf(1, 2, 3))), new ResultHandler<Number>() {
        @Override
        public void execute(Number result, Exception exception) {
            // result is 6
        }
    });

Note that lambda expressions can optionally be used instead of anonymous inner classes to implement result handlers, reducing the invocation code to the following:

    // Get sum of "a" and "b"
    serviceProxy.invoke("GET", "/math/sum", mapOf(entry("a", 2), entry("b", 4)), (result, exception) -> {
        // result is 6
    });

    // Get sum of all values
    serviceProxy.invoke("GET", "/math/sum", mapOf(entry("values", listOf(1, 2, 3))), (result, exception) -> {
        // result is 6
    });

# Additional Information
For examples and additional information, see the [the wiki](https://github.com/gk-brown/HTTP-RPC/wiki).



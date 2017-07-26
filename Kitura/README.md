# Kitura.next Manifesto

A long stated goal of the Kitura framework has been to depended on and adopt the Swift standard and core libraries (Dispatch and Foundation) where possible in order to make the framework feel familiar to the developer, remove the need to convert types in order to reuse existing modules, and maximise the portability of application code.

With the upcoming release of Swift 4, and the progress being made on the Swift Server APIs project, there are now additional standard capabilities that can be adopted. In particular, the introduction of Codable enables the removal of the dependency on SwiftyJSON and the move to a standard approach to JSON parsing.

The side effect of adopting these standard capabilities is that Kitura's API surface needs to be updated, which means the release of a Kitura.next. This also provides us with an opportunity to deliver a number of additional goals, features and capabilities. 

## Proposed new features in Kitura.next

**1. Optional typing and data validation of HTTP request parameters**  

The Kitura router currently has the following API specification:

```
public func get(_ path: String?=nil, handler: @escaping RouterHandler...) -> Router 
```

This allows you to define the route path as a String, and respond with a `RouterHandler` callback which is defined as:

```
typealias RouterHandler = (RouterRequest, RouterResponse, @escaping () -> Void) throws -> Void
```

Here you can access a Dictionary of URL Encoded HTTP request parameters using `RouterRequest.queryParameters`, or use the BodyParser middleware to access the body converted to a type such as JSON using `RouterRequest.body`.

These approaches directly reflect the REST API exposed by the server. Whilst this might be the expectation for a pure server developer, it doesn't facilitate the what a full stack developer wants to do: have code hosted on a client interact with code hosted on the server. Additionally it presents String based data - which is a limitation of the transport - and puts the emphasis on the developer to validate the data and convert to concrete Swift types.

_Proposed additional capabilities:_  
* Add the ability to optionally request the incoming data is converted into a struct or class
* Add the ability to optionally request that incoming data is validated using a provided function
* Add the ability to optionally specify a struct or class as the return type
* Add the ability to optionally request that returned data is validated using a provided function


**2. Data streaming**  

The ability to read and write stream data over HTTP via asynchronous callbacks is one of the valued features of Node.js, contributing to its ability to respond efficiently to a large number of requests with a low memory overhead.

Whilst this is possible in Swift, the Kitura router however currently only processes complete sets of data: data associated with incoming requests is read fully and provided via the body field in the RouterRequest that can be accessed as a parsed data type (eg. JSON) or as a Data buffer or String.

This means that data being uploaded or downloaded needs to be fully read and then stored as a complete entity in memory, affecting the performance and memory usage of applications that stream data such as static files, video or audio.

_Proposed additional capabilities:_ 
* Add the ability to read from and write to the body of a request or response as a stream using asynchronous callbacks
* Add the ability for the request or response types to include an embedded stream as part of the defined struct or class

**3. Integrated support for OpenAPI (Swagger)**  

The OpenAPI specification format (aka Swagger) is the standard definition format used to describe RESTful APIs. The OpenAPI specification for a server is typically used by:
* Developers  
To understand what APIs are available and what parameters are accepted and returned.
* Client SDK Code generation tools  
To  create connector SDKs to make calls to the API as a convenience layer that implements the data types and API calls.
* Automated Testing tools  
To provide automated test generation and execute against the API. 
* API Gateways  
To allow the severs APIs to be registered with it for security and management of calls to the APIs.

The Swift Server Generator provided by Kitura will already use a provided OpenAPI specification to:
1. Create a the Kitura application that implements the described RESTful API
2. Host the OpenAPI specification as part of the application so that it can be registered with API Gateways
3. Create an iOS client SDK to make it easier to make calls to the Kitura server 

However this process is one way: the Kitura server and the iOS client can be created from the OpenAPI specification, but if the API is modified by adding, removing or updating the routes in the Kitura application, the OpenAPI spec becomes out of sync and must be updated manually.

_Proposed additional capabilities:_ 
* Closed loop development process whereby an OpenAPI specification can be generated from the application
* Ability to verify the OpenAPI specification from the application against a provided specification to provide an initial functional verification test.


**4. Support for alternative, pluggable, transports**  
The current de-facto standard for sending and receiving data via RESTful APIs is to format the data as JavaScript Object Notation (JSON) objects. This has been one of the drivers for full-stack development of web applications with Node.js on the backend, as the client and server are serialising and transmitting data in their native format.

Whilst a particularly appropriate transport for full-stack web applications, it is less so for full-stack Swift applications: JSON is a text based format that includes field names and therefore is not memory efficient, and the values stored in the format are not typed. The first means that data payloads being sent between an iOS client and a server are significantly larger than they could be, and the second means that type information is lost and complex serialisation and deserialisation conversions need to be carried out.

Alternative transports that are emerging that are both data optimised and strongly typed making it possible to share exact data models between iOS client and server and to minimise data requirements. The most popular of these is gRPC and protobufs, but others are available including Thrift.

Adoption alternative transports provides a huge benefit for full-stack Swift developers, however at the cost of interoperability: every client has to use the same transport - unless the Server framework is able to handle different transports concurrently without the application having to be modified.

_Proposed additional capabilities:_ 
* Add built-in support for alternative transports, eg. gRPC and Thrift based on configuration
* Enable handling of different transports dynamically


**5. Portability with Serverless Swift**  
"Serverless" frameworks like AWS Lambda and Apache OpenWhisk make it possible to create and run simple asynchronous actions, making it easy to add simple backend components to an application, without the complexity of running and managing a server framework.

Today, the programming models for server and serverless applications are different. For example, an OpenWhisk action is implemented as:

```
     func run(args: [String:Any]) -> [String:Any] {}
```

whereas a server framework receives a request object that contains a body which could be optionally formatted JSON, etc.

This means that a potentially significant amount of refactoring is required if you wanted to migrate a set of serverless actions into a full server application, or decomposing an existing server application into a set of serverless actions.

_Proposed additional capabilities:_ 
* Enable serverless actions to be registered to run inside the Kitura router
* Enable the Kitura router to be registered as a set of serverless actions


**6. More intuitive middlewares**  
The current approach to middlewares implements a single API call which is called twice: one during processing of the request and once during processing of the response:

```
func handle(request: RouterRequest, response: RouterResponse, next: @escaping () -> Void) 
```

As a single API is used, the middleware code needs to be able to determine whether a request or response processing and currently occurring, and both the request and response need to be mutable. Additionally there is no clear order in which middlewares execute, and middlewares must call `next()` in order to avoid causing a hang.

This makes implementation of middlewares unnecessarily complex and prone to errors, so would benefit from simplification of the programming model.

_Proposed additional capabilities:_ 
* Separate API calls for request and response processing
* Predictable ordering of middleware execution
* Use of mutable and immutable types to remove risk of incorrect modification
* Removal of the requirement to call `next()`


## Summary
The proposed new features are designed to provide a more intuitive programming model for Swift developers,  to make it easier to develop full-stack Swift applications, sharing models and code between client and server, and to make it easier to deploy Kitura as a highly available and scalable production framework.

Feedback is encouraged on whether these items achieve this, whether other features should be considered, and what the relative priorities are for each of the features.

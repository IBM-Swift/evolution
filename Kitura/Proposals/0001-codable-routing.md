## Codable Routing
* Proposal: KIT-0001
* Authors: [Chris Bailey](https://github.com/seabaylea), [Mike Tunnicliffe](https://github.com/tunniclm), [Ricardo Olivieri](https://github.com/rolivieri)
* Review Manager: [Lloyd Roseblade](https://github.com/lroseblade)
* Status: DRAFT
* Implementation: Link to branch
* Previous Revision: 1
* Previous Proposal: NNNN

### Introduction
The Kitura router currently requires you to interact with `RouterRequest` and `RouterResponse` types. This means inspecting and setting the correct HTTP headers, and working with the enclosed request and response body data directly, including carrying out data validation, error checking, JSON parsing, and setting the correct HTTP response codes.

This proposal adds a new set of convenience APIs that provides an abstraction layer for the developer, where the framework deals with data validation and processing, and provides on requested concrete Swift types to the application code.

### Motivation
The current API provided by the Kitura Router provides the developer with significant amounts of flexibility and low level control. This, however, also means that the developer must write relatively complex code to carry out simple tasks. For example, the following is a "best practice" implementation of a RESTful API for a `POST` request on `/` that expects to receive a `Name` object as a JSON payload, defined as follows:

```
{
    "name":   String
}
```

and that returns the same object (once its successfully stored on the server):

```swift
router.all("/*", middleware: BodyParser())
router.post("/", handler: handlePost)

func handlePost(request: RouterRequest, response: RouterResponse, next: @escaping () -> Void) {
    // Check that the HTTP "Content-Type" header is correctly set to "application/json"
    // If not return `.unsupportedMediaType`
        guard let contentType = request.headers["Content-Type"], contentType.hasPrefix("application/json") else {
        response.status(.unsupportedMediaType)
        response.send(json: JSON([ "error": "Request Content-Type must be application/json" ]))
        return next()
    }
    
    // Check that the body data could be parsed as JSON or return `.badRequest`
    guard case .json(let json)? = request.body else {
        response.status(.badRequest)
        response.send(json: JSON([ "error": "Request body could not be parsed as JSON" ]))
        return next()
    }
    
    // Validate the JSON objects matches the expected fields and types
    // Return `.unprocessableEntity` for any errors
    guard json["name"].exists() else {
        response.status(.unprocessableEntity)
        response.send(json: JSON([ "error": "Missing reqired property `name`" ]))
        next()
    }
    guard let name = json["name"].string else {
        response.status(.unprocessableEntity)
        response.send(json: JSON([ "error": "Property type mismatch, `name` is a json["name"].description but expected String" ]))
        next()
    }
    
    // Finally have our single field from the JSON object
    let receivedName = name
    
    //
    // Insert the application logic code here
    //
    
    // Return the Name object as JSON
    response.send(json: JSON(receivedname))
    }
```

### Proposed solution
This proposal covers an additional layer of API that provides abstraction and convenience over the existing lower level API. This new layer additionally leverages the new `Codable` capabilities delivered in Swift 4, allowing the developer to work solely with types that conform to `Codable`.

Utilizing the new API layer, the above code becomes:

```swift
struct Name: Codable {
   let name: String
   
   init(name: String) {
       self.name = name
    }
}

router.post("/", handler: handlePost)

func handlePost(name: Name, completion: (Name?, Error?) -> Void) {
    
    //
    // Insert the application logic code here
    //
    
    completion(name, error)
}
```

Here the developer has specified the `Codable` types that the handler expects and that it will return via a completion handler, while the framework handles HTTP header checks, encoding/decoding, data validation, and the setting of correct HTTP response codes.

### Detailed design

#### Codable Routing
The new API adds the ability to specify `Codable` route handlers that denote the types the handler expects to receive and that it will respond with. The framework is therefore expected to validate that the incoming request body can be converted to this type and reject non-conforming requests with the correct error codes.

Below is the new API specification for Codable routes:

```swift
extension Router {

// POST: handler receives body data as a Codable and responds with a Codable via the completion handler
post(_ route: String, handler: @escaping (Codable, completion: @escaping (Codable?, Error?) -> Void) throws -> Void)

// GET: handler receives no body data and responds with an array of Codable via the completion handler
get(_ route: String, handler: @escaping (completion: @escaping ([Codable]?, Error?) -> Void) throws -> Void)

// DELETE handler receives no body data and responds by calling the completion handler with no data
delete(_ route: String, handler: @escaping (completion: @escaping (Error?) -> Void) throws -> Void)

}
```

This however is limited, as most RESTful APIs also use URL encoded data. For example:
```
    GET /user/1
```
would typically be used to retrieve data associated with a user uniquely identified by ID 1. 

#### Identifier Routing
These use cases are handled by providing the additional ability for the developer to request that a type that conforms to `Identifier` is also passed to the handler, for example:

```swift
struct UserId: Identifier {
    public let id: Int
    public init(value: String) throws {
        if let id = Int(value) {
            self.id = id
        } else {
            throw TypeError.unidentifiable
        }
    }
}

router.get("/user", handler: handleGetUser)

func handleGetUser(id: UserID, completion: (User?, Error?) -> Void) {
    completion(userStore[id.id], nil)
}
```

Here the `Identifier` struct provides a constructor that accepts a String value. This is called by the Router with the value encoded in the root URL. 

Below is the new API specification for additionally `Identifier` routes:

```swift
// GET: handler receives an Identifier and responds with a Codable via the completion handler
get(_ route: String, handler: @escaping (Identifier, completion: @escaping (Codable?, Error?) -> Void) throws -> Void)

// PUT: handler receives an Identifier and responds with a Codable via the completion handler
put(_ route: String, handler: @escaping (Identifier, completion: @escaping (Codable?, Error?) -> Void) throws -> Void)

// PATCH: handler receives an Identifier and responds with a Codable via the completion handler
patch(_ route: String, handler: @escaping (Identifier, completion: @escaping (Codable?, Error?) -> Void) throws -> Void)

// DELETE: handler receives an Identifier and responds by calling the completion handler with no data
delete(_ route: String, handler: @escaping (Identifier, completion: @escaping (Error?) -> Void) throws -> Void)
```
**Note:** This is a simplification of how `Identifier` is intended to work. It will be covered in more detail in a subsequent proposal.

#### Error Handling
For each of the Codable and Identifier routes, the handler has the ability to return an error. This provides the handler with a mechanism to report an error which affects the headers and return codes used in the underlying HTTP response. The following error values are available, which matches the possible HTTP status codes:

```swift
public enum RouteHandlerError: Error {
    case accepted = 202, badGateway = 502, badRequest = 400, conflict = 409, `continue` = 100, created = 201
    case expectationFailed = 417, failedDependency  = 424, forbidden = 403, gatewayTimeout = 504, gone = 410
    case httpVersionNotSupported = 505, insufficientSpaceOnResource = 419, insufficientStorage = 507
    case internalServerError = 500, lengthRequired = 411, methodFailure = 420, methodNotAllowed = 405
    case movedPermanently = 301, movedTemporarily = 302, multiStatus = 207, multipleChoices = 300
    case networkAuthenticationRequired = 511, noContent = 204, nonAuthoritativeInformation = 203
    case notAcceptable = 406, notFound = 404, notImplemented = 501, notModified = 304, OK = 200
    case partialContent = 206, paymentRequired = 402, preconditionFailed = 412, preconditionRequired = 428
    case proxyAuthenticationRequired = 407, processing = 102, requestHeaderFieldsTooLarge = 431
    case requestTimeout = 408, requestTooLong = 413, requestURITooLong = 414, requestedRangeNotSatisfiable = 416
    case resetContent = 205, seeOther = 303, serviceUnavailable = 503, switchingProtocols = 101
    case temporaryRedirect = 307, tooManyRequests = 429, unauthorized = 401, unprocessableEntity = 422
    case unsupportedMediaType = 415, useProxy = 305, misdirectedRequest = 421, unknown = -1
}
```
**To Do:** Not all of these need to be supported. For example the scenarios where `.httpVersionNotSupported` would be relevant should be handled by the Codable router itself.

### Alternatives considered

#### Blocking/Synchronous API
Whilst a radical simplification over the work that a developer previously needed to do in order to implement a RESTful API, the new Codable Router APIs look, at least initially, to be convoluted and non-intuative. This is largely driven by the specification of two closures:

1. The handler closure, which provides the application logic to be called.
2. The completion closure, which the application logic uses to provide an asynchronous response.

The API could therefore be simplified by removing the completion closure (#2) and having the handler (#1) return the required result, e.g.:

```swift
 // POST: handler receives body data as a Codable and responds with a Codable via the completion handler
post(_ route: String, handler: @escaping (Codable, completion: @escaping (Codable?, Error?) -> Void) -> Void)
```

becomes:

```swift
 // POST: handler receives body data as a Codable and responds with a Codable via the completion handler
post(_ route: String, handler: @escaping (Codable) throws -> Codable)
```
This however comes as the cost of being able to do asynchronous processing.

It is conceivable to support both types of API, however doing so adds an additional level of complexity for the developer, who now has to understand the differences between the two APIs and choose which to use. This could however be partially mitigated by using the approach that is common in Node.js, which is to add a `sync` identifier to the function name, eg:

```swift
 // POST: handler receives body data as a Codable and responds with a Codable via the completion handler
postSync(_ route: String, handler: @escaping (Codable) throws -> Codable)
```

Whilst taking this approach is an option, this proposal does not cover adding these `sync` APIs.

### Further Work
As noted above, a further proposal is required to cover the implementation of the `Identifier` protocol in-depth. Additionally, Route handlers typically use two other types of data that are not covered by this proposal:

1. Authenticated user information.
2. URL encoded parameters.

These will also be covered in subsequent proposals, however the expectation is that these will take a similar approach, with additional parameter types requested by the developer being passed to their route handlers.

### Feedback
Feedback should be via either (or both) of the following routes:

1. The following GitHub issue:  
   https://github.com/IBM-Swift/evolution/issues/2
2. The "open-playback" channel in the Swift@IBM Slack:  
   https://swift-at-ibm-slack.mybluemix.net/

## Query Parameters for Codable Routing
* Proposal: KIT-0002
* Authors: [Ricardo Olivieri](https://github.com/rolivieri), [Chris Bailey](https://github.com/seabaylea)
* Review Manager: [Lloyd Roseblade](https://github.com/lroseblade)
* Status: DRAFT
* Implementation: Link to branch
* Previous Revision: 1
* Previous Proposal: N/A

### Introduction
The latest version of Kitura (which at the time of writing is `2.0.2`) provides a new set of "Codable Routing" APIs that developers can leverage for implementing route handlers that take advantage of the new [`Codable`](https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types) protocol that was made available with the release of [Swift 4](https://swift.org/blog/swift-4-0-released/). This new set of Kitura APIs provides developers with an abstraction layer, where the framework hides the complexity of processing the HTTP request and HTTP response objects and makes available the requested concrete Swift types to the application code.

This proposal seeks to augment the Codable Routing APIs so that [query parameters](https://en.wikipedia.org/wiki/Query_string) can be provided to the application code in a type-safe manner, without requiring the developer to understand the underlying pinnings of HTTP requests and the composition of query strings.

### Motivation
Though the Codable Routing APIs recently introduced in Kitura did simplify greatly the effort for processing incoming HTTP requests, these APIs currently do not provide to the application code the values that make up a query string included in an HTTP request.

For example, it is common to provide a REST API that allows the client to make a request that includes a set of filters as query parameters. In the case below the client application is making an HTTP `GET` request for a list of employees, but also requesting that the result is filtered by countries, position, and level using a URL encoded query string:

```
GET http://localhost:8080/employees?countries=US,UK&position=developer&level=55
```

The current Codable Routing APIs however do not provide a mechanism for accessing the query parameters. For example, let's take a look at the following snippet of code:

```swift
router.get("/employees") { (respondWith: ([Employees]?, RequestError?) -> Void) in
    // Get employee objects from storage
    if let employees = ... {
        // Return list of employees
        respondWith(employees, nil)
    } else {
        let error = ...
        respondWith(nil, error)
    }
}
```

This only allows a client to submit an HTTP `GET` request against the `http://localhost:8080/employees` URL and for the server to return a list of `Employee` entities.

Currently the only approach available for implementing a route with URL encoded query parameters is to fall back to the traditional "Raw Routing" APIs, requiring the interaction with `RouterRequest` and `RouterResponse` objects in your application handlers. This removes all of the ease of use and types safety advantages of using Codable Routing.

### Proposed solution
This proposal covers augmenting the new Codable Routing APIs in Kitura `2.x` to make the keys and values contained in query strings available to the appplication code in a type-safe manner. With the new proposed addition to the Codable Routing APIs, the above code becomes:

```swift
router.get("/employees") { (query: EmployeeQuery, respondWith: ([Employees]?, RequestError?) -> Void) in
    // Filter data using the query parameters provided to the application
    let employees = employeeStore.map({ $0.value }).filter( { 
        ( $0.level == query.level &&
        $0.position == query.position &&
        query.countries.index(of: $0.country) != nil )
    })
    
    // Return list of employees
    respondWith(employees, nil)    
}
```

Here the developer has specified the Swift type (e.g. `EmployeeQuery`) that the handler closure expects to encapsulate the query parameters provided in an incoming HTTP request. Using the new proposed APIs, developers define a Swift type that conforms to the `Codable` protocol and encapsulates the fields that make up the query parameters for a corresponding route (e.g. `/employees`). In the sample above, the Swift type named `EmployeeQuery` encapsulates the different fields (i.e. `countries`, `position`, and `level`) that are part of the query string sent from a client application (e.g. `?countries=US,UK&position=developer&level=55`).

```swift
public struct EmployeeQuery: Codable {
    public let countries: [String]
    public let position: String
    public let level: Int
}
```

Using a concrete Swift type as the embodiment for query parameters provides type safety to developers and, at the same time, allows the framework do the heavy lifting, on behalf developers, for parsing the query string, extracting the key-value pairs, and transforming these into the expected data types.

### Detailed design

#### Codable routing and query parameters
The new addition to the Codable Routing APIs in Kitura adds the ability to specify Codable route handlers that denote the concrete Swift type the handler expects to receive as the embodiment for query parameters. Therefore, the framework is expected to validate that the elements found in the query string of the incoming HTTP request can be converted to their corresponding types (e.g. `String`, `Int`, `Float`, `Double`, etc.) and reject non-conforming requests.

Below is the new API specifications for augmenting the Codable Routing API routes in Kitura (note that we intend to augment the APIs for the HTTP `GET` and HTTP `DELETE` methods with an additional Swift type for wrapping query parameters):

```swift
extension Router {

    // GET: handler receives query object, receives no body data, and responds with an array of Codable entities
    func get<O: Codable, Q: Codable>(_ route: String, handler: @escaping (Q, ([O]?, RequestError?) -> Void) -> Void)
      
    // DELETE: handler receives query object, receives no body data, and responds with no data
    func delete<Q: Codable>(_ route: String, handler: @escaping (Q, (RequestError?) -> Void) -> Void)
}
```

#### Decoding Swift types at runtime
As described above, for a developer to take advantage of the new proposed APIs, he/she first defines the Swift type that encapsultates the fields that make up the query parameters for a given route and then defines the handler that conforms to either one of the function signatures defined above. This implies that the framework needs a mechanism for decoding the query string found in an incoming HTTP `GET` or HTTP `DELETE` request into the corresponding Swift type. Initially, we planned on using the [Reflection API](https://developer.apple.com/documentation/swift/mirror) in Swift but determined that a few limitations would arise from doing so. For instance, all fields in the Swift type would need to be [optionals](https://developer.apple.com/documentation/swift/optional) since the Reflection API requires an instance of the type before we can do anything meaningful such as figuring out the names and types of the instance fields. Hence, instead of using the reflection API, a custom [`Decoder`](https://developer.apple.com/documentation/swift/decoder) will determine the internal composition (i.e. properties and types) of the Swift type. This will allow developers to define Swift types, for their query entities, that can include optional and non-optional values. We refer to this custom `Decoder` as `QueryDecoder`.

The following list includes the types that we intend to initialy support in `QueryDecoder`:

- `Array<Int>`
- `Int`
- `Array<UInt>`
- `UInt`
- `Float`
- `Array<Float>`
- `Double`
- `Array<Double>`
- `Bool`
- `String`
- `Array<String>`
- `Date`
- `Array<Date>`
- `Codable`
  
Hence, to encapsulate query parameters, developers can define a Swift type with fields that are of any of the types listed above (optional and non-optional). 

##### Date type
The `Date` type is of special interest since date values specified in a query string will need to conform to the `yyyy-MM-dd'T'HH:mm:ssZ` format and use `UTC` as the time zone. Since this format and time zone are widely used for specifying date values in JSON payloads, we expect most applications can utilize `QueryDecoder` as is. However, there could be applications that require a different date format and/or time zone in their JSON payloads. To satisfy this need, `QueryDecoder` exposes a `DateFormatter` static variable that developers can modify to their needs:

```swift
QueryDecoder.dateFormatter.dateFormat = ...
QueryDecoder.dateFormatter.timeZone = ...
```

If it becomes critical, at a later point, we can introduce a new proposal for providing more fine-grained control for decoding date strings. For instance, the `DateFormatter` instance field in `QueryDecoder` can be made non-static, which would allow different decoding formats and time zones for date strings within the same Swift application.

##### Nested types that conform to Codable
It is also worth noting that developers can also take advantage of the decoding capabilities in `QueryDecoder` for nested `Codable` types, as shown below:

```swift
public struct UserQuery: Codable {
    public let level: Int?
    public let gender: String
    public let roles: [String]
    public let nested: Nested
}

public struct Nested: Codable {
    public let nestedInt: Int
    public let nestedString: String
}
```

As part of the query string for filtering `User` entities, a `nested` key can be provided that contains the following as its value: 

```json
{
  "nestedInt": 1234,
  "nestedString": "string"
}
```

Here's a sample query string that can be decoded into a `UserQuery` instance that includes a `nested` key:

```
?level=25&gender=female&roles=developer,tester,manager&nested={"nestedInt": 1234, "nestedString": "string"}
```

As part of the decoding process, `QueryDecoder` will decode the JSON string value mapped to the `nested` key and then create a corresponding `Nested` instance.

### Feedback
Feedback should be via either (or both) of the following routes:

1. The following GitHub issue:  
   https://github.com/IBM-Swift/evolution/issues/3
2. The "open-playback" channel in the Swift@IBM Slack:  
   https://swift-at-ibm-slack.mybluemix.net/

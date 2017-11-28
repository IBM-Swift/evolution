## Codable Query Parameters
* Proposal: KIT-0002
* Authors: [Ricardo Olivieri](https://github.com/rolivieri), [Chris Bailey](https://github.com/seabaylea)
* Review Manager: [Lloyd Roseblade](https://github.com/lroseblade)
* Status: DRAFT
* Implementation: Link to branch
* Previous Revision: 1
* Previous Proposal: N/A

### Introduction
The latest version of Kitura (which at the time of writing is `2.0.2`) provides a new set of APIs that developers can leverage for implementing route handlers that take adavantage of the new [`Codable`](https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types) protocol that was made available with the release of [Swift 4](https://swift.org/blog/swift-4-0-released/). This new set of Kitura APIs provides developers with an abstraction layer, where the framework hides the complexity of processing the HTTP request and HTTP response objects and makes available the requested concrete Swift types to the application code.

This proposal seeks to augment these new Codable APIs so that [query parameters](https://en.wikipedia.org/wiki/Query_string) are provided to the application code in a type-safe manner, without exposing to the developer the underlying pinnings of HTTP requests and the composition of query strings.

### Motivation
Though the new Codable APIs recenty introduced in Kitura did simplify greatly the effort for processing incoming HTTP requests, these APIs currently do not provide to the applicaton code the values that make up a query string included in an HTTP request. For example, let's take a look at the following snippet of code:

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

The sample code shown above leverages the new Codable APIs for processing an HTTP `GET` request for obtaining `Employee` records. Assuming there is a Kitura server that hosts the code shown above running locally, a client that submits an HTTP `GET` request against the `http://localhost:8080/employees` URL should expect to receive in the HTTP response a list of `Employee` entities. However, say the client application included a set of filters in the HTTP `GET` request in the form of a query string, similar to `http://localhost:8080/employees?countries=US,UK&position=developer&level=55`. In such case, the Codable handler for the `/employees` route does not have a way to process those filters since they are not made available to the application code. A workaround to this limitaton would be to revert back to using the traditional *non-codable* APIs that do not take advantage of the Codable protocol. However, doing so would be the equivalent of taking a step back since you'd be losing type safety and you'd end up interacting with `RouterRequest` and `RouterResponse` objects in your application handlers.

### Proposed solution
This proposal covers augmenting the new Codable APIs in Kitura `2.x` to make the keys and values contained in query strings available to the appplication code in a type-safe manner. With the new proposed addition to the Codable APIs, the above code becomes:

```swift
router.get("/employees") { (query: EmployeeQuery, respondWith: ([Employees]?, RequestError?) -> Void) in
    // Get filters
    let countries: [String] = query.countries
    let position: String = query.position
    let level: Int = query.level
    // Get employee objects from storage by
    // filtering the data using the query parameters
    // provided to the application
    if let employees = ... {
        // Return list of employees
        respondWith(employees, nil)
    } else {
        let error = ...
        // Return error
        respondWith(nil, error)
    }
}
```

Here the developer has specified the Swift type (e.g. `EmployeeQuery`) that the handler closure expects to encapsulate the query parameters provided in the incoming HTTP request. Using the new proposed API, developers define a Swift type that conforms to the `Codable` protocol and encapsulates the fields that make up the query parameters for a corresponding route (e.g. `/employees`). In the sample above, the Swift type named `EmployeeQuery` encapsulates the different fields (i.e. `countries`, `position`, and `level`) that are part of the query string sent from the client application (e.g. `?countries=US,UK&position=developer&level=55`).

```swift
public struct EmployeeQuery: Codable {
    public let countries: [String]
    public let position: String
    public let level: Int
}
```

Using a concrete Swift type as the embodiment for query parameters provides type safety to developers and, at the same time, allows the framework do the heavy-lifting, on behalf developers, for parsing the query string, extracting the key-value pairs, and transforming these into the expected data types.

### Detailed design

#### Codable routing and query parameters
The new addition to the Codable APIs in Kitura adds the ability to specify Codable route handlers that denote the concrete Swift type the handler expects to receive as the embodiment for query parameters. Therefore, the framework is expected to validate that the elements found in the query string of the incoming HTTP request can be converted to their corresponding types (e.g. `String`, `Int`, `Float`, `Double`, etc.) and reject non-conforming requests.

Below is the new API specifications for augmentimng the Codable API routes in Kitura (note that we intend to augment the APIs for the `GET` and `DELETE` methods with the additional Swift type for wrapping query parameters):

```swift
extension Router {

    // GET: handler receives query object, receives no body data, and responds with an array of Codable entities
    func get<O: Codable, Q: Codable>(_ route: String, handler: @escaping (Q, ([O]?, RequestError?) -> Void) -> Void)
      
    // DELETE: handler receives query object, receives no body data, and responds with no data
    func delete<Q: Codable>(_ route: String, handler: @escaping (Q, (RequestError?) -> Void) -> Void)
}
```

#### Decoding Swift types at runtime
As described above, for a developer to take advantage of the new proposed APIs, he/she first defines the Swift type that encapsultates the fields that make up the query parameters for a given route and then defines the handler that conforms to either one of the function signatures defined above. This implies that the framework needs a mechanism for decoding the query string found in an incoming HTTP `GET` or HTTP `DELETE` request into the corresponding Swift type. Initially, we planned on using the [Reflection API](https://developer.apple.com/documentation/swift/mirror) in Swift but determined that a few limitations would arise from doing so. For instance, all fields in the Swift type would need to be optionals since the Reflection API requires an instance of the type before we can do anything meaningful such as figuring out the names and types of the instance fields. Hence, instead of using the reflection API, a custom [`Decoder`](https://developer.apple.com/documentation/swift/decoder) will determine the internal composition (i.e. properties and types) of a Swift type that conforms to the `Codable` protocol. This will allow developers to define Swift types, for their query entities, that can include optional and non-optional values. We refer to this custom `Decoder` as `QueryDecoder`.

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

The `Date` type is of special interest since, by default, it will require date values specified in a query string to conform to the `yyyy-MM-dd'T'HH:mm:ssZ` format. Since this format is widely used for specifying date values in JSON payloads, we expect most applications can utilize the custom `Decoder` for query parameters as-is. However, there could be applications that require a different date format in their JSON payloads. To satisfy this need, `QueryDecoder` exposes a `DateFormatter` static variable that developers can modify to their needs:

```swift
QueryDecoder.dateFormatter.dateFormat = ...
QueryDecoder.dateFormatter.timeZone = ...
```

[**RO - There is a limitation here since what I am describing above is a single `DateFormatter` instance for the QueryDecoder class. In other words, it is a static field. If we wanted to provide more fine-granular control, we would need to make the `DateFormatter` instance and instance field and not a static field of the QueryDecoder class. In addition, we would then need to allow developers to somehow provide the QueryDecoder instance that a given route handler should use... for now, I am staying away from this... unless we think that this is a must do as part of this proposal**].

Developers can also take advantage of the decoding capabilities for nested `Codable` types, as shown below:

```swift
public struct UserQuery: Codable {
    public let age: Int?
    public let name: String
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

As part of the decoding process, `QueryDecoder` will decode the string that contains the JSON shown above and then create the correspding `Nested` instance.

### Feedback
Feedback should be via either (or both) of the following routes:

1. The following GitHub issue:  
   https://github.com/IBM-Swift/evolution/issues/3
2. The "open-playback" channel in the Swift@IBM Slack:  
   https://swift-at-ibm-slack.mybluemix.net/

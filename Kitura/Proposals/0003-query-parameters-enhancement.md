## API Improvement for Query Parameters
* Proposal: KIT-0003
* Authors: [Ricardo Olivieri](https://github.com/rolivieri), [Chris Bailey](https://github.com/seabaylea)
* Review Manager: [Lloyd Roseblade](https://github.com/lroseblade)
* Status: DRAFT
* Implementation: Link to branch
* Previous Revision: 1
* Previous Proposal: N/A

### Introduction
Applications that leverage the traditional non-codable APIs in the Kitura framework can obtain [query parameters](https://en.wikipedia.org/wiki/Query_string) provided in an HTTP request by accessing the `queryParameters` field in the `RouterRequest` type. The `queryParameters` field is a dictionary that is of the `[String : String]` type.

This proposal seeks to improve the current API so that applications can extract, without much effort, the values from the  `queryParameters` field in the desired type (e.g. `Int`, `Float`, `Array<Int>`, etc.).

### Motivation
With the current API, application developers implement code similar to what you see below in order to process query parameters provided to their applications:

```swift
router.get("/employees") { (request: RouterRequest, response: RouterResponse, next: @escaping () -> Void) in
    let params = request.queryParameters
    if let value: String = params["countries"] as? String {
        let countries: [String] = value.components(separatedBy: ",")
        ...
    }

    if let value: String = params["level"] as? String, let level = Int(value) {
        ...
    }

    if let value: String = params["ratings"] as? String {
        let strs: [String] = value.components(separatedBy: ",")
        let floats: [Float] = strs.map { Float($0) }.filter { $0 != nil }.map { $0! }
        if floats.count == strs.count {
           // process floats array
        }        
    }

    ...
}
```

Though the code shown above is not complex, it is quite a bit of boilerplate code that application developers need to write, test, and maintain.

### Proposed solution
This proposal covers augmenting the API in Kitura `2.x` for extracting query parameter values. With the new proposed API, the above code becomes:

```swift
router.get("/employees") { (request: RouterRequest, response: RouterResponse, next: @escaping () -> Void) in
    let params = request.queryParameters
    if let countries: [String] = params["countries"]?.stringArray {
        ...
    }

     if let level: Int = queryParams["level"]?.int {
        ...
    }

    if let ratings: [Float] = queryParams["ratings"]?.floatArray {
        ...
    }

    ...
}
```

Here the developer can obtain a query parameter in the desired type by simply invoking the corresponding computed property. The amount of code is significantly reduced with this new API and it is less error prone. The proposed API adds convenience for developers and, at the same time, allows the framework do the heavy lifting, on behalf of developers, for massaging the query parameter values out of the `[String : String]` dictionary.

### Detailed design

#### Codable routing and query parameters
The new addition to the Codable APIs in Kitura adds the ability to specify Codable route handlers that denote the concrete Swift type the handler expects to receive as the embodiment for query parameters. Therefore, the framework is expected to validate that the elements found in the query string of the incoming HTTP request can be converted to their corresponding types (e.g. `String`, `Int`, `Float`, `Double`, etc.) and reject non-conforming requests.

Below is the new API specifications for augmentimng the Codable API routes in Kitura (note that we intend to augment the APIs for the `GET` and `DELETE` methods with an additional Swift type for wrapping query parameters):

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

[**RO - There is a limitation here since what I am describing above is a single `DateFormatter` instance for the QueryDecoder class. In other words, it is a static field. If we wanted to provide more fine-granular control, we would need to make the `DateFormatter` instance and instance field and not a static field of the QueryDecoder class. In addition, we would then need to allow developers to somehow provide the QueryDecoder instance that a given route handler should use... for now, I am staying away from this... unless we think that this is a must do as part of this proposal**].

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

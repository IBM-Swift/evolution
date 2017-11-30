## API Improvements for Query Parameters
* Proposal: KIT-0002
* Authors: [Ricardo Olivieri](https://github.com/rolivieri), [Chris Bailey](https://github.com/seabaylea)
* Review Manager: [Lloyd Roseblade](https://github.com/lroseblade)
* Status: DRAFT
* Previous Revision: 1
* Previous Proposal: N/A

### Introduction
Applications that leverage the traditional Raw Routing APIs in the Kitura framework can obtain URL encoded [query parameters](https://en.wikipedia.org/wiki/Query_string) provided in an HTTP request by accessing the `queryParameters` field in the `RouterRequest` type. The `queryParameters` field is a dictionary that is of the `[String : String]` type.

This proposal seeks to improve the current API so that applications can extract, without much effort, the values from the `queryParameters` field in the desired type (e.g. `Int`, `Float`, `Array<Int>`, etc.).

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
           ...
        }        
    }

    ...
}
```

Though the code shown above is not complex, there is a considerable amount of boilerplate code that developers need to write, test, and maintain.

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

Here the developer can obtain a query parameter in the desired type by simply invoking the corresponding computed property. The amount of boilerplate code is significantly reduced with this new API and it is less error prone. The proposed API adds convenience for developers and, at the same time, allows the framework do the heavy lifting, on behalf of developers, for processing the query parameter values out of the `[String : String]` dictionary.

### Detailed design

An internal extension to the `String` type will be added to the Kitura framework that provides the capabilities described in previous sections of this proposal. Hence, this will be a non-breaking API change. A possible implementation for the extension is shown next:

```swift
extension String {

    public var int: Int? {
        return Int(self)
    }

    public var uInt: UInt? {
        return UInt(self)
    }

    public var float: Float? {
        return Float(self)
    }

    public var double: Double? {         
        return Double(self)
    }

    public var boolean: Bool? {
        return Bool(self)
    }

    public var string: String {
        get { return self }
    }

    public var intArray: [Int]? {
        let strs: [String] = self.components(separatedBy: ",")
        let ints: [Int] = strs.map { Int($0) }.filter { $0 != nil }.map { $0! }
        if ints.count == strs.count {
            return ints
        }
        return nil
    }

    public var uIntArray: [Int]? {
        let strs: [String] = self.components(separatedBy: ",")
        let uInts: [Int] = strs.map { Int($0) }.filter { $0 != nil }.map { $0! }
        if uInts.count == strs.count {
            return uInts
        }
        return nil
    }

    public var floatArray: [Float]? {
        let strs: [String] = self.components(separatedBy: ",")
        let floats: [Float] = strs.map { Float($0) }.filter { $0 != nil }.map { $0! }
        if floats.count == strs.count {
            return floats
        }
        return nil
    }

    public var doubleArray: [Double]? { 
        let strs: [String] = self.components(separatedBy: ",")
        let doubles: [Double] = strs.map { Double($0) }.filter { $0 != nil }.map { $0! }
        if doubles.count == strs.count {
            return doubles
        }
        return nil
    }

    public var stringArray: [String] {
        let strs: [String] = self.components(separatedBy: ",")
        return strs
    }

    public func decodable<T: Decodable>(_ type: T.Type) -> T? {
        guard let data = self.data(using: .utf8) else {
            return nil
        }
        let obj: T? = try? JSONDecoder().decode(type, from: data)
        return obj
     }

    public func date(_ formatter: DateFormatter) -> Date? {
        return formatter.date(from: self)
    }

    public func dateArray(_ formatter: DateFormatter) -> [Date]? {
        let strs: [String] = self.components(separatedBy: ",")
        let dates = strs.map { formatter.date(from: $0) }.filter { $0 != nil }.map { $0! }
        if dates.count == strs.count {
            return dates
        }
        return nil
    }
```

### Feedback
Feedback should be via either (or both) of the following routes:

1. The following GitHub issue:  
   https://github.com/IBM-Swift/evolution/issues/6
2. The "open-playback" channel in the Swift@IBM Slack:  
   https://swift-at-ibm-slack.mybluemix.net/

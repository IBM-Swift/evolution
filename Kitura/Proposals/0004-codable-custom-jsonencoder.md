## Custom JSONEncoder for Codable Routing
* Proposal: KIT-0004
* Authors: [Tibor Bödecs](https://github.com/tib)
* Review Manager: TBD
* Status: DRAFT
* Previous Revision: 1
* Previous Proposal: N/A

### Introduction
The current version of [Kitura](https://github.com/IBM-Swift/Kitura) always encodes codable objects using the default JSONEncoder() instance.

### Motivation
The JSONEncoder class is a powerful tool, but with the current limitation of the Kitura framework there is no chance to set custom encoding strategies (key, date, data, non-conforming float), output formating or custom user information on the encoder object. This could lead to some serious problems, for example dates decoded with the `.secondsSince1970` strategy will be encoded with incorrect values. This proposal is trying to address the issues described above.

### Proposed solution
Kitura should provide a `customJSONEncoder` block property on the `Router` class which should return a `JSONEncoder` instance. By default it can return a simple `JSONEncoder()` object, so the current functionality - with thread safety in mind - would be kept, but users would be able to return their customized `JSONEncoder` instances.

Example usage of the custom JSONEncoder:

```swift
let router = Router()

router.customJSONEncoder = {
    let encoder = JSONEncoder()
    encoder.dateEncodingStrategy = .secondsSince1970
    return encoder
}
```


### Detailed design

The `Router` class needs to be extended with a new `customJSONEncoder` block property.

```swift
public class Router {

    ///Returns the JSONEncoder instance used for encoding Codable objects
    public var customJSONEncoder: (() -> JSONEncoder) = {
        return JSONEncoder()
    }

    ///...
}
```

Inside the `CodableRouter.swift` file, all `JSONEncoder()` occurances should be replaced to `self.customJSONEncoder()`.

Implementing this solution users would be able to globally specify encoding strategies. With the help of sub-routers this strategy could be overwritten even on a single route level.

Example:

```swift
let router = Router() //with custom encoding strategy

router.customJSONEncoder = {
    let encoder = JSONEncoder()
    encoder.dateEncodingStrategy = .secondsSince1970
    return encoder
}

let subrouter = Router() //this will use the default encoding strategy

subrouter.get("/default", handler: ...)

router.get("/hello", middleware: subrouter)

Kitura.addHTTPServer(onPort: port, with: router)
Kitura.run()
```


### Alternatives considered

Another idea was to provide custom `JSONEncoder` instances using middlewares, but that would add too much complexity to the existing codebase, so it should be avoided.


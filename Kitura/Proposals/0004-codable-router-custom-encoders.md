## Custom encoders for Codable Routing

* Proposal: KIT-0004
* Authors: [Tibor Bödecs](https://github.com/tib)
* Review Manager: TBD
* Status: DRAFT
* Previous Revision: 1
* Previous Proposal: N/A


### Introduction

The current version of [Kitura](https://github.com/IBM-Swift/Kitura) always encodes / decodes codable objects using the default JSONEncoder() / JSONDecoder() instances provided by the Apple Foundation framework.


### Motivation

The built-in encoder / decoder classes are powerful tools, but with the current limitation of the Kitura framework there is no chance to use custom ones, but only JSON.

A general issue is setting custom encoding strategies (key, date, data, non-conforming float), output formating or custom user information on the encoder object. This could lead to some serious problems, for example dates decoded with the `.secondsSince1970` strategy will be encoded with incorrect values.

Note that the charset of the output is also heavily dependent on these hardcoded objects, there is no way to send out a proper JSON header with `utf-8` charset, which could be useful for content with accent characters, or non-English languages.

Another case: if the communication requires different data format (for example instead of JSON we'd like to use the PLIST file format) currently there is no option to replace or modify the default behavior, because of the hardcoded JSON encoder / decoder objects. Also if we look at the future, it'd be good for people to use their very own customized coder objects. 

This proposal is trying to address the issues described above.


### Proposed solution

* The hardcoded classes should be abstracted away with new protocols.
* Codable Router should focus mainly on routing without the coding part.
* Object conversion should be completely customizable by the Kitura end-user.
* A new delegate should provide all the encoding / decoding logic for the Router.
* Proper (ContentType, Charset, Data) values should be returned instead of hardcoded ones.
* RouterRequest object should be available for custom per-route coding logic implementations.

### Detailed design

In order to solve this issue, we need to introduce a couple new protocols

```swift
public protocol RouterEncoder {
    func encode<T>(_ value: T) throws -> Data where T: Encodable
}
public protocol RouterDecoder {
    func decode<T>(_ type: T.Type, from data: Data) throws -> T where T: Decodable
}
```

These encoder protocols will put a new level of abstraction around the currently hardcoded coder objects. So we need to extend the following objects with these protocols.

```swift
extension JSONEncoder: RouterEncoder {}
extension JSONDecoder: RouterDecoder {}
```

We'll also need some new typealiases, just for syntactic sugar reasons.

```swift
public typealias RouterEncoderBuilderBlock = () -> RouterEncoder
public typealias RouterDecoderBuilderBlock = () -> RouterDecoder
public typealias RouterResponseContent = (type: String, charset: String, data: Data)
```

Now we should remove the hardcoded values from the CodableRouter, the general idea here is to introduce a CodableRouterDelegate which defaults to JSON, because of the heavy usege in RESTful API's. First let's start with the new delegate protocol.

```swift
public protocol CodableRouterDelegate {
    var encoder: RouterEncoderBuilderBlock { get }
    var decoder: RouterDecoderBuilderBlock { get }
    func encode<T>(_ value: T, for request: RouterRequest) throws -> RouterResponseContent where T: Encodable
    func decode<T>(_ type: T.Type, from data: Data, for request: RouterRequest) throws -> T where T: Decodable
}
```

The default implementation of the delegate protocol should return the original JSONEncoded values, but users would be able to create their own delegates and provide that for the `Router` instances. 

```swift
open class JSONCodableRouterDelegate: CodableRouterDelegate {

    open var encoder: RouterEncoderBuilderBlock { return { return JSONEncoder() } }
    open var decoder: RouterDecoderBuilderBlock { return { return JSONDecoder() } }

    public init() {}

    open func encode<T>(_ value: T, for request: RouterRequest) throws -> RouterResponseContent where T: Encodable {
        let data = try self.encoder().encode(value)
        return (type: "json", charset: "utf-8", data: data)
    }

    open func decode<T>(_ type: T.Type, from data: Data, for request: RouterRequest) throws -> T where T: Decodable {
        return try self.decoder().decode(type, from: data)
    }
}

```

The `Router` class will have a new `codableDelegate` property, which defaults to the `JSONCodableRouterDelegate` instance.

```swift
public class Router {

    ///A delegate class for customizing the behavior for encoding Codable classes
    public var codableDelegate: CodableRouterDelegate = JSONCodableRouterDelegate()
    //...
}
```

Note that with this approach, we should pass the `RouterRequest` object into the delegate, because users might want to check that and provide various outputs per route.

#### Usage examples

If you want to keep the original behavior you don't have to change your source code. However if you want to customize the encoders you can go with the following options:

```swift
class MyJSONCodableRouterDelegate: JSONCodableRouterDelegate {

    override var encoder: RouterEncoderBuilderBlock {
        return {
            let encoder = JSONEncoder()
            encoder.dateEncodingStrategy = .secondsSince1970
            return encoder
        }
    }
}

let router = Router()
router.codableDelegate = MyJSONCodableRouterDelegate()
```

Custom property list router delegate

```swift 
extension PropertyListEncoder: RouterEncoder {}
extension PropertyListDecoder: RouterDecoder {}

open class PLCodableRouterDelegate: CodableRouterDelegate {

    open var encoder: RouterEncoderBuilderBlock { return { return PropertyListEncoder() } }
    open var decoder: RouterDecoderBuilderBlock { return { return PropertyListDecoder() } }

    open func encode<T>(_ value: T, for request: RouterRequest) throws -> RouterResponseContent where T: Encodable {
        let data = try self.encoder().encode(value)
        return (type: "application/x-plist", charset: "utf-8", data: data)
    }

    open func decode<T>(_ type: T.Type, from data: Data, for request: RouterRequest) throws -> T where T: Decodable {
        return try self.decoder().decode(type, from: data)
    }
}

let router = Router()
router.codableDelegate = PLCodableRouterDelegate()

```

You can even use sub-routers for custom per-route encoding.

```swift
let router = Router() //with custom encoding strategy
router.codableDelegate = MyJSONCodableRouterDelegate()
let subrouter = Router() //this will use the default encoding strategy

subrouter.get("/default", handler: ...)
router.get("/hello", middleware: subrouter)

Kitura.addHTTPServer(onPort: port, with: router)
Kitura.run()
```


### Alternatives considered

The first idea was only to modify the Codable Routing in order to support custom coding strategies, but that would also just partially solve the underlying bigger design issue of the original system. Originally I was thinking of using middlewares, but that would add too much complexity to the existing codebase, so it should be avoided.


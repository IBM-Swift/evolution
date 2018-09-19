## Codable-Response-Middleware
* Proposal: KIT-0004
* Authors: [Andrew Lees](https://github.com/Andrew-lees11)
* Review Manager: [Ian Partridge](https://github.com/ianpartridge)
* Status: DRAFT
* Implementation: Link to branch
* Previous Revision: 1

### Introduction

when you are using Codable routes in Kitura you abstract away request and response and work directly with Codable objects. We have added TypeSafeMiddleware to handle middleware like sessions and authentication on the way in however you are still unable to render pages or redirect. This makes codable routes unusable for web apps.

This proposal adds a new protocol that would change the RouterResponse using the Codable object you return. This would allow you to use Codable routes for web apps and give it the same level of functionality as Raw routes.

### Motivation

Codable routes bring a lot of benefits in simplifying a users code and adding type safety. They also allow you to use TypeSafeMiddleware along with the benefits they bring. We would like to bring templating into Codable routes so we can use these benefits for web applications.

#### Raw stencil render:

```swift
 router.add(templateEngine: StencilTemplateEngine())
 router.post("/") { request, response, next in
    let friend = try request.read(as: Friend.self)
    try response.render("Example.stencil", with: friend)
    next()
}
```
### Proposed solution

This proposal covers the addition of a new protocol called ResponseWare and an implementation for rendering using this middleware. You would then return a Swift type that conforms to ResponseWare in your codable route and it would call a handle function on itself to change the Response as required.

#### Codable stencil render:

Using a ResponseWare called RenderStencil (defined at the bottom of the page), The above code would then become:

```swift
router.post("/", handler: Renderhandler)
func Renderhandler(friend: Friend, completion: (RenderStencil?, RequestError?) -> Void ) {
    completion(RenderStencil("Example.stencil", with: friend), nil)
}
```

Here the user has specified in the handler signiture that the route will render a stencil page, constructed the object with the required fields and returned it to the route. The RenderStencil object then handles itself to render the Example.stencil page with the Friend object it was given.

### Detailed design

The ResponseWare protocol would be as follows:

```swift
public protocol ResponseWare: Codable {
    func respond(request: RouterRequest, response: RouterResponse, completion: @escaping (RequestError?) -> Void)
}
```

Inside the Codable results handler you would check if the returned type is a ResponseWare and if it is you would call it's handle function:

```swift
if let responseWare = codableOutput as? ResponseWare {
    responseWare.respond(response: response, completion: { (error) in
        if let error = error {
            response.status(HTTPStatusCode(rawValue: error.httpCode) ?? .internalServerError)
        }
    })
} else {
    // run normal codable response
}
completion()
```

This means if you return a ResponseWare you let the object handle it's own response using it's respond function.

### Redirecting

A simple responseWare example that would allow a response redirect inside a codable routes would be:

```swift
struct Redirect: ResponseWare {
    var to: String
    func respond(response: RouterResponse, completion: @escaping (RequestError?) -> Void) {
        do {
            try response.redirect(to)
            completion(nil)
        } catch {
            completion(.internalServerError)
        }
    }
}
```

This would allow a user in a Codable route to redirect by returning a `Redirect` struct in their completion.

```swift
router.get("/redirect", handler: redirectHandler)
func redirectHandler(completion: (Redirect?, RequestError?) -> Void ) {
        completion(Redirect(to: "/example"), nil)
    }
```

### Rendering

Using ResponseWare we can create a class to render in codable routes.

We make a protocol with the fields required to render:

```swift
protocol RenderEngine: ResponseWare {
    static var templateEngine: TemplateEngine { get }
    var filePath: String { get }
    var value: Data { get }
    var viewsPath: String? { get }
    init?<T: Codable>(filePath: String, with value: T, forKey key: String?, viewsPath: String?)
}
extension RenderEngine {
    public func respond(response: RouterResponse, completion: @escaping (RequestError?) -> Void) {
        guard let dict = (try? JSONSerialization.jsonObject(with: self.value, options: .allowFragments)) as? [String: Any] else {
            return completion(.unprocessableEntity)
        }
        do {
            try response.render(self.filePath, context: dict, templateEngine: Self.templateEngine, viewsPath: self.viewsPath)
        } catch {
            return completion(.unprocessableEntity)
        }
    }
}
```

The template engine plugin would then implement this protocol. This would be the stencil example:
```swift
public final class RenderStencil: RenderEngine {
    static let templateEngine: TemplateEngine = StencilTemplateEngine()

    var filePath: String

    var value: Data

    var viewsPath: String?

    init?<T: Codable>(filePath: String, with value: T, forKey key: String? = nil, viewsPath: String? = nil) {
        if key == nil {
            let mirror = Mirror(reflecting: value)
            if mirror.displayStyle == .collection || mirror.displayStyle == .set {
                return nil
            }
        }

        if let contextKey = key {
            guard let value = try? JSONEncoder().encode([contextKey: value]) else {
                return nil
            }
            self.value = value
        } else {
            guard let value = try? JSONEncoder().encode(value) else {
                return nil
            }
            self.value = value
        }
        self.filePath = filePath
        self.viewsPath = viewsPath
    }
}
```

This looks like a lot of code but it would be implemented inside Kitura.
This then gives the user the API from the previous Codable stencil render example:

```swift
router.post("/", handler: Renderhandler)
func Renderhandler(friend: Friend, completion: (RenderStencil?, RequestError?) -> Void ) {
    completion(RenderStencil("Example.stencil", with: friend), nil)
}
```

### Alternatives considered

For ResponseWare it was considered whether or not to include the RouterResponse. It is not used in our rendering or redirecting examples however a user might want to access the request headers when deciding how to response. and since the function could have access to them it seems reasonable to including it for extensibility.

For ResponseWare it was considered to not have it as Codable and overload the Router functions (get, post etc) with returning ResponseWare instead of Codable. This would remove the `if T.self is responseWare.Type` check and allow non-Codable ResponseWare but it would double the number of Router functions. It would also cause problems when an object is both Codable and a ResponseWare in deciding which path to take.

For ResponseWare it was considered to have it as extra arguments similar to TypeSafeMiddleware. This would allow you to have multiple ResponseWare registered to a route and you return the one for how you want to handle the response. This would make a lot of extra routes and could cause confusion from people returning multiple ResponseWares.

For rendering it was considered to specify the render object and route in a new struct and then pass that to the handler. This adds safety by linking the template and required object type but was seen as adding to much complexity and code for the user.

For rendering you could just use RenderEngine(without providing a template engine) and keep the current method of selecting a template engine and requiring them to the added to routes. By specifying the template engine in the route signature you guarantee the template engine you expect will be used to render the object.

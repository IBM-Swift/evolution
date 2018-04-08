## Type safety for templating
* Proposal: KIT-0006
* Authors: [David Dunn](https://github.com/ddunn2)
* Review Manager: TBD
* Status: DRAFT
* Previous Revision: 1
* Previous Proposal: N/A

### Introduction
In the latest version of [Kitura](https://github.com/IBM-Swift/Kitura) (which at the time of writing is 2.2.0) there is no way of rendering a template in a type-safe manner. This proposal seeks to implement additional API to enable type-safe templating. 

### Motivation
Currently if you want to make use of the templating in Kitura as a user you need to create a `String:Any` dictionary. The creation of this can look something like this: 

```swift
var allMeals: [String: [[String:Any]]] = ["meals" :[]]
for meal in meals {
    allMeals["meals"]?.append(["name": meal.name, "rating": meal.rating])
}
```

Then the above `allMeals` array of `String: Any` dictionaries is passed through to the render method. This current solution provides no type-safety at all and it looks very 'ugly'. The idea is to provide API to allow users to pass through a codable object which will simplify the usage and add type-safety as well. 

### Proposed solution
This proposed solution suggests to add additional `render` methods to allow the option for users to pass through a codable oject. The current solution looks like this: 
```swift 
router.get("/foodtracker") { request, response, next in
    Meal.findAll { (result: [Meal]?, error: RequestError?) in
        guard let meals = result else {
            return
        }
        var allMeals: [String: [[String:Any]]] = ["meals" :[]]
        for meal in meals {
            allMeals["meals"]?.append(["name": meal.name, "rating": meal.rating])
        }
        do {
            try response.render("FoodTemplate.stencil", context: allMeals)
        } catch let error {
            response.send(json: ["Error": error.localizedDescription])
        }
        next()
    }
}
```

Using the above example, the proposed changes would simplify the solution to the following: 

```swift 
router.get("/foodtracker") { request, response, next in
    Meal.findAll { (result: [Meal]?, error: RequestError?) in
        guard let meals = result else {
            return
        }
        do {
            try response.render("FoodTemplate.stencil", objects: meals)
        } catch let error {
            response.send(json: ["Error": error.localizedDescription])
        }
        next()
    }
}
```

### Detailed design
This proposed solution suggests adding three new methods to cover three seperate cases; passing a single codable object, passing multiple of the same codable object and passing multiple different types of codable objects. 

#### General API modification
There's a few differences between this new API and the existing API. The first being the acceptance of Codable types which is the basis for this proposal but the other is a new parameter the new API allows. 

```swift
public func render<T: Encodable>(_ resource: String, object: T..., nameOfType: String = "", options: RenderingOptions = NullRenderingOptions()) throws -> RouterResponse
```

As seen in the code above the new parameter is the `nameOfType` parameter. With these new API's an issue arose, namely that the key in the `String:Any` dictionary needs to match a variable name within your template file. We currently have no way of enforcing this nor is there an easy way to enforce this. By default the new render methods will assume that the matching variable in your template file is the lowercased, pluralised, version of the Codable type name. For example if your codable type is called `Meal` then by default the code will assume this will match a variable in your template file called `meals` and `meals` will be added as the key in the `String:Any` dictionary. This is done with the following: 
```swift
var key = String()
if nameOfType != "" {
    key = nameOfType
} else {
    key = String(describing: T.self).lowercased()
    if key.suffix(1) != "s" {
        key = key + "s"
    }
}
```

As you can see in the code above the `nameOfType` parameter allows a user to overwrite this default behaviour and use a variable name of their choice. For example if you had a Codable type called `Animal` and in your template file you had a variable for this Codable type called `cats` then you could do the following: 
```swift
response.render("FoodTemplate.stencil", object: animals, nameOfType: "cats")
```
and if you don't want to use a custom type name you can just omit the `nameOfType` parameter in the function call: 
```swift
response.render("FoodTemplate.stencil", object: animals)
```

The final difference is we handle the conversion to a `[String: [String: Any]]` type. This type is an unavoidable conversion as this is what the temple engine is expecting to receive. This implementation will be discussed in greater detail below, as this implementation varies slightly for each of the methods. 


#### Passing a single Codable object
```swift
public func render<T: Encodable>(_ resource: String, object: T..., nameOfType: String = "", options: RenderingOptions = NullRenderingOptions()) throws -> RouterResponse
```

This fuction has a variadic parameter allowing it to accept one or many values to be passed in. In our case this can be a single codable object or a comma separated list of codable objects like so: 

```swift
response.render("FoodTemplate.stencil", object: singleMeal)
response.render("FoodTemplate.stencil", object: singleMeal, anotherMeal, aFinalMeal)
```

As Swift converts a variadic parameter into an array this method just calls out to another new render function that accepts an Array of codable objects as a parameter as follows: 

```swift
public func render<T: Encodable>(_ resource: String, object: T..., nameOfType: String = "", options: RenderingOptions = NullRenderingOptions()) throws -> RouterResponse {
     return try render(resource, objects: object, options: options)
}
```

As such this implementation will be discussed in greater detail in the next section. 

#### Passing multiple Codable objects
```swift
public func render<T: Encodable>(_ resource: String, objects: [T], nameOfType: String = "", options: RenderingOptions = NullRenderingOptions()) throws -> RouterResponse {
    guard let router = getRouterThatCanRender(resource: resource) else {
        throw TemplatingError.noTemplateEngineForExtension(extension: "")
    }
    var key = String()
    if nameOfType != "" {
        key = nameOfType
    } else {
        key = String(describing: T.self).lowercased()
        if key.suffix(1) != "s" {
            key = key + "s"
        }
    }
    var values: [String: [[String: Any]]] = [:]
    try objects.forEach { item in
        guard let data = try? JSONEncoder().encode(item) else {
            Log.error("Unable to encode \(item)")
            return
        }
        guard let json = try JSONSerialization.jsonObject(with: data, options: .allowFragments) as? [String: Any] else {
            Log.error("Unable to serialise \(data) as [String: Any]")
            return
        }
        if values[key] != nil {
            values[key]?.append(json)
        } else {
            values[key] = [json]
        }
    }
    let renderedResource = try router.render(template: resource, context: values, options: options)
    return send(renderedResource)
}
```

For this method we're dealing with an array of Codable objects that we need to convert to a `String: Any` type and use the `key` variable to create an array of dictionaries: `[String: [String: Any]]`. In this instance getting the key is relatively simple, as described above the key is either assigned a default value or assigned the user provided value of the `nameOfType` param. For the `String: Any` part, we have to convert each of the Codable objects in the `objects` array. To do this I chose to use the JSONSerialization method and cast the result as `[String: Any]`. 

#### Passing multiple Codable objects of different types
```swift
public func render<T: Encodable>(_ resource: String, context: [String: [T]], options: RenderingOptions = NullRenderingOptions()) throws -> RouterResponse {
    guard let router = getRouterThatCanRender(resource: resource) else {
        throw TemplatingError.noTemplateEngineForExtension(extension: "")
    }
    let keys = Array(context.keys)
    var values: [String: [[String: Any]]] = [:]
    for key in keys {
        try context[key]?.forEach { item in
            guard let data = try? JSONEncoder().encode(item) else {
                Log.error("Unable to encode \(item)")
                return
            }
            guard let json = try JSONSerialization.jsonObject(with: data, options: .allowFragments) as? [String: Any] else {
                Log.error("Unable to serialise \(data) as [String: Any]")
                return
            }
            if values[key] != nil {
                values[key]?.append(json)
            } else {
                values[key] = [json]
            }
        }
    }
    let renderedResource = try router.render(template: resource, context: values, options: options)
    return send(renderedResource)
}
```

This method accepts a more complex type to enable the user to provide multiple codable objects of different types like so: 

```swift
try response.render("FoodTemplate.stencil", context: ["meals": meals, "users": users])
```

This method does not need the `nameOfType` parameter as the key of the context dictionary is what will be used as the key in the `[String: [String: Any]]` type. This is done by creating an array of the context keys then iterating over the keys, then each Codable object for those keys, and constructing the `[String: [String: Any]]` type needed to be passed to the template engine. Like the other method, I opted to use JSONSerialization to create the `[String: Any]` part from the Codable objects. 

### Alternatives considered
Alternatives I've considered: 

* Stripping out the default assigning of the key and forcing users to use the `nameOfType` parameter. I opted to not do this as it seems to be common behaviour to name the variable the pluralized name of your Codable type, and if a user doesn't want this behaviour they have the option to overwrite but using the default behaviour results in a simpler cleaner solution. 


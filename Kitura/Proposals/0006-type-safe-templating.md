## Type safety for templating
* Proposal: KIT-0006
* Authors: [David Dunn](https://github.com/ddunn2)
* Review Manager: TBD
* Status: DRAFT
* Previous Revision: 1
* Previous Proposal: N/A

### Introduction
In the latest version of [Kitura](https://github.com/IBM-Swift/Kitura) (which at the time of writing is 2.3.0) there is no way of rendering a template in a type-safe manner. This proposal seeks to implement additional API to enable type-safe templating. 

### Motivation
Currently in Kitura the `render()` method in the `RouterResponse` class accepts a context of type `[String: Any]`. This allows you to pass whatever you want into these methods without offering any type safety at all. This also forces a user to contruct this `[String: Any]` type. Here is an example: 

Suppose we have a `Meal` struct: 
```swift
public struct Meal: Codable {
    let name: String
    let photo: Data
    let rating: Int
}
```

- For Stencil:  
Stencil does allow a struct to be passed through as a part of the `[String: Any]` Dictionary
```swift
...
let meals = [Meal(name: "myMeal", photo: Data(), rating: 5), Meal(name: "anotherMeal", photo: Data(), rating: 2)]
let context = ["meals": meals]

response.render("myTemplate.stencil", context: context)
```

- For Mustache:  
Mustache can't render a struct, this must be converted into a `[String: Any]`
```swift
...
let meals = [Meal(name: "myMeal", photo: Data(), rating: 5), Meal(name: "anotherMeal", photo: Data(), rating: 2)]
let data = try JSONEncoder().encode(meals)
let context = try JSONSerialization.jsonObject(with: data, options: .allowFragments) as! [String: Any]

response.render("myTemplate.mustache", context: context)
```

The idea is to allow users to pass their Codable structs through to the template engines in a type safe manner.

### Proposed solution

This solution proposes a way for users to pass their Codable structs straight through to the template engine offering a simpler API to use as well as type safety.  

* Passing a single Codable struct
```swift
...
let meal = Meal(name: "myMeal", photo: Data(), rating: 5)

response.render("myTemplate.template", using: meal, forTag: "meals")
```

* Passing an array of Codable structs
```swift
...
let meals = [Meal(name: "myMeal", photo: Data(), rating: 5), Meal(name: "anotherMeal", photo: Data(), rating: 2)]

response.render("myTemplate.template", using: meals, forTag: "meals")
```

* Passing multiple Codable structs of different types
```swift
struct ContextWrapper: Codable {
    let users = [User]
    let meals = [Meal]
}

let meals = [Meal(name: "myMeal", photo: Data(), rating: 5), Meal(name: "anotherMeal", photo: Data(), rating: 2)]
let user = User(name: "Ollie")

let context = ContextWrapper(meals: meals, users: [user])

response.render("myTemplate.template", using: context)
```

### Detailed design

#### General API modification

This proposal adds a new parameter, the `forTag` param. The method signature in [KituraTemplateEngine](https://github.com/IBM-Swift/Kitura-TemplateEngine) looks like this: 
```swift
func render<T: Codable>(filePath: String, context: T, forTag: String?,
                            options: RenderingOptions, templateName: String) throws -> String
```
When working with collections in Stencil and Mustache, they require a specific tag for their logic of dealing with multiple items. For example:

Stencil: 
```
{% for meal in meals %}
    ... Do something ...
{% endfor %}
```
Mustache:
```
{{# meals}}
    ... Do something ... 
{{/ meals}}
```

The `forTag` param allows users to pass through this specific tag, which must match the tag name you use in the template files. So in the example above the tag name to match is `meals`. 

A function call would look like this: 

```swift
response.render("myTemplate.template", using: meals, forTag: "meals")
```

The `forTag` param is optional and is assigned a default value of `nil` at the top level `render()` method in the `RouterReponse` class. This is to enable ease of use of the `ContextWrapper`: 

```swift
struct ContextWrapper: Codable {
    let users = [User]
    let meals = [Meal]
}

let meals = [Meal(name: "myMeal", photo: Data(), rating: 5), Meal(name: "anotherMeal", photo: Data(), rating: 2)]
let user = User(name: "Ollie")

let context = ContextWrapper(meals: meals, users: [user])

response.render("myTemplate.template", using: context)
```

In this case the property names in the `ContextWrapper` struct are used by the template engine to match against specific tags. 

This allows users to use this API: 
```swift
func render<T: Codable>(filePath: String, context: T, forTag: String?,
                            options: RenderingOptions, templateName: String) throws -> String
```

To pass through a `ContextWrapper` or a single Codable struct. For the latter they could use the `forTag` param for populating the template correctly. For example: 

```swift
...
let meal = Meal(name: "myMeal", photo: Data(), rating: 5)

response.render("myTemplate.template", using: meal, forTag: "meals")
```

#### Passing a single Codable type



#### Passing an array of Codable types

#### Passing multiple different Codable types

#### Passing an array of tuples of type (Identifier, Codable)

### Alternatives considered

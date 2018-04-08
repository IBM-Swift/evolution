## Type safety for templating
* Proposal: KIT-0006
* Authors: [David Dunn](https://github.com/ddunn2) & [Steven Van Impe](https://github.com/svanimpe)
* Review Manager: [Lloyd Roseblade](https://github.com/lroseblade)
* Status: DRAFT
* Previous Revision: 1
* Previous Proposal: N/A

### Introduction
In the latest version of [Kitura](https://github.com/IBM-Swift/Kitura) (which at the time of writing is 2.3.0) there is no way of rendering a template in a type-safe manner. This proposal seeks to implement additional API to enable type-safe templating. 

### Motivation
Currently in Kitura the `render()` method in the `RouterResponse` class accepts a context of type `[String: Any]`. This allows you to pass whatever you want into this method without offering any type safety at all. This also forces a user to contruct this `[String: Any]` type. Here is an example: 

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
let meal = Meal(name: "myMeal", photo: Data(), rating: 5)
let context = ["meal": meal]

response.render("myTemplate.stencil", context: context)
```

- For Mustache:  
Mustache can't render a struct, this must be converted into a `[String: Any]`
```swift
...
let meal = Meal(name: "myMeal", photo: Data(), rating: 5)
let data = try JSONEncoder().encode(meal)
let context = try JSONSerialization.jsonObject(with: data, options: .allowFragments) as! [String: Any]

response.render("myTemplate.mustache", context: context)
```
The idea is to allow users to pass their Codable models through to a template engine.

### Proposed solution

This solution proposes a way for users to pass their Codable structs straight through to the template engine offering a simpler API to use as well as type safety.  

### Detailed design

#### General API modification

This involved adding a new method to the [KituraTemplateEngine](https://github.com/IBM-Swift/Kitura-TemplateEngine) protocol: 
```swift
func render<T: Encodable>(filePath: String, with: T, forKey: String?,
                        options: RenderingOptions, templateName: String) throws -> String
                        
```

This has an implementation for each template engine:

- Kitura-StencilTemplateEngine 
```swift
public func render<T: Encodable>(filePath: String, with value: T, forKey: String?,
                                   options: RenderingOptions, templateName: String) throws -> String {
    if rootPaths.isEmpty {
        throw StencilTemplateEngineError.rootPathsEmpty
    }
    
    let loader = FileSystemLoader(paths: rootPaths)
    let environment = Environment(loader: loader, extensions: [`extension`])
    
    if let contextKey = key {
        return try environment.renderTemplate(name: templateName, context: [contextKey: value])
    }
    let contextDict = DictionaryEncoder().encode(value)
    return try environment.renderTemplate(name: templateName, context: contextDict)

}
```
- Kitura-MustacheTemplateEngine
```swift
public func render<T: Codable>(filePath: String, with value: T, forKey key: String?,
                                   options: RenderingOptions, templateName: String) throws -> String {
    let template = try Template(path: filePath)

    let data = try JSONEncoder().encode(value)
    let json = try JSONSerialization.jsonObject(with: data, options: .allowFragments) as! [String: Any]

    if let contextKey = key {
        return try template.render(with: Box([contextKey: json]))
    }
    return try template.render(with: Box(json))
}
```

This new API supports four new cases which will be discussed in greater detail over the next sections. In each case we follow similar steps to the current API. That is we have a top layer public render method in `RouterResponse` which checks we have a template engine registered. Then it calls out to an internal render method in `Router`:
```swift
@discardableResult
public func render<T: Encodable>(_ resource: String, with value: T, forKey key: String? = nil,
                   options: RenderingOptions = NullRenderingOptions()) throws -> RouterResponse {
    guard let router = getRouterThatCanRender(resource: resource) else {
        throw TemplatingError.noTemplateEngineForExtension(extension: "")
    }
    let renderedResource = try router.render(template: resource, with: value, forKey: key, options: options)
    return send(renderedResource)
}
```
The internal render method in `Router` checks we've provided an extension (or used a default template engine which may provide a default extension). It also gets the correct template engine for the template file the user provides. Then builds the absolute path to the template file and then calls the render method provided by whichever template engine will be rendering this file: 
```swift
internal func render<T: Encodable>(template: String, with value: T, forKey key: String?,
                         options: RenderingOptions = NullRenderingOptions()) throws -> String {
    let (optionalFileExtension, resourceWithExtension) = calculateExtension(template: template)
    // extension is nil (not the empty string), this should not happen
    guard let fileExtension = optionalFileExtension else {
        throw TemplatingError.noTemplateEngineForExtension(extension: "")
    }

    guard let templateEngine = getTemplateEngine(template: template) else {
        if fileExtension.isEmpty {
            throw TemplatingError.noDefaultTemplateEngineAndNoExtensionSpecified
        }

        throw TemplatingError.noTemplateEngineForExtension(extension: fileExtension)
    }

    let filePath: String
    if let decodedResourceExtension = resourceWithExtension.removingPercentEncoding {
        filePath = viewsPath + decodedResourceExtension
    } else {
        Log.warning("Unable to decode url \(resourceWithExtension)")
        filePath = viewsPath + resourceWithExtension
    }

    let absoluteFilePath = StaticFileServer.ResourcePathHandler.getAbsolutePath(for: filePath)
    return try templateEngine.render(filePath: absoluteFilePath, with: value, forKey: key, options: options, templateName: resourceWithExtension)
}
```

This proposal also introduces a new parameter, `forKey: String?`. This allows users to provide a value for their template file variables. The value is then used as the key in the dictionary that's passed to the template engine. The template engine uses that key to match the correct values to the correct variable in the template file. For example if a template file looks like this: 
```
{% for meal in meals %}
Meal name: {{ meal.name }}
{% endfor %}
```

And make a call to the render method:
```swift
response.render("myTemplate.template", with: meals, forKey: "meals")
```
Assuming the Codable values `meals` have a name property then it'll be used to generate the content for the above template file. Certain values passed in need a `forKey`, for example an Array `[Meal(...)]` needs a key. Error handling has been added for the case a key is not provided when passing an array. 

#### Passing a single Codable value

This covers the case where you know you only want to render one model, for example displaying the currently logged in user's name on a welcome screen.  
Example of template for single value (same for both Stencil and Mustache):
```
User's name: {{ name }}
```
```swift
struct User: Codable {
    let name: String
}

let user = getCurrentUser()

response.render("myTemplate.template", with: user)
```

#### Passing an array of Codable values

This covers the case where you know you want to render multiple Codable values, for example displaying all the meal ratings on a webpage.  
Example of Stencil template for multiple values:
```
{% for meal in meals %}
Meal name: {{ meal.name }}
{% endfor %}
```
Example of Mustache template for multiple values:
```
{{# meals}}
- Meal name: {{ meal.name }}.
{{/ meals}}
```


```swift
struct Meal: Codable {
    let name: String
    let rating: Int
}

let meals: [Meal] = getAllMeals()

response.render("myTemplate.template", with: meals, forKey: "meals")
```
or 

```swift
struct Meal: Codable {
    let name: String
    let rating: Int
}

let meals: [Meal] = getAllMeals()

response.render("myTemplate.template", with: ["meals": meals])
```
#### Passing multiple different Codable values
This covers the case where you know you want to render multiple Codable values of different types. Like displaying all the meal ratings for a particular user on a webpage. It uses the same API used for rendering a single Codable value.  
Example of Stencil template for multiple different values:
```
Logged in as: {{ user.name }}

{% for meal in meals %}
Meal name: {{ meal.name }}
{% endfor %}
```
Example of Mustache template for multiple different values:
```
User's name: {{ user.name }}

{{# meals}}
- Meal name: {{ meal.name }}.
{{/ meals}}
```

```swift
struct User: Codable {
    let name: String
}
struct Meal: Codable {
    let name: String
    let rating: Int
}

struct CodableWrapper: Codable {
    let meals: [Meal]
    let user: User
}

let meals: [Meal] = getAllMeals()
let user = getCurrentUser()
let context = CodableWrapper(meals: meals, user: user)
response.render("myTemplate.template", with: context)
```

#### Passing an array of tuples of type (Identifier, Codable)
This adds support for SwiftKueryORM's `findAll()` method that responds with an array of tuples. For this we have an aditional piece of logic in the toplayer render method in the `RouterResponse` class. The logic breaks up the array of tuples into an array of Codables and then passes this array into the 'passing an array of Codable models' method discussed above. 

```swift
public func render<I: Identifier, T: Encodable>(_ resource: String, with values: [(I, T)], forKey key: String? = nil,
                                   options: RenderingOptions = NullRenderingOptions()) throws -> RouterResponse {
    guard let router = getRouterThatCanRender(resource: resource) else {
        throw TemplatingError.noTemplateEngineForExtension(extension: "")
    }
    var items: [T] = []
    values.forEach { value in
        items.append(value.1)
    }
    let renderedResource = try router.render(template: resource, with: items, forKey: key, options: options)
    return send(renderedResource)
}
```

### Alternatives Considered
* Splitting the new API into several different implementations: 
```swift
func render<T: Codable>(filePath: String, with: T,
                options: RenderingOptions, templateName: String) throws -> String

func render<T: Codable>(filePath: String, with: [T], forKey: String?,
                options: RenderingOptions, templateName: String) throws -> String
                
```
However is really isn't needed, given `[T]` is an Array of types conforming to Codable and therefore is itself a Codable type `T`. So it was possible to squash these methods into the one in the actual proposal. 


### Further Work
* Adding a new utility to allow Codable models to be generated from template files  
This could possibly be a command line tool that would parse a template file, or files. It would look at the variables within the template files and generate Codable model representations of them. They would be stored in a Models directory. I haven't looked to much into this, but was a suggestion raised and something worth considering. 

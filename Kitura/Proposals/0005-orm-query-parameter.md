## Query Parameters for the ORM
* Proposal: KIT-0005
* Authors: [Enrique Lacal](https://github.com/EnriqueL8)
* Review Manager: TBD
* Status: DRAFT
* Previous Revision: 1
* Previous Proposal: N/A

### Introduction
In the latest version of [Kitura](https://github.com/IBM-Swift/Kitura) (which at the time of writing is 2.2.0) integrated Query Parameters discussed in KIT-0003. This proposal seeks to enable the usage of Query Parameters in [Swift-Kuery-ORM](http://github.com/IBM-Swift/Swift-Kuery-ORM) to filter on database queries in a type-safe manner.

### Motivation
Query Parameters are a set of filters passed when a client makes a request.For example, a GET request to get a list of users where the users are from the US:

`GET http://localhost:8080/users?country=US`

Currently passing those query parameters into database queries is quite painful. The idea is to accept the value from the Query Parameters as an argument in an ORM call and map them to a `WHERE` clause in SQL


### Proposed solution
This proposal suggests to modify the currenty Api in Swift-Kuery-ORM to make it accept a `QueryParams` instance such as:

```[swift]
struct User: Model {
  var name: String
  var age: Int
  var country: String
}

struct Query: QueryParams {
  var country: String
}

router.get("/users", handler: getUsers)

func getUsers(query: Query, completion: @escaping ([User]?, RequestError?) -> Void) -> Void {
  User.findAll(matching: query, completion)
}
```

### Detailed design

#### Modifying the API
In order to use the QueryParameters in the ORM, the ORM calls need to accept a `QueryParams` instance. There are couple of ways of doing this, this could be one:

```[swift]
public func findAll<Q: QueryParams>(matching: Q, _ completion: @escaping ([Self]?,RequestError?) -> Void) -> Void
```

#### Encoding the values
In order to pass the QueryParameters values to a `WHERE` clause in SQL, we need to add a new encode method in the encoder for QueryParameters which returns a dictionary [String: Any]. This is due to the fact that the current encode returns a dictionary [String: String] where the values are casted to Strings which makes sense for URLs but not for SQL queries. Something along these lines, following [this](https://github.com/IBM-Swift/KituraContracts/blob/master/Sources/KituraContracts/CodableQuery/QueryEncoder.swift#L69): 

```[swift]
public func encode<T: Encodable>(_ value: T) throws -> [String: Any] {
  // Case statement  
}
```

#### Creating SQL statements with the values

Once we have the dictionary [String: Any], we can map this down to a `WHERE` clause in a `SELECT` or `DELETE` statement. At this stage we have to take in consideration the logic from the URL, such as:

`GET http://localhost:8080/users?country=US`

would be:

```[swift]
Select(from: Users).where(country == "US")
```

which maps down to:

```[SQL]
SELECT FROM users WHERE country = 'US';
```

If we have multiple distinct keys, we could have an `AND`:

`GET http://localhost:8080/users?name=John&country=US`

would be:

```[swift]
Select(from: Users).where(name = "John" && country == "US")
```

which maps down to:

```[SQL]
SELECT FROM Users WHERE name = 'John' AND country = 'US';
```

If we have duplicate keys, we could have an `OR`:

`GET http://localhost:8080/users?country=UK&country=US`

would be:

```[swift]
Select(from: Users).where(country = "UK" || country == "US")
```

which maps down to:

```[SQL]
SELECT FROM Users WHERE country = 'UK' OR country = 'US';
```


### Alternatives considered
Describe alternative approaches to addressing the same problem, and why you chose this approach instead.

An alternative to the ORM Api could be adding a `Filter` function:

```[swift]
func getUsers(query: Query, completion: @escaping ([User]?, RequestError?) -> Void) -> Void {
  User.findAll(completion).filter(on: query)
}
```


### Further Work

At the moment, QueryParameters lack some features in filtering. Two of the key features are **Ranges** and **Pagination**.

- Ranges: 
  - Investigating how ranges are encoded into URL and if there is a guide to follow?
  - Possiblity following the `key=value` syntax could be `gt=value` for greater than
  - How to integrate this with the ORM?


- Pagination: 
  - Making a decision on how to approach pagination, again looking at a guide to follow?
  - Integration with ORM






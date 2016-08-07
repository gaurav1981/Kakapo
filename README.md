#Kakapo ![partyparrot](http://cultofthepartyparrot.com/sirocco.gif)
[![Language: Swift](https://img.shields.io/badge/lang-Swift-yellow.svg?style=flat)](https://developer.apple.com/swift/)
[![Build Status](https://travis-ci.org/devlucky/Kakapo.svg?branch=master)](https://travis-ci.org/devlucky/Kakapo)
[![Version](https://img.shields.io/cocoapods/v/Kakapo.svg?style=flat)](http://cocoapods.org/pods/Kakapo)
[![DocCov](https://img.shields.io/cocoapods/metrics/doc-percent/Kakapo.svg)](http://cocoadocs.org/docsets/Kakapo)
[![codecov](https://codecov.io/gh/devlucky/Kakapo/branch/master/graph/badge.svg)](https://codecov.io/gh/devlucky/Kakapo)
[![codebeat badge](https://codebeat.co/badges/69a42ece-740c-4a29-b25a-598deaf61fca)](https://codebeat.co/projects/github-com-devlucky-kakapo)
[![License](https://img.shields.io/cocoapods/l/Kakapo.svg?style=flat)](http://cocoapods.org/pods/Kakapo)
[![Platform](https://img.shields.io/cocoapods/p/Kakapo.svg?style=flat)](http://cocoapods.org/pods/Kakapo)

> Dynamically Mock server behaviors and responses.

## Contents
- [Why Kakapo?](#why-kakapo)
- [Features](#features)
- [Installation](#installation)
- [Usage](#usage)
  - [Serializable protocol](#serializable-protocol)
  - [Router: Register and Intercept](#router---register-and-intercept)  
    - [Third-party network libraries](#third-party-libraries)
  - [Leverage the database - Dynamic mocking](#leverage-the-database---dynamic-mocking)
  - [CustomSerializable](#customserializable)
  - [JSONAPI](#jsonapi)
  - [Expanding Null values with Property Policy](#expanding-null-values-with-property-policy)
  - [Key customization - Serialization Transformer](#key-customization---serialization-transformer)
  - [Full responses on ResponseFieldsProvider](full-responses-on-responsefieldsprovider)
- [Roadmap](#roadmap)
- [Examples](#examples)
- [Authors](#authors)

Kakapo is a dynamic mocking library. It allows you to replicate your backend APIs and logic.  
With Kakapo you can easily prototype your application based on your API specifications.

## Why Kakapo?

A common approach when testing network requests is to stub them with fake network responses from local files or recorded requests. This has some disadvantages:

- All files needs to be updated when the API are updated.
- Lot of files have to be generated and included in the project.
- Are just static responses that can only be used for unit tests since they don't reflect backend behaviors and state.

While still this approach may work good, Kakapo will be a game changer in your network tests: it will give you complete control when it comes to simulate backend behaviors. Moreover, is not just unit testing since you can even take a step further and prototype your application before having a real service behind!

## Features

  * Dynamic mocking
  * Prototyping
  * Swift 2.2 compatible
  * Compatible with [![Platform](https://img.shields.io/cocoapods/p/Kakapo.svg?style=flat)](http://cocoapods.org/pods/Kakapo)
  * Protocol oriented and pluggable
  * Fully customizable by defining custom serialization and custom responses
  * Out-of-the-box serialization
  * [JSONAPI](http://jsonapi.org) support

## Installation

Using [CocoaPods](http://cocoapods.org/):

```ruby
use_frameworks!
pod 'Kakapo'
```

## Usage

Kakapo is made with an easy-to-use design in mind. To quickly get started, you can create a `Router` that intercepts network requests like this:

```Swift
let router = Router.register("http://www.test.com")
router.get("/users"){ request in
  return ["id" : 2, "name": "Kakapo"]
}
```

You might be wondering where the dynamic part is; here is when the different modules of Kakapo take place:

```Swift
let db = KakapoDB()
db.create(User.self, number: 20)

router.get("/users"){ request in
  return db.findAll(User.self)
}
```

Now, we've created 20 random `User` objects and mocked our request to return them.  

Let's get a closer look to the different features:

### Serializable protocol

Kakapo uses the `Serializable` protocol in order to serialize objects to JSON. *Any type* can be serialized as long as it conforms to this protocol:

```Swift
struct User: Serializable {
  let name: String
}

let user = User(name: "Alex")
let serializedUser = user.serialize()
print(serializedUser["name"]) // Alex
```

Also, standard library types are supported : This means that `Array`, `Dictionary` or `Optional` can be serialized:

```Swift
let serializedUserArray = [user].serialize()
// -> [["name": "Alex"]]
let serializedUserDictionary = ["test": user].serialize()
// -> ["test": ["name": "Alex"]]
```

### Router - Register and Intercept

Kakapo uses `Router`s in order to keep track of the registered endpoints that have to be intercepted.  
You can match *any* relative path from the registered base URL, as long as the components are matching the request's components. You can use wildcard components:

```Swift
let router = Router.register("http://www.test.com")

// Will match http://www.test.com/users/28
router.get("/users/:id") { ... }

// Will match http://www.test.com/users/28/comments/123
router.get("/users/:id/comments/:comment_id") { ... }
```

The handler will have to return a `Serializable` object that will define the response once the URL of a request is matched.
When a `Router` intercepts a request, it  automatically serialize the `Serializable` object returned by the handler and converts it to `Data`.

```Swift
router.get("/users/:id") { request in
  return ["id": request.components["id"]!, "name": "Joan"]
}
```

Now everything is ready to test your mocked API; you can perform your request as you usually would do:

```Swift
session.dataTaskWithURL(NSURL(string: "http://www.test.com/users/1")!) { (data, _, _) in
  // handle response
}.resume()
```

> Note: query parameters are not affecting the route match
> `http://www.test.com/users/1?foo=bar` would also be matched 

In the previous example the handler was returning a simple `Dictionary`, while this works because it is `Serializable` you can also create your own entities that conform to `Serializable`:

```Swift
struct User: Storable, Serializable {
    let firstName: String
    let lastName: String
    let id: String
}

router.get("/users/:id") { request in
  return User(firstName: "Joan", lastName: "Romano", id: request.components["id"]!)
}
```

When a request is matched the RouteHandler receives a `Request` object that represents your request including components, query parmaters, HTTPBody and HTTPHeaders. The `Request` object can be usefull when building dynamic repsonses.

#### Third-Party Libraries

Third-Party libraries that use the Foundation networking APIs are also supported but you might need to set a proper `NSURLSessionConfiguration`. For example to setup `Alamofire`:

```swift
 let configuration = NSURLSessionConfiguration.defaultSessionConfiguration()
 configuration.protocolClasses = [KakapoServer.self]
 let manager = Manager(configuration: configuration)
```

### Leverage the database - Dynamic mocking

Kakapo gets even more powerful when using your Routers together with the Database. You can create, insert, remove, update or find objects.

This lets you mock the APIs behaviors, as if you were using a real backend. This is the **dynamic** side of Kakapo.

To create entities that can be used with the databsse, your types need to conform to the `Storable` protocol.

```Swift
struct Articles: Storable, Serializable {
    let id: String
    let text: String

    init(id: String, db: KakapoDB) {
        self.id = id
        self.text = randomString() // you might use some faker library like Fakery!
    }
}
```

An example usage could be to retreive a specific `Article`:

```Swift
let db = KakapoDB()
db.create(Article.self, number: 20)

router.get("/articles/:id") { request in
  let articleId = request.components["id"]
  return db.find(Article.self, id: articleId)
}
```

But, of course, you could perform any logic which fits your needs:

```Swift
router.post("/article/:id") { request in
  return db.insert { (id) -> User in
    return Article(text: "text from body", id: id)
  }
}

router.del("/article/:id") { request in
  let articleId = request.components["id"]
  let article = db.find(Article.self, id: articleId)
  db.delete(article)

  return ["status": "success"]
}
```

### CustomSerializable

In [Serializable](#serializable-protocol) we described how your classes can be serialized. The serialization, by default, `Mirror` (using swift's reflection) an entity recursively serialize its properties.

Whenever a different behavior is needed, you can instead conform to `CustomSerializable` to provide your custom serialization.

For instance, `Array` uses `CustomSerializable` to return an `Array` containing its serialized elements. `Dictionary`, similarly, is serialized by creating a `Dictionary` with the same keys and serialized values.

For other examples of `CustomSerializable` and how to use it to create more complex serializations take a look at the `JSONAPISerializer` implementation.

### JSONAPI

Kakapo was built with JSONAPI support in mind, `JSONAPISerializer` is able to serialize your entity into JSON conforming to [jsonapi.org](http://jsonapi.org).

Your entities, to be serialized conforming to JSONAPI, need to conform to `JSONAPIEntity` protocol.  

Let's see an example:

```Swift
struct Cat: JSONAPIEntity {
    let id: String
    let name: String
}

struct User: JSONAPIEntity {
    let id: String
    let name: String
    let cats: [Cat]
}
```

Note that `JSONAPIEntity` objects are already `Serializable` and you could just use them together with your Routers. However, to completely follow the JSONAPI structure in your responses, you should wrap them into a `JSONAPISerializer` struct:

```Swift
router.get("/users/:id"){ request in
  let cats = [Cat(id: "33", name: "Joan"), Cat(id: "44", name: "Hez")]
  let user = User(id: "11", name: "Alex", cats: cats)
  return JSONAPISerializer(user)
}
```

### Expanding Null values with Property Policy

When serializing to JSON, you may want to represent a property value as `null`. For this, you can use the `PropertyPolicy` enum. It is similar to `Optional` except it provides an additional case `.Null`:

```Swift
public enum PropertyPolicy<Wrapped>: CustomSerializable {
    case None
    case Null
    case Some(Wrapped)
}
```

It's only purpose is to be serialized in 3 different ways to cover all possible behaviors of an Optional property.
`PropertyPolicy` works exaclty as `Optional` properties:
- `.None` -> property not included in the serialization
- `.Some(wrapped)` -> serialize `wrapped`

The additional case `.Null` instead, is serialized as `null` when converted to json.

```Swift
PropertyPolicy<Int>.None.serialize() // nil
PropertyPolicy<Int>.Null.serialize() // NSNull
PropertyPolicy<Int>.Some(1).serialize() // 1
```

### Key customization - Serialization Transformer

The keys of the JSON generated by the serialization are directly reflecting the property names of your entities. However you might need different behaviors. For example many APIs use `snake_case` keys but almost everyone use `camelCase` properties in Swift.  
To transform the keys you can use `SerializationTransformer`. Objects conforming to this protocol are able to transform the keys of a wrapped object at serialization time.  

For a concrete implementation, check `SnakecaseTransformer`: a struct that implements SerializationTransformer to convert keys into snake case:

```Swift
let user = User(userName: "Alex")
let serialized = SnakecaseTransformer(user).serialize()
print(serialized) // [ "user_name" : "Alex" ]
```

### Customize responses with ResponseFieldsProvider

If your responses need to specify status code (which will be 200 by default) and/or header fields, you can take advantage of `ResponseFieldsProvider` to customize your responses.

Kakapo provides a default `ResponseFieldsProvider` implementation in the Response struct, which you can use to wrap your Serializable objects:

```Swift
router.get("/users/:id"){ request in
    return Response(statusCode: 400, body: ["id" : 2], headerFields: ["access_token" : "094850348502"])
}

session.dataTaskWithURL(NSURL(string: "http://www.test.com/users/2")!) { (data, response, _) in
    let allHeaders = response.allHeaderFields
    let statusCode = response.statusCode
    print(allHeaders["access_token"]) // 094850348502
    print(statusCode) // 400
    }.resume()
```

Otherwise your `Serializable` object can directly implement the protocol, take a look at `JSONAPIError` to see another example.

## Roadmap

Kakapo is ready to use, is not meant to be shipped to the app store but you can also do it! In fact you might see it in action in some Apple stores since it was used to mock some features of Runtastic's demo app; however it's at its early stage and we would love to ear your thoughts. We encourage you to open an issue if you have any questions,  feedbacks or you want to propose new features.

- Full JSON API support [#67](https://github.com/devlucky/Kakapo/issues/67)
- Reverse and Recursive relationships [#16](https://github.com/devlucky/Kakapo/issues/16)
- Custom Serializers for common json specifications

## Examples

### Newsfeed

Make sure you check the [demo app](https://github.com/devlucky/Kakapo/tree/feature/READMEDocumentation/Examples/NewsFeed) we created using Kakapo: a prototyped newsfeed app which lets the user create new posts and like/unlike them.

![](https://raw.githubusercontent.com/devlucky/Kakapo/feature/READMEDocumentation/Examples/NewsFeed/newsfeed.png)

## Authors

[@MP0w](https://github.com/MP0w) - [@zzarcon](https://github.com/zzarcon) - [@joanromano](https://github.com/joanromano)

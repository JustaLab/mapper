


# Mapper

Mapper is a simple Swift library to convert JSON to strongly typed
objects. One advantage to Mapper over some other libraries is you can
have immutable properties.

## Fork modifications

This fork includes few modifications to the original project:
- [Custom Operator](#custom-operator)
- [Default Convertibles](#default-convertibles)
- [Arrays](#arrays)
- [Dates](#dates)
- [Initialization](#initialization)


## Installation

#### With [CocoaPods](http://cocoapods.org/)

```ruby
use_frameworks!

pod "StreemMapper"
```

#### With [Carthage](https://github.com/Carthage/Carthage)

```
github "lyft/mapper"
```

## Usage

### Simple example:

```swift
import Mapper
// Conform to the Mappable protocol
struct User: Mappable {
  let id: String
  let photoURL: NSURL?

  // Implement this initializer
  init(map: Mapper) throws {
    try id = map.from("id")
    photoURL = map.optionalFrom("avatar_url")
  }
}

// Create a user!
let JSON: NSDictionary = ...
let user = User.from(JSON) // This is a 'User?'
```

### Using with enums:

```swift
enum UserType: String {
  case Normal = "normal"
  case Admin = "admin"
}

struct User: Mappable {
  let id: String
  let type: UserType

  init(map: Mapper) throws {
    try id = map.from("id")
    try type = map.from("user_type")
  }
}
```

### Nested `Mappable` objects:

```swift
struct User: Mappable {
  let id: String
  let name: String

  init(map: Mapper) throws {
    try id = map.from("id")
    try name = map.from("name")
  }
}

struct Group: Mappable {
  let id: String
  let users: [User]

  init(map: Mapper) throws {
    try id = map.from("id")
    users = map.optionalFrom("users") ?? []
  }
}
```

### Use `Convertible` to transparently convert other types from JSON:

```swift
extension CLLocationCoordinate2D: Convertible {
  static func fromMap(value: AnyObject?) throws -> CLLocationCoordinate2D {
    guard let location = value as? NSDictionary,
      let latitude = location["lat"] as? Double,
      let longitude = location["lng"] as? Double else
      {
         throw MapperError.ConvertibleError(value: value, type: [String: Double].self)
      }

      return CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
  }
}

struct Place: Mappable {
  let name: String
  let location: CLLocationCoordinate2D

  init(map: Mapper) throws {
    try name = map.from("name")
    try location = map.from("location")
  }
}

let JSON = [
  "name": "Lyft HQ",
  "location": [
    "lat": 37.7603392,
    "lng": -122.41267249999999,
  ],
]

let place = Place.from(JSON)
```

### Custom Transformations

```swift
private func extractFirstName(object: AnyObject?) throws -> String {
  guard let fullName = object as? String else {
    throw MapperError.ConvertibleError(value: object, type: String.self)
  }

  let parts = fullName.characters.split { $0 == " " }.map(String.init)
  if let firstName = parts.first {
    return firstName
  }

  throw MapperError.CustomError(field: nil, message: "Couldn't split the string!")
}

struct User: Mappable {
  let firstName: String

  init(map: Mapper) throws {
    try firstName = map.from("name", transformation: extractFirstName)
  }
}
```

### Parse nested or entire objects

```swift
struct User: Mappable {
  let name: String
  let JSON: AnyObject

  init(map: Mapper) throws {
    // Access the 'first' key nested in a 'name' dictionary
    try name = map.from("name.first")
    // Access the original JSON (maybe for use with a transformation)
    try JSON = map.from("")
  }
}
```
See the docstrings and tests for more information and examples.

## Custom Operator

This fork includes a custom operator that allows you to parse the Mapper object super cleanly:

```swift
import Mapper
// Conform to the Mappable protocol
struct User: Mappable {
  let id: String
  let photoURL: NSURL?

  // Implement this initializer
  init(map: Mapper) throws {
    try id   = map |> "id"
    photoURL = map |> "avatar_url"
  }
}

```

## Default Convertibles

Many convertibles are now supported in this fork, here is the list of all supported types, in bold are thoses added.

- String
- Int
- UInt
- **Int8**
- **UInt8**
- **Int16**
- **UInt16**
- **Int32**
- **UInt32**
- **Int64**
- **UInt64**
- Float
- Double
- Bool
- **NSNumber**
- NSDictionary
- NSArray
- NSURL
- **NSDate** from a timestamp

## Arrays

For consistency reasons, an Array is now initialized just like a object.

```swift
let users = [User].from(JSON) // Returns a [User]?
```

Also when parsing an array, the original library doesn't allow you to have a malformed object within the JSON array. It will just return nil. This fork allows the creation of an array and will just omit the malformed objects.

```swift
struct User: Mappable {
  let name: String
  init(map: Mapper) throws {
    try self.name = map.from("name")
  }
}
let JSON = [["name": "John"], ["firstname": "Bob"]]
let users = [User].from(JSON) // Returns  1 User object with name John
```
If no object can be parsed within the JSON array, this will return an empty array. If the JSON is not array, it will return nil.

Finally, this fork allows you to parse an optional array of enums, this is especially useful, if you're unsure about the JSON you receive and prefer to return nil or an empty array instead of throwing an exception.

```swift
enum DaysOfWeek {
    case Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday
}

struct User: Mappable {
  let name: String
  let availableDays: [DaysOfWeek]?
  init(map: Mapper) throws {
    try self.name = map |> "name"
    availableDays = map |> "available_days" //returns nil if  key is not there
  }
}
```

## Dates

Dates can now be parsed using a formatter or a timestamp:

```
struct User: Mappable {
  let date1: Date
  let date2: Date?
  init(map: Mapper) throws {
    try date1 = map |> ("date1", "MMMM dd, yyyy h:mm:ss a zzz")  // will try to parse as a string with the given format
    date2     = map |> "date2" //will try to parse it as a timestamp
  }
}
```

## Initialization

When creating an object there is no need any more to parse the initial object into a Dictionary or Array. The library takes care of it and will return nil if an array is passed instead of a dictionary and vice versa.

For instance:
```swift
let JSON:AnyObject = ...
let user = User.from(JSON)    // Returns nil if JSON is not dictionary
let users = [User].from(JSON) // Returns nil if JSON is not an array
```

Also, because of type inference over the optional type, methods **optionalFrom** and **from** have been merged into the single method **from**.


## Open Radars

These radars have affected the current implementation of Mapper

- [rdar://23376350](http://www.openradar.me/radar?id=5669622346940416)
  Protocol extensions with initializers do not work in extensions
- [rdar://23358609](http://www.openradar.me/radar?id=4926300410085376)
  Protocol extensions with initializers do not play well with classes
- [rdar://23226135](http://www.openradar.me/radar?id=5066210983018496)
  Can't conform to protocols with similar generic function signatures
- [rdar://23147654](http://www.openradar.me/radar?id=4991483920777216)
  Generic functions are not differentiated by their ability to throw
- [rdar://23695200](http://www.openradar.me/radar?id=5060084212170752)
  Using the `??` operator many times is unsustainable.
- [rdar://23697280](http://www.openradar.me/radar?id=5049833333194752)
  Lazy collection elements can be evaluated multiple times.
- [rdar://23718307](http://www.openradar.me/radar?id=4926133845884928)
  Non final class with protocol extensions returning `Self` don't work

## License

Mapper is maintained by [Lyft](https://www.lyft.com/) and released under
the Apache 2.0 license. See LICENSE for details

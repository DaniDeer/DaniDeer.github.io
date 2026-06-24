---
title: The Why and How of using Codecs (and why it matters for data integrity and maintainability)
published: 2026-06-18
updated: 2026-06-18
icon: lucide/code
tags:
  - Codecs
  - Codecs Basics
  - Data Integrity
  - Data Validation
  - Maintainability
---

# The Why and How of using Codecs (and why it matters for data integrity and maintainability)

*Published: {{ page.meta.published }} · Updated: {{ page.meta.updated }}*

When writing software that interacts with external systems, we often encode and decode data that comes and goes to/from the wire, like (e.g. JSON, XML, CSV, etc.) into some structs or objects in the ecosystem of your choice.

## The Problem

Usually it starts like this (because it is just a small PoC - which basically means it is PROD and you are the owner/responsible for it for the next 10+ years):

1. Define some struct that represents the data you want to encode/decode.

   ```go
   type User struct {
       Name string `json:"name"` // struct tags for encoding/decoding
       Age  int    `json:"age"`
   }
   ```

2. You use the idiomatic way of your ecosystem to encode/decode the data, e.g. in Go you would use the builtin [`encoding/json` package][enc] with `struct tags`:
[enc]: https://pkg.go.dev/encoding/json

    ```go
    func decodeUser(data []byte) (User, error) {
        var u User
        return u, json.Unmarshal(data, &u) // no validation
    }
    ```

3. You write validation logic somewhere in your codebase to ensure that the data you decode is valid and meets the requirements of your more or less defined domain model:

    ```go
    func validateUser(u User) error {
        if u.Name == "" {
            return errors.New("name: must not be empty")
        }
        if u.Age <= 0 {
            return errors.New("age: must be positive")
        }
        return nil
    }
    ```

This is only one data object and a simple example, but for your PoC you define a couple of them scattered around your codebase and you may write validation logic for the _same_ fields or even data objects in different places multiple times. And you have to make sure to call the validation logic every time you decode data, otherwise you might end up with invalid data in your system that can cause all sorts of problems down the line (e.g. bugs, security vulnerabilities, etc.).

Of course I assume you are a very accurate and disciplined software engineer and have some architecture or pattern to deal with this, otherwise you might end up with a lot of duplicated, scattered code that is hard to maintain. And since your service is a PoC, which we already know is PROD the moment you show your solution to management, this service will evolve over time and you will add more data models or change existing ones and you will have to change the validation logic for them as well.

In the moment another team asks, if there is an `OpenAPI`/`AsyncAPI` spec for API, you create a backlog item to add it, but you never get around to do it, because you have to work on the next feature or bug fix. So consuming your API is a pain for other teams. And even if you write the spec by hand, you add another layer of maintenance you have to consider when making changes.

The real problem is  not serialization. The real problem is that model definition, validation rules, and API documentation live in different places and inevitably drift apart. So is there a way to improve this? Yes, there is. You can use codecs!

!!! tip "What is your approach?"
    Provide an issue in [this :cat: repository][repo] if you have a different approach or solution, or you work in an ecosystem that deals with this problem in a different way. I am curious about it.

  [repo]: https://github.com/DaniDeer/DaniDeer.github.io

## The Solution

A `Codec` is the single source of truth for a data model. It bundles three concerns in one place:

- The definition of the data model (e.g. struct or object).
- The encoding/decoding logic (e.g. how to encode/decode the data to/from the wire format which could be JSON, YAML, XML, CSV, etc.).
- The validation logic or constraints (e.g. how to validate the data after decoding, like non-empty strings, positive numbers, enums, etc.).
- _Bonus_: The generation of the `OpenAPI`/`AsyncAPI` spec from the codec including all the validation rules, so you don't have to maintain it separately.

Bundling these concerns together allows you to encode, decode, validate, and document your data model from a single definition and serializing/deserializing it to/from every format the codec supports!.

Let´s make this clear with these short diagrams:

```TEXT
Traditional approach

Struct
  ↓
JSON Tags

Validation
  ↓
OpenAPI Spec

(3 sources of truth)


Codec approach

Codec
  ├─ Model
  ├─ Validation
  ├─ Encoding
  └─ OpenAPI Generation

(1 source of truth)

```

??? note "Who or what inspired me to embark on this journey? :rocket:"

    One of our senior devs sparked my creativeness by pitching the `Haskell` [autodocodec] lib and the idea of how codecs can be a useful tool to solve the problem of data integrity and maintainability in our codebases. 
    
    Also rising discussions in our organization around the topic of data quality and data products as a foundation for using AI with our manufacturing data, made me realize that we need to have a better way to define and validate our data models, and codecs can be a great solution for that. 
    
    What comes with those discussions are buzzwords like "data governance", "data ownership", "data quality", "data products", "data mesh", etc. but at the end of the day, if you don't have a good way to define and validate your data models, you will end up with a lot of technical debt and a lot of problems down the line. 
    
    The standard response from a legacy manufacturer when these topics come up: We write down central directives and define business roles and responsibilities, before we even know how we can technically implement this, or if we are even capable of implementing this with our current systems and landscape and expertise. 
    
    Another pattern is to work around the root cause: We have a data quality problem, but we don´t fix it at the data source (our machines), and hope we can fix it somewhere in a Data Lake/Warehouse."

    This entire section could be an article in its own right, but in my blog I’d actually like to focus on what really matters at the end of the day: working software patterns in software solutions. So I started to research and experiment and write even my own codec library in Go [go-codex][go-codex]. Currently in a very early stage, and just a PoC, but you already knwo what _PoC_ means...

    [autodocodec]: https://github.com/NorfairKing/autodocodec
    [go-codex]: https://github.com/DaniDeer/go-codex

In `GO` the workflow for a codec could look like this: 

1. Define a data model and its validation rules in one place:

    ```go
    type User struct {
        Name string
        Age  int
        EyeColor string
    }

    var UserCodec = codec.Struct[User](
      codec.RequiredField("name", codec.String().Refine(validate.NonEmptyString)),
      codec.RequiredField("age", codec.Int().Refine(validate.PositiveInt)),
      codec.OptionalField("eye_color", codec.String().Refine(validate.Enum("blue", "green", "brown"))),
    )
    ```

2. Encode/Decode and validate in one step:

    ```go
    data, err := UserCodec.Encode(user) // user is a User struct
    user, err := UserCodec.Decode(data) // data is []byte from the wire (e.g. JSON)
    ```

3. _Bonus_: Derive a schema for the data model (e.g. JSON schema):

    ```go
    schemaJSON, _ := json.MarshalIndent(UserCodec.Schema, "", "  ")
    // e.g:
    // {
    //   "type": "object",
    //   "properties": {
    //     "name": {
    //       "type": "string",
    //       "validation": "non-empty"
    //     },
    //     "age": {
    //       "type": "integer",
    //       "validation": "positive"
    //     },
    //     "eye_color": {
    //       "type": "string",
    //       "validation": "enum: blue, green, brown"
    //     }
    //   },
    //   "required": ["name", "age"]
    // }
    ```

With this approach, now you are able to change e.g. `Name` to `DisplayName` and you only have to change `codec.RequiredField("display_name", codec.String().Refine(validate.NonEmptyString)),`. Or you want to make `EyeColor` required and/or add more colors to the enum. The struct tag, the validator, and the schema all update automatically - nothing to forget.

### But... Why not just use `JSON schema` or `OpenAPI`?

A `JSON schema` or `OpenAPI` spec defines the contract only. A codec defines the contract AND _is executable_. A codec can validate data, encode/decode data, and generate `JSON schema`.

### Why not just use `constructors` or `factory functions`?

In `GO`, you can use constructors or factory functions `NewUser(name string, age int, eyeColor string)` to create instances of your structs and validate the data in the constructor. But this approach has some limitations:

- You have to write a constructor for every struct you want to validate, which can be tedious and error-prone.
- You have to call the constructor every time you want to create an instance of the struct, which can be cumbersome and lead to inconsistencies if you forget to call it.

Codecs are commplementary to constructors. Constructorrs protect data created inside your application, while codecs protect data crossing your application boundary (e.g. from the wire, from a database, from another service, etc.).

## Codec Convenience and Enhancements

Let's play a little "Make a Wish" game and think about what else a codec library could offer to help us as developers writing production grade code.

### Builtin Constraints

The codec library can provide a set of builtin constraints that you can use to define your validation rules, like basic `NonEmptyString`, `PositiveInt`, `Enum` but also `Email`, `Time` `URL`, `UUID`, `IPv4`, `IPv6`, `MQTTTopic`, etc. This can save you a lot of time and effort to write your own validation logic for common constraints and also ensure consistency across your codebase.

### Cross-Field Constraints

There are cases, where you need to validate accross fields, like a time picker that has a `start_time` and an `end_time` field, and you want to validate that `start_time` is before `end_time`.

With a codec, the definition of a cross-field constraints could look like this:

```go
var TimeRangeCodec = codec.Struct[TimeRange](
  codec.RequiredField("start_time", codec.Time()),
  codec.RequiredField("end_time", codec.Time()),
  ).RefineFunc(func(tr TimeRange) error {
    if !tr.EndTime.After(tr.StartTime) {
      return errors.New("end_time must be after start_time")
    }
    return nil
  })
```

You define the field constraints first and then you can add a cross-field constraint with `RefineFunc` that takes the whole struct as input and returns an error if the constraint is not met. 

### Validation without Encode/Decode

Sometimes you have domain boundaries within your service, that receives data not only from the wire, but from a module within your codebase, and you want to validate the data at your boundary before using it.

With a codec, you can use the `Validate` method to validate data without encoding/decoding it:

```go
  // Validate — explicit round-trip check
  if err := UserCodec.Validate(u); err != nil {
      return fmt.Errorf("constructed invalid user: %w", err)
  }
```

### Multiple Codecs for the same data model

In some cases, you might have different representations of the same data model for different use cases, e.g. a `User` struct that has a `Password` field that you want to encode/decode when receiving data from the wire, but you don't want to include it when encoding data to send to other services or when storing it in a database.

With codecs, you can define multiple codecs for the same data model with different fields and validation rules:

```go
var UserInputCodec = codec.Struct[User](
    codec.RequiredField("name", codec.String().Refine(validate.NonEmptyString)),
    codec.RequiredField("age", codec.Int().Refine(validate.PositiveInt)),
    codec.RequiredField("password", codec.String().Refine(validate.NonEmptyString)),
  )
var UserOutputCodec = codec.Struct[User](
  codec.RequiredField("name", codec.String().Refine(validate.NonEmptyString)),
  codec.RequiredField("age", codec.Int().Refine(validate.PositiveInt)),
)
```

### Codec Composition

You can compose codecs to create more complex data models from simpler ones, e.g. you have a `User` codec and you want to create a `Group` codec that has a list of users:

```go
type Group struct {
    Name string
    Users []User
}

var GroupCodec = codec.Struct[Group](
    codec.RequiredField("name", codec.String().Refine(validate.NonEmptyString)),
    codec.RequiredField("users", codec.List(UserCodec)), 
)
```

### Codec Inheritance

You can also create new codecs that inherit from existing ones and add or override fields and validation rules, e.g. you have a `User` codec and you want to create an `AdminUser` codec that has an additional `Role` field:

```go
type AdminUser struct {
    User // embed User struct to inherit its fields and validation rules
    Role string
}

var AdminUserCodec = codec.Struct[AdminUser](
    codec.RequiredField("role", codec.String().Refine(validate.NonEmptyString)),
).Inherit(UserCodec) // inherit fields and validation rules from UserCodec
```

### Builtin Metrics, Logging, and Tracing

The codec library can also provide builtin metrics, logging, and tracing for encoding/decoding/validation operations, so you can easily monitor the performance and reliability of your codecs and identify any issues or bottlenecks. For example, you can have a metric that counts the number of successful and failed decodings, or a log that shows the input data and the validation errors if any. 

Maybe using `GO`s `interfaces` and the `observer` pattern to allow users to implement their own custom metrics, logging, and tracing logic and plug it into the codec library?

These are just a few ideas on how to enhance the codec concept.

!!! tip "Inspired? You have an idea for enhancements?"
    Provide an issue in [this :cat: repository][repo] if you have an idea or suggestion. I am curious about it.

  [repo]: https://github.com/DaniDeer/DaniDeer.github.io


## Summary

At the beginning of my current role, we tried to standardize process data for various machines and vendors. But this standardization never really took off. We tried to maintain a separate documentation (manually) as a cumbersome and fiddly raw JSON schema that just lives as a reference instead of a real implementation. It was outdated the moment someone gather all the requirements and start writing it, and maintaining this was a mess. So few developers followed it (often because we enforce it on the machine side), and we ended up supporting a lot on how a valid process message should look like. And even as a maintainer of this standard, it was hard to keep track and tell developers what is valid according to schema. Of course we implemented our own data pipeline services, that had to follow the same schema on our side, so we also felt this pain.

With a Codec, it would have been much easier to define the data models and validation rules directly in the codebase together with the domain experts and generate the specs from it. Evolving and maintaining the data models and validation rules would have been much easier and more maintainable, and we could have focused more on the actual implementation of the services instead.

Admittedly, defining codecs in `GO` would require that all your services and systems use `GO` so that you can define codecs as contracts and provide them as `go packages` for other teams. This is not the case in the manufacturing landscape with many legacy systems and primarily the use of embedded hardware like PLCs or `C`/`C++` embedded controls. But as a team defining the machine interfaces and standards, codecs can help you to define, validate, and maintain your data models and contracts in a more efficient and reliable way, and also provide a better experience for other teams consuming your APIs. And it is always better than writing raw `JSON schemas` by hand.

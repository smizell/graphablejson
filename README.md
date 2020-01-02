# Graphable JSON Specification

Graphable JSON is a specification that defines a contract between a client and a server in order to reduce or eliminate common breaking changes. The goal is to allow APIs to evolve independently by weaking the coupling between clients and servers.

## Overview

Graphable JSON aims to reduce breaking changes and lower the barrier to entry for APIs. Some of the more common breaking changes happen when:

1. A value needs to change from a single value to an array of values
1. An object needs to become a separate resource that MAY be linked
1. A list of resources needs to become a paginated collection
1. A value needs to change its type

To handle these changes, Graphable JSON has the following tenets.

1. A property defines a relationship between an entity and a value
1. A value MAY be a literal value or a link
1. A value MAY be a collection or a link to a collection

### Property as relationship

In many API, a property in a JSON object denotes structure—it shows how an object has a property of a given value. There are tools for validating this specific structure that create strong coupling between the client and server. This creates rigidity and prevents the object from evolving organically over time.

Rather than structure, Graphable JSON denotes a property as a relationship to a stream of values. This means the stream MAY be:

1. Empty because the property is undefined
1. Empty because the property is null
1. A stream of one value if the literal value is a object, string, number, or boolean
1. A stream of one or more values if the literal value is an array

Treating the result as a stream of values allows for the relationship to evolve. It also reduced the errors that can happen from properties being undefined or null.

Below are examples of how these properties are handled as relationships that result in streams of data.

#### Literal value is undefined

Given object:

```js
{}
```

When property `email` is requested,

Then the result is:

```js
[]
```

#### Literal value is null

Given the object:

```js
{
  "email": null
}
```

When property `email` is requested by the client,

Then the result is:

```js
[]
```

#### Literal value is object, string, number, or boolean

Given the object:

```js
{
  "email": "jdoe@example.com"
}
```

When property `email` is requested by the client,

Then the result is:

```js
["jdoe@example.com"]
```

#### Literal value is an array

Given the object:

```js
{
  "email": ["jdoe@example.com", "jane.doe@example.com"]
}
```

When property `email` is requested by the client,

Then the result is:

```js
["jdoe@example.com", "jane.doe@example.com"]
```

### Linked Value

The first tenant of Graphable JSON allowed for the property to evolve in the context of the object. However, there is a need many times to convert an object into its own resource in the API. This can be achieved by converting the literal value into a link.

Links use the [RESTful JSON](https://restfuljson.org) pattern. The properties for links denote relationship, so the literal values for the link properties MAY be a single URI or an array of URIs.

#### From literal value to link

Given the object:

```js
{
  "address": {
    "street": "123 Main St.",
    "city": "New York",
    "state": "New York",
    "zipCode": "10101"
  }
}
```

When the `address` property is converted to a link:

```js
{
  "address_url": "http://example.com/address/444"
}
```

And when it's moved to its own resource,

Then the client developer should get a stream of one address when the `address` relationship is specified.

#### From a single link to many links

Given the object:

```js
{
  "address_url": "http://example.com/address/444"
}
```

When the property is converted to a list of links:


```js
{
  "address_url": [
    "http://example.com/address/444",
    "http://example.com/address/723"
  ]
}
```

Then the client developer should get a stream of two addresses when the `address` relationship is specified.

### Collection Resource

It's common for an API to include collections for resources. These collections can be paginated and filtered based on the individual API needs.

A collection resources is denoted by an object with a `profile` links set to `https://github.com/smizell/graphablejson/wiki/Collection` (this is subject to change later). When that value is found, the object MUST be treated as a collection.

A collection allows for the `item` relationship to denote a stream of items for the collection.

A collection allows for these relationships for pagination:

1. `next` denotes the next representation for the collection
1. `prev` denotes the previous representation for the collection

#### Converting array to collection

Given the object:

```js
{
  "address": [
    {
      "street": "123 Main St.",
      "city": "New York",
      "state": "New York",
      "zipCode": "10101"
    },
    {
      "street": "343 Elm St.",
      "city": "New York",
      "state": "New York",
      "zipCode": "10101"
    }
  ]
}
```

When the `address` property is converted a collection,

Then the result would be:

```js
{
  "address": {
    "profile_url": "https://github.com/smizell/graphablejson/wiki/Collection",
    "url": "http://example.com/addresses/",
    "item": [
      {
        "street": "123 Main St.",
        "city": "New York",
        "state": "New York",
        "zipCode": "10101"
      },
      {
        "street": "343 Elm St.",
        "city": "New York",
        "state": "New York",
        "zipCode": "10101"
      }
    ]
  }
}
```

The `address` property can also be converted into a link and its value moved to another resource

```js
{
  "address_url": "http://example.com/addresses/"
}
```

A client should continue to be able to specify the relationship `address` and get the same result whether it's a single address, array of addresses, a link, a collection, or a link to a collection. It will always be:

```js
[
  {
    "street": "123 Main St.",
    "city": "New York",
    "state": "New York",
    "zipCode": "10101"
  },
  {
    "street": "343 Elm St.",
    "city": "New York",
    "state": "New York",
    "zipCode": "10101"
  }
]
```

#### Paginated Collection

A collection MAY be paginated. 

```js
{
  "profile_url": "https://github.com/smizell/graphablejson/wiki/Collection",
  "url": "http://example.com/addresses/?page=3",
  "item": [
      {
        "street": "123 Main St.",
        "city": "New York",
        "state": "New York",
        "zipCode": "10101"
      },
      {
        "street": "343 Elm St.",
        "city": "New York",
        "state": "New York",
        "zipCode": "10101"
      }
  ],
  "next_url": "http://example.com/addresses/?page=4"
  "prev_url": "http://example.com/addresses/?page=2"
}
```

The stream of values should be opaque to the client developer. They should be able to iterate over the values whether they are in an array locally or spread across many pages of the collection.

#### Collection of links

Collections can have the literal values included in `item` or they can have links to the items. The above collection might look like this:

```js
{
  "profile_url": "https://github.com/smizell/graphablejson/wiki/Collection",
  "url": "http://example.com/addresses/",
  "item_url": [
    "http://example.com/address/444",
    "http://example.com/address/723"
  ],
  "next_url": "http://example.com/addresses/?page=2"
}
```

Since the `item` value is a relationship, it takes on the same characteristics as any other type of value. Therefore, it can be `item`, a link, or an array of links.

### Changing Types (Experimental)

Warning: this section is experimental. The others have been developed and tested, but this section has not.

Properties themselves MAY be versioned in order to allow them to change over time. This means that an API can have multiple versions of a property living alongside other versions.

Versioning an API at the global level means that each version of an API must live alongside every previous version until the older versions are removed. The scope of the version is for the entire API. Versioning at the property level can reduce the scope considerably and allow resources to evolve easier.

Versions are denoted by appending `__<version>` to the property name.

#### Converting from a string to object

Given the object:

```js
{
  "address": "123 Main St., New York, New York 10101"
}
```

When the `address` property is moved to the next version and changed to an object,

```js
{
  "address": "123 Main St., New York, New York 10101"
  "address_v2": {
    "street": "123 Main St.",
    "city": "New York",
    "state": "New York",
    "zipCode": "10101"
  }
}
```

Then the client developer should be able to specify the version for `address` and get the correct value.

Once the original version is deprecated (note, this specification does not handle deprecation), the `address` property can be converted and the `v2` can be removed.

```js
{
  "address": {
    "street": "123 Main St.",
    "city": "New York",
    "state": "New York",
    "zipCode": "10101"
  }
}
```

### Evolving the API

As shown, Graphable JSON allows for the properties to evolve over time. Properties can evolve from a single value to an array of values to links and collections. They can also evolve to different types.

If the client becomes a [tolerant reader](https://martinfowler.com/bliki/TolerantReader.html), the client can be protected from many changes that break clients today.

There are also other areas to consider to protect against breaking changes.

1. Clients SHOULD NOT be coded to specific HTTP status codes. They should look for successes, redirects, and failures. Client developers should not have to write code that specifies status codes unless necessary.
1. Clients SHOULD NOT be coded to specific HTTP methods. Those methods should be derived from a contract or—better yet—from forms found in the API responses.
1. Clients SHOULD NOT build URLs, but rather rely on the server to include relevant links in the responses.

Following rules like these can allow the client to get all of the benefits of something like GraphQL without using GraphQL itself. In other words, the client developer can specify what they want from the API and the client itself can go out and get the necessary data—whether it's in a single response or many responses.

This brings up an important point. Starting out, an API _could_ be a single response. This could allow API designers to think only about the vocabulary and relationships of an API and break things out later into resources with individual URLs. Clients shouldn't break when those changes happen.

## About this document

This document was authored by Stephen Mizell in an effort to lower the barrier to entry for implementing and consuming APIs. This document is licensed under the MIT license. The requirements here conform to RFC 2119.

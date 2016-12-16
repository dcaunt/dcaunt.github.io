---
layout: post
title:  "Himotoki Tutorial"
date:   2016-07-14 19:00:00 +0000
categories: swift ios
---
[Himotoki](https://github.com/ikesyo/Himotoki) is a simple yet powerful library for decoding JSON. From the project page:

> Himotoki (紐解き) is a type-safe JSON decoding library purely written in Swift. This library is highly inspired by popular JSON parsing libraries in Swift: Argo and ObjectMapper.

This article provides a brief tutorial and Xcode Playground to help learn how to use it. You can clone the Playground from [GitHub here](https://github.com/dcaunt/himotoki-playground).

# Basic Decoding

As you can see from the Himotoki README, creating a model and specifying how to decode it from JSON is fairly straightforward:
{% highlight swift %}
struct Group: Decodable {
    let name: String
    let floor: Int
    let locationName: String
    let optional: [String]?

    // MARK: Decodable

    static func decode(e: Extractor) throws -> Group {
        return try Group(
            name: e <| "name",
            floor: e <| "floor",
            locationName: e <| [ "location", "name" ], // Parse nested objects
            optional: e <||? "optional" // Parse optional arrays of values
        )
    }
}
{% endhighlight %}

Himotoki can decode any generic type `T`  conforming to its `Decodable` protocol using the operators below. We’ll see how to implement Decodable in our own code later.

| Operator                        | Decode element as | Remarks                          |
|:--------------------------------|:------------------|:---------------------------------|
| <code>&lt;&#124;</code>         | `T`               | A value                          |
| <code>&lt;&#124;?</code>        | `T?`              | An optional value                |
| <code>&lt;&#124;&#124;</code>   | `[T]`             | An array of values               |
| <code>&lt;&#124;&#124;?</code>  | `[T]?`            | An optional array of values      |
| <code>&lt;&#124;-&#124;</code>  | `[String: T]`     | A dictionary of values           |
| <code>&lt;&#124;-&#124;?</code> | `[String: T]?`    | An optional dictionary of values |

# Advanced Decoding

Let’s assume we have the following JSON representing a series of musical bands:
{% highlight json %}
{
  "objects": [
    {
      "name": "Beatles",
      "homepage": "http://www.thebeatles.com/",
      "active": "inactive",
      "members": [
        {
          "name": "George Harrison",
          "birth_date": "1943-02-25"
        },
        {
          "name": "John Lennon",
          "birth_date": "1940-10-09"
        },
        {
          "name": "Paul McCartney",
          "birth_date": "1942-06-18"
        },
        {
          "name": "Ringo Starr",
          "birth_date": "1940-07-07"
        }
      ]
    },
    {
      "name": "The Rolling Stones",
      "active": "active",
      "members": [
        {
          "name": "Charlie Watts",
          "birth_date": "1941-06-02"
        },
        {
          "name": "Keith Richards",
          "birth_date": "1943-12-18"
        },
        {
          "name": "Mick Jagger",
          "birth_date": "1943-07-26"
        },
        {
          "name": "Ronnie Wood",
          "birth_date": "1947-06-01"
        }
      ]
    }
  ],
  "last_updated": "2016-05-17T22:43:04.000000+00:00"
}
{% endhighlight %}

This JSON is straightforward, yet interesting enough to give our model several requirements. There are a couple of different date formats (`last_updated` and `birth_date`), and a field which can be represented as an enumerated type (`active`). The optional `homepage` property should really be represented as an `NSURL`. Finally, this structure represents a generic collection of `objects`, so it would be nice to re-use that collection for other types.

### Models

We define the following `struct` types to model our data.

{% highlight swift %}
struct BandResponse {
    let objects: [Band]
    let lastUpdated: NSDate?
}

struct Band {
    enum Status: String {
        case Active = "active"
        case Hiatus = "hiatus"
        case Disbanded = "inactive"
    }

    let name: String
    let members: [BandMember]
    let homepageURL: NSURL?
    let status: Status
}

struct BandMember {
    let name: String
    let birthDate: NSDate
}
{% endhighlight %}

### Decodable

Through the power of generics, Himotoki can automatically decode extracted values from JSON to their correct types. To add support for `NSURL`, we define an Extension:


{% highlight swift %}
extension NSURL: Decodable {
    public static func decode(e: Extractor) throws -> Self {
        let rawValue = try String.decode(e)

        guard let result = self.init(string: rawValue) else {
            throw DecodeError.Custom("Error parsing URL from string")
        }

        return result
    }
}
{% endhighlight %}

We also need to implement Decodable for our own types, `Band`, `BandMember` and `BandResponse`:

{% highlight swift %}
extension BandResponse: Decodable {
    static func decode(e: Extractor) throws -> BandResponse {
        return try BandResponse(objects: e <|| "objects",
            lastUpdated: DateTimeTransformer.apply(e <|? "last_updated")
        )
    }
}

extension BandMember: Decodable {
    static func decode(e: Extractor) throws -> BandMember {
        return try BandMember(name: e <| "name",
            birthDate: DateTransformer.apply(e <| "birth_date")
        )
    }
}

extension Band: Decodable {
    static func decode(e: Extractor) throws -> Band {
        return try Band(name: e <| "name",
            members: e <|| "members",
            homepageURL: e <|? "homepage",
            status: e <| "active"
        )
    }
}

extension Band.Status: Decodable {}
{% endhighlight %}

Note that Himotoki provides a protocol extension for `RawRepresentable` types, so conforming to `Decodable` is enough as enums with a `RawValue` conform to this protocol.

Finally, we implement two transformers for decoding the two date formats:


{% highlight swift %}
public let DateTransformer = Transformer<String, NSDate> { dateString throws -> NSDate in
    let dateFormatter = NSDateFormatter()
    dateFormatter.locale = NSLocale(localeIdentifier: "en_US_POSIX")
    dateFormatter.dateFormat = "yyyy-MM-dd"
    dateFormatter.timeZone = NSTimeZone(abbreviation: "UTC")

    if let date = dateFormatter.dateFromString(dateString) {
        return date
    }

    throw customError("Invalid date string: \(dateString)")
}

public let DateTimeTransformer = Transformer<String, NSDate> { dateString throws -> NSDate in
    let dateFormatter = NSDateFormatter()
    dateFormatter.locale = NSLocale(localeIdentifier: "en_US_POSIX")
    dateFormatter.dateFormat = "yyyy-MM-dd'T'HH:mm:ss.SSSZ'"
    dateFormatter.timeZone = NSTimeZone(abbreviation: "UTC")

    if let date = dateFormatter.dateFromString(dateString) {
        return date
    }

    throw customError("Invalid datetime string: \(dateString)")
}
{% endhighlight %}

These only vary in the `dateFormat` provided to the `NSDateFormatter`, so we could probably factor this out.

If all date representations in JSON have the same format, we can define an `Extension` on `NSDate` to implement `Decodable`, just as we did with `NSURL`.

Given the above, we can now decode some JSON:

{% highlight swift %}
do {
    let response = try BandResponse.decodeValue(bandJSON)
    print(response?.objects.first?.name) // Beatles
} catch {
    print(error)
}
{% endhighlight %}

If the JSON is valid, you’ll have a `BandResponse` representation, and if not, `error` will contain information on the first failure.

### A generic Response wrapper

We can improve upon our BandResponse struct by using generics. We define a `struct` `Response` which is generic over `Decodable`:

{% highlight swift %}
struct Response<T: Decodable> {
    let objects: [T]
    let lastUpdated: NSDate?
}
{% endhighlight %}
and implement `Decodable`:

{% highlight swift %}
extension Response: Decodable {
    static func decode<T: Decodable>(e: Extractor) throws -> Response<T> {
        return try Response<T>(objects: e <|| "objects",
            lastUpdated: DateTimeTransformer.apply(e <|? "last_updated")
        )
    }
}
{% endhighlight %}

Now we can decode the JSON to our generic collection:

{% highlight swift %}
do {
    let response = try Response<Band>.decodeValue(bandJSON)
	print(response?.objects.first?.name) // Beatles
} catch {
    print(error)
}
{% endhighlight %}

And we’re done!

Whether you’re building an app or an API wrapper, I recommend trying Himotoki for your next project. You can even clone my Playground from [GitHub](https://github.com/dcaunt/himotoki-playground) and have a play with your own models and JSON.

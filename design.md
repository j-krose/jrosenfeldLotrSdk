# Goals

My SDK attempts to expose to the user the set of functionality of the API while abstracting the REST layer itself from the user's view.  The following design decisions support this goal:

1. The rest layer is represented as a subpackage `rest`, mostly contained in the [rest.go](rest/rest.go) file.  Due to Go's package management scheme, the end-user does not have direct access to this functionality.
1. Data types are created to represent each object returned by the API, so that the user can take advantage of strong typing and not have to deal with JSON directly
1. Two functions are used to represent each route, one for batch queries and one for id queries expecting a single return value; e.g. `GetChapters()` vs `GetChapter(id)`. Separating these into two different functions allows the singular function to check that it has one reponse, and hand it back to the user as an object rather than handing back an array of objects that the caller would need to check and retrieve from the array.
1. Options such as filtering are represented as objects, rather than requiring the user to format the filtering url parameters themselves

# Implementation Details

I decided to use a single `Sdk` object to represent the SDK.  This object has two functions: it holds the api key and exposes the querying functions. Since the SDK requires so little data to operate, all of the SDK functions _could_ be represented as static functions that take the api key as a parameter, but I felt that using an object the represent the SDK was cleaner. This could also allow for improvements in the future such as caching build into the sdk itself.

When the user calls into a function of the `Sdk` object, the SDK in turn makes a static call into the `rest` package to execute its calls.  Various helpers functions and abstractions are used to make each `Sdk` call require a minimal amount of boilerplate, with as much generic code as possible living in the `rest` functionality, rather than the `Sdk` itself.

I exposed all the basic routes of the API, as well as their query-by-id counterparts (e.g. `/book` and `/book/{id}`), but did not expose the compound routes (e.g. `/book/{id}/chapter`) due to the time constaint.  I believe that my design is well suited to handle these compond routes, since the `rest` layer is generic enough to fill in any object as the result of any call, so it would be as simple as adding a few additional lines of boilerplate in [sdk.go](sdk.go) to expose these additional routes.

As a proof of concept, I have implemented the API's filtering mechanism using a generic `UrlParameter` interface which can be implemented in various ways by the SDK.  The current set of exposed implementations can be found in [filterOption.go](filterOption.go).  I did not expose objects for pagination, sorting, or advanced filtering, but I believe my design is easily extensible to handle these types of parameters.  The `rest.UrlParameter` interface in [urlParameter.go](rest/urlParameter.go) is extensible to any options represented by a URL parameter, and the SDK can easily implement new objects to handle different kinds of URL parameters.

The `Quote` and `Chapter` objects reference other objects by way of their id.  To represent a more comprehensive data format, I additionally created `FullQuote` and `FullChapter` objects that represent a join of the additional data onto the data object.  For example `Quote` contains simply a `MovieId` string, whereas this is replaced by a fully-filled `Move` object in the `FullQuote`.  Callers are given two ways to access this `Full` data.  They can either fill an existing object by calling the appropriate `Sdk.Fill...` method, or they can get the filled data from the start with an `Sdk.GetFull...` method.  For example, to get a full quote, the user can either call `Sdk.GetQuote(id)` followed by `Sdk.FillQuote(quote)`, or they can simply call `Sdk.GetFullQuote(id)`.

To prevent abuse of the API, batched `Full` functions (such as `GetFullQuotes`) use a local cache from id->data (lifecycled to the call in question) to prevent unnecessary redundant calls to the API.  I chose not to extend the lifecycle of these caches because I thought it might break the contract of an SDK being a lightweight, stateless, object.  If I were to extend the lifecycle of this cache, I would like to impose certain size limitations, as well as consider whether the data in the system may change over time and need to be invalidated after a certain perioud of time on the client.
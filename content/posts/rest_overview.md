---
title: "Study Notes 2: REST"
date: 2023-11-14T22:27:33Z
draft: true
---

An Application Programming Interface (API) is a user interface to data and systems that is consumed by applications rather than humans. It establishes a well defined contract between the consumer and the API provider.

With remote procedure calls (RPCs), clients can invoke procedures or functions from a remote server as if it were a local function. The objective of RPC is to make remote calls as simple as local calls.

A REST API is an RPC mechanism that follows the REST design principles, where REST stands for Representational State Transfer.

## Why REST?

Common set of design principles that all REST APIs must follow. As a result, the developer of an API doesn't have to start from scratch, he can simply adopt these principles which makes it easy to develop APIs.

Best practices for building and managing REST APIs.

REST APIs are not tied to any specific data format.

The simplicity of building REST APIs makes it very attractive and there is no standard, simply guidelines, which means it is very flexible.

Usually built with JSON (JavaScript Object Notation) because it is simple, meant for human consumption and natively supported in web platforms.

## REST API Concepts

All real life objects or resources may be describe by the way of attributes. These attributes are managed usually via some persistent storage. The representational state of resources is managed in the database.

## Types of API

An API provider is an organisation that exposes APIs. API consumers are entities that consume these APIs. There are several types of API consumers:
* Private or internal consumers - part of the organisation that is building the API
* Public or external consumer - outside of the provider organisation
* Partner API consumer - these are basically trusted consumers

With the types of consumers we can define a similar set of types of APIs
* Private or internal
* Public or external
* Partner

Note that there is no difference in implementation between the different types of APIs, the main difference is in how these are managed.

What API capabilities are available to each type of consumer is purely a business decision. Usually the amount of features offered follows the following logic: private consumers have access to the most features, partner consumers have access to a subset of the features of private consumers, and public consumers have access to the smallest feature set.

The main considerations between the different types of APIs are related to API security, access request, documentation, and SLA management. How we implement these depends on the type of consumers we are targeting.

### API Security

* Trusted developers - basic authentication and proprietary schemes
* Untrusted deverlopers - key/secret, OAuth

### Documentation

* Private and partner - internal websites, PDFs
* Public - developer portal

Arguably, you should always have API docs in a developer portal, even if they are simply an API for internal consumption.

### Access request

* Private and Partner - emails, internal ticketing or some other similar process
* Public - developer portal

### SLA Management

This requires defining several SLA tiers and assigning each tier to a given type of consumer/API.


## API Value Chain

An API makes the asset of value easily accessible within and outside the bounds of the organisation. Business wise, this has the direct benefit of creating a new revenue stream via exposing the API. Indirectly, it also benefits the business side of life by, for example, empowering the partner ecosystem or increasing brand recognition and awareness.

The end consumers of the asset can be categorized into two types:
* Individual users - public domain apps, browsers, app stores;
* Enterprise users - SAAS solutions, packaged solutions, home grown applications.

Applications are just a way of delivering the value-added asset to the end consumers. Along the API value chain, we have the following:
* Asset - the asset itself;
* API developer - must understand the end costumer's needs, as well as the needs of the app developers and must support the application development process;
* API - exposes the asset;
* App developer - must understand end customer's needs, build apps that give value and enrich the asset via customer experience, and should contribute to enhancing the API via feedback
  
In essence, before building an API, we can ask ourselves the following questions:
* What asset fo value does your organisation hold?
* Who will get the most benefit from the asset?
* Is there a business value in making the asset easily accessible?
* Are there partners who will help in delivering the value?

## REST Architecture Constraints

For an architecture to be RESTful, it must follow six design rules, which are referred to as the REST architecture constraints:
* Client-server
* Uniform interface
* Statelessness
* Caching
* Layered
* Code on demand

### Client Server 

REST applications should have a client-server architecture. 

* Client - requests the resource;
* Server - serves the resource as a response, may serve multiple resources, serves multiple clients.

The client and server should not run in the same process space. The client-server model will continue to work as long as there is a uniform interface between them.

The client and the server can evolve independently.

In summary, decoupling the service into a client-server architecture has the following advantages:
* no impact of changes;
* separation of concerns;
* independent evolution.

### Uniform Interface

The client and server share a common technical interface. The technical interface devines the contract between the client and the server. It is implied bu the word *technical* that there is no business context here. The contract is creating following four guiding principles:

* Individual resources are identified in the request (URI/URL);
* Representation of the resources
  * the API client receives the representation of the resource;
  * the client uses the representation to carry out the manipulation of the resource, meaning that you can use the ID to then change the internal representation via another request;
  * the format of the data on the server side is not necessarily the same as what is desired on the client end.
* Self descriptive messages (metadata) - content-type, HTTP status code, etc;
* Hypermedia - the server not only sends the response data but it also sends back the hypermedia that the client can use for discovery.

### Statelessness

The client must manage its own state thus making the server stateless. This simplifies server design, facilitates server horizontal scaling and reduces resouces needs for the server.

The requests received from the client applications are self container, i.e., the server receives all of the state information that it needs for processing the client request. Each request received by the server is treated as an independent request without regards to any previous request from the same client or from other clients.

### Caching

Statelessness has some challenges:
* increased chattiness, i.e., multiple calls to get resource representation;
* request data size increases since the client also has to send state data;
* performance might degrade due to these reasons.

Use caching to achieve higher scalability and performance, it is used to counterbalance the negative effects of a stateless implementation. We can have:
* database caching - better performance from the database perspective;
* local cache within the server to cache responses that it will send and the data from the database;
* client-side caching
* proxy or load balancer level caching

The API server can use cache-control directives in responses to control caching behaviour on the applications and its intermediares. In some scenarios where responses cannot be cached, the server should explicitly mark those responses as non-cacheable. The cache-control directives are specified in the HTTP headers in the response. This is achieved via the "Cache-Control" HTTP header.

### Layered

Each of the layers has a unidirectional dependency on the layer next to it. A layer can only connect with the layer that it is dependent on, it cannot bypass its dependencies and reach out to other layers. Layering simplifies the architecture, reducing dependencies. The architecture may evolve with the changing needs. Layer changes at most impact only one of the layers.

### Code on Demand

The server can extend the functionality of the client by sending it code. This is an optional constraint. In other words, the server can send, for example, JavaScript code for the browser to execute.

The idea behind HATEOAS (Hypertext As The Engine Of Application State) is similar to code-on-demand. The response might contain hyperlinks, basically. This means that the actions that might be taken by the client are made available by the server via hyperlinks in the response. The main benefits are:
* the server is aware of the full resource state and can decide on actions that can be taken on it;
* the server can change the URI on the resource without breaking the client;
* the server may add new functionality over a period of time.

## Designing REST APIs

https://domain/product/version/resource/{id}

* https://domain - base domain
* product/ - grouping name (optional)
* version - API version
* resource - REST resource
* {id} - resource ID

Do not use the www sub-domain for your API. Use dedicated domains for your APIs.

The benefit of grouping APIs by product is that the APIs can be managed independently by multiple teams. Another benefit is that it makes it easier for the client to understand the API/resource.

An API can be thought of as a software product and, like so, you need to manage multiple versions at the same time. This makes it easier to insulate API consumers from API changes. You can also manage versions via headers or query parameters. The root URL is everything before the actual resource.

The names of resources are always nouns. Can be either plural or singular, there is not a defined standard here. What if the operation on a resource is not a CRUD operation? For example reporting, searching or calculations? In this case, we can define the action using a verb. This is usually tied to query parameters, when no resource is specified. When the action is tied to a resource, it should be part of the hierarchy.

### REST API Contract and HTTP Status Codes

The HTTP verb the client uses depends on the operation that will be carried out in the resource. The request data is consumed by the server to carry out the action on the resource. The server sends back data by the way of a response. The request can provide information via:
* query parameters;
* HTTP headers;
* HTTP body.

The response can provide information via:
* HTTP headers;
* HTTP body.

The request and response need to be specified independently for each of the HTTP methods. These specific aspects are provided in the Uniform Interface. Needs to define:
* endpoint;
* allowed query parameters;
* HTTP verbs;
* standard headers and custom headers (response and request);
* request/response body format and schema.

An HTTP server always sends back the status code which is a 3 digit code with the first number categorizing it:
* 1xx - informational
* 2xx - success
* 3xx - redirection
* 4xx - client error
* 5xx - server error

Do not invent new HTTP codes, use the standard HTTP codes for your API.

### CRUD Practices

Use the appropriate HTTP verb for CRUD:
* Create - POST
* Read - GET
* Update - PUT
* Delete - DELETE

Responses to POS requests may return the JSON of the new resource or a link to the new resource. Likewise, DELETE requests may return back the delete resource in the response. For updates, we can actually use two different verbs:
* PUT - update all of the attributes of the resource
* PATCH - update some of the attributes of the resource. This may be more performant and easier to use for large size objects.

## REST API Error Response

The `Status-Code` and `Reason-Phrase` are used in the implementation for sending status information back to the API client.

The body may contain application status codes defined by the API designer. It is suggested that these are standardized across all APIs from a given organization.

Options for sending error information:
1. Error information only in HTTP Header: Status-Code, Reason-Phrase, x-Custom-header
2. Error information only in body (not a great approach)
3. Error information in header and in body (common approach)

It is recommended that the number of status codes a given API returns be limited to about 10. The most common are: 200, 201, 400, 404, 401, 403, 415, 500.

You should limit these because it makes it hard for the development team to manage a lot of status codes and eventually leads to an inconsistent API. Also makes the app developer's lives easier.

### REST API Error Response Format

The error info sent to the API client is meant for use by the application developer to define the runtime behaviour. It can also be used for logging for root cause analysis and for creating reports.

The error response should be informative and actionable. It may send back the required field that was missing, for example. It should also be simple and consistent across APIs in the organisation:
* Provide links to docs in the error message
* Hints to address the issue
* Messages that may be presented to end users
* Define and use numeric application status codes
* Maintain/share the app code with all developers in the team

```json
{
  code: 666,
  text: "missing field ASDF is required",
  hints: ["please check the user has provided a non null ASDF"],
  info: "http://link.to.docs"
}
```

You should not expose database specific errors because:
* It would expose the internal implementation details of your API;
* The format of the errors may change in future versions of the database, requiring apps to change;
* you may change the database technology and hence lead to a change in error message formats.

## REST API Versioning Patterns

### Handling Changes

Assume that your API will change over time: because it will, like all software.

Changes to an API impact external and internal consumers.

Non-breaking changes:
* adding stuff 
* adding optional stuff;

Breaking changes:
* field name changes;
* HTTP verb changes;
* Removing an endpoint;
* Removing fields.

Always attempt to eliminate or minimize impact on end consumers. Provide planning opportunity to the application developers. Support backward compatibility if possible:
* Provide support to application developers with the change (i.e., docs);
* Minimize change frequency;
* Version your API right from day one.

### Versioning

API versioning should be managed like any other software product. Having a clear version roadmap is important. Support for multiple versions is usually a must. One should ask the following questions:
* How will the consumer specify the version?
* What will be the format for version information?

There are three options for specifying a version (by order or popularity):
* HTTP Header
* Query parameter
* URL path

Version formats:
* Date of release as version: 2023-02-01
* major.minor, typically prefixed with a v: v1.2
* number, typically prefixed with a v: v1
* Date and number of in-month release: 2023-01-89
* Semantic versioning

API change strategy:
* Available => deprecated => retired, should support at least two versions at a time;
* Support at least 1 previous version; maker previous versions as deprecated; publish a rollout plan in advance; manage changelog that clearly motivates the new version.

## REST API Caching

Caching can be built on all touchpoints:
* client caching;
* ISP caching;
* Gateway caching;
* DB caching;
* API caching.

This improves performance as well as scalability/throughput. Factors that need to be taken into account when caching:
* Speed of change;
* Time sensitivity;
* Security.

Design decisions:
* Which component should control caching?
* What to cache?
* Who can cache?
* For how long is the cache data valid?

The last three bullet points are controlled via the HTTP Cache-Control directives.

Cache-control directives must be obeyed by all caching mechanisms along the request/response chain. 
* Response side cache-control:
  * who can cache this response;
  * for how long;
  * under what conditions.
* Request side cache-control:
  * override the caching behaviour
  * protect sensitive data from caching

Sensitive data should not be cached on intermediares. Private data is meant for a single user. Sensitive data should not be stored anywhere. Always get the data from the server.

Subsequent resquests to the same URL will return different data. Etag header can be used to check if the data has changed. Lifetime of cache data depends on for how long the data is intrinsically valid. Practices:
* Take advantage of caching specially for high volume APIs;
* Consider no-store and private for sensitive data;
* Provide the validation tag (Etag) especially for large responses;
* Carefully decide on the optimal max-age.

### REST API Partial Response

Typically the REST API server will expose a common endpoint for all REST clients. However, some consumers might not require the entirety of the data, which might lead to an unnecessary use of resources. The "one size fits all" approach is rarely optimal from the client perspective. You should let the client control the granularity of the data. Ways to support partial responses:
* custom built partial response support;
* GraphQL

There are two ways which are commonly used for the field specification by the API client:
* single query parameter - holds expression that identifies the fields, which are known as projections;
* multiple query paramters - provides ways to filter the fields in the response, aka filters for the fields via query parameters

### REST API Pagination

Pagination gives the API consumer control over the responses, it kind of works as requesting a given number of rows. The API server splits the response from the database into several pages and sends only the requested page. Benefits:
* Better performance and optimized resource usage (CPU, memory, bandwidth);
* Common API version for all consumers (supports multiple devices).

There are three common ways in which REST APIs implement pagination:
* cursor based
* offset based
* use of HTTP headers

A cursor is a control structure that enables transversal of records. Cursor based pagination is considered the most efficient. A cursor is a random string that points to a specific item in a collection. When the API is invoked, it sends back an envelope based response, with metadata about paging:

```json
{
  paging: {
    cursors: {
      after: "something",
      before: "something else"
    }
    previous: "full link",
    next: "full link"
  }
}
```

Offset based pagination is the most common approach. It usually works via query parameters. The requester has to provide the starting row and the number of rows to receive.

HTTP Link headers is a way to create relationships between resources.


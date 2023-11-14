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

## Summary

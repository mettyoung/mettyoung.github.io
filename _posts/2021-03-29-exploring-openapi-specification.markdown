---
layout: post
title:  "Exploring OpenAPI Specification"
date:   2021-03-29 19:00 +0800
categories: tooling
---
Does your team have a centralized document for your APIs?

Without an API document, QA and UI engineers will have to ask an API engineer to get the necessary information to complete their tasks. Furthermore, the API engineer would need to check the source code in case he forgot the details of the API.

With an API document which can be accessed by all of the members, the team benefits on having a central repository of knowledge of the APIs. 

You can use any format for the API document. But as for me, I would prefer to use a standard called  **OpenAPI specification.**

## What is OpenAPI Specification?

[OpenAPI specification](https://swagger.io/specification/) was formerly known as Swagger specification. It can work with [Swagger UI](https://swagger.io/tools/swagger-ui/) or [Redoc](https://redoc.ly) to generate a Web UI to visualize the API specification better.

*An added benefit of OpenAPI specification is the amount of [tooling](https://openapi.tools) provided by the developer community. Please make sure to check them out!*

The specification can be written in either YAML or JSON format.

For this short blog article, I will not be showing how to write an OpenAPI specification but only to raise awareness about the tool and how the tool can help the development team.

## Practical Example

We shall see an example of a small API design to see how OpenAPI specification can be incorporated.

Let's assume we're going to write an expense manager API following the use cases:
  - User can record an expense transaction given price, memo and date.
  - User can amend an expense transaction given price, memo and date.
  - User can delete an expense transaction.
  - User can view all of the expenses recorded.
  - User can view the expense report containing total expenses per month.

How are we going to proceed with designing our API? 

One guideline that we can actually use is known as the [Richardson Maturity Model](https://martinfowler.com/articles/richardsonMaturityModel.html).

## What is Richardson Maturity Model?

Basically, it is a model breaking down the principal elements of a REST API. Here's a TLDR description of it:

1. **Level 0** - Merely uses HTTP as a transport protocol. Different actions are invoked by having different payloads (in simple terms).
2. **Level 1** - Introduces resources which are used to invoke different actions.
3. **Level 2** - Introduces HTTP verbs to distinguish between safe and non-safe operations and response codes to distinguish between different success and failure cases. 
4. **Level 3** - Introduces discoverability via hyperlinks. HTTP response body will contain the next possible actions based on the invoked endpoint.

**For our example, we will be using REST Level 3 and see how it looks like in the specification.**

## Epilogue

Since the main focus of this blog post is the OpenAPI specification, I will take the privilege of skipping the explanation on designing API endpoints and just present you the [API documentation](/static/i-openapi.html) for our expense manager application! 

Please feel free to browse it. I hope this short blog post help you in some way! 

Happy coding!
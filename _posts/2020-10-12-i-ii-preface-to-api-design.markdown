---
layout: post
title:  "Preface to API Design"
date:   2020-10-12 00:00 +0800
categories: series I
published: false
---
## Use Cases

We will always start with identifying the use cases first. From the use cases, we will be able to derive what API endpoints to create.

Please see the previous [post](/series/i/2020/10/12/i-i-getting-started-with-expense-manager.html#user-stories) for the user stories.

The functional specification requires the application to be able to create, amend and delete an expense transaction. It can also view all of the transactions recorded. If we look closely, the four use cases actually resembles what we call the CRUD paradigm. 

CRUD stands for Create, Read, Update and Delete. These are the four basic functions that are very common across many applications. To such an extent that there were already efforts made by the development community to create frameworks that automatically generates scaffolds using framework-specific CLIs like [AnduxJS](https://medium.com/@fccoelho7/creating-a-crud-application-in-less-than-2-minutes-with-ruby-on-rails-and-anduxjs-f42b60499ca7).

Anyway, the following API endpoints can be the following:

- **POST /expenses** (Create an expense transaction)
- **PUT /expenses/${expenseId}** (Amend an expense transaction)
- **DELETE /expenses/${expenseId}** (Delete an expense transaction)
- **GET /expenses** (Get all expenses)
- **GET /expenses/stats?queryMonth=** (Get statistical report per month)

## API Design Guidelines

There are two guidelines that I wish to use in my API design. 

**First, I want to utilize [Richardson Maturity Model](https://martinfowler.com/articles/richardsonMaturityModel.html).** Basically, it is a model breaking down the principal elements of a REST API. Here's a TLDR description of it:

1. **Level 0** - Merely uses HTTP as a transport protocol. Different actions are invoked by different payloads.
2. **Level 1** - Introduces resources which are used to invoke different actions.
3. **Level 2** - Introduces HTTP verbs to distinguish between safe and non-safe operations and response codes to distinguish between different success and failure cases. 
4. **Level 3** - Introduces discoverability via hyperlinks. HTTP response body will contain the next possible actions based on the invoked endpoint.

**In my case, I'm going to use REST Level 3.**

**Secondly, I wish to sit down and design how our API will look like first.** *This is my preferred way of doing things.* Based on my experience, it is helpful to design first because it pushes us to consider things we might miss if we directly implement the application. For designing APIs, I think it will be beneficial if we conform to some standardized way of writing API specification. 

In our case, **I will be using OpenAPI specification**. OpenAPI specification can work with Swagger to generate a Web UI to visualized the API specification better.

I'll be seeing you in the next post for the API design!
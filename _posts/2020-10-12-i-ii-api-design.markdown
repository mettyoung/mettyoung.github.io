---
layout: post
title:  "API Design"
date:   2020-10-12 00:00 +0800
categories: series I
published: false
---
## Use Cases

We will always start with identifying the use cases first. From the use cases, we will be able to derive what API endpoints to create.

Please see the previous [post](/series/i/2020/10/12/i-i-getting-started-with-expense-manager.html#user-stories) for the user stories.

The functional specification requires the application to be able to create, amend and delete an expense transaction. It can also view all of the transactions recorded. If we look closely, the four use cases actually resembles what we call the CRUD paradigm. 

CRUD stands for Create, Read, Update and Delete. These are the four basic functions that are very common across many applications. To such an extent that there were already efforts made by the development community to create frameworks that automatically generates scaffolds using framework-specific CLIs like [AnduxJS](https://medium.com/@fccoelho7/creating-a-crud-application-in-less-than-2-minutes-with-ruby-on-rails-and-anduxjs-f42b60499ca7).

## API Design Guidelines

There are two guidelines that I wish to use in my API design. 

**First, I want to utilize [Richardson Maturity Model](https://martinfowler.com/articles/richardsonMaturityModel.html).** Basically, it is a model breaking down the principal elements of a REST API. Here's a TLDR description of it:

1. **Level 0** - Merely uses HTTP as a transport protocol. Different actions are invoked by having different payloads (in simple terms).
2. **Level 1** - Introduces resources which are used to invoke different actions.
3. **Level 2** - Introduces HTTP verbs to distinguish between safe and non-safe operations and response codes to distinguish between different success and failure cases. 
4. **Level 3** - Introduces discoverability via hyperlinks. HTTP response body will contain the next possible actions based on the invoked endpoint.

**In my case, I'm going to use REST Level 3.**

**Secondly, I wish to sit down and design how our API will look like first.** *This is my preferred way of doing things.* Based on my experience, it is helpful to design first because it pushes us to consider things we might miss if we immediately jumping to coding. For designing APIs, I think it will be beneficial if we conform to some standardized way of writing API specification. 

In our case, **I will be using [OpenAPI specification](https://swagger.io/specification/)**. OpenAPI specification can work with [Swagger UI](https://swagger.io/tools/swagger-ui/) or [Redoc](ttps://redoc.ly) to generate a Web UI to visualized the API specification better. 

*An added benefit of OpenAPI specification is the amount of [tooling](https://openapi.tools) provided by the developer community. Please make sure to check them out!*

## Epilogue

We can now finally design our API for our expense manager application!

Please click [here](/static/i-openapi.html) to view the beautiful API documentation.

Now that we've finished the API design, let's proceed with system design.

{% comment %}
Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
{% endcomment %}

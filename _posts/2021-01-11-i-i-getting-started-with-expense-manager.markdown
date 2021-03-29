---
layout: post
title:  "Getting Started with Expense Manager"
date:   2021-01-11 00:00 +0800
categories: series I
published: false
---
This is my first blog post and I thought of writing a series that will walk us through how to build a full-blown web application that can be deployed to the cloud.

In this series, I came up with a project idea that is personally very useful to me which is an **expense manager**.

**An expense manager is an application that allows you to track all of your expenses.** I have always been logging all of my expenses since I started working. This allows me to assess if I'm ~~underspending so I can spend more~~ overspending and should probably slow down. I thought this would be an interesting application to create.

Enough with idle talk. Let's cut to the chase!

## Functional Specifications
We will build a minimum viable product that provides enough value for users to use it. When I think about expense manager, without all the fancy features, all I need are the following:

### User Stories
- User can record an expense transaction given price, memo and date.
- User can amend an expense transaction given price, memo and date.
- User can delete an expense transaction.
- User can view all of the expenses recorded.
- User can view the expense report containing total expenses per month.

### Limitations
- Only cash outflow is recorded. There is no cash inflow.
- There will be no categories for all of the expenses.
- No wallet balance.
- No currency used.
- Multi-tenancy not supported.

## Technical Specifications
We are going to build a REST API which is usually paired with client-side rendering Web UI. Another alternative is using server-side rendering Web UI which is usually paired with MVC web application. My reason for choosing REST API with client-side Web UI is simple: **With client-side rendering, the user interface is more fluid than server-side rendering. Of course, this comes with a price of more complexity but I think it is worth it!**

- For REST API, we are going to use **Spring Boot framework**! 
- For Web UI, we are going to use **ReactJS**!

## Epilogue
All right! The features aren't much but I think this is enough to provide users some prototype to gauge their interest for the product. Through the specifications listed above, the user will be able to log all of his expenses to the Web UI which can be accessed through different devices. Since the records will be persisted to a database, data sync will automatically happen between devices. The user will also be able to see his total expenses per month to gauge if he's overspending. 

Let us now proceed with API design!
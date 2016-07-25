---
layout: post
title:  "Building a Node.js RESTful API: The Database"
date:   2016-07-23 12:56:45
categories: Node.js
banner_image: "/media/desk.jpg"
featured: false
comments: true
---

## Introduction

Welcome again! If you reached this post from nowhere and don't understand what all this is about, you can check out
the previous posts: 

- [Intro and Initial Setup]({% post_url 2016-07-22-building-a-node-restful-api  %})
- [Setting up the Web Server]({% post_url 2016-07-23-building-a-node-restful-api-2  %})

<!--more-->

## Setting up the DB

Ok, if everything went well, you should have `package.json` file on your project folder and it should look like the
example one shown to you by the `npm init` wizard. After this, there are lots of things to be done to have our project
setup and running. So let's get started by setting up our database.

### MongoDB

We'll be storing all our tasks data into a MongoDB database. The reasons I choose MongoDB are simple: I'm familiar with
it, it makes a great couple with Node.js and lastly because we won't have a super complex application. We're just building
a task manager, so there's no need to worry about atomic operations, perfomance when querying related entities, etc.
And also we're doing this to learn how to build an API with Node.js, so if in the future you'll use the same project
to build a more powerful API you can easily replace MongoDB with the DBMS you consider the best for your project purpose.

Obviously you'll need to install MongoDB if you didn't yet. It's not a part of this tutorial the MongoDB installation,
but you can find plenty of help and documentation on how to do that in the [MongoDB Installation](https://docs.mongodb.com/manual/installation/)
page.

### The ORM

Once you have MongoDB installed, it's time to choose an [ORM](https://en.wikipedia.org/wiki/Object-relational_mapping).
It's not that we need one, but it's always handy to have one of them cause they usually abstract a lot the database
management operations. In our case we'll use [Mongoose](http://mongoosejs.com/) and the reasons are simple: it has proven
to be a great ODM (yeah, note that I used O**D**M - D for Document - and not O**R**M, it's basically the same an ORM is 
but for a non relational database like MongoDB) for MongoDB and it has a LOT of documentation and community support.

Let's install mongoose then:

{% highlight bash %}
npm install --save mongoose@4.3.7 kerberos@~0.0
{% endhighlight %}

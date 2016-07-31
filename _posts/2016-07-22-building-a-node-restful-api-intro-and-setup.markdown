---
layout: post
title:  "Building a Node.js REST API 1: Intro and Initial Setup"
date:   2016-07-22 12:56:45
categories: Node.js
banner_image: "/media/node-js-hexa.jpg"
featured: true
comments: true
---

## Introduction

<!--from-->
Welcome to the first post of this series! In this collection of posts I'll explain how to build a RESTful API using 
Node.js, MongoDB and Babel to make use of the power that ES6 brings to the javascript world.
I've found myself multiple times building a RESTful API for different projects and always taken different approaches.
Mostly because the Node.js ecosystem is growing so rapidly that whenever I start a new project there are already new
tools and frameworks, or new versions of the frameworks I used in the past.
<!--to-->

You can also find lots of boilerplates ready to be used, if you're an experienced Node developer you can just grab one
of those. The goal of this posts is to provide an understanding on how such a boilerplate can be built from scratch
and understand the core pieces of it. I'm not planning to go deep in very basic concepts, but you can always post a comment
in the post or find information online if there's anything you don't understand.

You can find the code for this tutorial in [GitHub](https://github.com/mpayetta/node-es6-rest-api).


## Why a RESTful API?

Well, we're living in the golden era of the internet. Nowadays to have presence in the global network means not only
having a website that people can find in their desktop or laptop computers, but also to have mobile versions of your
website, mobile applications for all the popular platforms (Android, iOS, Windows, etc) and even provide a way to 
3rd parties to use your system.
Following the basic concepts of systems design and architecture, this means that you need to put the access to your core
data and functionality into a module that can be used by all of your client modules (web app, mobile app, 3rd parties...).
If we are 100% sure of something is that we don't want to replicate the same code to access our data into each of our clients.

Here is when your API comes into play, and the diagram below shows clearly what I wanted to explain in the previous paragraph:

 
<img src="/media/REST-API.jpg">

As you can see, any kind of client can access the REST API and display your system data in any way they want. A 
successful action might be shown in a modal window in the web client, as a toast in the android app and as an alert
in iOS. It doesn't matter how clients use the data, what matters is that all of them access the data in the same
way. 
If you want to learn more about REST APIs you can find lots of information online.

## Project scope

We'll build a simple task management API and some of the tools we'll be using to build it are:

- [Express](https://expressjs.com/): will be our web server which will be running our API
- [Babel](https://babeljs.io/): will provide the support to write ES6 code
- [MongoDB](https://www.mongodb.com/): will be our project database system
- [Mocha](https://mochajs.org/): will be our testing framework
- [Apidoc](http://apidocjs.com/): will allow us to build automatically API Docs from our code comments
- [Gulp](http://gulpjs.com/): will be our task manager

The project scope involves developing only the API, we'll not develop any kind of API client (we'll use just the console
to test our API endpoints).

I will assume you already have [Node](https://nodejs.org/en/) and [NPM](https://www.npmjs.com/) installed. You can find
instructions to do this [here](https://docs.npmjs.com/getting-started/installing-node). 
If you already have them installed, make sure you have NPM v3 installed, since we'll use Babel and the dependency hierarchy
it's too deep and complex. Hence the NPM 3 [way of installing](https://docs.npmjs.com/how-npm-works/npm3) dependencies 
in a "flat" way will highly decrease the time to transpile from Babel ES6 to ES5.

## Let's get it started

Ok, enough introduction, let's get our hands dirty!

First let's open a console window and kickoff our project. For this we'll use the `npm init` command which will guide 
us through the process of creating our project `package.json` file. 

{% highlight bash %}
mkdir node-es6-api
cd node-es6-api
npm init
{% endhighlight %}

Follow the wizard by providing the information it asks (it's all trivial information, you can even ignore the description 
and keywords fields for now). You can see an example in the snippet below:

{% highlight bash %}
mbp-mauricio:node-es6-api mauricio$ npm init
name: (node-es6-api)
version: (1.0.0)
description: A node.js RESTful API built with ES6
entry point: (index.js)
test command:
git repository:
keywords: node rest api es6
license: (ISC)
About to write to ~/node-es6-api/package.json:

{
  "name": "node-es6-api",
  "version": "1.0.0",
  "description": "A node.js RESTful API built with ES6",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [ "node", "rest", "api", "es6" ],
  "author": "Mauricio <mauricio@bithive.io>",
  "license": "ISC"
}

Is this ok? (yes)
{% endhighlight %}

The `"test": "echo...` part in the `scripts` property can be ignored for now, we'll see later how to add unit tests to 
our API and how to configure npm to run them easily.

## Other things you can do

Before we move on to the next post about setting up the database, you could do a few more things like:

- Create a `README.md` file with a little explanation of what your project is about. Later you can fill it with details
on how to install and run the project.
- Create a `.gitignore` file to list the files you don't want to be checked in. In the beginning you can start with all the
IDE specific files (for example add `.idea` if you work with Webstorm)
- Push all the changes you have so far into your own git repository (or whatever versioning system you prefer)

## Coming up next...

That was all for the first post. At least we can say that we understand why REST APIs are useful nowadays and we already
have a node project started thanks to the `npm init` wizard.
In the next posts we'll start giving some life to our project starting by setting up the database that will feed our 
tasks API.
Thanks for reading! 

See you in the next post: [Setting up the Web Server]({% post_url 2016-07-23-building-a-node-restful-api-the-web-server  %})

If you enjoyed reading, retweet or like this one!

<blockquote class="twitter-tweet" data-lang="es"><p lang="en" dir="ltr">The complete guide to write a RESTful API with Node and ES6 starts here! <a href="https://t.co/9Wj6iiS3Qc">https://t.co/9Wj6iiS3Qc</a></p>&mdash; Mauricio Payetta (@mpayetta) <a href="https://twitter.com/mpayetta/status/759583208501448704">31 de julio de 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

<div class="cc">
    <a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
        <img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc/4.0/88x31.png" />
    </a>
    <br/>
    <span xmlns:dct="http://purl.org/dc/terms/" href="http://purl.org/dc/dcmitype/Text" property="dct:title" rel="dct:type">
        Building a Node.js REST API 1: Intro and Initial Setup
    </span> 
    by 
    <a xmlns:cc="http://creativecommons.org/ns#" href="http://blog.mpayetta.com" property="cc:attributionName" rel="cc:attributionURL">
        Mauricio Payetta
    </a> 
    is licensed under a 
    <a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
        Creative Commons Attribution-NonCommercial 4.0 International License
    </a>.
</div>
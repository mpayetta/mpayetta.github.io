---
layout: post
title:  "Dockerizing a Node.js and MongoDB App"
subtitle: "A lightweight approach with Alpine Linux"
date:   2016-09-04 00:00:45
categories: Node.js Docker MongoDB
banner_image: "/media/docker-node.png"
banner_idx: "/media/docker-node_idx.png"
banner_position: -270px
featured: true
comments: true
---

## Introduction

<!--from-->

Docker has become an extremely popular tool not only among DevOps and Infrastructure people, but also for the daily work
of any developer. And as a developer, during the past week I decided it was time to have a look at it and see by myself
all the good things people talk about. And I don't regret it! In a matter of days I could understand the basics (the
very basics) and get up and running with what I mostly do these days, a Node.js and MongoDB application.

<!--to-->

As I said, these are my first experiences with Docker, so please don't take this as an expert tutorial. This post is aimed
to the Docker beginners like me, who want to get a fully working instance of an Express and MongoDB app up and running
in a matter of seconds for daily work. This applies for any Node.js app, but for the matter of the post I'll work with 
a dummy Express server that returns some dummy data from a MongoDB.

Before starting make sure you have Node.js, npm and Docker installed.

## The dummy Node app

We'll start by creating the Node.js app that will be "dockerized" later on. For this purpose we'll use the fast express
generator. Make sure you have express installed globally:

{% highlight bash %}
# you might need to add a `sudo` to the command
npm i -g express
{% endhighlight %}

Then `cd` into any directory you want and create the `dummy-app`:

{% highlight bash %}
express dummy-app

   create : dummy-app
   create : dummy-app/package.json
   create : dummy-app/app.js
   create : dummy-app/public
   create : dummy-app/public/javascripts
   create : dummy-app/public/images
   create : dummy-app/public/stylesheets
   create : dummy-app/public/stylesheets/style.css
   create : dummy-app/routes
   create : dummy-app/routes/index.js
   create : dummy-app/routes/users.js
   create : dummy-app/views
   create : dummy-app/views/index.jade
   create : dummy-app/views/layout.jade
   create : dummy-app/views/error.jade
   create : dummy-app/bin
   create : dummy-app/bin/www

   install dependencies:
     $ cd dummy-app && npm install

   run the app:
     $ DEBUG=dummy-app:* npm start
{% endhighlight %}

Follow the instructions and go inside the `dummy-app` dir and run `npm install` to get all the needed dependencies.
And after that run the app:

{% highlight bash %}
# The DEBUG variable lets us see in the console the express app logs
DEBUG=dummy-app:* npm start

> dummy-app@0.0.0 start dummy-app
> node ./bin/www

  dummy-app:server Listening on port 3000 +0ms
{% endhighlight %}

Now go try it in the browser, you should see the friendly "Welcome to Express" page.

## Dockerizing: node vs alpine-node

Now that we have our very simple webapp, we can start with the dockerization process. If you don't know yet what a 
docker image and container is, I highly recommend reading [this](https://docs.docker.com/engine/getstarted/step_two/).

The first decision we have to make is which image we'll use to run our app container. There are many different choices,
actually a search of "Node" in [Docker Hub](https://hub.docker.com/search/?isAutomated=0&isOfficial=0&page=1&pullCount=0&q=Node.js&starCount=0) 
results (as of today) in 10799 repositories.

The most popular and easy choice might be the [official Node.js image](https://hub.docker.com/_/node/). The official Node.js
image is based on Debian Linux, and as the image description states:

> It, by design, has a large number of extremely common Debian packages

Which might be very helpful in some cases, specially if you want to extend the image and have maybe complex applications
which rely in many native libraries. But for the general use, this means that a lot of space is being used to have tools 
that you'll never use. According to the image details, the `wheezy` version (which is based on the wheezy release of
Debian) is 191 MB big when compressed.

So why use the Node official image? Because it's the easiest way to get a Node.js system up and running. Are there any
cheaper alternatives in terms of space? Yes.

We'll be using one of those smaller alternatives to save some space. And the alternative we'll use is Alpine Linux. If you
check the compressed size of the [alpine image](https://hub.docker.com/r/library/alpine/tags/) it's only 2 MB. When 
compared to the 191 MB of the Node.js image that's a huge improvement. But we have to understand that alpine it's a super
basic linux version, so on top we'll need to install things like Node.js and npm. Actually we don't have to install 
anything cause there's already someone who made a Node.js image based on alpine linux, and it's called (yeah, you guessed
it) [alpine-node](https://hub.docker.com/r/mhart/alpine-node/).

This image of Node.js based on Alpine Linux weights around 18 MB compressed (the full version with Node.js and npm, there's 
another version called `base` which does not have npm and it's way much smaller). We'll be using the full version and
in the end I'll explain how our image size can be reduced by using just the base version. So now that we decided which
image we'll use as a base, let's write our first Dockerfile!

## The initial Dockerfile

Go to your `dummy-project` and add a file named `Dockerfile`. This file contains the instructions we'll give Docker in
order to build our own project image. There are many commands you can use in a dockerfile, for the full reference you
can have a look [here](https://docs.docker.com/engine/reference/builder/). Once the file is created, we'll add our first
line to it which will tell Docker from which image are we going to base our own image:

{% highlight bash %}
FROM mhart/alpine-node:latest
{% endhighlight %}

So here we tell Docker that our image will be based on the image called `alpine-node` on the `mhart` namespace and we'll
be using the `latest` tag of this image.

The next thing to do is copying our application code into the image as well as installing all the dependencies. But first
we need to understand a bit how the Docker image cache works in order to get the most of our Dockerfile. Basically you
need to understand that every time Docker runs an instruction, it will first check in your local image cache if there's
an image that can be reused for that same instruction. Specially for the `COPY` and `ADD` commands, Docker will check
if the host files have changed compared to the files existing in the cached image. But why is this important?

As it's very well explained in [this post](https://www.ctl.io/developers/blog/post/caching-docker-images/)

> Each instruction in your Dockerfile results in a new image layer being created and added to your local image cache. 
  That image then becomes the parent for the image created by the next instruction
  
We can make a very good use of Docker's layer caching with our `package.json` dependencies. The natural way of thinking
would output the following 2 steps:

1. copy our whole project code to the image 
2. run npm install
3. done! all our code and dependencies, we're ready to run the app

### Taking advantage of image layers caching

The big issue with this approach is that whenever we change a file in our project code (which of course happens extremely
frequently), then when Docker tries to build the image layer for step 1 it will find that the cached image is no longer
valid, which means that subsequent steps can not be taken from cached images. 
 
What's the solution? 

1. Copy the `package.json` file alone in a single instruction into a temporary folder
2. Install all the dependencies with `npm install`
3. Copy our project source code into another directory in the image
4. Move the installed dependencies into the project directory in the image

Let's see how this looks like in our Dockerfile and I'll try to explain why this approach is better:

{% highlight bash linenos %}
ADD package.json /tmp/package.json
RUN cd /tmp && npm install
RUN mkdir -p /opt/app && cp -a /tmp/node_modules /opt/app/

WORKDIR /opt/app
ADD . /opt/app
{% endhighlight %}

In line `1` we copy our local `package.json` file into the `/tmp` directory on our image. Right after that on line `2`
we move into the `/tmp` dir and run `npm install` to install all the dependencies there.
The next step in line `3` creates the `/opt/app` directory where we'll hold all our project code. This means we'll be
running our app from this directory and hence we need to have our dependencies there. That's what the 
`cp -a /tmp/node_modules /opt/app` command does.
Then in line `5` we change our work directory to the `/opt/app` and finally we copy all our project code from our local
host into the image app directory.

Each line in our Dockerfile will generate an image layer which will be cached. The first time we build the image there's
nothing cached, so everything will be run from scratch, but let's see what happens with subsequent builds:

The first line will generate an image
layer where we copy the package.json file and cache it. If we try to build the image again, Docker will compare the 
local package.json contents (actually it compares a bit more, but let's keep it simple here) with the one in the cached
image layer. If they are the same, it will use the cached image and continue with the next line. Since the previous
image was taken from the cache, it means that the next one is eligible to be taken from the cache as well.

In line number `2` another image layer will be created or taken from the cache. For `RUN` commands, the only thing Docker
does is comparing if there's already a cached image with the exact same command (yes, it's just a string comparison).
This means that if the package.json didn't change, the `npm install` step will be always taken from the cache, saving us
lots of build time whenever dependencies didn't change. Got the point?

The same will happen to subsequent commands (layers), but since we installed all our dependencies in a parent layer,
if any file has changed in our project code, this will affect only the step in line `6`. This will save lots of precious
build time, since we can always reuse our cached `node_modules` from the `/tmp` directory.

Sorry for the long explanation, I think this is a key concept which makes sense to fully understand before continuing.
Now that we have our initial Dockerfile, let's build our first image.

## Building the image, running the container

In order to build our Docker image we'll use the `docker build` command. Head over to your console and in the project
root directory run the following:

{% highlight bash %}
docker build -t dummy-app .
{% endhighlight %}

We're telling Docker that we want to build an image called `dummy-app` using the content in the `.` directory. This means
that Docker will look for a Dockerfile in the current directory. For more details about the build program feel free to
type `docker build --help` in your console.

After some time and console logging, you should see a message similar to this one:

{% highlight bash %}
Successfully built 3b12626c478a
{% endhighlight %}

Now you should be able to see the built image in your local images list:

{% highlight bash %}
$ docker images
REPOSITORY     TAG         IMAGE ID            CREATED             SIZE
dummy-app      latest      3b12626c478a        22 hours ago        73.9 MB
{% endhighlight %}

Once you've built the image, you can run it on a container. Basically a container is a running instance of an image.
Images are stateless, they are static for their entire life. Once you run an image in a container it becomes alive
and all the state resides in the container. This means you can use the same image to run as many containers as you 
want, and the image will always look the same. But containers run from the same image might be very different in terms
of data and state they have.

So in the same console window, run this command to start a container using the `dummy-app` image (make sure you stopped
the local server we tried running before to leave the port 3000 free):

{% highlight bash %}
$ docker run -p 3000:3000 -ti dummy-app

> dummy-app@0.0.0 start /opt/app
> DEBUG=dummy-app:* node ./bin/www

  dummy-app:server Listening on port 3000 +0ms
{% endhighlight %}

Voila! If you go to your browser and load the localhost:3000 url, you'll see again the express welcome message. Here we
told Docker that we wanted to run a container using as a base the `dummy-app` image. We also gave the instruction to 
map the port 3000 in the host to the port 3000 in the container, which allows us to access the app from the browser in
the host. For more details about run options check the docs with `docker run --help`. 

## Using Docker Compose

To run a simple webapp like the one we made, it's very easy with a single Dockerfile and using `docker build` and `docker
run`. But when we have multiple containers that relate to each other, it becomes a bit messy.

Docker comes with a very useful tool called [`docker-compose`](https://docs.docker.com/compose/). This tool allows us to
define all of our container and the relationships between them in a single `docker-compose.yml` file. Let's build the
docker-compose file for the same configuration we ran before and later we'll see how can we define our MongoDB container
as well in the same file and easily connect both. Create a `docker-compose.yml` file in the project root and put the
following contents on it:

{% highlight yaml linenos %}
version: "2"
services:
  web:
    build: .
    ports:
      - "3000:3000"
{% endhighlight %}

First we define which version of Docker Compose config syntax are we using, being 2 the latest one. Then we start defining
all the services that will be run. For now we just have a single service which we called `web` and will be our webapp.
In line `4` we define from where this container should be built, in the `dummy-app` case, the container is built from the
local Dockerfile, and that's why we say that the current directory `.` is the target.

In line `5` we define which are the port mappings for the container, which are the same we used before with the `-p` option
of the `docker run` command. 

Now you're ready to build and run the container using docker-compose. On the console first run

{% highlight bash %}
docker-compose build
{% endhighlight %}

This will build the image. Then you can run the container with

{% highlight bash %}
docker-compose up
{% endhighlight %}

There's a shorthand for this which is `docker-compose up --build`. This command will build all images before starting
the containers. When starting the container you'll see something similar to this in your console:

{% highlight bash %}
$ docker-compose up
Starting dummyapp_web_1
Attaching to dummyapp_web_1
web_1  |
web_1  | > dummy-app@0.0.0 start /opt/app
web_1  | > DEBUG=dummy-app:* node ./bin/www
web_1  |
web_1  | Sun, 04 Sep 2016 07:01:23 GMT dummy-app:server Listening on port 3000
{% endhighlight %}

Test it again by using your browser to see if the Express welcome message shows up.

## A not so dummy app

OK, we have the basics working. Our webapp is ready and running using Docker Compose. Now it's time to add some interesting
stuff. Well, maybe not that interesting, but at least it will make our app slightly more complex and will allow us to 
see how containers can communicate to each other and can be configured together in the same `docker-compose.yml` file.

Let's start by adding the mongodb driver to our project. Open the package.json and add a line to the dependencies:

{% highlight bash %}
"mongodb": "^2.2.9"
{% endhighlight %}

Now open the `routes/index.js` file and add a new route that will connect to a mongodb and return some data from the 
`dummy` collection. The route file will look like this:

{% highlight javascript %}
var express = require('express');
var router = express.Router();
var mongodb = require('mongodb');
var client = mongodb.MongoClient;

var uri = "mongodb://mongo/dummy-app";

/* GET home page. */
router.get('/', function(req, res, next) {
  res.render('index', { title: 'Express' });
});

router.get('/data/from/db', function(req, res, next) {
    client.connect(uri, function (err, db) {
	    if (err) return next(err);    
    	var collection = db.collection('dummy');
    	collection.find({}).toArray(function(err, docs) {
			if (err) return next(err);
			return res.json(docs);
    	});			
	});
});

module.exports = router;
{% endhighlight %}

You probably noted that in the URL instead of the typical `localhost` I used the name `mongo` to refer to the host of 
the database. This is because our MongoDB container will have that name and we'll link it to our webapp container so that
they know each other, more on this later.

This is a very ugly way of connecting to the database, in a real world example you would have your db connection managed
by a separate module, or you would even use a library like Mongoose to manage your MongoDB models and so on. Since it's
not the goal of this project to show how to use Express and MongoDB together, I chose the simplest (ugly) way to fetch
some data from the db and return it in the response.

We'll add another route to create some dummy data in the collection as well:

{% highlight javascript %}
router.post('/data/into/db', function(req, res, next) {
	client.connect(uri, function (err, db) {
	    if (err) return next(err);
    	var collection = db.collection('dummy');
    	collection.insertMany(req.body, function(err, result) {
			return res.json({ result: "success" });
    	});
	});
});
{% endhighlight %}

This route simply takes an array sent in the request body and inserts all the objects in the array into the `dummy` 
collection.

## The MongoDB container

Our app is ready to connect to a MongoDB instance, now we need to setup the container to run the MongoDB container to
which our webapp will connect. We can easily achieve that with a few more lines in our `docker-compose.yml` file:

{% highlight yaml linenos %}
version: "2"
services:
  web:
    build: .
    ports:
      - "3000:3000"
    links:
      - mongo
  mongo:
    image: mongo
    volumes:
      - /data/mongodb/db:/data/db
    ports:
      - "27017:27017"
{% endhighlight %}

You can see that I've added a `links` property to the webapp configuration. This is where we "connect" the containers
so that they can reference each other. This is what allows us to use a MongoDB connection URL like `mongodb://mongo/dummy`.
The `mongo` in the URL references the mongo container that we define starting in line `9` of the docker-compose file.

Instead of building the mongo container from a local configuration, we tell Docker to build it from the image called
`mongo` (line `10`). When Dockers sees this, it will try to find the image in our local repository, if it's not there
it will pull it from the [Docker hub registry](https://hub.docker.com/_/mongo/). 
 
Then we tell Docker to mount our host directory `/data/mongodb/db` into the container `/data/db` directory. The main reason
I did this is because I have all my local MongoDB databases stored in that directory, and I want to have access to them
when running the container. There are many options to do this, you can specify any directory you want, or you can even
create another special container called "volume container" which is simply a volume you can mount in any container you
want. Feel free to use whichever approach best suits your needs.

Finally we tell Docker to map the port 27017 on the host to the port 27017 on the container so that we can connect to 
the MongoDB server from any client we have in our local host.

Having all defined, we can proceed now to run again `docker-compose build` and `docker-compose up` to see how it goes!
If everything went well, you'll see the logs for both the MongoDB server and our Express server. You can do the same
test as always by pointing your browser to the `localhost:3000` URL and you'll see the Express welcome message again.

Now let's try to insert some data into the database by using the dummy endpoint `POST /data/into/db`.
 
{% highlight bash %}
$ curl -X POST -H "Content-type: application/json" http://localhost:3000/data/into/db \
    -d '[ { "a": 1 }, { "b": 2 }, { "c": 3 } ]'
{"result":"success"}
{% endhighlight %}

We got the result success response! Now let's try to retrieve the data by using our other dummy endpoint 
`GET /data/from/db` and see if our data was persisted:

{% highlight bash %}
$ curl http://localhost:3000/data/from/db
[{"_id":"57cbda4786941a000f572f31","a":1},{"_id":"57cbda4786941a000f572f32","b":2},{"_id":"57cbda4786941a000f572f33","c":3}]
{% endhighlight %}

Got our data back! Everything works as expected :)

## Conclusion

During this looong post we learned how to setup a (very) basic Node.js app that connects to a MongoDB with Docker. This
allows us to have all our system environment running for development with a single `docker-compose up` command.
For development purposes, you'd maybe like to mount your local code into the web container instead of copying it every
time you build. This approach used along with something like [nodemon](https://github.com/remy/nodemon) will allow you 
to work on the project files and reflect the changes instantly in the running container. Feel free to ask if you need
more info on this approach.

We also saw how using other image rather than the official Node.js Docker image helped to save a lot of space. When I check
my images with `docker images`, the dummy web app that uses the Alpine Node image is 81.28 MB big. On the other side,
when doing the same app with the officla Node.js image it becomes around 500 MB big. That's like 600% bigger.

From now on, for me it's all about experimenting with Docker features and start learning more about it. I'm sure that 
from today developing with distributed teams as I'm used to work will be much easier for everybody. Hope your life
gets easier as well!

If you liked the post, please share it wherever you want or re-tweet the following tweet! :)

<blockquote class="twitter-tweet" data-lang="es"><p lang="en" dir="ltr">Setting up <a href="https://twitter.com/docker">@docker</a> to run a <a href="https://twitter.com/nodejs">@nodejs</a> and <a href="https://twitter.com/MongoDB">@MongoDB</a> application. A lightweight approach with <a href="https://twitter.com/alpinelinux">@alpinelinux</a> <a href="https://t.co/TrPuzH7Y63">https://t.co/TrPuzH7Y63</a></p>&mdash; Mauricio Payetta (@mpayetta) <a href="https://twitter.com/mpayetta/status/772360075964854273">4 de septiembre de 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

<div class="cc">
    <a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
        <img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc/4.0/88x31.png" />
    </a>
    <br/>
    <span xmlns:dct="http://purl.org/dc/terms/" href="http://purl.org/dc/dcmitype/Text" property="dct:title" rel="dct:type">
        Dockerizing a Node.js and MongoDB App
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
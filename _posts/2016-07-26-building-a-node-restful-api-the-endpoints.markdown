---
layout: post
title:  "Building a Node.js REST API 5: The Routes"
date:   2016-07-26 12:56:45
categories: Node.js
banner_image: "/media/endpoints.jpg"
featured: true
comments: true
---

## Introduction

Welcome again! If you reached this post from nowhere and don't understand what all this is about, you can check out
the previous posts: 

- [1. Intro and Initial Setup]({% post_url 2016-07-22-building-a-node-restful-api-intro-and-setup  %})
- [2. Setting up the Web Server]({% post_url 2016-07-23-building-a-node-restful-api-the-web-server  %})
- [3. The Database: Setup]({% post_url 2016-07-24-building-a-node-restful-api-the-database  %})
- [4. The Database: Connect and Config]({% post_url 2016-07-25-building-a-node-restful-api-the-database-2  %})

If you followed the steps from the previous posts your project directory structure should look like this:

{% highlight bash %}
├── config
│   ├── env
│   │   ├── development.js
│   │   └── index.js
│   └── express.js
├── gulpfile.babel.js
├── index.js
├── package.json
└── server
    └── models
        ├── task.js
        └── user.js
{% endhighlight %}

If it doesn't, please go back and check if you missed something from the previous posts.

## Creating the Server Routes

<!--from-->

Now it's time to play a bit with our express server and start adding some functionality (interesting functionality) to it.
Our first job is to organize the server code, so for that purpose we'll create two new directories under the `server`
directory<!--to-->:

{% highlight bash %}
cd server
mkdir routes
mkdir controllers
{% endhighlight %}

Basically in the `routes` directory we'll put all the route configuration related code, and in the `controllers` directory
we'll put all the implementations for the routes. 

Before we start creating the routes let's understand a bit how a REST API should be organized in terms of endpoints.

### RESTful Endpoints

RESTful APIs basically are the entry point to retrieve and modify "Resources". A resource might be any entity that lives
in your system context, for example users and tasks will be the resources of our API. So with our REST API we'll provide
clients with a way to retrieve, create, modify and delete resources (users and tasks). But how do we organize the endpoints
to do all those kinds of operations?

REST defines a clear and simple way to define the API endpoints which relies on the HTTP methods or verbs: `GET`, `POST`,
`PUT` and `DELETE`:

- GET: will be used to retrieve resources data
- POST: will be used to create new resources
- PUT: will be used to update existing resources
- DELETE: yeah, as you imagined, will be used to delete resources

REST goes also a step further and tells us how our endpoint URLs should look like. So let's say we have a resource called
user, which represents all the users in our system. We'll need to create at least five different endpoints for the user resource
(and any other resource):

-  `GET /users`: will return the list of users in our system
-  `POST /users`: will create a new user in our system
-  `GET /users/[userId]`: will return the user with the given `userId`
-  `PUT /users/[userId]`: will update the data for the user with the given `userId`
-  `DELETE /users/[userId]`: will delete the user with the given `userId`

You can also create other endpoints as [listed here](https://en.wikipedia.org/wiki/Representational_state_transfer#Relationship_between_URL_and_HTTP_Methods)
but it's not very common, and we're not going to create them for the purpose of this tutorial. So we'll follow this same
approach and create those endpoints for our users and tasks resources.

### Route Configuration

Let's start by creating an `index.js` file inside the routes folder. Inside this file we'll put the "health check" endpoint
we had defined in our `config/express.js` file and we'll mount the rest of the endpoints later. So for now the 
`server/routes/index.js` file will look like this:

{% highlight javascript %}
import express from 'express';

const router = express.Router();

/** GET /api-status - Check service status **/
router.get('/api-status', (req, res) =>
  res.json({
    status: "ok"
  })
);

export default router;
{% endhighlight %}

Here we define an express router (more about express routing [here](https://expressjs.com/en/guide/routing.html)). And in
this router we create a `get` endpoint on the `/api-status` path, which will answer with a JSON object saying that the 
API is ok when it's running.

Now that we'll have our routes configured separately in the `routes` directory we can mount them in our server. To do that
open the `config/express.js` file and do the following changes:

1. Remove the `app.get('/')...` since we now have our routes defined separately
2. Import the new routes configuration
3. Mount the routes in the `/api` path of the app.

After this 3 changes the `express.js` file will look like this:

{% highlight javascript %}
import express from 'express';
import routes from '../server/routes';

const app = express();

// mount all routes on /api path
app.use('/api', routes);

export default app;
{% endhighlight %}

Now if you run `gulp nodemon` and call the `api-status` endpoint we defined, you should get an answer:

{% highlight bash %}
$ curl http://localhost:3000/api/api-status
{"status":"ok"}
{% endhighlight %}

### User Routes

Before we define our user routes, we need to add a middleware to our express app that will be very helpful called
[body parser](https://github.com/expressjs/body-parser). As the body parser docs state it allows you to:

> Parse incoming request bodies in a middleware before your handlers, available under the req.body property.
  
So basically whenever a `PUT` or `POST` request arrives with data in the request body, it will parse it and put it as
a json object in the `req.body` field so that we can use that data in our route handlers. Let's install it!

{% highlight bash %}
npm install --save body-parser@^1.14.2
{% endhighlight %}

And to configure it, add three more lines to the `config/express.js` file, so it will look like this:

{% highlight javascript %}
import express from 'express';
import bodyParser from 'body-parser';
import routes from '../server/routes';

const app = express();

app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: true }));

// mount all routes on /api path
app.use('/api', routes);

export default app;
{% endhighlight %}

Here we configure two parsers, one for json request bodies (where `Content-Type` is `application/json`) and one for
urlencoded (where `Content-Type` is `x-ww-form-urlencoded`). 

#### The Users Route Controller

All set now, we're ready to define our first set of RESTful endpoints to manage users. Let's create a `users.js` file
within the `server/controllers` directory and put the following contents on it:

{% highlight javascript %}
import User from '../models/user';

function load(req, res, next, id) {
  User.findById(id)
    .exec()
    .then((user) => {
      req.dbUser = user;
      return next();
    }, (e) => next(e));
}

function get(req, res) {
  return res.json(req.dbUser);
}

function create(req, res, next) {
  User.create({
      username: req.body.username,
      password: req.body.password
    })
    .then((savedUser) => {
      return res.json(savedUser);
    }, (e) => next(e));
}

function update(req, res, next) {
  const user = req.dbUser;
  Object.assign(user, req.body);

  user.save()
    .then((savedUser) => res.sendStatus(204),
      (e) => next(e));
}

function list(req, res, next) {
  const { limit = 50, skip = 0 } = req.query;
  User.find()
    .skip(skip)
    .limit(limit)
    .exec()
    .then((users) => res.json(users),
      (e) => next(e));
}

function remove(req, res, next) {
  const user = req.dbUser;
  user.remove()
    .then(() => res.sendStatus(204),
      (e) => next(e));
}

export default { load, get, create, update, list, remove };
{% endhighlight %}

This will be our users controller module, and it contains all the necessary code to implement the five endpoints I've 
described before in this same post. Let's briefly explain what each of the functions in this controller does:

- `load`: we'll use this controller function for all the requests that contain a `userId` on the path. This would be for
the `GET /users/userId`, `PUT /users/userId` and `DELETE /users/userId`. This will make things easier by loading the
user from the database and making it accessible in the request object as `req.dbUser`.
- `get`: implementation for the `GET /users/userId` endpoint that returns a specific user's data as json.
- `create`: implementation for the `POST /users` endpoint. It creates a new user document in our database by using the
request body data sent by the client.
- `update`: implementation for the `PUT /users/userId` endpoint. It also takes the data sent in the request body and uses
it to update an existing user. It returns `204 No Content` as recommended in the [RFC7231 standard](https://tools.ietf.org/html/rfc7231#section-4.3.4)
- `list`: implements the `GET /users` endpoint. It queries the database to list all the users but limits the results. Since
we might have thousands of users in our database, we don't want to send a huge payload through the network, so we need to
paginate the results in a way that API clients can get them by batches (of 50 users in our case).
- `remove`: implements the `DELETE /users/userId` endpoint and returns `204 No Content` when it's successful.

Notice that in case of errors happening while accessing the data, we simply call the next middleware with the corresponding
error as parameter (`next(e)`). We'll cover error handling later, so for now just ignore that part.

#### The Users Route Configuration

Now that we have a controller ready, we can create our route configuration file. So let's create a `users.js` file inside
the `server/routes` directory and put the following contents on it:

{% highlight javascript %}
import express from 'express';
import userCtrl from '../controllers/users';

const router = express.Router();

router.route('/')
  /** GET /api/users - Get list of users */
  .get(userCtrl.list)

  /** POST /api/users - Create new user */
  .post(userCtrl.create);

router.route('/:userId')
  /** GET /api/users/:userId - Get user */
  .get(userCtrl.get)

  /** PUT /api/users/:userId - Update user */
  .put(userCtrl.update)

  /** DELETE /api/users/:userId - Delete user */
  .delete(userCtrl.remove);

/** Load user when API with userId route parameter is hit */
router.param('userId', userCtrl.load);

export default router;
{% endhighlight %}

This one looks simple, so here we're defining two routes: `/` and `/:userId`. We'll mount this two routes under the 
`/users` path later, so they will become `/users` and `/users/:userId`. Under each of this routes, we define the different
verbs supported on that route, and we assign a controller function to each of them. So as an example, let's take the 
first one:

`router.route('/').get(userCtrl.list)`

This is telling express, that whenever a request comes to the `/users` path with the HTTP method `GET`, the controller
function `list` must be called to handle the request. You can guess the rest of the routes very easily.

There's one special configuration in the end:

`router.param('userId', userCtrl.load)`

This tells express that whenever an incoming request has the `userId` parameter in the path, it should call first the
controller function `load` and then pass to the corresponding handler. So if for example the server gets the request:
`GET /users/some-user-id-here`, the server will first call the `userCtrl.load` function and after that it will call the
`userCtrl.get` function.

### Testing the Endpoints

All set, let's grab a console and try our set of endpoints. Let's start by creating a new user with our `POST /users`
endpoint:

{% highlight bash %}
# POST /users
$ curl -X POST http://localhost:3000/api/users \ 
    -d username=test_user \
    -d password=hello_world 
{% endhighlight %}

And the response should be something like this:

{% highlight json %}
{
    "__v":0,
    "username":"test_user",
    "password":"$2a$10$stMU9L4tiQBVTz1ng1Yi0uqvIrHAL...",
    "_id":"579949227038c8e0b2e399a7"
}
{% endhighlight %}

And if we request a list of users with our `GET /users` endpoint, the response will be similar but wrapped in an array:

{% highlight bash %}
# GET /users
$ curl http://localhost:3000/api/users
{% endhighlight %}
{% highlight json %}
[
  {
    "__v":0,
    "username":"test_user",
    "password":"$2a$10$stMU9L4tiQBVTz1ng1Yi0uqvIrHAL...",
    "_id":"579949227038c8e0b2e399a7"
  }
]
{% endhighlight %}


We have a new user with `_id` value `579949227038c8e0b2e399a7`. So let's use that id to try out other endpoints:

{% highlight bash %}
# GET /users/:userId
$ curl http://localhost:3000/api/users/579949227038c8e0b2e399a7
{% endhighlight %}
{% highlight json %}
{
    "__v":0,
    "username":"test_user",
    "password":"$2a$10$stMU9L4tiQBVTz1ng1Yi0uqvIrHAL...",
    "_id":"579949227038c8e0b2e399a7"
}
{% endhighlight %}

Now let's update the username property of our only user:

{% highlight bash %}
# PUT /users/:userId
$ curl -X PUT http://localhost:3000/api/users/579949227038c8e0b2e399a7 \
    -d username=test_user_changed
{% endhighlight %}

In this case the response is just the status `204 No Content`, so we won't get anything back

Finally let's remove the user with the `DELETE /users/:userId` endpoint:
 
{% highlight bash %}
# DELETE /users/:userId
$ curl -X DELETE http://localhost:3000/api/users/579949227038c8e0b2e399a7 
{% endhighlight %}

And again, no answer for the `DELETE` operations.

### Task Routes

The routes configuration and implementation for tasks will be very similar, so I'll just put here the contents for the
`routes/tasks.js` and `controllers/tasks.js` files, if something's not clear feel free to ask in the comments!

#### The Task Routes Controller

Create a `tasks.js` file inside the `server/controllers` directory and put the following contents on it:

{% highlight javascript %}
import Task from '../models/task';

function load(req, res, next, id) {
  Task.findById(id)
    .exec()
    .then((task) => {
      req.dbTask = task;
      return next();
    }, (e) => next(e));
}

function get(req, res) {
  return res.json(req.dbTask);
}

function create(req, res, next) {
  Task.create({
      user: req.body.user,
      description: req.body.description
    })
    .then((savedTask) => {
      return res.json(savedTask);
    }, (e) => next(e));
}

function update(req, res, next) {
  const task = req.dbTask;
  Object.assign(task, req.body);

  task.save()
    .then(() => res.sendStatus(204),
      (e) => next(e));
}

function list(req, res, next) {
  const { limit = 50, skip = 0 } = req.query;
  Task.find()
    .skip(skip)
    .limit(limit)
    .exec()
    .then((tasks) => res.json(tasks),
      (e) => next(e));
}

function remove(req, res, next) {
  const task = req.dbTask;
  task.remove()
    .then(() => res.sendStatus(204),
      (e) => next(e));
}

export default { load, get, create, update, list, remove };
{% endhighlight %}

#### The Task Routes Configuration

Create a `tasks.js` file inside the `server/routes` directory and put the following contents on it:

{% highlight javascript %}
import express from 'express';
import taskCtrl from '../controllers/tasks';

const router = express.Router();

router.route('/')
  /** GET /api/tasks - Get list of tasks */
  .get(taskCtrl.list)

  /** POST /api/tasks - Create new task */
  .post(taskCtrl.create);

router.route('/:taskId')
  /** GET /api/tasks/:taskId - Get task */
  .get(taskCtrl.get)

  /** PUT /api/tasks/:taskId - Update task */
  .put(taskCtrl.update)

  /** DELETE /api/tasks/:taskId - Delete task */
  .delete(taskCtrl.remove);

/** Load task when API with taskId route parameter is hit */
router.param('taskId', taskCtrl.load);

export default router;
{% endhighlight %}

Finally add a two new lines to `server/routes/index.js` to mount the tasks routes in the `/tasks` path:

{% highlight javascript %}
...
import taskRoutes from './tasks';
...
router.use('/tasks', taskRoutes);
{% endhighlight %}


## Coming up next...

We have our endpoints up and running, now it's time to add some security to our RESTful API. We'll see how to add 
authentication via [JSON Web Tokens](http://jwt.io) in the [next post]({% post_url 2016-07-22-building-a-node-restful-api-intro-and-setup  %})





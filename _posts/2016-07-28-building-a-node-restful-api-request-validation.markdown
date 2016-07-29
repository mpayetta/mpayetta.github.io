---
layout: post
title:  "Building a Node.js REST API 7: Request Validation"
date:   2016-07-27 12:56:45
categories: Node.js
banner_image: "/media/jwt.jpg"
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
- [5. The Endpoints]({% post_url 2016-07-26-building-a-node-restful-api-the-endpoints %})
- [6. Security with JWT]({% post_url 2016-07-27-building-a-node-restful-api-security-jwt %})

If you followed the steps from the previous posts your project directory structure should look like this:

{% highlight bash %}
├── config
│   ├── env
│   │   ├── development.js
│   │   └── index.js
│   ├── express.js
│   └── jwt.js
├── gulpfile.babel.js
├── index.js
├── package.json
└── server
    ├── controllers
    │   ├── auth.js
    │   ├── tasks.js
    │   └── users.js
    ├── models
    │   ├── task.js
    │   └── user.js
    └── routes
        ├── auth.js
        ├── index.js
        ├── tasks.js
        └── users.js
{% endhighlight %}

If it doesn't, please go back and check if you missed something from the previous posts.

## Request Payload Validation

<!--from-->
We have now a set of endpoints to manage users and tasks, and secured with JWT to avoid unknown clients to access our
API. Now we'll have a look at how can we add some validation to the request payload sent by clients. Since we want to
keep our database consistent, it's good to check what the user is sending us. You can do validations in different levels.
In the top level, you can validate data types and format of parameters, and in the next level you can validate that
a resource ID sent by the user exists in the database.
<!--to-->

During this post we'll focus on the first level, which is validating the data types and the format of the data sent
by the user. The second level explained before can be easily done with a database query at the beginning of your endpoint
controller.

We'll be using a few handy modules for the validation purpose:

{% highlight bash %}
npm install --save express-validation@^0.4.5 joi@^7.2.3
{% endhighlight %}

- [`express-validation`](https://www.npmjs.com/package/express-validation) provides a middleware function that can 
validate the request payload data given a set of rules provided by us.
- [`joi`](https://www.npmjs.com/package/joi) it's the module we'll use to define those rules

### The Validation Rules

Let's start by creating our validation rules. I'll explain how to create this rules for the tasks endpoints, you can
replicate easily the same approach for the rest of your API routes.

Go to the `server/routes` directory and create there a new directory called `validation`. Here we'll store all the
validation rules for our routes. The rule will be same name for both route and validation file. In our case we're
writing validations for the `server/routes/tasks.js` route file, so we'll create a `server/routes/validation/tasks.js`
file. Once created, put the following contents on it:

{% highlight javascript %}
import Joi from 'joi';

export default {
  // POST /api/tasks
  createTask: {
    body: {
      user: Joi.string().regex(/^[0-9a-fA-F]{24}$/).required(),
      description: Joi.string().required(),
      done: Joi.boolean()
    }
  },

  // GET-PUT-DELETE /api/tasks/:taskId
  getTask: {
    params: {
      taskId: Joi.string().regex(/^[0-9a-fA-F]{24}$/).required()
    }
  },

  // PUT /api/tasks/:taskId
  updateTask: {
    body: {
      user: Joi.string().regex(/^[0-9a-fA-F]{24}$/),
      description: Joi.string(),
      done: Joi.boolean()
    }
  }
};

{% endhighlight %}

Here we use some of the utilities provided by `joi` to validate fields. Joi has a very simple but complete API to do
validations of request data. Let's take the `createTask` example:

- `body.user`: we define that the user parameter in the request body must be a `string`, must match the provided regular
expression (which is just a regex to match MongoDB ObjectIds) and it's `required`.
- `body.description`: we define that the description parameter in the request body must be a `string` and is `required`.
- `body.done`: we define the done parameter as a `boolean`. Note that we don't place the `required` validation since this 
field is not required by our API and will have a default value of `false` in case it's missing.
 
Simple right? If you have a look at the `getTask` definition, we wrote a different set of rules called `params`:

- `body` rules will validate the data sent with the request payload
- `params` rules will validate the data sent as request path parameters

The `getTask` validation will run for all the endpoints that have a `taskId` path parameter.
There are three other kind of rules available which we're not using:

- `query` will validate the data sent in the request query parameters (things after the first `?` in the URL)
- `header` will validate the data sent in the reuqest headers.
- `cookies` will validate data sent on the request cookies.

This validation groups (body, params, query, header, cookies) are defined by the other library we'll be using (express-validation).
If you want a full set of validations provided by Joi you can check the module [API page](https://github.com/hapijs/joi/blob/v9.0.4/API.md).

### The Validation Middleware

Now that we have our set of rules defined, it's time to use them in our routes. So open the `server/routes/tasks.js` file
and let's add a few lines on it:

{% highlight javascript %}
import express from 'express';
import validate from 'express-validation';
import taskCtrl from '../controllers/tasks';
import validations from './validation/tasks';

const router = express.Router();

router.route('/')
  /** GET /api/tasks - Get list of tasks */
  .get(taskCtrl.list)

  /** POST /api/tasks - Create new task */
  .post(validate(validations.createTask),
        taskCtrl.create);

router.route('/:taskId')
  /** GET /api/tasks/:taskId - Get task */
  .get(taskCtrl.get)

  /** PUT /api/tasks/:taskId - Update task */
  .put(validate(validations.updateTask),
        taskCtrl.update)

  /** DELETE /api/tasks/:taskId - Delete task */
  .delete(taskCtrl.remove);

/** Load task when API with taskId route parameter is hit */
router.param('taskId', validate(validations.getTask));
router.param('taskId', taskCtrl.load);

export default router;
{% endhighlight %}

The first thing added is the two `import` statements, one for `express-validation` and other for the validation rules
created in the step before.

Then let's have a look at the end of the file, where we added an extra middleware for the `router.param('taskId')` matcher.
Here is where we define the validation of the `taskId` path parameter. If the `taskId` does not pass the validation, then
the API will answer with `400 Bad Request` and stop the middleware execution chain.

There are two other changes done in the route configuration file:

1. In the `POST /tasks` endpoint, where we add the validation middleware before the endpoint controller function.
2. In the `PUT /tasks/taskId` endpoint, where we add the validation before the endpoint controller function as well. In 
this case, two validation middlewares will be executed. The first one will be the one that validates the `taskId` path
parameter, and the second one will validate the request body data sent to update the corresponding task information.

All set, let's try the endpoints with some invalida data and see what happens. Grab a console and throw some requests on it:

{% highlight bash %}
$ curl http://localhost:3000/api/tasks/invalid-task-id
{% endhighlight %}
{% highlight json %}
{ "status": 400, "message": "validation error" }
{% endhighlight %}

Good! We got back our expected `400 Bad Request`. But we can do a lot better with error messages. Just telling the 
client there's a `validation error` is not enough. Imagine when you have complex endpoints where lots of data has to
be sent in the request body. The client would need to check one by one the fields to see which one is incorrect.

Let's go to the `config/express.js` server and tune up a bit the error handling middleware:

{% highlight javascript %}
...
import expressValidation from 'express-validation';
...

app.use((err, req, res, next) => {
  if (err instanceof expressValidation.ValidationError) {
    res.status(err.status).json(err);
  } else {
    res.status(err.status)
      .json({
        status: err.status,
        message: err.message
      });
  }
});
{% endhighlight %}

So basically we're delegating the error message details to the express validation module which can provide detailed 
information about the error. If you try again the same request, you'll get something like this now:

{% highlight bash %}
$ curl http://localhost:3000/api/tasks/invalid-task-id
{% endhighlight %}
{% highlight json %}
{
   "status":400,
   "statusText":"Bad Request",
   "errors":[
      {
         "field":"taskId",
         "location":"params",
         "messages":[
            "\"taskId\" with value \"invalid-task-id\" fails to match the required pattern: /^[0-9a-fA-F]{24}$/"
         ],
         "types":[
            "string.regex.base"
         ]
      }
   ]
}
{% endhighlight %}

Much better! Now clients can easily understand what's wrong with their requests. Now you can add validations to all of
your endpoints.


## Coming up next...

We're almost done with our API, the last post will be dedicated to unit testing our endpoints to prevent failures with
future changes. So if you're ready for it, head over to the [next post]({% post_url 2016-07-22-building-a-node-restful-api-intro-and-setup  %})!

<div class="cc">
    <a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
        <img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc/4.0/88x31.png" />
    </a>
    <br/>
    <span xmlns:dct="http://purl.org/dc/terms/" href="http://purl.org/dc/dcmitype/Text" property="dct:title" rel="dct:type">
        Building a Node.js REST API 7: Request Validation
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




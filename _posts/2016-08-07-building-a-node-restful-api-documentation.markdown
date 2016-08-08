---
layout: post
title:  "Building a Node.js REST API (Bonus Track)"
subtitle: "API Documentation"
date:   2016-08-07 11:30:45
categories: Node.js
banner_image: "/media/apidoc.jpg"
banner_idx: "/media/apidoc_idx.jpg"
featured: true
comments: true
---

## Introduction

<!--from-->
Having a good API documentation is crucial to let the clients understand how our API behaves and what does it expect
form every request. We'll see in this post how can we introduce into our Gulp build pipeline the generation of an
API docs static website with the help of a library called [apidoc](http://apidocjs.com/).
<!--to-->

If you want to build the API from scratch, please head over to the previous posts:

- [1. Intro and Initial Setup]({% post_url 2016-07-22-building-a-node-restful-api-intro-and-setup  %})
- [2. Setting up the Web Server]({% post_url 2016-07-23-building-a-node-restful-api-the-web-server  %})
- [3. The Database: Setup]({% post_url 2016-07-24-building-a-node-restful-api-the-database  %})
- [4. The Database: Connect and Config]({% post_url 2016-07-25-building-a-node-restful-api-the-database-2  %})
- [5. The Endpoints]({% post_url 2016-07-26-building-a-node-restful-api-the-endpoints %})
- [6. Security with JWT]({% post_url 2016-07-27-building-a-node-restful-api-security-jwt %})
- [7. Request Validation]({% post_url 2016-07-28-building-a-node-restful-api-request-validation %})
- [8. Unit Testing]({% post_url 2016-07-29-building-a-node-restful-api-unit-testing %})


## Inline Docs with Comments 

The library we'll use will parse our javascript files searching for comments with a specific syntax. Then it will parse
the comments and generate a full static website that will look like the one in [this example](http://apidocjs.com/example/).
Basically we can group our endpoints into groups, and each endpoint will have full documentation including endpoint
parameters, headers, responses, response examples, etc. Of course all of this has to be written by us in the inline comments,
but I'll show you also how we can define in a separate file all the things that are reused in all endpoints (like the
authorization header, no content responses, etc).

Let's start by installing the gulp dependency:

{% highlight bash %}
npm install --save-dev gulp-apidoc
{% endhighlight %}

There's also a plain node client for apidoc, but since we are already using Gulp it makes sense to manage the docs creation
in Gulp as well.

The basic building block of an endpoint documentation looks like this:
 
{% highlight javascript %}
/**
 * @api {get} /api/users List all users
 * @apiName GetUsers
 * @apiGroup Users
 **/
{% endhighlight %}

All endpoint docs must start with `@api`, followed by the HTTP method and a short description. Then we have to give the
endpoint an `@apiName` which must be unique among all endpoints. Finally we tell the library in which group we want to 
have this endpoint listed.

So let's start by adding some docs to the users listing endpoint. Head over to the route definition (you can place this
docs anywhere, but I think it makes sense to document the endpoint in the same place we define it in our app). Then add
the same snippet of code above the `.get(auth, userCtrl.list)` line:

{% highlight javascript %}
router.route('/')
  /**
   * @api {get} /api/users List all users
   * @apiName GetUsers
   * @apiGroup Users
   **/
  .get(auth, userCtrl.list)
{% endhighlight %}

Now let's see how it looks like in the generated docs (don't worry, we'll add more information to the api docs later).
We need to compile our docs and for that we need a gulp task.

## The apidoc Gulp Task

Let's add a gulp task that will create our apidocs based on the comments we have in the code. Open the `gulpfile.babel.js`
file and add the following `apidoc` task:

{% highlight javascript %}
gulp.task('apidoc', (done) => {
  plugins.apidoc({
    src: 'server/routes/',
    dest: 'docs/'
  }, done);
});
{% endhighlight %}

Basically we tell the plugin that it must look for api docs in the `server/routes` directory and output the documentation
into the `docs` directory. You'll want to add the `docs` dir to your `.gitignore` file. So go ahead and run the task to
see what happens:
 
{% highlight bash %}
gulp apidoc
[11:00:20] Requiring external module babel-register
[11:00:21] Starting 'apidoc'...
warn: Please create an apidoc.json configuration file.
info: Done.
[11:00:22] gulp-apidoc Apidoc created...   [    ] 
[11:00:22] Finished 'apidoc' after 386 ms
{% endhighlight %}

You should now have a `docs` directory with a full static website. Open the `index.html` file and you should see something
like this:

<img class="full" src="/media/posts/apidocs-1.jpg">

Empty but we know our gulp api task is working fine. Now it's time to add more details to the docs. If you checked the
output of the console when running the gulp task, you probably noticed this line:

{% highlight bash %}
warn: Please create an apidoc.json configuration file.
{% endhighlight %}

This `apidoc.json` file will contain some basic API info like a title, description, version, etc. Let's create it in the
root of the project and add some contents to it:

{% highlight json %}
{
  "name": "Node ES6 API",
  "version": "0.1.0",
  "description": "An example API built with Node and ES6",
  "title": "Node ES6 API Docs",
  "url" : "https://github.com/mpayetta/node-es6-rest-api"
}
{% endhighlight %}

On the github line you should put your repo URL. Also we need to add an extra configuration to the gulp task, because
by default the gulp-apidoc plugin will search for the apidoc.json in the source directory. In our case it will search
for the apidoc.json in `server/routes` and we don't want to put a config file there. So your gulp task should look like
this now:

{% highlight javascript %}
gulp.task('apidoc', (done) => {
  plugins.apidoc({
    src: 'server/routes/',
    dest: 'docs/',
    config: ''
  }, done);
});
{% endhighlight %}

We override the `config` property to an empty string which will make the plugin look for the configuration file in the
root of our project. Now generate the docs again by running `gulp apidoc` and you should see a bit more of info like
the title, version and description:

<img class="full" src="/media/posts/apidocs-2.jpg">

All set, let's add more details to the docs now.

## Detailed API Documentation

### Re-usable definitions

The first thing we'll do is creating the re-usable pieces of our API documentation. We're going to put this data in
a file called `_apidoc.js` within the routes directory. So create the `server/routes/_apidoc.js` file and put the following
contents on it:

{% highlight javascript %}
// -----------------------
// Headers
// -----------------------

/**
 * @apiDefine AuthorizationHeader
 *
 * @apiHeader {Object} Authorization header value must follow the pattern
 * "JWT [token sting]"
 *
 * @apiHeaderExample {json} Authorization Header Example:
 *    {
 *      "Authorization": "JWT eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ..."
 *    }
 *
 */

// -----------------------
// Error Responses
// -----------------------

/**
 * @apiDefine NotAuthorizedError
 *
 * @apiError Unauthorized The JWT is missing or not valid.
 *
 * @apiErrorExample Unauthorized Response:
 *     HTTP/1.1 401 Unauthorized
 *     {
 *        "status": 401,
 *        "message": "No authorization token was found"
 *     }
 */

/**
 * @apiDefine InternalServerError
 *
 * @apiError InternalError There was an internal error when trying to serve the request
 *
 * @apiErrorExample InternalError Response:
 *     HTTP/1.1 500 Internal Server Error
 *     {
 *       "status": 500,
 *       "message": "There was an internal error when trying to serve the request"
 *     }
 */


// -----------------------
// Success Responses
// -----------------------

/**
 * @apiDefine NoContentResponse
 * @apiDescription Empty successful response
 *
 * @apiSuccessExample NoContent Response:
 *    HTTP/1.1 204 No Content
 */
{% endhighlight %}

We defined three things here: Headers, Error Responses and Success Responses. You can see that we can define as many
things as we want to re-use later with the `@apiDefine DefinitionID` instruction which we can later reference by the
given `DefinitionID`. 

Now that we have this defined, let's see how can we use it in our users listing endpoint:

{% highlight javascript %}
router.route('/')
  /**
   * @api {get} /api/users List all users
   * @apiName GetUsers
   * @apiGroup Users
   *
   * @apiUse AuthorizationHeader
   *
   * @apiUse NotAuthorizedError
   * @apiUse InternalServerError
   **/
  .get(auth, userCtrl.list)
{% endhighlight %}

As you can see, by using the `@apiUse ID` instruction, we can reference all the definitions we created in the `_apidoc.js`
file. Now compile the api docs and you'll get something like this:

<img class="full" src="/media/posts/apidocs-3.jpg">

Nice! Our api docs are starting to look like some professional documentation! There's another error response that can be
re-used but that only applies to our users routes so far: the User Not Found response. Whenever we get a request starting
with `/users/:userId`, our API will look for a user with that ID, and if it doesn't exist, we should return a `404 Not Found`
error. So let's first add this to our controller in `controllers/user.js`:

{% highlight javascript %}
function load(req, res, next, id) {
  User.findById(id)
    .exec()
    .then((user) => {
      if (!user) {
        return res.status(404).json({
          status: 400,
          message: "User not found"
        });
      }
      req.dbUser = user;
      return next();
    }, (e) => next(e));
}
{% endhighlight %}

And now we'll add a definition for the user not found error in our apidocs. Since this error belongs only to the user routes,
it's better to place the definition in the `routes/users.js` file instead of the `_apidoc.js`.:

{% highlight javascript %}
import express from 'express';
import userCtrl from '../controllers/users';
import auth from '../../config/jwt';

const router = express.Router();

/**
 * @apiDefine UserNotFoundError
 *
 * @apiError UserNotFound The id of the User was not found.
 *
 * @apiErrorExample UserNotFound Response:
 *     HTTP/1.1 404 Not Found
 *     {
 *       "status": 400,
 *       "message": "User not found"
 *     }
 */
{% endhighlight %}

### Documenting the endpoint

Now that we have our definitions ready we're set to use them and complete our docs. 
Besides error responses and headers, another very important part of API documentation is the success response and 
providing examples of it, so that clients get more familiarized with what our API will be returning on successful requests.

Let's complete the users listing endpoint documentation with the success response and an example:
 
{% highlight javascript %}
router.route('/')
  /**
   * @api {get} /api/users List all users
   * @apiName GetUsers
   * @apiGroup Users
   *
   * @apiUse AuthorizationHeader
   *
   * @apiSuccess {Object[]} users List of users
   * @apiSuccess {String} users._id ID of the user
   * @apiSuccess {String} users.username Username of the user, usually same as mobile
   * @apiSuccess {String} users.password Encrypted password of the user.
   * username/password combination.
   *
   * @apiSuccessExample Success Response:
   *     HTTP/1.1 200 Success
   *     [
   *       {
   *        "_id":
   *        "username": "a_user",
   *        "password": "$2a$10$4kGSUCjFpSSIGS6T3Vpb7O..."
   *       },
   *       {
   *        "_id":
   *        "username": "another_user",
   *        "password": "$2a$10$4kGSUCjFpSSIGS6T3Vpb7O..."
   *       }
   *     ]
   *
   * @apiUse NotAuthorizedError
   * @apiUse InternalServerError
   **/
  .get(userCtrl.list)
{% endhighlight %}

With the `@apiSuccessExample` instruction we can easily show an example response when things go well with the request.
All we have to do now is run again our `gulp apidoc` task and this will be the result:

<img class="full" src="/media/posts/apidocs-4.jpg">

## Conclusion

We've learned how to add documentation in a pretty straightforward way with the help of the apidoc library.
Now it's time for you to play around with the apidoc library and complete the full docs for your API. Remember all the times
you've been using a well documented API and how satisfying it was. You should do the same for your API users! :)

I hope you enjoyed reading, if so, please retweet or like this tweet!

<!--
<blockquote class="twitter-tweet" data-lang="es"><p lang="en" dir="ltr">The complete guide to write a RESTful API with Node and ES6 starts here! <a href="https://t.co/9Wj6iiS3Qc">https://t.co/9Wj6iiS3Qc</a></p>&mdash; Mauricio Payetta (@mpayetta) <a href="https://twitter.com/mpayetta/status/759583208501448704">31 de julio de 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>
-->

<div class="cc">
    <a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
        <img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc/4.0/88x31.png" />
    </a>
    <br/>
    <span xmlns:dct="http://purl.org/dc/terms/" href="http://purl.org/dc/dcmitype/Text" property="dct:title" rel="dct:type">
        Building a Node.js REST API (Bonus Track): API Documentation
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
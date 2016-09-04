---
layout: post
title:  "Building a Node.js REST API 6"
subtitle: "Authentication with JWT"
date:   2016-07-27 12:56:45
categories: Node.js RESTful JWT
banner_image: "/media/jwt.jpg"
banner_idx: "/media/jwt_idx.jpg"
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

You can find the code for this tutorial in [GitHub](https://github.com/mpayetta/node-es6-rest-api).

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
    ├── controllers
    │   ├── tasks.js
    │   └── users.js
    ├── models
    │   ├── task.js
    │   └── user.js
    └── routes
        ├── index.js
        ├── tasks.js
        └── users.js
{% endhighlight %}

If it doesn't, please go back and check if you missed something from the previous posts.

## RESTful API Security

<!--from-->
It's time to take care of a key point on our RESTful API: the security. We have our endpoints to manage users and tasks,
but right now any client that knows where our API is running can access the endpoints. This is not safe, in the real 
world you'd want to control who has access to your API and you need to check with every request that it's coming from
a trusted source.
<!--to-->

There are at least two security aspects you want to take care of when building an API: authentication and authorization. 
There's always someone who doesn't know the difference between these two, so here a one sentence explanation for both:

- Authentication: the action of identifying a user in your platform 
- Authorization: the action of checking if whether or not the already authenticated user has access to a specific resource

So you can basically see it as a pipeline: request -> authenticate -> authorize -> response. First you check that the request
is coming from a trusted source, and after that you check that the trusted source has access to the resource it's trying
to get from your system.

In our API authentication will be provided by a user/password sign in endpoint which will return back a JWT (JSON Web Token)
- more about this later on - that can be used to authenticate subsequent requests. The authorization will be simply 
handled by our controllers to do some basic checks like not allowing a user to see or change the tasks of other users.

For the purpose of this project we'll only take care of authentication which is the most complex. Authorization won't
be covered here but I'll just say that it can be achieved easily with database checks in the simplest implementations 
and user roles management in more complex systems.

## <a name="auth-steps"></a> API Authentication

So let's start with the authentication process for our API. I'll first list here the authentication steps and we'll go
into further details after. Basically the flow to access our API will be the following:

1. Client requests a JWT to authenticate it's requests: 
`POST /token -d username=user -d password=pass`
2. API Server verifies the username/password combination and generates a JWT to be returned. If user/password combination
is not valid a `401 Unauthorized` error will be returned.
3. Client stores somewhere the JWT (for example in the `localStorage` if it's a web browser).
4. Client sends requests to access resources including the JWT in the `Authorization` header of the request.
5. API verifies that the JWT is valid and returns the resource data or `401 Unauthorized` if it's not valid.

So it's time to clear the doubts in that JWT thing I mentioned a few times already in this post.

### JWT (JSON Web Token)

As stated in the [JWT.io website](https://jwt.io/introduction/):

> JSON Web Token (JWT) is an open standard ([RFC 7519](https://tools.ietf.org/html/rfc7519)) that defines a compact and 
self-contained way for securely transmitting information between parties as a JSON object.

Basically a JWT is a JSON object encoded in a base 64 String which contains information about an entity (user or API client). The good
thing is that this String is compact and it can be sent as a request `POST` parameter or even in a request header
(as we'll use it).

At this point you might be asking yourself how is it that we should trust an encoded string that the client is sending.
What if the client makes up the encoded string with fake data? The secret is that the JWT that the API server returns to
the client is digitally signed with a secret key known by the server only and using the [HMAC algorithm](http://www.drdobbs.com/security/the-hmac-algorithm/184410908).
This way, the server will always check if the signature is valid before analysing the JWT contents. So unless you give
your secret key to people, nobody without username/password authentication would be able to access your API.

The JWT encoded String is divided into 3 sections: Header, Payload and Signature. You can find more details about JWT
sections in the [JWT.io](https://jwt.io/) website.

### The ST in REST

So why all this JWT thing instead of doing some kind of log in endpoint and keep a session active for some predefined
time? That's when the ST part of REST comes into play. The meaning of REST is "Representational State Transfer". 
State Transfer means that in every HTTP request we transfer the state, we don't need to store the state in any server.
So the session status is always maintained by the clients, and clients are the ones sending session information to the
server with every request.

This has many advantages, but is specially important to scale our API. The fact of having no session data in our API
servers allows us to have multiple instances to serve as many clients as we want, and any instance can serve any client
at any time. There's no session data on the server, which means that different servers can take care of consecutive
requests from the same client. Because the client sends the session data in the request (the JWT), each server can
verify if it's a valid client or not with every request. A very good explanation can be found on [this](http://stackoverflow.com/a/3105337/1566669)
stack overflow answer by [Jarrod Roberson](http://stackoverflow.com/users/177800/jarrod-roberson).

## Implementing JWT 

Ok, now that we understand all this JWT thing let's see how can we implement it in our API to make it more secure.
We'll need to install the [`jsonwebtoken`](https://github.com/auth0/node-jsonwebtoken) module in order to generate JWT 
for valid users. We also need to install the [`express-jwt`](https://github.com/auth0/express-jwt) module which provides 
a useful middleware that will check if the JWT sent with the requests is valid or not.
 
{% highlight bash %}
npm install --save jsonwebtoken@^7.0.1 express-jwt@^3.4.0
{% endhighlight %}

### The Auth Controller

Let's first create the functions that will handle the initial authentication of a user. This is steps 1 and 2 of the
[authentication flow](#auth-steps). Go to the `server/controllers` directory and create an `auth.js` file.
We'll organize our authentication flow with three steps, which will be implemented by a controller function each:

1. `authenticate`: Will check for the username/password correctness 
2. `generateToken`: Will generate the JWT
3. `respondJWT`: Will send the JWT back to the client

For the step number 1, add the following code to the `server/controllers/auth.js` file you just created:

{% highlight javascript %}
import User from '../models/user';

function authenticate(req, res, next) {
  User.findOne({
      username: req.body.username
    })
    .exec()
    .then((user) => {
      if (!user) return next();
      user.comparePassword(req.body.password, (e, isMatch) => {
        if (e) return next(e);
        if (isMatch) {
          req.user = user;
          next();
        } else {
          return next();
        }
      });
    }, (e) => next(e))
}
{% endhighlight %}

Here we simply check first that a user with the provided `req.body.username` exists. If it exists, we then check if the
provided `req.body.password` is the correct one by using the `comparePassword` instance method we created in the previous
post ["The Database: Setup"]({% post_url 2016-07-24-building-a-node-restful-api-the-database  %}).
If password is correct we then call the next middleware for the route. 

Now let's add some code to generate a JWT for the authenticated user. In order to create a JWT, we'll need to define a
secret key which we'll use to sign the JWT and recognize later if the JWT sent by the client wasn't manipulated. So
we need to create a new configuration property in our `config/env/development.js` file named `jwtSecret`. And we will 
also add another configuration property for the JWT duration:

{% highlight javascript %}
export default {
  env: 'development',
  db: 'mongodb://localhost/node-es6-api-dev',
  port: 3000,
  jwtSecret: 'my-api-secret',
  jwtDuration: '2 hours'
};
{% endhighlight %}

After this we're ready to add some code to generate a JWT to our auth controller:

{% highlight javascript %}
import jwt from 'jsonwebtoken';
import config from '../../config/env';
import User from '../models/user';

...

function generateToken(req, res, next) {
  if (!req.user) return next();

  const jwtPayload = {
    id: req.user._id
  };
  const jwtData = {
    expiresIn: config.jwtDuration,
  };
  const secret = config.jwtSecret;
  req.token = jwt.sign(jwtPayload, secret, jwtData);

  next();
}
{% endhighlight %}

Note the two added imports on the top. Within this function we define our JWT payload which will include only the
user `_id` property, which is enough for our server to identify a user. Then we set the expiration date for the token
and we finally sign it using the secret defined in our configuration file. After signing we pass control to the next
middleware which will be the one sending the response back. So let's add the code for the last authentication 
middleware:

{% highlight javascript %}
function respondJWT(req, res) {
  if (!req.user) {
    res.status(401).json({
      error: 'Unauthorized'
    });
  } else {
    res.status(200).json({
      jwt: req.token
    });
  }
}

export default { authenticate, generateToken, respondJWT };
{% endhighlight %}

Note that I've added the export statement as well at the end of the controller.

All set in the controller side, let's configure the authentication route.

### The Auth Route

Create a new `auth.js` file within the `server/routes` directory and put the following contents on it:

{% highlight javascript %}
import express from 'express';
import authCtrl from '../controllers/auth';

const router = express.Router();

router.route('/token')
  /** POST /api/auth/token Get JWT authentication token */
  .post(authCtrl.authenticate,
    authCtrl.generateToken,
    authCtrl.respondJWT);

export default router;
{% endhighlight %}

Now let's hook the route definition under the `/auth` path on our api by adding to the `server/routes/index.js` file this
lines:

{% highlight javascript %}
import authRoutes from './auth';

router.use('/auth', authRoutes);
{% endhighlight %}

All set! We're ready to check if our API can generate a JWT and return it back to the client. Go to your console and try
it:

{% highlight bash %}
curl http://localhost:3000/api/auth/token -d username=mauricio -d password=mauricio
{% endhighlight %}
The answer should look like this:
{% highlight json %}
{ "jwt": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjU3OTlhNT..." }
{% endhighlight %}
And if you send a wrong password or username you'll get this:
{% highlight json %}
{ "error": "Unauthorized" }
{% endhighlight %}

You can even go to the [JWT.io Debugger](https://jwt.io/#debugger-io) and paste the JWT that your API is returning to 
see if it's correct. And you can also try putting the `jwtSecret` from the config file in the debugger to check if the 
signature is valid.

### Validating Incoming JWTs

So far we provided our clients with a JWT that they can use in subsequent requests, but we didn't add any code to 
validate if the JWT that the client is sending is valid or not. Here is where we'll use the `express-jwt` module
we installed a while ago. 

Go to the `config` directory and create a `jwt.js` file there with the following contents:
 
{% highlight javascript %}
import config from './env';
import jwt from 'express-jwt';

const authenticate = jwt({
  secret: config.jwtSecret
});

export default authenticate;
{% endhighlight %}

We're exporting the `jwt` function which is nothing else than an express middleware that checks for the incoming requests
`Authorization` header. By default it will check for values equal to `JWT [JWT_STRING]`, so a valid header would be for
example:

{% highlight bash %}
Authorization: JWT eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjU3OTlhNTcyMjg0ZD... 
{% endhighlight %}

Now let's use this middleware in our routes configuration to add the authentication layer. I'll use the `server/routes/users.js`
as an example, you can later add authentication to any endpoint you want to restrict to registered clients.

{% highlight javascript %}
import express from 'express';
import userCtrl from '../controllers/users';
import auth from '../../config/jwt';

const router = express.Router();

router.route('/')
  /** GET /api/users - Get list of users */
  .get(auth, userCtrl.list);
{% endhighlight %}

That's it! By simply adding the authentication middleware at the beginning of the route middleware list you're done.
Now try to access the users listing endpoint without sending an Authorization header and you'll get a response like 
this one:

{% highlight bash %}
UnauthorizedError: No authorization token was found<br> &nbsp; &nbsp;at middleware ....
{% endhighlight %}

Good that we have protected our endpoint, but that response doesn't look very nice right? Let's make things right. Go 
to the `config/express.js` file and add the following error handling middleware after the `app.use('/api', routes);`
statement:

{% highlight javascript %}
app.use((err, req, res, next) => {
  res.status(err.status)
    .json({
      status: err.status,
      message: err.message
    });
});
{% endhighlight %}

This will take care of all errors carried by the calls to `next(err)` in our controller functions. Now try again to access
the user listing endpoint without a JWT, the response will look like this:

{% highlight json %}
{ "status": 401, "message": "No authorization token was found" }
{% endhighlight %}

Better right? So that's it, you can now add the `auth` middleware to any route you want to protect it from unauthorized
users.


## Coming up next...

We have our endpoints secured with a session-less approach with the help of JWT. We're almost there, the last two things
we have to do are adding validation of data to our endpoints and add some unit testing with mocha. To continue with
the validation head over to the [next post]({% post_url 2016-07-28-building-a-node-restful-api-request-validation  %})!

If you enjoyed reading, retweet or like this tweet!

<blockquote class="twitter-tweet" data-lang="es"><p lang="en" dir="ltr">Adding security to our RESTful API through the use of JSON Web Tokens <a href="https://t.co/65STooJf9t">https://t.co/65STooJf9t</a></p>&mdash; Mauricio Payetta (@mpayetta) <a href="https://twitter.com/mpayetta/status/760614577981698050">2 de agosto de 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

<div class="cc">
    <a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
        <img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc/4.0/88x31.png" />
    </a>
    <br/>
    <span xmlns:dct="http://purl.org/dc/terms/" href="http://purl.org/dc/dcmitype/Text" property="dct:title" rel="dct:type">
        Building a Node.js REST API 6: Authentication
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




---
layout: post
title:  "Building a Node.js RESTful API: The Database"
date:   2016-07-24 12:56:45
categories: Node.js
banner_image: "/media/mongodb.jpeg"
featured: true
comments: true
---

## Introduction

Welcome again! If you reached this post from nowhere and don't understand what all this is about, you can check out
the previous posts: 

- [Intro and Initial Setup]({% post_url 2016-07-22-building-a-node-restful-api-intro-and-setup  %})
- [Setting up the Web Server]({% post_url 2016-07-23-building-a-node-restful-api-the-web-server  %})

<!--more-->

## Setting up the DB

Ok, if everything went well, you should have `package.json` file on your project folder and it should look like the
example one shown to you by the `npm init` wizard. After this, there are lots of things to be done to have our project
setup and running. So let's get started by setting up our database.

### MongoDB

We'll be storing all our tasks data into a MongoDB database. The reasons I choose MongoDB are simple: I'm familiar with
it, it makes a great couple with Node.js and lastly because we won't have a super complex application. We're just building
a task manager, so there's no need to worry about atomic operations, performance when querying related entities, etc.
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
npm install --save mongoose@4.3.7
{% endhighlight %}

With mongoose installed we're ready to start defining our models. Mongoose models provide a simple way to define what
kinds of documents our database will have, and they include by default multiple functions to access and modify the data
as well. We can say that one mongoose model will map to a MongoDB collection. You can find a lot of details about 
mongoose as well as it's APIs in the [project website](http://mongoosejs.com/).

### The User Model

So let's begin with our first model implementation. Before starting let's create a few directories in our project to
keep things organized. First we'll create a `server` directory. This directory will contain all our server code, including
our models (and routes, controllers, tests, etc...). Then move inside the `server` directory and create a new directory called
`models`. As you imagine, this directory will hold all our Mongoose models.

{% highlight bash %}
# cd to the project root and then:
mkdir server
cd server
mkdir models
{% endhighlight %}

Good, we're all set to create our first model file: `user.js`. So go ahead and create it under the `models` directory we
created before and put the following code inside:

{% highlight javascript %}
import mongoose from 'mongoose';

const UserSchema = new mongoose.Schema({
  username: {
    type: String, 
    required: true, 
    trim: true
  },
  password: { 
    type: String, 
    required: true, 
    trim: true 
  }
});

export default mongoose.model('User', UserSchema);
{% endhighlight %}

Simple, right? So we have a very basic User model, in a real world application users would have many other properties, but
since our goal is just to understand how to build a RESTful API, we're not focusing on user properties. So with a 
`username` and a `password` property is enough. Both properties are `required` and `trim`med before being saved.

If we want our database to be secure, we want to protect our users data in case someone gets access to it. So we don't 
want to store the user password as a literal string, we should (I'd say must) encrypt it before saving it in the DB. This is
what we're going to do next, and for this purpose we need to install another handy module called [`bcrypt`](https://www.npmjs.com/package/bcrypt)

{% highlight bash %}
npm install --save bcrypt@^0.8.7
{% endhighlight %}

Now let's add a new `import` statement to use `bcrypt` and add the following code to our model:

{% highlight javascript %}
import bcrypt from 'bcrypt';

...

UserSchema.pre('save', function (next) {
  const user = this;

  if (!user.isModified('password')) {
    return next();
  }
  bcrypt.genSalt(10, (err, salt) => {
    if (err) return next(err);
    bcrypt.hash(user.password, salt, (hashErr, hash) => {
      if (hashErr) return next(hashErr);  
      user.password = hash;
      next();
    });
  });
});
{% endhighlight %}

Let's have a look at what's happening here. First we need to understand what does this `pre('save', fn)` thing does.
This defines what Mongoose calls [`middleware`](http://mongoosejs.com/docs/middleware.html) (or `hooks` or `pre`). And
as defined by Mongoose:

> "Middleware are functions which are passed control during execution of asynchronous functions"

Simply said, middleware will be executed in a specific order (specified by mongoose) whenever the defined action is 
executed in the model. In our case, we define a `pre` middleware that will trigger always before the `save` operation
is executed in the user model. So whenever we try to store (as a new document or as an update) any user, the middleware
function will be run before the data is persisted.

Now let's see what our middleware function does:

1. Check if the user password field has been modified. This will always be true for new users. If the password didn't
change then there's nothing we should do, so we can call the `next` function to pass control to the next middleware in the
list. On the other hand, if the password was changed, we move to the next step.
2. Generate a [salt](https://en.wikipedia.org/wiki/Salt_(cryptography)) with `bcrypt`. 
I'm not going to explain a lot about how bcrypt works (in case you're really into it
[here is the paper](https://www.usenix.org/legacy/events/usenix99/provos/provos.pdf) where it was presented). Basically
bcrypt is safe because it's super slow. You might think I'm crazy now... but yes, bcrypt was designed to be slow and to 
make it's "slowness" configurable. This allows the encryption algorithm to be slowed down in case newer technology and
supercomputers are developed that allow hackers to try millions of passwords a second. The good thing is that it's slow
enough to avoid hackers, but it's fast enough to don't let our users have a nap while waiting for their password to be
hashed.
3. With the generated salt, we hash the original password using `bcrypt.hash(password, salt, cb)`. If everything goes
well, the callback will be called with the hashed password (which contains the salt on it as well so we don't have to
store it separately). We can now replace the original password with the hashed one and move on to the next middleware
function by calling `next()`.

Good, we have our passwords safely stored in the database. But how do we authenticate users now? We can't do a simple 
string comparison, as the user will input a literal password and we have a hashed password in the db. We'll need to
add what Mongoose calls [instance methods](http://mongoosejs.com/docs/guide.html). So grab your user model code and 
add the following snippet before the `export` statement:

{% highlight javascript %}
UserSchema.methods.comparePassword = function (toCompare, done) {
  bcrypt.compare(toCompare, this.password, (err, isMatch) => {
    if (err) done(err);
    else done(err, isMatch);
  });
};
{% endhighlight %}

That's it, now whenever we want to check if the provided password is correct, we can just call our instance method
on the target user and provide a callback function. This callback will be called with an error if there was any and the
comparison result (boolean) as the second argument.

All set with users, not let's move to the next model.

### The Task Model

In the same `models` directory create a new javascript file for our tasks model and put the following code inside:

{% highlight javascript %}
import mongoose from 'mongoose';

const TaskSchema = new mongoose.Schema({
  user: {
    type: mongoose.Schema.Types.ObjectId,
    required: true,
    ref: 'users'
  },
  description: {
    type: String,
    required: true,
    trim: true
  },
  done: {
    type: Boolean,
    default: false
  }
});

export default mongoose.model('Task', TaskSchema);
{% endhighlight %}

The schema itself is quite simple and self-explanatory. The important part here is the `user` property: we're defining
it's type as `mongoose.Schema.Types.ObjectId`, which let us make a reference to the `users` collection. A simple note here:
we defined our users model as `'User'`, but we use the plural in the `ref` field. This is because as explained in
[mongoose docs](http://mongoosejs.com/docs/models.html):

> Mongoose automatically looks for the plural version of your model name. Thus, for the example above, the model Tank is for the tanks collection in the database.

So basically we're telling mongoose that the field `user` in our `tasks` collection documents, will be referencing the
`_id` field of the `users` collection. This allows us to map every task to a specific user in our system.

The rest of the model doesn't need explanation, besides we're using the `default: false` value for the `done` field.

## Coming up next...

Now that we have our database models ready, it's time to connect to the DB and start playing with it. But that will be
part of the [next post]({% post_url 2016-07-25-building-a-node-restful-api-the-database-2  %}). Thanks for reading!



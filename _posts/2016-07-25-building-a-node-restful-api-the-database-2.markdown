---
layout: post
title:  "Building a Node.js REST API 4: DB Connect and Config"
date:   2016-07-25 12:56:45
categories: Node.js
banner_image: "/media/mongodb2.jpg"
featured: true
comments: true
---

<!--more-->

## Introduction

Welcome again! If you reached this post from nowhere and don't understand what all this is about, you can check out
the previous posts: 

- [1. Intro and Initial Setup]({% post_url 2016-07-22-building-a-node-restful-api-intro-and-setup  %})
- [2. Setting up the Web Server]({% post_url 2016-07-23-building-a-node-restful-api-the-web-server  %})
- [3. The Database: Setup]({% post_url 2016-07-24-building-a-node-restful-api-the-database  %})

## The database connection

<!--from-->
So in our last post we left our project with the mongoose models ready to be used. All we need to do now is connect to 
a real mongo database and start using our models to manipulate the data on it. 
<!--to-->

In order to connect to the database we need to have some specific data from our MongoDB server. And in any real world
case, you would have multiple instances of your MongoDB server or at least 2: one for development and one for production.
So it makes sense to keep all this environment dependent information in a separate place, so that we can just tell our
API to connect to the database and by checking the current node environment (the `NODE_ENV` variable) it will connect
to the correct instance.

What we'll do to solve this issue is create a very simple configuration module. You can also use any of the several 
alternatives that already exist in the npm repository if you prefer.

## Environments Configuration

Let's start by creating a new directory under our existing `config` directory called `env`. This directory will hold
our environment dependent properties.

{% highlight bash %}
cd config
mkdir env
{% endhighlight %}

Now let's create an `index.js` file on this directory and put the following contents on it:

{% highlight javascript %}
const env = process.env.NODE_ENV || 'development';
const config = require(`./${env}`);

export default config;
{% endhighlight %}

This will be the entry point. Every time we import the configuration to access the different properties it has, we'll
do it by requiring (or importing) this file.

What this module does is simple: it checks what is the current `NODE_ENV` value defaulting to `development` in case the
variable is not set. Then it will return as configuration object the module within the same directory with the same name
as the `NODE_ENV` value. So if for example `NODE_ENV=production`, it will return the configuration in `config/env/production.js`.
This means we need to create one configuration file for each of the environments we have. So let's just create the
`development.js` file since we only have a development environment for now.

{% highlight javascript %}
export default {
  env: 'development',
  db: 'mongodb://localhost/node-es6-api-dev',
  port: 3000
};
{% endhighlight %}

So we have only three properties now:

- `env` which is just a shorthand to know in which environment are we currently
- `db` which stores the database connection string
- `port` which stores the port where we want to run our express server

Let's first update our root `index.js` file to make use of the configuration instead of the hardcoded port value:

{% highlight javascript %}
import app from './config/express';
import config from './config/env';

app.listen(config.port, () => {
  console.log(`API Server started and listening on port ${config.port} (${config.env})`);
});

export default app;
{% endhighlight %}

Now whenever we want to change the port where our server listens we can just update the environment configuration file.

### Fixing the export.default issue

As shown before in the `config/env/index.js` file code, we're exporting with `export default config`. If you try to compile
now our Babel code with `gulp babel`, you'll notice in the `dist` directory that the generated javascript code has this:

{% highlight javascript %}
exports.default = config;
{% endhighlight %}

This means that whenever we want to `require` or `import` this module, we need to do it like this:

{% highlight javascript %}
require('./config/env').default;
{% endhighlight %}

This looks ugly, but luckily there's a Babel plugin which helps us to get rid of this kind of transpilation behaviour, 
so let's install it!

{% highlight bash %}
npm install --save-dev babel-plugin-add-module-exports@0.2.1
{% endhighlight %}

And let's add it to our `.babelrc` configuration file:

{% highlight json %}
{
  "presets": [
    "es2015",
    "stage-2"
  ],
  "plugins": [
    "add-module-exports"
  ]
}
{% endhighlight %}

So if you now run the `gulp babel` task, you can check again the transpiled file in the `dist` directory and it should've
changed to have this as last line:

{% highlight javascript %}
module.exports = exports['default'];
{% endhighlight %}

This helps us to get rid of that ugly `require(...).default` :)


### Connecting to the DB

Now let's add a few more lines to our main `index.js` file so that it can connect to our MongoDB:

{% highlight javascript %}
import mongoose from 'mongoose';
import app from './config/express';
import config from './config/env';

mongoose.connect(config.db);
mongoose.connection.on('error', () => {
  throw new Error(`unable to connect to database: ${config.db}`);
});
mongoose.connection.on('connected', () => {
  console.log(`Connected to database: ${config.db}`);
});

if (config.env === 'development') {
  mongoose.set('debug', true);
}

app.listen(config.port, () => {
  console.log(`API Server started and listening on port ${config.port} (${config.env})`);
});

export default app;
{% endhighlight %}

That's it. We're getting the database connection string from the environment configuration. This way you can provide
configurations for development, staging, production or even test environments and the server will connect automatically
depending on the `NODE_ENV` set.
 
We are also setting the mongoose `debug` property to `true` when we're running on development environment. This flag is
useful to understand the queries that are hitting our database.

If everything went well, then you can run the server with `gulp nodemon` and you should see an output similar to this:

{% highlight bash %}
API Server started and listening on port 3000 (development)
Connected to database: mongodb://localhost/node-es6-api-dev
{% endhighlight %}


## Coming up next...

Now our server is running and connected to our MongoDB. The next step will be to add the first endpoints to start
playing with the database and express routes! Look forward to see you on the [next post]({% post_url 2016-07-26-building-a-node-restful-api-the-endpoints  %})



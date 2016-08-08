---
layout: post
title:  "Building a Node.js REST API 2: The Web Server"
date:   2016-07-23 12:56:45
categories: Node.js
banner_image: "/media/express.jpg"
banner_idx: "/media/express_idx.jpg"
featured: true
comments: true
---

## Introduction

Welcome again! If you reached this post from nowhere and don't understand what all this is about, you can check out
the previous post: 

- [1. Intro and Initial Setup]({% post_url 2016-07-22-building-a-node-restful-api-intro-and-setup  %})

You can find the code for this tutorial in [GitHub](https://github.com/mpayetta/node-es6-rest-api).

## The Web Server

<!--from-->
The web server will be the core of our API. It will handle all the incoming HTTP requests, validate the payload data,
do the necessary security checks, fetch the requested data from the database and finally answer with a valid HTTP
response to the requester.
<!--to-->

There are many web server options available in the [NPM registry](https://www.npmjs.com/search?q=web+server). We'll stick
with a good ol' friend of all Node.js Developers called [Express](https://expressjs.com/). Express has proven to be one
of the most easy to use and yet powerful web servers within the Node community. It has a very good documentation and
also a lot of community support as it's one of the most used packages for building web apps. You can read a lot more
about Express on the package website.

So let's stop reading and start doing stuff. Let's install express in our project with the following command:

{% highlight bash %}
npm install --save express@~4.13.1
{% endhighlight %}

Two things to note here:

1. I'm using specific versions with npm install (note the `@~4.13.1`). This is because at the moment of writing
this posts, I have the API up and running with those specific versions. You can go ahead and try newer versions, 
but I don't take responsibility if things don't work well :)
2. I'm using the `--save` flag which will save the dependency in our `package.json` file. After installation you can check
the `package.json` file and it should have a new entry for express.

### Express Initial Configuration

So we have express installed, now it's time to start coding. As I mentioned in the [first of all posts]({% post_url 2016-07-22-building-a-node-restful-api-intro-and-setup  %})
we'll be coding in ES6 being powered by Babel to transpile into ES5. So I will write directly Babel code in the JS files
and we'll see later how can we transpile that with [Gulp](http://gulpjs.com/) into plain ES5 javascript to run it.

Ok, let's create the first directory in our API project root and we'll call it `config`. In this directory we'll include
all kinds of configurations. For now the only configuration that matters to us is the express configuration. So we'll 
create a file called `express.js` and put the following code on it:

{% highlight javascript %}
import express from 'express';

const app = express();

app.get('/', (req, res) => res.send('Hello, this is API and I\'m ok!'));

export default app;
{% endhighlight %}

In this code snippet we're doing the following:

1. Import the express module
2. Creating our express `app` 
3. Adding our first endpoint which returns to the client a friendly message
4. Exporting our app so it can be used as a module

Now let's see how can we use the `app` module we're exporting there to run our API server. In the root of the project
create an `index.js` file and put the following contents on it:

{% highlight javascript %}
import app from './config/express';

app.listen(3000, () => {
  console.log('API Server started and listening on port 3000');
});

export default app;
{% endhighlight %}

Here we are:

1. Importing the app module we created before
2. Starting our express server by calling the `listen` function and passing `3000` as port number. The callback function
will be called when the app is running and print in the console a message to let us know that the server is working.
3. Export again the app module. When we go over unit testing we'll understand why we export the app module here.

Good! We are ready to run our API server! The only problem is that if we simply do `node index.js` in a console, we'll 
get an error like this:

{% highlight bash %}
SyntaxError: Unexpected token import
{% endhighlight %}

This is happening because the latest version of V8 (and thus, Node) does not support the complete set of ES6 features, being
one of this unsupported features the new modules `import`/`export` syntax.

### Babel to the rescue

Here is where we need to make use of what is called a `transpiler` (or `source-to-source compiler`). A transpiler is simply
a program that takes source code in a programming language and produces the equivalent source code in another programming
language. In our case, if we use Babel, we'll be transpiling code written in ES6 into it's equivalent code written in ES5.
Once we do this, we'll be able to run our app with the `node` command (or `nodemon` or `pm2` or whatever process runner 
you feel comfortable with).

There are different ways to transpile code using Babel ([quite a lot](https://babeljs.io/docs/setup/) I would say). What 
we will do is transpile our code by using [Gulp](http://gulpjs.com/) and the Babel plugins for Gulp. The main reason we're
doing this is that we will use Gulp to automate other tasks later (like generating api docs or running unit tests) and it's
always good to keep things consistent.

So, let's move forward and start with our transpilation journey. First of all we'll need to install a few more dependencies:

{% highlight bash %}
npm install --save-dev gulp@^3.9.0 gulp-babel@6.1.2 gulp-load-plugins@^1.2.0 gulp-nodemon@^2.0.6
{% endhighlight %}

Here we're installing (and saving as dev dependencies with the `--save-dev` flag):

1. [`gulp`](https://www.npmjs.com/package/gulp), the task manager that we'll use
2. [`gulp-babel`](https://www.npmjs.com/package/gulp-babel), the gulp plugin to transpile babel code
3. [`gulp-load-plugins`](https://www.npmjs.com/package/gulp-load-plugins), a gulp tool to automatically load all the gulp
plugins we have installed and access them easily.
4. [`gulp-nodemon`](https://www.npmjs.com/package/gulp-nodemon), the greatest node process manager `nodemon` in it's gulp
plugin version.

You'll want to install gulp as a global module as well to make running gulp tasks easier, so I'd recommend you run as well

{% highlight bash %}
npm install -g gulp
{% endhighlight %}

#### Our first Gulp task

Let's create a file in the root of our project called `gulpfile.babel.js`. Note that the suffix `.babel.js` is important
as it will let gulp know that it will need to run with the babel transpiler (our gulpfile will be written in ES6 as all
the js files in our project).
Now put the following in your `gulpfile.babel.js` file (I'll explain below what every line means):

{% highlight javascript %}
import gulp from 'gulp';
import loadPlugins from 'gulp-load-plugins';
import path from 'path';

// Load the gulp plugins into the `plugins` variable
const plugins = loadPlugins();

const paths = {
  js: ['./**/*.js', '!dist/**', '!node_modules/**']
};

// Compile all Babel Javascript into ES5 and put it into the dist dir
gulp.task('babel', () => {
  return gulp.src(paths.js, { base: '.' })
    .pipe(plugins.babel())
    .pipe(gulp.dest('dist'));
});

// Start server with restart on file change events
gulp.task('nodemon', ['babel'], () =>
  plugins.nodemon({
    script: path.join('dist', 'index.js'),
    ext: 'js',
    ignore: ['node_modules/**/*.js', 'dist/**/*.js'],
    tasks: ['babel']
  })
);
{% endhighlight%}

Let's first understand what we're trying to do, and then the whole gulpfile code will make sense:

1. We want to compile our Babel javascript into ES5 javascript
2. We want to put the compiled versions of our code into a directory called `dist`
3. We want to run our API server by calling the compiled `index.js` file

Now let's look at the `gulpfile.babel.js` contents:

1. First we import the needed modules
2. We instantiate the `const plugins` with the result of calling the `loadPlugins` function. This will automatically load
all the gulp plugins we have in the `node_modules` directory and append them to the `plugins` object. So if we have the 
`gulp-babel` plugin installed, we can use it by simply calling `plugins.babel()` (note that is not necessary to add the
`gulp-` prefix).
3. We create a `paths` object with a `js` property. This property lists the globs that we'll use to find all the javascript
files that need to be transpiled (not familiar with globs? more about them [here](https://www.npmjs.com/package/glob))
4. We define our transpiling task where we basically do the following:
    1. Load all the babel javascript files with `gulp.src(...)`
    2. Transpile all the files by calling the gulp-babel plugin (`plugins.babel()`)
    3. Put the result of the transpilation in the `dist` directory
5. We define another gulp task to run our API server with `nodemon`
    1. This task has our previous `babel` task as a dependency (note the `['babel']` array in the task definition)
    2. We run the `gulp-nodemon` plugin and tell it that our entry script is in `dist/index.js`, and that it should watch
for file changes in all javascript files (ignoring `node_modules` and `dist` directories). In case any of the javascript
files changes, it will run the `babel` task again.

### Configuring Babel

We're getting close to it! We now need to configure Babel and we're ready to go. So let's install a few babel dependencies:

{% highlight bash %}
npm install --save-dev babel-preset-es2015@6.5.0 babel-preset-stage-2@6.5.0 babel-core@^6.9.1
{% endhighlight %}

Here we're installing two babel presets. Babel presets are simply collections of plugins, so instead of loading a bunch
of plugins one by one, we can use presets with very common plugin combinations. If you wish to learn more about this 
you can find information on each of the presets we're using npm pages, or in the Babel website.
Finally we install the `babel-core` library which has the babel compiler core code.

Having this we can create now our `.babelrc` configuration file. So let's create this file in the root of our project and
put the following on it:

{% highlight json %}
{
  "presets": [
    "es2015",
    "stage-2"
  ]
}
{% endhighlight %}

This will tell the Babel compiler that we want to use both the es2015 and the stage-2 presets.

### Running the API server

All set! We're ready to run our API server. Go to your console, `cd` to your project directory and run:

{% highlight bash %}
gulp nodemon
{% endhighlight %}

If everything went well, you should see the message `API Server started and listening on port 3000`. In that case you
can try out the API server by going to your favourite browser and going to http://localhost:3000/.
You should see the "Hello, this is API and I'm ok" message returned to your browser.

## Coming up next...

With all this done, now it's time to make our API return dynamic data. For that to happen we'll need to setup a database
to store all our tasks and users, but that will be part of the [next post]({% post_url 2016-07-24-building-a-node-restful-api-the-database  %}).
Thanks for reading!

If you enjoyed reading, retweet or like this tweet! 

<blockquote class="twitter-tweet" data-lang="es"><p lang="en" dir="ltr">Building a Node.js API step 2: setting up the Express server. Available here! <a href="https://t.co/NHi6ZVczmv">https://t.co/NHi6ZVczmv</a></p>&mdash; Mauricio Payetta (@mpayetta) <a href="https://twitter.com/mpayetta/status/759874399524622336">31 de julio de 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

<div class="cc">
    <a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
        <img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc/4.0/88x31.png" />
    </a>
    <br/>
    <span xmlns:dct="http://purl.org/dc/terms/" href="http://purl.org/dc/dcmitype/Text" property="dct:title" rel="dct:type">
        Building a Node.js REST API 2: The Web Server
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
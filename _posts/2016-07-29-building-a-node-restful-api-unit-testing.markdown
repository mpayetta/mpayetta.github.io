---
layout: post
title:  "Building a Node.js REST API 8: Unit Testing"
date:   2016-07-29 12:56:45
categories: Node.js
banner_image: "/media/testing.jpg"
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
- [7. Request Validation]({% post_url 2016-07-28-building-a-node-restful-api-request-validation %})

You can find the code for this tutorial in [GitHub](https://github.com/mpayetta/node-es6-rest-api).

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
        ├── users.js
        └── validation
            └── tasks.js
{% endhighlight %}

If it doesn't, please go back and check if you missed something from the previous posts.

## Unit Testing

<!--from-->
This is the last post of the series, and we'll add some unit tests to the API we've built. It's always important to 
have unit tests, because if in the future we do any kind of changes to the API code, we can quickly check if all the
responses are still as expected or not. We can also check with unit tests what will happen when errors occur and see
if the error responses are correct as well.
<!--to-->

For the unit tests implementation we'll install a widely used test runner for Node.js called [Mocha](https://mochajs.org/)
and some other dependencies we'll install as they are needed during the post. So let's just install `mocha` for now:

{% highlight bash %}
npm install --save-dev mocha@^2.3.4
{% endhighlight %}

### The Testing Database

We're going to need a testing database to be used by our unit tests. Having a testing database is the only way to ensure
that things are being stored correctly. So we're going to create a new configuration file for the testing environment
(this is, when `NODE_ENV = test`). Inside the `config/env` directory create a new `test.js` file with the following 
contents:

{% highlight javascript %}
export default {
  env: 'test',
  db: 'mongodb://localhost/node-es6-api-test',
  port: 3000,
  jwtSecret: 'my-api-secret',
  jwtDuration: '2 hours'
};
{% endhighlight %}

Contents are very similar to the ones in the `development.js` configuration, the main change is in the database name
which is `node-es6-api-test` in this case.

One other thing we'll need for our unit tests is a script that can easily wipe all the data in our testing database.
We are going to run this script before each test to make sure we start with clean collections and the results of the
tests don't interfere with each other. In our db clearing script we'll use a super useful module called `async`, so let's
go ahead and install it:

{% highlight bash %}
npm install --save async
{% endhighlight %}

And now let's create the database clearing script. Create a new directory called `helpers` under the `server` directory
and create a `clearDb.js` file on it. Then put the following contents on the file:

{% highlight javascript linos %}
import mongoose from 'mongoose';
import async from 'async';

export function clearDatabase(callback) {
  if (process.env.NODE_ENV !== 'test') {
    throw new Error('Attempt to clear non testing database!');
  }

  const fns = [];

  function createAsyncFn(index) {
    fns.push((done) => {
      mongoose.connection.collections[index].remove(() => {
        done();
      });
    });
  }

  for (const i in mongoose.connection.collections) {
    if (mongoose.connection.collections.hasOwnProperty(i)) {
      createAsyncFn(i);
    }
  }

  async.parallel(fns, () => callback());
}
{% endhighlight %}

So let's have a look at what we're doing here to clean up the database:

1. First we check if the environment is the `test` environment, otherwise we throw an error since we don't want to
wipe any other data that is not testing data.
2. We iterate over all the mongodb collections in our database and create an asynchronous function that will remove them.
3. We use `async`'s parallel function to run all the asynchronous functions in parallel and once all of them finish we
call the callback function.

### The Test Running Tasks

Let's prepare ourselves to run the awesome unit tests we'll write in a while. We are going to use a few gulp tasks to 
run the tests, so first let's install a few more dev dependencies required for this:

{% highlight bash %}
npm install --save-dev del@^2.2.0 \
 gulp-mocha@^2.2.0 \
 gulp-plumber@^1.0.1 \
 gulp-env@^0.4.0 \
 run-sequence@^1.1.5
{% endhighlight %}

- `del` is just a utility to delete files, we'll use it to clean the `dist` directory
- `gulp-mocha` is just a gulp plugin to run mocha tests
- `gulp-plumber` is a plugin that, as their npm page says, "prevents pipe breaking caused by errors from gulp plugins"
- `gulp-env` is a plugin that allows to set environment variables
- `run-sequence` provides an easy way to run gulp tasks in a specific sequence (this will be included in gulp 4, but it's
not yet released).

Now grab your `gulpfile.babel.js` file and add the following imports:

{% highlight javascript %}
import del from 'del';
import runSequence from 'run-sequence';
import babelCompiler from 'babel-core/register';
{% endhighlight %}

We need to import the babelCompiler to tell mocha that our tests are written in babel ES6. Also let's add the tests files
directory path to our existing paths object (we'll create this path later when writing the unit tests):

{% highlight javascript %}
const paths = {
  js: ['./**/*.js', '!dist/**', '!node_modules/**'],
  tests: './server/test/**/*.test.js'
};
{% endhighlight %}

Let's add first our `clean` and `set-env` tasks which are very simple:

{% highlight javascript %}
// Clean up dist directory
gulp.task('clean', () => {
  return del('dist/**');
});

// Set environment variables
gulp.task('set-env', () => {
  plugins.env({
    vars: {
      NODE_ENV: 'test'
    }
  });
});
{% endhighlight %}

They are quite easy to understand: `gulp clean` will remove the `dist` directory where we keep all our transpiled javascript
code, and `gulp set-env` will set the NODE_ENV var to "test" while the gulp pipeline is being executed.

Now let's do the interesting part, let's create our `gulp test` task:

{% highlight javascript %}
// triggers mocha tests
gulp.task('test', ['set-env'], () => {
  let exitCode = 0;
  
  return gulp.src([paths.tests], { read: false })
    .pipe(plugins.plumber())
    .pipe(plugins.mocha({
      reporter:'spec',
      ui: 'bdd',
      timeout: 2000,
      compilers: {
        js: babelCompiler
      }
    }))
    .once('error', (err) => {
      console.log(err);
      exitCode = 1;
    })
    .once('end', () => {
      process.exit(exitCode);
    });
});
{% endhighlight %}

A bit more complex than the previous ones, let's see what this task does:

1. It runs the `set-env` task which it depends on.
2. It reads all the tests inside our tests path.
3. Pipes to plumber to avoid unwanted behaviour when test errors are thrown.
4. Pipes the test files to the mocha plugin to run all the tests. It also tells mocha that our tests should be compiled
with the `babelCompiler` first.
5. It catches errors and logs them, but doesn't stop execution. We don't want to stop all tests because one of them failed.
6. Exits the process once all tests have been run.

Finally let's add our entry point task for unit tests, we'll call it `mocha`:

{% highlight javascript %}
gulp.task('mocha', ['clean'], () => {
  return runSequence('babel', 'test');
});
{% endhighlight %}

Here we first clean the `dist` directory, and then we run in sequence the `babel` task and the `test` task we defined 
before. That's it, we have our gulp tasks ready to run our unit tests, now let's start writing tests.

### Writing The Unit Tests

Before starting to write our first unit tests we'll install a few more dependencies:

{% highlight bash %}
npm install --save-dev chai@^3.4.1 sinon@^1.17.4 \
sinon-as-promised@^3.0.1 \
sinon-mongoose@^1.2.1 \ 
supertest@^1.1.0 \
supertest-as-promised@^2.0.2 \
http-status
{% endhighlight %}

Let's see what are all these about:

- [`chai`](http://chaijs.com/) is an assertion library which allows you to write things like `expect(foo).to.be.a('string')`
 in your tests.
- [`sinon`](http://sinonjs.org/) is the library we'll use to stub and mock things in our tests.
- [`sinon-as-promised`](https://www.npmjs.com/package/sinon-as-promised) and [`sinon-mongoose`](https://www.npmjs.com/package/sinon-mongoose) 
are extensions to sinon to work with mongoose models and promise style chainable callbacks.
- [`supertest`](https://github.com/visionmedia/supertest) is a module that will facilitate the HTTP requests we'll make
to our API server.
- [`supertest-as-promised`](https://www.npmjs.com/package/supertest-as-promised) allows to use supertest in a promise style way with chainable calls
- [`http-status`](https://www.npmjs.com/package/http-status) is just a collection of HTTP statuses and their readable name
 
All right, time to write our first test! Within the `server` directory create a new `test` directory. This one will
hold all our unit test files. Now create a new directory called `routes` (the path would be `server/test/routes`).
Finally create a `tasks.test.js` file there and we'll start coding our unit test there.
 
The "header" of our test file will have the following imports and require statements:

{% highlight javascript %}
import request from 'supertest-as-promised';
import httpStatus from 'http-status';
import chai, { expect } from 'chai';
import sinon from 'sinon';
import config from '../../../config/env';
import app from '../../../index';
import { clearDatabase } from '../../helpers/ClearDB';
import Task from '../../models/task';
import User from '../../models/user';

require('sinon-mongoose');
require('sinon-as-promised');
{% endhighlight %}

We are importing all the modules we installed before and requiring both sinon add-ons which will extend the imported 
sinon functionality.

All our unit tests will start with a call to mocha's `describe` function, which will be the starting point for the test.
The describe function has two parameters, the first one is a String that should represent what we're testing in this suite.
The second one is a callback function with no parameters. So we can start by describing our test suite as the "Tasks API
Tests":

{% highlight javascript %}
describe('## Tasks API Tests', () => {
  // test suite contents here
});
{% endhighlight %}

Now it's time to define what to do before and after each of the tests is run. Usually there are things that need to be
done for all tests, for example clearing the database in our case. Mocha provides four utility functions to handle this:
`before`, `beforeEach`, `after` and `afterEach`. The `before` and `after` functions will be run once before and after all
the tests in the suite, while their `Each` counterparts will run before and after every single test in the suite runs.

In our case we want to clean up the database before each test runs and we want to create a [sinon sandbox](http://sinonjs.org/docs/#sinon-sandbox) 
where we'll put all our mocks and stubs. And in the "afters" side, we want to restore the sandbox to it's initial state
so that the defined mocks and stubs don't interfere with other tests mocks and stubs. So let's do that:

{% highlight javascript %}
describe('## Tasks API Tests', () => {
  let sandbox;
  
  beforeEach((done) => {
    clearDatabase(() => {
      sandbox = sinon.sandbox.create();
      done();
    });
  });
  
  afterEach((done) => {
    sandbox.restore();
    done();
  });
});
{% endhighlight %}

Since we are testing the tasks endpoints, we'll need to have a created user in our database to whom we can assign tasks.
For this purpose we'll use a `before` function that will create the user for us before all the tests are run:

{% highlight javascript %}
  let sandbox, user;
  
  before((done) => {
    User.create({
      username: 'testuser',
      password: 'testuser'
    }).then((u) => {
      user = u;
      done();
    })
  });

  beforeEach((done) => {
    ...
  });

  afterEach((done) => {
    ...  
  });
{% endhighlight %}

Now we need to decide how to organize our tests suite. In our case, we'll have an extra `describe` level which will group
the tests by endpoint:

{% highlight javascript %}
describe('## Tasks API Tests', () => {
  let sandbox;
  
  beforeEach((done) => { ... });
  
  afterEach((done) => { ... });
  
  describe('### GET /tasks', () => {
      
  });
  
  describe('### GET /tasks/:taskId', () => {
  
  });
  
  describe('### POST /tasks', () => {
  
  });
  
  describe('### PUT /tasks/:taskId', () => {
  
  });
  
  describe('### DELETE /tasks/:taskId', () => {
  
  });
});
{% endhighlight %}

We can now decide how to organize the tests within each describe group. I usually write one test for each possible 
execution branch. I'll write the tests for the task creation endpoint only, but it should be enough to understand how
you can write the rest of the tests.

Each test in mocha must be defined in an `it` function which is similar to `describe` but it accepts a `done` callback
as a parameter of the function. So for example if we want to test the happy path execution flow for the create task endpoint,
the corresponding `it` function would look like:

{% highlight javascript %}
it('should return the created task successfully', (done) => {
    // test code here
    done();
});
{% endhighlight %}

Let's see how can we write the happy path test code:

{% highlight javascript %}
  describe('### POST /tasks', () => {
    it('should return the created task successfully', (done) => {
      request(app)
        .post('/api/tasks')
        .send({
          user: user._id,
          description: 'this is a test task'
        })
        .expect(httpStatus.OK)
        .then(res => {
          expect(res.body.user).to.equal(user._id.toString());
          expect(res.body.description).to.equal('this is a test task');
          expect(res.body._id).to.exist;
          done();
        });
    });
  });
{% endhighlight %}

So we use the `supertest` request function in order to send a request to our `app` instance. We tell the supertest that
we want to send a POST request to the `/api/tasks` endpoint, and we also set the data we want to `send`.
Finally, since this is the happy path, we expect the response status to be the `HTTP 200 Ok` status and we do a few 
assertions on the response contents which are very simple to understand.

Now we can give a try to our tests running task, so grab a console, go to your project directory and run `gulp mocha`. 
If everything went well, you'll see in the end an output like this:

{% highlight bash %}
API Server started and listening on port 3000 (test)
Connected to database: mongodb://localhost/node-es6-api-test
  ## Tasks API Tests
    ### POST /tasks
      ✓ should return the created task successfully (70ms)


  1 passing (235ms)
{% endhighlight %}

Our test is passing as expected, so let's move on and create a few tests to check how our API is handling error situations:

{% highlight javascript %}
  describe('### POST /tasks', () => {
    ...
  });

  it('should return Internal Server Error when mongoose fails to save task', (done) => {
    const createStub = sandbox.stub(Task, 'create');
    createStub.rejects({});
    request(app)
      .post('/api/tasks')
      .send({
        user: user._id,
        description: 'this is a test task'
      })
      .expect(httpStatus.INTERNAL_SERVER_ERROR)
      .then(() => done());
    });
  });
{% endhighlight %}

Here we are creating a stub in the `Task` model to simulate that it will fail to create a new task document in the database.
So we tell the stub to return a rejected promise, which our controller should catch and handle properly sending back
a `500 Internal Server Error` status. If you run again the `gulp mocha` task, the output will look like this:

{% highlight bash %}
API Server started and listening on port 3000 (test)
Connected to database: mongodb://localhost/node-es6-api-test
  ## Tasks API Tests
    ### POST /tasks
      ✓ should return the created task successfully (70ms)
      ✓ should return Internal Server Error when mongoose fails to save task

  1 passing (245ms)
{% endhighlight %}

All good! Our controller is handling internal errors correctly. Finally let's add some test to see if our validation
layer is working properly as well.

{% highlight javascript %}
  describe('### POST /tasks', () => {
    ...
  });

  it('should return Internal Server Error when mongoose fails to save task', (done) => {
    ...
  });
  
  it('should return Bad Request when missing user', (done) => {
    request(app)
      .post('/api/tasks')
      .send({
        description: 'this is a test task'
      })
      .expect(httpStatus.BAD_REQUEST)
      .then(() => done());
  });
{% endhighlight %}

So here we basically try to create a task without sending the user id in the payload data. Our validation layer should
detect this and return back a `400 Bad Request` response. So if your validation layer works fine, after running again
`gulp mocha` you'll get an output like this:

{% highlight bash %}
API Server started and listening on port 3000 (test)
Connected to database: mongodb://localhost/node-es6-api-test
  ## Tasks API Tests
    ### POST /tasks
      ✓ should return the created task successfully (70ms)
      ✓ should return Internal Server Error when mongoose fails to save task
      ✓ should return Bad Request when missing user
      
  1 passing (275ms)
{% endhighlight %}

With this you should have enough tools and example to cover most of the API code. There will be more complex unit tests as 
the API gets more complex as well.

## Conclusion

Over the eight posts in this series we learnt how to build a RESTful API with Node.js from scratch. We understood how
to use different node modules to facilitate essential tasks related to API development such as security, validation and
database access.

Even if this project is not meant to produce a super complex API, I hope it was enough for you to understand the main
concepts and concerns that API development involves, and also helped you learn and meet new technologies and modules
that you can start using for other projects in the future.

That's all, feel free to leave any comments with suggestions or questions below!


<div class="cc">
    <a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
        <img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc/4.0/88x31.png" />
    </a>
    <br/>
    <span xmlns:dct="http://purl.org/dc/terms/" href="http://purl.org/dc/dcmitype/Text" property="dct:title" rel="dct:type">
        Building a Node.js REST API 8: Unit Testing
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




---
title: "Creating a REST API with Express.js (1/X): Express Scaffolding"
description: "Learning to create an express project with support for ES6 modules and a hot reloading dev environment."
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [express, webdev, es6]
image: images/express.png
---

# Overview
In the [project 2 description post](https://www.kyso.dev/webdev/twitter/project%202/2021/03/15/twitter.html) I set a goal to learn Express.js by implementing some of the basic functionality I enjoyed from Adonis.js.

Over this series of posts I'll be learning to implement many of these functionalities by making a basic REST API with Express.js.

## What's covered in this post
This first post is mostly just a summarized version of [this tutorial](https://www.freecodecamp.org/news/how-to-enable-es6-and-beyond-syntax-with-node-and-express-68d3e11fe1ab/) on setting up an express project with ES6 module support and a hot reload dev environment. I highly suggest reading the original article, I'm just going to reiterate the same stuff for self-reference later and get into other stuff in the next post.

 
## Code
All code from this series can be found here

https://github.com/wrathofrathma/rest-express

All code from this post can be found here

https://github.com/wrathofrathma/rest-express/tree/ES6-Express-Generator-Boilerplate


# Intro
## Scaffolding with babel & ES6 module support with express-generator

The goal of this post is to setup a development environment...
- where we contain our coding to a source directory
- with ES6 module support
- that uses babel to transpile the ES6 code in our source directory to ES5 in a dist directory
- where the development runtime reloads everytime we save changes to our code (hot reloading)
- with scripts in our package.json do all of our dirty work.

## Why do we need ES6+ Support?
Honestly, we probably don't entirely but it's really convenient. Enabling ``import from`` syntax for more controlled imports and modules being built into ES6 are reason enough to use it. Also almost every tutorial online seems to be using it, I tried avoiding it for awhile but this makes life a bit simpler.

# Project Setup
## express-generator
First thing we need to is use express-generator to create the starting scaffold of our project. We pass our ``project-name``(mine being rest-express) to the generator as well as ``--no-view`` since we don't need any sort of frontend templating.
```
npx express-generator rest-express --no-view
```

Now cd into the directory and install all of our dependencies

```
cd rest-express
npm install
```

Next, or while they install, we need to restructure our scaffolded project.

- Create a ``server/`` folder
- Put ``bin/``, ``app.js``, and ``routes/`` in the ``server/`` folder
- Rename ``www`` in the ``bin/`` folder to ``www.js``

Lastly, we need to modify our start script to point to the binary in the ``dist/bin/`` folder rather than ``bin/``. Keep in mind we'll be using babel to transpile our ``server/`` directory to our ``dist/`` directory.

```json
// package.json
{
  "name": "your-project-name",
  // ....other details
  "scripts": {
    "server": "node ./dist/bin/www"
  }
}
```

## Converting source to ES6
So now we have to go to each of our source files and convert them to ES6. This is mostly just replacing ``require()`` with ``import from`` syntax and some exports. 

The article was kind enough to give copy & pastable code, or you can go through and do it yourself. Just focus on imports(tops) and exports (bottom).

### Code for ``bin/www.js``
```javascript
// bin/www.js
/**
 * Module dependencies.
 */
import app from '../app';
import debugLib from 'debug';
import http from 'http';
const debug = debugLib('your-project-name:server');
// ..generated code below.
```

### Code for ``routes/index.js`` and ``routes/users.js``
```javascript
// routes/index.js and users.js
import express from 'express';
var router = express.Router();
// ..stuff below
export default router;
```

### Code for ``app.js``
```javascript
// app.js
import express from 'express';
import path from 'path';
import cookieParser from 'cookie-parser';
import logger from 'morgan';

import indexRouter from './routes/index';
import usersRouter from './routes/users';

var app = express();

app.use(logger('dev'));
app.use(express.json());
app.use(express.urlencoded({ extended: false }));
app.use(cookieParser());
app.use(express.static(path.join(__dirname, '../public')));

app.use('/', indexRouter);
app.use('/users', usersRouter);

export default app;
```

Be sure to change the path to ``public/`` which is still at the root directory, to ``../public``. 

After that we're finished with the conversion to ES6 syntax and move onto scripts.

## Script Setup
In the ``package.json`` file at the root of your project you can define scripts. These scripts can be called by ``npm run script-name`` and are used and composed together to perform a multitude of tasks in your project. 

### Install npm-run-all
In order to compose multiple tasks, we need to install the package npm-run-all.
```
npm install npm-run-all
```

### Install babel, nodemon, and rimraf
- Babel is our ES6 -> ES5 javascript transpiler. 
- nodemon watches a directory for changes and performs specified tasks when changes are detected. 
- rimraf is a package that makes deleting files & directories simple
```
npm install @babel/core @babel/cli @babel/preset-env nodemon rimraf
```

### Adding transpile script
Now we can setup our ``package.json`` to transpile our code whenever we run ``npm run transpile`` in our project directory.

First thing we need to do is configure babel to know what syntax we want to transpile to, and thankfully the ``preset-env`` is what we want. So we get away with just adding this babel object to our ``package.json``.

```json
// package.json
{  
  // .. contents above
  "babel": {
    "presets": ["@babel/preset-env"]
  },
}
```

Next we need to add the transpile script to the ``scripts`` section of our ``package.json``.
```json
// package.json
"scripts": {
    "server": "node ./dist/bin/www",
    "transpile": "babel ./server --out-dir dist",
}
```

We can now test this by running this command in your project directory.
```
npm run transpile
```

This should take all of the code in the source directory ``server/`` and transpile it to ES5 code in the ``dist/`` directory. Now would be a good time to try running our server. To give it a run we can run this command.
```
npm run server
```

### Clean Script
Now that we have our transpile script creating a new directory, we need a clean script to delete our ``dist/`` directory so we have a clean directory to work with. This is where rimraf comes in.

Add this to your ``package.json``
```json
"scripts": {
    "server": "node ./dist/bin/www",
    "transpile": "babel ./server --out-dir dist",
    "clean": "rimraf dist"
}
```

This new 'clean' command will remove the folder dist/ from our project root whenever we call ``npm run clean``.

Now we can use ``npm-run-all`` to combine these two scripts, clean and transpile, into a build script that does both for us automatically.

```json
"scripts": {
    "server": "node ./dist/bin/www",
    "transpile": "babel ./server --out-dir dist",
    "clean": "rimraf dist",
    "build": "npm-run-all clean transpile"
}
```

### Develoment enviroment script
For our development environment script, we want to remove our old distribution directory, transpile our code, set the node environment to development and then run the server. 

```json
"scripts": {
    "server": "node ./dist/bin/www",
    "transpile": "babel ./server --out-dir dist",
    "clean": "rimraf dist",
    "build": "npm-run-all clean transpile",
    "dev": "NODE_ENV=development npm-run-all build server",
}    
```

### Production environment script
Similar to the dev script, we also want a production script that peforms the same tasks, but sets the node environment to production. 

Additionally, we want to add a ``start`` script since most deployment platforms like AWS, Firebase, Heroku call the start script to start the server.

```json
"scripts": {
    "server": "node ./dist/bin/www",
    "transpile": "babel ./server --out-dir dist",
    "clean": "rimraf dist",
    "build": "npm-run-all clean transpile",
    "dev": "NODE_ENV=development npm-run-all build server",
    "prod": "NODE_ENV=production npm-run-all build server",
    "start": "npm run prod",
}
```


### Hot reloading
The last thing to setup from this section is hot reloading on the development environment. Remember this uses the ``nodemon`` package, so we need to configure it a bit in our ``package.json``.

```json
// package.json
...
"nodemonConfig": { 
  "exec": "npm run dev",
  "watch": ["server/*", "public/*"],
  "ignore": ["**/__tests__/**", "*.test.js", "*.spec.js"]
},
"scripts": { 
  // ... other scripts
  "watch:dev": "nodemon"
}
```

In the nodemon config, you can see we specify what command to run when a file changes under ``"exec": "npm run dev"``, what directories and files to watch for changes, and what patterns to ignore. 

Now we can start a hot reloading development environment with 
```
npm run watch:dev
```

# Closing
It felt a little weird mostly summarizing for one post, but I wanted to avoid another monolithic post if I could and wanted to document the entire process. 

Anyways, now you know how to setup a fairly decent dev environment! 

Thanks for reading.

# References
- https://expressjs.com/
- https://www.freecodecamp.org/news/how-to-enable-es6-and-beyond-syntax-with-node-and-express-68d3e11fe1ab/
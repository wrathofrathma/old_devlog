---
title: "Hosting an Express.js server on Firebase functions"
description: "Today I learned how to host an express server on firebase!"
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [express, firebase]
image: images/firebase.jpg
---

# Overview
For my next few projects I was planning on learning express.js and today I learned I can actually host an express.js server on firebase. 

In short, since firebase functions can run a node.js server and the request & response objects passed to the firebase functions are basically the same as express, it's incredibly simple to setup.

# Setup
## Create a firebase project
In the [Firebase console](https://console.firebase.google.com) click on "Add Project" and follow the dialogues until the project is created. Or use an existing project.

Then from the project page you'll need to upgrade your project plan from the free "Spark" plan to the pay as you go "Blaze" plan. Most everything can still be free, and just make sure to set the budget to something you're comfortable with(such as $0).

## Install firebase tools
You can either globally install firebase-tools or run them from npx
```
npm install -g firebase-tools
```

## Create a project
I'm not sure about adding firebase to an existing project, but the firebase binary can create a blank project. 

You'll need to login at least once. This will open the browser and prompt you for logging in.
```
firebase login
```

Next you create the project in the directory you want. This command will ask you to select a project from your firebase to link to this folder and then create a skeleton project with firebase features that you select(firestore, functions, etc...).
```
firebase init
```

## Install express.js
Now that we have a blank project in the directory functions/, we cd into it and install our express server.  

```
cd functions
npm install express
```

## Create the express app
Lastly we just edit the index.js

The base skeleton index.js is just
```javascript
const functions = require("firebase-functions");
```

We initialize our express app like normal
```javascript
const functions = require("firebase-functions");
const express = require("express");

const app = express();

app.get('/', (req, res) => {
    res.send("Hello world!")
})
```

Now we use the functions api to tell firebase to send requests to our express app and export it.

```javascript
const functions = require("firebase-functions");
const express = require("express");

const app = express();

app.get('/', (req, res) => {
    res.send('Hello world')
})

const server = functions.https.onRequest(app);

module.exports = {
    server
}
```

## Deploying to firebase
Lastly we need to upload it to our firebase project.

Deploying is just one command
```
firebase deploy --only functions
```

After this, back in your firebase console for your project, under build -> functions on the left, you should be able to see your deployed server with a link that looks like

https://REGION-APP_ID.cloudfunctions.net/server 

# Closing
Well, that's it. The server should be queryable at that address. 

Now I'm going to have to learn how to actually use express lol.


# Resources
From now on I'm going to start adding all the resources I read/use.
- https://codeburst.io/express-js-on-cloud-functions-for-firebase-f76b5506179
- https://firebase.google.com/docs/functions/get-started
- https://expressjs.com/en/starter/hello-world.html
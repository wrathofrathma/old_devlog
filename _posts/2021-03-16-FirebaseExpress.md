---
title: "Hosting an Express.js server on Firebase functions"
description: "Some quick and simple notes on getting an express server running on firebase."
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [express, firebase, twitter, project 2]
image: images/firebase.jpg
---

# Overview
For my next few projects I knew I was planning on learning express.js, but hosting was a concern in my mind. I definitely wanted a place to host my progress for cheap. I was thinking of renting a VPS from ovh, which I might still do, but I found out a few days ago that it's possible to host express.js servers on firebase using their "functions". 

It's also free for the first 2,000,000 function invocations, or in our case the first 2,000,000 API calls. So it's basically perfect for our little hobby projects. We also can set monthly budgets, so a random influx of traffic won't fuck us either. 

## So why is it even compatible in the first place? 
In short, the parameters passed to firebase functions' onRequest() method were designed after Express.js' Request & Response objects.  

One of the firebase engineers explained why on StackOverflow.
> This all works because under the covers, an Express app is actually just a function that takes a Node.js HTTP request and response and acts on them with some automatic sugaring such as routing. So you can pass an Express router or app to a Cloud Function handler without issue, because Express's req and res objects are compatible with the standard Node.js versions. Basically, it's a "double Express" app where one app is calling another.

> As far as function lifecycle and shared state goes: functions are spun up in ephemeral compute instances that may survive to process multiple requests, but may not. You cannot tune or guarantee whether or not a function will be invoked in the same compute instance from one invocation to the next.

>You can create resources (such as an Express app) outside of the function invocation and it will be executed when the compute resources are spun up for that function. This will survive as long as the instance does; however, CPU/network are throttled down to effectively zero between invocations, so you can't do any "work" outside of a function invocation's lifecycle. Once the promise resolves (or you've responded to the HTTP request), your compute resources will be clamped down via throttling and may be terminated at any moment.

So it all works, pog. Let's get started.

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
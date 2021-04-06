---
title: "Creating a REST API with Express.js (5/X): Route controllers, Services, express error handling, basic authentication, and auth middleware"
description: "Route controllers, services, basic authentication, express error handling"
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [express, webdev]
image: images/express.png
---

# Overview
Now that most of the foundation is laid, we can finally start building out endpoint to the outside world. This process will involve building route controllers, a service layer between the models and controllers, more useful errors, setting up basic authentication, and adding auth middleware. So this post will be a bit longer than the others.

[Previous post: Database Abstraction](https://www.kyso.dev/express/webdev/sqlite/2021/03/29/Database-Abstraction.html)

All code from this series can be found here

[https://github.com/wrathofrathma/rest-express](https://github.com/wrathofrathma/rest-express)

All code from this post can be found here

[Git repo at the time of this post](https://github.com/wrathofrathma/rest-express/tree/ae018194343e21cee3358203a993d0a4b944dae0)

# Controllers, Services, Models 
Controllers, services, models, and middleware each serve a unique purpose in the layers of abstraction and flow execution of our backend. I've adopted a 3 layer approach similar to what's described in this [Bulletproof node.js project architecture](https://softwareontheroad.com/ideal-nodejs-project-structure/) article. Adonis.js and other major frameworks have similar concepts, but perhaps with slightly different goals for specific components.

## Controllers
Controllers are meant to handle groups of similar requests that hit the route endpoints of our API. These are the functions that are exposd to the world. 

An example would be if we had the endpoint for user related requests at ``http://localhost:port/api/user``, we might build a ``UserController`` class or object with functions to handle the various user-related tasks. Allowing us to keep our route files clean and our logic in one place. 

The concept of a controller isn't new at all, most major frameworks have their own implementation of a controller to handle route requests. You can put most of your logic here, but I think the logic inside of controllers should be related to offloading the request to the correct service, handling errors, etc. All business logic should be saved for the service layer. 

## Services
Services are the next layer of abstraction, they typically contain all of the business logic and handle the "actions" of the request. The example ``http://localhost:port/api/user`` endpoint I mentioned earlier would likely also have a ``UserService`` class defined somewhere to handle majority of the logic related to user tasks. This is typically the layer we would interact with the database models.

One of the upsides of offloading all of the major task logic to the service layer is that if we throw an error at this level, we can catch it in our controller and propagate it to express.js' error handler. 

## Models
Models are just some form of database abstraction. Using models allows us to define a very pretty interface to interact with our database and allows us to swap database backends without breaking outside functionality if we keep the interface the same. Models typically contain all database-related logic, sanitization, error handling, etc. 

## Middleware
When our API receives a request, typically it'll go to the defined route controller method, but there are times when we want to perform some action on the request before the controller touches it. Sometimes we want to log the request, maybe process the body of the request, check whether the request has valid authentication, etc. We can do all of this by defining middleware functions that run before our controller methods. 

This is another one of those huge concepts in web dev that you'll see on every major framework.

# More useful errors
Now that we're dealing with express.js endpoints, now is a good time to talk about how we can use express.js to send useful errors to our users.

Uncaught errors will crash our node.js project, however if we catch them at the controller level, we can propagate them to express.js' error handler by using the ``next()`` method similar to how we would call the next middleware method. The difference is that we are passing the error object ``next(err)`` which won't match the function identities of any route endpoint methods or middleware which all take 3 arguments ``(req, res, next)``.

You can build your own error handler, but express' default error handler is good enough for the purposes of this exercise as long as we tailor our error messages to their handler. If you're intersted in using your own error handler, refer to [express' error handling page](https://expressjs.com/en/guide/error-handling.html).

By default, express' error handler checks for an ``err.status`` or ``err.statusCode`` property that's used to generate the HTML of the error and set the status of the request. I also notice the ``err.message`` that's set when you ``throw new Error("some message");`` also appears, but I'm not sure if that's part of the stack trace that's left out in production environments.

For now we can get away with a basic exception class.

```javascript
class Exception extends Error {
    constructor({message, status}) {
        super(message);
        this.message = message;
        this.status = status;
    }
}

module.exports = Exception;
```

I put my exceptions in their own ``server/exceptions/`` directory with the expectation that someday I might built more tailored ones. This code specifically lives in ``server/exceptions/GenericException.js``.

# Building our auth endpoint
The first endpoint we need to build is one for users to register and login through. That's going to be our auth endpoint. This will involve us creating a ``router`` to catch the requests, an ``AuthController`` to contain the methods that will handle the route requests and an ``AuthService`` to handle registration and login logic.

## Setup
### Controller
To get started, we want to create a ``server/controllers/`` folder. Inside of it we're going to create our ``AuthController.js``. 

Starting out, we don't need anything fancy, just something to test with

```javascript
const AuthController = {
    login(req, res, next) {
        res.send({message: "Hello world"})
    },

    register(req, res, next) {
        res.send({message: "Hello world"})
    }
}

export default AuthController;
```

Now we want to connect this controller to our router. 

### Router

When we generated our project using the ``express-generator``, it created a ``routes/`` folder that we put into ``server/routes``.

In that folder we want to create a file ``auth.js`` that contains the following.

```javascript
import express from 'express';
import AuthController from '../controllers/AuthController';
var router = express.Router();

router.post('/login', AuthController.login);
router.post('/register', AuthController.register);

export default router;
```

What's happening is we're using the ``express.Router`` class to setup new routes and linking them to our ``AuthController``.

### Connecting the router
Lastly, we need to tell our app to use the new router.

In ``server/app.js``, we want to add the lines to connect our main express instance to our router on the ``/auth`` endpoint.

```javascript
import indexRouter from './routes/index';
import authRouter from './routes/auth';

... 

app.use('/', indexRouter);
app.use('/auth', authRouter);

...
```

Now if we use some RESTClient and hit our ``localhost:3000/auth/login``, and ``localhost:3000/auth/register`` endpoints with POST requests, we will receive the response

```json
{
    "message": "Hello world"
}
```

## Auth service

This is the layer that will handle all of the business logic. It doesn't care about the ``req, res, next`` objects in the express routing methods, just what we use. 

I modeled this service off of [You don't need passport.js - Guide to node.js authentication](https://softwareontheroad.com/nodejs-jwt-authentication-oauth/). There are a few differences, but the original article is worth the read. 

### Setup
We're going to be using a few different libraries to assist us with our authentication. 

#### argon2
The argon2 library is a javascript cryptographic library. We're going to be using it to hash our passwords before saving them to the database and verifying user-passwords. 

Coincidentally it also protects against timing-based attacks. So that's neat too.

#### jsonwebtoken
jsonwebtoken or JWT at a basic level is just a sign/encrypted json object. They're commonly used for carrying authentication and user data, and in this project we'll be using them for authentication.

To install these libraries

```
npm install argon2 jsonwebtoken
```

#### services folder
Lastly, much like our models, controllers, exceptions, etc..., we are going to create a dedicated services folder at ``server/services/``. 

### AuthService.js

#### Login
When our ``AuthService`` receives a login request from our ``AuthController``, this is what will happen

- Client sends public identification & private key in the form of username/password.
- Server searches the database for a matching user
- If the user exists, server hashes the password and compares it to the hashed password stored in the database
- If the password matches, generate a JWT to send back to the user.

If either the user isn't found or we receive invalid login details, we should expect to throw an error.

```javascript
import User from '../models/UserModel';
import Exception from '../exceptions/GenericException';
import * as argon2 from 'argon2';
import * as jwt from 'jsonwebtoken';

const AuthService = {
    async login({username, password}) {
        const user = await User.get({username});

        const correct_pwd = await argon2.verify(user.password, password);
        if(!correct_pwd){
            throw new Exception({
                message: "Invalid login details",
                status: 401,
                code: "E_INVALID_LOGIN"
            })
        }
        
        return {
            user: {
                id: user.id,
                username: user.username
            },
            token: AuthService.generateJWT(user)
        }
    }
}
```

In the above code, you can see that we're using the ``argon2.verify(hashedPassword, plainPassword)`` method to validate the user password. 

I also like my services to return json to be sent back to the user by the controller layer.

#### Registration
Registration is straightforward. If we receive a registration request we want to

- Check for duplicate usernames (handled by our UserModel)
- Hash the password (argon2)
- Create a new user (handled by our UserModel)
- Return a login token & user info (we can chain our registration into a login call to handle this)

```javascript
const AuthService {
    ...
    async register({username, password}) {
        await User.create({
            username, 
            password: await argon2.hash(password)
        });
        return await AuthService.login(...arguments);
    },
    ...
}
```

#### Token generation
In the ``AuthService.login()`` method we called an ``AuthService.generateJWT()`` method, and we're going to define it here.

Earlier we talked about how jsonwebtokens are mostly just encrypted/signed json objects. So we need to decide 

- What data we want to store in our json?
- What our signature will be?
- Do we want an expiration?

Our data is just going to be the username & userid of the user we're authenticating, that way we can decrypt it later and find out what user is making requests with this token. 

Our signature can be anything, but you should create a strong secret and keep it out of your code. An environmental variable of some sort would be ideal I believe. We're not doing that for our small example, but it's just a thought for open source projects.

Expiration doesn't matter, but we're setting it to 6 hours here.

```javascript
const AuthService = {
    ...
    generateJWT(user) {
        const data = {
            id: user.id,
            username: user.username
        };

        const signature = "SomeRandomSekrit";
        const expiration = "6h";
        
        return jwt.sign({data, }, signature, { expiresIn: expiration});
    }
}
```

### AuthController.js
Now it's time to fill our our AuthController.js we created earlier. Right now if we hit either of our endpoints, we'll just receive the response

```json
{
    "message": "Hello world"
}
```

We want to link our requests up to our AuthService and send the data returned from our service back to the user. 

```javascript
import AuthService from '../services/AuthService';

const AuthController = {
    async login(req, res, next) {
        await AuthService.login({
            username: req.body.username, 
            password: req.body.password
        })
        .then((login_data) => {
            res.send(login_data);
        })
    },

    async register(req, res, next) {
        await AuthService.register({
            username: req.body.username, 
            password: req.body.password
        })
        .then((login_data) => {
            res.send(login_data);
        })
    }
}
```

We also want to handle any errors that might be thrown and pass them to the express error handler. We can do this by adding a ``.catch()`` callback at the end of our calls.

```javascript
const AuthController = {
    async login(req, res, next) {
        await AuthService.login({
            username: req.body.username, 
            password: req.body.password
        })
        .then((login_data) => {
            res.send(login_data);
        })
        .catch((err) => {
            next(err);
        })
    },

    async register(req, res, next) {
        await AuthService.register({
            username: req.body.username, 
            password: req.body.password
        })
        .then((login_data) => {
            res.send(login_data);
        })
        .catch((err) => {
            next(err);
        })
    }
}
```

Now we should be able to register & login our users and receive our token on the RESTClient side. Below is the response I received from a registration request, which shows that everything is working as intended.

![Rest client registration response]({{site.baseurl}}/images/rest-express/rest-express-postman-login.png)

# Auth middleware
Now that we've built our auth endpoint and users can register and login, we want to start building requests that require authentication. Before we can get to all of that, we need a convenient way to get authenticated. This is where middleware comes in. 

We know middleware is executed before the request reaches our route controllers, so if we want to authenticate a user before they hit the controller, we want to write middleware that authenticates the user. The process of authenticating a user will involve

- Parsing the header for an ``authorization`` field that contains a ``Bearer SOMEJWT``.
- Splitting the JWT from the authorization string.
- Validating the JWT(checking it's not malformed or expired)
- Extracting the decoded user data
- Checking our database for that user
- Attaching the user to the ``req`` object for controllers to know which user is making the calls.
- Call ``next()`` to pass control to the controllers

So let's create a ``server/middleware/`` directory with an ``AuthMiddleware.js`` inside of it to handle all of the above requirements.

```javascript
import * as jwt from 'jsonwebtoken';
import User from '../models/UserModel';

function parseAuthToken(req) {
    if(req.headers.authorization  && req.headers.authorization.split(' ')[0] === "Bearer")
        return req.headers.authorization.split(' ')[1];
}

export default async function (req, res, next) {
    const token = parseAuthToken(req);
    return await jwt.verify(token, "SomeRandomSekrit", async (err, decoded) => {
        if(err) 
            next(new Error("Invalid token"));
        req.user = await User.get({id: decoded.data.id});
        return next();
    })
    .catch((err) => {
        next(err);
    })
}
```

# Adding authenticated user endpoint
Now to create an endpoint that requires authentication, we need to tell our express router to use our middleware. So let's add that to our userRouter by modifying ``server/routes/users.js``.

```javascript
import AuthMiddleware from '../middleware/AuthMiddleware';
import express from 'express';
var router = express.Router();

router.use(AuthMiddleware);

/* GET users listing. */
router.get('/', function(req, res, next) {
  res.send('respond with a resource');
});

export default router;
```

Now the default endpoint ``http://localhost:3000/users`` will throw an error if the client doesn't include an ``authorization`` header or the JWT in the ``Bearer SOMEJWT`` string isn't valid.

We could also use the ``req.user`` object that was attached by the auth middleware if we wanted to.

```javascript
import AuthMiddleware from '../middleware/AuthMiddleware';
import express from 'express';
var router = express.Router();

router.use(AuthMiddleware);

/* GET users listing. */
router.get('/', function(req, res, next) {
  res.send({
      id: req.user.id,
      username: req.user.username
  });
});

export default router;
```

This method isn't really useful, but shows how we might access the user object in the req object.

# Validating user access to a resource
The last key we need to have all the tools to build the rest of the API is some way to verify whether a user has access to a given resource. Right now our only real resource are tweets, but most of our resources will have a ``user_id`` field for the owner of the resource. 

In our ``AuthService.js`` we're going to add one more method that will take in a user, and a resource(tweet), validate that it exists, and that the user id matches the owner's id.

```javascript
const AuthService = {
    ...

    async verifyPermission(user, resource) {
        if(!resource){
            throw new Exception({
                message: "Resource not exist",
                status: 404
            });
        }
        if(user.id !== resource.user_id) {
            throw new Exception({
                message: "Invalid access",
                status: 401
            });
        }
    }
}
    ...
```

# Closing
I think I'm learning that in the future, I might finish a project before writing about it. So much revision happens during the process. 

Anyways, now that we have all of the tools we need to build the API, the rest is just busywork. Nothing of note to write about most likely. So the next post will probably be about either unit testing the express routes, or converting to the ORM library sequelize.

[Previous post: Database Abstraction](https://www.kyso.dev/express/webdev/sqlite/2021/03/29/Database-Abstraction.html)

# References
- [https://softwareontheroad.com/nodejs-jwt-authentication-oauth/](https://softwareontheroad.com/nodejs-jwt-authentication-oauth/)
- [Mozilla HTTP Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [https://expressjs.com/en/guide/error-handling.html](https://expressjs.com/en/guide/error-handling.html)
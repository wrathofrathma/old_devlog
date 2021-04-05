---
title: "Creating a REST API with Express.js (4/X): Database abstraction"
description: "Building a class to abstract database interactions and error handling."
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [express, webdev, sqlite]
image: images/express.png
---

# Overview
So I was considering what I wanted to do next and I think the short term roadmap for a functional REST API is realistically to handle routing, functions to handle the requests, authentication, and error handling. However, before we do any of that, it'd be nice to abstract our database interactions. This would allow us to perform unit tests on specific actions, as well as prevent us from rewriting the same code over and over. 

So this post will cover building these abstractions, sanitizing input, and handling database errors.

[Previous Post: Unit Testing](https://www.kyso.dev/express/webdev/jest/unit%20test/sqlite/2021/03/25/Unit-Testing.html)

All code from this project can be found here

[https://github.com/wrathofrathma/rest-express](https://github.com/wrathofrathma/rest-express)

Code from this post can be found here

[https://github.com/wrathofrathma/rest-express/tree/b5bd089a2520d356dc4523dc8af95b7ad4f0fe0e](https://github.com/wrathofrathma/rest-express/tree/b5bd089a2520d356dc4523dc8af95b7ad4f0fe0e)

# Sqlite error handling
To be honest, I think the error handling is a complete shit show. Maybe it's my lack of experience with javascript, but neither the [sqlite documentation]() nor the [sqlite3 documentation](https://github.com/mapbox/node-sqlite3/wiki/API#databaserunsql-param--callback) cover how to do error handling properly with this setup. Since this is more about the journey, I'll go over my problem solving process and the correct solution will be at the end.

## Callbacks
The first way I tried was what the [sqlite3 documentation](https://github.com/mapbox/node-sqlite3/wiki/API#databaserunsql-param--callback) suggests. Passing a callback to ``db.run(sql, params, cb)``.

I tried creating a very basic print callback and not once would it be executed during my unit testing.
```javascript
await db.run("INSERT INTO users (username, password) VALUES (?, ?);", [username, password], function (err) {
    if(err) {
        console.error(err);
    }
    console.log("Normal");
})
```

I tried a handful of variations with arrow functions, ``db.exec()`` instead, dropping the parameters, etc. I get the feeling that this wasn't ported to the ``sqlite`` package.

## db.on('event', callback)
After failing with that, I figured I'd try tracing sqlite errors the way [both documentations](https://www.npmjs.com/package/sqlite#tracing-sql-errors) suggest.

When I create the database object, use ``db.on('some_event', [callbacks])`` to create callbacks when specific events fire on the database object. The [sqlite3 API](https://github.com/mapbox/node-sqlite3/wiki/API) states

### db.run(sql, params, callback)
> sql: The SQL query to run. If the SQL query is invalid and a callback was passed to the function, it is called with an error object containing the error message from SQLite. If no callback was passed and preparing fails, an error event will be emitted on the underlying Statement object.

### db.exec(sql, callback)
> Runs all SQL queries in the supplied string. No result rows are retrieved. The function returns the Database object to allow for function chaining. If a query fails, no subsequent statements will be executed (wrap it in a transaction if you want all or none to be executed). When all statements have been executed successfully, or when an error occurs, the callback function is called, with the first parameter being either null or an error object. When no callback is provided and an error occurs, an error event will be emitted on the database object.

If we don't specify an error callback, which ours isn't even working, then the errors are emitted on the database object. So I attempted to listen for the ``error`` event by adding a callback when I create the database.

```javascript
import sqlite3 from 'sqlite3';
import { open } from 'sqlite';

sqlite3.verbose();
export async function openDb() {
    const db = await open({
        filename: "./sqlite.db",
        driver: sqlite3.Database
    });
    await db.exec("PRAGMA foreign_keys = ON;");

    db.on('error', (sql) => {
        console.log(sql);
    })

    return db;
}
```

Alas, this didn't work either. I didn't really get anything out of the error event. Yet another thing that didn't transfer from ``sqlite3`` to the  ``sqlite`` package. I also tried hooking into the inner ``sqlite3.Database`` property of the ``db`` object, but that too didn't work.

### Exception catching
So, I didn't think to try this method immediately since the documentation didn't mention it, and I had issues error handling this way with passing errors to express' route error handles. However, catching the errors using promise catches or vanilla try/catch statements worked fine for not crashing the entire program.

```javascript
await db.run("INSERT INTO users (username, password) VALUES (?, ?);", [username, password])
.catch((err) => {
    console.log(err);
});
```

This also worked just as well

```javascript
try {
    await db.run("INSERT INTO users (username, password) VALUES (?, ?);", [username, password]);
    
}
catch (err) {
    console.log(err);
}
```

We do need to still handle the error, but at least we have a starting place.

# Database abstraction 

Database abstraction is a good idea because it allows us to narrow the surface area of our testing for database interactions, maximize code reuse, and is likely going to be significantly more readable. 
## Input sanitization
A big issue in the database world is SQL injections, which typically occur when you don't sanitize your inputs or process them properly. 

Curious about this, I looked around at how to best structure my inputs and found this [git issue: Defending against SQL injections](https://github.com/mapbox/node-sqlite3/issues/57).

> SQLite protects you against SQL injections if you specify user-supplied data as part of the params rather than stringing together an SQL query:
>
>BAD: db.prepare("INSERT INTO foo VALUES(" + variable + ")");
>
>GOOD: db.prepare("INSERT INTO foo VALUES (?)", variable);
>
>By using the placeholder ?, SQLite automatically treats the data as input data and it does not interfere with parsing the actual SQL statement.

## More useful exceptions

The errors generated from our database are good, but they're a bit low level for how we will want express.js to represent these errors later. 

So let's create a generic exception class to just contain a bit of extra data. I created a ``server/exceptions/`` folder to house possible future exceptions, and created a ``GenericException.js`` in that folder to start with.
```javascript
// We're going to have our own, slightly more descriptive exception class.
/**
 * Slightly more descriptive error handling.
 * - message: Descriptive error message 
 * - status: HTTP status code
 * - code: Error code
 * 
 */
class Exception extends Error {
    constructor({message, status, code}) {
        super(message);
        this.message = message;
        this.status = status;
        this.code = code;
    }
}

module.exports = Exception;
```

Now when we encounter an error, we can propagate useful information with it.

To use our new exception class, all we need to do is the following

```javascript
import Exception from 'some_path/exceptions/GenericException';

// do stuff
...
throw new Exception({
    message: "Some useful error message",
    status: 404, // any http code
    code: "E_SOME_USEFUL_ERROR"
})
```

## Models
Models are classes that will encapsulate most, or all, of our database logic. There are many different ways to design these, but as long as we stay true to our goal of abstracting the actions we'll perform on the database, we'll be fine. Functionally most of the SQL and code will be similar to when we were unit testing our database, we're just making it easier to use and handling exceptions.

We first should create a directory for our models, ``server/models/``.

### User Model
Inside the models directory, we'll create a ``UserModel.js`` file.

When I scaffold out my models, and most of my classes, I ask myself what kind of actions I want to perform and what data I want my class to store/track.

```javascript
class UserModel {
    constructor({id, username password}) {
        this.id = id;
        this.username = username;
        this.password = password;
    }
    static async create({username, password}){
        // Create new user
    }
    static async get({id, username}) {
        // Fetch user based off of id or username
    }
    async update({username, password}) {
        // Update user data
    }
    async delete() {
        // Delete user
    }
    async tweets() {
        // Fetches all of the user's tweets
    }
}

module.exports = UserModel;
```

Then I start to fill out the functionality of each one.

#### Creating a user
I would start by checking for whether the variables we require were passed successfully. It'd be even better if we validated the types too.

If the username and password aren't passed, we're gonna want to throw an exception for express to handle later. We're using the HTTP status code 422, for "Unprocessable Entity" and defining an internal error code to catch later. You can find a list of HTTP response codes [on mozilla's HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status) page.

```javascript
class UserModel {
    ...

    static async create({username, password}) {
        // Check for username and password
        if(!username || !password) {
            throw new Exception({
                message: "Missing username or password",
                code: "E_NOT_PROCESSABLE",
                status: 422
            })
        }
    ...
    }
}
```


After that we need to insert the user into the database, check for duplicate user error, and I also want this method to return a new UserModel object so that we can begin work with the new user immediately.

```javascript
class UserModel {
    ...

    static async create({username, password}) {
        // Check for username and password
        if(!username || !password) {
            throw new Exception({
                message: "Missing username or password",
                code: "E_NOT_PROCESSABLE",
                status: 422
            })
        }
        const db = await openDb();

        return await db.run("INSERT INTO users (username, password) VALUES (?, ?);", [username, password])
        .catch((err) => {
            if(err.message==='SQLITE_CONSTRAINT: UNIQUE constraint failed: users.username'){
                throw new Exception({
                    message: 'User already exists',
                    code: 'E_DUPLICATE_RESOURCE',
                    status: 409
                });
            }
        })
        .then((res) => {
            //If we make it here, then we created a new user successfully.
            return new UserModel({
                username,
                password,
                id: res.lastID
            });
        })
    }
    ...
}
```

#### Fetching a user
Fetching a user is a very similar process
- Checking for user id or username
- Querying the database
- Checking for errors if the user doesn't exist
- Returning a UserModel on success

```javascript
class UserModel {
    ...
    static async get({id, username}) {
        const db = await openDb();

        var res;
        if(id) { 
            res = await db.get("SELECT * FROM users where id = ?;", id);
        }
        else if(username) {
            res = await db.get("SELECT * FROM users where username = ?;", username);
        }
        else {
            throw new Exception({
                message: "Missing user_id or username",
                code: "E_UNPROCESSABLE_PARAMS",
                status: 422
            })
        }

        if(!res) {
            throw new Exception({
                message: "User doesn't exist",
                code: "E_RESOURCE_DOES_NOT_EXIST",
                status: 404
            })
        }
        return new UserModel({
            username: res.username,
            password: res.password,
            id: res.id
         })
    }
    ...
}
```

#### Updating user data
Updating a user is no different, we just need to build our sql statement based on whether we're updating the username, password, or both. We need to throw an exception if we didn't pass a property to update. Lastly, I want to update the current class object with the new data so the user can access it immediately without re-querying.
```javascript
...
class UserModel {
    ...
    async update({username, password}){
        var self = this;
        const db = await openDb();
        var stmt;
        var params;
        if(username && password) {
            stmt = "UPDATE users SET username = ?, password = ? WHERE id = ?;";
            params = [username, password, self.id];
        } 
        else if(username) {
            stmt = "UPDATE users SET username = ? WHERE id = ?;";
            params = [username, self.id];

        }
        else if (password) {
            stmt = "UPDATE users SET password = ? WHERE id = ?;";
            params = [password, self.id];
        }
        else{
            throw new Exception({
                message: "Missing updated property.",
                code: "E_NOT_PROCESSABLE",
                status: 422
            });
        }
        await db.run(stmt, params)
        .then(function () {
            // After it runs, we should update our local data. We can assume it updated successfully.
            if(username){
                self.username = username;
            }
            if(password) {
                self.password = password;
            }
        })
    }
    ...
}
```
#### Deleting a user
Deleting a user is probably the easiest out of all of these. Literally the same as our unit testing lol.

```javascript
class UserModel {
    ...
    async delete() {
        const db = await openDb();
        await db.run("DELETE FROM users WHERE id = ?;", this.id);
    }
    ...
}
```
#### Fetching user tweets
When fetching user tweets, I want it to return a list of Tweet objects created from our TweetModel, or an empty list if no tweets are found. There shouldn't be any specific errors we need to watch for here. 

We're just querying the database with ``db.all(sql)``, which fetches all rows which match the query, rather than ``db.get(sql)`` which only fetches the first match.

Then using array mapping to transform the data the query returned, to a list of TweetModel objects(imported as Tweet). 

```javascript
class UserModel {
    async tweets() {
        const db = await openDb();
        const res = await db.all("SELECT * FROM tweets WHERE user_id = ?", this.id);
        
        if(!res)
            return [];
        
        const user_tweets = res.map(({t}) => new Tweet({
            id: t.id,
            user_id: t.user_id,
            content: t.content
        }));

        return user_tweets;
   }
}
```
### TweetModel
It's basically the same as the UserModel and I already hate how repetitive a single one gets. You can figure out this one or check the source code.

# Unit Testing
Now that we're familiar with testing, I want to get into the habit of unit testing everything as I build it.

Most of this is similar to when we CRUD tested the database, only this is abstracted.

## Unit testing UserModel
After creating the ``tests/models.test.js`` file, the first thing I want to do is create a group for unit testing the user model, and variables related to multiple tests(we'll need them not to fall out of scope). 

```javascript
import User from '../dist/models/UserModel';
import Tweet from '../dist/models/TweetModel';

describe('Models:User', () => {
    const username = 'models_u_rathma';
    const password = 'somepass';
    const newusername = 'models_u2_rathma';
    const newpassword = 'somepass2';

    var user;
}
```

### Creating a user
Since we're working with a fresh database every time, we should be able to create a new user without checking for duplicate errors. We're just going to use our ``UserModel`` class to create a new user and check that it returned a UserModel object. 

```javascript
describe('Models:User', () => {
    ...
    test('Models:User:Create', async () => {
        user = await User.create({
            username,
            password
        });
        // Check that it returned a user correctly.
        expect(user).toEqual(expect.any(User));
    })
    ...
}
```

### Fetching a user
I don't think I need to comment on this one. We're just checking that it returns a valid user object.

```javascript
describe('Models:User', () => {
    ...
    test('Models:User:Read', async () => {
        user = await User.get({username});
        expect(user).toEqual(expect.any(User));
    })
    ...
})
```

### Updating a user
All we want to test here is that 
1. When we update, the local object is updated
2. The database object is also updated
```javascript
describe('Models:User', () => {
    ...
    test('Models:User:Update', async () => {
        // Separate updates first
        await user.update({username: newusername});
        await user.update({password: newpassword});
        //Check if they're changed in the local object.
        expect(user.username).toEqual(expect.stringMatching(newusername));
        expect(user.password).toEqual(expect.stringMatching(newpassword));
        // Now let's fetch the user again and check if it exists
        user = await User.get({username: newusername});
        expect(user).toEqual(expect.any(User));
        //Check variables set in properly.
        expect(user.username).toEqual(expect.stringMatching(newusername));
        expect(user.password).toEqual(expect.stringMatching(newpassword));
    })
    ...
})
```

### Deleting a user
Lastly, we want to check if our users are properly being removed from the database when we delete using the user object. Even if we attempt to delete a row that doesn't exist in the database, it won't throw an error, so no worries about error checking. 

Since UserModel throws a 404, "User doesn't exist" exception when we try to fetch a user that doesn't exist, we can just test for that.

```javascript
describe('Models:User', () => {
    ...
    test('Models:User:Delete', async () => {
        // Delete our user
        await user.delete();

        // Try to fetch deleted user, should throw a 404 exception
        const get_user = async () => {
            await User.get({username});
        }
        await expect(get_user()).rejects.toThrow("User doesn't exist");
    })
})
```

# Closing
Even though we could have just used some sort of pre-baked ORM like sequelize, I think there's a lot of value in designing stuff by hand occasionally. When I finish with the REST functionality, I'll try out sequelize and firestore as alternative storage solutions. 

[Previous Post: Unit Testing](https://www.kyso.dev/express/webdev/jest/unit%20test/sqlite/2021/03/25/Unit-Testing.html)

# References
- [Sqlite3 API](https://github.com/mapbox/node-sqlite3/wiki/API)
- [Sqlite reference](https://www.npmjs.com/package/sqlite)
- [Git issue: Defending against SQL injections](https://github.com/mapbox/node-sqlite3/issues/57)
- [Git issue: Unhandled null parameters](https://github.com/mapbox/node-sqlite3/issues/116)
- [Javascript class reference](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes)
- [Mozilla HTTP Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
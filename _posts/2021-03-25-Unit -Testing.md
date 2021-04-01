---
title: "Creating a REST API with Express.js (3/X): Unit Testing"
description: "Unit testing our database with Jest!"
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [express, webdev, jest, unit test, sqlite]
image: images/express.png
---

# Overview
In this post I'll be learning unit testing with the Jest framework by testing the database we built in the previous post.

## What is Unit Testing
Unit testing is where you test individual components of software. By testing and ensuring certain expectations from eatch unit/component, it's very easy to track down errors down the line and even automate testing large programs. 

Unit testing is used by most professionals in software development, so it can't hurt to get a hang of it early.

## Jest
[Jest](https://jestjs.io/) is a popular javascript web testing framework. It has support for just about every major frontend framework, typescript, nodejs, and babel. 

## Code
All code from this series can be found here

https://github.com/wrathofrathma/rest-express

All code from this post can be found here

https://github.com/wrathofrathma/rest-express/tree/b86cad2b5feafbd807f6d60f789f471dff448529


# Setting up Jest
The first thing we need to do is install jest to our devDependencies.

```
npm install -D jest
```

Once that's installed we'll edit our ``package.json`` to tell jest what our test environment is(node), and create a script to run our tests.

```json
// package.json
...
"jest": { 
  "testEnvironment": "node"
},
"scripts": {
  // ..other scripts 
  "test": "jest"
}
```

# Writing a basic test
By default Jest runs any tests with the naming scheme ``{test-name}.test.js`` found anywhere in the project directory. I'm not sure if it has a depth limit as I've only used tests 1 folder deep. 


A basic test in jest is composed of three things, 
- The ``test()`` method that starts the test
- A driver function to run our test
- Setting expectations within the driver function using the ``expect()`` method.


The ``Test()`` method takes in a few arguments
- A name for the test(best to be descriptive)
- A driver function to run the test
- And an optional timeout for when to give up

Below is an example test that shows us creating a test some ``some_fn()``. Our driver method is an arrow function that sets the expectation for the return value of ``some_fn()`` to be 0.


```javascript
function some_fn() {
    return 0;
}

test('Some_fn:ReturnValue', () => {
    expect(some_fn()).toEqual(0);
});
```

You can throw the above code into a ``sample.test.js`` file and then run the following command to run the test.
```
npm run test
```
I highly suggest using the [expect reference](https://jestjs.io/docs/expect), or the [API itself](https://jestjs.io/docs/api) to guide you in creating your tests.

# Unit testing our database
## Setup
Now that we understand the basic structure of a test, we'll be writing tests for our database to ensure the following
- Our export method is properly working and opening the database
- We can perform CRUD operations on the User table
- We can perform CRUD operations on the Tweets table
- Tweets are deleted on user deletion

For now I've taken to creating a directory to house my unit tests under the project root ``project-root/tests/``.

Create a file in the ``tests/`` directory called ``database.test.js``, the rest of this section will be working in that file.

## Testing whether our database opens correctly.
Our first objective is to test whether our database is correctly being opened and exported by our database module.

To do this we import our database module just as we would in our server code, and then write a method for our test. Since we're using async/await to access our database, we need to make our test function also async. 

After we open our database, we test whether the object returned from the ``openDb()`` method actually returned a valid sqlite3 database by comparing the returned object with the structure a valid sqlite3 database would have.
```json
{
    "config": { 
        "filename": "somefile", 
        "driver": sqlite3.Database driver
        },
    "db": Database Instance
}
```
To test for this structure, we use the following expectation functions.
- ``expect(value).toEqual(value)``: Recursively compares all properties for equality.
- ``expect.objectContaining(object)``: Checks if the object contains properties with expectations you set.
- ``expect.anything()``: matches anything that isn't ``undefined`` or ``null``.
- ``expect.any(Constructor)``: Matches anything created by the given constructor.

```javascript
import { openDb } from '../server/database';
import sqlite3 from 'sqlite3';

async function test_open() {
    const db = await openDb();
    expect(db).toEqual(
        expect.objectContaining({
            config: expect.anything(),
            db: expect.any(sqlite3.Database)
        })
    ); 
}

test('Database:Open', test_open);
```

Then we can run our test
```
npm run test
```

![Open Test]({{site.baseurl}}/images/rest-express/opentest.png)

Looks to be functioning well.

## Unit testing CRUD functionality
Now that we've done one, doing others is actually quite straightforward.

In order to test CRUD functionality, we need to be able to create, read, update, and delete records from our database. We can test for these in the following ways
- Create/Read: If we create a record, we should be able to read it and get the same result. 
- Update: We should be able to update the record, query it again and test for the updated result.
- Delete: I had to check what happens if you query a record that doesn't exist, in sqlite3 we get an undefined object. So we need to delete something, query it again, and then test for undefined.

The only new expectation tests I used here were
- ``expect.stringMatching(String)``: Tests string for equality
- ``expect(value).toBeUndefined()``: Assures value is undefined

Everything else should look familiar, other than maybe the sql.
```javascript
async function test_crud_users() {
    const db = await openDb();
    // User data
    const username = 'rathma';
    const password = 'somepass';
    const new_password = 'newpassword';

    // Create/insert
    await db.exec(`INSERT INTO users (username, password) VALUES ('${username}','${password}');`)

    //read/fetch
    var res = await db.get('SELECT * FROM users;');
    expect(res).toEqual(expect.objectContaining({
        username: expect.stringMatching(username),
        password: expect.stringMatching(password)
    }));

    //Update
    await db.run(`UPDATE users SET password = '${new_password}' WHERE username = '${username}';`);
    res = await db.get(`SELECT password FROM users WHERE username = '${username}';`);
    expect(res).toEqual(expect.objectContaining({
        password: expect.stringMatching(new_password)
    }));

    //Delete
    await db.exec(`DELETE FROM users WHERE username = '${username}';`);
    res = await db.get(`SELECT * FROM users WHERE username = '${username}';`)
    expect(res).toBeUndefined();
}

test('Database:CRUD', test_crud_users);
```

If we run this test, we can see it passes with flying colors
```
npm run test
```

![CRUDTest]({{site.baseurl}}/images/rest-express/crudtest.png)

This is functionally going to be the same as testing the Tweets table. So you can do that on your own or look in the source code.

## Unit testing tweet constraints
If you recall, when we created our tweet table we added the constraint

```sql
CONSTRAINT fk_userid FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE
```

This creates a constraint on the tweet to delete this database entry if the forerign key we're referencing is deleted. So let's verify that this actually works.

I tacked my test onto the end of my ``Database:Tweets`` test method, so we already have a user defined. There isn't anything here that you haven't seen yet. The test is similar to our CRUD delete test, where we test for undefined after deleting a user. The only difference is we're querying the tweets table to make sure that they get deleted.

```javascript
async function test_tweets() {
    ...
    // Test for tweet constraints
    // We'll insert a few tweets real quick
    await db.exec(`INSERT INTO tweets (user_id,contents) VALUES (${user_id}, '${content}'), (${user_id}, '${content}'), (${user_id}, '${content}');`);
    // Make sure they exist and we can get at least one
    res = await db.get(`SELECT * FROM tweets WHERE user_id = ${user_id};`);
    expect(res).toEqual(expect.objectContaining({
        id: expect.anything(),
        contents: expect.anything(),
        user_id: expect.anything()
    }));
    // Delete our user
    await db.exec(`DELETE FROM users WHERE username = '${username}';`);
    // Try to fetch a tweet, should all be deleted
    res = await db.get(`SELECT * FROM tweets WHERE user_id = ${user_id};`);
    expect(res).toBeUndefined();
}
```

Our final test results....

![Final Test]({{site.baseurl}}/images/rest-express/final_test.png)

## Troubleshooting
There really wasn't any big challenges I was met with in playing with unit tests. The only real annoyance was when a test would error out or I'd forget to clean up after doing some tests, I'd get SQL errors for duplicate entries. I got into the bad habit of just deleting the database & even adding it to my test script until they worked without it lol.

# Closing
There is so much depth in the jest framework that I didn't even scratch, but hopefully in the future I'll have the opportunity to dive deeper into other tests. 

Thanks for reading!

# References
- https://jestjs.io/
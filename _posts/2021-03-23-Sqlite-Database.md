---
title: "Creating a REST API with Express.js (2/X): Sqlite Database"
description: "Implementing support for an asychronous sqlite database."
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [express, webdev, sqlite, migrations]
image: images/express.png
---

# Overview
In this post we'll be implementing a basic SQLite database to contain our user data and tweets, adding async support to our database, and using basic migration scripts to setup our database.

[Previous Post: Express Scaffolding](https://www.kyso.dev/express/webdev/es6/2021/03/22/Express-Scaffolding.html)

[Next Post: Unit Testing](https://www.kyso.dev/express/webdev/jest/unit%20test/sqlite/2021/03/25/Unit-Testing.html)

## Why async?
By default the ``sqlite3`` database doesn't have async support build in. This is probably fine, but I was running into issues where express wouldn't wait for my query to return to continue processing the request. Specifically in cases where I was implementing error handling, the request would complete before the error was generated. I'm hoping to abuse the ``await`` keyword to so that I can wait for my query to resolve before continuing. It also allows us to use a callback style structure, which is nice.

## Code
All code from this series can be found here

[https://github.com/wrathofrathma/rest-express](https://github.com/wrathofrathma/rest-express)

All code from this post can be found here

[https://github.com/wrathofrathma/rest-express/tree/Async-SQLite-%26-Migrations](https://github.com/wrathofrathma/rest-express/tree/Async-SQLite-%26-Migrations)

# Setup
The first thing we need to do is install the node drivers for the database we're using, ``sqlite3``. Additionally we need to install the wrapper package ``sqlite`` which provides async support. To install both use the command below.
```
npm install sqlite3 sqlite
```

If we try to use the database now we'll run into issues with the babel transpiler since it doesn't come with async/await support out of the box. So we need to install a few more packages. 
```
npm install @babel/runtime @babel/plugin-transform-runtime
```

Now we need to edit our ``package.json`` to configure and enable the plugin.

```json
//package.json
...
"babel": {
  "presets": [
    "@babel/preset-env"
  ],
  "plugins":[
    ["@babel/plugin-transform-runtime",
    {
      "regenerator": true
    }]
  ]
},
```

# Creating an importable database module
When using node modules, if you use an ``import`` statement on a directory, it'll search for an ``index.js`` file in that directory and import that. So for our database, we would like to be able to just do an ``import something from './database'``.

## Basic Database Module
Let's create our ``index.js`` file in a new directory called ``server/database/`` so that we can import it as a module. The example code provided by the [sqlite reference on npm](https://www.npmjs.com/package/sqlite) for exporting the database via a module is below. You can look on the reference for the configuration options you can pass to the ``open()`` method.
```javascript
import sqlite3 from 'sqlite3';
import { open } from 'sqlite';

export async function openDb() {
    return open({
        filename: "./sqlite.db",
        driver: sqlite3.Database
    });
}
```

## Enabling foreign keys
This works if we don't want to work with foreign keys in our SQL database, but if you do, you'll run into the same issue I did. Sqlite kept erroring out with a message about a foreign key mismatch, but what was really happening is SQLite doesn't have foreign key support out of the box. You need to specify during each database session.

The command we need to run to enable it during every runtime is this... 
```sql
sqlite> PRAGMA foreign_keys = ON;
```

So let's add it to our export function so we always use it.
```javascript
import sqlite3 from 'sqlite3';
import { open } from 'sqlite';

export async function openDb() {
    const db = await open({
        filename: "./sqlite.db",
        driver: sqlite3.Database
    });
    await db.exec("PRAGMA foreign_keys = ON;");
    return db;
}
```
## Using our database module
Now our database is importable as a module! All we need to do to use our database in a file is
```javascript
import { openDb } from './dist/database';

async function somefunction() {
  const db = await openDb();
  //do stuff
}
```

You can alternatively use ``Promise`` syntax
```javascript
import { openDb } from './dist/database';

async function somefunction() {
  await openDb().then(async (db) => {
    //do stuff
  })
}
```

# Migrations
I was sorely missing the migration scripts provided in Adonis.js and was going to look into a migration framework, but the ``sqlite`` package that provides our async support also provides basic sql migrations!

## Setup
### migrations.js
The first thing we need to do is create a ``migrations.js`` file in our ``server/database/`` folder to house our code. 

Once we do that, we need to add code to open our database, similar to how we did in our module, and then call ``db.migrate()``. 
```javascript
import { open } from 'sqlite';
import sqlite3 from 'sqlite3';

const db_path = "./sqlite.db"

open({
    filename: db_path,
    driver: sqlite3.Database
})
.then(async (db) => {
    await db.migrate()
});
```
The default parameters of ``db.migrate()`` will look for migration scripts in a ``./migrations`` directory. Or you can pass a configuration object with a ``migrationsPath`` string. 

Since we know where it will look for migration scripts by default, let's just create a directory in our project root called ``migrations/``. 
```
mkdir migrations
```
Realistically this can be anywhere, as long as we pass the ``migrationsPath`` string. Just make sure to not have the string look in the ``dist/`` folder since that gets overwritten by babel & isn't included/copied in the transpile from the ``server/`` folder by default.

### Migration files
The migration files themselves need to follow the naming convention ``XXX-somename.sql``, where ``XXX`` is incrementing numbers starting at 001. If you don't follow this convention, the migration script won't register our files. Other than that, you can use [the example migrations](https://github.com/kriasoft/node-sqlite/tree/master/migrations) provided by the ``sqlite`` package as reference for how to create your sql based migration files.

Since I'm making a twitter clone, I need to have a table for users and one for tweets. So I'll make a migration script for both
- 001-users.sql
- 002-tweets.sql

Notice in the following code snippets, that Up & Down have their own comment blocks. I believe this is how the migration script differentiates the up(build) and down(delete) scripts. Besides that it's basic SQL.

I created a basic user table off of my minimalistic requirements
- a unique user id
- requiring a unique username
- a required password

#### migrations/users.sql

```sql

--------------------------------------------------------------------------------
-- Up
--------------------------------------------------------------------------------

CREATE TABLE users (
  id INTEGER PRIMARY KEY,
  username TEXT NOT NULL UNIQUE,
  password TEXT NOT NULL
);

--------------------------------------------------------------------------------
-- Down
--------------------------------------------------------------------------------

DROP TABLE users;
```

This snippet will create our ``users`` table to our spec, and then when we need to rebuild or tear down using our migration script, the line ``DROP TABLE users;`` will be run. 

One cool thing about sqlite is that when we use this table, we don't have to pass in an id field. By default an ``INTEGER PRIMARY KEY`` will be equal to the rowid. Definitely need to do some testing to see if anything weird can arise from that later.

The tweets table also has fairly minimalist requirements right now
- a unique tweet id
- the tweet contents
- storing the user who tweeted

#### migrations/tweets.sql

```sql

--------------------------------------------------------------------------------
-- Up
--------------------------------------------------------------------------------

CREATE TABLE tweets (
    id INTEGER PRIMARY KEY,
    contents TEXT NOT NULL,
    hashtags TEXT,
    user_id INTEGER NOT NULL,
    CONSTRAINT fk_userid FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE
);

--------------------------------------------------------------------------------
-- Down
--------------------------------------------------------------------------------

DROP TABLE tweets;
```

The only notable thing about this table is this line
```sql
    CONSTRAINT fk_userid FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE
```
This creates a constraint on the tweet to delete this database entry if the forerign key we're referencing is deleted. 

### Running migrations
Now that we have very basic migration scripts, we need to setup our project to run them. 

Since we've already setup our ``migrations.js`` file, we just need to configure how to run them. Typically migrations aren't run at runtime, so we'll be creating a separate script in our ``package.json`` to run them for us. 

I separated the command into two parts since our migration script is in the ``server/`` directory that babel transpiles. We'll want to transpile before executing.

```json
//package.json
...
"scripts": {
  ...
  "dbinit": "node ./dist/database/migrations.js",
  "migrations": "npm-run-all clean transpile dbinit"
}

```
Now you should be able to just run the following command to create your database =)
```
npm run migrations
```

# Closing
Honestly, the migrations seem a bit weaker than adonis.js, so maybe in the future I'll look into a proper migration framework. But for now we have migration scripts and an importable method to open our database with async support. In the next post we'll make sure it all works with unit tests!

Thanks for reading!

[Previous Post: Express Scaffolding](https://www.kyso.dev/express/webdev/es6/2021/03/22/Express-Scaffolding.html)

[Next Post: Unit Testing](https://www.kyso.dev/express/webdev/jest/unit%20test/sqlite/2021/03/25/Unit-Testing.html)

# References
- [https://www.sqlitetutorial.net/](https://www.sqlitetutorial.net/)
- [https://www.npmjs.com/package/sqlite](https://www.npmjs.com/package/sqlite)
- [https://babeljs.io/docs/en/babel-plugin-transform-runtime#docsNav](https://babeljs.io/docs/en/babel-plugin-transform-runtime#docsNav)
- [https://github.com/kriasoft/node-sqlite/tree/master/migrations](https://github.com/kriasoft/node-sqlite/tree/master/migrations)
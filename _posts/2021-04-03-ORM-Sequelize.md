---
title: "Exploring REST APIs with Express.js (6/X): ORMs with Sequelize"
description: "Replacing our backend with sequelize"
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [express, webdev]
image: images/sequelize.png
---

# Overview
We've already completed building the API, so now it's just about making life easier. Instead of using our user-built models & migrations, we can use a full fledged ORM package like sequelize to handle our database. 

In this post we will be converting our server to use the sequelize package and observing how simple the transition is because of our separation of model, service, and controller logic.

[Previous post: Route controllers, services, error handling, authentication, and middleware](https://www.kyso.dev/express/webdev/2021/03/31/Controllers.html)

All the code from this project

[https://github.com/wrathofrathma/rest-express](https://github.com/wrathofrathma/rest-express)

Code from this post

[https://github.com/wrathofrathma/rest-express/tree/d199402b428fa7be177378181e0980255df81452](https://github.com/wrathofrathma/rest-express/tree/d199402b428fa7be177378181e0980255df81452)


# Setup
## Delete old database stuff
In order to switch databases, we need to delete all the database related code

Files to delete
- Delete ``server/migrations/``
- Delete ``server/database/``
- Delete ``server/models/``
- Delete ``tests/database.test.js``
- Delete ``tests/models.test.js``

Uninstall our sqlite package

```
npm uninstall sqlite
```

Edit our package.json scripts to remove the dbinit and migration scripts

```json
{
    ...
    "scripts": {
        "dbinit": "node ./dist/database/migrations.js",
        "migrations": "npm-run-all clean transpile dbinit"
    },
    ...
}
```
## Sequelize
Sequelize is a Node.js ORM with support for most of the major databases and we're going to be using this to handle our migrations and building our models.

### Install
```
npm install sequelize
```

### Init
To initialize sequelize in our project and add its scaffold to our project, we want to run the following command in the root of our project

```
npx sequelize-cli init
```

This will generate a handful of files and folders in our project root for the ``sequelize-cli`` to interact with.

```
config/
config/config.json
migrations/
models/
models/index.js
seeders/
```

### Configuration
Before proceeding, you need to configure sequelize to interact with your preferred database by editing the ``config/config.js`` file.

The default configuration(at the time of writing)

```json
{
  "development": {
    "username": "root",
    "password": null,
    "database": "database_development",
    "host": "127.0.0.1",
    "dialect": "mysql"
  },
  "test": {
    "username": "root",
    "password": null,
    "database": "database_test",
    "host": "127.0.0.1",
    "dialect": "mysql"
  },
  "production": {
    "username": "root",
    "password": null,
    "database": "database_production",
    "host": "127.0.0.1",
    "dialect": "mysql"
  }
}
```

To tell sequelize that we want to use sqlite, we need to 
- change the ``dialect`` field to 'sqlite'
- add a ``storage`` field to the path of our database
- remove the rest

```json
{
  "development": {
    "dialect": "sqlite",
    "storage": "./sqlite.db"
  },
  "test": {
    "dialect": "sqlite",
    "storage": "./sqlite.db"
  },
  "production": {
    "dialect": "sqlite",
    "storage": "./sqlite.db"
  }
}
```

# Creating models & migrations
## Intro
When working with sequelize to handle models and migrations, you have two options. 

The first option is to write them yourself following [the model documentation](https://sequelize.org/master/manual/model-basics.html), where you setup the database schema yourself and if your models change format, you can use ``Model.sync({force: true})`` to force update your database(frowned upon for production). 

The second option is what we're doing, using the ``npx sequelize-cli`` tool to scaffold out and run our migrations & models. This option is closer to what happens in production environments. This post will be exclusively working with the ``sequelize-cli``.

## Creating a model & migration pair
To create the first model, we use the ``sequelize-cli model:generate`` command. 

The command requires two arguments
- name: name of the model
- attributes: fields of the model

```
npx sequelize-cli model:generate --name User --attributes username:string,password:string
```

This will 
- Create a user model in ``models/user.js``
- Create a migrations file in ``migrations/XXXXXXXXXXXXX-create-user.js``

Our model/table will contain fields for the attributes we specified, but also ``id``, ``createdAt``, and ``updatedAt`` fields. Below is a list of all the fields created by the above command

- username:string
- password:string
- id:integer primary key
- createdAt:date
- updatedAt:date

## Running migrations
To run our migrations and generate our database, we need to run the following command

```
npx sequelize-cli db:migrate
```

This will go through every migrations file in our ``migrations/`` folder, check whether they've been run on the current database by comparing to a ``SequelizeMeta`` table, and generate tables for migrations that haven't been run yet in the database. 


## Editing our migrations
When we generated our migration/model, we only told the cli what fields we want and their base types, but nothing about foreign keys, unique columns, etc. We can edit our migration scripts before running ``db:migrate`` to handle this.

The default ``migrations-XXXXXXXXXXXX-create-user.js`` from our command earlier.

```javascript
'use strict';
module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.createTable('Users', {
      id: {
        allowNull: false,
        autoIncrement: true,
        primaryKey: true,
        type: Sequelize.INTEGER
      },
      username: {
        type: Sequelize.STRING
      },
      password: {
        type: Sequelize.STRING
      },
      createdAt: {
        allowNull: false,
        type: Sequelize.DATE
      },
      updatedAt: {
        allowNull: false,
        type: Sequelize.DATE
      }
    });
  },
  down: async (queryInterface, Sequelize) => {
    await queryInterface.dropTable('Users');
  }
};
```
### Making a field unique
Let's say that we wanted to make the username field unique, we could just add the property ``unique: true`` to our username property.

```javascript
'use strict';
module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.createTable('Users', {
      ...
      username: {
        type: Sequelize.STRING,
        unique: true
      },
      ...
};
```

Then we want to run our migrations again, but if we just run ``sequelize-cli db:migrate``, our migrations won't be run because the ``SequelizeMeta`` table says we've run our user script already. So we can either ``db:migrate:undo`` the last migration, or delete the database file...

```
npx sequelize-cli db:migrate:undo
npx sequelize-cli db:migrate
```

Now if we attempt to insert a duplicate username, the database will throw an error.

### Working with foreign keys & associations
Before we can work with foreign keys, we need a second table. Let's model our second table after our tweets table from our previous posts.

Our tweets table only needs to contain the tweet contents since we'll be using associations to handle getting the user that owns the tweet.

```
npx sequelize-cli model:generate --name Tweet --attributes contents:string
```

This will generate a model file that looks like this

```javascript
'use strict';
const {
  Model
} = require('sequelize');
module.exports = (sequelize, DataTypes) => {
  class Tweet extends Model {
    /**
     * Helper method for defining associations.
     * This method is not a part of Sequelize lifecycle.
     * The `models/index` file will call this method automatically.
     */
    static associate(models) {
      // define association here
    }
  };
  Tweet.init({
    contents: DataTypes.STRING,
  }, {
    sequelize,
    modelName: 'Tweet',
  });
  return Tweet;
};
```

Now we'll want to use associations to link each ``Tweet`` to a ``User``. The available types of associations are
- belongsTo
- hasMany
- hasOne
- belongsToMany

Typically these will occur in pairs, where if we define a ``Tweet`` belongsTo, we will likely have a hasMany association on the ``User`` model.

To define this association with the user model, we want to add our ``belongsTo`` line in the ``static associate(models)`` function and the ``models/index.js`` file will handle the associations.

```javascript
    ...
    class Tweet extends Model {
        ...
        static associate(models) {
            this.belongsTo(models.User);
        }
        ...
    }
    ...
```

We should also add the ``hasMany()`` association to the ``User`` model.

```javascript
...
  class User extends Model {
    /**
     * Helper method for defining associations.
     * This method is not a part of Sequelize lifecycle.
     * The `models/index` file will call this method automatically.
     */
    static associate(models) {
      // define association here
      this.hasMany(models.Tweet);
    }
  };
  ...
```

Now when we work with a ``Tweet`` model object, the ``Tweet.getUser()`` method is added to the model because of the ``hasOne(models.User)`` association. The model will look for the userId field in the tweets table, but it isn't defined yet, so we need to still add that to our migrations.

In our ``migrations/XXXXXXXXXXXXX-create-tweet.js`` file we want to add a field for the userId. 

```javascript
...
      userId: {
        type: Sequelize.INTEGER,
        allowNull: false,
        references: {
          model: "users",
          key: "id"
        },
        onDelete: "cascade",
        onUpdate: "cascade"
      },
...
```

The meat of this is really the ``references`` object that defines it as a foreign key. ``onDelete: cascade`` and ``onUpdate: cascade`` forces the tweet to be updated if the foreign key gets altered or removed. Lastly, if you don't add the ``allowNull: false`` field to ``userID``, you won't be required to define a foreign key, and we can't have ghost tweets. 

Now we will want to run our migration scripts once more to make sure our tweets table is added. We didn't change the user script, so we don't have to undo our previous migrations.

```
npx sequelize-cli db:migrate
```

# Unit testing the new models
To validate that everything is working, I set up a handful of new unit tests based on the sequelize [model query guide](https://sequelize.org/master/manual/model-querying-basics.html) or the [reference](https://sequelize.org/v3/docs/querying/).

## Setup
First I import both models and setup the test environment and some variables I know I'll be using for creating and updating users.

```javascript
const Models = require('../models')
const User = Models.User;
const Tweet = Models.Tweet;

describe('sequelize:CRUD', () => {
    var user;
    const username = "rathma";
    const password = "somepass";
    const username2 = "rathma2";
}
```

## User creation
User creation is basically the same as our previous models. The create method takes in an object for the properties we want to initialize the entry with and it returns a ``User`` model object.

```javascript
...
    test('Create User', async () => {
        user = await User.create({
            username,
            password
        });
        expect(user).toEqual(expect.any(User));
    })
...
```

## Fetching users
Fetching users is similar to other database models where we use ``Model.findOne()`` and ``Model.findAll()`` methods to select rows.

To specify which user, we pass in the object ``where: { username: "someusername" }`` which acts as a ``SELECT`` SQL operation. 

``User.findOne()`` should return one ``User`` model instance for the row, whereas ``User.findAll()`` would return all matching rows in a list of ``User`` objects.

```javascript
...
    test('Get User', async () => {
        user = await User.findOne({
            where: {
                username
            }
        });
        expect(user).toEqual(expect.any(User));
    })
...
```

## Updating users
When updating users, we use the ``Model.update(updatedFields, whichFields)`` method. Similar to fetching users, we can use the ``where`` object to specify which users to select/update.

Update and delete operations return an array containing 1 for successful or 0 for failure. I haven't checked, but it might be an array to represent which rows were successful and which were failures when updating multiple rows?

```javascript
...
    test('Update User', async () => {
        const res = await User.update({username: username2}, {
            where: {
                id: user.id
            }
        })
        expect(res).toEqual([1])
    })
...
```

## Creating a tweet
```javascript
...
    var tweet;
    test('Create tweet', async () => {
        tweet = await Tweet.create({
            UserId: user.id,
            contents: "Hello world!"
        })
        expect(tweet).toEqual(expect.any(Tweet))
    })
...
```

## Deleting the user
Deleting our user should also cascade delete our tweets created by the user. I didn't check that here, but I looked in the database manually to verify.

```javascript
...
    test('Delete User', async () => {
        const res = await User.destroy({
            where: {
                id: user.id
            }
        });
        expect(res).toEqual(1); // 1 = deleted, 0 = not
    })
...
```

# Updating our services to support the new models
In order to make our API use the new models, we only need to update 4 files (which are the only 4 files that interact with the model layer)

- Auth middleware (since we fetch the user)
- Auth service
- User service
- Tweet service

The main changes I made to accomodate the change were just the imports and calls,

Here is the git diff of the AuthService.js. We only changed the User import, how we're fetching the user, and we're comparing to ``resource.User.id`` instead of ``resource.user_id`` because of the way the sequelize associations work.

Pay close attention to the ``import`` vs ``require`` used for fetching the models.

```diff
diff --git a/server/services/AuthService.js b/server/services/AuthService.js
index 444010f..4f4a073 100644
--- a/server/services/AuthService.js
+++ b/server/services/AuthService.js
@@ -1,11 +1,13 @@
-import User from '../models/UserModel';
+const User = require("../../models").User;
 import Exception from '../exceptions/GenericException';
 import * as argon2 from 'argon2';
 import * as jwt from 'jsonwebtoken';
 
 const AuthService = {
     async login({username, password}) {
-        const user = await User.get({username});
+        const user = await User.findOne({
+            where: {username}
+        });
 
         const correct_pwd = await argon2.verify(user.password, password);
         if(!correct_pwd){
@@ -51,7 +53,8 @@ const AuthService = {
                 status: 404
             });
         }
-        if(user.id !== resource.user_id) {
+
+        if(user.id !== resource.User.id) {
             throw new Exception({
                 message: "Invalid access",
                 status: 401
```

The other main change is that our models no longer have a ``Model.json()`` method, so I added ``Service.jsonify()`` methods to both Tweet & User services.

Here's an example of the UserService

```diff
diff --git a/server/services/UserService.js b/server/services/UserService.js
index 23bce7b..1f6e8be 100644
--- a/server/services/UserService.js
+++ b/server/services/UserService.js
@@ -1,23 +1,18 @@
 import Exception from '../exceptions/GenericException';
+import TweetService from './TweetService';
 
 const UserService = {
     async delete({user}) {
-        return await user.delete()
+        return await user.destroy()
         .then(() => {
-            return {
-                id: user.id,
-                username: user.username
-            }
+            return UserService.jsonify(user);
         })
     },
 
     async update({user, username, password}) {
         return await user.update({username, password})
         .then(() => {
-            return {
-                id: user.id,
-                username: user.username,
-            }
+            return UserService.jsonify(user);
         })
         .catch((err) => {
             throw new Exception({
@@ -28,10 +23,20 @@ const UserService = {
     },
 
     async tweets({user}) {
-        return await user.tweets()
-        .then((utweets) => {
-            return utweets.map(t => t.json());
+        return await user.getTweets()
+        .then((ts) => {
+            return ts.map(t => {
+                t.User = user;
+                return TweetService.jsonify(t);
+            })
         })
+    },
+
+    jsonify(user) {
+        return {
+            id: user.id,
+            username: user.username
+        }
     }
 }
```

Applying similar changes to the TweetService brings our application back up to full functionality.

# Closing
Well, that's about it for getting started with sequelize. It was actually quite painless once I got rolling with the CLI. Doing things by hand before that was kinda painful though. Next time I'm going to look into getting into firebase & firestore since that's my goal. I was just learning express until then.

[Previous post: Route controllers, services, error handling, authentication, and middleware](https://www.kyso.dev/express/webdev/2021/03/31/Controllers.html)

# References 
- [https://sequelize.org/master/](https://sequelize.org/master/)
- [https://sequelize.org/master/manual/migrations.html](https://sequelize.org/master/manual/migrations.html)
- [https://sequelize.org/master/manual/model-querying-basics.html](https://sequelize.org/master/manual/model-querying-basics.html)
- [https://sequelize.org/v3/docs/querying/](https://sequelize.org/v3/docs/querying/)
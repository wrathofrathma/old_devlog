---
title: "Building a CRUD TODO with Vue 3.0 & Adonis.js"
description: "These are just some of my notes and challenges I faced while building a CRUD TODO website using Vue 3.0, Primevue, and Adonis.js"
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [vue, primevue, webdev]
image: images/vue.png
---

# Overview
So this is my first real attempt at learning web development. I played around with it about a year ago and have certainly used a handful of microframeworks with python for a variety of projects, but I've never really committed to learning frontend or even proper backend. This time we're committed.

All code for this project can be found here

https://github.com/wrathofrathma/vue-adonis-todo

## My decision making on the stack I choose for my first project. 
### Backend Runtime: Node.js
I went with a javascript based engine for a few reasons.
- I already need to work with javascript for the frontend
- There are tons of jobs in the express.js market
- I predict there will be first class support for things like websockets and webrtc 
- Learning php + javascript seemed like a bit much for a first project. So keeping it simple and consistent.

### Backend Framework: Adonis.js
Honestly, this was a hard decision to make. There are **a lot** of good node based frameworks, so why adonis? It came down to two points
- Laravel is the king of php frameworks from what I hear, and adonis is attempting to be the javascript version of laravel. So why not give it a try.
- I rather use a batteries included framework to start so I don't get discouraged easier.

### Frontend Framework: Vue 3
This was probably the simplest decision for me. From what I've read, out of Angular, React, and Vue, Vue is the most comfortable to program in. Even looking at React versus Vue code, Vue just seemed more approachable. Also the market for React developers is probably very saturated and competitive. 

I choose the Vue 3 preview over Vue 2 just because I rather get ahead of the game.

### Frontend Component Library: Primevue
This was a rather limited, but still tough decision. Since most component libraries aren't updated for Vue 3 yet, I was pretty much limited to Element, Ionic, Primevue, or making my own in Tailwind. 

Making my own components in Tailwind seems very attractive...but I think a large part of the battle as a developer is just knowing what is possible and where to look. Right now I don't have any experience, so anything I design from scratch will be limited I think. 

I went with Primevue because it has a lot of components, the free theme selection is pretty top notch, and they have versions for react & angular(should I switch in the future).

## A project to get started
From what I understand, CRUD TODO lists are basically the "hello world" of web development, so I sought out a tutorial matching these technologies and found a real gem of a tutorial by freecodecamp. 

[Full Stack Todo List tutorial using Vue.js & Adonis.js](https://www.youtube.com/watch?v=dfEZlcPvez8)

### Differences
There are a few key differences between the stack we're using and the one in the tutorial video from 3 years ago. 
- Adonis.js' structure has changed a little since the video release
- Vue.js has been updated to Vue 3, which forces further changes
- We're using Primevue instead of Vuetify because Vuetify isn't updated for Vue 3.
- We've also added the material icon set because primeicons is quite limited.


# Building the backend
I'm a little torn between whether I should actually document the whole process or just note the challenges/differences I encountered. For posterity sake, I'm just going to go over everything but instead of show the full process, skip the repetitive parts and show mostly finished code.

## Setup
The first hour or so of the tutorial is setting up the REST API using Adonis. 
### Installation
Installation of Adonis is quite straightforward. I installed it their CLI tool globally, but you can also invoke it with npx.
```bash
sudo npm i -g @adonisjs/cli
```
### Scaffolding a new project
Using the Adonis CLI you can scaffold out a basic project easily with
```
adonis new server --api-only
```
The --api-only prevents it from using the fullstack blueprint and only includes the necessary files for building an API.

### Running the Adonis development server
Running the adonis dev server is about as simple as it could be.
```
adonis serve --dev
```

While the dev server is running, it will watch for all changes in the directory and live update the runtime. 

Right now, before any changes are made, if we visit the address the server is running on(defaulted to localhost:3333) we will get the response

```json
{
    greeting: "Hello world in JSON"
}
```

## Routing
To add any meaningful behavior to our server we need to add routes. 

All of the routes in Adonis.js are defined in start/routes.js. Below you can find the default configuration which includes the response we saw above.

```javascript
'use strict'

/*
|--------------------------------------------------------------------------
| Routes
|--------------------------------------------------------------------------
|
| Http routes are entry points to your web application. You can create
| routes for different URLs and bind Controller actions to them.
|
| A complete guide on routing is available here.
| http://adonisjs.com/docs/4.1/routing
|
*/

/** @type {typeof import('@adonisjs/framework/src/Route/Manager')} */
const Route = use('Route')

Route.get('/', () => {
  return { greeting: 'Hello world in JSON' }
})
```

While in the default file they define a callback function in the route definition, typically the route logic is defined in a "controller". Controllers are just objects that have functions for the various routes related to the theme of the controller. 

We can use the Adonis CLI to create a controller for users to register/login with.

```
adonis make:controller User
```
It'll ask whether you want to create an HTTP or websocket controller, and for all cases of our REST API we'll pick HTTP since we don't need the realtime connection provided by websockets.

The controller skeleton is created at app/Controllers/HTTP/UserController.js
```javascript
'use strict'

class UserController {
}

module.exports = UserController
```

We defined a few methods inside of the UserController object to handle authentication. 
```js
'use strict'

// use() is an adonis specific method that they recommend replacing require() with
// Here we're importing the user lucid model
const User = use("App/Models/User")

class UserController {
    //Adding auth to the object passed gives us access to the Auth from the AuthProvider defined in app.js
    // Auth type is defined in config/auth.js
    async login({request, auth}) {
        // request.all groups together all the request parameters, then we use object deconstruction to get the necessary items
        const {email, password} = request.all();
        //Auth attempt
        const token = await auth.attempt(email, password);
        return token;

    }

    async register({request}) {
        // request.all groups together all the request parameters, then we use object deconstruction to get the necessary items
        const { email, password } = request.all();

        //Calls the asynchronous model method to create a new user.
        // Adonis comes with a UserSchema that already includes username, email, and password
        const user = await User.create({
            email,
            password,
            username: email,
        })

        return this.login(...arguments);
    }
}

module.exports = UserController
```

We can access these by creating routes in start/routes.js. 

In this completed example, you'll notice a few things going on
- We've grouped the routes together and applied an 'api' route prefix. So now you access the registration route by posting a request to localhost:3333/api/auth/register
- We've added authentication middleware to routes that require a user to be logged in to access
- Some routes have parameters/variables in the route, such as tasks/:id. This :id variable is passed to the controller as a parameter.

```js
'use strict'

/*
|--------------------------------------------------------------------------
| Routes
|--------------------------------------------------------------------------
|
| Http routes are entry points to your web application. You can create
| routes for different URLs and bind Controller actions to them.
|
| A complete guide on routing is available here.
| http://adonisjs.com/docs/4.1/routing
|
*/

/** @type {typeof import('@adonisjs/framework/src/Route/Manager')} */
const Route = use('Route')

// Route.get('/', () => {
//   return { greeting: 'Hello world in JSON' }
// })

// Groups routes together by a prefix
Route.group(() => {
  //Sends the requests to our controller method created using adonis make:controller User
  // then defining the register method
  Route.post('auth/register', "UserController.register")
  Route.post('auth/login', "UserController.login")

  Route.get('projects', "ProjectController.index").middleware('auth')
  Route.post('projects', "ProjectController.create").middleware('auth')
  Route.delete('projects/:id', "ProjectController.destroy").middleware('auth')
  Route.patch('projects/:id', "ProjectController.update").middleware('auth')

  Route.post("projects/:id/tasks", "TaskController.create").middleware('auth')
  Route.get("projects/:id/tasks", "TaskController.index").middleware('auth')

  Route.delete("tasks/:id", "TaskController.destroy").middleware("auth")
  Route.patch("tasks/:id", "TaskController.update").middleware("auth")
}).prefix("api")
```
## Database
### Setup
By default the database is configured to be sqlite. To change this, there is a line in the config/database.js file where you can change the sqlite to another provider.
```js
  connection: Env.get('DB_CONNECTION', 'sqlite'),
```

Whichever provider you choose, you need to install the nodejs driver for it via npm/yarn. I believe you can omit the --save, as it's a default flag.
```
npm install sqlite3 --save
```
### Migrations
Now that the database is selected and the driver is installed, we need to setup our database schema using migration scripts. 

You can find a default user table defined in database/migrations/somenumber_user.js
```js
'use strict'

/** @type {import('@adonisjs/lucid/src/Schema')} */
const Schema = use('Schema')

class UserSchema extends Schema {
  up () {
    this.create('users', (table) => {
      table.increments()
      table.string('username', 80).notNullable().unique()
      table.string('email', 254).notNullable().unique()
      table.string('password', 60).notNullable()
      table.timestamps()
    })
  }

  down () {
    this.drop('users')
  }
}

module.exports = UserSchema
```
Another example from the project schema created in the tutorial to show how to reference fields in another value. 
```js
'use strict'

/** @type {import('@adonisjs/lucid/src/Schema')} */
const Schema = use('Schema')

class ProjectSchema extends Schema {
  up () {
    this.create('projects', (table) => {
      table.increments()
      table.integer('user_id').unsigned().references('id').inTable('users')
      table.string('title', 255)
      table.timestamps()
    })
  }

  down () {
    this.drop('projects')
  }
}

module.exports = ProjectSchema
```

Once you define a few of your database schemas in the migration scripts, you can generate the database tables by running. 
```
adonis migration:run
```
If you ever need to drop the database and recreate from scratch you can run
```
adonis migration:refresh
```
Now that the database schema is generated in the database, we need a way to access the data.
### Lucid Models
Adonis.js uses something called Lucid ORM(Object-Relational Mapping) which creates a class-like interface for database content.

The User model default configuration is shown below. The only real things to note is the hook definition for hashing the password before adding it to the database and how the tokens() method is defined.
```js
'use strict'

/** @type {typeof import('@adonisjs/lucid/src/Lucid/Model')} */
const Model = use('Model')

/** @type {import('@adonisjs/framework/src/Hash')} */
const Hash = use('Hash')

class User extends Model {
  static boot () {
    super.boot()

    /**
     * A hook to hash the user password before saving
     * it to the database.
     */
    this.addHook('beforeSave', async (userInstance) => {
      if (userInstance.dirty.password) {
        userInstance.password = await Hash.make(userInstance.password)
      }
    })
  }

  /**
   * A relationship on tokens is required for auth to
   * work. Since features like `refreshTokens` or
   * `rememberToken` will be saved inside the
   * tokens table.
   *
   * @method tokens
   *
   * @return {Object}
   */
  tokens () {
    return this.hasMany('App/Models/Token')
  }
}

module.exports = User
```

The configuration built in the tutorial for the projects model is much simpler
```js
'use strict'

/** @type {typeof import('@adonisjs/lucid/src/Lucid/Model')} */
const Model = use('Model')

class Project extends Model {
    user() {
        return this.belongsTo("App/Models/User")
    }

    tasks() {
        return this.hasMany("App/Models/Task")
    }
}

module.exports = Project
``` 

We can see the different ways we interact with the Lucid ORM models here in our ProjectController.
```js
'use strict'

const Project = use("App/Models/Project")
const AuthService = use("App/Services/AuthorizationService")

class ProjectController {
    async index({auth}) {
        const user = await auth.getUser();
        return await user.projects().fetch();
    }

    async create({request, auth}) {
        //Authenticating with our token
        const user = await auth.getUser();
        //Object deconstruction to get the title of hte project
        const {title} = request.all();
        //There are 3 ways we can do this in a way that associates the project with the user.
        /* 1. Use the constructor and give it the user_id manually.
        const project = await Project.create({
            user_id: user.id,
            title
        })
         * */
        /* Use project.fill({}) or project.title="something" on a new project object
         * const project = new Project();
         project.title = "Hello world"
         * or
         project.fill({
            title
         })

         Then save it with await user.projects().save(project);
         */
        const project = new Project();
        project.fill({
            title
        });
        //Then saving it to associate it with the user
        await user.projects().save(project);
        return project;
    }

    //Adding params to the input object gives us the query parameters from the route.
    async destroy({response, auth, params}) {
        const user = await auth.getUser();
        const { id } = params;

        // Will return us the project model if it exists
        const project = await Project.find(id);
        //Ensure the user that is auth'd is the owner of the project.
        AuthService.verifyPermission(project, user);

        await project.delete();
        // return response.status(403);
        return project;
    }

    async update({auth, params, request}) {
        const user = await auth.getUser();
        const { id } = params;
        const project = await Project.find(id);
        AuthService.verifyPermission(project,user);

        project.merge(request.only('title'))
        await project.save();
        return project;
    }
}

module.exports = ProjectController
```

## Authentication
Authentication in Adonis is really simple. Similar to the database configuration, we choose an auth provider in config/auth.js. 

Below you can see a trimmed down verison of the default configuration where we are using jwt tokens and the jwt object associates that with the lucid orm. Adonis also provides various other auth schemes and even social authentication so you don't need things like firebase auth.
```js
'use strict'

/** @type {import('@adonisjs/framework/src/Env')} */
const Env = use('Env')

module.exports = {
  authenticator: 'jwt',

  jwt: {
    serializer: 'lucid',
    model: 'App/Models/User',
    scheme: 'jwt',
    uid: 'email',
    password: 'password',
    options: {
      secret: Env.get('APP_KEY')
    }
  },
```

To use the authentication we can refer back to the UserController.login method where we pass the auth object in with the request, and attempt authentication.
```js
async login({request, auth}) {
    // request.all groups together all the request parameters, then we use object deconstruction to get the necessary items
    const {email, password} = request.all();
    //Auth attempt
    const token = await auth.attempt(email, password);
    return token;
}
```

Then every route that needed authentication we defined the route with the auth middleware
```js
Route.get('projects', "ProjectController.index").middleware('auth')
```

Below is an example from the ProjectController.index method where we pass in the auth object and get the user specific model from the database.
```js
async index({auth}) {
  const user = await auth.getUser();
  return await user.projects().fetch();
}
```

## Testing the API with Postman
The last thing about the backend to note, is perhaps one of the most important. Testing your API is something you do from the first route configuration, so having a good tool to use is imperative. 

Postman is a fantastic tool for doing API testing that even if I don't use the web technologies in this tutorial ever again, I'll certainly keep postman around for the future. 

Here is a general overview of what postman looks like
![postman overview](images/postman/postman.png)

The key features I wanted to show from postman were 
- Collections / folders
- Creating a request
- Environments
- Test scripts

### Collections & Folders
As you can see on the left, we have a collections tab where we can define groups of requests and subdivide them into their own folders. This is a fantastic way to organize your projects. To create a folder, simple right click on the collection and select "create folder". 

![postman collection](images/postman/postman-collections.png)

### Creating a request
To create a request in the subfolder or collection, you can either click on the + button in the middle of the screen shown in the first screenshot, or right click on the collection/folder and select "create a request".

From there it's simple to define what type of request it is, the body, headers, etc. 

when you're finished, you can save it to the specific collection/folder and even write a description on the request.

### Environments
Environments are one of the coolest things in Postman that I surprisingly didn't see in other REST client extensions I had on my browser. Basically what they do is allow you to define global variables for use in the selected environment.

You can create an environment by selecting the environment button on the top right, creating one and defining variables in the pop-up screen below.
![environment](images/postman/postman-environment.png)

Once you've defined your variables, you can access them anywhere in your requests using handlebar style expressions {{variable_name}}. In the below screenshot you can see we've substituted both the server address, the email, and password with our environmental variables.

![environmental variables](images/postman/postman-variables.png)

### Test scripts
This is something not covered in the tutorial, but something I found fascinating is that you can define pre & post request scripts to run. I imagine this is used often for response validation, but I found few other uses for it

- Setting the token after users login to manage multiple users
- Changing project & task IDs after index / create requests

This is all done in the Tests tab on the request.

![scripts](images/postman/postman-scripts.png)

# Frontend Development
I'm probably not going to go into as much depth as was necessary on the backend, I'm just going to touch on some of the key topics that I might forget in the future.

Topics I will cover
- Setup
- Components
- Routing / views
- Vuex store
- Differences & problems I encountered

## Setup
### Installation
The vue documentation suggests installing their official CLI, but most enthusiasts seem to be switching to Vite. 

I used their CLI for this project and installed it globally
```
npm install -g @vue/cli
```

Then to create the project we use the CLI binary and select Vue 3 preview as our preset

```
vue create client
```

At this point, we're able to launch the dev server from within the client folder which will allow for hot reloading(changes are seen live).
```
npm run serve
```

Now with the project scaffolded out, we can install our various frontend dependencies
### Installing dependencies
Our project uses the following depdencies

- vuex - State management
- vue-router - Frontend routing
- vuex-persistedstate - Allows the vuex state to be saved to the client, great for login tokens.
- vuex-router-sync - Syncs the router with vuex so you can access the current route from the store. Not sure why this is terribly useful yet.
- axios - For HTTP requests from the client
- primevue - The choosen frontend component library
- primeflex - Flexbox css stuff for primevue
- ~~primeicons~~ - Primeface's official icon library, I tried this and it was a bit too limited in selection.
- ~~lodash~~ - They included this dependency but I don't remember ever using it.

Later I included material icons stylesheet in the index.html header, but you can probably install that here too.

To install all of these depdencies you can just run this in your client folder.
```
npm install vuex-persistedstate vuex-router-sync axios primevue primeflex
vue add vuex
vue add vue-router
```


### Setting up vuex
If we installed vuex from vue-cli, then it should have auto-generated a folder src/store with a file called index.js inside of it. It's also already setup

The only thing we've really changed is we are using vuex in strict mode, which doesn't allow code outside of our mutations to alter our Vuex state. This allows for easier debugging.
```javascript
import { createStore } from 'vuex'

export default createStore({
  strict: true,
  state: {
  },
  mutations: {
  },
  actions: {
  },
  modules: {
  }
})
```

### Setting up vuex-persistedstate
In order to setup vuex-persistedstate we need to modify our vuex store index file under src/store/index.js

Setup is simple by just importing it and adding it to our plugins property of our base store.
```js
import createPersistedState from 'vuex-persistedstate'
import { createStore } from 'vuex'

export default createStore({
  strict: true,
  state: {
  },
  mutations: {
  },
  actions: {
  },
  modules: {
  },
  plugins: [
    createPersistedState()
  ]
})
```
Note, during development it might be better to disable persisted state.

### Vuex-router-sync
Vuex-router-sync is quick to setup. We just need to add two lines to our src/main.js. Add these two lines before creating the app and doing .use(router), and we're golden.

```javascript
//Import vuex-router-sync
import { sync } from 'vuex-router-sync'
//Syncs the store with the router
sync(store, router);
```


### Axios
In order to make web requests, we need to define our base Axios object.

So we created a file in the root server directory called http.js which uses the store to grab the baseURL & the auth token. 
```js
import axios from 'axios';
import store from './store/';

export default ()  => {
    return axios.create({
        baseURL: store.state.baseURL,
        timeout: 4000,
        headers: {
            Authorization: `Bearer ${store.state.authentication.token}`
        }
    })
}
```
### Setting up primevue
First we need to tell Vue to use Primevue for components, and we can do that by editing src/main.js

Our final main.js looks like this.
```javascript
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'
import store from './store'

//Import vuex-router-sync
import { sync } from 'vuex-router-sync'
//Import PrimeVue to use
import PrimeVue from "primevue/config"

//Syncs the store with the router
sync(store, router);

createApp(App).use(store).use(router).use(PrimeVue).mount('#app')
```

Then in order to actually use primevue css, we need to import it somewhere and since App.vue is our root component, it's a fantastic place. So in the script section of App.vue
```js
import "primevue/resources/themes/vela-blue/theme.css"
import "primevue/resources/primevue.min.css"
import "primeflex/primeflex.css"
```

Then I also set the background color of the body in the css section of the App.vue to match the theme
```css
body {
  background-color: var(--surface-b);
  color: var(--text-color);
}
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
}
```

### Proxying our requests to our adonis dev server
In order to code our requests as they would be on the live server, we need to create a proxy to our adonis dev server. 

To do this we need to create a file in our root client directory called "vue.config.js" and define our target like this.
```js
module.exports = {
    devServer: {
        proxy: {
            '/api': {
                target: 'http://localhost:3333'
            }
        }
    }
}
```
Now all requests from our axios requests that start with /api will proxy to the adonis server as long as our environment is set to development. 

## Components
Components are the heart and soul of Vue.js. They can be thought of as the complete encapsulation of the logic, styling, and templating of any object in your webpage. I [highly suggest reading up on them](https://v3.vuejs.org/guide/component-basics.html) if you're not familiar, because you will be **lost** if you don't. 

Projects scaffolded out using the Vue CLI will often use single file components with a structure like this. Where the template tags contain HTML, script contains the javascript logic and lifecycle hooks, and the style tags define css relative to this component.
```html
<template>
  <div class="something">
    Hello world
  </div>
</template>

<script>
export default {
}
</script>

<style>
.something {
  color: blue;
}
</style>
```

A minimalistic example of a real component we used in our project is the CreateRecord component. 

This component 
- Is composed of multiple sub-components
- Emits events based on actions we defined.
- Defines 'props' which is data we can pass to the component at time of creation.
- Can be reused anywhere in our application and is used in both the Projects & Tasks panels.

```html
<template>
    <div class="p-grid">
        <InputText
            :placeholder="placeholder" 
            @update:model-value="$emit('input', $event)"
            :value="value"
            @keyup.enter="$emit('create')"
            class="p-inputtext-lg p-col" 
            :disabled="disabled"
        />
        <Button 
        class="p-col-fixed p-ml-2" 
        style="width: 90px;"
        @click="$emit('create')"
        :disabled="disabled"
        >
            <i class="material-icons" @click="iclicked">add_circle</i>
            Create
        </Button>
    </div>
</template>

<script>
import Button from "primevue/button"
import InputText from "primevue/inputtext"

export default {
    components: {
        Button,
        InputText
    },
    emits: ['input', 'create'],
    props: {
        placeholder: {
            type: String,
            required: false
        },
        value: {
            type: String,
            required: false
        },
        disabled: {
            type: Boolean,
            default: false
        }
    }
}
</script>
```

I could keep going on, but this is more of a brief overview. 
## Routing & Views
Earlier we installed a frontend router called "vue-router". What frontend routers let us do is navigate our webpage without refreshing, which is extremely useful considering most single page applications are quite hefty.

In order to create a page we can navigate to on the frontend, we define a component for it under the views/ directory. These are no different than any other component other than they get mounted in our App.vue in place of
```html 
<router-view />
```

An example view is our Projects.vue. Notice how concise this view is. We stuffed all of our logic in the sub-components project-panel and tasks-panel, so we have a much cleaner view component.
```html
<template>
    <div class="p-grid p-mt-4">
        <project-panel class="p-col-4"></project-panel>
        <tasks-panel class="p-col-8"></tasks-panel>
    </div>
</template>

<script>
import ProjectPanel from "../components/Projects.vue"
import TasksPanel from "../components/Tasks.vue"

import { mapGetters } from 'vuex'

export default {
    components: {
        ProjectPanel,
        TasksPanel
    },
    computed: {
        ...mapGetters('authentication', [
            'isLoggedIn'
        ])
    },
    mounted() {
        if(!this.isLoggedIn) {
            this.$router.push('/login')
        }
    }
}
</script>
```

Now that we have a Projects.vue defined, in order to navigate to it we need to define the route in our router/index.js. The final router for this project ended up looking like this. Notice it lazy loading everything other than the root component.
```javascript
import { createRouter, createWebHashHistory } from 'vue-router'
import Projects from '../views/Projects.vue'

const routes = [
  {
    path: '/',
    name: 'Projects',
    component: Projects
  },
  {
    path: '/about',
    name: 'About',
    // route level code-splitting
    // this generates a separate chunk (about.[hash].js) for this route
    // which is lazy-loaded when the route is visited.
    component: () => import(/* webpackChunkName: "about" */ '../views/About.vue')
  },
  {
    path: '/register',
    name: 'Register',
    component: () => import ('../views/Register.vue')
  },
  {
    path: '/login',
    name: 'Login',
    component: () => import ('../views/Login.vue')
  }
]

const router = createRouter({
  history: createWebHashHistory(),
  routes
})

export default router
```

With both our view created and our route setup, in order to navigate to one of these routes we can either use router-links
```html
<router-link to="/">Some text</router-link>
```
or we can push a route programatically
```javascript
$router.push("/")
```

We mostly use the latter since we're navigating based on events. An example of this is in our Nav component we defined methods for our buttons to use as on click actions. 
```javascript
import { mapGetters, mapActions } from 'vuex'
import Toolbar from "primevue/toolbar"
import Button from "primevue/button"
export default {
    components: {
        Toolbar,
        Button

    },
    data() {
        return {
        }
    },
    computed: {
        ...mapGetters('authentication', [
            'isLoggedIn'
        ]),
    },
    methods: {
        ...mapActions('authentication', [
            'logout'
        ]),
        on_projects() {
            this.$router.push("/")
        },
        on_register() {
            this.$router.push("/register")
        },
        on_login() {
            this.$router.push("/login")
        },
        open_video() {
            window.open("https://www.youtube.com/watch?v=dfEZlcPvez8")
        }
    }
}
```

## Vuex store
### Overview
So, vuex, or state management in general creates a synchronized central data store for all of the components and logic of the application to access.

This allows us to 
- Centralize our logic
- Synchronize multiple fields to one value
- Live track other components
- Cache hella data

### Creating a store & store anatomy
By default, when we install vuex from the Vue CLI (not npm), it creates src/store/index.js as a default store file. We've already added persisted-state to it, but let's talk about the anatomy of this real quick. 

Each store may contain
- state - The truth / value of the objects
- mutations - Synchronous methods defined that are allowed to alter the state values
- actions - Methods that may be asynchronous that perform actions, which are sometimes groups of mutations & asynchronous calls to the web. Think of this as the primary logic methods.
- getters - Get methods for retrieving the state or a value based on the state. 
- modules - nested stores

An example that shows off most of thes concepts is our authentication.js module. A note, by passing namespaced: true, all of the method and objects are namespaced to store.authentication.state.stuff
```javascript
import HTTP from '@/http';
import router from '../router';
export default {
    namespaced: true,
    state: {
        registerEmail: '',
        registerPassword: '',
        registerError: null,
        token: null,
        loginEmail: '',
        loginPassword: '',
        loginError: null
    },
    mutations: {
        setRegisterEmail(state, email) {
            state.registerEmail = email;
        },
        setRegisterPassword(state, password) {
            state.registerPassword = password;
        },
        setToken(state, token) {
            state.token = token;
        },
        setRegisterError(state, error) {
            state.registerError = error;
        },
        setLoginEmail(state, email) {
            state.loginEmail = email;
        },
        setLoginPassword(state, password) {
            state.loginPassword = password;
        },
        setLoginError(state, error) {
            state.loginError = error;
        }
    },
    getters: {
        isLoggedIn(state) {
            return !!state.token;
        }
    },
    actions: {
        register({ commit, state }) {
            commit('setRegisterError', null);
            return HTTP().post('/api/auth/register', {
                email: state.registerEmail,
                password: state.registerPassword
            }).then(({data}) => {
                commit('setToken', data.token);
                router.push('/');

            }).catch(() => {
                commit('setRegisterError', 'An error has occured trying to create your account.');
            })
        },
        logout({commit}) {
            commit('setToken', null);
            router.push("/login")
        },
        login({state, commit}) {
            commit('setLoginError', null)
            return HTTP().post('/api/auth/login', {
                email: state.loginEmail, 
                password: state.loginPassword
            }).then(({data}) => {
                commit('setToken', data.token)
                router.push('/')
            }).catch(() => {
                commit('setLoginError', "An error occured while logging in.")
            })
        }
    }
}
```

Then to register the modules, we do so in the root store, index.js
```javascript
import createPersistedState from 'vuex-persistedstate'
import { createStore } from 'vuex'
import authentication from "./authentication"
import projects from "./projects"
import tasks from "./tasks"

export default createStore({
  strict: true,
  state: {
    baseUrl: '/api'
  },
  mutations: {
  },
  actions: {
  },
  modules: {
    authentication,
    projects,
    tasks
  },
  plugins: [
    createPersistedState()
  ]
})
```
### Accessing the store
Accessing the store is typically done in the script area of components. Vuex contains a bunch of helper methods you can import and use to map the vuex methods and values to our components. You can see this in our Projects.vue panel component.

```javascript
import Panel from "primevue/panel";
import EditableRecord from "./EditableRecord"
import CreateRecord from './CreateRecord';

import { mapState, mapMutations, mapActions, mapGetters } from 'vuex'
export default {
    components: {
        Panel,
        EditableRecord,
        CreateRecord
    },
    computed: {
        ...mapState('projects', [
            'newProjectName',
            'projects',
            'activeProjectID'
        ]),
        ...mapGetters('authentication', [
            'isLoggedIn'
        ])
    },
    methods: {
        ...mapMutations('projects', [
            'setNewProjectName',
            'updateRecordTitle'
        ]),
        ...mapActions('projects', [
            'createProject',
            'getProjects',
            'deleteProject',
            'saveProjectName',
            'selectProject'
        ])
    },
    mounted() {
        if(this.isLoggedIn) {
            this.getProjects()
        }
    }
}
```
## Differences & challenges I encountered
I encountered a few differences between the tutorial and my own code. Mostly because of Vue 3.0 is a bit different and I'm using a different component library.

### Text Fields
In the tutorial, he got away with being able to use the @input event mapping with vuetify text fields
```html
<v-text-field
  autofocus
  v-if="isEditMode"
  :value="title"
  @keyup.enter="$emit('onSave')"
  @input="$emit('onInput', $event)"
></v-text-field>
```
However this didn't work for whatever reason with the PrimeVue InputText component. I thought it would work regardless because I was under the impression that events were passed to the root element of a component if you don't define them, but perhaps I am wrong or we were overwriting some important event. 

So I checked the source from their github to see what might be happening
```html
<template>
    <input :class="['p-inputtext p-component', {'p-filled': filled}]" :value="modelValue" @input="onInput" />
</template>

<script>
export default {
    emits: ['update:modelValue'],
    props: {
        modelValue: null
    },
    methods: {
        onInput(event) {
            this.$emit('update:modelValue', event.target.value);
        }
    },
    computed: {
        filled() {
            return (this.modelValue != null && this.modelValue.toString().length > 0)
        }
    }
}
</script>
```
It looks like they don't propagate the @input event and instead they implemented it with the expectation we'd be using the two way data binding provided by v-model. I don't think that's a good fit for use with vuex because we're not allowed to change the state outside of mutations....

So instead we hooked into the event they do propagate.
```html
<InputText
    :placeholder="placeholder" 
    @update:model-value="$emit('input', $event)"
    :value="value"
    @keyup.enter="$emit('create')"
    class="p-inputtext-lg p-col" 
    :disabled="disabled"
/>
```

### Panel
In the tutorial video they created their own panel component since Vuetify didn't have one out of the box. However, PrimeVue does and I'm not a fan of extra work lol. So we just skipped this.

### Infinite recursion bug?
In the tutorial they used a component called Projects inside of their view called Projects. When I followed diligently I encountered an infinite loop breaking the page. 

Code that breaks the universe (Inside of src/views/Projects.vue)
```html
<template>
    <div class="p-grid p-mt-4">
        <Projects class="p-col-4"></Projects>
        <tasks-panel class="p-col-8"></tasks-panel>
    </div>
</template>

<script>
import Projects from "../components/Projects.vue"
import TasksPanel from "../components/Tasks.vue"

import { mapGetters } from 'vuex'

export default {
    components: {
        Projects,
        TasksPanel
    },
    computed: {
        ...mapGetters('authentication', [
            'isLoggedIn'
        ])
    },
    mounted() {
        if(!this.isLoggedIn) {
            this.$router.push('/login')
        }
    }
}
</script>
```

Our routes.js
```javascript
import { createRouter, createWebHashHistory } from 'vue-router'
import Projects from '../views/Projects.vue'

const routes = [
  {
    path: '/',
    name: 'Projects',
    component: Projects
  },
  {
    path: '/about',
    name: 'About',
    // route level code-splitting
    // this generates a separate chunk (about.[hash].js) for this route
    // which is lazy-loaded when the route is visited.
    component: () => import(/* webpackChunkName: "about" */ '../views/About.vue')
  },
  {
    path: '/register',
    name: 'Register',
    component: () => import ('../views/Register.vue')
  },
  {
    path: '/login',
    name: 'Login',
    component: () => import ('../views/Login.vue')
  }
]

const router = createRouter({
  history: createWebHashHistory(),
  routes
})

export default router
```

My assumption is that either the Projects object we create in the import statement in routes.js is taking priority over the Projects object we import in the component or there is some sort of internal Vue object defined under the same name.

In any case, I fixed it by just importing it in the component as ProjectPanel instead of Projects.

### Vue.set()
Vue3 doesn't support Ie11, so we don't need to use Vue.set since modern browsers can detect changes via proxies and we don't need to explicitly tell vue to overwrite the getters and setters.

This doesn't impact things too much tutorial-wise, but we didn't need to use Vue.set on the tasks to set completed and track it. 

# Gluing things together
## Building the frontend
In order to compress and package the frontend, we need to run the build script in our client directory. This will compile our frontend and stick the packaged version into the client/dist folder.

```
npm run build
```

## Serving via adonis
In order to serve this with adonis, we just need to copy all the files from the client/dist folder to our server/public folder. 

After that we need to tell adonis we want to serve the static folder public by enabling the static file middleware. This is done by uncommenting this line in the server/start/kernel.js file.
```javascript
const serverMiddleware = [
  //'Adonis/Middleware/Static',
  'Adonis/Middleware/Cors'
]
```

Now we just edit our .env file to change our host to 0.0.0.0 so we allow all connections, our port to match whatever port we want to serve on, and our NODE_ENV to production. 
```
HOST=0.0.0.0
PORT=8080
NODE_ENV=production
APP_NAME=AdonisJs
APP_URL=http://${HOST}:${PORT}
CACHE_VIEWS=false
APP_KEY=dHImA9BVBgQ5MXeyw2RZqoQdzNFMyZcP
DB_CONNECTION=sqlite
DB_HOST=127.0.0.1
DB_PORT=3306
DB_USER=root
DB_PASSWORD=
DB_DATABASE=adonis
HASH_DRIVER=bcrypt
```

Now we just can call node on server.js in the server folder and our project is live.
```
node server.js
```

# Closing
Even though this was written as my notes and thoughts during the project, I hope someone finds it useful someday.

Thanks for reading.
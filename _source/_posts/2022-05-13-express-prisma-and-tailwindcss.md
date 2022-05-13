---
layout: blog_post
title: ""
author:
by: advocate|contractor
communities: [devops,security,mobile,.net,java,javascript,go,php,python,ruby]
description: ""
tags: []
tweets:
- ""
- ""
- ""
image:
type: awareness|conversion
---
## Building an Express application using Prisma 

[Express](https://expressjs.com/) is one of the most popular web frameworks available today.  Understanding the tools available when building any web application is critical to being a good developer.

[Prisma](https://www.prisma.io/) is an ORM for Node.js using TypeScript.  It integrates with many of the most popular databases today including MySQL, SQL Server, SQLite, and MongoDB.  CockroachDB is available in preview mode.  Prisma emphasizes a human readable schema with a type-safe database client.  Prisma also includes other features such as migrations, seed data, and a virtual database browser.  

In this project, you will use Prisma to connect to a database server.  You will build a schema to model a workout tracker application.  Then you will create some seed data to begin working with and use Prisma to migrate and seed your database.  You will then create the workout web application using [Pug](https://pugjs.org/api/getting-started.html) and [tailwindcss](https://tailwindcss.com/) on your front end to compliment your express backend.  

### Prequresites

To continue you will need:

- An IDE for javascript.  I will use [Visual Studio Code](https://code.visualstudio.com/) but you can use Atom, Webstorm, or any other IDE you wish
- [Node.js](https://nodejs.org/en/)
- A database, one of PostgreSQL, MySQL, SQLite, SQL Server, or MongoDB.  I will be using SQL Server
- An [Okta Developer Account](https://developer.okta.com/) (free forever, to handle your OAuth needs)
- [Okta CLI](https://cli.okta.com)

If you wish to look at the app you can [view it on github](https://github.com/nickolasfisher/Okta_Primsa).

## Create your Okta application

Create a new directory for your application.  

{% include setup/cli.md type="web" loginRedirectUri="http://localhost:3000/authorization-code/callback" %}

## Create your Express Application

Next, you will use the `express-generator` application generator tool to quickly scaffold your application.  Run the following command.

```console
npx express-generator --view=pug
```

Next, you will need to install your packages.  

```console
npm i @prisma/client@3.13.0
npm i dotenv@16.0.0
npm i passport@0.5.2
npm i passport-openidconnect@0.1.1
npm i express-session@1.17
npm i -D tailwindcss@3.0.24
npm i -D prisma@3.13.0
```

> `@prisma/client` is used to access the database from your server code
> `dotenv` reads configuration settings from `.env` files like the one produced by the Okta CLI
> `passport` is a middleware for Node.js that is flexible enough to handle most authentication scenarios, including Okta.  `passport-openidconnect` is a module for passport that lets you authenticate with OpenID Connect
> `express-session` is required for passport.
> `tailwindcss` is the CSS framework you will use.  If you've never used tailwindcss before you may wonder why it's a development dependency.  This is because tailwindcss will dynamically build your CSS files from your views and configuration.  More on that later
> `prisma` is the ORM you are using.  This is a dev dependency because the prisma library handles the migrations, seed data, etc, while the `@prisma/client` package is used in your application at runtime.

At this point, you should initialize tailwindcss and prisma even though you won't configure to use either quite yet.

Start with prisma.  Run the following command.

```console
npx prisma init
```

This command will add a new folder named `prisma` to your application. This also adds a file called `.env` to your project with some default configuration. Replace the contents of that file with the code below.

```bash
# Environment variables declared in this file are automatically made available to Prisma.
# See the documentation for more detail: https://pris.ly/d/prisma-schema#accessing-environment-variables-from-the-schema

# Prisma supports the native connection string format for PostgreSQL, MySQL, SQLite, SQL Server, MongoDB, and CockroachDB (Preview).
# See the documentation for all the connection string options: https://pris.ly/d/connection-strings

DATABASE_URL={yourServersConnectionString}
SEED_USER_NAME={yourOktaUserName}
```

In this example I used Microsoft's SQL Server because that's what I had installed at the time of writing this application.  But one of the cool things about Prisma and most ORMs is that they are built to support many databases.  The boilerplate code includes comments about how to configure your application for other database servers.  Feel free to use what you feel most comfortable with and use the appropriate connection string.

The `SEED_USER_NAME` setting is used in your seed data.  You want to make sure that it's the same as the Okta username you sign in to.  This will allow the application to associate your logged-in user with the data that you will seed into your database.  

 Next, update your `package.json` file with the following code.

```JSON
{
  "name": "workout-app",
  "version": "0.0.0",
  "private": true,
  "prisma": {
    "seed": "node prisma/seed.js"
  },
  "scripts": {
    "start": "node ./bin/www"
  },
  "dependencies": {
    "@prisma/client": "^3.13.0",
    "cookie-parser": "~1.4.4",
    "debug": "~2.6.9",
    "dotenv": "^16.0.0",
    "express": "~4.16.1",
    "http-errors": "~1.6.3",
    "morgan": "~1.9.1",
    "passport": "^0.5.2",
    "passport-openidconnect": "^0.1.1",
    "pug": "2.0.0-beta11"
  },
  "devDependencies": {
    "prisma": "^3.13.0",
    "tailwindcss": "^3.0.24"
  }
}
```

This will add the `prisma.seed` command you will need later.  

Now you can initialize tailwindcss.  

```console
npx tailwindcss init
```

This will add a new file called `tailwind.config.js` to the root of your application.  Replace the contents of that file with the code below.

```javascript
module.exports = {
  content: ["./views/**/*.{pug}"],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

This will tell tailwindcss where to find the classes that you used in your application.  In this case, you want tailwind to look in `.pug` files located in your `views` directory.  Tailwindcss is extremely extensible as you can see by the `theme` and `plugins` settings in the configuration object.  A deep dive into this is out of the scope of this article but I encourage you to look into tailwindcss's site for more information.  

### Use Prisma to create your database

The next task is to create the database for your application.  This will involve creating the schema, writing some seed data, creating your migration, and applying the migration which will also seed your data.

The `prisma init` task from above should have added a folder called `schema.prisma` to your `prisma` directory.  Replace the code there with the code below.

```javascript
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "{yourDatabaseProvider}"
  url      = env("DATABASE_URL")
}

model WorkoutLog {
  id        Int      @id @default(autoincrement())
  userId    Int
  exercise  String
  amount    Decimal
  units     String
  date      DateTime
  minutes   Int
  calories  Int
  user      User     @relation(fields: [userId], references: [id])
}

model User {
  id          Int          @id @default(autoincrement())
  username    String       @unique @db.VarChar(255)
  workoutLogs WorkoutLog[]
}
```

Make sure you replace `{yourDatabaseProvider}` with [the provider you are using](https://www.prisma.io/docs/reference/api-reference/prisma-schema-reference#datasource).  As you can see this file defines the `User` and `WorkoutLog` objects that you will store in the database.  The `WorkoutLog` has some basic information about what the user did on a specific day and then associates that record with the `User` object.  You can define keys, constraints, and indices from this file as well.  Note the `user` property on the `WorkoutLog` has a relation defined by the `userId` to the `id` of the `User` table.  

Add a file to the `prisma` directory named `seed.js` and add the following code.

```javascript
const { PrismaClient } = require("@prisma/client");
const prisma = new PrismaClient();

require("dotenv").config();

async function main() {
  const user = await prisma.user.upsert({
    where: { username: process.env.SEED_USER_NAME },
    update: {},
    create: {
      username: process.env.SEED_USER_NAME,
      workoutLogs: {
        create: [
          {
            exercise: "Running",
            amount: 1,
            units: "Miles",
            date: new Date(2022, 1, 1),
            minutes: 8,
            calories: 100
          },
          {
            exercise: "Running",
            amount: 1.2,
            units: "Miles",
            date: new Date(2022, 1, 3),
            minutes: 10,
            calories: 120
          },
          {
            exercise: "Running",
            amount: 1.5,
            units: "Miles",
            date: new Date(2022, 1, 5),
            minutes: 12,
            calories: 150
          },
          {
            exercise: "Heavy Bag",
            amount: 4,
            units: "Rounds",
            date: new Date(2022, 1, 1),
            minutes: 15,
            calories: 100
          },
          {
            exercise: "Heavy Bag",
            amount: 6,
            units: "Rounds",
            date: new Date(2022, 1, 3),
            minutes: 22,
            calories: 150
          },
          {
            exercise: "Heavy Bag",
            amount: 4,
            units: "Rounds",
            date: new Date(2022, 1, 5),
            minutes: 15,
            calories: 100
          },
          {
            exercise: "Situps",
            amount: 50,
            units: "Reps",
            date: new Date(2022, 1, 2),
            minutes: 5,
            calories: 50
          },
          {
            exercise: "Pushups",
            amount: 100,
            units: "Reps",
            date: new Date(2022, 1, 2),
            minutes: 10,
            calories: 100
          },
          {
            exercise: "Situps",
            amount: 50,
            units: "Reps",
            date: new Date(2022, 1, 4),
            minutes: 5,
            calories: 50
          },
          {
            exercise: "Pushups",
            amount: 100,
            units: "Reps",
            date: new Date(2022, 1, 4),
            minutes: 10,
            calories: 100
          },
        ],
      },
    },
  });
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

Prisma knows to use the `seed.js` file from the command you added to `package.json`.  This file will insert the data when you run that command.  

You can now add and apply the migration using the Prisma CLI.

```console
npx prisma migrate dev --name init
```

The CLI has several ways to add and apply migrations.  I suggest you understand the best way to manage migrations for your environment.  The method above is the easiest and fastest way to get your database ready.  As part of the migration process, this command will look for the `seed` command from your `package.json` and run that as well.  Once this process is complete your database should be ready with a database full of seed data ready to work.

### Complete the development of your express application

Now that your database is set up you can turn your attention to the core of your application.

First, add a new file in the root of your application called `ensureLoggedIn.js` and add the following code to it.

```javascript
function ensureLoggedIn(req, res, next) {
    if (req.isAuthenticated()) {
      return next();
    }
    res.redirect('/login')
  }

  module.exports = ensureLoggedIn;
```

This little piece of middleware will just make sure the user is authenticated.  If they aren't then you will redirect them to the `login` route which you will configure to use Okta.  Otherwise, you will allow the user to the next instruction.  

Next, you can update your routes.  First, remove `users.js` as you won't be using that.  Next, replace the code in `index.js` with the following.

```javascript
var express = require('express');
var router = express.Router();

/* GET home page. */
router.get('/', function(req, res, next) {
  res.render('index', { title: 'The Workout Tracker - Home', isAuthenticated: req.isAuthenticated() });
});

module.exports = router;
```

Here you will pass a property for `isAuthenticated` so your layout page can properly display a login or logout button.  

Next, add a file in the `routes` directory for `dashboard.js` with the following code.

```javascript
var express = require("express");
var router = express.Router();

const { PrismaClient } = require("@prisma/client");
const prisma = new PrismaClient();

const ensureLoggedIn = require("../ensureLoggedIn");

/* GET home page. */
router.get("/", ensureLoggedIn, async function (req, res) {
  const username = req.user.username;

  const dbUser = await prisma.user.findUnique({
    where: {
      username: username,
    },
    include: {
      workoutLogs: true,
    },
  });

  res.render("dashboard", {
    title: "The Workout Tracker - Dashboard",
    isAuthenticated: req.isAuthenticated(),
    user: dbUser
  });
});

module.exports = router;
```

There's a lot of magic in this route.  First, you are using the `ensureLoggedIn` middleware to protect this route.  Then you can extract the username from the request and lookup the user in the database using the prisma client.  Since the user has a property for its workout logs, you can simply pass the user as the model and parse the logs on the pug view you will create.  This will give the view access to the user's name as well.  Of course, you could query the `workoutLogs` table directly and include features such as pagination and searching if that better fit your workflow.  

Next, replace the code in `app.js` with the following.

```javascript
var createError = require('http-errors');
var express = require('express');
var path = require('path');
var cookieParser = require('cookie-parser');
var logger = require('morgan');

var indexRouter = require('./routes/index');
var dashboardRouter = require('./routes/dashboard');

const session = require('express-session');
const passport = require('passport');
const { Strategy } = require('passport-openidconnect');

var app = express();

const { PrismaClient } = require('@prisma/client')

const prisma = new PrismaClient();

require('dotenv').config({path:'./.okta.env'});

// view engine setup
app.set('views', path.join(__dirname, 'views'));
app.set('view engine', 'pug');

app.use(session({
  secret: 'CanYouLookTheOtherWay',
  resave: false,
  saveUninitialized: true
}));

app.use(passport.initialize());
app.use(passport.session());


const {OKTA_OAUTH2_CLIENT_ID, OKTA_OAUTH2_CLIENT_SECRET, OKTA_OAUTH2_ISSUER} = process.env;

// set up passport
passport.use('oidc', new Strategy({
  issuer: OKTA_OAUTH2_ISSUER,
  authorizationURL: `${OKTA_OAUTH2_ISSUER}/v1/authorize`,
  tokenURL: `${OKTA_OAUTH2_ISSUER}/v1/token`,
  userInfoURL: `${OKTA_OAUTH2_ISSUER}/v1/userinfo`,
  clientID: OKTA_OAUTH2_CLIENT_ID,
  clientSecret: OKTA_OAUTH2_CLIENT_SECRET,
  callbackURL: 'http://localhost:3000/authorization-code/callback',
  scope: 'openid profile'
}, (issuer, profile, done) => {
  return done(null, profile);
}));

passport.serializeUser((user, next) => {
  next(null, user);
});

passport.deserializeUser((obj, next) => {
  next(null, obj);
});

app.use(logger('dev'));
app.use(express.json());
app.use(express.urlencoded({ extended: false }));
app.use(cookieParser());
app.use(express.static(path.join(__dirname, 'public')));

app.use('/login', passport.authenticate('oidc'));

app.use('/authorization-code/callback',
  passport.authenticate('oidc', { failureRedirect: '/error' }),
  (req, res, next) => {
    res.redirect('/dashboard');
  }
);

app.post('/logout', (req, res) => {
  req.logout();
  req.session.destroy();
  res.redirect('/');
});

app.use('/', indexRouter);
app.use('/dashboard', dashboardRouter);

// catch 404 and forward to error handler
app.use(function(req, res, next) {
  next(createError(404));
});

// error handler
app.use(function(err, req, res, next) {
  // set locals, only providing error in development
  res.locals.message = err.message;
  res.locals.error = req.app.get('env') === 'development' ? err : {};

  // render the error page
  res.status(err.status || 500);
  res.render('error');
});

module.exports = app;
```

You needed to do a little work to update your routes to properly use the `dashboard` routes as well as any login or logout routes that Okta will use.  You also set up passport to use Okta using the values the Okta CLI produced in the `.okta.env` file.  

### Add your views to bring your project to life

By default, express-generator will add a few views for you.  You will need to edit a couple and add a new one of your own.  Start by opening the `layout.pug` file in the `views` directory and replacing the code there with the following.

```pug
doctype html
html
  head
    title= title
    link(rel='stylesheet', href='/stylesheets/output.css')
  body
    nav.flex.items-center.justify-between.flex-wrap.bg-teal-500.p-6.flex.items-center.flex-shrink-0.text-white.mr-6
      span.font-semibold.text-xl.tracking-tight.pr-2 The Workout Tracker
      .w-full.block.flex-grow(class='lg:flex lg:items-center lg:w-auto')
        .text-sm(class='lg:flex-grow')
        div
          if(isAuthenticated)
            form(action="logout" method="POST") 
              button(type="submit").inline-block.text-sm.px-4.py-2.leading-none.border.rounded.text-white.border-white.mt-4(href='#' class='hover:border-transparent hover:text-teal-500 hover:bg-white lg:mt-0') Logout
          else 
            a.inline-block.text-sm.px-4.py-2.leading-none.border.rounded.text-white.border-white.mt-4(href='Login' class='hover:border-transparent hover:text-teal-500 hover:bg-white lg:mt-0') Login
    
    block content
```

You'll notice a couple of things about this file.  First, the classes are very specific and plentiful.  This is typical of tailwindcss.  Rather than defining how a specific element looks, tailwindcss strives to define how a particular style looks, then you can add and remove styles from elements to fit the desired look.  

The other thing you'll see is the Login/Logout button that will change based on the `isAuthenticated` property.  

Next, your can update `index.pug` with the following.

```pug
extends layout

block content
  .py-12.bg-white
    .max-w-7xl.mx-auto.px-4(class='sm:px-6 lg:px-8')
      div.c(class='lg:text-center')
        p.mt-2.text-3xl.leading-8.font-extrabold.tracking-tight.text-gray-900(class='sm:text-4xl') The Workout Tracker
        p.mt-4.max-w-2xl.text-xl.text-gray-500(class='lg:mx-auto')
          | A small demo built with 
          a(href="https://expressjs.com/") Express on 
          a(href="https://nodejs.org/en/") Node.JS 
          | Secured with 
          a(href="https://developer.okta.com/signup") Okta.
          br          
          | This app uses 
          a(href="https://tailwindcss.com/") tailwindcss 
          | for its CSS framework and 
          a(href="https://www.prisma.io/") Prisma.JS 
          | for its ORM.  
```

There's not much to talk about here.  Just a splash page with some more tailwindcss and some links to the technologies used.

Finally, add a file for `dashboard.pug` with the following code.

```pug
extends layout

block content 
    .flex.flex-wrap
        .w-full.p-6(class='md:w-1/2 xl:w-1/3')
            .bg-gradient-to-b.from-green-200.to-green-100.border-b-4.border-green-600.rounded-lg.shadow-xl.p-5
                .flex.flex-row.items-center
                    .flex-shrink.pr-4
                        .rounded-full.p-5.bg-green-600
                            i.fa.fa-wallet.fa-2x.fa-inverse
                    .flex-1.text-right(class='md:text-center')
                        h2.font-bold.uppercase.text-gray-600 Total Minutes Worked
                        p.font-bold.text-3xl
                            span= user.workoutLogs.reduce(function(total, item) { return total + item.minutes }, 0) 
                            span.text-green-500
                                i.fas.fa-caret-up
        .w-full.p-6(class='md:w-1/2 xl:w-1/3')
            .bg-gradient-to-b.from-pink-200.to-pink-100.border-b-4.border-pink-500.rounded-lg.shadow-xl.p-5
                .flex.flex-row.items-center
                    .flex-shrink.pr-4
                        .rounded-full.p-5.bg-pink-600
                            i.fas.fa-users.fa-2x.fa-inverse
                    .flex-1.text-right(class='md:text-center')
                        h2.font-bold.uppercase.text-gray-600 Total Calories Burned
                        p.font-bold.text-3xl
                            span= user.workoutLogs.reduce(function(total, item) { return total + item.calories }, 0)  
                            span.text-pink-500
                                i.fas.fa-exchange-alt
        .w-full.p-6(class='md:w-1/2 xl:w-1/3')
            .bg-gradient-to-b.from-yellow-200.to-yellow-100.border-b-4.border-yellow-600.rounded-lg.shadow-xl.p-5
                .flex.flex-row.items-center
                    .flex-shrink.pr-4
                        .rounded-full.p-5.bg-yellow-600
                            i.fas.fa-user-plus.fa-2x.fa-inverse
                    .flex-1.text-right(class='md:text-center')
                        h2.font-bold.uppercase.text-gray-600 Total Days Worked
                        p.font-bold.text-3xl
                            span= user.workoutLogs.map(r => r.date).filter((date, i, self) => self.findIndex(d => d.getTime() === date.getTime()) === i).length
                            span.text-yellow-600
                                i.fas.fa-caret-up
    .w-full.p-6
        .bg-white.border-transparent.rounded-lg.shadow-xl
            .bg-gradient-to-b.from-gray-300.to-gray-100.uppercase.text-gray-800.border-b-2.border-gray-300.rounded-tl-lg.rounded-tr-lg.p-2
                h2.font-bold.uppercase.text-gray-600 Workout Log
            .p-5
                table.w-full.p-5.text-gray-700
                    thead
                        tr
                            th.text-left.text-blue-900 Date
                            th.text-left.text-blue-900 Execise
                            th.text-left.text-blue-900 Amount
                            th.text-left.text-blue-900 Time
                            th.text-left.text-blue-900 Calories Burned
                    tbody
                        each log in user.workoutLogs.sort((a,b) => a.date - b.date)
                            tr
                                td=log.date.toLocaleDateString()
                                td=log.exercise
                                td=log.amount + ' ' + log.units 
                                td=log.minutes + ' Minutes'
                                td=log.calories                        
```

As you saw before this page is protected, therefore it's required that you have a user access this page.  The application will pull the user details and workout logs and reform them into a summary at the top of the page with a couple of tables of the logs based on the exercises the user has performed.  

### Create the CSS file using tailwindcss

The last part of this process is to create the actual CSS file that your application will use.  In your `public/stylesheets` directory, you'll see a file called `style.css`.  replace the code in that file with the code below.

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

body {
  padding: 50px;
  font: 14px "Lucida Grande", Helvetica, Arial, sans-serif;
}

a {
  color: #00B7FF;
}
```

This is largely the same however you are importing the directives for each of Tailwind's layers.

Next, you can compile the classes you used in your pug files along with the style sheet you just modified to create a new CSS file.  Run the following command.

```bash
npx tailwindcss -i ./public/stylesheets/style.css -o ./public/stylesheets/output.css --watch
```

Now you can open the `output.css` file and see the full tailwindcss code.  The `--watch` parameter here is useful for local development.  This will rebuild the `output.css` file each time you make a change to one of your layouts.  If you do not want to make any changes you can disregard them.

## Test your application

You're not ready to start your application.  Run the following command.

```bash
npm run start
```

Once your application is started you should be able to navigate to *http://localhost:3000/* and see your home page.

{% img blog/express-prisma-and-tailwindcss/HomePage.PNG alt:"Home page" width:"800" %}{: .center-image }

Use the login button at the top of the screen and navigate out to the Okta login page.  Once you've logged in you will be redirected back to your website where you will be able to see your dashboard page.

{% img blog/express-prisma-and-tailwindcss/DashboardPage.PNG alt:"Dashboard page" width:"800" %}{: .center-image }

## What you learned in this article

In this article, you learned how to use Prisma to define a schema for your database.  You then learned how to create a migration from that schema and how to seed your database.  You saw that Prisma can be used with the most popular database options including SQL Server, MySql, MongoDB, and more.  

You also learned how to build a CSS file with tailwindcss.  You learned the simple tailwind configuration and how the tailwind CLI uses your code to build its CSS file.

Finally, you learned how to secure your express application using passport.

## Further Reading

[Build and Deploy a Node.js App to Heroku](https://developer.okta.com/blog/2022/02/28/build-deploy-node-app-heroku)
[Node.js Login with Express and OIDC](https://developer.okta.com/blog/2020/06/16/nodejs-login)
[Build a Node.js API with TypeScript](https://developer.okta.com/blog/2019/05/07/nodejs-typescript-api)
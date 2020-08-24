# How the Node Reverse Authentication App Works

This user authentication app with node and express helps users create usernames with an email and password. Allowing them to register and then signin with those credentials. Here is how the code base works. 

Let's start with the file structure. 
<pre>
* config/
    * middleware/
        isAuthenticated.js
    * config.json
    * passport.js

* models/
    * index.js
    * user.js

* public/
    * js/
        * login.js
        * members.js
        * signup.js
    * stylesheets/
        * style.css
    * login.html
    * members.html
    * signup.html 

* routes/
    * api-routes.js
    * html-routes.js

* package.json
* server.js
</pre>

As you can see server.js is our server's entry point, but we will go over the package.json and the software depenencies and then go into the rest of the code base. 

# package.json

Let's start with the package.json Here are some of the relevant pieces of code to consider. 

<pre> {
  "name": "1-Passport-Example",
  "version": "1.0.0",
  "description": "",
  "main": "server.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "start": "node server.js",
    "watch": "nodemon server.js"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "bcryptjs": "2.4.3",
    "express": "^4.17.0",
    "express-session": "^1.16.1",
    "mysql2": "^1.6.5",
    "passport": "^0.4.0",
    "passport-local": "^1.0.0",
    "sequelize": "^5.8.6"
  }
}</pre>

## main 
This is the entry for our server. So when we run node . in this directory, this is the file that will be run. 

## dependencies

### bcryptjs
bcryptjs is a software dependency that will help us to safely store our users passwords through encryption. It comes fully loaded with the ability to create salts and hashes - that allow us to store the same password in our database many times with all unique hashes. '

### express-session
express-session is a simple node middleware - that will allow us to keep session data that will help us keep track of if the user is logged in or not. 

### passport
Passport is an express-compatible authentication middleware for Node.js that allows us to authenticate requests through the use of various strategies. 

### passport-local
Passport-local is one of those passport strategies - thats purpose is to allow us to authenticate usernames and passwords.


# Server.js

Here's our entry file for the application (this is the main from package.json)

<pre>// Requiring necessary npm packages
var express = require("express");
var session = require("express-session");
// Requiring passport as we've configured it
var passport = require("./config/passport");

// Setting up port and requiring models for syncing
var PORT = process.env.PORT || 8080;
var db = require("./models");

// Creating express app and configuring middleware needed for authentication
var app = express();
app.use(express.urlencoded({ extended: true }));
app.use(express.json());
app.use(express.static("public"));
// We need to use sessions to keep track of our user's login status
app.use(session({ secret: "keyboard cat", resave: true, saveUninitialized: true }));
app.use(passport.initialize());
app.use(passport.session());

// Requiring our routes
require("./routes/html-routes.js")(app);
require("./routes/api-routes.js")(app);

// Syncing our database and logging a message to the user upon success
db.sequelize.sync().then(function() {
  app.listen(PORT, function() {
    console.log("==> 🌎  Listening on port %s. Visit http://localhost:%s/ in your browser.", PORT, PORT);
  });
});
</pre>

Lines 2 and 3; brings in the required dependencies that we saw in the package.json - express, express-sessions and passport. 

Line 8 sets the PORT and Line 9 pulls the database from the models folder within the directory. 

Lines 12 - 15 are setting up the express middleware and the express app, telling the app where to find the static files, which are inside the ("public") folder in the directory. 

lines: 17-19 create a session to keep track of the login status. That likely is resislent to refreshing the page. 

Lines 22, 23 are requiring both the API and html routes. 

Lines 26-28 are syncing the sequelize database and telling our server to listen at port 8080, and consoling logging the output in successful. 

Next we will go into the passport.js file inside the config folder. 

# config/passport.js

This file requires passport and passport local and runs the user authentication logic and error handling. 

<pre>var passport = require("passport");
var LocalStrategy = require("passport-local").Strategy;

var db = require("../models");

// Telling passport we want to use a Local Strategy. In other words, we want login with a username/email and password
passport.use(new LocalStrategy(
  // Our user will sign in using an email, rather than a "username"
  {
    usernameField: "email"
  },
  function(email, password, done) {
    // When a user tries to sign in this code runs
    db.User.findOne({
      where: {
        email: email
      }
    }).then(function(dbUser) {
      // If there's no user with the given email
      if (!dbUser) {
        return done(null, false, {
          message: "Incorrect email."
        });
      }
      // If there is a user with the given email, but the password the user gives us is incorrect
      else if (!dbUser.validPassword(password)) {
        return done(null, false, {
          message: "Incorrect password."
        });
      }
      // If none of the above, return the user
      return done(null, dbUser);
    });
  }
));

// In order to help keep authentication state across HTTP requests,
// Sequelize needs to serialize and deserialize the user
// Just consider this part boilerplate needed to make it all work
passport.serializeUser(function(user, cb) {
  cb(null, user);
});

passport.deserializeUser(function(obj, cb) {
  cb(null, obj);
});

// Exporting our configured passport
module.exports = passport;
</pre>

Lines 1 and 2 require passport and passport-local as a dependencies.

Line 4 requires the db from models for storing and authenticating usernames and passwords. 

On line 7, we are telling passport we want to use passport-local that our username will be signing in with a email rather than username. 

And then to find an email - when user signs in with one. 

Then if there is no email registered to send back an error message - if there is one and the wrong password is submitting give another error message "incorrect message", else if - return done, and log in the user. 

Lines 40, 45 are boilerplate code for sequelize and passport to work within the context of this applicaiton, because of the need to serialize and deserialize the user. 

Next we will look into the models folder. 

# models/ 

## index.js

This file is simple as it sets up the db and sequelize connection and links to the credentials from our config/config.json file. 

<pre>'use strict';

var fs        = require('fs');
var path      = require('path');
var Sequelize = require('sequelize');
var basename  = path.basename(module.filename);
var env       = process.env.NODE_ENV || 'development';
var config    = require(__dirname + '/../config/config.json')[env];
var db        = {};

if (config.use_env_variable) {
  var sequelize = new Sequelize(process.env[config.use_env_variable]);
} else {
  var sequelize = new Sequelize(config.database, config.username, config.password, config);
}

fs
  .readdirSync(__dirname)
  .filter(function(file) {
    return (file.indexOf('.') !== 0) && (file !== basename) && (file.slice(-3) === '.js');
  })
  .forEach(function(file) {
    var model = sequelize['import'](path.join(__dirname, file));
    db[model.name] = model;
  });

Object.keys(db).forEach(function(modelName) {
  if (db[modelName].associate) {
    db[modelName].associate(db);
  }
});

db.sequelize = sequelize;
db.Sequelize = Sequelize;

module.exports = db;
</pre>

## user.js
This file creates our user model and sets up the bcrypt encryption + salt generation that will ultimately make up our stored password, and also allows us to check to see whether the password the user is using to log in - is the one that is encrypted in our database. 

<pre>// Requiring bcrypt for password hashing. Using the bcryptjs version as the regular bcrypt module sometimes causes errors on Windows machines
var bcrypt = require("bcryptjs");
// Creating our User model
module.exports = function(sequelize, DataTypes) {
  var User = sequelize.define("User", {
    // The email cannot be null, and must be a proper email before creation
    email: {
      type: DataTypes.STRING,
      allowNull: false,
      unique: true,
      validate: {
        isEmail: true
      }
    },
    // The password cannot be null
    password: {
      type: DataTypes.STRING,
      allowNull: false
    }
  });
  // Creating a custom method for our User model. This will check if an unhashed password entered by the user can be compared to the hashed password stored in our database
  User.prototype.validPassword = function(password) {
    return bcrypt.compareSync(password, this.password);
  };
  // Hooks are automatic methods that run during various phases of the User Model lifecycle
  // In this case, before a User is created, we will automatically hash their password
  User.addHook("beforeCreate", function(user) {
    user.password = bcrypt.hashSync(user.password, bcrypt.genSaltSync(10), null);
  });
  return User;
};
</pre>

# public/
Inside of our public folder, we have all of the static files - whether that is the html, css or javascript. All of which is fairly simple. As the set up the user interface, experience, and styling. And also help to make the buttons on the page functional and pull the email input and password input through the proper processes. 

## /js/

## login.js
This file correlates with the login.html page and is fairly straight-forward. 

<pre>$(document).ready(function() {
    // Getting references to our form and inputs
    var loginForm = $("form.login");
    var emailInput = $("input#email-input");
    var passwordInput = $("input#password-input");
  
    // When the form is submitted, we validate there's an email and password entered
    loginForm.on("submit", function(event) {
      event.preventDefault();
      var userData = {
        email: emailInput.val().trim(),
        password: passwordInput.val().trim()
      };
  
      if (!userData.email || !userData.password) {
        return;
      }
  
      // If we have an email and password we run the loginUser function and clear the form
      loginUser(userData.email, userData.password);
      emailInput.val("");
      passwordInput.val("");
    });
  
    // loginUser does a post to our "api/login" route and if successful, redirects us the the members page
    function loginUser(email, password) {
      $.post("/api/login", {
        email: email,
        password: password
      })
        .then(function() {
          window.location.replace("/members");
          // If there's an error, log the error
        })
        .catch(function(err) {
          console.log(err);
        });
    }
  });
  </pre>
## members.js

This file is simple. It checks to see if the user is logged in or not. 

<pre>$(document).ready(function() {
    // This file just does a GET request to figure out which user is logged in
    // and updates the HTML on the page
    $.get("/api/user_data").then(function(data) {
      $(".member-name").text(data.email);
    });
  });</pre>

## signup.js

This file pulls the signup form data and passes it through to the api-routes that will run through our other dependencies. 

<pre>$(document).ready(function() {
    // Getting references to our form and input
    var signUpForm = $("form.signup");
    var emailInput = $("input#email-input");
    var passwordInput = $("input#password-input");
  
    // When the signup button is clicked, we validate the email and password are not blank
    signUpForm.on("submit", function(event) {
      event.preventDefault();
      var userData = {
        email: emailInput.val().trim(),
        password: passwordInput.val().trim()
      };
  
      if (!userData.email || !userData.password) {
        return;
      }
      // If we have an email and password, run the signUpUser function
      signUpUser(userData.email, userData.password);
      emailInput.val("");
      passwordInput.val("");
    });
  
    // Does a post to the signup route. If successful, we are redirected to the members page
    // Otherwise we log any errors
    function signUpUser(email, password) {
      $.post("/api/signup", {
        email: email,
        password: password
      })
        .then(function(data) {
          window.location.replace("/members");
          // If there's an error, handle it by throwing up a bootstrap alert
        })
        .catch(handleLoginErr);
    }
  
    function handleLoginErr(err) {
      $("#alert .msg").text(err.responseJSON);
      $("#alert").fadeIn(500);
    }
  });</pre>

# routes/
## api-routes.js
This file is where we take the user input and then look to check the database for already registered users and then if the user is signing up where we will ultimately create a hash and store that user in our database. Which then we will send back our response(s).

<pre>// Requiring our models and passport as we've configured it
var db = require("../models");
var passport = require("../config/passport");

module.exports = function(app) {
  // Using the passport.authenticate middleware with our local strategy.
  // If the user has valid login credentials, send them to the members page.
  // Otherwise the user will be sent an error
  app.post("/api/login", passport.authenticate("local"), function(req, res) {
    res.json(req.user);
  });

  // Route for signing up a user. The user's password is automatically hashed and stored securely thanks to
  // how we configured our Sequelize User Model. If the user is created successfully, proceed to log the user in,
  // otherwise send back an error
  app.post("/api/signup", function(req, res) {
    db.User.create({
      email: req.body.email,
      password: req.body.password
    })
      .then(function() {
        res.redirect(307, "/api/login");
      })
      .catch(function(err) {
        res.status(401).json(err);
      });
  });

  // Route for logging user out
  app.get("/logout", function(req, res) {
    req.logout();
    res.redirect("/");
  });

  // Route for getting some data about our user to be used client side
  app.get("/api/user_data", function(req, res) {
    if (!req.user) {
      // The user is not logged in, send back an empty object
      res.json({});
    } else {
      // Otherwise send back the user's email and id
      // Sending back a password, even a hashed password, isn't a good idea
      res.json({
        email: req.user.email,
        id: req.user.id
      });
    }
  });
};</pre>

## html-routes.js
This file is as described. It creates our relative routes and then sets up our user auth middleware to direct users to the proper routes.

<pre>// Requiring path to so we can use relative routes to our HTML files
var path = require("path");

// Requiring our custom middleware for checking if a user is logged in
var isAuthenticated = require("../config/middleware/isAuthenticated");

module.exports = function(app) {

  app.get("/", function(req, res) {
    // If the user already has an account send them to the members page
    if (req.user) {
      res.redirect("/members");
    }
    res.sendFile(path.join(__dirname, "../public/signup.html"));
  });

  app.get("/login", function(req, res) {
    // If the user already has an account send them to the members page
    if (req.user) {
      res.redirect("/members");
    }
    res.sendFile(path.join(__dirname, "../public/login.html"));
  });

  // Here we've add our isAuthenticated middleware to this route.
  // If a user who is not logged in tries to access this route they will be redirected to the signup page
  app.get("/members", isAuthenticated, function(req, res) {
    res.sendFile(path.join(__dirname, "../public/members.html"));
  });

};</pre>
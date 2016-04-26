#Passport

[Passport](http://passportjs.org/docs) is an authentication middleware for Node.js. 

It’s extremely flexible and modular and can be dropped in to any Express-based web application, and can authenticate users via many different authentication mechanisms called “strategies”.

Strategies are packaged as individual modules so you can choose which strategies to employ, without creating unnecessary dependencies.

Let’s add Passport to our application.

      npm install passport --save

To use it, we need to require it in server.js (our main application file).

      var passport = require('passport');

Passport uses sessions. Session provides a way to identify a user across more than one page request or visit to a Web site and to store information about that user. 

We have to install and use  express-session:

      npm install express-session --save
      
And then

      var session = require('express-session');
      
      app.use(session({
         secret: 'secret',
         resave: true,
         saveUninitialized: false,
         cookie: { maxAge: 60000, secure: false }
      }));

The most widely used way for websites to authenticate users is via a username and password. Support for this mechanism is provided by the “passport-local” module.

Let’s add Passport-Local to our application:

      npm install passport-local --save

To use it, we need to require it in server.js, and get a reference to the module’s strategy object.

      var localStrategy = require('passport-local').Strategy;

We need to initialize passport:

      app.use(passport.initialize());
      app.use(passport.session());

Now, we have to tell passport which strategy to use inside our server.js file. 

      passport.use('local', new localStrategy({ passReqToCallback : true, usernameField: 'username' },
         function(req, username, password, done) {
         
            //We will come back to complete this
         
         }
      ));


The verify callback for local authentication accepts username and password arguments, which are submitted to the application via a login form. Inside this form we’ll authenticate users. However, we don’t have users set up correctly yet.

Then create the rest of the function for authenticating users. Serialize and deserialize allow user information to be stored and retrieved from session.

      passport.serializeUser(function(user, done) {
         done(null, user.id);
      });
      
      passport.deserializeUser(function(id, done) {
            console.log('called deserializeUser');
            pg.connect(connection, function (err, client) {
            
              var user = {};
              console.log('called deserializeUser - pg');
                var query = client.query("SELECT * FROM users WHERE id = $1", [id]);
            
                query.on('row', function (row) {
                  console.log('User row', row);
                  user = row;
                  done(null, user);
                });
            
                // After all data is returned, close connection and return results
                query.on('end', function () {
                    client.end();
                });
            
                // Handle Errors
                if (err) {
                    console.log(err);
                }
            });
      
      });
      
Now return to your localStrategy that we previously started (with comment).  
      
      passport.use('local', new localStrategy({
             passReqToCallback : true,
             usernameField: 'username'
         },
      function(req, username, password, done){
        console.log('called local');
          pg.connect(connectionString, function (err, client) {
            
            console.log('called local - pg');
      
            var user = {};
      
              var query = client.query("SELECT * FROM users WHERE username = $1", [username]);
      
              query.on('row', function (row) {
                console.log('User obj', row);
                console.log('Password', password)
                user = row;
                if(password == user.password){
                  console.log('match!')
                  done(null, user);
                } else {
                  done(null, false, { message: 'Incorrect username and password.' });
                }
                
              });
      
              // After all data is returned, close connection and return results
              query.on('end', function () {
                  client.end();
                  res.send(results);
              });
      
              // Handle Errors
              if (err) {
                  console.log(err);
              }
          });
      
      }));

Next create an index.html page and add a login form.

      <form action="/" method="post">
         <div>
             <label for="username">Username:</label>
             <input type="text" name="username" id="username"/>
         </div>
         <div>
             <label for="password">Password:</label>
             <input type="password" name="password" id="password"/>
         </div>
         <div>
             <input type="submit" value="Log In"/>
             <a href="/register">Register</a>
         </div>
      </form>

Create a route for the new index file. Passport.authenticate is specifying our ‘local’ strategy that we created, and specifies a failure and success redirect.

      var express = require('express');
      var router = express.Router();
      var passport = require('passport');
      var path = require('path');
      
      router.get("/", function(req,res,next){
         res.sendFile(path.resolve(__dirname, '../views/index.html'));
      });
      
      router.post('/',
         passport.authenticate('local', {
             successRedirect: '/users',
             failureRedirect: '/'
         })
      );
//Note: Users at this point should point to an HTML file. So an example would be, 
successRedirect: "/assets/views/users.html",
This would mean that you actually need to create this page to be served back to the client side.

      module.exports = router;

We also need a way for users to register. Create a register.html file with the following form in it:

      <form action="/register" method="post">
         <div>
             <label for="username">Username:</label>
             <input type="text" name="username" id="username"/>
         </div>
         <div>
             <label for="password">Password:</label>
             <input type="password" name="password" id="password"/>
         </div>
         <div>
             <input type="submit" value="Register"/>
         </div>
      </form>

Also create a register.js route file.

      var express = require('express');
      var router = express.Router();
      var passport = require('passport');
      var path = require('path');
      var Users = require('../models/user');
      
      router.get('/', function(req, res, next){
         res.sendFile(path.resolve(__dirname, '../views/register.html'));
      });
      
      router.post('/', function(req,res,next) {
        pg.connect(connectionString, function(err, client){
      
          var query = client.query('INSERT INTO users (username, password) VALUES ($1, $2)', [request.body.nameuser, request.body.wordpass]);
      
          query.on('error', function(err){
            console.log(err);
          })
      
          query.on('end', function(){
            response.sendStatus(200);
            client.end();
          })
      
        })
      });
      
      module.exports = router;

Add a register route to server.js:

      var register = require('./routes/register');
      
      app.use('/register', register);

Finally, let’s test user.isAuthenticated() in the users.js route

//Note: you will need to create this route, so in your routes file, create a users.js file, inside of it, you will need to include:

      var express = require('express');
      var router = express.Router();
      
      router.get('/', function(req, res, next) {
         res.send(req.isAuthenticated());
      });
      
      module.exports = router;
This alone will do nothing, but if you do a call from the client (a GET request) to the ‘/users’ route, and then console log out the response in the success callback function, you will see what the server sends back for whether or not the user is authenticated. You could also add a route to send a response of req.user to get information about the user.

      router.get('/', function(req, res, next) {
       res.send(req.isAuthenticated());
      });

Once you’ve got users saving to the database, go to Postico and verify your data.


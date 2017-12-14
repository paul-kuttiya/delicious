* [setting up mongo](#setting-up-mongo)  
* [variable and start](#variable-and-start)  
* [routing and middleware](#routing-and-middleware)  
* [Templating](#templating)  
  * [helper](#helper)  
* [MVC pattern](#mvc-pattern)  


## Setting up mongo  
* download mongo community server from mongodb.com  
* install with mongo compass(mongo GUI)  
* open command line from installed folder then cd into bin and run `mongod` command   
> for windows create `C:\data\db` folder for mongo to work  

## Variable and start  
* all backend secret variable store in `variables.env`  
* connect mongo db with mongoose and config mongoose in `start.js`  
* `start.js` use dependency `dotenv` which will point and load secrect variable from `variables.env`  
* start script included in `start.js`  
* run `npm start`; watch with `nodemon`(watch and restart when file changes)  
* data base will auto detect and create with `start.js` and `variables.env` if not exists  

## Routing and middleware
* `app.js` contains all of middleware(scripts that load between request response cycle)  
* import route from route modules in `routes` folder then use express to specify route  
```js
//app.js
const index = require('./routes/index');
const app = express();
app.use('/', index);
```

* define request response in node server at `routes/route_path_name.js` then export as module  
```js
//index.js
const express = require('express');
const router = express.Router();

router.get('/', (req, res) => {
  res.send('Hello world!');
});

module.exports = router;
```

> req and res properties are provided by `bodyParser` middleware which parsing JSON, plain text, or just returning a raw Buffer object

* some useful methods for req, res  
```js
// router

// send need to execute only once in router
// Header can only send once per response
res.send("data");
res.json({name: 'abc', age: 20}); //send response as json

//get params with request
//ex url: www.something.com/?name=abc&age=21
req.query; // {name: "abc", age: "21"}
req.query.name; //abc 

//params routes
router.get('/users/:name', (req, res) => {
  req.params.name; //access param :name
});
```

> check express docs for more API

## Templating
* render template for request(will use pug template)  
* all of template will be in `/views`  
* `pug` middleware is configed and pointed to views path in `app.js`  
```js
//router
router.get('/', (req, res) => {
  res.render('hello', {
    name: 'abc',
    age: req.query.name
  });
});
```

* `pug` reuseable parts of template with `block`  
```pug
<!-- layout -->
.content
  block content
```
```pug
<!-- content -->
extends layout

block content
  p Hello
```

* if specified inside `block` will use default for `block`, `extend` and define `block` in content to overwrite the default  
```pug
<!-- layout -->
block header
  header.top
    nav.nav
      p some header
```

### helper
* defined reusable variables, libraries and functions in `helpers.js` to export and easily use in our template or file  

* import `helpers` in `app.js` and define it as `locals` for express middleware to be able to use across express req/res cycle  
```js
// app.js
const h = require('./helpers');

// define locals to use across res before routes
app.use((req, res, next) => {
  res.locals.h = helpers;
  //...
  next();
});
```

> `res.locals` is an object that contains response local variables scoped to the request, and available only to the view rendered during that request / response cycle (if any)

* `app.locals` has properties that are local variables within the application.
```js
app.locals.title = 'My App';
app.locals.strftime = require('strftime');
app.locals.email = 'me@myapp.com';
```

## MVC pattern  
* Model: query data  
* View: template  
* Controller: control route and paths  


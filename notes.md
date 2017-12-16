* [setting up mongo](#setting-up-mongo)  
* [variable and start](#variable-and-start)  
* [routing and middleware](#routing-and-middleware)  
* [Templating](#templating)  
  * [helper](#helper)  
  * [mixins](#mixins)  
* [MVC pattern](#mvc-pattern)  
* [Store](#store)  
  * [create store model with mongoose](#create-store-model-with-mongoose)  
  * [get add store page](#get-add-store-page)  
  * [post add store with async await](#post-add-store-with-async-await)  
  * [flash message](#flash-message)  
  * [query db for stores and get stores page](#query-db-for-stores-and-get-stores-page)  
  * [find and edit](#find-and-edit)  
  * [add timestamp and location to Model](#add-timestamp-and-location-to-model)  
  * [geolocation with google map](#geolocation-with-google-map)  
  * [upload and resizing image](#upload-and-resizing-image)  
  

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
* after config all middleware, import route from route modules in `routes` folder then use express to specify route  
* handle errors after define routes in `app.js`  

```js
//app.js
const index = require('./routes/index');
const app = express();

//...middleware
app.use('/', index);

//...handle errors after lookup for routes
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

### mixins
* can use as reusable template or script for pug  
* define pug mixins in `views/mixins/_fileName.pug`  
```pug
mixin something(store={})
  p something
```

```pug
extends layout

<!-- include the mixins -->
include mixins/_fileName
```

## MVC pattern  
* Model: query data, create tables, define schema, slug  
* View: template  
* Controller: control route, paths and maybe some custom middleware    

```js
//controller
//controllers/storeController.js
exports.homePage = (req, res, next) => {
  res.render('index');
}

//routes
//...other code
const storeController = require('../controllers/storeController');

// router.get('/', controllerRoutes, [middleware]);
router.get('/', storeController.homePage);
```

## Store
### create store model with mongoose
* create model file `models/store.js`  
* define and config mongo schema with mongoose(package to interface with mongo)  
```js
// models/Store.js

//require mongoose dependency
const mongoose = require('mongoose');
//config mongoose how to get data back from db
//use js promise, asyn await or lib such as bluebird
mongoose.Promise = global.Promise;
//use slug lib for url friendly
const slug = require('slugs');

//define schema; all its properties and data type
const storeSchema = new mongoose.Schema({
  name: {
    type: String,
    trim: true,
    required: 'Please enter a store name', //overwrite mongoose default errors with message
  },
  slug: String,
  description: {
    type: String,
    trim: true
  },
  tags: [String]
});

// mongoose middleware
// mongoose hook to hook in something before mongo save to db
storeSchema.pre('save', function(next) {
  // use es5 not arrow function, need the storeSchema context
  // if use arrow function this will be window 

  // run hook only when name is modified
  if (!this.isModified('name')) {
    //skip and return
    return next(); 
  }

  this.slug = slug(this.name);
  // call next to finish the hook
  next();
});

//export schema as module; export as a function
//mongoose.model('ModelName', modelSchema);
module.exports = mongoose.model('Store', storeSchema);
```

* in `start.js` instantiated connection only once and then reused it; Singleton  
```js
//start.js
//...other code
// Singleton mongo model connection
require('./models/Store');
```
How errorHandler(`errorHandlers.js`) is handled from request:
> when set model schema `required` field  to `true` or overwrite with custom string, if request comes in without those required field, in `app.js` the error handler `catchErrors` which wrapped in routes will call `next`.  <br><br> 
Then `errorHandlers.flashValidationErrors` function will check for request errors. If there is an error, will iterate and display errors as flash messages and redirect back to previous request. Alternatively, if there is no error, will call next and pass `error` to other errorHandler.

### get add store page
* create view, route and controller  

* controller implementation
```js
// storeController.js
// exports as exports properties
exports.addStore = (req, res) => {
  res.render('editStore', { title: 'Add Store' })
}
```

* routes implementation  
```js
// routes/index.js
// ...other code

router.get('/add', storeController.addStore);

// ...other code
```

* view implementation for form mixin
```pug
<!-- views/mixins/_storeForm.pug -->
mixin storeForm(store = {})
  form(action="/add" method="POST" class="card")
    label(for="name") Name
    input(type="text" name="name")
    label(for="description") Description
    textarea(name="description")
    - const tags = ["restaurant", "Wifi", "cheap", "cash only", "credit card"];
    ul.tags
      each tag in tags
        .tag.tag__choice
          input(type="checkbox" id=tag, value=tag name="tags")
          label(for="choice")= tag
    input(type="submit" value="Save" class="button")      
```

### post add store with async await
* implement route to handle post request  
```js
// routes/index.js
// ...other code

router.post('/add', storeController.createStore);
```

* implement controller, use mongoose to import schema and save to db when post asyncronously

> Mongoose is already define in start.js to use promise for db interaction `mongoose.Promise = global.Promise`

Mongoose use promise to interact with db, save data as model obj using schema  
```js
// controllers/storeController.js
// ...other code
const mongoose = require('mongoose');
// with singleton, mongoose need to import model(schema) once, which already did in start.js
// after singleton mongoose model can be reference anywhere right away
const Store = mongoose.model('Store'); // exported from models/Store.js as 'Store'

// controller 
exports.createStore = (req, res) => {
  // use Store model schema to create new Store obj
  const store = new Store(req.body);
  // save store asyncronously with mongoose
  store.save() // store.save() will return the Promise resolve if successful
       .then(stores => {
         // return promise
       })
       .then(stores => {
         res.render('templateView', { stores: stores }); // render with resolve from promise
       })
       .catch(err => {
         throw Error(err); // catch error if has one
       })
};
```

Use async await instead of Promise  
```js
// controllers/storeController.js
// ...other code
const mongoose = require('mongoose');

// exported from models/Store.js as 'Store'
// mongoose connect to Store model and schema 
const Store = mongoose.model('Store'); 

// controller with async await
// mark function that is async
exports.createStore = async (req, res) => {
  try {
    const store = new Store(req.body);
    await store.save(); // mark the async return with await
  } catch(err) {
    // catch error
  }
};
```
Instead of catching errors in all controller wrap the function with higher order function(function that take another func as arg)
```js
// handlers/errorHandlers.js

// We will wrap the routes with catchError, which will call next if errors occur in routes
// after next is call in express router middleware in app.js, it will go and look below app.use("path", "route") for a suitable errorhandler code

// export as properties
exports.catchError = (fn) => {
  return function(req, res, next) {
    return fn(req, res, next).catch(next);
  }
}
```

```js
// routes/index.js
// ...other code
const { catchErrors } = require('../handlers/errorHandlers'); // ES6 decomposition pull prop from obj and assign to const

// use js composition to wrap function
// will return next and find error in app.js if errors occured
router.post('/add', catchErrors(storeController.createStore));
```

```js
// controllers/storeController.js
exports.createStore = async (req, res) => {
  // let error happen which we wrapped with handle errors
  // and push along in middleware in app.js to find suitable errors
  const store = new Store(req.body);
  await store.save();
  res.redirect("/");
};
```

### flash message
* flash is implemented with `connect-flash` middleware  

* use middleware to pass flash request to response's locals before hit the route and response back  

* config middleware to config and store flash message  
```js
// app.js
// ... other code
app.use((req, res, next) => {
  // ... others
  // set flashes as locals variables before hit the routes
  // flashes variable will available to use in all response templates
  res.locals.flashes = req.flash(); 
  next();
})
```

* implement flash in view template
```pug
<!-- layout -->
block messages
  if flashes
    .flash-messages
      
```

* implement flash in controller  
```js
// controllers/storeController.js
// ...

exports.createStore = async (req, res) => {
  // save and return mongo db object as promise resolve and store in store const
  const store = await (new Store(req.body)).save(); 

  // flash middleware
  // req.flash('flashType', 'message')
  req.flash('success', `Successfully created ${store.name}. Leave a review?`);

  // when redirects new request is issued with flashes in req
  // flash in req will then store in res.locals.flashes
  // then it will be available in template as res variables
  res.redirect(`/store/${store.slug}`); // store return db promise resolve obj
}
```

> Flash only works when session is implemented, and is a part of session  

### query db for store and get stores page
* create controller, query the db with mongoose asyncronously  

> don't forget to wrap error handler in `routes` for any controller that use async callback 

```js
// storeController.js
// other code...
const Store = mongoose.model('Store'); // conect to model

exports.getStores = async (req, res) => {
  //1. Query db for all stores
  const stores = await Store.find(); // find all data in Store and return promise

  res.render('stores', { title: 'Stores', stores }); //ES6 stores: stores
}
```

* create reuseable mixin UI then loop and display stores in `stores` view
```pug
<!-- views/mixins/_store.pug -->
mixin store(store={})
  .store
    .store__hero
      .store__actions
        .store__action.store__action--edit
          a(href=`/stores/${store._id}/edit`)
            != h.icon('pencil')
      img(src=`/uploads/${store.photo || 'store.png'}`)
      h2.title
        a(href=`/store/${store.slug}`)= store.name
    .store__details
      p= store.description.split(' ').slice(0, 25).join(' ')
```

```pug
<!-- stores -->
extends layout

include mixins/_store

block content
  .inner
    h2= title
    .stores
      each store in stores
        +store(store)
```

> tips: `!=` in pug will tell pug to render and escape html

### find and edit 
* include ui like for edit  
```pug
.store__action.store__action--edit
  a(href=`/stores/${store._id}/edit`)
    != h.icon('pencil')
```

* Implement the route for GET 
```js
// router.js
// other code...

router.get("/stores/:id/edit", catchErrors(storeController.editStore));
```

* Implement controller  
```js
// storeController.js
// other code...
const mongoose = require('mongoose');
const Store = mongoose.model('Store');

exports.editStore = async (req, res) => {
  // find the store with ID from params url
  const store = await Store.findOne({ _id: req.params.id });

  // render edit form, pass query from DB to view
  res.render('editStore', { title: `Edit ${store.name}`, store }); // ES6 store: store
};
```

* modify mixin `_storeForm` to show the value and config route
```pug
<!-- if no store pass in value is default to {} -->
mixin storeForm(store={})
  form(action=`/add/${store.id || ''}` method="POST" class="card")
    label(for="name") Name
    input(type="text" name="name" value=store.name)
    label(for="description") Description
    textarea(name="description")= store.description
    - const tags = ["restaurant", "Wifi", "cheap", "cash only", "credit card"];
    - const storeTags = store.tags || [];
    ul.tags
      each tag in tags
        - const checked = storeTags.includes(tag);
        .tag.tag__choice
          input(type="checkbox" id=tag, value=tag name="tags" checked=checked)
          label(for="choice")= tag
    input(type="submit" value="Save" class="button")  
```

* Implement the route for POST
```js
// route
// ...
router.post("/add/:id/", catchErrors(storecontroller.updateStore));
```

* Implement the controller to handle POST
```js
// storeController
// ...
exports.updateStore = async (req, res) => {
  // find store from db with params and update in one method
  // Model.findOneAndUpdate({ query }, data, [options]);
  const store = await Store.findOneAndUpdate({ _id: req.params.id }, req.body, {
    new: true, // return updated data not old one from mongo
    runValidators: true // validates the required field in schema not only on create but on update as well
  }).exec(); // method to ensure execution for mongo

  // flash
  req.flash("success", `Successfully updated <a href=/stores/${store.slug}><strong>${store.name}</strong></a>`);

  // redirect
  res.redirect(`/add/${store._id}/edit`);
};
```

### add timestamp and location to Model
* implement more field in Store model  
```js
// models/Store.js
// ...other code
const storeSchema = new mongoose.Schema({
  name: {
    type: String,
    trim: true,
    required: 'Please enter a store name', //overwrite mongoose default errors with message
  },
  slug: String,
  description: {
    type: String,
    trim: true
  },
  tags: [String],
  // add new fields below
  created: {
    type: Date,
    default: Date.now()
  },
  // nested attrbites store in db
  location: {
    type: {
      type: String,
      default: 'Point' // will located point map in mongo compass GUI
    },
    coordinates: [{
      type: Number,
      required: 'Must have coordinates!'
    }],
    address: {
      type: String,
      required: 'Must have address!'
    }
  }
});
``` 

> in middleware config `app.js`:  <br><br>
`app.use(bodyParser.urlencoded({ extended: true }))` will allow us to use rich objects and arrays to retrive data from db or save db into db as nested attributes <br><br>
For example, the string `person[name]=bobby` and `person[age]=3` will be converted to:

```js 
person: {
    name: 'bobby',
    age: 3
}
```

* implement new model fields in view
```pug
<!-- _storeForm.pug -->
mixin storeForm(store={})
  form(action=`/add/${store.id || ''}` method="POST" class="card")
    //- other field
    //- add address
    label(for="address") Address
    input(type="text" id="address" name="location[address]" value=(store.location && store.location.address))
    label(for="lng") Address Lng
    input(type="text" id="lng" name="location[coordinates][0]" value=(store.location && store.location.coordinates[0]) required)
    label(for="lat") Address Lat
    input(type="text" id="lat" name="location[coordinates][1]" value=(store.location && store.location.coordinates[1]) required)
    //- other field
```

* type of `'Point'` will lose when user update query, in controller force to update location type to `'Point'`
```js
// StoreController.js
exports.updateStore = async (req, res) => {
  // force update location to Point
  req.body.location.type = 'Point';

  // other code...
}
```

### geolocation with google map
> will use `bling.js` library which convert Javascipt DOM query to `$` and `$$`, additionally convert `addEventListener` to `on`

* include `layout` view with google API  

* implement client side using google place api to autocomplete when type in address  

* wrap google API with custom module to insert lat ang long coordinates in input with enter address  
```js
// public/javascripts/delicious-app.js
import { $, $$ } from './modules/bling';
import autocomplete from './modules/autocomplete';

// call imported function from custom module autocomplete and pass in input DOM
autocomplete($('#address'), $('#lat'), $('#lng'));
```
```js
// public/javascripts/modules/autocomplete.js
function autocomplete(address, lat, lng) {
  if (!address) { return; } // no input

  const addresses = new google.maps.places.Autocomplete(address); // google api

  // google API; addListener, place_changed, getPlace()
  addresses.addListener('place_changed', () => {
    // get google place obj from google API
    const place = addresses.getPlace();

    // insert lat and lng input dom with google place API
    lat.value = place.geometry.location.lat();
    lng.value = place.geometry.location.lng();
  });

  // prevent enter to submit form
  address.on('keydown', (e) => {
    if (e.keyCode === 13) { e.preventDefault(); };
  });
}

// export as module
export default autocomplete;
```

### upload and resizing image
* use middleware `multer` to upload the file and `jimp` to resize the file  

* store image NAME.EXT in MongoDB, store ACTUAL image file in server(or cloud)

* file needs to be miltipart when uploaded, modify the form  
```pug
//- _storeForm.pug
mixin storeForm(store={})
  form(action=`/add/${store._id || ''}` method="POST" class="card" enctype="multipart/form-data")
  //- ...
``` 

* uploading file process  
  * use `multer` middleware to handle multipart(file) upload request in controller  
  * `multer` will read into temp memory as buffer  
  * then async with `jimp`(implement with promise) for resizing  
    * detect file type with mimetype and rename file, then point the file name to `req.body.fieldName`, which will be save in MongoDB when controller processed  
    * then assign `photo` variable to await for jimp's resolve from reading the buffer  
    * after resolve from await buffer reading, next await for `photo` to resize  
    * await for `photo` to write to the image save path   
```js
//storeController.js
//...
const multer = require('multer'); // save file to server
const jimp = require('jimp'); // resize
const uuid = require('uuid'); // unique id for anything
const imgPath = "./public/uploads";

// config multer middleware
const multerOptions = {
  storage: multer.memoryStorage(), // store into memory, not in server disk
  fileFilter(req, file, next) {
    const isPhoto = file.mimetype.startsWith('image/'); // server side validation image with mimetype
    if(isPhoto) {
      // next(err); pass on error
      // next(null, true); no errors, pass on true 
      next(null, true);
    } else {
      next({ message: 'File is not allowed!' }, false); // error obj, pass on false
    }
  }
}

// set middleware to accpet a single file with the A 'name'(name on req.body from form 'name'), can check for multiple fields as well(array).
// The single file will be stored in req.file
exports.upload = multer(multerOptions).single('photo');

// resize asynchronously 
exports.resize = async (req, res, next) => {
  // multer read and process request as file obj in req.file
  // use req.file to check if there is any file being attached
  if (!req.file) {
    // no any file attached, skip to the next middleware
    return next();
  }

  // check the mime type, rename file, and store that name to mongo DB
  const ext = req.file.mimetype.split('/')[1];
  // assign req.body.photo to filename.ext string, will be posted with other requested fields in controller
  const fileName = req.body.photo = `${uuid.v4()}.${ext}`; 

  // next we concern with saving actual file to db
  // read from temp memory and resize with jimp
  const photo = await jimp.read(req.file.buffer);
  // resize the photo obj, width, height
  await photo.resize(800, jimp.AUTO);
  // write file to server path --> file.write(filePath)
  await photo.write(`imgPath/${fileName}`);
}

// controllers ...
```  

* modify form to upload photo  
```pug
//- _storeForm.pug
mixin storeForm(store={})
  form(action=`/add/${store.id || ''}` method="POST" class="card" enctype="multipart/form-data")
    //- other field
    label(for="photo") Photo
    //- client side validation
    input(type="file" name="photo" id="photo" accept="image/gif, image/png, image/jpeg")
    - if store.photo
      img(src=`uploads/${store.photo}` alt=store.name width=200)
```

* add Model schema to store fileName in DB
```js
// models/Store.js
// ...
const storeSchema = new mongoose.Schema({
  {
    //...
  },
  photo: String
}); 
```

* implement middleware in routes
```js
// routes/index.js
// ...
// express router API --> router.post('path', [middleware..], req, res, [next] => {...})

// implement upload and resize middleware for create
router.post('/add', 
  storeController.upload, 
  catchErrors(storeController.resize), // async
  catchErrors(storeController.createStore) // async
);

// implement upload and resize middleware for update store
router.post('/add/:id', 
  storeController.upload, 
  catchErrors(storeController.resize), // async
  catchErrors(storeController.updateStore) // async
);
```

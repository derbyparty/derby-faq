FAQ for Derby 0.6
=================

The purpose of this repository is to document frequently asked questions and
common pitfalls for [derbyjs](http://derbyjs.com).

### Contributing

Pull requests are welcome. If you feel like adding something, fork away!
Call out @zag2art on github or email (zag2art@gmail.com) if you need a hand.

#### Unanswered questions

Please make an [issue](https://github.com/derbyparty/derby-faq/issues) in the
repository. We'll try to answer your question directly in the issue if we can.
If the answer is important and common enough, we'll add it to the FAQ.

## General Information

#### What background should I have before I try derby.js?

  - web development basics (html, css, javascript)
  - [node.js](http://nodejs.org/), particularly [commonjs modules](http://nodejs.org/api/modules.html), [npm](https://www.npmjs.org/doc/), and how to run an http server
  - [express](http://expressjs.com/guide.html), the framework that Derby itself is built on top of.

---
#### How do I get started with Derby 0.6 (the latest version)?

Work through the tutorials:

1. [Running Derby 0.6, Example # 1](https://github.com/dmapper/derby-tutorials/blob/master/derby1.md)
2. [Running Derby 0.6, Example # 2](https://github.com/dmapper/derby-tutorials/blob/master/derby2.md)
3. [Running Derby 0.6, Example # 3](https://github.com/dmapper/derby-tutorials/blob/master/derby3.md)
4. [Derby 0.6: Introduction to components](https://github.com/dmapper/derby-tutorials/blob/master/derby4.md)
5. [Multiplayer game on Derby.js](https://github.com/dmapper/derby-tutorials/blob/master/derby5.md)


Then study the official [derby-examples](https://github.com/codeparty/derby-examples).

---
## Queries

#### How do I count the number of elements in a remote collection? I don't want to fetch the elements themselves.

```js
  var topicsCount = model.query('topics', {
    $count: true,
    $query: {
      active: true
    }
  });

  model.subscribe(topicsCount, function onTopicsCount(err, next) {

    // livedb-mongo returns count queries as an `extraSegment`  https://github.com/share/livedb-mongo/blob/v0.3.2/mongo.js#L227
    // unlike the other supported metaOperators                 https://github.com/share/livedb-mongo/blob/v0.3.2/mongo.js#L7
    // and 0.6 now allows references to `extraSegments`         https://github.com/codeparty/racer/blob/v0.6.0-alpha3/lib/Model/Query.js#L494
    topicsCount.refExtra('_page.topicsCount');

    // ...

    // Async functions must always run a callback. Always.
    next(err); // or page.render()
  });
```

---
#### How do I subscribe to specific objects in a collection (I already have an array of ids for exising elements)?

If the second argument to `model.query` is a string instead of a query object,
the parameter will be interpreted as a path in the local model:

```js
model.query(collection, path)
```

This path should be to a local (non-synchronized) collection, and you probably
also want it scoped to the current page, so you might pick a path such as
`_page.userIds`. You should be able to perform a `model.get(_page.userIds)` and
receive an array of strings, each of which is the id of an object you'd like to
synchronize with the remote. You can also use a special [racer] datatype called
a [refList], but more on that later.

[racer]: https://github.com/codeparty/racer/tree/v0.6.0-alpha3
[refList]: https://github.com/codeparty/racer/blob/v0.6.0-alpha3/lib/Model/refList.js

Where is this necessary? Imagine the "Hello World" of real time apps: a chat.
This chat app has pages of chat rooms where there are thousands of people
constantly coming and going. (Thousands? Duh, you write hot shit!)
Each message posted only contains the `userId` of its author. All the user
info (names, handles, and avatar urls) is locked away in the `users`
collection. We need all the authors' user objects, but if we subscribe to the
entire collection, our browser would crash. We need  to subscribe to just the
members of this room and no more.

Let's build this:
 1. Subscribe to all posts in the room (we're not ready for paging yet).
 2. Start a reactive function that will collect every `userId` that posts a message to the room.
 3. Subscribe to a mutable collection of users, using the array output of the reactive function.

```js
app.get('/chat/:room', function chatRoom(page, model, params, next) {
  var room = params.room;
  var messages = model.query('messages', {room: room});

  model.subscribe(messages, function onMessages(err, next) {
    if (err) next(err);

    // This is that sneaky little refList. To be covered later.
    messages.ref('_page.messages');

    // Runs the model function `pluckUserIds` whenever the data
    // at the `messages` path changes, and writes the array of `userIds`
    // to `_page.userIds`. It's smart: it will run only once on the
    // server when the page is pre-rendered, but will run continuously
    // on the client.
    //
    // Yes, the argument order has changed. If curious, read
    // https://github.com/codeparty/racer/blob/v0.6.0-alpha3/lib/Model/fn.js#L32
    model.start('_page.userIds', 'messages', 'pluckUserIds');

    var users = model.query('users', '_page.userIds');

    model.subscribe(users, function onUsers(err, next) {
      if (err) next(err);

      // ...
      page.render();
    });
  });
});

// Derby renders on both client and server, and does some sneaky little tricks
// to make everything work properly. You must declare your model functions on
// the model event. Ignore this warning at your own peril.
app.on('model', function onModel(model) {
  model.fn('pluckUserIds', function pluckUserIds(messages) {
    var ids = {};

    for (var key in messages) ids[messages[key].userId] = true;

    // Reactive model functions are *synchronous* so
    // you *absolutely* must `return` a value. Do this now.
    return Object.keys(ids);
  });
});
```
---
## Components

#### How to get an access to model's root scope in component's templates?
As is known, components have its own isolated scope so in order to access model's root a #root prefix should be used:

```html
  <ul>
    {{each #root._page.topics as #topic}}
      <!-- ... -->
    {{/}}
  </ul>
```

---
#### How to access the model's root scope within component's code?
Since components have its own isolated scope in order to access a root model you would
use model.root or get the root model from app. For example:

```js
function MyComponent() {}

MyComponent.prototype.init = function(model){
  // model.get('_page.topics') won't work
  var topics = model.root.get('_page.topics');
  // ...

}

MyComponent.prototype.onClick = function(event, element){
  var topics = this.model.root.get('_page.topics');
  // The string below does the same:
  var topics = this.app.model.get('_page.topics');
  // ...

}
```
---
#### How to reference query's result with a local model path in a component?

```javascript
app.get('/', function(page) {
  page.render('home');
});

function Home() {}

app.component('home', Home);

Home.prototype.view = __dirname + '/home.html';
Home.prototype.create = function(model) {
  // var $query = model.query('somedata', {}); Model reference won't work
  var $query = model.root.query('somedata', {}); // Works
  model.subscribe($query, function() {
      model.ref('items', $query);
      // Now the local path 'items' can be used in a component's template
  });
}
```
home.html
```html
<index:>
  <ul>
    {{each items}}
    <li>{{this.title}}</li>
    {{/}}
  </ul>
```
Note that making queries within a component is an anti-pattern. All subscriptions
should be made in a controller (while handling a particular route), components
are intended to work with and displaying data. Getting all the data in a controller
is effective while getting the data separately for each component is not, especially
in case of multiple components.

---
#### How to run reactive functions in a component?
Component's `init` handler suites best for the purpose since it works both
on a server (server's rendering) and a client (client's rendering) sides. It allows us
to send the reactive function via callback of `start` function and not to use
a `model.fn` serialization.

Couple examples:

```js
// 'count' and 'items' are component's private paths
Comp.prototype.init = function(){
  this.model.start('count', 'items', function(items){
    return Object.keys(items).length;
  });
}
```
```js
// 'count' - component's private path, 'items' - global path
Comp.prototype.init = function(){
  // istead of path we can use scope, and scope always takes data from the root
  this.model.start('count', this.model.scope('items'), function(items){
    return Object.keys(items).length;
  });
}
```

---
## Model

#### I don't need all collection's fields in a browser, how to get only particular fields (collection's projection)?

All projections should be declared in a server part of a derby application
```js
// Take into account ShareDB API changes
var backend = store.backend || (store.shareClient && store.shareClient.backend);
backend.addProjection("topic_headers", "topics", "json0", {
  id: true,
  header: true,
  autor: true,
  createAt: true
});

backend.addProjection("users", "auth", "json0", {
  id: true,
  username: true,
  email: true
});
```

Then we can work with 'users' and 'topic_headers' projections same way as with regular collections
```js
model.subscribe('users' function(){
  model.ref('_page.users', 'users');
  // ...
});
```
Note, that 'id' field is required for projections creation. Currently, only white list is supported
(list of fields that are present in a projection). Also, you can set the fields only of the first level of nesting so far.

---
## Database

#### They say there's a possibility to make it work without redis by using only mongodb, how to achieve this?

In a server part of a derby application when creating a store object you should use only mongodb data:

```js
  var store = derby.createStore({
    db: liveDbMongo(mongoUrl + '?auto_reconnect', {safe: true})
  });
```

Keep in mind, that redis is required, if you're planning horizontal scaling (run multiple derby servers
simultaneously)

---
#### I've already got a mongodb database, how can I use it with derby?

You can use it after a certain processing. The thing is, that share.js [sharejs](http://sharejs.org/)
is used for solving conflicts in derby. It adds additional data for each record in a collection like number of version, object's type (from share.js perspective) etc. Also, with each existing collection there's another collection, that has "ops" suffix, for example "users_ops". It is used for storing operations log. This log also requires initialization. Module [igor](https://github.com/share/igor) does it for you.


```bash
npm install igor

coffee itsalive.coffee --db project
```
For details visit [page](https://github.com/share/igor)

---
## Styles

#### How to use less styles in derby?

Starting from derby 0.6.0-alpha9, the block which is responsible for compilation
of *.less files into *.css is placed in a separate module called derby-less.

It should be included into a project as follows:
```bash
npm install derby-less
```
then in a [main] js file of your application you'll need to add a line:
```js
// Adding Less support
app.serverUse(module, 'derby-less');
```
Note that the line above should be placed before any app.loadStyles() call

For instance:
```js
var derby = require('derby');
var app = module.exports = derby.createApp('example', __filename);

// Add Less support (before loadStyles)
app.serverUse(module, 'derby-less');

app.loadViews(__dirname);
app.loadStyles(__dirname);

app.get('/', function(page, model) {
  page.render();
});
```

That's it, you can now use less files instead of css.

---
#### How to use stylus in derby?

The process is absolutely the same as it is for less, you'll just need to install
a module called derby-stylus instead.

---
## Views

#### How to insert an unescaped html fragment in a template?

You would use an 'unescaped' modifier:
```html
<header>
  {{topic.header}}
<header>

<!-- did this only on example purpose, don't know when it can be useful -->
<article title="{{unescaped topic.unescapedTitle}}">
  {{unescaped topic.html}}
</article>
```

Warning: this is a potential security hole. Your data should be cleaned up
   from dangerous tags, attributes data should be escaped. Generally, think twice
   before using a code like this.

---
#### How to make a particular block in a template non-reactive (it won't update immediately after the model data has changed)?

First, it should be mentioned that if we don't need a reactivity at all then instead of
data subscription we should just make a request for its current state, i.e. do model.fetch
instead of model.subscribe:

```js
  // Here the data will be changing reactively
  model.subscribe('topics', function(){
    // ...
  });

  // And here it won't
  model.fetch('topics', function(){
    // ...
  });
```
It's important to understand, that for now we're talking only about model and updates coming from the server. Once we had done model.fetch and then if we added something to the collection using model.add - our data will appear in a database. But if data was added to the collection by somebody else - it won't come to the server and not appear in the db.

Now let's talk about reactivity in html templates. All data bindings are reactive by default, i.e. as soon as data has changed (doesn't matter whether updates came from the server or model has changed), the changes are immediately shown in an html template though you can control the behaviour.

In order to control a reactivity in templates, reserved words 'bound' and 'unbound' are used either in block notation or as expressions modifiers:

```html
<p>
  <!-- the binding is reactive by default -->
  {{_page.text}}
</p>
<p>
  <!-- reactive binding is set explicitly -->
  {{bound _page.text}}
</p>

<!-- Within the block all bindings are reactive by default -->
{{bound}}
  <p>
    <!-- the binding is reactive since it's within a bound-block -->
    {{_page.text}}
  </p>
  <p>
    <!-- the binding isn't reactive because it's set explicitly -->
    {{unbound _page.text2}}
  </p>
{{/}}

{{unbound}}
  <p>
    <!-- the binding is not reactive because it's within a unbound-block -->
    {{_page.text}}
  </p>

  <p>
    <!-- the binding is reactive because it's set explicitly -->
    {{bound _page.text}}
  </p>
{{/}}

```

Naturally, blocks can be nested for convenience.

---
#### How to update a non-reactive block in a certain moment?

In order to re-render an unbound-block you would use a key word 'on', as follows:

```html
{{on _page.trigger}}
  {{unbound}}
    <!-- non-reactive html -->
  {{/}}
{{/}}

<!-- click button to update -->
<a href="#" on-click="refresh()">Refresh</a>
```
Changing _page.trigger after button was pressed:
```js
app.proto.refresh = function(){
  app.model.set('_page.trigger', !app.model.get('_page.trigger'))
}
```
---
#### How to bind a reactive variable to the select element?

No events required, derby does it for you. Learn from the example:

```html
<!-- array _page.filters has objects with id and name fields - the filter's identifier and filter's name -->

<select>
  {{each _page.filters as #filter}}
      <option selected="{{_page.filterId === #filter.id}}" value="{{#filter.id}}">
        {{#filter.name}}
      </option>
  {{/}}
</select>
```

After selection _page.filteredId will have an id of the selected filter.

When binding to option, derby expects to meet an equity check in the 'selected' attribute (equities itself are needed for initial value setting). It's assumed that the left parameter of the check would be a path that will take a value from the option's value attribute when selected.

---

#### How to bind a reactive variable to the input type="radio" element?

No need in catching events, derby does it for you:

```html
<label>
<input type="radio" name="opt" value="one"   checked="{{_page.radioVal === 'one'  }}">One
</label>
<label>
<input type="radio" name="opt" value="two"   checked="{{_page.radioVal === 'two'  }}">Two
</label>
<label>
<input type="radio" name="opt" value="three" checked="{{_page.radioVal === 'three'}}">Three
</label>
```

Eventually _page.radioVal will contain either 'one', 'two' or 'three' depending on what user's selected.

When binding to
```html
<input type="radio">
```, derby expects to meet an equity check in a checked attribute (equities itself are required for initial value setting). It's assumed that the left parameter of the check would be a path that will update from the option's value attribute when selected.

---
#### How to bind a reactive variable to the textarea element?

Very simple:

before 0.6.0-alpha6
```html
 <textarea value="{{message}}"></textarea>
```
starting from the 0.6.0-alpha6
```html
 <textarea>{{message}}</textarea>
```

---
#### How to bind a view-function to the input (any kind of html input) element? (How to write `get/set` view-function)

Let's say we want to write a function that transforms RGB values and optional alpha channel into the hex-color (alpha channel makes hex-color closer to white). But we also want it to work the other way around in case we bind it to the input -- so that when we type the hex color it decodes it into RGB (with alpha channel equal to 1):

```html
<input type='text' value='{{ hexify(menuColor.r, menuColor.g, menuColor.b, menuColor.alpha) }}'>
```

To define a get/set view-function you difine an object with `set` and `get` methods:

```js
app.proto.hexify = {
  get: function(r, g, b, alpha){
    var color = convertRgbToHex(r, g, b);
    // Make color closer to white if 'alpha' is specified
    color = lightenColor(hex, 1 - alpha);
    return color;
  },
  // First argument in 'set' function is 'inputValue' -- each time you type something
  // into the input 'set' function is going to execute with 'value' being equal to
  // the new input value.
  // ATTENTION! WE DON'T COUNT 'inputValue' ARGUMENT WHEN WE RETURN STUFF OUT OF 'set'
  set: function(inputValue, r, g, b, alpha){
    var color = convertHexToRgb(inputValue);
    // The 'set' function has to return an array -- all the arguments you want to set back
    // into the paths that you have specified as arguments of view function.
    // In this case it's 'menuColor.r', 'menuColor.g', 'menuColor.b', 'menuColor.alpha'.
    // In the same order they are specified when you call the function, starting with
    // the 0th argument (WARNING! The 0th argument as defined in 'get' function --
    // we don't count the 'inputValue' argument of 'set' function).
    return [color.r, color.g, color.b, 1];
  }
};
```

Now a couple of usecases you need to know about:

1. If you only need to set first 3 arguments out of 4 you should just write those 3 in the array:

    ```js
    return [color.r, color.g, color.b];
    ```

2. If you only need the last value, you can pass an object instead of an array specifiyng the
    index of returning path that you want to set as a key. So to set `menuColor.alpha` back you
    would do:

    ```js
    return {3: color.alpha};
    ```

    You can also use the object-way of return even when you want to set all the pathes:

    ```js
    return {0: color.r, 1: color.g, 2: color.b, 3: color.alpha};
    ```

    but array syntax is preferrable when you need to return value/values starting with 0th one

3. Even when you only want to set back a single path which is the 0th argument you still need to return it
    wrapped into an array:

    ```js
    return [color.a];
    ```

4. (Considered BAD practice) If you want 'set' of view function to set some stuff which is not
    related to what is going on in 'get' function, you can do it by simply adding this additional path
    when you call the view-function:

    ```html
    <input type='number' value='{{ formatNumber(price, priceIsModified) }}'>
    ```

    ```js
    app.proto.formatNumber = {
      get: function(number){
        // limit to 2 decimals
        return ~~((number || 0) * 100) / 100
      },
      set: function(number, modificationFlag){
        number = parseFloat(number) || undefined;
        return [number, true];
      }
    };
    ```

5. 'Set' of view function for select is triggered for each option with value true (selected option) or false (all other option's), so we should ignore all those values which false::

  ```html
  <select>
    <option selected="selectCity(myCity, 'Moscow')">Moscow</option>
    <option selected="selectCity(myCity, 'Krasnodar')">Kransodar</option>
    <option selected="selectCity(myCity, 'Penza')">Penza</option>
  </select>
  ```

  ```js
  app.proto.selectCity = {
    get: function(myCity, testCity){
      return myCity === testCity;
    },
    set: function(value, myCity, testCity){
      if (!value) return;
      return [testCity];
    }
  };
  ```

---

## Events

#### How to access the event object in an event handler?

You should pass a predefined '$event' parameter into the event handler function.
```html
<a href="http://derbyjs.com" on-click="click($event)">
```
```js
app.proto.click = function(e) {
  e.preventDefault();

  // ..
}
```

---
#### How to access the dom element in an event handler?

A predefined '$element' parameter should be passed into the event handler function.
```html
<a href="http://derbyjs.com" on-click="click($event, $element)">
```
```js
app.proto.click = function(e, el) {
  e.preventDefault();

  $(el.target).hide();
  // ..
}
```

---
#### Does derby 0.6 have handler similar to app.ready in version 0.5 which runs only once in a browser in the very beginning required for client script initialization?

The handler is in app.proto.create. Let's have a look:

index.js
```js
// ...

// Called only once in a browser after the first page rendering
app.proto.create = function(model){
  // Initializing jQuery AJAX
  $.ajaxSetup({
    dataType: 'json',
    contentType: 'application/json; charset=utf-8',
    processData: false
  });
}
```

---
#### Derby 0.5 had an app.enter handler allowed you to catch a moment of a page creation, how to make it in derby 0.6?

In order to catch the moment of a page creation (when dom is formed) you need every page to be a component. Component's method 'create' is used for this.

The approach looks something like this:

index.html
```html
<!-- точка входа - это наш лейаут -->
<!-- entry point is our layout -->
<import: src="./home">
<import: src="./about">

<Body:>
  <!-- place for a header -->

  <!-- pages content goes here -->
  <view name="{{$render.ns}}"/>

  <!-- place for a footer -->
```
home.html - 'home' page will be a separate component
```html
<index:>
  <h1> Home </h1>
  <!-- ... -->
```

about.html - 'about' page will be a separate component
```html
<index:>
  <h1> About </h1>
  <!-- ... -->
```
index.js
```js
// ...

app.component('home',  Home);
app.component('about', About);

function Home (){};
function About(){};

Home.prototype.create = function(model, dom){
  // 'home' page has been rendered
}

About.prototype.create = function(model, dom){
  // 'about' page has been rendered
}

// ...

app.get('/', function(page){
  page.render('home');
});

app.get('/about', function(page){
  page.render('about');
});
```

---
#### How to listen to a specific control event instead of the whole document event in a component?

In order to do this you will need to specify a 'target' parameter (it's 'self.input' in the example) when subscribing to the event:

View:
```html
<index: element='demo'>
    <input as='input' type='text' value="{{@value}}" />
```
Code:
```js
module.exports = Demo;

function Demo() {
}

Demo.prototype.view = __dirname + '/demo.html';

Demo.prototype.create = function (model, dom) {
    var self = this;
    dom.on('keydown', self.input, function (e) {
        if (e.keyCode === 13) {

        }
    });
};
```
---

## Modules

#### How to add a jade template engine to a derby application?
In order to include jade into a derby app use [derby-jade](https://github.com/cray0000/derby-jade) module.

Installing the module:
```bash
npm install derby-jade
```

In a derby app the module should be included before calling app.loadViews():
```js
app.serverUse(module, 'derby-jade');
```

Make sure that you have derby version 0.6.0-alpha7 or newer

---
#### What module should I use for authorization in derby?

Use newly created specially for derby 0.6. [derby-login](https://github.com/vmakhaev/derby-login) module.

The module also supports registration/authorization via social networks since it utilizes [passportjs](passportjs.org).

---
#### How to include client scripts into a derby-app, e.g. jQuery?

If the script is on cdn-server, you can include it with a script tag within an html template, for instance:

```html
<Body:>
  <!-- ... -->
  <script src="//code.jquery.com/jquery-1.11.0.min.js"></script>
```

Surely, you can put entire script there but a better solution would be to use a Browserify interface for the purpose. The method is also good because your script is included into a bundle, which is loaded to a page as a second query (after html itself with embedded styles was rendered) which gives a high render speed.

Here is the recommended way:

```js
// On the server side in server.js file

store.on('bundle', function(browserify){
  // local path (from project's root) of the script file
  browserify.add("../js/minified/jquery-1.11.0.min.js");
});
```

We usually use bower when we want to include client scripts. In grunt file we have tasks that are set up for minification and concatenation of all vendor scripts into a single one. Result vendor.min.js is included via the method described above.

---
## Writing modules

#### How to check whether derby code is running on the client or on the server?

There's a module with utilities in derby that has required flag:

```js
var derby = require('derby');

if (derby.util.isServer) {
  // the code runs only on the server
}

if (!derby.util.isServer) {
  // the code runs only on the client
}
```

---
#### How to require a module that will run only on the server of a derby application?

Let me explain first the specificity of requiring modules we need only on the server side. The thing is that all modules of a derby application which are included through a classic CommonJS require statement are loaded into 'bundle' hence they'll be copied to client's browser. We don't want redundant data to be in client's browser especially if the module will work only on the server. That's why instead of regular require we use serverRequire from the derby utils set:

```js
var derby = require('derby');

if (derby.util.isServer) {
  // The package will not be loaded into client's browser
  var myPackage = derby.util.serverRequire(module, 'mypackage');
  // ...
}
```

---
#### How to check if an application is running in production (isProduction)?

A flag derby.util.isProduction is available both on client and server. If we look at util source we'll see that the flag is set using a standard nodejs approach:

```js
  var isProduction = process.env.NODE_ENV === 'production';
```

You should set NODE_ENV environment variable if necessary.

Example of usage (works both on the client and the server):

```js
  if (derby.util.isProduction) {
    // ...
  } else {
    // ...
  }
```

---
## Client and server interaction

#### Is it possible to catch client connection/disconnection on the server?

Yes it is, on the server:
```js

// Works when new client has connected
store.on('client', function(client){
  console.log('Client connected: ', client.id);

  // Close event fires when client has disconnected
  // (losing connection / closing a page)
  client.on('close', function(reason){
    console.log('Client disconnected: ', client.id);
  });
});
```

It should be mentioned, that on the server each application has 'clients' field - hash consists of all clients that are connected in the moment. It means we can walk through each of them, e.g:

```js
console.log('Clients connected in the moment:');
for (var id in app.clients) {
  console.log('Client: ', app.clients[id].id);
}
```

---
#### What if I need to run a chunk of code on the server only, does derby have an rpc analog?

Yes it does. On the client you do:

```js
// Sending data to the server, getting result as a response
app.model.channel.send('myEvent', data, function(result) {
  //
  // Handle the response, if needed, callback is not required.
  //
});
```

On the server:
```js
// On each new client's connection
//
store.on('client', function(client) {

  // Register handler for the event called myEvent
  client.channel.on('myEvent', function(data, cb) {

    //
    // Logic is here, not available on the client
    // Suppose, the logic's result is in result variable.
    //

    cb(result);
  });
});
```
---
#### How to notify all clients from the server?

There's a hash on the server with all connected clients info, so we can send a message to all of the clients as follows:


```js
for (var id in app.clients) {
  app.clients[id].channel.send('myEvent', data);
}
```

Handling of myEvent on the client would be:
```js
this.model.channel.on('myEvent', function(data) {  
  // The event reaction
});
```

Note, that when sending the message we can also pass a callback, which will contain a response from the client. On the client the response should be sent via `return` statement.

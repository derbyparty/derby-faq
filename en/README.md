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

1. [Running Derby 0.6, Example # 1 (Russian)](http://habrahabr.ru/post/221027/)
2. [Running Derby 0.6, Example # 2 (Russian)](http://habrahabr.ru/post/221703/)
3. [Running Derby 0.6, Example # 3 (Russian)](http://habrahabr.ru/post/222399/)

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
}

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
  var count = this.model.at('count');
  // count.path() returns something like: $components._1.count
  this.model.root.start(count.path(), 'items', function(items){
    return Object.keys(items).length;
  });
}
```

---
## Model

#### I don't need all collection's fields in a browser,
how to get only particular fields (collection's projection)?

All projections should be declared in a server part of a derby application
```js
store.shareClient.backend.addProjection("topic_headers", "topics", "json0", {
  id: true,
  header: true,
  autor: true,
  createAt: true
});

store.shareClient.backend.addProjection("users", "auth", "json0", {
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

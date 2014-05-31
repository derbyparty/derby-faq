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

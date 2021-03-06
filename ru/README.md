FAQ по Derby 0.6 (на русском)
=====================

В этом репозитории я буду собирать часто задаваемые вопросы, а так же всевозможные тонкости [derbyjs](http://derbyjs.com)

### Как я могу добавить что-то в FAQ?

Напишите мне @zag2art сообщение через гитхаб или на почту (zag2art@gmail.com), а лучше сделайте форк, добавьте информацию и посылайте пулл-реквест.

### У меня есть вопрос, ответа я не знаю, но думаю всем бы было полезно

Оформляйте [issue](https://github.com/zag2art/derby-faq-ru/issues) в данном репозитории - постараюсь ответить, и если посчитаю вопрос достаточно интересным и общим - добавлю его в FAQ.

## Общая информация

#### Как на данный момент лучше всего изучать derby 0.6?

Проработать туториалы:

1. [Изучаем Derby 0.6, пример #1](http://habrahabr.ru/post/221027/)
2. [Изучаем Derby 0.6, пример #2](http://habrahabr.ru/post/221703/)
3. [Изучаем Derby 0.6, пример #3](http://habrahabr.ru/post/222399/)
4. [Введине в компоненты derby 0.6](http://habrahabr.ru/post/224831/)

Далее изучить примеры из [derby-examples](https://github.com/codeparty/derby-examples)

---
#### Что нужно знать до начала изучения derby.js?

1. необходимы базовые знания по веб-разработке (html, css, javascript);
2. nodejs — нужно понимать commonjs-модули, npm, знать, как запустить стандартный http-сервер;
3. expressjs — приложения derby строятся поверх приложений express, поэтому хорошо бы иметь об экспресс и серверном js какие-то базовые знания (подключение модулей, обработка запроссов и т.д.)

По nodejs и expressjs я бы посоветовал пройти курс [Ильи Кантора](http://learn.javascript.ru/nodejs-screencast) с обязательной практикой.

---
#### Как быстро создать базовый проект Derby 0.6?

Используйте [generator-derby](https://github.com/derbyparty/generator-derby). Он поддерживает javascript/coffeescript, опционально jade, stylus, redis.

---
#### Что делать, если я нашел баг?

1. `rm -rf node_modules & npm cache clear & npm install`
2. попробуйте в другом браузере
3. отключите расширения браузера
4. поищите в [Issues](https://github.com/derbyjs/derby/issues), если ничего не нашли, то открывайте новое. В идеале приложите минимальный пример, повторяющий баг.

---
## Запросы

#### Как сделать реактивный запрос к количеству элементов в коллекции (сами элементы мне не нужны)?

```js
  var topicsCount = model.query('topics', {
    $count: true,
    $query: {
      active: true
    }
  });

  model.subscribe(topicsCount, function(){
    topicsCount.refExtra('_page.topicsCount');

    // ...
  });
```

---
#### Как подписаться на определенные элементы коллекции (у меня уже есть реактивный список id, нужных элементов)?

В model.query обычно передаются 2 параметра, имя коллекции и объект с параметрами запроса, но есть еще один синтаксис:
```js
model.query(collection, path)
```

Здесь path - это путь к массиву id-шников документов, например, "_page.userIds", причем сам массив, вполне себе может быть реактивным.

Для чего это может быть нужно. Представьте себе чат, мы выводим страницу с одной из комнат чата. Туда постоянно входят новые люди, что-то там пишут. В сообщениях чата мы храним только id юзера, а остальная информация хранится в коллекции users. Естественно, для вывода сообщений имена юзеров нам нужны. Всех юзеров грузить на клиента смысла нет, нужны только те, сообщения которых есть в нашей комнате.

Cделаем так:
 1. подпишемся на сообщения в комнате,
 2. запустим реактивную функцию, которая будет собирать id всех юзеров, от которых в нашей комнате есть сообщения,
 3. подпишемся на коллекцию users, используя в качестве списка id, результат выполнения реактивной функции

```js
app.get('/chat/:room', function(page, model, params, next){
  var room = params.room;

  var messages = model.query('messages', {room: room});

  model.subscribe(messages, function(){

    messages.ref('_page.messages');

    // Запускаем реактивную функцию, она будет срабатывать при изменении messages
    // и записывать id-шки всех user-ов в _page.userIds

    model.start('_page.userIds', 'messages', 'pluckUserIds');

    var users = model.query('users', '_page.userIds');

    model.subscribe(users, function(){
      // ...
      page.render();
    });
  });
});

// Реактивные функции необходимо регистрировать после того,
// как модель создана
app.on('model', function(model){
  model.fn('pluckUserIds', function (messages) {
    var ids = {};

    for (var key in messages) ids[messages[key].userId] = true;

    return Object.keys(ids);
  });
});
```

---
#### Как получить не сами элементы коллекции, а только их id?
```js
  var query = model.query('topics');

  model.subscribe(query, function(){
    query.refIds('_page.topicIds');
  });
```
---
  Но необходимо учитывать, что сама коллекция topics в данном случае будет копироваться в браузер, чтобы этого избежать используйте проекции. В серверной части derby, в server.js определите проекцию для коллекции topics:
```
    backend.addProjection("topicIds", "topics", {id: true});
```
  Далее с проекцией можно работать, как с обычной коллекцией.
```js
  var query = model.query('topicsIds');

  model.subscribe(query, function(){
    query.refIds('_page.topicIds');
  });
```

---
#### Можно ли в derby делать distinct запросы к базе mongodb

Да, пример:
```js
  // В коллекции items отбираем элементы с типом: public
  // и получаем все уникальные имена
  var query = model.query('items', {
    $distinct: true,
    $field: 'name',
    $query: {
      type: 'public'
    }
  })

  model.subscribe(query, function(){
    query.refExtra('_page.uniqueNames');
  });
```

---
#### Можно ли в derby делать mapReduce запросы к базе mongodb

Да, но предварительно эту возможность нужно включить на сервере:

```js
var liveDbMongo = require('livedb-mongo');
var derby = require('derby');

var store = derby.createStore({
  db: liveDbMongo(process.env.MONGO_URL + '?auto_reconnect', {
    safe: true,
    allowJSQueries: true // Вот здесь
  }),
  redis: redisClient
});
```
Далее:

```js
  // Требования к функциям map и reduce полностью согласуются
  // с документацией mongo
  var query = model.query('items', {
    $mapReduce: true,
    $map: function(){
      emit this.player, this.score
    },
    $reduce: function(key, values){
      values.reduce(function(a, b) {
        return a + b;
      });
    },
    $query: {
      type: 'public'
    }
  })

  model.subscribe(query, function(){
    query.refExtra('_page.scores');
  });
```
---
#### Можно ли в derby делать aggregate запросы к базе mongodb

Да, но предварительно эту возможность нужно включить на сервере:

```js
var liveDbMongo = require('livedb-mongo');
var derby = require('derby');

var store = derby.createStore({
  db: liveDbMongo(process.env.MONGO_URL + '?auto_reconnect', {
    safe: true,
    allowAggregateQueries: true // Вот здесь
  }),
  redis: redisClient
});
```
Далее:

```js
  // Внутри aggregate запрос абсолютно соответствует документации
  // к mongoDb
  var query = model.query('items', {
    $aggregate: [{
      $group: {
        _id: '$y',
        count: {$sum: 1}
      }
    }, {
      $sort: {count: 1}
    }]
  })

  model.subscribe(query, function(){
    query.refExtra('_page.results');
  });
```

---
## Компоненты

#### Как в шаблонах компонент получить доступ к корневой области видимости модели?

Как известно, в компонентах своя, изолированная область видимости, поэтому для доступа к корню необходимо использовать префикс #root, например:
```html
<ul>
  {{each #root._page.topics as #topic}}
    <!-- ... -->
  {{/}}
</ul>
```
---
#### Как в коде компонент получить доступ к корневой области видимости модели?

Как известно, в компонентах своя, изолированная область видимости, поэтому, чтобы обратиться к корневой модели, вместо model, здесь необходимо использовать model.root либо доставать корневую модель из app. Например:
```js
function MyComponent() {}

MyComponent.prototype.init = function(model){
  // model.get('_page.topics') работать не будет
  var topics = model.root.get('_page.topics');
  // ...

}

MyComponent.prototype.onClick = function(event, element){
  var topics = this.model.root.get('_page.topics');
  // То же самое можно сделать так:
  var topics = this.app.model.get('_page.topics');
  // ...

}
```
---
#### Как в компоненте связать результаты запроса с локальным (для компонента) путем в модели?

```javascript
app.get('/', function(page) {
  page.render('home');
});

function Home() {}

app.component('home', Home);

Home.prototype.view = __dirname + '/home.html';
Home.prototype.create = function(model) {
  // var $query = model.query('somedata', {});  Так создание ссылок не работает
  var $query = model.root.query('somedata', {}); // a так работает
  model.subscribe($query, function() {
      model.ref('items', $query);
      // теперь в шаблоне компонента можно использовать локальный путь items
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

Стоить отметить, что запросы внутри компоненты - антипаттерн. Все подписки на данные должны выполняться в контроллере (при обработке роута), компоненты предназначены только для отображения данных и работы с ними. Получить все данные за раз в контроллере - это эффективно, получать данные для каждого компонента отдельно - очень неэффективно, особенно в случае множества компонент.

---
#### Как в компоненте запускать реактивные функции?
В компонентах для этого идеально подходит обработчик `init` - так как он срабатывает и на сервере и на клиенте (при серверном рендеринге), и на клиенте (при клиентском рендеринге). Это дает нам возможность прислать реактивную функцию непосредственно в колбеке фунции `start`, и не пользоваться серилизацией: `model.fn`

Вот пара примеров:

```js
// count и items - приватные пути компоненты
Comp.prototype.init = function(){
  this.model.start('count', 'items', function(items){
    return Object.keys(items).length;
  });
}
```
```js
// count - приватный путь компоненты, items - глобальный путь
Comp.prototype.init = function(){

  // вместо пути можно передавать scope, а scope берет items из корня - #root.items
  this.model.start('count', this.model.scope('items'), function(items){
    return Object.keys(items).length;
  });
}
```

---
#### Как лучше всего подписываться на события модели в компонентах?

Если делать `on` и `once` используя обычную модель, то необходимо будет отписывать при `destroy` компонента.
```javascript
  MyComponent.prototype.init = function(model){
    var this.handler = model.root.on('change', 'chats.*', function(){ ... });
    this.on('destroy', function(){
      this.model.removeListener('change', this.handler);
    });
  }

```
Если делать подписку через scoped-модель, компонента при уничтожении сама позаботиться о подписках:

```javascript
  MyComponent.prototype.init = function(model){
    model.scope('chats').on('change', function(){

    });
  }

```
---
#### Есть ли возможность давая псевдонимы компонентам через as вынести их в единую глобальную область видимости?

Да, это можно сделать вкладывая их в `page`

Допустим у нас есть компонетны A, B и С

```js
  app.component('a', A);
  app.component('b', B);
  app.component('c', C);

  function A(){};
  function B(){};
  function C(){};

```

index.html
```html
<Body:>
  <view is="a" as="page.a"/>

<a:>
  <view is="b" as="page.b"/>
  It's A

<b:>
  <view is="c" as="page.c"/>
  It's B

<c:>
  It's C

```

Теперь в коде любой из компонет A, B или С доступны все эти 3 комопненты, несмотря на иерархию.
```js
  A.prototype.show = function(){
    // ...
  };

  C.prototype.onClick = function(){
    this.page.A.show(); // Самое важное здесь
  };

```
Поятно, что если бы именование производилось не через 'page', то в компоненте A была бы доступна только компонента B, а в B - компонента С.

---
## Модель

#### Мне не нужны все поля коллекции в браузере, как получать только определенные поля (проекцию коллекции)?

В серверной части derby-приложения прописываются все проекции:
```js
// Учитываем последние изменения в ShareDb API
backend.addProjection("topic_headers", "topics", {
  id: true,
  header: true,
  autor: true,
  createAt: true
});

backend.addProjection("users", "auth", {
  id: true,
  username: true,
  email: true
});
```
Далее с проекциями users и topic_headers в derby-приложении можно работать, как с обычными коллекциями.
```js
model.subscribe('users' function(){
  model.ref('_page.users', 'users');
  // ...
});
```
При создании проекций обратите внимание: поле id обязательно, пока поддерживается только белый список полей (перечисляем только поля, которые должны попасть в проекцию), так же поддерживается пока только возможность задавать поля первого уровня вложенности.

---
#### Что такое model.context и как его использовать?

Перставим себе такую ситуацию. В приложении на странице у нас есть список чатов и окошко с выбраным чатом. Если в спике выбрать какой-то определенный чат и кликнуть по нему - в окошке с чатом, без перезагрузки страницы (без смены url) - должен загрузиться выбранный чат. То-есть за сценой должно произойти следующее: нужно отписаться от всех подписок на данные, на которые окошко с чатом было подписано, выгрузить эти данные, и подписаться на новые. В общем случае, подписок может быть достаточно много и управлением ими могло бы превратиться в достаточно сложную задачу, но здесь нам поможет `model.context`. Он создан для решения как раз этой задачи. С его помощью мы можем, создавая подписки, привязывать их к определенному  пространству имен. Когда же нам понадобиться отписаться от всего разом - мы сможем просто указать, какое пространство имен выгрузить.

Схематично, как это будет работать в компоненте чата:
```js
  // Компонент чата подписывается на данные
  var model = this.model.context('chat');

  // Переменная model - это обычная модель компоненты,
  // только находящаяся в контексте "chat"

  var messagesQuery = model.query('messages', {
    // ..
  });

  // Подписка делается в контексте
  model.subscribe('messagesQuery', function(){
    // ...
  });
```  

Теперь посмотрим как отписаться разом от всех подписок контекста chat и выгрузить все данные:

```js  
  var model = this.model.context('chat');
  model.unload();

```

---
## База данных

#### Говорят появилась возможность обходится без redis-а, используя только mongodb, как это сделать?

В серверной части derby-приложения в момент создания объекта store необходимо прописать только mongodb-данные:
```js
  var backend = derby.createBackend({
    db: shareDbMonto(mongoUrl + '?auto_reconnect', {safe: true})
  });
```

Но стоит учесть то, что redis необходим, если вы планируете горизонтальное масштабирование (запускать несколько derby-серверов параллельно).

---
#### А что делать, если у меня уже есть готовая mongodb-база данных, ее можно использовать с derby?

Ее можно будет использовать только после небольшой обработки. Дело в том, что для разрешения конфликтов в derby используется модуль [sharejs](http://sharejs.org/), он добавляет к каждой записи в коллекции вспомогательные данные: номер версии, тип объекта (в понимании sharejs) и т.д. Так же для хранения журнала операций рядом с каждой коллекцией создается еще одна с суффиксом "ops", например, "users_ops". Так вот, этот журнал тоже требует инициализации. За нас все это уже, конечно, автоматизировано в модуле - [igor](https://github.com/share/igor)

```bash
npm install igor

coffee itsalive.coffee --db project
```
Подробней смотрите на странице [модуля](https://github.com/share/igor)

---
## Стили

#### Как в derby использовать less-стили?

Начиная с версии derby 0.6.0-alpha9, блок, отвечающий за компиляцию less-файлов в css, вынесен в отдельный модуль derby-less.

Теперь, чтобы использовать, его нужно сначала подключить к проекту:
```bash
npm install derby-less
```
далее в js-файле вашего приложения нужно прописать:
```js
// Добавляем поддержку Less
app.serverUse(module, 'derby-less');
```
Причем эта строка должна быть раньше любого вызова app.loadStyles()

Например, так:
```js
var derby = require('derby');
var app = module.exports = derby.createApp('example', __filename);

// Добавить поддержку Less (перед вызовом loadStyles)
app.serverUse(module, 'derby-less');

app.loadViews (__dirname);
app.loadStyles(__dirname);

app.get('/', function(page, model) {
  page.render();
});
```

Все, теперь вместо css-файлов можно использовать less-файлы.

---
#### Как в derby использовать stylus-стили?

Все абсолютно то же самое, что и для less, только модуль называется derby-stylus

---
## View'хи

#### Как вставить в шаблон неэкранированный html или текст?

Необходимо использовать unescaped модификатор, например:
```html
<header>
  {{topic.header}}
</header>

<!-- topic.unescapedTitle сделал только для примера, не знаю зачем такое может понадобиться -->
<article title="{{unescaped topic.unescapedTitle}}">
  {{unescaped topic.html}}
</article>
```

Учитывайте то, что это - потенциальная дыра в безопасности. Ваши данные должны быть полностью очищены от опасных тегов, данные для атрибутов должны быть экранированы. В общем, прежде чем использовать такое, убедитесь, что вы понимаете, что делаете.

---
#### Как в шаблоне определенный блок сделать нереактивным (чтобы он не обновлялся сразу при изменении данных в модели)?

Во-первых, стоит сказать о том, что если нам в приложении вообще не нужна реактивность, то вместо подписки на данные нам стоит просто запрашивать их текущее состояние - вместо model.subscribe делать model.fetch.
```js
  // Так данные будут реактивно обновляться
  model.subscribe('topics', function(){
    // ...
  });

  // А так не будут
  model.fetch('topics', function(){
    // ...
  });
```
Важно понимать, что здесь мы пока говорим только о модели и только об обновлениях, приходящих с сервера. Важно, что сделав fetch, если мы что-то добавили в коллекцию, использовав model.add - наши данные вполне себе попадут на сервер в б.д., с другой стороны, если данные в коллекцию добавил кто-то другой - они к нам не придут.

Теперь поговорим о реактивности html-шаблонов, по умолчанию, все привязки к данным там реактивные, то есть, как только поменялись данные в модели (не важно пришли ли обновления с сервера, или модель изменена кодом), изменения сразу же отразятся в html-шаблоне, но этим можно управлять.

Для управления реактивностью в шаблонах, используются такие зарезервированные слова, как bound и unbound, их можно использовать как в блочной нотации, так и в виде модификатора для выражений. Продемонстрирую:

```html
<p>
  <!-- по умолчанию привязка рекативная-->
  {{_page.text}}
</p>
<p>
  <!-- явно заданная реактивная привязка-->
  {{bound _page.text}}
</p>

<!-- Внутри этого блока все привязки по-умолчанию реактивные -->
{{bound}}
  <p>
    <!-- прявязка реактивная, так как лежит внутри bound-блока-->
    {{_page.text}}
  </p>
  <p>
    <!-- прявязка не реактивная, так как это указано явно-->
    {{unbound _page.text2}}
  </p>
{{/}}

{{unbound}}
  <p>
    <!-- прявязка не реактивная, так как лежит внутри unbound-блока-->
    {{_page.text}}
  </p>

  <p>
    <!-- прявязка реактивная, так как это указано явно-->
    {{bound _page.text}}
  </p>
{{/}}

```

Естественно, для удобства, блоки могут быть вложены один в другой.

---
#### Как в шаблоне сделать так, чтобы в определенный момент обновился нереактивный блок?

Для того, чтобы в определенный момент перерисовать unbound-блок, нужно использовать ключевое слово on, примерно так:

```html
{{on _page.trigger}}
  {{unbound}}
    <!-- нереактивный html -->
  {{/}}
{{/}}

<!-- кнопка, по которой будем все это обновлять -->
<a href="#" on-click="refresh()">Refresh</a>
```
При нажатии на кнопку изменяем _page.trigger:
```js
app.proto.refresh = function(){
  app.model.set('_page.trigger', !app.model.get('_page.trigger'))
}
```
---
#### Как привязать реактивную переменную к элементу select?

Никаких событий ловить не понадобится, derby все делает за нас. Изучите пример:

```html
<!-- в массиве _page.filters находятся объекты с полями id и name - идентификатор-->
<!-- соответствующего фильтра и имя -->

<select>
  {{each #root._page.filters as #filter}}
      <option selected="{{#root._page.filterId === #filter.id}}">
        {{#filter.name}}
      </option>
  {{/}}
</select>
```

В результате выбора в _page.filterId будет лежать id выбранного по имени фильтра.

При реактивной привязке к option-у, derby предполагает, что встретит в атрибуте selected выражение проверки на равенство (сами равенства нужны для первоначальной установки занчения). Пердполагается, что левым параметром проверки на равенство будет путь, который обновится значением из правого параметра проверки (который будет разным на каждой итерации цикла each) соответствующего option-а, при его выборе пользователем.

---
#### Как привязать реактивную переменную к элементу input type=radio?

Никаких событий ловить не понадобится, derby все делает за нас:

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

В результате выбора в _page.radioVal будет либо 'one', либо 'two', либо 'three', в зависимости от того, что выберет пользователь.


При реактивной привязке к input-у c типом radio, derby предполагает, что встретит в атрибуте checked выражение проверки на равенство (сами равенства нужны для первоначальной установки занчения). Пердполагается, что левым параметром проверки на равенство будет путь, который обновится значением из атрибута value соответствующего input-а, при его выборе пользователем.---

#### Как привязать реактивную переменную к элементу textarea?

Все очень просто:

так было до 0.6.0-alpha6
```html
 <textarea value="{{message}}"></textarea>
```
так стало, начиная от 0.6.0-alpha6
```html
 <textarea>{{message}}</textarea>
```

---

#### Как связать view-функцию с каким-либо html-инпутом? (Как писать`get/set` view-функции?)

Допустим нам нужно написать функцию, которая преобразует значения RGB и опциональный альфа-канал в hex-значение цвета (альфа канал сдвигает цвет ближе к белому). Но нам нужно, чтобы преобразование работало в обе стороны, для того чтобы связать его с input-ом - то-есть, чтобы если пользователь ввел в инпуте hex-начение цвета, мы могли автоматом преобразовать их в RGB (с альфаканалом равным 1).

```html
<input type='text' value='{{ hexify(menuColor.r, menuColor.g, menuColor.b, menuColor.alpha) }}'>
```

get/set view-функция создается из объекта с методами `set` и `get`:

```js
app.proto.hexify = {
  get: function(r, g, b, alpha){
    var color = convertRgbToHex(r, g, b);
    // Осветляем цвет, если альфа-канал задан
    color = lightenColor(hex, 1 - alpha);
    return color;
  },
  // Первый аргумент в 'set'-функции это 'inputValue' -- каждый раз, когда вы вводите
  // что-либо в инпуте, 'set'-функция будет вызвана с этим новым значением, в качестве
  // первого параметра
  //
  // ВНИМАНИЕ! 'inputValue' не учитывается при расчете номера параметра для возврата
  // результатов из 'set'-функции (об этом далее)
  set: function(inputValue, r, g, b, alpha){
    var color = convertHexToRgb(inputValue);
    // 'set'-функция возвращает массив значений, которые будут записаны в пути, указанные
    // в качестве аргументов view-функции.
    // В нашем случае - это 'menuColor.r', 'menuColor.g', 'menuColor.b', 'menuColor.alpha'.
    // Порядок будет соответствовать тому порядку, который был при вызове, счет начинается
    // с 0 (ВНИМАНИЕ! Ориентируйтесь на 'get'-функцию, так как в 'set'-функции есть еще
    // параметр 'inputValue', который не учитывается).
    return [color.r, color.g, color.b, 1];
  }
};
```

Несколько тонкостей:

1. Если вы хотите установить только 3 первых параметра из 4-х, просто верните их в массиве:

    ```js
    return [color.r, color.g, color.b];
    ```

2. Если нужно вернуть только последний параметр, можно вернуть объект, где индексом будет номен возвращаемого параметра. Так для возврата тольо `menuColor.alpha` пишем так:
    would do:

    ```js
    return {3: color.alpha};
    ```

    Можно, конечно, использовать объектную нотацию и для возврата всех значений:

    ```js
    return {0: color.r, 1: color.g, 2: color.b, 3: color.alpha};
    ```

    но использование массивов предпочтительней, когда индексы начинаются с 0 идут последовательно.

3. Даже когда у вас всего 1 аргумент в важей view-функции, для его установки, его необходимо оборачивать в массив:

    ```js
    return [color.a];
    ```

4. (Считается ПЛОХОЙ практикой) Если вы используете 'set'-функцию, для установки путей, не связанных с тем, что происходит в 'get'-функции. Это можно сделать просто передав дополнительный атрибут во view-функцию, напримет:

    ```html
    <input type='number' value='{{ formatNumber(price, priceIsModified) }}'>
    ```

    ```js
    app.proto.formatNumber = {
      get: function(number){
        // округлим до двух знаков при выводе
        return ~~((number || 0) * 100) / 100
      },
      set: function(number, modificationFlag){
        number = parseFloat(number) || undefined;
        return [number, true];
      }
    };
    ```
5. 'Set'-функция для  select'a запускается для каждого option'a со значением true (выбранный option) либо false (все остальные option'ы), поэтому надо игнорировать все те значения которые false:

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

## События

#### Как получить доступ к объекту event в обработчике событий?

Необходимо в функцию-обработчик события передать дополнительный, предопределенный параметр $event
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
#### Как получить доступ к dom-элементу в обработчике событий?

Необходимо в функцию-обработчик события передать дополнительный, предопределенный параметр $element
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
#### Есть ли в derby 0.6 обработчик, аналогичный app.ready в derby 0.5, срабатывющий всего один раз в браузере, в самом начале, нужный для инициализации клиентских скриптов?

Этот обработчик теперь находтся в app.proto.create. Проиллюстрирую:

index.js
```js
// ...

// Вызывается в браузере после рендеринга первой страницы
// (вызовется всего один раз)
app.proto.create = function(model){
  // Инициализируем jQuery AJAX
  $.ajaxSetup({
    dataType: 'json',
    contentType: 'application/json; charset=utf-8',
    processData: false
  });
}
```

---
#### В derby 0.5 был обработчик app.enter, в котором можно было отловить момент создания страницы, а как это сделать в derby 0.6?

Чтобы отлавливать момент создания страницы (когда dom уже сформирован) необходимо, чтобы каждая страница была компонентом. В компонете для этого используется метод create.

В полном виде подход выглядет примерно так:

index.html
```html
<!-- точка входа - это наш лейаут -->
<import: src="./home">
<import: src="./about">

<Body:>
  <!-- здесь будет header -->

  <!-- сюда будет генерится контент страниц -->
  <view is="{{$render.ns}}"/>

  <!-- здесь будет footer -->
```
home.html - страница home будет отдельным компонентом
```html
<index:>
  <h1> Home </h1>
  <!-- ... -->
```

about.html - страница about будет отдельным компонентом
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
  // страница отрендерена
}

About.prototype.create = function(model, dom){
  // страница отрендерена
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
#### Как в компоненте подписаться на событие не всего document'a, а необходимого контрола?

Для этого необходимо указать параметр target (в примере это self.input) при подписывании на событие, например:

View:
```html
<index: tag='demo'>
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

## Модули

#### Как к derby подключить шаблонизатор jade?
Для подключения jade к дерби необходимо использовать модуль [derby-jade](https://github.com/cray0000/derby-jade).

Устанавливаем его:
```bash
npm install derby-jade
```
В derby-приложении до использования app.loadViews() необходимо подключить этот модуль вот таким образом:
```js
app.serverUse(module, 'derby-jade');
```
Убедитесь, что у вас derby версии не младше 0.6.0-alpha7

---
#### Какой модуль использовать для авторизации в derby?

Используйте, надавно созданный специально под 0.6 версию derby, модуль [derby-login](https://github.com/vmakhaev/derby-login).

Доступна регистрация/авторизация через соц. сети, ведь внутри этот модуль использует [passportjs](passportjs.org).

---
#### Как подключать клиентские скрипты к derby-приложению, например, jquery?

Если скрипт находится где-то на cdn-сервере, можно подключить его, просто вставив в html-шаблон тег script, например:

```html
<Body:>
  <!-- ... -->
  <script src="//code.jquery.com/jquery-1.11.0.min.js"></script>
```
Можно, конечно, сюда засунуть и сам скрипт, но лучше воспользоваться для этого предоставляемым Browserify интерфейсом. Этот способ хорош еще и тем, что скрипт попадает в "бандл", который к странице подтягивается вторым запросом (после самого html-я страницы со встроенными стилями), что дает высокую скорость отрисовки.

Итак, рекомендуемый способ:

```js
// на серверной стороне derby? в файле server.js

backend.on('bundle', function(browserify){
  // ваш локальный путь (от корня проекта) до файла скрипта
  browserify.add("../js/minified/jquery-1.11.0.min.js");
});
```

У себя для подключения клиентских скриптов, мы бычно используем bower, далее в grunt-e у нас настроены задачи по минификации и конкатенации всех вендорских скриптов в один. Итоговый vendor.min.js мы подключаем, используя вышеизложенный метод.

---
## Написание модулей

#### Как проверить, что код в derby-приложении выполняется на клиенте/на сервере?

В derby есть модуль с утилитами, в нем присутствуют нужные нам флаги:
```js
var derby = require('derby');

if (derby.util.isServer) {
  // код, который должен выполняться,
  // только если мы находимся на сервере
}

if (!derby.util.isServer) {
  // код, который должен выполняться,
  // только если мы находимся на клиенте
}
```

---
#### Как сделать require модуля в derby-приложении, код которого будет выполняться только на сервере?

Объясню сначала в чем вообще особенность require модулей, которые нужны нам только для сервера. Дело в том, что все модули derby-приложения, которые подключаются к нему через классическое CommonJS-ное require полюбому попадают в "бандл", а следовательно будут копироваться в браузер к клинету. Нам же не особо хочется, чтобы в браузер попадали лишние данные, тем более, если модуль заведомо нужен только для работы на сервере. Поэтому вместо require используем serverRequire из набора утилит derby:

```js
var derby = require('derby');

if (derby.util.isServer) {
  // Пакет точно не попадет к клиентам в браузер
  var myPackage = derby.util.serverRequire(module, 'mypackage');
  // ...
}
```

---
#### Как проверить, что приложение выполняется в режиме промышленной эксплуатации (isProduction)?

И на клиенте, и на сервере доступен флаг derby.util.isProduction. Если заглянуть внутрь util мы увидим, что флаг этот устанавливается стандартным для nodejs способом.

```js
  var isProduction = process.env.NODE_ENV === 'production';
```

Для этого используется переменная окружения NODE_ENV - установите ее, если нужно.

Пример использования (можно и на клиенте и на сервере):

```js
  if (derby.util.isProduction) {
    // ...
  } else {
    // ...
  }
```

---
## Взаимодействие клиента с сервером

#### Можно ли на cервере отловить подключение/отключение клиентов?

Да можно, для этого на сервере:
```js
// Срабатывает при каждом новом подключении клиентов
backend.on('client', function(client){
  console.log('Подключился клиент:', client.id);

  // Событие close происходит при отключении клиента
  // (потеря связи / закрытие страницы)
  client.on('close', function(reason){
    console.log('Отключился клиент:', client.id);
  });
});
```

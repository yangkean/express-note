# [Express](http://expressjs.com/)

Table of Contents

* [路由](#路由-routing)
* [中间件](#中间件-middleware)
* [使用模版引擎](#使用模版引擎)
* [DEBUG](#debug)
* [数据库整合](#数据库整合-database-integration)
* [API Reference](#api-reference)
* [express 的官方使用示例](#express-的官方使用示例)

## 路由 (*routing*)

**路由**是指一个应用程序为不同的客户端请求指定不同的处理方法，客户端请求包括请求路径和 HTTP 请求方法。

```javascript
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send('hello world!');
});

app.post('/', (req, res) => {
  res.send('Got a POST request!');
});
```

上面的代码中，`get`和`post`是 HTTP 请求方法， 之后的第一个参数`/`是请求路径，第二个参数是一个回调函数，当用相应的请求方法访问相应的路径时，就会调用这个回调函数来处理相应的逻辑。

### 路由方法 (*route methods*)

除了常用的`get`、`post`以及其他的 HTTP 请求方法外，还有一个特殊的路由方法：`all`，使用方法如下。

```javascript
/* ... */

app.all('/', (req, res, next) => {
  console.log('Start services...');
  next();
});
```

`app.all(path, handler)`方法用于请求相应的路径时，不管 HTTP 请求方法是什么都会调用其回调函数。比如上面的代码中，当请求根路径 (/) 时，不管是用`post`、`get`还是`put`等方法，都会调用其后的回调函数，然后用`next()`把控制权传给下一个控制器函数。

### 路由路径 (*route paths*)

即前面所讲的请求路径，可以是字符串、字符串模式和正则表达式。字符串模式中`?`、`+`、`*`分别表示匹配前一个字符 0 或 1 次、匹配前一个字符 1 或无限次、匹配任意字符 (包括无字符的情况)。`()`则可以表示字符的组合。连字符`-`和点字符`.`都会被直接解释为路由路径中的字面量。路径字符串需要含有`$`符号时可以写为`([\$])`。

对于正则表达式的请求路径，则是进行正则匹配，如下：

```javascript
/* ... */

// 会匹配含有 “blog” 的路径
app.get(/blog/, (req, res) => {
  res.send('hello');
});
```

当路由路径中的某个位置字符不同但表示的含义本质是一样的时候，可以用路由参数替代，如下，用`userId`和`bookId`分别表示了不同的用户和不同的书。

```
Route path: /users/:userId/books/:bookId
Request url: http://localhost:3000/users/34/books/8989、http://localhost:3000/users/35/books/8977
```
这些路由参数都被保存在`req.params`对象中，它们的健名即是各自的路由参数名，可以像下面这样获取路由参数的实际值。

```javascript
/* ... */

app.get('/users/:userId/books/:bookId', (req, res) => {
  res.send(`hello, ${req.params.userId}! You have bought book ${req.params.bookId}.`);
});
```
注意：路由参数的范围为[0-9a-zA-Z_]。

### 路由处理器 (*route handlers*)

模式`app.method(path, handlers)`中，`handlers`即是路由处理器，它一般是一个函数，也可以是包含多个函数的数组或前两者的结合。当使用数组或者两者的结合时应增加回调函数的第三个参数`next`并在每个回调函数的末尾调用`next()`继续执行下一个回调函数。

### 响应方法 (*response methods*)

路由处理器中的`res`是响应对象，包含了一些响应客户端请求的方法。路由处理器中需要调用其中任意一个方法，否则客户端请求会保持挂起的状态。常用方法有`res.end()`、`res.redirect()`、`res.render()`、`res.send()`。其他可见[Response methods](http://expressjs.com/en/guide/routing.html#response-methods)。

#### res.end([data][, encoding])

结束响应，一般用于无需数据地快速地结束响应。

#### res.redirect([status, ]path)

重定向到一个特定的 URL 路径，这个路径可以是相对当前网站的路径或者一个不同网站的地址，同时可以用`status`指定 HTTP 状态码。示例如下：

```javascript
res.redirect('/foo/bar');
res.redirect('http://example.com');
res.redirect(301, 'http://example.com');
res.redirect('../login');
res.redirect('http://google.com');
res.redirect('back');
```

上面的`res.redirect('back')`用于返回当前页面的上一个页面，准确地说是返回到 **referer**，这是一个 HTTP 表头的字段，表示从哪儿链接到当前的页面。

#### res.send([body])

发送 HTTP 响应。`body`可以是普通对象、支付串、数组等。这个方法同时也会自动设置`Content-Length`和`Content-Type`。示例如下：

```javascript
res.send(new Buffer('whoop'));
res.send({ some: 'json' });
res.send('<p>some html</p>');
res.status(404).send('Sorry, we cannot find that!');
res.status(500).send({ error: 'something blew up' });
```

#### res.render(view[, locals][, callback])

渲染模版并把渲染好的 HTML 文档发送到客户端。`view`是一个等待渲染的模版文件路径名。`locals`是一个对象，它的属性定义了模版文件的本地变量。如果提供了`callback`回调函数，这个回调会得到可能的错误和渲染好的 HTML 文档，但需要在回调内部手动发送响应。示例如下：

```javascript
res.render('signup', {title: 'my blog'});

res.render('signup', {title: 'my blog'}, (err, html) => {
  res.send(html);
});
```

### 链式调用路由处理器函数

同一个路径因为请求方法不同而写了多个路由显得有点多余，这时可以用`app.route()`链式调用这些路径相同的处理器函数。示例如下：

```javascript
app.route('/book')
  .get((req, res) => {
    res.send('Get a random book')
  })
  .post((req, res) => {
    res.send('Add a book')
  })
  .put((req, res) => {
    res.send('Update the book')
  });
```

### express.Router

这个类的实例是一个缩小版的 **express app**，可以用来注册中间件 (*middleware*) 和挂载路由 (*route*)，然后可以把这个实例构建的路由器模块挂载到主 **express app** 上供主 **app** 使用。示例如下：

```javascript
// birds.js

const express = require('express');
const router = express.Router();

// middleware that is specific to this router
router.use(function timeLog (req, res, next) {
  console.log('Time: ', Date.now());
  next();
});

// define the home page route
router.get('/', function (req, res) {
  res.send('Birds home page');
});

// define the about route
router.get('/about', function (req, res) {
  res.send('About birds');
})

module.exports = router;
```

```javascript
// index.js

var birds = require('./birds');

// ...

app.use('/birds', birds);
```

## 中间件 (*middleware*)

中间件就相当于一个底层和应用层之间的中间人，每一个中间件都完成一个基础功能，你只需要在应用层逐个调用这些中间件，就可以完成一个复杂的功能。对于不同基础功能的封装可以对主模块和基础功能进行解耦，让开发者更加专注于主模块的编写。*Express* 中可以用`app.use([path, ]callback[,  callback...])`注册中间件，此处的`callback`就是是中间件函数 (可以有多个)，接受`req`、`res`、`next`三个参数，可操作`req`和`res`对象，当当前中间件函数执行完毕后，如有下一个中间件，可以调用`next()`执行下一个中间件。另外，`app.METHOD(path, handler)`中`handler`也相当于是中间件，不过这个中间件执行完后当前的**请求-响应**循环就结束了。中间件是按顺序执行的，所以中间件的加载顺序十分重要，需要格外注意。示例如下：

```javascript
const express = require('express');
const app = express();

const myLogger = (req, res, next) => {
  console.log('LOGGED');
  next();
};

app.use(myLogger);

app.get('/', (req, res) => {
  res.send('Hello World!');
});

app.listen(3000);
```

### 应用级别中间件 (*Application-level middleware*)

这类中间件是直接通过`app.use()`或者`app.METHOD()`绑定在`express app`实例上的中间件函数。示例如下：

```javascript
var app = express();

app.use((req, res, next) => {
  console.log('Time:', Date.now());
  next();
});
```

`app.use()`可以一次注册多个中间件函数，可以调用`next('route')`跳过剩余的中间件函数，执行下一个路由，但是`next('route')`只在`app.METHOD()`或者`router.METHOD()`中调用才起作用。示例如下：

```javascript
app.get('/user/:id', (req, res, next) => {
  // if the user ID is 0, skip to the next route
  if (req.params.id === '0') next('route');
  // otherwise pass the control to the next middleware function in this stack
  else next();
}, (req, res, next) => {
  // render a regular page
  res.render('regular');
});

// handler for the /user/:id path, which renders a special page
app.get('/user/:id', (req, res, next) => {
  res.render('special');
});
```

### 路由级别中间件 (*Router-level middleware*)

路由级别的中间件使用方法和应用级别的中间件使用方法相同，区别在于中间件是绑定到`express.Router()`实例上的，并使用`router.use()`或者`router.METHOD()`方法注册中间件函数，然后可以把路由模块挂载到主`express app`上。示例如下：

```javascript
const express = require('express');
const app = express();
const router = express.Router();

router.use((req, res, next) => {
  console.log('Time:', Date.now());
  next();
});

router.get('/user/:id', (req, res, next) => {
  console.log(req.params.id);
  res.render('special');
});

app.use('/', router);
```

### 错误处理中间件 (*Error-handling middleware*)

使用`app.use()`注册错误处理中间件，需要格外注意的是错误处理中间件函数接受的参数必须是 4 个，即`(err, req, res, next)`。示例如下：

```javascript
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).send('Something broke!');
});
```

### 内建中间件 (*Built-in middleware*)

从 **express 4.x** 开始，内建的中间件只剩下了`express.static(root, [options])`，这个中间件是为静态资源文件服务的，`root`是一个字符串，用来指定静态资源文件的根目录，**express** 会相对这个目录寻找静态资源文件，所以这个目录名是不包含在请求 URL 中的。`options`是一个包含选项的对象。示例如下：

```javascript
app.use(express.static('public'));
```

可以这样找到静态资源文件：

```
http://localhost:3000/images/kitten.jpg
http://localhost:3000/css/style.css
http://localhost:3000/js/app.js
http://localhost:3000/images/bg.png
http://localhost:3000/hello.html
```

值得注意的是，当提供给`express.static(root, [options])`的是相对路径时，这个相对路径是相对你启动 node 进程的位置的，如果你在其他目录中启动了你的 *express app*，就无法正确找到静态资源文件了，这个时候，你应该传递静态资源根目录的绝对路径给`express.static(root, [options])`，如下：

```javascript
app.use(express.static(path.join(__dirname, 'public')));
```

### 第三方中间件 (*Third-party middleware*)

手动安装第三方编写的中间件并注册使用。*express* 官方自己的第三方中间件见[Express middleware](http://expressjs.com/en/resources/middleware.html)。 示例如下：

```
$ npm install cookie-parser --save
```

```javascript
var express = require('express');
var app = express();
var cookieParser = require('cookie-parser');

// load the cookie-parsing middleware
app.use(cookieParser());
```

## 使用模版引擎

使用模版引擎之前需要用`app.set(name, value)`做两个设置。

```javascript
app.set('views', path.join(__dirname, 'views')); // 设置模版文件所在的路径
app.set('view engine', 'ejs'); // 设置解析模版文件的模版引擎
```

然后安装这个包：

```
$ npm install ejs --save
```

假设已有一个写好的模版文件`index.ejs`在`views`目录中，可以这样使用：

```javascript
app.get('/', function (req, res) {
  res.render('index', { title: 'Hey', message: 'Hello there!' });
});
```

上面的代码中，如果先前指定了模版引擎，可以省略`res.render()`中模版文件的后缀，否则应加上后缀。

## DEBUG

可以用下面这条指令记录应用运行情况：

```
$ DEBUG=express:* node index.js
```

详情可见[Debugging Express](http://expressjs.com/en/guide/debugging.html)。

## 数据库整合 (*Database integration*)

这里只说怎么连接 MySQL 和 MongoDB 数据库。

[MySQL](https://github.com/mysqljs/mysql?_ga=1.190460972.665337408.1491205614)是这样的：

```
$ npm install mysql --save
```

```javascript
const mysql = require('mysql');
const connection = mysql.createConnection({
  host     : 'localhost',
  user     : 'me',
  password : 'secret',
  database : 'my_db',
});

connection.connect();

connection.query('SELECT 1 + 1 AS solution', function (error, results, fields) {
  if (error) throw error;
  console.log('The solution is: ', results[0].solution);
});

connection.end();
```

MongoDB 采用其对象模型驱动器 (*object model driver*) [mongoose](http://mongoosejs.com/)进行连接，是这样的：

```
$ npm install mongoose --save
```

```javascript
const mongoose = require('mongoose');
mongoose.connect('mongodb://localhost/test');

let Cat = mongoose.model('Cat', { name: String });

let kitty = new Cat({ name: 'Zildjian' });

kitty.save(function (err) {
  if (err) {
    console.log(err);
  }
  else {
    console.log('meow');
  }
});
```

## API Reference

官方 API 参考见[API Reference 4.x](http://expressjs.com/en/4x/api.html)。

## express 的官方使用示例

详情见[express examples](https://github.com/expressjs/express/tree/master/examples)。

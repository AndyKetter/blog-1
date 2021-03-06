title: 在 Express 项目中使用 Waterline
date: 2015-10-26 01:40:22
tags:
  - Node.js
  - Waterline
categories:
  - 编程杂记
---

在上一篇[Node.js ORM 数据操作中间件 Waterline ](http://chensd.com/2015-10/Node-ORM-Waterline.html)中，我们介绍了 Waterline 的功能与特点，这篇文章中我们将用一个实例 waterline-sample （[Github](https://github.com/stiekel/waterline-sample)、[Coding.NET](https://coding.net/u/Stiekel/p/waterline-sample/git)），来看看在 Express 项目中如何使用 Waterline。

要在项目中使用 Waterline ，无非是解决如何配置，在什么时机初始化，怎样组织所有的数据集合，以及在控制器中怎么调用 Waterline 中的数据集合这几个问题。
<!--more-->

### 项目的功能与结构

这个示例项目中，将会实现两个关于 Post 的 API ，添加和获取列表的接口，功能比较简单，但代码的组织，依然遵循模块化和 MVC 的设计原则，主要的文件和目录如下：

```sh
|-- app
|   |-- controllers
|   |   `-- post.server.controller.js
|   |-- models
|   |   `-- post.server.model.js
|   `-- routes
|       `-- post.server.routes.js
|-- config
|   |-- config.js
|   |-- env
|   |   `-- development.js
|   |-- express.js
|   `-- waterline.js
|-- app.js
|-- bin
|   `-- www
`-- package.json

```

其中， `config` 目录存储基本配置、 Express 配置和 Waterline 的配置文件，其中的 `env` 目录存储根据环境而不同的基本配置。 `app` 目录下的三个文件夹，分别按 MVC 的结构组织 Waterline 的 Collections ，Expresss 的控制器以及路由文件。根目录下的 `app.js` 将使用 `config/express.js` 中的配置来生成 Express 的实例，而入口文件 `bin/www` 则完成 Waterline 的初始化和启动 Express 实例的端口监听。

### 数据集合的组织

所有的数据集合，全部组织在 `app/models` 目录中，直接使用 `module.exports` 来将 `Waterline.Collections` 实例导出，以便在 Waterline 的配置文件中调用。

```js
// app/models/post.server.model.js
var Waterline = require('waterline');

module.exports = Waterline.Collection.extend({
  identity: 'post',
  connection: 'mongo',
  schema: true,
  attributes: {
    title: {
      type: 'string',
      required: true
    },
    content: 'string',
    createTime: 'date',
    lastModifyTime: 'date'
  },
  beforeCreate: function(v, cb){
    v.createTime = new Date();
    return cb();
  },
  print: function(v) {
    console.log('\tTitle:', v.title, 'create at:', v.createTime);
    console.log('\tContent:', v.content);
  }
});
```

这里也演示了如何定义生命周期回调和自定义的方法。

### 配置

配置里主要是生成 Waterline 的实例，并加载上面的数据集合配置文件。下面是 Waterline 的配置文件，导出了配置所需要的代码，但并没有执行初始化。

```js
// config/waterline.js
var Waterline = require('waterline');
var mongoAdapter = require('sails-mongo');
var config = require('./config');

// models
var Post = require('../app/models/post.server.model');

var orm = new Waterline();
var wlconfig = {
  adapters: {
    'default': mongoAdapter,
    mongo: mongoAdapter
  },
  connections: {
    'mongo': {
      adapter: 'mongo',
      url: config.mongo
    }
  }
};
orm.loadCollection(Post);

exports.orm = orm;
exports.config = wlconfig;
```

这里使用的是 `exports` 来导出，因为我们有两个值需要导出，一个是 Waterline 的实例，另外一个是 Waterline 的初始化配置。这在 Waterline 初始化的时候会用到。

### 初始化

初始化是在 `bin/www` 里完成的，原因是保证 Express 启动监听，必须在 Waterline 的成功初始化之后进行。

```js
// bin/www
var app = require('../app');
var config = require('../config/config');
var waterline = require('../config/waterline');

waterline.orm.initialize(waterline.config, function(err, models){
  if(err) {
    console.log('waterline initialize failed, err:', err);
    return;
  }
  console.log('waterline initialize success.');
  
  app.set('models', models.collections);

  app.listen(config.port, function(){
    console.log('Express listening on port:', config.port);
  });
});
```

初始化是直接使用 Waterline 实例的 `intialize()` 方法，需要传入对应的配置，这两个都是在 `waterline.js` 配置文件中导出的。为了方便我们在控制器代码中调用 Waterline 的数据集合，这里先将它加入到 Express 实例的配置列表中。

### 在 Express 控制器中使用 Waterline 数据集合

由于 Waterline 的初始化过程是异步的，所以我们没有办法直接使用 `module.exports` 或 `exports` 方法来导出它的实例，也就无法直接以 JavaScript 模块化的方式调用它实例中的数据集合。这里将借助 Express 的实例，来在控制器代码中使用它。

但实际上，在控制器代码中，也是没有办法直接访问 Express 的实例的，所以这里我们在 Express 的配置里，增加一个中间件，将附加在 Express 实例上的数据集合，再加入到 Express 请求对象中，这样便可以在控制器代码中通过请求对象来访问 Waterline 实例的数据集合了。当然，加到响应对象也可以。下面的代码展示了如何在控制器中使用 Waterline 的数据集合。

```js
// config/express.js
var express = require('express');
var waterline = require('./waterline');

module.exports = function(){
  console.log('express initialing...');
  var app = express();
  // ...
  app.use(function(req, res, next){
    req.models = app.get('models');
    next();
  });

  require('../app/routes/post.server.routes')(app);
  // ...
  return app;
};
```

在控制器里，便可以通过请求对象的 `models` 成员来调用了。

```js
// app/controllers/post.server.controller.js
module.exports = {
  list: function(req, res, next){
    var page = parseInt(req.query.page, 10) ? parseInt(req.query.page, 10) : 1;
    var limit = parseInt(req.query.limit, 10) ? parseInt(req.query.limit, 10) : 1;
    req.models.post.find().paginate({page: page, limit: limit}).exec(function(err, docs){
      res.json(docs);
    });
  }
};
```

虽然也可以通过全局变量来调用数据集合，不过在有其它办法的情况下，还是尽量不要使用全局变量吧。
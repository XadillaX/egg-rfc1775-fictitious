# Egg.js RFC1775 - Model 层规划

## 目标

目前在 Egg.js 中，Model 层的编写比较混乱，没有一个较为统一的方案，各自 ORM 插件也各自为战，限制较多，多 ORM 共存时也存在较大的兼容性问题。

这里将要重新规划一下 Egg.js 的 Model 层方案，主要解决以下各问题：

1. 使用限制更少，使得多 ORM、多库等问题能得到比较好的解决，如使 MySQL 与 MongoDB 能在同一个 Egg.js 项目中共存；
2. 外层 API 风格一致性，使得各 ORM 不各自为战（不包括各 ORM 的模型对象中的 API）；
3. Model 文件递归查找，支持多级子目录形式。

## 规划构思（egg-model）

本层规划主要由一个新的插件 **egg-model** 实现。它主要分以下几个功能。

+ 引入 Connection 机制，用于注册和实例化各数据源实例（包括但不局限于各种 ORM 插件）；
+ 加入 Model 目录的 Loader 机制，自动加载 **model** 目录并载入各 Model 实例。

若要使用 Egg.js 的 Model 层特性，则需要先将 **egg-model** 安装，并在插件配置中开启它：

```js
// package.json
...
"dependencies": {
  "egg-model": "...",
  ...
},
...

// config/plugin.js
...
exports.model = {
  enable: true,
  package: "egg-model"
};
...
```

### Connection 机制

#### Connection Loader（连接源加载器）

Connection Loader 为 egg-model 中的一个子功能。主要用于配置和加载各连接源实例。关于连接源的概念请阅读下一节内容。

在应用中，首先开启 **config/plugin.js** 文件中相应连接源插件：

```js
// package.json
...
"dependencies": {
  "egg-sequelize": "...",
  ...
}

// config/plugin.js
exports.sequelize = {
  enable: true,
  package: "egg-sequelize"
};
...
```

然后在连接源配置（`connection`）中配置好实例，可多实例：

```js
// config/config.default.js
exports.sequelize = {
  // 一些 sequelize 的默认配置
  host: "127.0.0.1"
};

exports.connections = {
  main: {
    adapter: "sequelize", // adapter 字段名可再议

    // 其它配置
    username: "root",
    password: "temp"
  },

  foo: {
    adapter: "sequelize",

    // 其它配置
    host: "10.10.10.10"
  },

  bar: {
    adapter: "mongoose",

    // 其它配置
    ...
  }
};
...
```

在 Egg.js 启动时，egg-model 开始加载的时候将会遍历 `connections` 配置，并将所有配置好的连接源加载好，挂载到 `app.connection` 对象下，如：

```js
const mainConn = app.connection.main;

// 定义一个 Model
const TestModel = mainConn.define(...);
```

#### Connection（连接源）

在 Egg.js 的 Model 层概念披露后，将引入 Connection 概念，可理解为连接源，某种意义上可以理解为数据源，数据源为连接源的子集。

连接源包括但不局限于各种 ORM 插件，这些插件都有可能成为 Egg.js Model 层的连接源：

+ egg-sequelize
+ egg-mongoose
+ egg-toshihiko
+ egg-rocketmq
+ egg-socketio
+ ...

> 以上各插件若已存在于生态中，则将会发布一个 Breaking 的大版本，或者命名一个新的插件。

上面的插件中，都是 Model 的连接源，其中 egg-sequelize、egg-mongoose、egg-toshihiko 又是数据源。

每个连接源插件的连接源实例都要实现自身的一些方法，如 egg-sequelize 需要实现定义 Model 的方法 `.define()`。还有重要的一点就是，连接源插件要实现一个自身往连接源加载器注册自身成为一个 Adapter 的函数，初步预想如下：

```js
class SequelizeWrapper {
  constructor(database, username, password, options) {
    this.sequelize = new Sequelize(database, username, password, options);
  }

  define(name, columns) {
    const m = this.sequelize.define(name, columns);

    const proto = Object.keys(m);

    return class ModelWrapper {
      constructor(ctx) {
        this.model = m;
        this.ctx = ctx;

        for(key of proto) {
          if(typeof proto[key] !== "function") continue;
          this[key] = function(...) {
            this.ctx....;
            m[key](...);
          };
        }
      }
    };
  }
};

app.connection.register("sequelize", function(options) {
  const m = new SequelizeWrapper(options.database, options.username, options.password, options);
});
```

首先会有一个 `SequelizeWrapper` 基类，egg-sequelize 将自身注册进 Connection Loader 的时候工厂函数返回相应的一个 SequelizeWrapper 实例，这个实例将会在 Model 定义的时候被用到；在 Model 定义的时候，执行的是这个 SequelizeWrapper 实例的 `.define()` 函数，返回一个 `ModelWrapper`，这个类将在每次请求生命周期被实例化一次，并带有 `ctx` 信息，最后将被挂载到请求上下文中。

如上一节中看到的 `TestModel` 实际上就是一个 `ModelWrapper` 的类。

### Model Loader（模型加载器）

在应用中如果启用了 egg-model 插件，则插件在加载时期会递归遍历 **app/model/** 目录，并将其导出的类缓存在自身插件内部。在请求周期会将模型实例化到 `ctx` 中去。

#### 模型声明

所有的模型都将以类的形式存在于 **app/model/** 目录下，并且可接受 `ctx` 且只接受对象。如这将是一个合法的模型：

```js
// app/model/foo.js

class FooModel {
  constructor(ctx) {
    this.ctx = ctx;
  }

  getData() {
    return "hello world";
  }
};

module.exports = FooModel;
```

如果是一个 Sequelize 插件模型，则 egg-sequelize 可能的一个实现方式如下：

```js
// 插件中
class SequelizeWrapper {
  ...

  define(name, columns) {
    const Model = this.sequelize.define(name, columns);

    return class EggSequelizeModel {
      constructor(ctx) {
        this.inner = Model;
        this.ctx = ctx;

        // 将 this.inner 中所有函数经包装后挂载到该类下
        ...
      }
    };
  }
};

app.connection.register("sequelize", (options) => { return new SequelizeWrapper(options); });

// app/model/seq.js
const sequelize = app.connection.main; // 一个名为 main 的 SequelizeWrapper 实例

const Seq = sequelize.define("seq", { ... });

// 这里的 Seq 就是一个 EggSequelizeModel
Seq.prototype.findByDate = function* (date, limit) {
  // ...
};

module.exports = Seq;

// 或者
const Base = sequelize.define("seq", { ... });

class Seq extends Base {
  constructor(ctx) {
    super(ctx);
  }

  findByDate(date, limite) {
    // ...
  }
};

module.exports = Seq;
```

#### 模型获取及使用

在请求生命周期中，所有模型会以目录的形式被挂载到 `ctx.model` 对象中去。

如这是几个文件名与 `ctx.model` 模型对象的一个映射举例：

```js
ctx.model.Foo;      // app/model/foo.js
ctx.model.SeqFoo;   // app/model/seq_foo.js 或者 app/model/seqFoo.js
ctx.model.Seq.Foo;  // app/model/seq/foo.js
```

所以在日常使用中的例子如下：

```js
const Foo = ctx.model.Foo;
const ret = yield Foo.findByDate(date, limit);
```

### 注意事项

Connection 与 Model 都存在于 egg-model 插件中，但它们两个是独立的概念，只是交集的关系。

+ Connection 可以不在 Model 中使用。开发者可以在一个自己想要的地方（如 **lib/** 目录下）获取一个 RabbitMQ 的 Connection 并开始监听；
+ Connection 实例中的 `.define()` 函数并不是强制的，鉴于各 ORM 的 API 本身就不同，而开发者在使用的时候自己当前在用什么 ORM，所以只需要看文档就可以。如 Sequelize 中可以叫 `.define()`，而另一个不知道什么 ORM 也许也可以叫 `.createModel()`；
+ Model 不一定要由 Connection 创建，正如前文所述，一个放在 **app/model** 目录下并且能接受 `ctx` 的类就能是一个 Model；
+ 通常情况下，推荐 Connection 与 Model 结合使用。

## 小结

本次 RFC 主要为了统一 Egg.js 的 Model 层，有以下几点：

1. 引入 Connection 与 Connection Loader 概念，从而能引进各种连接源，其不止是提供给 Model 层使用，而且可以在 Egg.js 任何一个角落被引用；
2. 新增 Model Loader 概念，使得 Model 的定义权不再在某个 ORM 的包上，而在于 **egg-model** 这个包中。

## 相关 Issue

+ https://github.com/eggjs/egg/issues/1775
+ https://github.com/eggjs/egg/issues/1749
+ https://github.com/eggjs/egg/issues/1675
+ https://github.com/eggjs/egg/issues/1670

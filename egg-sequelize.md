# egg-sequelize

Egg.js的[Sequelize](http://sequelizejs.com) 插件。

> 注意: 这个插件仅仅是将Sequelize整合进Egg.js中，更多的文档请访问 http://sequelizejs.com查看。

[![NPM version][npm-image]][npm-url]
[![build status][travis-image]][travis-url]
[![Test coverage][codecov-image]][codecov-url]
[![David deps][david-image]][david-url]
[![Known Vulnerabilities][snyk-image]][snyk-url]
[![npm download][download-image]][download-url]

[npm-image]: https://img.shields.io/npm/v/egg-sequelize.svg?style=flat-square
[npm-url]: https://npmjs.org/package/egg-sequelize
[travis-image]: https://img.shields.io/travis/eggjs/egg-sequelize.svg?style=flat-square
[travis-url]: https://travis-ci.org/eggjs/egg-sequelize
[codecov-image]: https://codecov.io/gh/eggjs/egg-sequelize/branch/master/graph/badge.svg
[codecov-url]: https://codecov.io/gh/eggjs/egg-sequelize
[david-image]: https://img.shields.io/david/eggjs/egg-sequelize.svg?style=flat-square
[david-url]: https://david-dm.org/eggjs/egg-sequelize
[snyk-image]: https://snyk.io/test/npm/egg-sequelize/badge.svg?style=flat-square
[snyk-url]: https://snyk.io/test/npm/egg-sequelize
[download-image]: https://img.shields.io/npm/dm/egg-sequelize.svg?style=flat-square
[download-url]: https://npmjs.org/package/egg-sequelize

## 安装

```bash
$ npm i --save egg-sequelize
$ npm install --save mysql2 # 针对mysql和mariadb

# 或使用其他数据库后端.
$ npm install --save pg pg-hstore # PostgreSQL
$ npm install --save tedious # MSSQL
```


## 用法 & 配置

- `config.default.js`

```js
exports.sequelize = {
  dialect: 'mysql', // 支持: mysql, mariadb, postgres, mssql
  database: 'test',
  host: 'localhost',
  port: '3306',
  username: 'root',
  password: '',
};
```

- `config/plugin.js`

``` js
exports.sequelize = {
  enable: true,
  package: 'egg-sequelize'
}
```
- `package.json`
```json
{
  "scripts": {
    "migrate:new": "egg-sequelize migration:create",
    "migrate:up": "egg-sequelize db:migrate",
    "migrate:down": "egg-sequelize db:migrate:undo"
  }
}
```


更多文档请访问 [Sequelize.js](http://sequelize.readthedocs.io/en/v3/)

## 模型文件

请将模型（model）文件放入 `app/model` 文件夹中

## 惯例

| 模型文件        | 类名                  |
| --------------- | --------------------- |
| `user.js`       | `app.model.User`      |
| `person.js`     | `app.model.Person`    |
| `user_group.js` | `app.model.UserGroup` |

- 表总是会有时间戳字段： `created_at datetime`, `updated_at datetime`。
- 使用下划线风格的列明, 例如： `user_id`, `comments_count`。

## 示例

### 标准

首先定义一个模型。

> 注意: `app.model` 是一个 [Sequelize的实例](http://docs.sequelizejs.com/class/lib/sequelize.js~Sequelize.html#instance-constructor-constructor), 所以你可以像这样使用它上面的方法： `app.model.sync, app.model.query ...`

```js
// app/model/user.js

module.exports = app => {
  const { STRING, INTEGER, DATE } = app.Sequelize;

  const User = app.model.define('user', {
    login: STRING,
    name: STRING(30),
    password: STRING(32),
    age: INTEGER,
    last_sign_in_at: DATE,
    created_at: DATE,
    updated_at: DATE,
  });

  User.findByLogin = function* (login) {
    return yield this.findOne({
      where: {
        login: login
      }
    });
  }

  User.prototype.logSignin = function* () {
    yield this.update({ last_sign_in_at: new Date() });
  }

  return User;
};

```

现在你可以在你的控制器中使用它:

```js
// app/controller/user.js
module.exports = app => {
  return class UserController extends app.Controller {
    * index() {
      const users = yield this.ctx.model.User.findAll();
      this.ctx.body = users;
    }

    * show() {
      const user = yield this.ctx.model.User.findByLogin(this.ctx.params.login);
      yield user.logSignin();
      this.ctx.body = user;
    }
  }
}
```

### 完整的示例

```js
// app/model/post.js

module.exports = app => {
  const { STRING, INTEGER, DATE } = app.Sequelize;

  const Post = app.model.define('Post', {
    name: STRING(30),
    user_id: INTEGER,
    created_at: DATE,
    updated_at: DATE,
  });

  Post.associate = function() {
    app.model.Post.belongsTo(app.model.User, { as: 'user' });
  }

  return Post;
};
```


```js
// app/controller/post.js
module.exports = app => {
  return class PostController extends app.Controller {
    * index() {
      const posts = yield this.ctx.model.Post.findAll({
        attributes: [ 'id', 'user_id' ],
        include: { model: this.ctx.model.User, as: 'user' },
        where: { status: 'publish' },
        order: 'id desc',
      });

      this.ctx.body = posts;
    }

    * show() {
      const post = yield this.ctx.model.Post.findById(this.params.id);
      const user = yield post.getUser();
      post.setDataValue('user', user);
      this.ctx.body = post;
    }

    * destroy() {
      const post = yield this.ctx.model.Post.findById(this.params.id);
      yield post.destroy();
      this.ctx.body = { success: true };
    }
  }
}
```

## 将模型同步到数据库中

**我们强烈建议您使用 [migrations](https://github.com/eggjs/egg-sequelize#migrations) 来创建或迁移数据库。**

**下面的代码仅适合在开发环境中使用。**

```js
// {app_root}/app.js
  module.exports = app => {
    if (app.config.env === 'local') {
      app.beforeStart(function* () {
        yield app.model.sync({force: true});
      });
    }
  };
```

## 迁移

如果你已经将egg-sequelize的NPM脚本添加进你的 `package.json`里，那么你就可以

| 命令 | 描述 |
|-----|------|
| npm run migrate:ne | 在./migrations/里生成新的迁移文件 |
| npm run migrate:up | 运行迁移 |
| npm run migrate:down | 回滚一次迁移 |

例如:

```bash
$ npm run migrate:up
```

在 `unittest` 环境下：

```bash
$ EGG_SERVER_ENV=unittest npm run migrate:up
```

或者在 `prod`环境下：

```bash
$ EGG_SERVER_ENV=prod npm run migrate:up
```

或是其他的环境：

```bash
$ EGG_SERVER_ENV=pre npm run migrate:up
```

将会从 `config/config.pre.js`载入数据库配置。

想要用生成器写出友好的迁移你需要使用 `co.wrap` 方法：

```js
'use strict';
const co = require('co');

module.exports = {
  up: co.wrap(function *(db, Sequelize) {
    const { STRING, INTEGER, DATE } = Sequelize;

    yield db.createTable('users', {
      id: { type: INTEGER, primaryKey: true, autoIncrement: true },
      name: { type: STRING, allowNull: false },
      email: { type: STRING, allowNull: false },
      created_at: DATE,
      updated_at: DATE,
    });

    yield db.addIndex('users', ['email'], { indicesType: 'UNIQUE' });
  }),

  down: co.wrap(function *(db, Sequelize) {
    yield db.dropTable('users');
  }),
};
```

也许你需要阅读 [Sequelize - Migrations](http://docs.sequelizejs.com/manual/tutorial/migrations.html) 来学习如何写迁移。

## 推荐的示例

- https://github.com/eggjs/examples/tree/master/sequelize-example/

## 问题 & 建议

请在[这里](https://github.com/eggjs/egg/issues)创建一个issue。

## 许可证

[MIT](LICENSE)

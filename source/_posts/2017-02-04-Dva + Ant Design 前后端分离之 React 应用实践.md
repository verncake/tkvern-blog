---
title: Dva + Ant Design 前后端分离之 React 应用实践
categories:
  - 技术
  - Front-end

date: 2017-02-04T09:31:07+00:00
---

![image](https://github.com/user-attachments/assets/feb7394d-277c-43bb-8b94-6c386de654cf)

继 [Rails 从入门到完全放弃 拥抱 Elixir + Phoenix + React + Redux](https://ruby-china.org/topics/30594) 这篇文章被喷之后，笔者很长一段时候没有上社区逛了。现在又回归了，给大家带来 React 实践的一些经验，一些踩坑的经验。

Rails 嘛，很好用，Laravel 也好用。Phoenix 也好用。都好，哪个方便用哪个。

还有关于 Turbolinks 之争，不能单从页面渲染时间去对比，要综合考虑。

<!-- more -->

## Why Dva？
Dva 是基于 Redux 做了一层封装，对于 React 的 state 管理，有很多方案，我选择了轻量、简单的 Dva。至于 Mobx，还没应用到项目中来。先等友军踩踩坑，再往里面跳。

- [Why dva and what’s dva](https://github.com/dvajs/dva/issues/1)
- [支付宝前端应用架构的发展和选择](https://www.github.com/sorrycc/blog/issues/6)

顺便贴下 Dva 的特性：

- 易学易用：仅有 5 个 api，对 redux 用户尤其友好
- elm 概念：通过 `reducers`, `effects` 和 `subscriptions` 组织 model
- 支持 mobile 和 react-native：跨平台 ([react-native 例子](https://github.com/sorrycc/dva-example-react-native))
- 支持 HMR：目前基于 [babel-plugin-dva-hmr](https://github.com/dvajs/babel-plugin-dva-hmr) 支持 components 和 routes 的 HMR
- 动态加载 Model 和路由：按需加载加快访问速度 ([例子](https://github.com/dvajs/dva/tree/master/examples/dynamic-load))
- 插件机制：比如 [dva-loading](https://github.com/dvajs/dva-loading) 可以自动处理 loading 状态，不用一遍遍地写 showLoading 和 hideLoading
- 完善的语法分析库 [dva-ast](https://github.com/dvajs/dva-ast)：[dva-cli](https://github.com/dvajs/dva-cli) 基于此实现了智能创建 model, router 等
- 支持 TypeScript：通过 d.ts ([例子](https://github.com/sorrycc/dva-boilerplate-typescript))

## Why Ant Design
做为传道士，这么好的 UI 设计语言，肯定不会藏着掖着啦。蚂蚁金服的东西，确实不错，除了 Ant Design 外，还有 Ant Design Mobile、AntV、AntMotion、G2。

## Why yarn?
`npm install` 太慢，试试 [yarn](https://yarnpkg.com/) 吧。建议用 `npm install yarn -g` 进行安装。

## 开发过程中的前后端分离
项目开始了，前端视图写完，要开始数据交互了，后端提供的 API 还没好。
那么问题来了，如何在不依靠后端提供 API 的情况下，实现数据交互？
使用 [Mock.js](http://mockjs.com/) 可以解决这个问题。先对接好 API 数据格式，然后使用 Mockjs 拦截 Ajax 请求，模拟后端真实数据。

在 Mockjs 官方提供的 API 不够用的情况下，还可以使用正则产生模拟数据。

### 如何对模拟做数据持久化处理?
这里给出一个模拟用户数据并持久化的实例实例：[mock/users.js](https://github.com/tkvern/dva-passport/blob/develop/resources/assets/ant_passport/mock/users.js)

代码摘要:

```typescript
"use strict";

const qs = require("qs");
const mockjs = require("mockjs");

const Random = mockjs.Random;

// 数据持久化
let tableListData = {};

if (!global.tableListData) {
  const data = mockjs.mock({
    "data|100": [
      {
        "id|+1": 1,
        name: () => {
          return Random.cname();
        },
        mobile: /1(3[0-9]|4[57]|5[0-35-9]|7[01678]|8[0-9])\d{8}/,
        avatar: () => {
          return Random.image("125x125");
        },
        "status|1-2": 1,
        email: () => {
          return Random.email("visiondk.com");
        },
        "isadmin|0-1": 1,
        created_at: () => {
          return Random.datetime("yyyy-MM-dd HH:mm:ss");
        },
        updated_at: () => {
          return Random.datetime("yyyy-MM-dd HH:mm:ss");
        },
      },
    ],
    page: {
      total: 100,
      current: 1,
    },
  });
  tableListData = data;
  global.tableListData = tableListData;
} else {
  tableListData = global.tableListData;
}
```

### 模拟 API 怎么写?
完成持久化处理后，就可以像操作数据库一样进行增、删、改、查

下面是一个删除用户的 API

参见 [mock/users.js#L106](https://github.com/tkvern/dva-passport/blob/develop/resources/assets/ant_passport/mock/users.js#L106)：

```typescript
'DELETE /api/users' (req, res) {
    setTimeout(() => {
      const deleteItem = qs.parse(req.body);

      tableListData.data = tableListData.data.filter((item) => {
        if (item.id === deleteItem.id) {
          return false;
        }

        return true;
      });

      tableListData.page.total = tableListData.data.length;

      global.tableListData = tableListData;

      res.json({
        success: true,
        data: tableListData.data,
        page: tableListData.page,
      });
    }, 200);
  },
```

### 还有一步
模拟数据和 API 写好了，还需要拦截 Ajax 请求

修改 `package.json`

```json
.
.
.
"scripts": {
  "start": "dora --plugins \"proxy,webpack,webpack-hmr\"",
  "build": "atool-build -o ../../../public",
  "test": "atool-test-mocha ./src/**/*-test.js"
}
.
.
.
```

如果与 `dora` 有端口冲突可修改 `dora` 的端口号

```json
"start": "dora --port 8888 --plugins \"proxy,webpack,webpack-hmr\"",
```

完成这些基本工作就做好了

### 友情提示
在模拟数据环境，`services` 下的模块这么写就好了，真实 API 则替换为真实 API 的地址。可将地址前缀写到统一配置中去。

```typescript
import request from "../utils/request";
import qs from "qs";
export async function query(params) {
  return request(`/api/users?${qs.stringify(params)}`);
}

export async function create(params) {
  return request("/api/users", {
    method: "post",
    body: qs.stringify(params),
  });
}

export async function remove(params) {
  return request("/api/users", {
    method: "delete",
    body: qs.stringify(params),
  });
}

export async function update(params) {
  return request("/api/users", {
    method: "put",
    body: qs.stringify(params),
  });
}
```

真实 API 参考实例: [src/services/users.js](https://github.com/tkvern/dva-passport/blob/develop/resources/assets/ant_passport/src/services/users.js)

## 如何保持登录状态
在看 dva 的引导手册时，并没有介绍登录相关的内容。因为不同的项目，对于登录这块的实现会有所不同，并不是唯一的。通常我们会使用 Cookie 的方式保持登录状态，或者 Auth 2.0 的技术。

这里介绍 Cookie 的方式。

登录成功之后服务器会设置一个当前域可以使用的 Cookie，例如 token 啥的。然后在每次数据请求的时候在 Request Headers 中携带 token，后端会基于这个 token 进行权限验证。思路清晰了，来看看具体实现吧。（注：在这次项目中使用了统一登录模块，通过 Header 中的 Authorization 进行验证，将只介绍拿到 token 之后的数据处理）

### 准备工作
对于操作 Cookie 的一些操作，建议先封装到工具类模块下。同时我把操作 LocalStorage 的一些操作也写进来了。

参见 [src/utils/helper.js](https://github.com/tkvern/dva-passport/blob/develop/resources/assets/ant_passport/src/utils/helper.js)

```typescript
.
.
.
// Operation Cookie
export function getCookie(name) {
  const reg = new RegExp('(^| )' + name + '=([^;]*)(;|$)');
  const arr = document.cookie.match(reg);
  if (arr) {
    return decodeURIComponent(arr[2]);
  } else {
    return null;
  }
}

export function delCookie({ name, domain, path }) {
  if (getCookie(name)) {
    document.cookie = name + '=; expires=Thu, 01-Jan-70 00:00:01 GMT; path=' +
                      path + '; domain=' +
                      domain;
  }
}
.
.
.
```

Header 的预处理我放在了 [src/utils/auth.js#L5](https://github.com/tkvern/dva-passport/blob/develop/resources/assets/ant_passport/src/utils/auth.js#L5)，这里后端返回的数据都是 JSON 格式，所以在 Header 里面需要添加 application/json 进去，而 Authorization 是后端用来验证用户信息的。变量 sso_token 为了方便代码阅读就没有按照规范命名了。

```typescript
export function getAuthHeader(sso_token) {
  return {
    headers: {
      Accept: "application/json",
      Authorization: "Bearer " + sso_token,
      "Content-Type": "application/json",
    },
  };
}
```

### 修改 Request

这里没有使用自带的 catch 机制来处理请求错误，在开发过程中，最开始打算使用统一错误处理，但是发现请求失败后，不能在 models 层处理 components，所以就换了一种方式处理，后面会讲到。

参见 [src/utils/request.js#L29](https://github.com/tkvern/dva-passport/blob/develop/resources/assets/ant_passport/src/utils/request.js#L29)

```typescript
export default function request(url, options) {
  const sso_token = getCookie("sso_token");
  const authHeader = getAuthHeader(sso_token);
  return fetch(url, { ...options, ...authHeader })
    .then(checkStatus)
    .then(parseJSON)
    .then((data) => ({ data }));
  // .catch((err) => ({ err }));
}
```

完成这些配置之后，每次向服务器发送的请求就都携带了用户 token 了。在 token 无效时，服务器会抛出 401 错误，这时就需要在中间件中处理 401 错误。

参见 [src/utils/request.js#L10](https://github.com/tkvern/dva-passport/blob/develop/resources/assets/ant_passport/src/utils/request.js#L10)

redirectLogin 是工具类 [src/utils/auth.js](https://github.com/tkvern/dva-passport/blob/develop/resources/assets/ant_passport/src/utils/auth) 中的重定向登录方法。

```typescript
function checkStatus(response) {
  if (response && response.status === 401) {
    redirectLogin();
  }
  if (response.status >= 200 && response.status < 500) {
    return response;
  }
  const error = new Error(response.statusText);
  error.response = response;
  throw error;
}
```

到此为止，登录状态的配置基本完成。

## Router
我们的应用中会有多个页面，而且有的需要登录才可见，那么如何控制呢？

React 的路由控制是比较灵活的，来看看下面这个例子：

[src/router.jsx](https://github.com/tkvern/dva-passport/blob/develop/resources/assets/ant_passport/src/router.jsx)

```tsx
import React from "react";
import { Router, Route } from "dva/router";
import { authenticated } from "./utils/auth";
import Dashboard from "./routes/Dashboard";
import Users from "./routes/Users";
import User from "./routes/User";
import Password from "./routes/Password";
import Roles from "./routes/Roles";
import Permissions from "./routes/Permissions";

export default function ({ history }) {
  return (
    <Router history={history}>
      <Route path="/" component={Dashboard} onEnter={authenticated} />
      <Route path="/user" component={User} onEnter={authenticated} />
      <Route path="/password" component={Password} onEnter={authenticated} />
      <Route path="/users" component={Users} onEnter={authenticated} />
      <Route path="/roles" component={Roles} onEnter={authenticated} />
      <Route
        path="/permissions"
        component={Permissions}
        onEnter={authenticated}
      />
    </Router>
  );
}
```

对于路由的验证配置在 onEnter 属性中，authenticated 方法可统一进行路由验证，要注意每一个 Route 节点的验证都需要配置相应的 onEnter 属性。如果权限较为复杂需对每一个 Route 单独验证。其实这种基于客户端渲染的应用，如果页面限制有遗漏也关系不太，后端提供的 API 会对数据进行验证，即使前端访问到没有权限的页面，也同样不用担心，做好客户端错误处理即可。

## 数据缓存
对于一个 React 应用来说，缓存是很重要的一步。前后端分离后，频繁的 Ajax 请求会消耗大量的服务器资源，如果一些不长变动的持久化数据不做缓存的话，会浪费许多资源。所以，比较常见的方法就是将数据缓存在 LocalStorage 中。针对一些敏感信息可适当进行加密混淆处理，我这里就不介绍了。

### 什么时候做数据缓存?

例：用户信息缓存

参见 [src/models/auth.js#L64](https://github.com/tkvern/dva-passport/blob/develop/resources/assets/ant_passport/src/models/auth.js#L64)

在 subscriptions 中配置了 setup 检测 LocalStorage 中的 user 是否存在。不存在时会去 query 用户信息，然后保存到 user 中，如果存在就将 user 中的数据添加到 state 的 user: {}中。当然在进行请求时，已经在 src/utils/auth.js 验证用户信息是否正确，同时做了相应的限制 [src/utils/auth.js#L20](https://github.com/tkvern/dva-passport/blob/develop/resources/assets/ant_passport/src/utils/auth.js#L20)

```typescript
import { parse } from 'qs';
import { message } from 'antd';
import { query, update, password } from '../services/auth';
import { getLocalStorage, setLocalStorage } from '../utils/helper';

export default {
  namespace: 'auth',
  state: {
    user: {},
    isLogined: false,
    currentMenu: [],
  },
  reducers: {
    querySuccess(state, action) {
      return { ...state, ...action.payload, isLogined: true };
    },
  },
  effects: {
    *query({ payload }, { call, put }) {
      const { data } = yield call(query, parse(payload));
      if (data && data.err_msg === 'SUCCESS') {
        setLocalStorage('user', data.data);
        yield put({
          type: 'querySuccess',
          payload: {
            user: data.data,
          },
        });
      }
    },
  }
  subscriptions: {
    setup({ dispatch }) {
      const data = getLocalStorage('user');
      if (!data) {
        dispatch({
          type: 'query',
          payload: {},
        });
      } else {
        dispatch({
          type: 'querySuccess',
          payload: {
            user: data,
          },
        });
      }
    },
  },
}
```

简单来说，就是没有缓存的时候缓存。

### 什么时候更新数据缓存？
例如，roles 中添加和修改功能都需要用到 permissions 的数据，哪我怎么拿到最新的 permissions 数据呢。首先，我在加载 roles 列表页面时就需要将 permissions 的数据缓存，这样，在每次点添加或修改功能时就不需要再去拉取已缓存的数据了。

参见 [src/models/roles.js#L166](https://github.com/tkvern/dva-passport/blob/develop/resources/assets/ant_passport/src/models/roles.js#L166)

```typescript
.
.
.
  subscriptions: {
    setup({ dispatch, history }) {
      history.listen((location) => {
        const match = pathToRegexp('/roles').exec(location.pathname);
        if (match) {
          const data = getLocalStorage('permissions');
          if (!data) {
            dispatch({
              type: 'permissions/updateCache',
            });
          }
          dispatch({
            type: 'query',
            payload: location.query,
          });
        }
      });
    },
  },
.
.
.
```

### 什么时候删除数据缓存？
删除缓存的配置是比较灵活的，这里的业务场景并不复杂所以，我用了比较简单的处理方式。

参见 [src/models/permissions.js#L112](https://github.com/tkvern/dva-passport/blob/develop/resources/assets/ant_passport/src/models/permissions.js#L112)

在执行新增或更新操作成功后，将本地原有的缓存删除。加上数据联动的特性，当再次回到 roles 操作时，缓存已经更新了。

```typescript
.
.
.
  *update({ payload }, { select, call, put }) {
      yield put({ type: 'hideModal' });
      yield put({ type: 'showLoading' });
      const id = yield select(({ permissions }) => permissions.currentItem.id);
      const newRole = { ...payload, id };
      const { data } = yield call(update, newRole);
      if (data && data.err_msg === 'SUCCESS') {
        yield put({
          type: 'updateSuccess',
          payload: newRole,
        });
        localStorage.removeItem('permissions');
        message.success('更新成功!');
      }
    },
.
.
.
```

### State 的临时缓存
state 的中的数据是变化的，刷新页面之后会重置掉，也可以将部分 models 中的 state 存到 LocalStorage 中，让 state 的数据从 LocalStorage 读取，但不是必要的。而 list 数据的更新，是直接操作 state 中的数据的。

如下(这样就不用更新整个 list 的数据了)。

```typescript
.
.
.
    grantSuccess(state, action) {
      const grantUser = action.payload;
      const newList = state.list.map((user) => {
        if (user.id === grantUser.id) {
          user.roles = grantUser.roles;
          return { ...user };
        }
        return user;
      });
      return { ...state, ...newList, loading: false };
    },
.
.
.
```

## 视图组件运用
Ant 提供的组件非常多，但用起来还是需要一些学习成本的，同时多个组件组合使用时也需要有很多地方注意的。

### Modal 注意事项
在使用 Modal 组件时，难免会出现一个页面多个 Modal 的情况，首先要注意的就是 Modal 的命名，在多 Modal 情况下，命名不注意很容易出现分不清用的是哪个 Modal。建议命名时能望名知意。然后就是 Modal 需要用到别的 Models 的数据时，如果在弹窗时通过 Ajax 获取需要的数据再显示 Modal，这样就会出现 Modal 延迟，而且 Modal 的动画也无法加载出来。所以，我的处理方式是，在进入这一级 Route 的时候就将需要的数据预缓存，这样调用时就可随用随取，不会出现延迟了。

参见 [src/components/user/UserModalGrant.jsx#L33](https://github.com/tkvern/dva-passport/blob/develop/resources/assets/ant_passport/src/components/user/UserModalGrant.jsx#L33)

### Form 注意
Ant 的 form 组件很完善，需要注意的就是表单的多条件查询。如果单单是一个条件查询的处理比较简单，将查询关键词设成 string 类型存到相应的 Models 中的 state 即可，多条件的话，稍微麻烦一点，需存成 Hash 对象。灵活处理即可。

### 其他
官方文档的描述很清楚，我就不充大头了。注意写法规范即可，直接复制粘贴官方例子代码会很难看。

## 跨域问题
终于说到点子上了，前后端分离遇到跨域问题很正常，而这种基于 RESTful API 的前后端分离就更好弄了。我这以 Fetch + PHP + Laravel 为例，这种并不是最有解决方案！仅供参考！

在 header 中进行如下配置

- Access-Control-Allow-Origin 配置允许的域
- Access-Control-Allow-Methods 配置允许的请求方式
- Access-Control-Allow-Headers 配置允许的请求头

```php
<?php

use Illuminate\Http\Request;

/*
|--------------------------------------------------------------------------
| API Routes
|--------------------------------------------------------------------------
|
| Here is where you can register API routes for your application. These
| routes are loaded by the RouteServiceProvider within a group which
| is assigned the "api" middleware group. Enjoy building your API!
|
*/

Route::group(['middleware'=> ['auth:api']], function() {
    header("Access-Control-Allow-Origin: *");
    header("Access-Control-Allow-Methods: GET, HEAD, POST, PUT, PATCH, DELETE");
    header("Access-Control-Allow-Headers: Access-Control-Allow-Headers, Origin, Accept, Authorization, X-Requested-With, Content-Type, Access-Control-Request-Method, Access-Control-Request-Headers");
    require base_path('routes/common.php');
});
```

基于其他编程语言的处理类似。

## 结语
了解前端、熟悉前端、精通前端、熟悉前端、不懂前端

了解 X X 、熟悉 X X 、精通 X X 、熟悉 X X 、不懂 X X
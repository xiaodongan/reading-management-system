# 用户登录（下）（后端内容）

## 后端 API 处理流程

![login_api.png](http://www.youbaobao.xyz/admin-docs/assets/img/login_api.35a53c35.png)

## 搭建 https 服务

首先需要将 https 证书拷贝到 node 项目中，然后添加下列代码：

```js
const fs = require('fs')
const https = require('https')

const privateKey = fs.readFileSync('https/book_youbaobao_xyz.key', 'utf8')
const certificate = fs.readFileSync('https/book_youbaobao_xyz.pem', 'utf8')
const credentials = { key: privateKey, cert: certificate }
const httpsServer = https.createServer(credentials, app)
const SSLPORT = 18082
httpsServer.listen(SSLPORT, function() {
  console.log('HTTPS Server is running on: https://localhost:%s', SSLPORT)
})
```

启动 https 服务需要证书对象 credentials，包含了私钥和证书，重新启动 node 服务：

```bash
node app.js
```

在浏览器中输入：

```bash
https://book.youbaobao.xyz:18082
```

可以看到：

```bash
欢迎学习小慕读书管理后台
```

说明 https 服务启动成功



## 创建 `/user/login` API

在 `router/user.js` 中填入以下代码：

```js
router.post('/login', function(req, res, next) {
  console.log('/user/login', req.body)
  res.json({
    code: 0,
    msg: '登录成功'
  })
})
```

```bash
$ curl https://book.youbaobao.xyz:18082/user/login -X POST -d "username=sam&password=123456"

{"code":0,"msg":"登录成功"}
```

上述命令可以简写为：

```bash
curl https://book.youbaobao.xyz:18082/user/login -d "username=sam&password=123456"
```

这里我们通过 `req.body` 获取 POST 请求中的参数，但是没有获取成功，我们需要通过 `body-parser` 中间件来解决这个问题：

```bash
npm i -S body-parser
```

在 app.js 中加入：

```js
const bodyParser = require('body-parser')

// 创建 express 应用
const app = express()

app.use(bodyParser.urlencoded({ extended: true }))
app.use(bodyParser.json())
```

TIP

关于 body-parser 的实现原理与细节可以参考这篇文档，说得非常明白：https://juejin.im/post/59222c5d2f301e006b1616ae

返回前端使用登录按钮请求登录接口，发现控制台报错：

```bash
Access to XMLHttpRequest at 'https://book.youbaobao.xyz:18082/user/login' from origin 'http://localhost:9527' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

这是由于前端部署在 `http://localhost:9527` 而后端部署在 `https://book.youbaobao.xyz:18082`，所以导致了跨域错误，我们需要在 node 服务中添加跨域中间件 cors：

```bash
npm i -S cors
```

然后修改 app.js：

```js
const cors = require('cors')

// ...
app.use(cors())
```

再次请求即可成功，这里我们在 Network 中会发现发起了两次 https 请求，这是因为由于触发跨域，所以会首先进行 OPTIONS 请求，判断服务端是否允许跨域请求，如果允许才能实际进行请求

TIP

关于为什么要发起 OPTIONS 请求，大家可以参考这篇文档：https://juejin.im/post/5cb3eedcf265da038f7734c4

## 响应结果封装

在 `/user/login` 我们看到返回值是：

```js
res.json({
  code: 0,
  msg: '登录成功'
})
```

之后我们还要定义错误返回值，但如果每个接口都编写以上代码就显得非常冗余，而且不易维护，比如我们要将 code 默认值从 0 改为 1，就要修改每个接口，所以我们创建一个 Result 类来解决这个问题

创建 `/models/Result.js` 文件：

```js
const {
  CODE_ERROR,
  CODE_SUCCESS
} = require('../utils/constant')

class Result {
  constructor(data, msg = '操作成功', options) {
    this.data = null
    if (arguments.length === 0) {
      this.msg = '操作成功'
    } else if (arguments.length === 1) {
      this.msg = data
    } else {
      this.data = data
      this.msg = msg
      if (options) {
        this.options = options
      }
    }
  }

  createResult() {
    if (!this.code) {
      this.code = CODE_SUCCESS
    }
    let base = {
      code: this.code,
      msg: this.msg
    }
    if (this.data) {
      base.data = this.data
    }
    if (this.options) {
      base = { ...base, ...this.options }
    }
    console.log(base)
    return base
  }

  json(res) {
    res.json(this.createResult())
  }

  success(res) {
    this.code = CODE_SUCCESS
    this.json(res)
  }

  fail(res) {
    this.code = CODE_ERROR
    this.json(res)
  }
}

module.exports = Result
```

我们还需要创建 `/utils/constant.js`：

```js
module.exports = {
  CODE_ERROR: -1,
  CODE_SUCCESS: 0
}
```

Result 使用了 ES6 的 Class，使用方法如下：

```js
// 调用成功时
new Result().success(res)
new Result('登录成功').success(res)
// 调用成功时，包含参数
new Result({ token }, '登录成功').success(res)
// 调用失败时
new Result('用户名或密码不存在').fail(res)
```

有了 Result 类后，我们可以将登录 API 改为：

```js
router.post('/login', function(req, res, next) {
  const username = req.body.username
  const password = req.body.password

  if (username === 'admin' && password === '123456') {
    new Result('登录成功').success(res)
  } else {
    new Result('登录失败').fail(res)
  }
})
```

> 如果在响应前抛出 Error，此时 Error 将被我们自定义的异常处理捕获，并返回 500 至前端

## 登录用户数据库查询

响应过程封装完毕后，我们需要在数据库中查询用户信息来验证用户名和密码是否准确

#### 安装

安装 mysql 库：

```bash
 npm i -S mysql
```

#### 配置

创建 db 目录，新建两个文件：

```bash
index.js
config.js
```

config.js 源码如下：

```js
module.exports = {
  host: 'localhost',
  user: 'root',
  password: '12345678',
  database: 'book'
}
```

#### 连接

连接数据库需要提供使用 mysql 库的 createConnection 方法，我们在 index.js 中创建如下方法：

```js
function connect() {
  return mysql.createConnection({
    host,
    user,
    password,
    database,
    multipleStatements: true
  })
}
```

> multipleStatements：允许每条 mysql 语句有多条查询.使用它时要非常注意，因为它很容易引起 sql 注入（默认：false）

#### 查询

查询需要调用 connection 对象的 query 方法：

```js
function querySql(sql) {
  const conn = connect()
  debug && console.log(sql)
  return new Promise((resolve, reject) => {
    try {
      conn.query(sql, (err, results) => {
        if (err) {
          debug && console.log('查询失败，原因:' + JSON.stringify(err))
          reject(err)
        } else {
          debug && console.log('查询成功', JSON.stringify(results))
          resolve(results)
        }
      })
    } catch (e) {
      reject(e)
    } finally {
      conn.end()
    }
  })
}
```

我们在 constant.js 创建一个 debug 参数控制日志打印：

```js
const debug = require('../utils/constant').debug
```

> 这里需要注意 conn 对象使用完毕后需要调用 end 进行关闭，否则会导致内存泄露

调用方法如下：

```js
db.querySql('select * from book').then(result => {
  console.log(result)
})
```

这里我们需要基于 mysql 查询库封装一层 service，用来协调业务逻辑和数据库查询，我们不希望直接把业务逻辑写在 router 中，创建 `/service/user.js`：

```js
const { querySql } = require('../db')

function login(username, password) {
  const sql = `select * from admin_user where username='${username}' and password='${password}'`
  return querySql(sql)
}

module.exports = {
  login
}
```

改造 `/user/login` API：

```js
router.post('/login', function(req, res, next) {
  const username = req.body.username
  const password = req.body.password

  login(username, password).then(user => {
    if (!user || user.length === 0) {
      new Result('登录失败').fail(res)
    } else {
      new Result('登录成功').success(res)
    }
  })
})
```

此时即使我们输入正确的用户名和密码仍然无法登录，这是因为密码采用了 MD5 + SALT 加密，所以我们需要对密码进行对等加密，才能查询成功。在 `/utils/constant.js` 中加入 SALT：

```js
module.exports = {
  // ...
  PWD_SALT: 'admin_imooc_node',
}
```

安装 crypto 库：

```bash
npm i -S crypto
```

然后在 `/utils/index.js` 中创建 md5 方法：

```js
const crypto = require('crypto')

function md5(s) {
  // 注意参数需要为 String 类型，否则会出错
  return crypto.createHash('md5')
    .update(String(s)).digest('hex');
}
```

再次输入正确的用户名和密码，查询成功：

```bash
select * from admin_user where username='admin' and password='91fe0e80d07390750d46ab6ed3a99316'
查询成功 [{"id":1,"username":"admin","password":"91fe0e80d07390750d46ab6ed3a99316","role":"admin","nicknamedmin","avatar":"https://www.youbaobao.xyz/mpvue-res/logo.jpg"}]
{ code: 0, msg: '登录成功' }
```

## express-validator

express-validator 是一个功能强大的表单验证器，它是 validator.js 的中间件

TIP

源码地址：https://github.com/express-validator/express-validator

使用 express-validator 可以简化 POST 请求的参数验证，使用方法如下：

安装

```bash
npm i -S express-validator
```

验证

```js
const { body, validationResult } = require('express-validator')
const boom = require('boom')

router.post(
  '/login',
  [
    body('username').isString().withMessage('username类型不正确'),
    body('password').isString().withMessage('password类型不正确')
  ],
  function(req, res, next) {
    const err = validationResult(req)
    if (!err.isEmpty()) {
      const [{ msg }] = err.errors
      next(boom.badRequest(msg))
    } else {
      const username = req.body.username
      const password = md5(`${req.body.password}${PWD_SALT}`)

      login(username, password).then(user => {
        if (!user || user.length === 0) {
          new Result('登录失败').fail(res)
        } else {
          new Result('登录成功').success(res)
        }
      })
    }
  })
```

express-validator 使用技巧：

- 在 `router.post` 方法中使用 body 方法判断参数类型，并指定出错时的提示信息
- 使用 `const err = validationResult(req)` 获取错误信息，`err.errors` 是一个数组，包含所有错误信息，如果 `err.errors` 为空则表示校验成功，没有参数错误
- 如果发现错误我们可以使用 `next(boom.badRequest(msg))` 抛出异常，交给我们自定义的异常处理方法进行处理

## JWT 基本概念

### Token 简析

#### Token 是什么

Token 本质是字符串，用于请求时附带在请求头中，校验请求是否合法及判断用户身份

#### Token 与 Session、Cookie 的区别

- Session 保存在服务端，用于客户端与服务端连接时，临时保存用户信息，当用户释放连接后，Session 将被释放；
- Cookie 保存在客户端，当客户端发起请求时，Cookie 会附带在 http header 中，提供给服务端辨识用户身份；
- Token 请求时提供，用于校验用户是否具备访问接口的权限。

#### Token 的用途

Token 的用途主要有三点：

- 拦截无效请求，降低服务器处理压力；
- 实现第三方 API 授权，无需每次都输入用户名密码鉴权；
- 身份校验，防止 CSRF 攻击。

### JWT 简析

JSON Web Token（JWT）是非常流行的跨域身份验证解决方案。

![jwt](http://www.youbaobao.xyz/admin-docs/assets/img/jwt.2b8365ae.png)

### JWT 构成

下面是一串 JWT 字符串：

```bash
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.EJzdDeNLOQhy-2WmXuK1B49xF17Tk0pja1tCPp81YjY
```

它将被解析为以下三部分：

#### HEADER: ALGORITHM & TOKEN TYPE

JWT 头部分是一个描述 JWT 元数据的 JSON 对象：

- alg：表示加密算法，HS256 是 HMAC SHA256 的缩写
- typ：token 类型

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

#### PAYLOAD: DATA

JWT 数据部分，payload 是 JWT 的主体内容部分，也是一个 JSON 字符串，包含需要传递的数据，注意 payload 部分不要存储隐私数据，防止信息泄露

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022
}
```

#### VERIFY SIGNATURE

JWT 签名部分是对上面两部分数据加密后生成的字符串，通过 header 指定的算法生成加密字符串，以确保数据不会被篡改。

生成签名时需要使用密钥（即下方示例中的 abcdefg），密钥只保存在服务端，不能向用户公开，它是一个字符串，我们可以自由指定。

生成签名时需要根据 header 中指定的签名算法，并根据下方的公式，即将 header 和 payload 的数据通过 base64加密后用 `.` 进行连接，然后通过密钥进行 SHA256 加密，由于加入了密钥，所以生成的字符串将无法被破译和篡改，只有在服务端才能还原

```js
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  abcdefg
)
```

我们可以在 https://jwt.io/ 调试 JWT 字符串

## 生成 JWT Token

安装 jsonwebtoken

```bash
npm i -S jsonwebtoken
```

使用

```js
const jwt = require('jsonwebtoken')
const { PRIVATE_KEY, JWT_EXPIRED } = require('../utils/constant')

login(username, password).then(user => {
    if (!user || user.length === 0) {
      new Result('登录失败').fail(res)
    } else {
      const token = jwt.sign(
        { username },
        PRIVATE_KEY,
        { expiresIn: JWT_EXPIRED }
      )
      new Result({ token }, '登录成功').success(res)
    }
})
```

这里需要定义 jwt 的私钥和过期时间，过期时间不宜过短，也不宜过长，课程里设置为 1 小时，实际业务中可根据场景来判断，通常建议不超过 24 小时，保密性要求高的业务可以设置为 1-2 小时：

```js
module.exports = {
  // ...
  PRIVATE_KEY: 'admin_imooc_node_test_youbaobao_xyz',
  JWT_EXPIRED: 60 * 60, // token失效时间
}
```

前端再次请求，结果如下：

```json
{
  "code":0,
  "msg":"登录成功",
  "data":{
    "token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTc0NDk1NzA0LCJleHAiOjE1NzQ0OTkzMDR9.9lnxdTn1MmMbKsPvhvRHDRIufbMcUD437CWjnoJsmfo"
  }
}
```

我们可以将该 token 在 `jwt.io` 网站上进行验证，可以得到如下结果：

```json
{
  "username": "admin",
  "iat": 1574495704,
  "exp": 1574499304
}
```

可以看到 username 被正确解析，说明 token 生成成功

## 前端登录请求改造

修改 `src/utils/request.js`：

```js
service.interceptors.response.use(
  response => {
    const res = response.data

    if (res.code !== 0) {
      Message({
        message: res.msg || 'Error',
        type: 'error',
        duration: 5 * 1000
      })
      // 判断 token 失效的场景
      if (res.code === -2) {
        // 如果 token 失效，则弹出确认对话框，用户点击后，清空 token 并返回登录页面
        MessageBox.confirm('Token 失效，请重新登录', '确认退出登录', {
          confirmButtonText: '重新登录',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          store.dispatch('user/resetToken').then(() => {
            location.reload()
          })
        })
      }
      return Promise.reject(new Error(res.msg || '请求失败'))
    } else {
      return res
    }
  },
  error => {
    let message = error.message || '请求失败'
    if (error.response && error.response.data) {
      const { data } = error.response
      message = data.msg
    }
    Message({
      message,
      type: 'error',
      duration: 5 * 1000
    })
    return Promise.reject(error)
  }
)
```

## JWT 认证

安装 express-jwt

```bash
npm i -S express-jwt
```

创建 `/router/jwt.js`

```js
const expressJwt = require('express-jwt');
const { PRIVATE_KEY } = require('../utils/constant');

const jwtAuth = expressJwt({
  secret: PRIVATE_KEY,
  credentialsRequired: true // 设置为false就不进行校验了，游客也可以访问
}).unless({
  path: [
    '/',
    '/user/login'
  ], // 设置 jwt 认证白名单
});

module.exports = jwtAuth;
```

在 `/router/index.js` 中使用中间件

```js
const jwtAuth = require('./jwt')

// 注册路由
const router = express.Router()

// 对所有路由进行 jwt 认证
router.use(jwtAuth)
```

在 `/utils/contants.js` 中添加：

```js
module.exports = {
  // ...
  CODE_TOKEN_EXPIRED: -2
}
```

修改 `/model/Result.js`：

```js
expired(res) {
  this.code = CODE_TOKEN_EXPIRED
  this.json(res)
}
```

修改自定义异常：

```js
router.use((err, req, res, next) => {
  if (err.name === 'UnauthorizedError') {
    new Result(null, 'token失效', {
      error: err.status,
      errorMsg: err.name
    }).expired(res.status(err.status))
  } else {
    const msg = (err && err.message) || '系统错误'
    const statusCode = (err.output && err.output.statusCode) || 500;
    const errorMsg = (err.output && err.output.payload && err.output.payload.error) || err.message
    new Result(null, msg, {
      error: statusCode,
      errorMsg
    }).fail(res.status(statusCode))
  }
})
```

## 前端传入 JWT Token

后端添加路由的 jwt 认证后，再次请求 `/user/info` 将抛出 401 错误，这是由于前端未传递合理的 Token 导致，下面我们就修改 `/utils/request.js`，使得前端请求时可以传递 Token：

```js
service.interceptors.request.use(
  config => {
    // 如果存在 token 则附带在 http header 中
    if (store.getters.token) {
      config.headers['Authorization'] = `Bearer ${getToken()}`
    }
    return config
  },
  error => {
    return Promise.reject(error)
  }
)
```

前端去掉 `/user/info` 请求时传入的 token，因为我们已经从 token 中传入，修改 `src/api/user.js`：

```js
export function getInfo() {
  return request({
    url: '/user/info',
    method: 'get'
  })
}
```

## 用户查询 `/user/info` API

在 `/db/index.js` 中添加：

```js
function queryOne(sql) {
  return new Promise((resolve, reject) => {
    querySql(sql)
      .then(results => {
        if (results && results.length > 0) {
          resolve(results[0])
        } else {
          resolve(null)
        }
      })
      .catch(error => {
        reject(error)
      })
  })
}
```

在 `/services/user.js` 中添加：

```js
function findUser(username) {
  const sql = `select * from admin_user where username='${username}'`
  return queryOne(sql)
}
```

此时有个问题，前端仅在 Http Header 中传入了 Token，如果通过 Token 获取 username 呢？这里就需要通过对 JWT Token 进行解析了，在 `/utils/index.js` 中添加 decode 方法：

```js
const jwt = require('jsonwebtoken')
const { PRIVATE_KEY } = require('./constant')

function decode(req) {
  const authorization = req.get('Authorization')
  let token = ''
  if (authorization.indexOf('Bearer') >= 0) {
    token = authorization.replace('Bearer ', '')
  } else {
    token = authorization
  }
  return jwt.verify(token, PRIVATE_KEY)
}
```

修改 `/router/user.js`：

```js
router.get('/info', function(req, res) {
  const decoded = decode(req)
  if (decoded && decoded.username) {
    findUser(decoded.username).then(user => {
      if (user) {
        user.roles = [user.role]
        new Result(user, '获取用户信息成功').success(res)
      } else {
        new Result('获取用户信息失败').fail(res)
      }
    })
  } else {
    new Result('用户信息解析失败').fail(res)
  }
})
```

此时在前端重新登录，登录终于成功了！

## 修改 Logout 方法

修改 `src/store/modules/user.js`：

```js
logout({ commit, state, dispatch }) {
    return new Promise((resolve, reject) => {
      try {
        commit('SET_TOKEN', '')
        commit('SET_ROLES', [])
        removeToken()
        resetRouter()
        // reset visited views and cached views
        // to fixed https://github.com/PanJiaChen/vue-element-admin/issues/2485
        dispatch('tagsView/delAllViews', null, { root: true })
        resolve()
      } catch (e) {
        reject(e)
      }
    })
}
```

## 关于 RefreshToken

如果你的场景需要授权给第三方 app，那么通常我们需要再增加一个 RefreshToken 的 API，该 API 的用途是根据现有的 Token 获取用户名，然后生成一个新的 Token，这样做的目的是为了防止 Token 失效后退出登录，所以 app 一般会在打开时刷新一次 Token，该 API 的实现方法比较简单，所需的技术之前都已经介绍过，大家可以参考之前的文档进行实现
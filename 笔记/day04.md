# 用户登录（中）

## 登录组件分析

### 布局分析

登录组件 login.vue 布局要点如下：

- el-form 容器，包含 username 和 password 两个 el-form-item，el-form 主要属性：
  - model 为 loginForm
  - rules 为 loginRules
- password 使用了 el-tooltip 提示，当用户打开大小写时，会进行提示，主要属性：
  - manual：手动控制模式，设置为 true 后，mouseenter 和 mouseleave 事件将不会生效
  - placement：提示出现的位置
- password 对应的 el-input 主要属性：
  - `@keyup.native="checkCapslock"` 键盘按键时绑定 checkCapslock 事件
  - `@keyup.enter.native="handleLogin"` 监听键盘 enter 按下后的事件

> 这里绑定 `@keyup` 事件时需要添加 `.native` 修饰符，这是因为我们的事件绑定在 el-input 组件上，所以如果不添加 `.native` 修饰符，事件将无法绑定到原生的 input 标签上

- 包含一个 el-button，点击时调用 `handleLogin` 方法，并触发 loading 效果

### checkCapslock 方法

`checkCapslock` 方法的主要用途是监听用户键盘输入，显示提示文字的判断逻辑如下：

- 按住 shift 时输入小写字符
- 未按 shift 时输入大写字符

当按下 CapsLock 按键时，如果按下后是小写模式，则会立即消除提示文字

```js
checkCapslock({ shiftKey, key } = {}) {
  if (key && key.length === 1) {
    if (shiftKey && (key >= 'a' && key <= 'z') || !shiftKey && (key >= 'A' && key <= 'Z')) {
      this.capsTooltip = true
    } else {
      this.capsTooltip = false
    }
  }
  if (key === 'CapsLock' && this.capsTooltip === true) {
    this.capsTooltip = false
  }
}
```

### handleLogin 方法

`handleLogin` 方法处理流程如下：

- 调用 el-form 的 `validate` 方法对 rules 进行验证；
- 如果验证通过，则会调用 vuex 的 `user/login` action 进行登录验证；
- 登录验证通过后，会重定向到 redirect 路由，如果 redirect 路由不存在，则直接重定向到 `/` 路由

> 这里需要注意：由于 vuex 中的 user 指定了 namespaced 为 true，所以 dispatch 时需要加上 namespace，否则将无法调用 vuex 中的 action

```js
handleLogin() {
  this.$refs.loginForm.validate(valid => {
    if (valid) {
      this.loading = true
      this.$store.dispatch('user/login', this.loginForm)
        .then(() => {
          this.$router.push({ path: this.redirect || '/', query: this.otherQuery })
          this.loading = false
        })
        .catch(() => {
          this.loading = false
        })
    } else {
      console.log('error submit!!')
      return false
    }
  })
}
```

### user/login 方法

`user/login` 方法调用了 login API，传入 username 和 password 参数，请求成功后会从 response 中获取 token，然后将 token 保存到 Cookie 中，之后返回。如果请求失败，将调用 reject 方法，交由我们自定义的 request 模块来处理异常

```js
login({ commit }, userInfo) {
    const { username, password } = userInfo
    return new Promise((resolve, reject) => {
      login({ username: username.trim(), password: password }).then(response => {
        const { data } = response
        commit('SET_TOKEN', data.token)
        setToken(data.token)
        resolve()
      }).catch(error => {
        reject(error)
      })
    })
}
```

login API 的方法如下：

```js
import request from '@/utils/request'

export function login(data) {
  return request({
    url: '/user/login',
    method: 'post',
    data
  })
}
```

这里使用 request 方法，它是一个基于 axios 封装的库，目前我们的 `/user/login` 接口是通过 mock 实现的，用户数据位于 `/mock/user.js`

## axios 用法分析

request 库使用了 axios 的手动实例化方法 create 来封装请求，要理解其中的用法，我们需要首先学习 axios 库的用法

> [查看](http://www.axios-js.com/zh-cn/docs/) axios 官网

### axios 基本案例

我们先从一个普通的 axios 示例开始：

```js
import axios from 'axios'

const url = 'https://test.youbaobao.xyz:18081/book/home/v2?openId=1234'
axios.get(url).then(response => {
  console.log(response)
})
```

上述代码可以改为：

```js
const url = 'https://test.youbaobao.xyz:18081/book/home/v2'
axios.get(url, { 
  params: { openId: '1234' }
})
```

如果我们在请求时需要在 http header 中添加一个 token，需要将代码修改为：

```js
const url = 'https://test.youbaobao.xyz:18081/book/home/v2'
axios.get(url, { 
  params: { openId: '1234' },
  headers: { token: 'abcd' }
}).then(response => {
  console.log(response)
})
```

如果要捕获服务端抛出的异常，即返回非 200 请求，需要将代码修改为：

```js
const url = 'https://test.youbaobao.xyz:18081/book/home/v2'
axios.get(url, { 
  params: { openId: '1234' },
  headers: { token: 'abcd' }
}).then(response => {
  console.log(response)
}).catch(err => {
  console.log(err)
})
```

这样改动可以实现我们的需求，但是有两个问题：

- 每个需要传入 token 的请求都需要添加 headers 对象，会造成大量重复代码
- 每个请求都需要手动定义异常处理，而异常处理的逻辑大多是一致的，如果将其封装成通用的异常处理方法，那么每个请求都要调用一遍

### axios.create 示例

下面我们使用 axios.create 对整个请求进行重构：

```js
const url = '/book/home/v2'
const request = axios.create({
  baseURL: 'https://test.youbaobao.xyz:18081',
  timeout: 5000
})
request({
  url, 
  method: 'get',
  params: {
    openId: '1234'
  }
})
```

首先我们通过 `axios.create` 生成一个函数，该函数是 axios 实例，通过执行该方法完成请求，它与直接调用 `axios.get` 区别如下：

- 需要传入 url 参数，`axios.get` 方法的第一个参数是 url
- 需要传入 method 参数，`axios.get` 方法已经表示发起 get 请求

### axios 请求拦截器

上述代码完成了基本请求的功能，下面我们需要为 http 请求的 headers 中添加 token，同时进行白名单校验，如 `/login` 不需要添加 token，并实现异步捕获和自定义处理：

```js
const whiteUrl = [ '/login', '/book/home/v2' ]
const url = '/book/home/v2'
const request = axios.create({
  baseURL: 'https://test.youbaobao.xyz:18081',
  timeout: 5000
})
request.interceptors.request.use(
  config => {
    // throw new Error('error...')
    const url = config.url.replace(config.baseURL, '')
      if (whiteUrl.some(wl => url === wl)) {
        return config
      }
    config.headers['token'] = 'abcd'
    return config
  },
  error => {
    return Promise.reject(error)
  }
)
request({
  url, 
  method: 'get',
  params: {
    openId: '1234'
  }
}).catch(err => {
  console.log(err)
})
```

这里核心是调用了 `request.interceptors.request.use` 方法，即 axios 的请求拦截器，该方法需要传入两个参数，第一个参数是拦截器方法，包含一个 config 参数，我们可以在这个方法中修改 config 并且进行回传，第二个参数是异常处理方法，我们可以使用 `Promise.reject(error)` 将异常返回给用户进行处理，所以我们在 request 请求后可以通过 catch 捕获异常进行自定义处理

### axios 响应拦截器

下面我们进一步增强 axios 功能，我们在实际开发中除了需要保障 http statusCode 为 200，还需要保证业务代码正确，上述案例中，我定义了 error_code 为 0 时，表示业务返回正常，如果返回值不为 0 则说明业务处理出错，此时我们通过 `request.interceptors.response.use` 方法定义响应拦截器，它仍然需要2个参数，与请求拦截器类似，注意第二个参数主要处理 statusCode 非 200 的异常请求，源码如下：

```js
const whiteUrl = [ '/login', '/book/home/v2' ]
const url = '/book/home/v2'
const request = axios.create({
  baseURL: 'https://test.youbaobao.xyz:18081',
  timeout: 5000
})
request.interceptors.request.use(
  config => {
    const url = config.url.replace(config.baseURL, '')
    if (whiteUrl.some(wl => url === wl)) {
      return config
    }
    config.headers['token'] = 'abcd'
    return config
  },
  error => {
    return Promise.reject(error)
  }
)

request.interceptors.response.use(
  response => {
    const res = response.data
    if (res.error_code != 0) {
      alert(res.msg)
      return Promise.reject(new Error(res.msg))
    } else {
      return res
    }
  },
  error => {
    return Promise.reject(error)
  }
)

request({
  url, 
  method: 'get',
  params: {
    openId: '1234'
  }
}).then(response => {
  console.log(response)
}).catch(err => {
  console.log(err)
})
```

## request 库源码分析

有了上述基础后，我们再看 request 库源码就非常容易了

```js
const service = axios.create({
  baseURL: process.env.VUE_APP_BASE_API,
  timeout: 5000
})

service.interceptors.request.use(
  config => {
    // 如果存在 token 则附带在 http header 中
    if (store.getters.token) {
      config.headers['X-Token'] = getToken()
    }
    return config
  },
  error => {
    return Promise.reject(error)
  }
)

service.interceptors.response.use(
  response => {
    const res = response.data

    if (res.code !== 20000) {
      Message({
        message: res.message || 'Error',
        type: 'error',
        duration: 5 * 1000
      })
      // 判断 token 失效的场景
      if (res.code === 50008 || res.code === 50012 || res.code === 50014) {
        // 如果 token 失效，则弹出确认对话框，用户点击后，清空 token 并返回登录页面
        MessageBox.confirm('You have been logged out, you can cancel to stay on this page, or log in again', 'Confirm logout', {
          confirmButtonText: 'Re-Login',
          cancelButtonText: 'Cancel',
          type: 'warning'
        }).then(() => {
          store.dispatch('user/resetToken').then(() => {
            location.reload()
          })
        })
      }
      return Promise.reject(new Error(res.message || 'Error'))
    } else {
      return res
    }
  },
  error => {
    Message({
      message: error.message,
      type: 'error',
      duration: 5 * 1000
    })
    return Promise.reject(error)
  }
)

export default service
```

## 登录细节分析

### 细节一：页面启动后自动聚焦

检查用户名或密码是否为空，如果发现为空，则自动聚焦：

```js
mounted() {
    if (this.loginForm.username === '') {
      this.$refs.username.focus()
    } else if (this.loginForm.password === '') {
      this.$refs.password.focus()
    }
}
```

### 细节二：显示密码后自动聚焦

切换密码显示状态后，自动聚焦 password 输入框：

```js
showPwd() {
  if (this.passwordType === 'password') {
    this.passwordType = ''
  } else {
    this.passwordType = 'password'
  }
  this.$nextTick(() => {
    this.$refs.password.focus()
  })
}
```

### 细节三：通过 reduce 过滤对象属性

```js
const query = {
  redirect: '/book/list',
  name: 'sam',
  id: '1234'
}
// 直接删除 query.redirect，会直接改动 query
// delete query.redirect

// 通过浅拷贝实现属性过滤
// const _query = Object.assign({}, query)
// delete _query.redirect

const _query = Object.keys(query).reduce((acc, cur) => {
    if (cur !== 'redirect') {
      acc[cur] = query[cur]
    }
    return acc
  }, {})
console.log(query, _query)
```

## 关闭 Mock 接口

去掉 main.js 中 mock 相关代码：

```js
import { mockXHR } from '../mock'
if (process.env.NODE_ENV === 'production') {
  mockXHR()
}
```

删除 `src/api` 目录下 2 个 api 文件：

```bash
article.js
qiniu.js
```

删除 `vue.config.js` 中的相关配置：

```js
proxy: {
  // change xxx-api/login => mock/login
  // detail: https://cli.vuejs.org/config/#devserver-proxy
  [process.env.VUE_APP_BASE_API]: {
    target: `http://127.0.0.1:${port}/mock`,
    changeOrigin: true,
    pathRewrite: {
      ['^' + process.env.VUE_APP_BASE_API]: ''
    }
  }
},
after: require('./mock/mock-server.js')
```

修改后我们的项目里就不能使用 mock 接口，会直接请求到 http 接口，我们需要打开 SwitchHosts 配置 host 映射，让域名映射到本地 node 项目：

```bash
127.0.0.1	book.youbaobao.xyz
```

1

也可以直接修改 `/etc/hosts` 文件

## 修改接口地址

我们将发布到开发环境和生产环境，所以需要修改 `.env.development` 和 `.env.production` 两个配置文件：

```bash
VUE_APP_BASE_API = 'https://book.youbaobao.xyz:18082'
# VUE_APP_BASE_API = '/dev-api'
```

有两点需要注意：

- 这里我使用了域名 `book.youbaobao.xyz`，大家可以将其替换为你自己注册的域名，如果你还没注册域名，用 `localhost` 也是可行的，但如果要发布到互联网需要注册域名
- 如果没有申请 https 证书，可以采用 http 协议，同样可以实现登录请求，但是如果你要发布到互联网上建议使用 https 协议安全性会更好

重新启动项目后，发现登录接口已指向我们指定的接口：

```bash
Request URL: https://book.youbaobao.xyz:18082/user/login
```


# Express 快速开始

和 Koa 类似，Express 也是采用中间件的机制来处理响应。下面是全局响应：

```js
app.use((req, res) => {
  res.send({ ping: 'pong' })
})
```

返回的结果是会自动进行 JSON 序列化的，然而 body 中的内容需要使用内置的中间件完成解析：

```js
app.use(express.json())
```

这样在 POST PUT 等请求中携带的 JSON 可以被自动反序列化为 JavaScript 对象。

可以直接在 `app` 上调用 get post 等方法创建路由和请求处理函数：

```js
app.get('/', (req, res) => {
  res.send({ id: nanoid(), title: new Date().toString() })
})
```

## 关于 HTTP 状态码

创建资源成功的状态码建议使用 201 并且返回完整数据。

```js
app.post('/', (req, res) => {
  console.log(req.body)
  res.status(201).send({ id: nanoid(), title: new Date().toString() })
})
```

删除资源成功的状态码建议使用 204 表明没有返回内容。

```js
app.delete('/:id', (req, res) => {
  console.log(req.params.id, req.body)
  res.status(204).send()
})
```

只有调用 `res` 的 `send()` 方法，请求才真正开始发送。如果没有显示设置状态码，中途也未出现异常，那么默认的状态码为 200。

## 路由组织

调用 `app.get` 等方法设置请求处理函数是全局，不方便组织多个资源类型，因此可以使用路由来转发请求到特定路由器。

这里的路由、转发、路由器的概念要和计算机网络的概念结合起来进行理解。

```js
// 创建路由器
const router = express.Router()

// 关联请求处理函数
router.get('/', () => {})
router.post('/', () => {})

// 关联某路径到特定路由器
app.use('/prefix', router)
```

为了避免在 `app.js` 中注册路由，可以在 `routes/index.js` 中导出 `withRoutes` 方法来为 `app` 注册路由（装饰器模式）。

当然这里也可以使用声明式路由，遍历 `routes` 下的目录结构来动态导入模块来绑定路由。

## 接口设计

创建：资源的创建需要满足一定的模式，一般需要在后台做数据的校验和过滤。ID 须在创建完成之后返回。

查询：资源的查询需要一些条件，这个条件可能是复合条件，也可能是简单条件（如根据 ID 查询或特定年龄段的查询）。前端可提供条件，返回结果的类型仅有 2 种。资源和资源集合，资源在提供唯一标识时返回资源（不要局限于主键，也可以是唯一约束，甚至是复合唯一约束）；否则，返回资源的集合（可能是空集）。集合的一般表现形式为数组，如果没有展示先后顺序的要求，也可以返回对象，对象的键为主键或唯一约束。当然，查询唯一标识也返回集合是可行的，这样可以做统一处理。针对于返回集合的情形，也应该考虑到分页。分页需要前后端协作提供和处理额外的元数据，具体的方式可以封装在响应体中，也可以定义在 HTTP 头中。

修改：资源的修改分为增量更新（patch）和全量更新（put）。全量更新需要符合创建资源时的要求，原有的资源被完全覆盖，即便当前是 NULL 也会覆盖原来的数据。全量更新在实际应用中比较少见，因为增添了许多不安全的因素。增量更新则需要符合特定的子模式，子模式继承于创建资源时的模式。修改资源时需要提供一些查询条件（见「查询」）。

删除：删除资源时需要提供一些查询条件（唯一标识或条件），可根据条件进行硬删除或软删除。

## 关于设计的一些技巧

请求处理函数中的 `res` 参数可以被连续调用，很简单其实是返回了 `this`。

如果一个函数并不需要返回额外的数据，而仅仅是做配置，那么可以考虑返回 `this` 以便进行链式调用。

## 使用 HTTPie 测试

```bash
# ping
http :3000
# GET post
http :3000/post
# POST post
http POST :3000/post name='John Snow' age:=29 title=KingInTheNorth
# PATCH DELETE ...
```

https://devhints.io/httpie

## 连接到 MongoDB Atlas

找到数据库集群（Clusters），点击 CONNECT 可以获得连接到数据库的地址。地址类似

```text
mongodb+srv://<user>:<password>@<host>/<database>?retryWrites=true&w=majority
```

可以注意到这里除了 `mongodb` 协议，还有 `srv` 协议。关于 SRV 记录见：

SRV 记录 - 维基百科，自由的百科全书
https://zh.wikipedia.org/wiki/SRV%E8%AE%B0%E5%BD%95

简而言之是 DNS 中的一条记录，用于发现特定服务的端口号。

官方给出的例子：

```js
// 创建客户端的类
const { MongoClient } = require('mongodb');

// 创建客户端并配置连接参数
const uri = 'mongodb+srv://<**>';
const client = new MongoClient(uri, { useNewUrlParser: true, useUnifiedTopology: true });

// 创建连接
client.connect(err => {
  // 连接失败可处理 err
  const collection = client.db('test').collection('devices');
  // 在这里操作数据库中的集合
  client.close();
});
```

实际上除了回调函数的用法，也有返回的 Promise 的重载函数：

```typescript
connect(): Promise<MongoClient>;
connect(callback: MongoCallback<MongoClient>): void;
```

注意：如果没有调用客户端的 `close()` 方法，那么程序不会退出。因此在测试数据库操作的代码中一定要调用该方法关闭连接。

## 模型定义

根据用户删除评论：传入的参数除了路径参数还有请求体，请求体的格式为：

```json
{ "user": "Thomas" }
```

用于评论是文章的嵌套文档，并且是一个数组。所以可以这样理解，将 `comments` 中匹配规则 `{ user }` 的子文档删除。所以应该这样写：

```js
{ $pull: { comments: { user } } }
```

# OPA
https://www.openpolicyagent.org

## 1. 介绍
* 支持动态创建、更新策略；
* 已接入的服务，可以在不变更程序的前提下，使用最新的策略；
* OPA是一个轻量级的、通用的策略引擎，支持的集成方式有：sidecar、host-level daemon和library；
* 

## 2. 如何工作
![基本流程图](https://www.openpolicyagent.org/docs/images/request-response.svg)

目前OPA不支持策略更新时通知到关注的服务或用户。

### 2.1 Data 和 Policies
在OPA中，主体数据称之为document，类似于JSON字符串。可以通过OPA的RESTful接口进行增删改。
![内部组织方式](https://www.openpolicyagent.org/docs/images/data-model-dependencies.svg)
![data](https://www.openpolicyagent.org/docs/images/data-model-logical.svg)

#### 2.1.1 Base Documents
用于保存：

1. 服务相关：现有状态、服务机器ip列表等
2. 用户相关：部署信息、现有状态等

使用OPA的Data API进行管理。

#### 2.1.2 Policies
使用OPA的Rego语言进行编写，定义了一组规则。
使用OPA的Policy API进行管理。

#### 2.1.3 Rules 和 Virtual Documents
Virtual Documents 是policy中规则(rule)的计算结果， 当base document或policy发生更新时，virtual document会被及时计算更新。
Rules allow policy authors to write questions with yes-no answers (that is, predicates) and to generate structured values from raw data found in base documents as well as from intermediate data found in other virtual documents.

#### 2.1.4 The data Document
OPA中所有用户创建的、计算得到的document都嵌套在OPA内建的名为data的document。
一个 data document例子：
```
{
    "servers": [...],
    "ports": [...],
    "networks": [...],
    "opa": {
        "examples": {
            "violations": [...],
            "public_severs": [...]
        }
    }
}
```

data document下的任何document都可以从根节点开始被访问到，例如
```
import data.servers
import data.opa.examples.violations
```
也可以通过URI访问：
```
GET https://example.com/v1/data/servers HTTP/1.1
GET https://example.com/v1/data/opa/examples/violations HTTP/1.1
```
#### 2.1.5 The input Document
在某些情况下，policy需要有输入参数。OPA使用内建的名为input的document，在调用OPA的query接口时，使用设置input document中的值。
```
{
    "method": "GET",
    "path": "/servers/s2",
    "user": "alice"
}
```

同样，我们可以方便的访问input document中的值：
```
# let 'bob' perform read-only operations
allow {
    input.user = "bob"
    input.method = "GET"
}

# let 'alice' perform any operation
allow {
    input.user = "alice"
}
```

## 3. 如何写policy
使用Rego，原生查询语言(native query language)
Rego 受 [Datalog](https://en.wikipedia.org/wiki/Datalog) 启发，对其进行了扩展支持结构化的文档模式例如JSON。

### 3.1 基础知识
```
pi = 3.14159
> pi
3.14159

rect = {"width": 2, "height": 4}
> rect
{
    "height": 4,
    "width": 2
}

# 表达式：rule-name IS value IF body
# if the value is omitted, it defaults to true
v {"hello" = "world"}
# check if it is equal to true:
> v = true
false

t {x = 42; y = 41; x > y}
# 等价于，使用换行，而非分号
t {
    x = 42
    y = 41
    x > y
}
# 等价于，语句顺序无关
t {
    x > y
    x = 41
    y = 42
}

# references
sites = [{"name": "prod"}, {"name": "smoke1"}, {"name": "dev"}]
r { sites[i].name = "prod"}
> r
true

# 定义一个set document
q[name] { sites[i].name = name }
> q[x]
+----------+
|    x     |
+----------+
| "prod"   |
| "smoke1" |
| "dev"    |
+----------+

p { q["prod"] }
> p
true
> q["smoke2"]
undefined
```

### 3.2 Scalar Values
可以是字符串、数字、布尔值、null
```
greeting = "hello"
max_height = 42
pi = 3.14159
allowed = true
sentinel = null
```

### 3.3 Strings
支持2种方式：
1. 使用双引号，需要转义
2. 使用反引号 ` , 无视转义符，适用于正则表达式。称之为 raw string。

### 3.4 Composite Values
用于定义collections
```
cube = {"width": 3, "height": 4, "depth": 5}

a = 42; b = false; c = null; d = {"a": a, "x": [b,c]}
```

#### 3.4.1 Sets
```
s = {cube.width, cube.height, cube.depth}

# 空set 的定义方法
set()
```

### 3.5 Variables
出现在规则(rule)的head和body部分。

* 如果出现在head，variable可被同时当作输入和输出。如果已赋予值，那么被用作输入，如果没有赋予值，那么用作输出。

### 3.6 References
Reference用于访问嵌套document。

```
> sites[0].servers[1].hostname
"helium"
> sites[0]["servers"][1]["hostname"]
"helium"
```

以下4种情况必须使用方括号
1. string key包含了非字母、数字 或 _
2. 非string key，例如是数字、null、布尔值
3. variable key
4. composite key 

#### 3.6.1 Variable Keys
用于遍历collection中的每一个元素。

```
> sites[i].servers[j].hostname
+---+---+------------------------------+
| i | j | sites[i].servers[j].hostname |
+---+---+------------------------------+
| 0 | 0 | "hydrogen"                   |
| 0 | 1 | "helium"                     |
| 0 | 2 | "lithium"                    |
| 1 | 0 | "beryllium"                  |
| 1 | 1 | "boron"                      |
| 1 | 2 | "carbon"                     |
| 2 | 0 | "nitrogen"                   |
| 2 | 1 | "oxygen"                     |
+---+---+------------------------------+

# 如果i、j在后续的代码中不会被使用到，可以使用 _
> sites[_].servers[_].hostname
+------------------------------+
| sites[_].servers[_].hostname |
+------------------------------+
| "hydrogen"                   |
| "helium"                     |
| "lithium"                    |
| "beryllium"                  |
| "boron"                      |
| "carbon"                     |
| "nitrogen"                   |
| "oxygen"                     |
+------------------------------+
```

#### 3.6.2 Composite Keys
Composite key 只适用于virtual document，并不适用base data documents。

一般使用场景，
* 检查collection中是否有这个key
* 按照某种模式提取出对应的值

```
> s = {[1,2],[1,4],[2,6]}
> s[[1,2]]
[
    1,
    2
]

> s[[1,x]]
+---+
| x |
+---+
| 2 |
| 4 |
+---+
```

#### 3.6.3 Multiple Expressions
```
apps_and_hostnames[[name, hostname]] {
    apps[i].name = name
    apps[i].servers[_] = server
    sites[j].servers[k].name = server
    sites[j].servers[k].hostname = hostname
}
```
2点说明：
* 一个variable出现在多个地方，OPA只会将该variable赋予同一个值。
* 上述的rule，join了apps和sites。所有的join 在Rego中隐式的。

#### 3.6.4 Self-Joins
Using a different key on the same array or object provides the equivalent of self-join in SQL.

### 3.7 Comprehensions
Comprehension 提供了一个准确的方法用于从query构建一个Composite Values。

和rule一样，comprehension也分为head和body。
* body，和rule的body一样，其中的表达式都必须为true
* body中的表达式可以引用外部声明的variable

```
> region = "west"; names = [name | sites[i].region = region; sites[i].name = name]
+-----------------+--------+
|      names      | region |
+-----------------+--------+
| ["smoke","dev"] | "west" |
+-----------------+--------+
```

#### 3.7.1 Array Comprehensions
用于构建array values，形式如
```
[ <term> | <body>]

#
hostnames = [hostname | name = app.servers[_]
                        sites[_].servers[_] = s
                        s.name = name
                        hostname = s.hostname]
```

#### 3.7.2 Object Comprehensions
用于构建object values
```
{ <key>: <term> | <body>}

#
app_to_hostnames = {app.name: hostnames |
    apps[_] = app
    hostnames = [hostname |
                    name = app.servers[_]
                    sites[_].servers[_] = s
                    s.name = name
                    hostname = s.hostname]
}
```

#### 3.7.3 Set Comprehensions
用于构建set values
```
{ <term> | <body>}

#
> a = [1,2,3,4,3,4,3,4,5]
> b = {x | x = a[_]}
> b
[1,2,3,4,5]
```

### 3.8 Rules
Rules 用于定义virtual documents的内容。

#### 3.8.1 Generating Sets

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


Composite 
cube = { "width": 3, "height": 4, "depth": 5}

Variables
    They appear in both the head and body of rules.
    Variables appearing in the head of a rule can be thought of as input and output of the rule. They are simultaneously an input and an output.
    If a query supplies a value for a variable, that variable is an input, and if the query does not supply a value for a variable, that variable is an output.
sites = [
    {"name": "prod"},
    {"name": "smoke1"},
    {"name": "dev"}
]

q[name] { sites[i].name = name }

>q[x]
+----------+
| x |
+----------+
| "prod" |
| "smoke1" |
| "dev" |
+----------+

> q["smoke2"]
undefined
> q["dev"]
true

Variable Keys
    References can include variables as keys. References written this way are used to select a value from every element in a collection.
> sites[i].servers[j].hostname
+---+---+------------------------------+
| i | j | sites[i].servers[j].hostname |
+---+---+------------------------------+
| 0 | 0 | "hydrogen" |
| 0 | 1 | "helium" |
| 0 | 2 | "lithium" |
| 1 | 0 | "beryllium" |
| 1 | 1 | "boron" |
| 1 | 2 | "carbon" |
| 2 | 0 | "nitrogen" |
| 2 | 1 | "oxygen" |
+---+---+------------------------------+

> sites[_].servers[_].hostname
+------------------------------+
| sites[_].servers[_].hostname |
+------------------------------+
| "hydrogen" |
| "helium" |
| "lithium" |
| "beryllium" |
| "boron" |
| "carbon" |
| "nitrogen" |
| "oxygen" |
+------------------------------+

Composite Keys
    References can include Composite Values as keys if the key is being used to refer into a set. Composite keys may not be used in refs for base data documents, they are only valid for references into virtual documents.
> s = {[1, 2], [1, 4], [2, 6]}
> s[[1, 2]]
[
1,
2
]
> s[[1, x]]
+---+
| x |
+---+
| 2 |
| 4 |
+---+

Multiple Expresssions
    
apps_and_hostnames[[name, hostname]] {
    apps[i].name = name
    apps[i].servers[_] = server
    sites[j].servers[k].name = server
    sites[j].servers[k].hostname = hostname
}

> apps_and_hostnames[x]
+----------------------+
| x |
+----------------------+
| ["web","hydrogen"] |
| ["web","helium"] |
| ["web","beryllium"] |
| ["web","boron"] |
| ["web","nitrogen"] |
| ["mysql","lithium"] |
| ["mysql","carbon"] |
| ["mongodb","oxygen"] |
+----------------------+

Comprehensions
    Comprehensions provide a concise way of building Composite Values from sub-queries.
    由 head 和 body 两部分组成。
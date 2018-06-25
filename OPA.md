# OPA
https://www.openpolicyagent.org

## 介绍
* 支持动态创建、更新策略；
* 已接入的服务，可以在不变更程序的前提下，使用最新的策略；
* OPA是一个轻量级的、通用的策略引擎，支持的集成方式有：sidecar、host-level daemon和library；
* 

## 如何工作



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
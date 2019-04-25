---
title: GraphQL学习
tags:
  - GraphQL
categories:
  - GraphQL
date: '2019-04-25 19:53'
abbrlink: 42393
---

GraphQL 既是一种用于 API 的查询语言也是一个满足你数据查询的运行时。 GraphQL 对你的 API 中的数据提供了一套易于理解的完整描述，使得客户端能够准确地获得它需要的数据，而且没有任何冗余，也让 API 更容易地随着时间推移而演进，还能用于构建强大的开发者工具。

<!-- more -->

## 对象类型和字段

    type Character {
      name: String!
      appearsIn: [Episode!]!
    }

- **Character** 是一个 GraphQL 对象类型，表示其是一个拥有一些字段的类型。你的 schema 中的大多数类型都会是对象类型。
name 和 appearsIn 是 Character 类型上的字段。这意味着在一个操作 Character 类型的 GraphQL 查询中的任何部分，都只能出现 name 和 appearsIn 字段。
- **String** 是内置的标量类型之一 —— 标量类型是解析到单个标量对象的类型，无法在查询中对它进行次级选择。后面我们将细述标量类型。
- **String!** 表示这个字段是非空的，GraphQL 服务保证当你查询这个字段后总会给你返回一个值。在类型语言里面，我们用一个感叹号来表示这个特性。
- **[Episode!]!** 表示一个 Episode 数组。因为它也是非空的，所以当你查询 appearsIn 字段的时候，你也总能得到一个数组（零个或者多个元素）。且由于 Episode! 也是非空的，你总是可以预期到数组中的每个项目都是一个 Episode 对象。

## 标量类型
- **Int**：有符号 32 位整数。
- **Float**：有符号双精度浮点值。
- **String**：UTF‐8 字符序列。
- **Boolean**：true 或者 false。
- **ID**：ID 标量类型表示一个唯一标识符，通常用以重新获取对象或者作为缓存中的键。ID 类型使用和 String 一样的方式序列化；然而将其定义为 ID 意味着并不需要人类可读型。

## 基本查询
~~~
type Account {   // 定义一个接收返回的类型，包含返回的所需要的字段
    name: String // 返回的字段名称及类型
    age: Int
    sex: String
    department: String
}
type Query {   // 定义查询函数及返回类型
    hello: String  // key是函数， value是标准类型或者上述定义的类型
    accountName: String
    age: Int
    account: Account  // 此时返回是上述定义的类型
}
~~~

## 参数传递
~~~
type Account {
    name: String
    age: Int
    sex: String
    department: String
    salary(city: String): Int  // 左侧可以还是函数
}
type Query {
    getClassMates(classNo: Int!): [String] // classNo是参数名， Int是参数类型 !表示必传参数 [string]表示返回的是string数组
    account(username: String): Account
}
~~~
## 查询示例
~~~
{
"operationName":"getuserid",  // 去掉也是ok的
"variables":{   // 这里是变量的key value， key可以不用，但是query中组要的key必须存在
    "id":"1",
    "date":"2019.03.26",
    "start":"1553563272",
    "end":"1553566872",
    "limit":20 // 多余的字段不影响
},
"query":"query getuserid($id: ID!){  // 执行一个query 这里的 getuserid 可以是任意名字，没有限制， 参数里面制定类型
    getUser(id : $id) { // 这个是query里面声明的函数及参数形式
        id,  // 这里是需要的返回字段，也可以是声明的type，但是字段必须被包含在Account中
        name
    }
}"
}
~~~
## Mutation(变更)
### 示例
~~~
input AccountInput { // 这是输入类型, 使用input定义
    name: String
    age: Int
    sex: String
    department: String
}
type Account {  // 定义输出类型
    name: String
    age: Int
    sex: String
    department: String
}
type Mutation {
    createAccount(input: AccountInput): Account
    updateAccount(id: ID!, input: AccountInput): Account
}
type Query {
    accounts: [Account]
}

~~~
### 请求示例
~~~
{
"variables":{   // 这里是变量的key value， key可以不用，但是query中组要的key必须存在
    "name":"test",
    "age":18,
    "sex":"male",
    "department":"luban",
},
"mutation":"mutation createAccount($input: AccountInput){  // 执行一个query 这里的 getuserid 可以是任意名字，没有限制， 参数里面制定类型
    createAccount(input : $input) {
        age,
        name
    }
}"
}

~~~
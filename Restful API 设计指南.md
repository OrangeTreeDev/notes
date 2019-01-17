# 概述

目前大多数系统，客户端和服务端之间是通过HTTP API 交换数据，而编写服务端API并没有统一的标准规范。如果公司和组织内部有一套API设计规范，不仅可以提高开发人员的编码质量，而且可以对外输出格式一致的API文档。Restful API设计规范是被广泛使用的API设计规范的一种，本文对Restful API规范倡导的一些基本原则做阐述。

# HTTP URL

在Restful API中，URL表示一个远程资源。例如 `/topics` 表示话题这一类资源。

资源名使用名词，而非动词；

```
# 以下的资源命名方式是不推荐的
/getAllTopics
/createTopic
/updateTopic
/deleteTopic
```

资源名使用复数形式；

```
# 资源集合
/topics
# 具体某一个资源
/topics/12
```

# HTTP Methods

使用HTTP方法表示对资源的操作。利用HTTP 中的GET，POST，PUT，PATCH和DELETE，基本可以完成对资源的CRUD操作。

```
# 读取话题列表
GET /topics
# 读取id为12的资源
GET /topics/12
# 创建资源
POST /topics
# 更新id为12的资源
PUT /topics/12
# 删除id为12的资源
DELETE /topics/12
```

接下来，详细讲解HTTP Method的语义；

- GET
  - 幂等（重复地执行相同的请求，每一次请求对资源带来的状态变化都是相同的）；
  - 只读的。GET不会改变服务端资源的状态。

- PUT
  - 幂等
  - PUT用来更新资源。需要强调是，PUT是对资源的所有字段进行更新，因此，请求体（request body）中需要包含资源的所有字段和相应的值。PUT请求体中缺少的字段应该被视为空值，并清空数据库中对应记录的字段值 。

- POST
  - 非幂等
  - POST表示执行创建新的资源操作。多个POST请求会创建多个不同的资源，因此，POST不是幂等操作。通常，POST请求处理成功后，会使用响应头Location返回创建成功的资源路径（例如：Location: /topics/13）。

- PATCH
  - 幂等
  - PATCH表示对资源执行部分更新操作，请求体中只包含资源需要修改的字段，其他不被包含的字段的值不受影响。

- DELETE
  - 幂等
  - DELETE指示执行删除资源操作

# Query String

通过GET方法读取我们应该尽可能的保持URL的简洁。对于资源的筛选，排序或者高级搜索操作（比如，查找指定类型的话题资源）可以通过查询字符串（Query String）实现。

例如，服务端有`/topics`资源，我们仅仅对it类型的话题感兴趣，请求/topics指定查询参数type的值为it；

```
# 查询it类型的资源
GET /topics?type=it
```

再比如，我们想对获取的资源进行排序，可以添加一个sort查询参数；

```
# 查询it类型的资源，并且按照创建时间降序排列
GET /topics?type=id&sort=-createTime
```

# HTTP Body

## JSON

不论是HTTP Request Body，还是HTTP Response Body，数据内容都使用JSON格式传递，也就是说 `Content-Type` 设置为 `application/json`。

```
# 创建新的topic
POST /topics
Content-Type: application/json

# 请求体
{
  "authorId": "122",
  "type": "it",
  "content": "content",
  "title": "一起学 Node.js",  
}
```



## HTTP响应使用data字段

有些时候，HTTP响应结果不仅包含资源本身的信息，还可能包含其他元数据（分页数据，错误数据，链接等），因此，为了保持响应体结构的统一，把返回的实际数据设置为data字段。

```
# 获取topic
GET /topics

# 响应结果
{
	data: [
        {
        	"topicId": "12",
        	"authorId": "122",
        	"type": "it",
        	"content": "node",
        	"title": "一起学 Node.js",   
        },
        {
        	"topicId": "13",
             "authorId": "123",
        	"type": "it",
        	"content": "js",
        	"title": "一起学JavaScript",
        }
	]

}
```



## 分页

一次性返回所有的资源数据是一种很差的设计方式，通常，建议提供分页的功能。有两种常用的分页实现方法：

- 基于偏移量的分页（offset-based pagination）

这种分页方法接受两个参数：

1. offset：表示偏移量，默认值为0；
2. pageSize：当前页面获取资源最大数量，建议设置一个默认值，避免返回全部资源

同时，响应体数据中，除了资源本身数据，还应该包括分页的元数据，最好也包括前一页和后一页资源的URL

```
# 返回从第30条开始，39条结束的共10条数据
GET /topics?offset=30&pageSize=10

# 返回结果
{
    pagination: {
        offset: 30,
        limit: 10,
        total: 100
    },
    data: [
        {
        	"_id": "13",
             "authorId": "123",
        	"type": "it",
        	"createTime": "1547724974000",
        	"title": "一起学JavaScript",
        },
        {
        	"_id": "14",
             "authorId": "123",
        	"type": "it",
        	"createTime": "1547724984000",
        	"title": "一起学Node.js",
        }
    ],
    links: {
    	next: "http://www.domain.com/topics?offset=40&pageSize=10",
        prev: "http://www.domain.com/topics?offset=20&pageSize=10"
    }
}
```

但是，这种分页方式有个弊端就是，当前offset索引变的很大的时候，`sql`查询变慢。

- 基于索引列的分页

这种分页技术需要数据表包含一个时间戳字段，例如topics的`createTime`。每一次查询，都需要把上一次查询得到的最后一条数据的`createTime`作为本次查询的依据。

```
# 获取最早的20条记录（根据createTime排序）
GET /topics？pageSize=20

# 假设第一次查询的最后一条记录的createTime为 1504224000000
# 第二次查询会获取自从 1504224000000标记的时间开始，最近的20条数据
GET /topics？createSine=1504224000000&pageSize=20
```

建议响应体包含上一次查询记录的最后一条记录的`createTime`，以及拼装好的下一次请求的URL；

```
# 返回结果
{
    pagination: {
        lastTime: 1504224000000,
    },
    data: [
        {
        	"_id": "13",
             "authorId": "123",
        	"type": "it",
        	"createTime": "1547724974000",
        	"title": "一起学JavaScript",
        },
        {
        	"_id": "14",
             "authorId": "123",
        	"type": "it",
        	"createTime": "1547724984000",
        	"title": "一起学Node.js",
        }
    ],
    links: {
    	next: "http://www.domain.com/topics?createSine=1547724984000&pageSize=20,
    }
}
```

# HTTP Status Code

HTTP响应应该充分利用HTTP Status code表示HTTP的响应状态。

下面列表描述最常用的code，

- 2xx：成功
  - 200 - GET读取资源或者PUT，PATCH修改资源成功；
  - 201 - POST创建资源成功；
  - 204 - DELETE删除资源成功。
- 3xx：重定向
  - 301 - 资源之前存在过，现在已经被永久移除；
  - 304 - 自从上次请求获取资源之后，资源仍未被修改，表示可以使用缓存中的资源。
- 4xx：客户端错误
  - 400 - 错误请求，例如查询参数错误或者请求体格式错误；
  - 401 - 客户端未授权，表示客户端未提供有效的认证信息；
  - 403 - 被禁止访问，表示客户端认证通过，但是没有权限访问资源；
  - 404 - 未发现，表示资源不存在；
  - 410 - 已丢失，表示请求的资源已经过时，通常用于指示客户端请求API是旧版的。
- 5xx：服务端错误
  - 500 - 服务器内部错误。

# Errors

Restful API不仅要充分利用HTTP状态码表述HTTP响应状态，而且需要提供易于API使用者阅读的错误信息。典型地，API错误分为两大类，`4xx`表示客户端的错误，`5xx`表示服务端错误。

建议，API的错误响应体包含三个字段：

- code：唯一的错误码。方便API使用者根据文档查阅更详细的信息；
- message：简明扼要的错误描述；
- description：更详细的错误描述。

`4xx`这一类客户端错误，通常是因为客户端请求参数不符合要求导致，建议`description`字段使用数组，每个数组元素表示字段的报错信息。

```

{
  "code" : 1024,
  "message" : "Validation Failed",
  "errors" : [
    {
      "code" : 5432,
      "field" : "first_name",
      "message" : "First name cannot have fancy characters"
    },
    {
       "code" : 5622,
       "field" : "password",
       "message" : "Password cannot be blank"
    }
  ]
}
```

服务端产生的错误信息，因为可能存在敏感信息，建议开发环境直接暴露错误信息，生产环境使用`Internal Server Error`替代。

```
{
  "message" : "Internal Server Error"
}
```

# 参考地址

https://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api

https://blog.philipphauer.de/restful-api-design-best-practices/

title: Restful API设计

tags:

  - SpringBoot

categories:

  - SpringBoot
  - Restful API设计

---
### 一、设计原则
##### 1. 版本
最适合放置版本号的位置URL中，或者是头信息(HTTP Headers)中在 Accept 段中使用自定义类型(content type)与其他元数据(metadata)一起提交。
<!--more-->
```
https://api.example.com/v1/
或
Accept: application/vnd.heroku+json; version=3
```
##### 2. 提供 Request-Id
为每一个请求响应包含一个Request-Id字段，并使用UUID作为该值。通过在客户端、服务器或任何支持服务上记录该值，它能主我们提供一种机制来跟踪、诊断和调试请求。

### 二、路径

##### 1. 行为（actions）
好的末尾不需要为资源指定特殊的行为，但在特殊情况下，为某些资源指定行为却是必要的。为了描述清楚，在行为前加上一个标准的actions：
`/resources/:resource/actions/:action`
如：
`/runs/{run_id}/actions/stop`

##### 2. 过滤信息
```
?limit=10：指定返回记录的数量
?offset=10：指定返回记录的开始位置。
?page=2&per_page=100：指定第几页，以及每页的记录数。
?sortby=name&order=asc：指定返回结果按照哪个属性排序，以及排序顺序。
?animal_type_id=1：指定筛选条件
```
### 三、响应

##### 1. 状态码
```
201 CREATED - [POST/PUT/PATCH]：用户新建或修改数据成功。
202 Accepted - [*]：表示一个请求已经进入后台排队（异步任务）
204 NO CONTENT - [DELETE]：用户删除数据成功。
401 Unauthorized - [*]：表示用户没有权限（令牌、用户名、密码错误）。
403 Forbidden - [*] 表示用户得到授权（与401错误相对），但是访问是被禁止的。
406 Not Acceptable - [GET]：用户请求的格式不可得（比如用户请求JSON格式，但是只有XML格式）。
410 Gone -[GET]：用户请求的资源被永久删除，且不会再得到的。
```

title: OAuth2 （一） - 简介

tags:
  - SpringBoot

categories:
  - SpringBoot
---
## 应用场景
- 资源拥有者（你自己）打算通过qq登录今日头条；
- 打开客户端（今日头条），点击qq登录，客户端要求用户给予授权；
- 同意给予客户端授权；
- 客户端使用上一步获得的授权，向认证服务器（腾讯认证服务器）申请令牌；
- 认证服务器对客户端进行认证以后，确认无误，同意发放令牌；
- 客户端使用令牌，向资源服务器（腾讯资源服务器）申请获取资源；
- 资源服务器确认令牌无误，同意向客户端开放资源（今日头条拿到了你的头像、昵称等信息）。
下图是流程图：

![](/img/springboot/oauth.png)

## 客户端的授权模式
客户端必须得到用户的授权（authorization grant），才能获得令牌（access token）。OAuth 2.0定义了四种授权方式。
- 授权码模式（authorization code）
- 简化模式（implicit）
- 密码模式（resource owner password credentials，常用）
- 客户端模式（client credentials，常用）

主要介绍后两种模式，如果系统已经有了一套用户体系，每个用户也有了一定的权限，可以采用密码模式；如果仅仅是接口的对接，不考虑用户，则可以使用客户端模式。

### 1. 密码模式
在这种模式中，系统本身有一套用户体系，在认证时需要带上自己的用户名和密码，以及客户端的client_id,client_secret。此时，accessToken所包含的权限是用户本身的权限，而不是客户端的权限。这通常用在用户对客户端高度信任的情况下，一般用于内部系统，比如前后端分离的登录实现，或者微服务架构不同服务之间的认证获取。

**步骤：**
- 用户向客户端提供用户名和密码；
- 客户端将用户名和密码发给认证服务器，向后者请求令牌；
- 认证服务器确认无误后，向客户端提供访问令牌。

**举例：**

```js
//Request:
//grant_type：表示授权类型，值固定为password，必选项；
//username：表示用户名，必选项;
//password：表示用户的密码，必选项；
//scope：表示权限范围，可选项。
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=password&username=johndoe&password=A3ddj3w

//Response:
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "access_token":"2YotnFZFEjr1zCsicMWpAA",
  "token_type":"example",
  "expires_in":3600,
  "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
  "example_parameter":"example_value"
}
```
### 2. 客户端模式
客户端模式（Client Credentials），用户直接向客户端注册，客户端以自己的名义要求"服务提供商"提供服务，其实不存在授权问题。用配置中的客户端信息去申请accessToken，客户端有自己的client_id,client_secret对应于用户的username,password，而客户端也拥有自己的authorities，当采取client模式认证时，对应的权限也就是客户端自己的authorities。

**步骤：**
- 客户端向认证服务器进行身份认证，并要求一个访问令牌；
- 认证服务器确认无误后，向客户端提供访问令牌。

**举例：**

```js
//Request:
//granttype：表示授权类型，此处的值固定为"clientcredentials"，必选项;
//scope：表示权限范围，可选项。
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials

//Response:
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "access_token":"2YotnFZFEjr1zCsicMWpAA",
  "token_type":"example",
  "expires_in":3600,
  "example_parameter":"example_value"
}
```

### 3. Refresh Token
如果用户访问的时候，客户端的access token已经过期，则需要使用refresh token申请一个新的access token。

**举例：**

```js
//granttype：表示使用的授权模式，此处的值固定为"refreshtoken"，必选项；
//refresh_token：表示早前收到的更新令牌，必选项；
//scope：表示申请的授权范围，不可以超出上一次申请的范围，如果省略该参数，则表示与上一次一致。
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
```

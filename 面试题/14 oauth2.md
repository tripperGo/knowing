## 1 定义

`oAuth`是一个关于授权的开放网络标准协议，用来授权第三方应用，获取用户的数据。其最终的目的是为了给第三方应用颁发一个有时效性的令牌`access_token`，第三方应用根据这个`access_token`就可以去获取用户的相关资源。

## 2 授权方式

在oAuth2当中，定义了四种授权方式，针对不同的业务场景：

- 授权码模式(authorization code)： 流程最完整和严密的一种授权方式，服务器和客户端配合使用，主要是针对web服务器的情况采用
- 简化模式(implicit)：主要用于移动应用程序或纯前端的web应用程序，主要是针对没有web服务器的情况采用
- 密码模式(resource owner password credentials)：不推荐，用户需要向客户端提供自己的账号和密码，如果客户端是自家应用的话，也是可以的
- 客户端模式(client credentials)：客户端以自己的名义，而不是用户的名义，向“服务提供商”进行认证，如微信公众号以此access_token来拉取所有已关注用户的信息，docker到dockerhub拉取镜像等

### 2.1 协议流程

Oauth2协议中有几种不同的角色：

- Resource Owner，资源所有者，因为是请求用户的头像和昵称的一些信息，所以资源的所有者一般指用户自己。

- Client，客户端，如web网站，app等

- Resource Server，资源服务器，托管受保护资源的服务器

- Authorization Server，授权服务器，一般和资源服务器是同一家公司的应用，主要是用来处理授权，给客户端颁发令牌

- User-agent，用户代理，一般为web浏览器，在手机上就是app

```java
 +--------+                               +---------------+
 |        |--(A)- Authorization Request ->|   Resource    |
 |        |                               |     Owner     |
 |        |<-(B)-- Authorization Grant ---|               |
 |        |                               +---------------+
 |        |
 |        |                               +---------------+
 |        |--(C)-- Authorization Grant -->| Authorization |
 | Client |                               |     Server    |
 |        |<-(D)----- Access Token -------|               |
 |        |                               +---------------+
 |        |
 |        |                               +---------------+
 |        |--(E)----- Access Token ------>|    Resource   |
 |        |                               |     Server    |
 |        |<-(F)--- Protected Resource ---|               |
 +--------+                               +---------------+
```

- 用户打开客户端(Client)，客户端向授权服务器(Resource Owner)发送一个授权请求

- 用户同意给客户端(Client)授权

- 客户端使用刚才的授权去向认证服务器(Authorization Server)认证

- 认证服务器认证通过后，会给客户端发放令牌(Access Token)

- 客户端拿着令牌(Access Token)，去向资源服务器(Resource Server)申请获取资源

- 资源服务器确认令牌之后，给客户端返回受保护的资源(Protected Resource)



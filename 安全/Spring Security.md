## Spring Security使用

1. 引入相应依赖

   ```
   spring-boot-starter-security
   ```

2. 写一个配置类，继承WebSecurityConfigurerAdapter

   开启@EnableWebSecurity，

3. 在配置类中配置HttpSecurity登陆认证的规则

   添加拦截路径，登出的页面，登出的Handler，以及登陆和鉴权的Filter

4. 配置AuthenticationManagerBuilder

   写一个Impl类实现UserDetailsService，查询数据到对应Security中的实体类

   设置userDetailService和passwordEncoder

## Spring Security原理

 过滤器调用链路

> SecurityContextPersistenceFilter
>
> **UsernamePasswordAuthenticationFilter**
>
> **BasicAuthenticationFilter**
>
> ExceptionTranslationFilter
>
> FilterSecurityInterceptor

它的原理就是一串的过滤器链，中间两个加粗的就是我们进行认证的，输入用户名和密码，**UsernamePasswordAuthenticationFilter**这个过滤器就会查找请求中是否有它要验证的消息，如果通过验证了，就会在请求放置一个标记，继续向下传，经过一系列的过滤器过后到达最后一个过滤器FilterSecurityInterceptor，如果通过就会转发到后面的REST服务调用API，如果失败的话就调用前方的ExceptionTranslationFilter抛出异常。多个请求的认证结果就是在SecurityContextPersistenceFilter中去存取的，其实就是存取session的过程 

如果它没有提供我们想要的认证方式，我们也可以自己写一个Filter把它加在过滤链上



## 认证授权实现思路

如果系统的模块众多，每个模块都需要就行授权与认证，所以我们选择基于token的形式进行授权与认证，用户根据用户名密码认证成功，然后**获取当前用户角色的一系列权限值**，并以用户名为key，权限列表为value的形式存入redis缓存中，根据用户名相关信息生成token返回，浏览器将token记录到cookie中，每次调用api接口都默认将token携带到header请求头中，Spring-security解析header头获取token信息，解析token获取当前用户名，根据用户名就可以从redis中获取权限列表，这样Spring-security就能够判断当前请求是否有权限访问

 当点击注销后，会执行LogoutHandler中的logout方法，在TokenManager中删除token，并清除redis缓存

## 单点登录SingleSignOn

### 方案1：Session共享

Session中有sid唯一标识当前用户，以及数据

在B/S架构下，浏览器存储Cookie存储sid，浏览器同源策略，Session不共享

Session复制，在容器中使用拦截器，复制到另外的系统，先同步再返回，每次Session修改需要各个系统之间来回同步

使用Tomcat插件可以实现

### 方案2：Redis+cookie

Session不复制，公用

1. Redis中，key：生成唯一id值（ip、用户id），value：用户数据

2. Cookie中，保存redis生成的key

把Session放入第三方存储，但是不同系统的入口不一致，需要跨域访问，使用反向代理，负载均衡，网关服务，所有用户使用一个域名，根据uri分发不同服务

适用于流量比较少

### 方案3：token

使用无状态Session

不把Session存入数据库中（Redis），只去数据库中校验帐号，把sid置为有意义的唯一标识userId，也就是token，服务器端使用信息生成签名，防止

Token就是按照一定规则(JWT制定好了一套规则)生成的字符串，可以包含用户信息

1. 在项目某个模块进行登录，登录之后，按照规则生成字符串，把登录之后用户包含到生成字符串里面，把字符串返回

   (1) 把token通过cookie返回

   (2) 把字符串通过地址栏返回

2. 再去访问项目其他模块，每次访问在地址栏带着生成字符串，在访问块里获取地址栏字符串，根据字符串获取用户信息。如何获取到，就是登录

 

## JWT

Json web token (JWT)，该对象为一个很长的字符串，字符之间通过"."分隔符分为三个子串。

### header头部

- 声明类型，这里是jwt
- 声明加密的算法，通常直接使用 HMAC SHA256

然后使用base64加密

### payload载荷

载荷就是存放有效的主体信息的地方。

然后使用base64加密

### signature签证

签名哈希部分是对上面两部分数据签名，通过指定的算法生成哈希，以确保数据不会被篡改。

- header (base64后的)
- payload (base64后的)
- secret

首先，需要指定一个私钥（secret），该私钥仅仅为保存在服务器中，并且不能向用户公开。然后，使用标头中指定的签名算法（默认情况下为HMAC SHA256）根据以下公式生成签名。

```
HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(claims), secret)
```

在计算出签名哈希后，JWT头，有效载荷和签名哈希的三个部分组合成一个字符串，每个部分用"."分隔，就构成整个JWT对象。



问题：

\1. Token被盗？

Https(tls、ssl)使用密文传输，一般无法盗取

\2. 盐变化了会怎么样？

Token会失效

\3. 谁知道盐？

SpringCloudConfig是基于Git（pull下来test、dev、prod），Dev和Prod会有不同的盐

\4. 如何防止程序员盗取盐？

System.out，日志记录，一些敏感变量，使用代码审查

\5. Jwt如何续期？

两端配合，后端和前端

\6. 如何踢用户下线？

同一入口，

 

## 开放系统权限方法

### 方式一：用户名密码复制

适用于同一公司内部的多个系统，不适用于不受信的第三方应用

### 方式二：通用开发者key

适用于合作商或者授信的不同业务部门之间

### 方式三：办法令牌

接近OAuth2方式，需要考虑如何管理令牌、颁发令牌、吊销令牌，需要统一的协议，因此就有了OAuth2协议



## OAuth2

解决两个问题：

1. 开放系统权限
2. 分布式访问

![](../imgs\OAuth2.jpg)

## 跨域

### 什么是跨域？

浏览器从一个域名的网页去请求另一个域名的资源时，域名、端口、协议任一不同，都是跨域

### 跨域的条件



### 为什么会有跨域？

在前后端分离的模式下，前后端的域名是不一致的，此时就会发生跨域访问问题。在请求的过程中我们要想回去数据一般都是post/get请求，所以..跨域问题出现

跨域问题来源于JavaScript的同源策略，即只有 协议+主机名+端口号(如存在)相同，则允许相互访问。也就是说JavaScript只能访问和操作自己域下的资源，不能访问和操作其他域下的资源。跨域问题是针对JS和ajax的，html本身没有跨域问题，比如a标签、script标签、甚至form标签（可以直接跨域发送数据并接收数据）等



## 微信登陆过程

1. 生成二维码

2. 当扫描二维码后，跳转到callback，后端得到code值，使用code请求微信固定地址得到access_token和openid

3. 再拿着accsess_token 和 openid，再去请求微信提供固定的地址，得到扫描人的信息

4. 使用JWT把扫描人的信息封装成token字符串

5. 重定向至首页，并附带token字符串

   

## 微信支付过程

1. 前端点击支付生成二维码
2. 后端接口查询得到订单的信息
3. 设置xml格式的信息，HTTPclient调用微信支付固定地址
4. 得到返回的xml格式二维码地址、以及操作状态码，返回前端
5. 前端开启定时任务，每隔几秒查询订单支付状态
6. 支付成功后，更新订单状态
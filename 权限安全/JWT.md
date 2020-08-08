### JWT

#### 什么是JWT

[JWT](https://jwt.io/introduction/)(Json Web Token)是为了在网络应用环境间传递声明而执行的一种基于Json的开放标准(RFC 7519)。JWT的声明一般被用来在身份提供者和服务提供者间传递被认证的用户身份信息，以便于从资源服务器获取资源。

#### JWT使用场景

- **授权**（`Authorization`）
  - 比如用户在登陆时，提供用户名和密码给认证服务器，认证服务器验证用户提交信息的合法性；如果验证成功，认证服务器会产生并返回一个`Token`，用户可以携带`Token`，访问服务器上手保护的资源。单点登录是现在广泛使用的JWT的一个特性，因为它的开销很小，并且可以轻松地跨域使用。
- **信息交换**（`Information Exchange`）
  - 对于安全的在各方之间传输信息而言，JSON Web Tokens无疑是一种很好的方式。因为JWTs可以被签名，例如，用公钥/私钥对，你可以确定发送人就是它们所说的那个人。另外，由于签名是使用头和有效负载计算的，您还可以验证内容没有被篡改。

#### JWT 的结构

JWT由三部分构成，分别是：`header`、`payload`、`signature`，它们之前通过半角圆点【.】连接。

一个典型的JWT看起来应该是：

```json
token = encodeBase64(header)+"."+encodeBase64(payload)+"."+encodeBase64(signature)
```

- `header`:头部分通常由两部分组成：令牌的的类型(`"JWT"`)和使用的签名算法(`"HS256、RSA等"`)

  ```json
  {"alg":"HS256","typ":"JWT"}
  ```

  然后用`Base64`进行编码就得到了JWT的第一部分

- `payload`:包含声明(要求)。声明是关于实体(通常是用户)和其他数据的声明。声明有三种类型: `registered`、`public` 和 `private`

  - `registered claims`：这里有一组预定义的声明，它们不是强制的，但是推荐。比如：iss (issuer), exp (expiration time), sub (subject), aud (audience)等
  - `public`：可以随意定义
  - `private`：用于在同意使用它们的各方之间共享信息，并且不是注册的或公开的声明

  ```json
  {
      "sub":"1234567890",
      "name":"hanson.q",
      "admin":true
  }
  ```

  然后用`Base64`进行编码就得到了JWT的第二部分

  注：不要在JWT的payload或header中放置敏感信息，除非它们是加密的。

- `signature`:用来判断消息在传递的过程中是否被篡改，从而保证数据的安全性，对于使用私钥签名的token，它还可以验证JWT的发送方是否为它所称的发送方

  - 为了得到签名部分，你必须有编码过的header、编码过的payload、一个秘钥，签名算法是header中指定的那个，然对它们签名即可

  ```
  HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)
  ```









参考：<https://www.cnblogs.com/cjsblog/p/9277677.html>

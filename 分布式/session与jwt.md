## 分布式下session

```
问题描述:
	用户输入用户名密码，在认证服务器登录了，然后将session存放在认证服务器
	用户跳转首页，想要购买订单，请求指向订单服务器，订单服务器里面没有用户session, 这时判定用户未登录
```

### 基于session的用户认证

```
1. 用户输入其登录信息

2. 服务器验证信息是否正确，并创建一个session，然后将其存储在数据库中

3. 服务器为用户生成一个sessionId，将具有sesssionId的Cookie将放置在用户浏览器中

4. 在后续请求中，会根据数据库验证sessionID，如果有效，则接受请求

5. 一旦用户logout，会话将在服务器端销毁, session.remove(sessionId)即可
```

### session原理

![image-20210915122810394](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20210915122810394.png)

### session复制

![image-20210915123847884](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20210915123847884.png)





### session统一存储

![image-20210915125353023](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20210915125353023.png)



### session共享解决

```tex
子域系统的jsessionid这个cookie,发送给父域,然后父域访问子域就能成功
然后不同微服务之间都去redis,根据这个jsessionid拿到session
即order:100 给浏览器发了一个jsessionid,并且将这个session存入redis
  order:101 可以根据jsessionid从redis拿到order:100的session,实现了session共享
```



![image-20210915130515290](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20210915130515290.png)



### SpringSession解决方案

- maxInactiveInterval为session过期时间，默认30分钟
  - 默认情况下，每次用户请求时都会自动续期（也称为延长过期时间）

```sql
spring session中的key大概为 spring:session:key:sessionId
value为hash结构
Value (Hash):
- creationTime -> 1696859140000
- lastAccessedTime -> 1696859245000
- maxInactiveInterval -> 1800
- sessionAttr:userId -> 23123
- sessionAttr:nickname -> varerleet
- expired -> 0
```



**1.分布式下需要用到session的模块,引入依赖**

```xml
        <dependency>
            <groupId>org.springframework.session</groupId>
            <artifactId>spring-session-data-redis</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
```

**2.简单配置**

```yaml
spring:
  session:
    store-type: redis  #SpringSession配合redis解决分布式下session问题
```

**3.注解配置**

```java
@EnableRedisHttpSession
@Configuration
public class SessionConfig {
    /**
     * 存入redis以json方式存入,否则每个存入session的类都需要序列化implements Serializable,否则报错
     * @return
     */
    @Bean
    public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
        return new GenericFastJsonRedisSerializer();
    }

    /**
     * cookieName默认是SESSION
     * 域名一定要写父域名
     * cookie的作用域是domain本身以及domain下的所有子域名。
     * 给用户发这个cookie, value就是sessionId
     * @return
     */
    @Bean
    public CookieSerializer cookieSerializer(){
        DefaultCookieSerializer cookieSerializer = new DefaultCookieSerializer();
        cookieSerializer.setCookieName("GULIMALLCOOKIE");
        cookieSerializer.setDomainName("gulimall.com");
        return cookieSerializer;
    }

}
```

**4.原理分析**

**核心类:**

- RedisOperationsSessionRepository
- SessionRepositoryFilter
- DefaultCookieSerializer

```java
/**
 * 核心原理[装饰者模式]
 * 1) @EnableRedisHttpSession
 *      @Import({RedisHttpSessionConfiguration.class})
 *      1.给容器中添加了 SessionRepository 【RedisOperationsSessionRepository】 :redis操作session,session的增删改查封装类
 *      2. SessionRepositoryFilter ->session存储过滤器,每个请求过来都必须经过filter
 *          SessionRepositoryFilter构造函数传入一个SessionRepository,即RedisOperationsSessionRepository
 *          原生的request,response被包装SessionRepositoryRequestWrapper
 *              SessionRepositoryRequestWrapper通过SessionRepository取session
 *
 *          通过如下方式取session
 *          private S getRequestedSession() {
 *             if (!this.requestedSessionCached) {
 *
 *                 List<String> sessionIds =
 *                      SessionRepositoryFilter.this.httpSessionIdResolver.resolveSessionIds(this){
 *                            //此方法找sessionId是通过cookie的名字找的this.cookieName.equals(cookie.getName())
 *                            //cookieName默认是SESSION,当可以自己指定cookieSerializer.setCookieName
 *                            //而cookieSerializer就是配置好了的DefaultCookieSerializer
 *                            //即前端只要传来的cookieName=GULIMALLCOOKIE,那么就可以拿到这个cookie携带的sessionId了
 *                            //1.拿到了sessionId,就可以去redis查到session
 *                            //2.再设置好cookie的域名是父域
 *                            //由1和2就实现了分布式下session共享
 *                            return this.cookieSerializer.readCookieValues(request);
 *                      }
 *                 Iterator var2 = sessionIds.iterator();
 *
 *                 while(var2.hasNext()) {
 *                     String sessionId = (String)var2.next();
 *                     if (this.requestedSessionId == null) {
 *                         this.requestedSessionId = sessionId;
 *                     }
 *                      //从redis找到这个session
 *                     S session = SessionRepositoryFilter.this.sessionRepository.findById(sessionId);
 *                     if (session != null) {
 *                         this.requestedSession = session;
 *                         this.requestedSessionId = sessionId;
 *                         break;
 *                     }
 *                 }
 *
 *                 this.requestedSessionCached = true;
 *             }
 *
 *             return this.requestedSession;
 *         }
 */
```



## JWT

Json web token (JWT),该token被设计为紧凑且安全的，适用于分布式站点的单点登录（SSO）场景。JWT的声明一般被用来在身份提供者和服务提供者间传递被认证的用户身份信息，以便于从资源服务器获取资源，也可以增加一些额外的其它业务逻辑所必须的声明信息，该token也可直接被用于认证，也可被加密。

### 基于token的鉴权机制

```
基于token的鉴权机制类似于http协议也是无状态的，它不需要在服务端去保留用户的认证信息或者会话信息。这就意味着基于token认证机制的应用不需要去考虑用户在哪一台服务器登录了，这就为应用的扩展提供了便利。

流程上是这样的：
- 1、用户登录,根据账号密码查数据库
- 2、服务的认证，通过后根据 密钥 生成 token
- 3、将生成的 token 返回给浏览器
- 4、用户每次请求携带 token
- 5、服务端利用秘钥解读 jwt 签名，判断签名有效后，从 Payload 中获取用户信息
- 6、处理请求，返回响应结果
这个token必须要在每次请求时传递给服务端，它应该保存在请求头里
```

### JWT的组成

![image-20241109211749147](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20241109211749147.png)

#### 头部（Header）

头部用于描述关于该JWT的最基本的信息，例如其类型以及签名所用的算法等。

```json
{
  "typ": "JWT",
  "alg": "HS256"
}
```

在这里，我们说明了这是一个JWT，并且我们所用的签名算法是HS256算法。

#### 载荷（Payload）

载荷可以用来放一些不敏感的信息。

```json
{
    "iss": "John Wu JWT",
    "iat": 1441593502,
    "exp": 1441594722,
    "aud": "www.example.com",
    "sub": "jrocket@example.com",
    "from_user": "B",
    "target_user": "A"
}
```

这里面的前五个字段都是由JWT的标准所定义的。

- `iss`: 该JWT的签发者
- `sub`: 该JWT所面向的用户
- `aud`: 接收该JWT的一方
- `exp`(expires): 什么时候过期，这里是一个Unix时间戳
- `iat`(issued at): 在什么时候签发的

把头部和载荷分别进行Base64编码之后得到两个字符串，然后再将这两个编码后的字符串用英文句号`.`连接在一起（头部在前），形成新的字符串：

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJmcm9tX3VzZXIiOiJCIiwidGFyZ2V0X3VzZXIiOiJBIn0
```

#### 签名（signature）

最后，我们将上面拼接完的字符串用HS256算法进行加密。在加密的时候，我们还需要提供一个密钥（secret）。加密后的内容也是一个字符串，最后这个字符串就是签名，把这个签名拼接在刚才的字符串后面就能得到完整的jwt。header部分和payload部分如果被篡改，由于篡改者不知道密钥是什么，也无法生成新的signature部分，服务端也就无法通过，在jwt中，消息体是透明的，使用签名可以保证消息不被篡改。

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJmcm9tX3VzZXIiOiJCIiwidGFyZ2V0X3VzZXIiOiJBIn0.rSWamyAYwuHCo7IFAgd1oRpSP7nzL7BF5t7ItqpKViM
```

 

### JWT工具类

```xml
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt</artifactId>
            <version>0.7.0</version>
        </dependency>
```

```java
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jws;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.util.StringUtils;

import javax.servlet.http.HttpServletRequest;
import java.util.Date;

/**
 * @author helen
 * @since 2019/10/16
 */
public class JwtUtils {

    public static final long EXPIRE = 1000 * 60 * 60 * 24;  //token过期时间
    public static final String APP_SECRET = "ukc8BDbRigUDaY6pZFfWus2jZWLPHO";  //密钥,由自己生成

    //生成token字符串
    public static String getJwtToken(String id, String nickname){

        String JwtToken = Jwts.builder()
                //头信息
                .setHeaderParam("typ", "JWT")
                .setHeaderParam("alg", "HS256")
                //过期时间
                .setSubject("guli-user")
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + EXPIRE))
                //设置token主体部分
                .claim("id", id)
                .claim("nickname", nickname)
                /*
                签名哈希部分是对上面两部分数据签名，通过指定的算法生成哈希，以确保数据不会被篡改。
                首先，需要指定一个密码（secret）。该密码仅仅为保存在服务器中，并且不能向用户公开。
                然后，使用标头中指定的签名算法（默认情况下为HMAC SHA256）根据以下公式生成签名。
                HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(claims), secret)
                 */
                .signWith(SignatureAlgorithm.HS256, APP_SECRET)
                .compact();

        return JwtToken;
    }

    /**
     * 判断token是否存在与有效
     * @param jwtToken
     * @return
     */
    public static boolean checkToken(String jwtToken) {
        if(StringUtils.isEmpty(jwtToken)) return false;
        try {
            Jwts.parser().setSigningKey(APP_SECRET).parseClaimsJws(jwtToken);
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
        return true;
    }

    /**
     * 判断token是否存在与有效
     * @param request
     * @return
     */
    public static boolean checkToken(HttpServletRequest request) {
        try {
            String jwtToken = request.getHeader("token");
            if(StringUtils.isEmpty(jwtToken)) return false;
            Jwts.parser().setSigningKey(APP_SECRET).parseClaimsJws(jwtToken);
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
        return true;
    }

    /**
     * 根据token获取会员id
     * @param request
     * @return
     */
    public static String getMemberIdByJwtToken(HttpServletRequest request) {
        String jwtToken = request.getHeader("token");
        if(StringUtils.isEmpty(jwtToken)) return "";
        Jws<Claims> claimsJws = Jwts.parser().setSigningKey(APP_SECRET).parseClaimsJws(jwtToken);
        Claims claims = claimsJws.getBody();
        return (String)claims.get("id");
    }
}

```



## 区别与优缺点

基于session和基于jwt的方式的主要区别就是用户的状态保存的位置，session是保存在服务端的，而jwt是保存在客户端的。

jwt经过base64编码后的字符串是比较长的，而cookie-session中，sessionId是比较短的



**jwt优点:**

- 可扩展性好, 应用程序分布式部署的情况下，session需要做多机数据共享，通常可以存在数据库或者redis里面。而jwt不需要。
- 无状态, jwt不在服务端存储任何状态。



**jwt缺点：**

- 安全性， 由于jwt的payload是使用base64编码的，并没有加密，因此jwt中不能存储敏感数据。而session的信息是存在服务端的，相对来说更安全。
- 性能， jwt太长。由于是无状态使用JWT，所有的数据都被放到JWT里，如果还要进行一些数据交换，那载荷会更大，经过编码之后导致jwt非常长，cookie的限制大小一般是4k，cookie很可能放不下，所以jwt一般放在local storage里面。并且用户在系统中的每一次http请求都会把jwt携带在Header里面，http请求的Header可能比Body还要大。而sessionId只是很短的一个字符串，因此使用jwt的http请求比使用session的开销大得多。
- 一次性， 无状态是jwt的特点，但也导致了这个问题，jwt是一次性的。想修改里面的内容，就必须签发一个新的jwt。
  - 无法废弃， 通过上面jwt的验证机制可以看出来，一旦签发一个jwt，在到期之前就会始终有效，无法中途废弃。例如你在payload中存储了一些信息，当信息需要更新时，则重新签发一个jwt，但是由于旧的jwt还没过期，拿着这个旧的jwt依旧可以登录，那登录后服务端从jwt中拿到的信息就是过时的。为了解决这个问题，我们就需要在服务端部署额外的逻辑，例如设置一个黑名单，一旦签发了新的jwt，那么旧的就加入黑名单（比如存到redis里面），避免被再次使用。
  - 如果你使用jwt做会话管理，传统的cookie续签方案一般都是框架自带的，session有效期30分钟，30分钟内如果有访问，有效期被刷新至30分钟。一样的道理，要改变jwt的有效时间，就要签发新的jwt。最简单的一种方式是每次请求刷新jwt，即每个http请求都返回一个新的jwt。这个方法不仅暴力不优雅，而且每次请求都要做jwt的加密解密，会带来性能问题。另一种方法是在redis中单独为每个jwt设置过期时间，每次访问时刷新jwt的过期时间。
  - 可以看出想要破解jwt一次性的特性，就需要在服务端存储jwt的状态。但是引入 redis 之后，就把无状态的jwt硬生生变成了有状态了，违背了jwt的初衷。而且这个方案和session都差不多了。
  - 因此由于web网站需要管理用户登录，登出，以及信息修改后重新登录，所以采用session比较合适。



适合使用jwt的场景：

- 有效期短
- 只希望被使用一次

比如，用户注册后发一封邮件让其激活账户，通常邮件中需要有一个链接，这个链接需要具备以下的特性：能够标识用户，该链接具有时效性（通常只允许几小时之内激活），不能被篡改以激活其他可能的账户，一次性的。这种场景就适合使用jwt。

而由于jwt具有一次性的特性。单点登录和会话管理非常不适合用jwt，如果在服务端部署额外的逻辑存储jwt的状态，那还不如使用session。基于session有很多成熟的框架可以开箱即用，但是用jwt还要自己实现逻辑。

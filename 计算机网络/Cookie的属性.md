# Cookie的属性

Cookie截图，属性分别有**Name**、**Value**、**Domain**、**Path**、**Expires/Max-age**、**Size**、**HttpOnly**、**Secure**、**SameSite**和**Priority**。

![image-20220814111317727](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220814111317727.png)

### Name和Value

Name和Value是一个键值对。Name是Cookie的名称，Cookie一旦创建，名称便不可更改，一般名称不区分大小写；Value是该名称对应的Cookie的值，如果值为Unicode字符，需要为字符编码。如果值为二进制数据，则需要使用BASE64编码。


### Domain

Domain决定Cookie在哪个域是有效的，也就是决定在向该域发送请求时是否携带此Cookie，Domain的设置是对子域生效的，如Doamin设置为 .a.com,则b.a.com和c.a.com均可使用该Cookie，但如果设置为b.a.com,则c.a.com不可使用该Cookie。Domain参数必须以点(".")开始。
 

### Path

Path是Cookie的有效路径，和Domain类似，也对子路径生效，如Cookie1和Cookie2的Domain均为a.com，但Path不同，Cookie1的Path为 /b/,而Cookie的Path为 /b/c/,则在a.com/b页面时只可以访问Cookie1，在a.com/b/c页面时，可访问Cookie1和Cookie2。Path属性需要使用符号“/”结尾。
 

### Expires/Max-age

Expires和Max-age均为Cookie的有效期，Expires是该Cookie被删除时的时间戳，格式为GMT,若设置为以前的时间，则该Cookie立刻被删除，并且该时间戳是服务器时间，不是本地时间！若不设置则默认页面关闭时删除该Cookie。
Max-age也是Cookie的有效期，但它的单位为秒，即多少秒之后失效，若Max-age设置为0，则立刻失效，设置为负数，则在页面关闭时失效。Max-age默认为 -1。
 

### Size

Szie是此Cookie的大小。在所有浏览器中，任何cookie大小超过限制都被忽略，且永远不会被设置。各个浏览器对Cookie的最大值和最大数目有不同的限制

![image-20220814111621110](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220814111621110.png) 

### HttpOnly

HttpOnly值为 true 或 false,若设置为true，则不允许通过脚本document.cookie去更改这个值，同样这个值在document.cookie中也不可见，但在发送请求时依旧会携带此Cookie。
 

### Secure

Secure为Cookie的安全属性，若设置为true，则浏览器只会在HTTPS和SSL等安全协议中传输此Cookie，不会在不安全的HTTP协议中传输此Cookie。
 

### Priority

优先级，chrome的提案，定义了三种优先级，Low/Medium/High，当cookie数量超出时，低优先级的cookie会被优先清除。
在360极速浏览器和FireFox中，不存在Priority属性，不清楚在此类浏览器中设置该属性后是否生效。
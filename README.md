一、获取token过程
1、zuul，转发请求到认证服务器，获取token


![Image text]( https://img-blog.csdnimg.cn/20200403172144676.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xoODcyNzAyMDI=,size_16,color_FFFFFF,t_70)

2、

最后调到定义的redisTokenStore.getAccessToken返回token
Authenticaton对象

用前缀+name,clientid,scope MD5后组成key
10f0ad1bd4f6c7cb5744016a8a125f04
auth_to_access:10f0ad1bd4f6c7cb5744016a8a125f04

生成token:82c30174-b024-498d-b19d-053e4dd05e92
存储到redis,  key: access:82c30174-b024-498d-b19d-053e4dd05e92

获取token的接口，不用走过滤器，是因为：开始会匹配Url,

如果在以上的url中，那么就会直接请求到对应的方法

二、用token认证过程

1、通过zull转发，请求认证服务器actuator 端点，由于认证服务器对actuator端点有拦截配置，所以会走一个 fileter

这个fiter会去从requet中获取到Authorization 请求头，拿出里面的token，然后再调用
Authentication authResult = authenticationManager.authenticate(authentication);
调这个方法校验，
OAuth2Authentication auth = tokenServices.loadAuthentication(token);
OAuth2AccessToken accessToken = tokenStore.readAccessToken(accessTokenValue);
再调到我们重写的tokenStore的方法，获取accessToken

从redis中获取token ，key为：  access:82c30174-b024-498d-b19d-053e4dd05e92

请求头中需要带上从第一步获取的认证码，token前面要带上token的类型

三、微服务认证过程

1、请求zuul地址，会转发到micro-web-security
2、mirco微服务会对user 开头的请求路径要求 auth认证，由于这它不是认证服务器，所以没有配置 authManager 和 storeService
配置文件中配置了：security.oauth2.resource.user-info-uri=http://127.0.0.1:7070/auth/security/check
，当转到micro-web-sercutiry微服务时， 会被outh2的过滤器拦截，然后找到 这个uer-info-uri 去调用这个接口 认证，这个是认证服务器的地址，会返回 Principl对象，如果获取成功，那么就会再执行自己的方法，如果失败 则报错。

再细想想，只配置这个地是否还不够， 认证服务器 肯定会走过滤器，校验客户的合法性，于是还需要下面配置：
security.oauth2.resource.prefer-token-info=false
#security.oauth2.client.id=micro-web
security.oauth2.client.clientId=micro-web
security.oauth2.client.client-secret=123456
security.oauth2.client.access-token-uri=http://api-gateway/auth/oauth/token
security.oauth2.client.grant-type=client_credentials
security.oauth2.client.scope=all
请求的客户端是谁，获取token的地址， 
认证服务器获取到这些信息后，会拿这个客户端生成的token 和 转过来的token对比，如果相同证明是合法请求，说明 这个客户端之前从认证服务器获取过token

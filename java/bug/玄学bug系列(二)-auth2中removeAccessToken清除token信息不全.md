《玄学bug系列(二)-auth2中**removeAccessToken**清除token信息不全》首发[牧马人博客](http://www.luckyhe.com/post/88.html)转发请加此提示

欢迎大家观看玄学bug系列第二篇，好久没遇到值得写篇文章描述的bug了，话不多说吗，直接开干。

玄学程度5颗星。难度4颗星，爆肝程度一天。



#### 读前需知

本文所说**oauth2**版本为`2.3.6`

正式所用环境为**Redis**集群环境

本文所说的**tokenStore**是**oauth2**中的**RedisTokenStore**这个类，所以我的**token**信息是存在于**redis**服务器中的。

#### 前因

简单的介绍了一下我的项目结构，我们是一个微服务项目，所以我们有自己的认证中心，也就是`oauth2`认证中心。这次出现的问题就在正式环境下，权限信息，清除不掉。但是开发环境，以及测试环境是正常的。我的系统只有在退出登录会调用**removeAccessToken**方法的情况下会清除**token**信息。清除完token后正常情况是会掉线的，也就是需要重新获取**token**。正常情况下是会重新查表，然后缓存到**redis**。这次正式环境出的问题就是清除**token**信息不全。导致用户重新登录后，获取的还是老**token**信息。也就是**RedisTokenStore.removeAccessToken**在正式环境出了bug，清除**token**信息不全。



这里先给大家普及下，在我的**oauth2**认证服务中，如果登录成功，会往**redis**插入9条信息。分别是：

![authtoken](G:\Learnning\pic\java\bug\authtoken.png)

这里我主要讲两个:`access`   `auth_to_access`
    这两个存的都是用户的`token信息`(一模一样)。但是生成的`redis`key的规则不一样。``access`   `是跟据 access` +  token序列化后的值作为key。把**token**存进去。`auth_to_access`是以auth_to_access为前缀+username+client_id+scope的值md5加密后作为key存在redis中。其实别看它生成那么多，其实最终存入**redis**中的东西都是一样的。其中我这边爆出生产问题的就是因为`auth_to_access`为前缀的**token**值没删除掉。

#### 排查思路

发现这个问题后，我第一时间就反应过来了，查看redis集群看看是否**`token`**没有清除掉。果不其然。有一个**`token`**信息没有清理掉。前缀为**auth_to_access**的token信息没清除掉。



知道这个问题后，第一时间我就去看了退出登录这个方法

```java
	@DeleteMapping("/{token}")
	public R<Boolean> delToken(@PathVariable("token") String token) {
		OAuth2AccessToken accessToken = tokenStore.readAccessToken(token);
		if (accessToken == null || StrUtil.isBlank(accessToken.getValue())) {
			return R.ok(Boolean.TRUE, "退出失败，token 无效");
		}

		OAuth2Authentication auth2Authentication = tokenStore.readAuthentication(accessToken);
		// 清空用户信息
		cacheManager.getCache(CacheConstants.USER_DETAILS)
				.evict(auth2Authentication.getName());

        // 清空access token
		tokenStore.removeAccessToken(accessToken);

        // 清空 refresh token
		OAuth2RefreshToken refreshToken = accessToken.getRefreshToken();
		tokenStore.removeRefreshToken(refreshToken);
		return R.ok();
	}

```

从上诉方法中可以看出我是**tokenStore.removeAccessToken(accessToken);**出问题了。然后我进去源码看了下

```java
public void removeAccessToken(String tokenValue) {
		byte[] accessKey = serializeKey(ACCESS + tokenValue);
		byte[] authKey = serializeKey(AUTH + tokenValue);
		byte[] accessToRefreshKey = serializeKey(ACCESS_TO_REFRESH + tokenValue);
		RedisConnection conn = getConnection();
		try {
			conn.openPipeline();
			conn.get(accessKey);
			conn.get(authKey);
			conn.del(accessKey);
			conn.del(accessToRefreshKey);
			// Don't remove the refresh token - it's up to the caller to do that
			conn.del(authKey);
			List<Object> results = conn.closePipeline();
			byte[] access = (byte[]) results.get(0);
			byte[] auth = (byte[]) results.get(1);

			OAuth2Authentication authentication = deserializeAuthentication(auth);
			if (authentication != null) {
               
				String key = authenticationKeyGenerator.extractKey(authentication);
				byte[] authToAccessKey = serializeKey(AUTH_TO_ACCESS + key);
				byte[] unameKey = serializeKey(UNAME_TO_ACCESS + getApprovalKey(authentication));
				byte[] clientId = serializeKey(CLIENT_ID_TO_ACCESS + authentication.getOAuth2Request().getClientId());
				conn.openPipeline();
                //我是这一步没删除掉，所以可能有两原因这个key生成跟插入的时候不一致，第二点就是单纯的删除不掉。
				conn.del(authToAccessKey);
				conn.sRem(unameKey, access);
				conn.sRem(clientId, access);
				conn.del(serialize(ACCESS + key));
				conn.closePipeline();
			}
		} finally {
			conn.close();
		}
	}
```

这一步一共删除了9个，为什么单单其中一个删除不掉。这时候我们有理由怀疑`authenticationKeyGenerator.extractKey(authentication)`这个方法出现了问题

**AUTH_TO_ACCESS**这个前缀的key是这样生成的

```java
String key = authenticationKeyGenerator.extractKey(auth2Authentication);
String redisKey =  SecurityConstants.AUTH_TO_ACCESS + key;
```

而**extractKey**这个方法是这样的

```java
public String extractKey(OAuth2Authentication authentication) {
		Map<String, String> values = new LinkedHashMap<String, String>();
		OAuth2Request authorizationRequest = authentication.getOAuth2Request();
		if (!authentication.isClientOnly()) {
			values.put(USERNAME, authentication.getName());
		}
		values.put(CLIENT_ID, authorizationRequest.getClientId());
		if (authorizationRequest.getScope() != null) {
			values.put(SCOPE, OAuth2Utils.formatParameterList(new TreeSet<String>(authorizationRequest.getScope())));
		}
		return generateKey(values);
	}

```

**generateKey**这个方法仅仅只是做了一个md5加密。这一步可以确定没问题了。

```java
protected String generateKey(Map<String, String> values) {
		MessageDigest digest;
		try {
			digest = MessageDigest.getInstance("MD5");
			byte[] bytes = digest.digest(values.toString().getBytes("UTF-8"));
			return String.format("%032x", new BigInteger(1, bytes));
		} catch (NoSuchAlgorithmException nsae) {
			throw new IllegalStateException("MD5 algorithm not available.  Fatal (should be in the JDK).", nsae);
		} catch (UnsupportedEncodingException uee) {
			throw new IllegalStateException("UTF-8 encoding not available.  Fatal (should be in the JDK).", uee);
		}
	}
```

由上面源码可得唯一可能的错就是**authentication**认证信息跟一开始认证的时候不对。然后我在退出登录那里把信息打印出来了

```java
       String key = authenticationKeyGenerator.extractKey(auth2Authentication);
       String name = auth2Authentication.getName();
       String clientId = auth2Authentication.getOAuth2Request().getClientId();
       String scope = OAuth2Utils.formatParameterList(new TreeSet<String>(auth2Authentication.getOAuth2Request().getScope()));
       log.error("NAME:--"+name+"-CLIENTID:"+clientId+"-SCOPE---"+scope+"-KEY:"+key);
        System.out.println(key);
```

离奇的事发生了，信息都是对应得上的。也就是说`authenticationKeyGenerator.extractKey(auth2Authentication);`key值一样。但是正式环境，由于网络权限问题，我没办法远程debug。所以我进入不到**removeAccessToken**这个方法调试看那个**key**是不是一致的。我只能在自己的外围方法里打debug调用。模拟**oauth2**自带的**key**生成方式生成一个**key**来进行对比。但是对比结果却是一样的。最终我经过很多尝试，还是没能找到问题出在哪。所以我就想了个笨方法来解决这个bug。

#### 解决方案（并非最佳）

这个方法比较笨，其实就是自动手动去删除**token**信息，因为你知道了**key**的生成规则，所以完全是可行的。代码如下。事后紧急打了个修复包。亲测可行。

```java
  /**
     * 手动清理AuthToAccessTOken
     * 自带方法在正式环境有个未知bug处理不了
     *
     * @param auth2Authentication
     */
    private void delAuthToAccess(OAuth2Authentication auth2Authentication) {

        if (null == auth2Authentication) return;

        AuthenticationKeyGenerator authenticationKeyGenerator
                = new DefaultAuthenticationKeyGenerator();
        String key = authenticationKeyGenerator.extractKey(auth2Authentication);
//        String name = auth2Authentication.getName();
//        String clientId = auth2Authentication.getOAuth2Request().getClientId();
//        String scope = OAuth2Utils.formatParameterList(new TreeSet<String>(auth2Authentication.getOAuth2Request().getScope()));
//        log.error("NAME:--"+name+"-CLIENTID:"+clientId+"-SCOPE---"+scope+"-KEY:"+key);
        System.out.println(key);
        String redisKey = 
                SecurityConstants
                        .AUTH_TO_ACCESS + key ;
        // 清空AUTH_TO_ACCESS access token
        redisTemplate.delete(redisKey);

    }
```



#### 后话

这个问题，至今为止，我还没发现这个问题是为什么在正式环境才出现。为什么**oauth2**自带的删除方法其余8个都能正常清理，但是唯独非常关键的一个却清理不了。事后我还对**oauth2**登录代码研究下，解决我的另一个疑问。**<u>就是我都重新登录了，为什么不能重新生成token信息</u>**。我发现了一个疑是bug的设计。就是登录那里，他在你认证，授权都成功后。**token**都重新生成了。他不是把**token**信息插入**redis**。而是去找**redis**有没有**token**信息，如果有就直接返回老的token信息，而不是把新生成的**token**信息存进redis。然后把新的返回给调用者。在**DefaultTokenServices.createAccessToken**这个类里面

```java
@Transactional
	public OAuth2AccessToken createAccessToken(OAuth2Authentication authentication) throws AuthenticationException {

        //这一步就是我的疑问，竟然把新的认证信息都拿过来了，重新生成个新的，把新的返回去不就好了，为啥还去找有没有老的。
        //这一步我只站在业务的角度上去考虑这个事情，没有在整个架构去考虑，如果这不是bug希望有人解决我这个疑问
		OAuth2AccessToken existingAccessToken = tokenStore.getAccessToken(authentication);
		OAuth2RefreshToken refreshToken = null;
		if (existingAccessToken != null) {
			if (existingAccessToken.isExpired()) {
				if (existingAccessToken.getRefreshToken() != null) {
					refreshToken = existingAccessToken.getRefreshToken();
					// The token store could remove the refresh token when the
					// access token is removed, but we want to
					// be sure...
					tokenStore.removeRefreshToken(refreshToken);
				}
				tokenStore.removeAccessToken(existingAccessToken);
			}
			else {
				// Re-store the access token in case the authentication has changed
				tokenStore.storeAccessToken(existingAccessToken, authentication);
				return existingAccessToken;
			}
		}

		// Only create a new refresh token if there wasn't an existing one
		// associated with an expired access token.
		// Clients might be holding existing refresh tokens, so we re-use it in
		// the case that the old access token
		// expired.
		if (refreshToken == null) {
			refreshToken = createRefreshToken(authentication);
		}
		// But the refresh token itself might need to be re-issued if it has
		// expired.
		else if (refreshToken instanceof ExpiringOAuth2RefreshToken) {
			ExpiringOAuth2RefreshToken expiring = (ExpiringOAuth2RefreshToken) refreshToken;
			if (System.currentTimeMillis() > expiring.getExpiration().getTime()) {
				refreshToken = createRefreshToken(authentication);
			}
		}

		OAuth2AccessToken accessToken = createAccessToken(authentication, refreshToken);
		tokenStore.storeAccessToken(accessToken, authentication);
		// In case it was modified
		refreshToken = accessToken.getRefreshToken();
		if (refreshToken != null) {
			tokenStore.storeRefreshToken(refreshToken, authentication);
		}
		return accessToken;

	}
```

后续我会把**/oauth/token**这个方法的源码解析讲一下，解决一下大家对于**/oaurh/token**这个链接请求的一些疑问。有啥问题欢迎大家和我一起探讨。


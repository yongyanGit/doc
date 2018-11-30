### Spring Security 核心组件概述

> 本系列参考自https://www.cnkirito.moe/categories/Spring-Security/

#### 1. SecurityContextHolder

SecurityContextHolder用于存储安全上下文（security context）的信息，如：当前操作的用户是谁，该用户是否已经被认证，它拥有的角色权限等，这些都被存放在SecurityContextHolder中。

如获取当前登陆用户名称：

```java
Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();
		
if (principal instanceof UserDetails){
	String userName = ((UserDetails) principal).getUsername();
}
```

getAuthentication返回了认证信息，getPrincipal()返回了登陆用户的身份信息。UserDetails是Spring对身份信息封装的一个接口。

#### 2. Authentication

通过AuthenTication接口可以获取用户的权限信息列表、密码、用户细节信息、用户身份信息、认证信息。

```java
Collection<? extends GrantedAuthority> getAuthorities();
Object getCredentials();	
Object getDetails();
Object getPrincipal();
```

* getAuthorities()：权限信息列表，默认是GrantedAuthority接口的一些实现类，通常是代表权限信息的一系列字符串。
* getCredentials()：密码信息，用户输入的密码字符串，在认证过后通常会被移除，用于保障安全。
* getDetails()：细节信息，web应用中的实现通常为WebAuthenticationDetails，它记录了访问者的ip地址和sessionId的值。
* getPrincipal()：大部分下返回的是UserDetails接口的实现类。

#### 3. UserDetails接口

UserDetails接口代表了用户最详细的信息，这个接口涵盖了一些必要的用户信息字段，具体的实现类可以对它进行拓展。

```java
public interface UserDetails extends Serializable {
	
	Collection<? extends GrantedAuthority> getAuthorities();

	String getPassword();

	String getUsername();

	boolean isAccountNonExpired();

	boolean isAccountNonLocked();

	boolean isCredentialsNonExpired();
	boolean isEnabled();
}
```

UserDetails接口和Authentication接口很相似，但是它们也有一些区别，如：Authentication中的getCredentials()与UserDetails中的getPassword()需要区别对待，前者是用户提交的密码凭证，后者是用户正确的密码，认证器其实就是对这两者的比对。Authentication中的getAuthorities()实际是由UserDetails的getAuthorities()传递而形成的。

#### 4.  AuthenticationManager

AuthenticationManager是认证相关的核心接口，也是发起认证的出发点。但是AuthenticatonManager 一般都不会直接认证，它有一个子类ProviderManager，内部会维护一个List<AuthenticationProvider>列表，用来存放多种认证方式如：账号、密码登陆，自定义登陆认证(短信登陆)。

```java
//ProviderManager
public Authentication authenticate(Authentication authentication)
throws AuthenticationException {
	Class<? extends Authentication> toTest = authentication.getClass();
	AuthenticationException lastException = null;
	Authentication result = null;
    //遍历循环list集合
	for (AuthenticationProvider provider : getProviders()) {
		if (!provider.supports(toTest)) {
			continue;
		}

		try {
   			//进行认证   
			result = provider.authenticate(authentication);
				if (result != null) {
					copyDetails(authentication, result);
					break;
				}
			}
			...
			catch (AuthenticationException e) {
				lastException = e;
			}
		}
		...
        //如果Authentication 不为空，则返回
		if (result != null) {
			if (eraseCredentialsAfterAuthentication
					&& (result instanceof CredentialsContainer)) {
                //移除密码
				((CredentialsContainer) result).eraseCredentials();
			}
			//发布登陆成功消息
			eventPublisher.publishAuthenticationSuccess(result);
			return result;
		}
		//登陆失败
		if (lastException == null) {
			lastException = new ProviderNotFoundException(messages.getMessage(
					"ProviderManager.providerNotFound",
					new Object[] { toTest.getName() },
					"No AuthenticationProvider found for {0}"));
		}

		prepareException(lastException, authentication);

		throw lastException;
	}
```

ProviderManager中的list 会依次循环去认证，如果有认证结果返回则会停止循环，将认证成功的信息返回，如果认证结果为空，则会继续执行下一个AuthenticationProvider认证方法，如果所有的认证器都无法认证成功，则ProviderManager会抛出一个providerNotFoundException异常。

#### 5. UserDetailsService

UserdetailsService 负责从特定的地方加载用户信息（如从数据库中查询用户），UserDetailsService常见的实现有JdbcDaoImpl，InMemoryUserDetailsManager，前者从数据库中加载用户，后者从内存中加载用户，当然也可以自己实现UserDetailsService。






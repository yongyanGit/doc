### Spring Security核心配置

#### 1. @EnableWebSecurity

我们在定义配置类WebSecurityConfig加上@EnableWebSecurity注解，同时继承了WebSecurityConfigurerAdapter。如下，EnableWebSecurity注解的内容：

```java
@Retention(value = java.lang.annotation.RetentionPolicy.RUNTIME)
@Target(value = { java.lang.annotation.ElementType.TYPE })
@Documented
@Import({ WebSecurityConfiguration.class,
		SpringWebMvcImportSelector.class })
@EnableGlobalAuthentication
@Configuration
public @interface EnableWebSecurity {

	/**
	 * Controls debugging support for Spring Security. Default is false.
	 * @return if true, enables debug support with Spring Security
	 */
	boolean debug() default false;
}
```

@Import是springboot提供的用于引入外部的配置的注解：

* WebSecurityConfiguration：用来配置web安全。
* SpringWebMvcImportSelector：判断当前环境释放包含springmvc,因为spring security可以在非spring环境中使用，为了避免DispathcherServlet的重复配置。
* EnableGlobalAuthentication注解圆满如下：

```java
@Retention(value = java.lang.annotation.RetentionPolicy.RUNTIME)
@Target(value = { java.lang.annotation.ElementType.TYPE })
@Documented
@Import(AuthenticationConfiguration.class)
@Configuration
public @interface EnableGlobalAuthentication {
}
```

同样在@Import中导入了AuthenticationConfiguration用来配置认证相关的核心类。

也就是说，@EnableWebSecurity完成的工作便是加载WebSecurityConfiguration、AuthenticationConfiguration这两个核心配置类，即将spring security的职责划分配置安全信息和配置认证信息两部分。

#### WebSecurityConfiguration

在这个配置类中有一个非常重要的Bean(springSecurityFilterChain)被注册了,它是spring security的核心过滤器，是整个认证的入口。它在WebsecurityConfiguration声明，并且负责拦截请求的任务交给了DelegatingFilterProxy这个代理类。

```java
@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
public Filter springSecurityFilterChain() throws Exception {
	boolean hasConfigurers = webSecurityConfigurers != null
				&& !webSecurityConfigurers.isEmpty();
	if (!hasConfigurers) {
		WebSecurityConfigurerAdapter adapter = objectObjectPostProcessor
					.postProcess(new WebSecurityConfigurerAdapter() {
					});
		webSecurity.apply(adapter);
	}
	return webSecurity.build();
}
```

#### AuthenticationConfiguration

AuthenticationConfiguration的主要任务是负责生成全局的身份认证管理者AuthenticationManager.

```java
@Configuration
@Import(ObjectPostProcessorConfiguration.class)
public class AuthenticationConfiguration {

	@Bean
	public AuthenticationManagerBuilder authenticationManagerBuilder(
	ObjectPostProcessor<Object> objectPostProcessor, ApplicationContext context) {
		LazyPasswordEncoder defaultPasswordEncoder = new LazyPasswordEncoder(context);
		AuthenticationEventPublisher authenticationEventPublisher = getBeanOrNull(context, AuthenticationEventPublisher.class);

		DefaultPasswordEncoderAuthenticationManagerBuilder result = new DefaultPasswordEncoderAuthenticationManagerBuilder(objectPostProcessor, defaultPasswordEncoder);
		if (authenticationEventPublisher != null) {
			result.authenticationEventPublisher(authenticationEventPublisher);
		}
		return result;
	}

}
```

### 2. WebSecurityConfigurerAdapter

适配器模式在spring中被广泛的使用，在配置中使用Adapter的好处是我们可以选择性的配置想要修改的那一部分配置，而不用覆盖其它不相关的配置。WebSecurityConfigurerAdapter提供了三个configure重载方法来供我们配置：

* HttpSecurity常用配置：


```java
@Configuration
@EnableWebSecurity
public class CustomWebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/resources/**", "/signup", "/about").permitAll()
                .antMatchers("/admin/**").hasRole("ADMIN")
                .antMatchers("/db/**").access("hasRole('ADMIN') and hasRole('DBA')")
                .anyRequest().authenticated()
                .and()
            .formLogin()
                .usernameParameter("username")
                .passwordParameter("password")
                .failureForwardUrl("/login?error")
                .loginPage("/login")
                .permitAll()
                .and()
            .logout()
                .logoutUrl("/logout")
                .logoutSuccessUrl("/index")
                .permitAll()
                .and()
            .httpBasic()
                .disable();
    }
}
```

上面是一个使用Java Configuration配置HttpSecurity的典型配置，其中http作为根开始配置，每一个and() 对应一个模块的配置，并且and()返回了HttpSecurity本身，形成了一个调用链。

1. authorizeRequests()配置路径拦截。authenticated：表明访问该路径需要进行权限认证；permitAll允许访问资源，不需要进行权限认证。
2. formLogin()：对应表单认证相关的配置。
3. logout()：对应注销的相关配置。

* WebSecurity

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    public void configure(WebSecurity web) throws Exception {
        web
            .ignoring()
            .antMatchers("/resources/**");
    }
}
```

* AuthenticationManagerBuilder

```java

@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth
            .inMemoryAuthentication()
            .withUser("admin").password("admin").roles("USER");
    }

```

如果要在WebSecurityConfigurerAdapter中进行认证相关的配置，可以使用configure(AuthenticationManagerBuilder auth)暴露一个AuthenticationManager的构造器

:AuthenticationManagerBuilder。如上：我们便完成了内存中用户的配置。





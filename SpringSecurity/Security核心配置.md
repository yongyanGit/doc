### Spring Security核心配置

#### 1. 功能介绍

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/", "/home").permitAll()
                .anyRequest().authenticated()
                .and()
            .formLogin()
                .loginPage("/login")
                .permitAll()
                .and()
            .logout()
                .permitAll();
    }

    @Bean
    @Override
    public UserDetailsService userDetailsService() {
        UserDetails user =
             User.withDefaultPasswordEncoder()
                .username("user")
                .password("password")
                .roles("USER")
                .build();

        return new InMemoryUserDetailsManager(user);
    }
}
```

经过上述的配置后，我们的应用便具备了如下功能：

* 除了"/"、“home”、"login"、"logout"之外，其它路径都需要认证。
* 指定“/login”该路径为登陆页面，当未认证的用户尝试访问任何受保护的资源时，都会跳到“/login”。
* 默认指定"/logout"为注销页面。
* 配置一个内存中的用户认证器，使用user/password作为用户名和密码，具有user角色。

#### 1. @EnableWebSecurity

我们在定义配置类WebSecurityConfig加上@EnableWebSecurity注解，同时继承了WebSecurityConfigurerAdapter。如下，EnableWebSecurity注解的内容：

```java
@Import({ WebSecurityConfiguration.class,
		SpringWebMvcImportSelector.class })
@EnableGlobalAuthentication
@Configuration
public @interface EnableWebSecurity {
	boolean debug() default false;
}
```

@Import是springboot提供的用于引入外部的配置的注解，可以理解为@EnableWebSecurity激活了@Import注解中包含的配置类。

* WebSecurityConfiguration：用来配置web安全。
* SpringWebMvcImportSelector：判断当前环境释放包含springmvc，因为spring security可以在非spring环境中使用，为了避免DispathcherServlet的重复配置。
* EnableGlobalAuthentication注解源码如下：

```java
@Import(AuthenticationConfiguration.class)
@Configuration
public @interface EnableGlobalAuthentication {
}
```

同样在@Import中导入了AuthenticationConfiguration用来配置认证相关的核心类。

也就是说，@EnableWebSecurity完成的工作便是加载WebSecurityConfiguration、AuthenticationConfiguration这两个核心配置类，也就此将Spring Security的职责划分配置安全信息、配置认证信息两部分。

#### WebSecurityConfiguration

在这个配置类中有一个非常重要的Bean被注册了,它是Spring Security的核心过滤器，是整个认证的入口。它在WebsecurityConfiguration声明，并且负责拦截请求的任务交给了DelegatingFilterProxy这个代理类。

```java
@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
public Filter springSecurityFilterChain() throws Exception {
	...
}
```

#### AuthenticationConfiguration

AuthenticationConfiguration的主要任务是负责生成全局的身份认证管理者AuthenticationManager。

```java
@Configuration
@Import(ObjectPostProcessorConfiguration.class)
public class AuthenticationConfiguration {

	@Bean
	public AuthenticationManagerBuilder authenticationManagerBuilder(
	ObjectPostProcessor<Object> objectPostProcessor, ApplicationContext context) {
		...
	}
}

//获取AuthenticationManager
public AuthenticationManager getAuthenticationManager() throws Exception {
		
		authenticationManager = authBuilder.build();
		return authenticationManager;
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
4. httpBasic()可以配置basic登陆。

* WebSecurity，可以设置忽略某些资源，不进行资源认证。

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

如果要在WebSecurityConfigurerAdapter中进行认证相关的配置，可以使用configure(AuthenticationManagerBuilder auth)方法，在它里面构造一个用户。





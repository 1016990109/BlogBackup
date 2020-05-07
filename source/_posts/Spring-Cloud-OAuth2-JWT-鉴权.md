---
title: Spring Cloud OAuth2 + JWT 鉴权
date: 2019-05-29 23:30:21
tags:
  - Spring Cloud
  - OAuth2
  - JWT
  - 鉴权
---

我们希望自己的微服务能够在用户登录之后才可以访问，而单独给每个微服务单独做用户权限模块就显得很弱了，从复用角度来说是需要重构的，从功能角度来说，也是欠缺的。尤其是前后端完全分离之后，我们的用户信息不一定存在于 `Session` 会话中，本文使用 `Spring OAuth2` + `JWT` 来实现鉴权服务。

<!-- more -->

## OAuth2

`OAuth2` 根据使用场景不同，分成了 4 种模式：

- 授权码模式（authorization code）
- 简化模式（implicit）
- 密码模式（resource owner password credentials）
- 客户端模式（client credentials）

关于 `OAuth2` 更详细的信息科查看 [理解 OAuth 2.0](https://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)。

## Maven 依赖

需要用到 `Spring Security`（提供 `OAuth2` 和 `JWT`）、`Spring Web`（提供授权、鉴权接口，即 `endpoints`）、`MySQL Connector`（连接 `OAuth Client` 存储数据）、`Spring JDBC`（连接 `OAuth Client` 存储数据）、`Spring Redis`（`Token` 缓存在 `Redis` 中）、`Spring OpenFeign`（用于请求用户服务验证用户名密码是否正确）等等：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.13</version>
</dependency>
```

## WebSecurityConfigurerAdapter 配置

为了能正常授权及鉴权，需要允许 `OAuth2` 的 `endpoints` 接口，以 `/oauth/**` 形式的接口：

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    private CustomAuthenticationProvider customAuthenticationProvider;

    @Autowired
    public WebSecurityConfig(CustomAuthenticationProvider customAuthenticationProvider) {
        this.customAuthenticationProvider = customAuthenticationProvider;
    }

    @Override
    @Bean
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/oauth/*")
                .permitAll()
                .anyRequest()
                .authenticated()
                .and().formLogin().permitAll()
                .and().logout()
                .logoutUrl("/logout")
                .logoutSuccessHandler(new HttpStatusReturningLogoutSuccessHandler())
                .and()
                .csrf().disable();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) {
        auth.authenticationProvider(customAuthenticationProvider);
    }

    /**
     * 使用自定义的 PasswordEncoder，NoOpPasswordEncoder 已经过时，自动验证 client 和 user（如果使用默认的的用户密码验证的话，也就是 DaoAuthenticationProvider） 的密码
     * 竟然用的同一套 encoder？
     * 建议使用 BCryptPasswordEncoder，这里为了方便和 client 数据表的 secret 比较所以直接存储明文
     * @return
     */
    @Bean
    public PasswordEncoder passwordEncoder() {
        return Oauth2ClientPasswordEncoder.getInstance();
    }
}

/**
 * 自定义的密码加密方法，这里使用原生字符串（不加密）
 * ！！！推荐还是使用 BCryptPassEncoder
 */
public class Oauth2ClientPasswordEncoder implements PasswordEncoder {

    @Override
    public String encode(CharSequence rawPassword) {
        return rawPassword.toString();
    }

    @Override
    public boolean matches(CharSequence rawPassword, String encodedPassword) {
        return rawPassword.toString().equals(encodedPassword);
    }

    public static PasswordEncoder getInstance() {
        return INSTANCE;
    }

    private static final PasswordEncoder INSTANCE = new Oauth2ClientPasswordEncoder();

    private Oauth2ClientPasswordEncoder() {
    }
}
```

## 自定义验证用户名密码

可以发现在 `WebSecurityConfigurerAdapter` 中配置了 `AuthenticationProvider`，这个就是自定义用户名密码验证。

`Provider` 的类型有很多种，这里讲述两种：

- `AnonymousAuthenticationProvider`：顾名思义，匿名的 `provider`，默认的 `provider`，默认是拒绝所有未授权请求的，需要配置开放的接口。
- `DaoAuthenticationProvider`：用于验证 `UserNamePasswordAuthenticationToken`，也就是“用户”和“client”验证时所使用的 `token`。

而自定义的 `CustomAuthenticationProvider` 代码如下：

```java
@Component
public class CustomAuthenticationProvider implements AuthenticationProvider, InitializingBean, MessageSourceAware {
    private final Log logger = LogFactory.getLog(this.getClass());
    /**
     * 可考虑缓存用户
     */
    private MessageSourceAccessor messages = SpringSecurityMessageSource.getAccessor();

    private UserServerClient userServerClient;

    @Autowired
    public CustomAuthenticationProvider(UserServerClient userServerClient) {
        this.userServerClient = userServerClient;
    }

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication, () -> this.messages.getMessage("CustomAuthenticationProvider.onlySupports", "Only UsernamePasswordAuthenticationToken is supported"));

        String username = authentication.getPrincipal() == null ? "NONE_PROVIDED" : authentication.getName();
        if (authentication.getCredentials() == null) {
            this.logger.debug("Authentication failed: no credentials provided");
            throw new BadCredentialsException(this.messages.getMessage("CustomAuthenticationProvider.badCredentials", "Bad credentials"));
        } else {
            UserDTO user = userServerClient.checkUserPassword(new LoginFormBuilder().setUsername(username).setPassword(authentication.getCredentials().toString()).createLoginForm());
            if (user == null) {
                this.logger.debug("Authentication failed: password does not match stored value");
                throw new BadCredentialsException(this.messages.getMessage("CustomAuthenticationProvider.badCredentials", "Bad credentials"));
            }

            CustomUser customUser = new CustomUser.CustomUserBuilder()
                    .setId(user.getId())
                    .setPermissions(user.getPermissions())
                    .setUsername(user.getUsername())
                    .setPassword((String) authentication.getCredentials())
                    .setAuthorities(AuthorityUtils.createAuthorityList("USER"))
                    .createCustomUser();

            return new CustomAuthenticationToken(customUser);
        }
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        Assert.notNull(this.messages, "A message source must be set");
    }

    @Override
    public void setMessageSource(MessageSource messageSource) {
        this.messages = new MessageSourceAccessor(messageSource);
    }
}
```



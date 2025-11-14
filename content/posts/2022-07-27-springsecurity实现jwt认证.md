---
title: SpringSecurity实现JWT认证
date: 2022-07-27T13:41:28+00:00
categories:
  - Java
  - SpringBoot
  - 编程
tags:
  - jwt
---

## 什么是JWT？

JWT全称Json web token，是目前较为流行的认证方案，相较于传统基于Session和Cookie的认证机制，服务端无状态，可扩展性好

有个比喻可以可以很贴切的表示Jwt和Session的区别：

> Session机制是把一把钥匙保存在客户端，请求的时候送到服务端，用来打开服务端的保险箱（当然，服务端的保险箱一般情况下都没有加密）；
> 
> JWT机制就是把一个保险箱放在客户端，请求的时候把整个保险箱送到服务端，服务端使用自己独有的钥匙打开保险箱获得或者更新里面的东西

JWT主要由三部分组成：Header、Payload 和 Signature

header: 包含令牌的类型，以及使用的加密算法

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

payload: 包含一些自定义的数据，以及可选的预定义好的建议使用的key（比如iss、exp等）

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022
}
```

Signature：签名主要是将header和payload进行base64Url编码后使用“.”拼接起来的字符进行Hash运算，Hash算法即header中指定好的算法

```generic
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),  
  your-256-bit-secret
)
```

最后，将三者合在一起即得到最终的Jwt Token

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

## SpringSecurity实现JWT认证

首先，需要在POM文件中引入相关的依赖项目

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.6.7</version>
		<relativePath /> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.datasource</groupId>
	<artifactId>demo</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>demo</name>
	<description>Demo project for Spring Boot</description>
	<properties>
		<java.version>8</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.mybatis.spring.boot</groupId>
			<artifactId>mybatis-spring-boot-starter</artifactId>
			<version>2.2.2</version>
		</dependency>

		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<optional>true</optional>
		</dependency>
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>druid-spring-boot-starter</artifactId>
			<version>1.2.8</version>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>
		<dependency>
			<groupId>io.jsonwebtoken</groupId>
			<artifactId>jjwt</artifactId>
			<version>0.9.1</version>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
				<configuration>
					<excludes>
						<exclude>
							<groupId>org.projectlombok</groupId>
							<artifactId>lombok</artifactId>
						</exclude>
					</excludes>
				</configuration>
			</plugin>
		</plugins>
	</build>

</project>
```

SpringSecurity的具体配置如下，这里将**/error**放入白名单的目的在于获悉报错信息，SpringBoot对于异常的默认处理机制是将报错信息路由到**/error**处理，但是由于SpringSecurity的拦截，最终系统抛出异常的时候，只会得到401响应，而无有用的信息。

issue见https://github.com/spring-projects/spring-security/issues/4467

可以看到具体的配置信息相较于上一篇文章并无异同，只不过增加了jwtRequestFilter用于token的过滤认证，以及认证失败后的异常处理

```java
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter{
    
    @Autowired
    private MyUserDetailsService userDetailsService;

    @Autowired
    private AuthenticationEntryPoint unauthorizedHandler;

    @Autowired
    private JwtRequestFilter jwtRequestFilter;

    @Override
	protected void configure(HttpSecurity http) throws Exception {
		http
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
        http.csrf().disable();
        http.authorizeRequests().antMatchers("/api/login", "/api/register", "/error").permitAll();
        http.authorizeRequests().anyRequest().authenticated();
        http.exceptionHandling().authenticationEntryPoint(unauthorizedHandler);
        http.addFilterBefore(jwtRequestFilter, UsernamePasswordAuthenticationFilter.class);
	}

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService).passwordEncoder(new BCryptPasswordEncoder());
    }

    @Override
    @Bean
    protected AuthenticationManager authenticationManager() throws Exception {
        return super.authenticationManager();
    }
    
}
```

新建TokenService类，负责token的生成校验等

```java
@Component
public class TokenService {
    
    @Value("${token.secret}")
    private String secret;

    private static final long JWT_TOKEN_VALIDITY = 5 * 60 * 60;


    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        return doGenerateToken(claims, userDetails.getUsername());
    }

    public Boolean validateToken(String token, UserDetails userDetails) {
        String username = getUsernameFromToken(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }

    private Boolean isTokenExpired(String token) {
        Date expiration = getExpirationDateFromToken(token);
        return expiration.before(new Date());
    }

    public String getUsernameFromToken(String token) {
        return getClaimFromToken(token, Claims::getSubject);
    }

    public Date getExpirationDateFromToken(String token) {
        return getClaimFromToken(token, Claims::getExpiration);
    }


    public <T> T getClaimFromToken(String token, Function<Claims, T> claimsResolver) {
        Claims claims = getAllClaimsFromToken(token);
        return claimsResolver.apply(claims);
    }

    private Claims getAllClaimsFromToken(String token) {
        return Jwts.parser().setSigningKey(secret).parseClaimsJws(token).getBody(); 
    }

    private String doGenerateToken(Map<String, Object> claims, String subject) {
        return Jwts.builder().setClaims(claims)
                    .setSubject(subject)
                    .setIssuedAt(new Date(System.currentTimeMillis()))
                    .setExpiration(new Date(System.currentTimeMillis() + JWT_TOKEN_VALIDITY * 1000))
                    .signWith(SignatureAlgorithm.HS512, secret).compact();

    }
}
```

这里有个比较有意思的实现**getClaimFromToken**，使用了函数式编程的思想，传入需要使用的方法，得到想要的属性信息

UserDetailService的实现类如下，这里使用了默认的User类作为返回，避免破坏自定义的UserEntity继承关系，实现解耦

```java
@Service
public class MyUserDetailsService implements UserDetailsService{

    @Autowired
    private UserMapper userMapper;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        UserEntity userEntity = userMapper.selectByUsername(username);
        if (userEntity == null) throw new UsernameNotFoundException("username not found");
        return new User(userEntity.getUsername(), userEntity.getPassword(), AuthorityUtils.NO_AUTHORITIES);
    }

}
```

认证异常处理类如下：

```java
@Component
public class UnauthorizedHandler implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
            AuthenticationException authException) throws IOException, ServletException {
            HashMap<String, String> map = new HashMap<>(2); 
            map.put("uri", request.getRequestURI());
            map.put("msg", "认证失败"); 
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.setCharacterEncoding("utf-8");
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            ObjectMapper objectMapper = new ObjectMapper();
            String resBody = objectMapper.writeValueAsString(map);
            PrintWriter printWriter = response.getWriter();
            printWriter.print(resBody);
            printWriter.flush();
            printWriter.close();
    }


}
```

在未认证或认证失败的时候将会返回如下信息，这是由该异常处理类决定的

```shell
➜  ~ curl -i localhost:8080/api/hello
HTTP/1.1 401
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
X-Frame-Options: DENY
Content-Type: application/json;charset=utf-8
Transfer-Encoding: chunked
Date: Wed, 27 Jul 2022 13:16:38 GMT

{"msg":"认证失败","uri":"/api/hello"}
```

接下来看下具体的LoginController，这里authenticationManager.authenticate方法认证通过后将会调用tokenServie返回生成的token，否则将会抛出异常由UnauthorizedHandler处理，这里有个注意点，如果没有该异常处理器，SpringBoot并不会主动处理该异常，生成报错信息。

```java
@RestController
@RequestMapping("/api")
public class LoginController {
    
    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private TokenService tokenService;

    @Autowired
    private UserDetailsService myUserDetailsService;

    @PostMapping("/login")
    public String login(@RequestBody UserEntity userEntity) {
        Authentication token = new UsernamePasswordAuthenticationToken(userEntity.getUsername(), userEntity.getPassword());
        authenticationManager.authenticate(token);
        UserDetails userDetails = myUserDetailsService.loadUserByUsername(userEntity.getUsername());
        return tokenService.generateToken(userDetails);
    }

    @GetMapping("/hello")
    public String hello() {
        return "hello";
    }

}
```

接下来输入正确的用户名密码进行测试

```shell
curl -i -X POST -H "Content-Type: application/json" -d '{"username": "admin", "password": "123"}' localhost:8080/api/login
```

可以看到成功的返回了token，接下来使用该token访问hello，测试jwtRequestFilter是否正常认证

```shell
➜  ~ curl -i -H "Content-Type: application/json;" -H "Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJhZG1pbiIsImV4cCI6MTY1ODk0NjI4MSwiaWF0IjoxNjU4OTI4MjgxfQ.5G5XCEBM3xkXqcTMoTGNesNrz5N8F77Ds1vE2atKEDjtef-_c11aVTxzTYSQZGZi04URxAXtn70uYB_Tc9UQhA" localhost:8080/api/hello
HTTP/1.1 200
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
X-Frame-Options: DENY
Content-Type: text/plain;charset=UTF-8
Content-Length: 5
Date: Wed, 27 Jul 2022 13:36:33 GMT

hello
```

## 参考

1. [JSON Web Token 入门教程](https://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html)

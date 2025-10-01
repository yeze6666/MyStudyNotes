## SpringBoot 的基本设置

#### 1.  jjwt和MyBatisPlus的依赖

```java
<!--mybatisplus-->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.4.2</version>
</dependency>


<!--jjwt-->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-gson</artifactId>
    <version>0.12.6</version>
</dependency>
    
<!--Apache Commons 工具包-->
<dependency>
	<groupId>org.apache.commons</groupId>
    <artifactId>commons-compress</artifactId>
    version>1.26.2</version>
</dependency>
```

####  2. Mysql数据库连接

###### properties格式

```java
server.port=8080
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/mpdemo?serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.max-idle=10
spring.datasource.max-wait=10000
spring.datasource.min-idle=5
spring.datasource.initial-size=5
```



##### yaml格式

``` java
server:
  port: 程序运行在那个端口（一般：8080）
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/springboot
    username: root
    password: Yeze2006
```

```java
server:
  port: 程序运行在那个端口（一般：8080）
spring:
  datasource:
    url: jdbc:jtds:sqlserver:// IP : 端口 ;DatabaseName= 某个库名 ;trustServerCertificate=true
    driver-class-name: net.sourceforge.jtds.jdbc.Driver
    username: 123 (填写自己的账号)
    password: 123 (填写自己的密码）
    hikari:
      connection-test-query: SELECT 1
```





#### 3. Token的工具包

```java
package com.example.text; //这里改为你自己的包名字，不要忘记了哦！！！

import io.jsonwebtoken.*;
import io.jsonwebtoken.io.Decoders;
import io.jsonwebtoken.security.Keys;

import javax.crypto.SecretKey;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;

/**
 * Token 工具类
 */
public class TokenUtil {

    // 建议使用 Base64 编码后的随机密钥（至少 256bit 才能保证 HS256 的安全性）
    private static final String SECRET = "yoursecretkeyyoursecretkeyyoursecretkey123"; //这个密钥自己要改一下哦
    private static final long EXPIRE_TIME = 1000 * 60 * 60 * 24; // 24小时过期，这里也可以自己改一下

    private static SecretKey getSecretKey() {
        byte[] keyBytes = Decoders.BASE64.decode(SECRET);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    /**
     * 生成Token
     * @param userId 用户ID
     * @return token字符串
     */
    public static String generateToken(Long userId) {
        //这里userId传入的是Long类型，也有可能是int，这里可以自己改一下
        Map<String, Object> claims = new HashMap<>();
        claims.put("userId", userId);

        return Jwts.builder()
                .claims(claims)
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + EXPIRE_TIME))
                .signWith(getSecretKey(), Jwts.SIG.HS256)  // 指定算法和密钥
                .compact();
    }

    /**
     * 解析Token
     * @param token token字符串
     * @return Claims
     */
    public static Claims parseToken(String token) {
        return Jwts.parser()
                .verifyWith(getSecretKey())
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }

    /**
     * 获取Token中的userId
     * @param token token字符串
     * @return 用户ID
     */
    public static Long getUserIdFromToken(String token) {
        Claims claims = parseToken(token);
        return claims.get("userId", Long.class);
    }

    /**
     * 判断Token是否合法（未过期 + 签名正确）
     * @param token token字符串
     * @return true 合法，false 不合法
     */
    public static boolean validateToken(String token) {
        try {
            Claims claims = parseToken(token);
            return claims.getExpiration().after(new Date());
        } catch (JwtException | IllegalArgumentException e) {
            return false; // 解析失败或过期
        }
    }
    
     /**
     * 刷新Token
     * 如果Token快过期（比如剩余时间小于30分钟），则生成新的Token
     * 否则直接返回原来的Token
     *
     * @param token 旧的Token
     * @return 新的Token（或原Token）
     */
    public static String refreshToken(String token) {
        try {
            Claims claims = parseToken(token);
            Date expiration = claims.getExpiration();
            long now = System.currentTimeMillis();
            long remainingTime = expiration.getTime() - now;

            // 设定一个“刷新阈值”，比如30分钟
            long refreshThreshold = 1000 * 60 * 30;

            if (remainingTime <= refreshThreshold) {
                // 重新生成新的Token，保留userId
                Long userId = claims.get("userId", Long.class);
                return generateToken(userId);
            } else {
                // 还没接近过期，直接返回旧的
                return token;
            }
        } catch (JwtException | IllegalArgumentException e) {
            // 如果Token不合法，直接返回null（也可以抛异常，看你业务需要）
            return null;
        }
    }


}

```





####  3. AdminInterceptor拦截器的代码

```java
package com.example.rent.Interceptor; //这里改为自己的包


import com.example.rent.Mapper.AdminMapper;
import com.example.rent.Utils.TokenUtil;
import io.jsonwebtoken.Claims;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

@Component
public class AdminInterceptor implements HandlerInterceptor {

    @Autowired
    private AdminMapper adminMapper;  // 用于查询数据库里的用户信息

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String header = request.getHeader("Authorization");

        // 没有 Authorization
        if (header == null || header.isBlank()) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED); // 401
            return false;
        }

        // 必须是 Bearer 开头
        if (!header.startsWith("Bearer ")) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED); // 401
            return false;
        }

        String token = header.substring(7);

        // 验证 token
        if (!TokenUtil.validateToken(token)) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED); // 401
            return false;
        }

        try {
            Claims claims = TokenUtil.parseToken(token);
            Long userId = claims.get("userId", Long.class);

            // 查询数据库里的用户信息
            String username = adminMapper.selectById(userId).getUsername();

            // 校验是否 admin
            if (!"admin".equalsIgnoreCase(username)) {
                response.setStatus(HttpServletResponse.SC_FORBIDDEN); // 403
                return false;
            }

            // 存放 userId，后面 controller 可以直接用
            request.setAttribute("userId", userId);

        } catch (Exception e) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return false;
        }

        return true; // 校验通过，放行
    }
}

```



#### 4. InterceptorConfig 的配置

```java
package com.example.rent.Config; //这里改为自己的包


import com.example.rent.Interceptor.AdminInterceptor;
import jakarta.annotation.Resource;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class InterceptorConfig  implements WebMvcConfigurer {

    @Resource
    private AdminInterceptor adminInterceptor;

    //  这里给/admin/下的所有api接口添加拦截器，放在黑客入侵！！！
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(adminInterceptor).addPathPatterns("/admin/**");
    }
}
```


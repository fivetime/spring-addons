# 用于 Google reCAPTCHA 验证的 Spring Boot Starter
用于在 Spring Boot 应用中验证客户端提交的 Google reCAPTCHA 的轻量级库。

## 使用方法
得益于 `@AutoConfiguration` 的魔力，只需三个简单步骤：

### 将本库添加到 classpath
```xml
        <dependency>
            <groupId>com.c4-soft.springaddons.starter</groupId>
            <artifactId>spring-addons-starters-recaptcha</artifactId>
            <version>${spring-addons.version}</version>
        </dependency>
```

### 声明几个配置属性（`secret-key` 的值需从 https://www.google.com/recaptcha/admin/site 获取）
```properties
com.c4-soft.springaddons.recaptcha.secret-key=machin
com.c4-soft.springaddons.recaptcha.siteverify-url=https://localhost/recaptcha/api/siteverify
com.c4-soft.springaddons.recaptcha.v3-threshold=0.8
```

### 在需要的地方注入 `ReCaptchaValidationService`
```java
@RestController
@RequestMapping("/greet")
@RequiredArgsConstructor
public class GreetingController {
    private final ReCaptchaValidationService captcha;

    @GetMapping("/{who}")
    public Mono<String> greet(@PathVariable("who") String who, @RequestParam("reCaptcha") String reCaptcha) {
        return captcha.checkV2(reCaptcha).map(isHuman -> Boolean.TRUE.equals(isHuman) ? String.format("Hi %s", who) : "Hello Mr. Robot");
    }
}
```

## 代理配置

本库依赖 `spring-addons-starters-webclient` 向验证服务器发起 HTTP 请求。因此，可以通过 `com.c4-soft.springaddons.proxy.*` 配置属性或 `HTTP_PROXY`、`NO_PROXY` 标准环境变量来配置代理：
```properties
com.c4-soft.springaddons.proxy.hostname=http://localhost
com.c4-soft.springaddons.proxy.port=8080
# 更多配置项请通过 IDE 自动补全查看
```
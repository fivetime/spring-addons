# OAuth2 Backend for Frontend 与 Spring Cloud Gateway 使用指南

> 原文来源：Baeldung｜最后更新：2025年12月27日

## 1. 概述

本教程将介绍如何使用 Spring Cloud Gateway 与 spring-addons 实现 OAuth2 Backend for Frontend（BFF）模式，以便三个不同的单页应用（Angular、React 和 Vue）能够消费一个无状态的 REST API。

使用调试工具检查 Google、Facebook、Github、LinkedIn 等主流网站的请求时，你会发现网络请求中找不到任何 Bearer 令牌，这是为什么？

安全专家指出，即便启用了 PKCE，我们也**不应该**将运行在用户设备上的应用配置为"公开（public）"OAuth2 客户端。推荐的替代方案是：在我们信任的服务器上运行一个 BFF，使用 Session 来为移动端和 Web 端应用授权。

本教程将演示 SPA 如何通过 OAuth2 BFF 轻松消费 REST API，以及现有资源服务器（使用 Bearer 访问令牌授权的无状态 REST API）**无需任何改动**即可接入。

---

## 2. OAuth2 Backend for Frontend 模式

在进入实现细节之前，先来了解 OAuth2 BFF 是什么、它带来了什么价值，以及代价是什么。

### 2.1 定义

Backend for Frontend（BFF）是前端与 REST API 之间的一个中间层，可以服务于不同的目的。本文关注的是 OAuth2 BFF——它在前端的 Session Cookie 授权与资源服务器期望的 Bearer 令牌授权之间起到桥梁作用。

其主要职责包括：

- 使用"机密（confidential）"OAuth2 客户端驱动授权码流程和刷新令牌流程
- 维护 Session 并将令牌存储其中
- 在将前端请求转发给资源服务器前，将 Session Cookie 替换为 Session 中的访问令牌

### 2.2 相较于公开 OAuth2 客户端的优势

主要优势在于**安全性**：

- BFF 运行在我们信任的服务器上，授权服务器的令牌端点可以通过密钥和防火墙规则保护，只允许来自后端的请求，大幅降低令牌被恶意客户端获取的风险。
- 令牌保存在服务器端（Session 中），防止其在终端用户设备上被恶意程序窃取。
- Session Cookie 需要 CSRF 防护，但 Cookie 可以标记 `HttpOnly`、`Secure` 和 `SameSite`，这样设备上的 Cookie 保护由浏览器本身来执行。相比之下，将 SPA 配置为公开客户端时，SPA 必须能访问令牌，这对令牌存储方式提出了很高的安全要求——一旦恶意程序读取到访问令牌，后果可能非常严重；刷新令牌被盗则更糟，身份冒用可能持续很长时间。

另一个优势是对用户 Session 的**完全掌控**，以及即时撤销访问的能力。JWT 无法被主动失效，存储在终端用户设备上的令牌也几乎无法在服务器端终止 Session 时一并删除。若将 JWT 访问令牌通过网络发送出去，我们唯一能做的就是等它过期，在此之前资源服务器仍会持续授权访问。但如果令牌从不离开后端，我们就可以随用户 Session 一起删除令牌，**立即撤销对资源的访问**。

### 2.3 代价

BFF 是系统中的额外一层，处于关键路径上。在生产环境中，这意味着略多的资源消耗和略高的延迟，同时也需要监控。

此外，BFF 后面的资源服务器可以（也应该）是无状态的，但 OAuth2 BFF 本身需要 Session，这对其可扩展性和容错性提出了要求。

Spring Cloud Gateway 可以轻松打包为原生镜像，使其极其轻量且启动迅速，但单实例能承载的流量始终有上限。当流量增加时，需要在多个 BFF 实例间共享 Session，此时 Spring Session 项目会很有帮助。另一种选择是使用智能代理，将同一设备的所有请求路由到同一个 BFF 实例。

### 2.4 实现方案的选择

有些框架实现了 OAuth2 BFF 模式，但并未明确宣传或以此命名。例如 NextAuth 库就使用服务端组件来实现 OAuth2（在服务器上的 Node 实例中作为机密客户端运行），同样能够获得 OAuth2 BFF 模式的安全优势。

但在监控、可扩展性和容错性方面，几乎没有什么方案能比 Spring Cloud Gateway 更得心应手：

- `spring-boot-starter-actuator` 提供强大的审计功能
- Spring Session 是相对简单的分布式 Session 解决方案
- `spring-boot-starter-oauth2-client` 与 `oauth2Login()` 处理授权码流程和刷新令牌流程，并将令牌存储在 Session 中
- `TokenRelay=` 过滤器在将前端请求转发给资源服务器时，将 Session Cookie 替换为 Session 中的访问令牌

---

## 3. 架构

前面列举了不少服务：前端（SPA）、REST API、BFF 和授权服务器。下面来看这些服务如何构成一个完整的系统。

### 3.1 系统概览

下图展示了使用主（main）Profile 时各服务的端口和路径前缀：

![系统架构图](<images/OAuth2 BFF With Spring Cloud Gateway/arch.jpg>)

需要特别注意两点：

- 从终端用户设备的角度来看，BFF 和 SPA 静态资源至少共用一个接入点：反向代理。
- 资源服务器通过 BFF 访问。

将授权服务器置于反向代理之后是可选的。在类生产环境中，可以使用子域名替代路径前缀来区分不同 SPA。

### 3.2 快速启动

配套代码仓库包含一个构建脚本，可用于构建并启动上述各服务的 Docker 镜像。

要运行整个系统，需确保：

- 路径中存在 JDK 17 到 21 之间的版本（可运行 `java --version` 验证）
- Docker Desktop 已安装并运行
- 路径中存在最新 LTS 版本的 Node（推荐使用 nvm 或 nvm-windows 管理）

然后运行以下 Shell 脚本（Windows 用户可使用 Git Bash）：

```bash
git clone https://github.com/eugenp/tutorials.git
cd tutorials/spring-security-modules/spring-security-oauth2-bff/
sh ./build.sh
```

后续章节将介绍如何用本地服务替换各个容器。

---

## 4. 使用 Spring Cloud Gateway 与 spring-addons-starter-oidc 实现 BFF

首先，使用 IDE 或 [https://start.spring.io/](https://start.spring.io/) 创建一个名为 `bff` 的新 Spring Boot 项目，依赖选择 **Reactive Gateway** 和 **OAuth2 Client**。

然后将 `src/main/resources/application.properties` 重命名为 `src/main/resources/application.yml`。

最后，在依赖中添加 `spring-addons-starter-oidc`：

```xml
<dependency>
    <groupId>com.c4-soft.springaddons</groupId>
    <artifactId>spring-addons-starter-oidc</artifactId>
    <version>7.7.0</version>
</dependency>
```

### 4.1 公共属性

先在 `application.yml` 中定义一些常量，便于在其他配置节中引用，也方便在命令行或 IDE 启动配置中覆盖：

```yaml
scheme: http
hostname: localhost
reverse-proxy-port: 7080
reverse-proxy-uri: ${scheme}://${hostname}:${reverse-proxy-port}
authorization-server-prefix: /auth
issuer: ${reverse-proxy-uri}${authorization-server-prefix}/realms/baeldung
client-id: baeldung-confidential
client-secret: secret
username-claim-json-path: $.preferred_username
authorities-json-path: $.realm_access.roles
bff-port: 7081
bff-prefix: /bff
resource-server-port: 7084
audience:
```

`client-secret` 的值在实际使用时需要通过环境变量、命令行参数或 IDE 启动配置来覆盖。

### 4.2 服务器属性

标准的服务器端口配置：

```yaml
server:
  port: ${bff-port}
```

### 4.3 Spring Cloud Gateway 路由配置

由于网关后面只有一个资源服务器，只需定义一条路由：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: bff
        uri: ${scheme}://${hostname}:${resource-server-port}
        predicates:
        - Path=/api/**
        filters:
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
        - TokenRelay=
        - SaveSession
        - StripPrefix=1
```

最核心的部分是 `SaveSession` 和 `TokenRelay=`，它们是 OAuth2 BFF 模式实现的基石。前者确保 Session 被持久化（包含 `oauth2Login()` 获取的令牌），后者在路由请求时将 Session Cookie 替换为 Session 中的访问令牌。

`StripPrefix=1` 过滤器在路由请求时去除路径中的 `/api` 前缀（注意：`/bff` 前缀已在反向代理路由时去除）。因此，前端发往 `/bff/api/me` 的请求，到达资源服务器时变为 `/me`。

### 4.4 Spring Security 配置

使用标准的 Boot 属性配置 OAuth2 客户端安全：

```yaml
spring:
  security:
    oauth2:
      client:
        provider:
          baeldung:
            issuer-uri: ${issuer}
        registration:
          baeldung:
            provider: baeldung
            authorization-grant-type: authorization_code
            client-id: ${client-id}
            client-secret: ${client-secret}
            scope: openid,profile,email,offline_access
```

这里没有什么特别之处，就是标准的 OpenID Provider 声明，配合一个使用授权码和刷新令牌的注册项。

### 4.5 spring-addons-starter-oidc 配置

使用 spring-addons-starter-oidc 对 Spring Security 进行进一步调整：

```yaml
com:
  c4-soft:
    springaddons:
      oidc:
        # 受信任的 OpenID Provider 配置（含权限映射）
        ops:
        - iss: ${issuer}
          authorities:
          - path: ${authorities-json-path}
          aud: ${audience}
        # 含 oauth2Login() 的 SecurityFilterChain（启用 Session 和 CSRF 保护）
        client:
          client-uri: ${reverse-proxy-uri}${bff-prefix}
          security-matchers:
          - /api/**
          - /login/**
          - /oauth2/**
          - /logout
          permit-all:
          - /api/**
          - /login/**
          - /oauth2/**
          csrf: cookie-accessible-from-js
          oauth2-redirections:
            rp-initiated-logout: ACCEPTED
        # 含 oauth2ResourceServer() 的 SecurityFilterChain（禁用 Session 和 CSRF 保护）
        resourceserver:
          permit-all:
          - /login-options
          - /error
          - /actuator/health/readiness
          - /actuator/health/liveness
```

配置分为三个主要部分：

**`ops`（OpenID Provider 配置）**：指定要转换为 Spring 权限的 Claims JSON 路径（可选地添加前缀或大小写转换）。如果 `aud` 属性不为空，spring-addons 会在 JWT 解码器中添加受众（audience）校验器。

**`client`**：当 `security-matchers` 不为空时，该部分触发创建一个带 `oauth2Login()` 的 `SecurityFilterChain` Bean。通过 `client-uri` 属性，强制使用反向代理 URI 作为所有重定向的基础（而非 BFF 内部 URI）。由于使用的是 SPA，需要 BFF 在 JavaScript 可访问的 Cookie 中暴露 CSRF 令牌。最后，为避免 CORS 错误，要求 BFF 以 2xx 状态码响应 RP 发起的登出请求（而非 3xx），这样 SPA 就能拦截该响应，并通过新 Origin 的请求让浏览器处理跳转。

**`resourceserver`**：请求创建第二个带 `oauth2ResourceServer()` 的 `SecurityFilterChain` Bean。该过滤链的 `@Order` 优先级最低，处理所有未被 client 过滤链匹配到的请求——用于处理不需要 Session 的资源端点（不涉及登录或 TokenRelay 路由的端点）。

### 4.6 /login-options 端点

BFF 是定义登录配置的地方（带授权码的 Spring OAuth2 客户端注册）。为避免在每个 SPA 中重复配置（以及可能产生的不一致），我们在 BFF 上暴露一个 REST 端点，向用户提供其支持的登录选项。

只需暴露一个 `@RestController`，包含一个从配置属性构建响应的端点：

```java
@RestController
public class LoginOptionsController {
    private final List<LoginOptionDto> loginOptions;

    public LoginOptionsController(OAuth2ClientProperties clientProps,
                                  SpringAddonsOidcProperties addonsProperties) {
        final var clientAuthority = addonsProperties.getClient()
            .getClientUri().getAuthority();
        this.loginOptions = clientProps.getRegistration()
            .entrySet().stream()
            .filter(e -> "authorization_code".equals(
                e.getValue().getAuthorizationGrantType()))
            .map(e -> {
                final var label = e.getValue().getProvider();
                final var loginUri = "%s/oauth2/authorization/%s"
                    .formatted(addonsProperties.getClient().getClientUri(), e.getKey());
                final var providerId = clientProps.getRegistration()
                    .get(e.getKey()).getProvider();
                final var providerIssuerAuthority = URI.create(
                    clientProps.getProvider().get(providerId).getIssuerUri())
                    .getAuthority();
                return new LoginOptionDto(label, loginUri,
                    Objects.equals(clientAuthority, providerIssuerAuthority));
            })
            .toList();
    }

    @GetMapping(path = "/login-options", produces = MediaType.APPLICATION_JSON_VALUE)
    public Mono<List<LoginOptionDto>> getLoginOptions() throws URISyntaxException {
        return Mono.just(this.loginOptions);
    }

    public static record LoginOptionDto(
        @NotEmpty String label,
        @NotEmpty String loginUri,
        boolean isSameAuthority) {}
}
```

现在可以停止 `baeldung-bff.bff` Docker 容器，在命令行或运行配置中提供以下参数后启动 BFF 应用：

- `hostname`：`hostname` 命令或 `HOSTNAME` 环境变量的值（转为小写）
- `client-secret`：在授权服务器中为 `baeldung-confidential` 客户端声明的密钥（默认为 `secret`）

### 4.7 非标准 RP 发起的登出

RP 发起的登出（RP-Initiated Logout）是 OpenID 标准的一部分，但部分提供商并未严格实现。例如 Auth0 和 Amazon Cognito，它们未在 OpenID 配置中提供 `end_session` 端点，而是使用自定义查询参数来处理登出。

spring-addons-starter-oidc 支持这类"基本符合"标准的登出端点。配套项目的 BFF 配置中包含对应的 Profile：

```yaml
---
spring:
  config:
    activate:
      on-profile: cognito
issuer: https://cognito-idp.us-west-2.amazonaws.com/us-west-2_RzhmgLwjl
client-id: 12olioff63qklfe9nio746es9f
client-secret: change-me
username-claim-json-path: username
authorities-json-path: $.cognito:groups
com:
  c4-soft:
    springaddons:
      oidc:
        client:
          oauth2-logout:
            baeldung:
              uri: https://spring-addons.auth.us-west-2.amazoncognito.com/logout
              client-id-request-param: client_id
              post-logout-uri-request-param: logout_uri

---
spring:
  config:
    activate:
      on-profile: auth0
issuer: https://dev-ch4mpy.eu.auth0.com/
client-id: yWgZDRJLAksXta8BoudYfkF5kus2zv2Q
client-secret: change-me
username-claim-json-path: $['https://c4-soft.com/user']['name']
authorities-json-path: $['https://c4-soft.com/user']['roles']
audience: bff.baeldung.com
com:
  c4-soft:
    springaddons:
      oidc:
        client:
          authorization-params:
            baeldung:
              audience: ${audience}
          oauth2-logout:
            baeldung:
              uri: ${issuer}v2/logout
              client-id-request-param: client_id
              post-logout-uri-request-param: returnTo
```

上面配置中的 `baeldung` 是 Spring Boot 属性中客户端注册的 key。如果在 `spring.security.oauth2.client.registration` 中使用了不同的 key，这里也需要对应修改。

另外，在第二个 Profile（Auth0）中，可以注意到向 Auth0 发送授权请求时附加了额外的请求参数 `audience`。

---

## 5. 使用 spring-addons-starter-oidc 构建资源服务器

本系统对资源服务器的需求很简单：一个使用 JWT 访问令牌授权的无状态 REST API，暴露单个端点以反映令牌中的部分用户信息（若请求未授权，则返回空值）。

创建一个名为 `resource-server` 的新 Spring Boot 项目，依赖选择 **Spring Web** 和 **OAuth2 Resource Server**，同样将配置文件重命名为 `application.yml`，并添加 `spring-addons-starter-oidc` 依赖（同第 4 节）。

### 5.1 配置

资源服务器所需的属性：

```yaml
scheme: http
hostname: localhost
reverse-proxy-port: 7080
reverse-proxy-uri: ${scheme}://${hostname}:${reverse-proxy-port}
authorization-server-prefix: /auth
issuer: ${reverse-proxy-uri}${authorization-server-prefix}/realms/baeldung
username-claim-json-path: $.preferred_username
authorities-json-path: $.realm_access.roles
resource-server-port: 7084
audience:

server:
  port: ${resource-server-port}

com:
  c4-soft:
    springaddons:
      oidc:
        ops:
        - iss: ${issuer}
          username-claim: ${username-claim-json-path}
          authorities:
          - path: ${authorities-json-path}
          aud: ${audience}
        resourceserver:
          permit-all:
          - /me
```

借助 spring-addons-starter-oidc，以上配置就足以声明一个无状态资源服务器，具备：

- 从自定义 Claim 映射权限（Keycloak Realm 角色对应 `realm_access.roles`）
- 允许匿名请求访问 `/me` 端点

配套代码仓库的 `application.yaml` 中还包含其他 OpenID Provider 使用私有 Claim 存储角色时的 Profile 配置。

### 5.2 REST 控制器

实现一个 REST 端点，返回安全上下文中 `Authentication` 的部分数据（如有）：

```java
@RestController
public class MeController {
    @GetMapping("/me")
    public UserInfoDto getMe(Authentication auth) {
        if (auth instanceof JwtAuthenticationToken jwtAuth) {
            final var email = (String) jwtAuth.getTokenAttributes()
                .getOrDefault(StandardClaimNames.EMAIL, "");
            final var roles = auth.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .toList();
            final var exp = Optional.ofNullable(
                jwtAuth.getTokenAttributes().get(JwtClaimNames.EXP))
                .map(expClaim -> {
                    if (expClaim instanceof Long lexp)    return lexp;
                    if (expClaim instanceof Instant iexp) return iexp.getEpochSecond();
                    if (expClaim instanceof Date dexp)    return dexp.toInstant().getEpochSecond();
                    return Long.MAX_VALUE;
                }).orElse(Long.MAX_VALUE);
            return new UserInfoDto(auth.getName(), email, roles, exp);
        }
        return UserInfoDto.ANONYMOUS;
    }

    /**
     * @param username 令牌中资源所有者的唯一标识（默认为 sub Claim）
     * @param email    OpenID email Claim
     * @param roles    安全上下文中当前认证所解析出的 Spring 权限
     * @param exp      访问令牌过期时间（UTC，从 1970-01-01T00:00:00Z 起的秒数）
     */
    public static record UserInfoDto(String username, String email,
                                     List<String> roles, Long exp) {
        public static final UserInfoDto ANONYMOUS =
            new UserInfoDto("", "", List.of(), Long.MAX_VALUE);
    }
}
```

与 BFF 相同，现在可以停止 `baeldung-bff.resource-server` Docker 容器，在命令行或运行配置中提供 `hostname` 参数后启动资源服务器。

### 5.3 资源服务器多租户

如果消费 REST API 的多个前端应用在不同的授权服务器或 Realm 上进行用户认证（或提供多个授权服务器供选择），该如何处理？

使用 spring-security-starter-oidc 时，这非常简单：`com.c4-soft.springaddons.oidc.ops` 配置属性是一个数组，可以添加任意数量的受信 Issuer，并为每个 Issuer 分别配置用户名和权限的映射规则。任何一个受信 Issuer 签发的有效令牌，资源服务器都会接受，且角色均能正确映射为 Spring 权限。

---

## 6. SPA 接入

由于不同框架与 OAuth2 BFF 的接入方式存在差异，我们将分别介绍三个主流框架：Angular、React 和 Vue。

创建 SPA 本身不在本文讨论范围内，这里只聚焦于 Web 应用通过 OAuth2 BFF 实现用户登录/登出，以及调用 BFF 后面 REST API 所需的内容。完整实现请参考配套代码仓库。

各框架的应用结构尽量保持一致：

- 两个路由，用于演示认证后如何恢复当前页面
- **Login 组件**：在 iframe 和默认登录体验均可用时提供选择；处理 iframe 的显示状态，或跳转至授权服务器
- **Logout 组件**：向 BFF 的 `/logout` 端点发送 POST 请求，然后跳转至授权服务器完成 RP 发起的登出
- **UserService**：通过 BFF 从资源服务器获取当前用户数据；包含在访问令牌过期前定时刷新数据的逻辑

由于各框架的状态管理方式差异较大，当前用户数据的管理方式有所不同：

- **Angular**：UserService 是一个单例，使用 `BehaviorSubject` 管理当前用户
- **React**：在 `app/layout.tsx` 中使用 `createContext` 向所有组件暴露当前用户，在需要访问时使用 `useContext`
- **Vue**：UserService 是一个单例（在 `main.ts` 中实例化），使用 `ref` 管理当前用户

### 6.1 运行配套仓库中的 SPA

首先 `cd` 进入要运行的项目目录，然后执行 `npm install` 安装所需 npm 包。停止对应的 Docker 容器后，根据框架分别执行：

- **Angular**：`npm run start`，访问 `http://{hostname}:7080/angular-ui/`
- **Vue**：`npm run dev`，访问 `http://{hostname}:7080/vue-ui/`
- **React（Next.js）**：`npm run dev`，访问 `http://{hostname}:7080/react-ui/`

注意：必须使用指向反向代理的 URL（`http://{hostname}:7080`），而非 SPA 开发服务器的直接地址。

### 6.2 UserService

UserService 的职责是：

- 定义用户表示（内部模型与 DTO）
- 暴露一个函数，通过 BFF 从资源服务器获取用户数据
- 在访问令牌过期前调度 `refresh()` 以保持 Session 活跃

### 6.3 登录

如前所述，在条件允许时，我们提供两种登录体验：

- **默认方式（始终可用）**：在当前浏览器标签中重定向到授权服务器，SPA 暂时"离开"
- **iframe 方式**：在 SPA 内的 iframe 中展示授权服务器表单，要求 SPA 与授权服务器同源，因此仅在使用默认 Profile（Keycloak）时可用

Login 组件展示一个下拉框供用户选择登录体验（iframe 或默认），以及一个登录按钮。组件初始化时从 BFF 获取登录选项，本教程中只期望一个选项，所以取响应数据的第一条。

点击登录按钮后：

- 若选择 iframe 方式，将 iframe 的 src 设置为登录 URI 并显示包含 iframe 的模态框
- 否则，将 `window.location.href` 设置为登录 URI，使 SPA"离开"当前标签并以新 Origin 加载

选择 iframe 登录时，会注册一个监听 iframe load 事件的回调，以检测用户认证是否成功并隐藏模态框（每次 iframe 内发生重定向时触发）。

另外值得注意的是，SPA 在授权码流程发起请求中附加了 `post_login_success_uri` 请求参数。spring-addons-starter-oidc 会将该参数保存在 Session 中，在授权码换取令牌完成后，用它来构建返回给前端的重定向 URI。

### 6.4 登出

登出按钮及相关逻辑由 Logout 组件处理。

Spring 的 `/logout` 端点默认需要 POST 请求，而且与所有修改服务器有状态数据的请求一样，需要包含 CSRF 令牌。Angular 和 React 会自动处理标记为 `http-only=false` 的 CSRF Cookie 和请求头；但在 Vue 中，需要手动读取 `XSRF-TOKEN` Cookie 并为每个 POST、PUT、PATCH、DELETE 请求设置 `X-XSRF-TOKEN` 请求头。此外，要注意各前端框架文档中的细节差异——比如 Angular 只为不带 authority 部分的 URL 自动设置 `X-XSRF-TOKEN` 请求头（应请求 `/bff/api/me` 而非 `http://localhost:7080/bff/api/me`，即使 `window.location` 是 `http://localhost:7080/angular-ui/`）。

涉及 Spring OAuth2 客户端时，RP 发起的登出分两步完成：

1. 向 Spring OAuth2 客户端发送 POST 请求，关闭其 Session
2. 第一步响应的 `Location` 头包含授权服务器的 URI，用于关闭用户在授权服务器上的 Session

Spring 默认行为是对第一步响应使用 302 状态码，这会让浏览器自动跳转到授权服务器并保持相同 Origin，可能引发 CORS 错误。因此我们将 BFF 配置为返回 2xx 状态码，由 SPA 手动处理重定向，通过 `window.location.href` 以新 Origin 发起跳转。

另外，SPA 通过在登出请求中附加 `X-POST-LOGOUT-SUCCESS-URI` 请求头来提供登出后跳转 URI。spring-addons-starter-oidc 会将该头的值作为请求参数插入到向授权服务器发起的登出请求 URI 中。

### 6.5 客户端多租户

配套项目中只有一个使用授权码的 OAuth2 客户端注册。但如果有多个注册怎么办？这可能发生在多个前端共用一个 BFF，而不同前端使用不同 Issuer 或 Scope 的场景中。

用户应只能看到其有权限认证的 OpenID Provider 选项，在很多情况下可以对登录选项进行过滤，理想情况下只剩一个，无需用户选择。以下是几种可以大幅缩减选项数量的场景：

- SPA 配置了特定的登录选项
- 有多个反向代理，每个可以通过请求头等方式标明要使用的选项
- 前端设备的 IP 等技术信息可用于判断用户应在哪里认证

这种情况下有两种处理方式：在请求 `/login-options` 时携带过滤条件并在 BFF 控制器中过滤，或在前端中过滤 `/login-options` 的响应。

---

## 7. 后台通道登出（Back-Channel Logout）

在 SSO 场景下，如果用户通过另一个 OAuth2 客户端登出，而此时 BFF 上还有已开启的 Session，会发生什么？

OIDC 的后台通道登出（Back-Channel Logout）规范正是为此设计的：在授权服务器上声明客户端时，可以注册一个回调 URL，当用户登出时授权服务器会调用该 URL。

由于 BFF 运行在服务器上，它可以暴露端点来接收此类登出事件通知。自 Spring Security 6.2 起，框架已支持后台通道登出，且 spring-addons-starter-oidc 提供了一个开关来启用它。

一旦 BFF 通过后台通道登出结束 Session，前端向资源服务器的请求将不再被授权（甚至在令牌过期前就会失效）。因此，为了更好的用户体验，在 BFF 上启用后台通道登出时，建议同时添加 WebSocket 等机制来通知前端用户状态的变化。

---

## 8. 反向代理

SPA 与 BFF 需要同源，原因如下：

- 前端与 BFF 之间通过 Session Cookie 授权请求
- Spring 的 Session Cookie 标记了 `SameSite=Lax`

因此，我们使用反向代理作为浏览器的统一接入点。具体实现方案取决于环境：

- **Docker**：使用 Nginx 容器
- **Kubernetes**：配置 Ingress
- **IDE 本地开发**：可使用 Spring Cloud Gateway 实例（如果想减少运行服务数量，甚至可以在 BFF 的 Gateway 实例上增加额外路由，而非单独部署一个专用实例）

### 8.1 是否将授权服务器置于反向代理之后

出于安全考虑，授权服务器应始终设置 `X-Frame-Options` 响应头。Keycloak 允许将其设置为 `SAMEORIGIN`，这样如果授权服务器与 SPA 同源，就可以在 SPA 嵌入的 iframe 中展示 Keycloak 的登录和注册表单。

从用户体验角度来看，在同一个 Web 应用内以模态框展示授权表单，比在 SPA 和授权服务器之间来回重定向要好得多。

另一方面，单点登录（SSO）依赖标记了 `SameOrigin` 的 Cookie。因此，两个 SPA 要受益于 SSO，不仅需要在同一个授权服务器上认证用户，还需要使用相同的 Authority 来访问授权服务器（例如 `https://appa.net` 和 `https://appy.net` 都在 `https://sso.net` 上认证用户）。

同时满足这两个条件的方案是，将所有 SPA 和授权服务器都放在同一个 Origin 下，使用如下路径结构：

```
https://domain.net/appa
https://domain.net/appy
https://domain.net/auth
```

这是我们配合 Keycloak 使用的方案。不过，SPA 与授权服务器同源并非 BFF 模式运行的必要条件——必要条件只是 **SPA 与 BFF 同源**。

配套代码仓库中的项目已预配置为使用 Amazon Cognito 和 Auth0 各自的 Origin（无智能代理实时改写重定向 URL）。因此，iframe 登录体验仅在使用默认 Profile（Keycloak）时可用。

### 8.2 使用 Spring Cloud Gateway 实现反向代理

使用 IDE 或 [https://start.spring.io/](https://start.spring.io/) 创建一个名为 `reverse-proxy` 的新 Spring Boot 项目，依赖选择 **Reactive Gateway**，将配置文件重命名为 `application.yml`，然后定义路由属性：

```yaml
# 自定义属性，便于在命令行或 IDE 启动配置中覆盖
scheme: http
hostname: localhost
reverse-proxy-port: 7080
angular-port: 4201
angular-prefix: /angular-ui
angular-uri: http://${hostname}:${angular-port}${angular-prefix}
vue-port: 4202
vue-prefix: /vue-ui
vue-uri: http://${hostname}:${vue-port}${vue-prefix}
react-port: 4203
react-prefix: /react-ui
react-uri: http://${hostname}:${react-port}${react-prefix}
authorization-server-port: 8080
authorization-server-prefix: /auth
authorization-server-uri: ${scheme}://${hostname}:${authorization-server-port}${authorization-server-prefix}
bff-port: 7081
bff-prefix: /bff
bff-uri: ${scheme}://${hostname}:${bff-port}

server:
  port: ${reverse-proxy-port}

spring:
  cloud:
    gateway:
      default-filters:
      - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
      routes:
      # SPA 静态资源
      - id: angular-ui
        uri: ${angular-uri}
        predicates:
        - Path=${angular-prefix}/**
      - id: vue-ui
        uri: ${vue-uri}
        predicates:
        - Path=${vue-prefix}/**
      - id: react-ui
        uri: ${react-uri}
        predicates:
        - Path=${react-prefix}/**
      
      # 授权服务器
      - id: authorization-server
        uri: ${authorization-server-uri}
        predicates:
        - Path=${authorization-server-prefix}/**
      
      # BFF
      - id: bff
        uri: ${bff-uri}
        predicates:
        - Path=${bff-prefix}/**
        filters:
        - StripPrefix=1
```

现在可以停止对应 Docker 容器，在命令行参数或运行配置中提供 `hostname` 后启动反向代理。

---

## 9. 授权服务器

配套项目的默认 Profile 使用 Keycloak，但借助 spring-addons-starter-oidc，切换到任何其他 OpenID Provider 只需修改 `application.yml` 即可。配套项目提供了 Auth0 和 Amazon Cognito 的 Profile 配置，方便快速上手。

无论选择哪个 OpenID Provider，都需要：

- 声明一个机密（confidential）客户端
- 确定用作用户角色来源的私有 Claim
- 更新 BFF 和资源服务器的属性配置

---

## 10. 为什么使用 spring-addons-starter-oidc？

在本文中，我们对 `spring-boot-starter-oauth2-client` 和 `spring-boot-starter-oauth2-resource-server` 的多项默认行为进行了调整：

- 将 OAuth2 重定向 URI 改为指向反向代理，而非 OAuth2 客户端内部地址
- 让 SPA 控制登录/登出后的重定向目标
- 在 JavaScript 可访问的 Cookie 中暴露 CSRF 令牌
- 适配非完全标准的 RP 发起的登出（如 Auth0 和 Amazon Cognito）
- 向授权请求添加可选参数（如 Auth0 的 audience）
- 修改 OAuth2 重定向的 HTTP 状态码，使 SPA 能自行决定如何跟随 `Location` 头
- 注册两个 `SecurityFilterChain` Bean，分别使用 `oauth2Login()`（基于 Session 和 CSRF 保护）和 `oauth2ResourceServer()`（无状态，基于令牌）来保护不同资源组
- 定义哪些端点可匿名访问
- 在资源服务器上接受来自多个 OpenID Provider 的令牌
- 为 JWT 解码器添加受众（audience）校验
- 从任意 Claim 映射权限（可添加前缀或强制大小写转换）

这些修改通常需要大量 Java 代码以及对 Spring Security 的深入了解。但在本文中，仅通过 `application.yml` 属性配置便实现了所有这些功能，且可借助 IDE 自动补全来引导配置过程。

如需了解完整功能列表、自动配置调整选项和默认值覆盖方式，请参阅 [starter 的 GitHub README](https://github.com/ch4mpy/spring-addons)。

---

## 11. 总结

本教程演示了如何使用 Spring Cloud Gateway 与 spring-addons 实现 OAuth2 Backend for Frontend 模式。

同时我们也了解到：

- 为什么应优先选择 BFF 方案，而非将 SPA 配置为"公开" OAuth2 客户端
- 引入 BFF 对 SPA 本身的影响极小
- 该模式对资源服务器完全没有任何改动
- 由于使用的是服务端 OAuth2 客户端，借助后台通道登出可在 SSO 场景下完全掌控用户 Session

最后，我们初步探索了 spring-addons-starter-oidc 如何通过纯属性配置实现通常需要大量 Java 代码才能完成的功能。

完整代码可在 [GitHub](https://github.com/eugenp/tutorials) 上获取。

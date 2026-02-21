# Servlet OAuth2 BFF
使用 WebMvc（即 servlet/同步）版本的 Spring Cloud Gateway 实现 OAuth2 BFF 的示例。

Javascript UI 使用 JQuery 编写，并嵌入在 Thymeleaf 模板中。

由于登出（通过 POST 发送）和 PUT 请求均通过 ajax（XHR）发起，CSRF 保护配置为使用 `HttpOnly=false` 的 cookie。

`webmvc-jwt-default` 模块可用作*"下游微服务"*。
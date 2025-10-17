---
tags:
  - 八股
  - SSM
  - SpringMVC
status: 待复习
created: 2026-05-02
---

# SpringMVC 请求处理流程

> [!summary] 我的速记
> **一切请求从 DispatcherServlet 开始：收到请求 → HandlerMapping 找处理器 → HandlerAdapter 执行 → 返回 ModelAndView → ViewResolver 解析视图 → 响应。**
> **Spring Boot 后基本不用视图，@RestController = @Controller + @ResponseBody → 方法返回值直接写 JSON。**

---

## 一、DispatcherServlet 完整流程（面试必画）

```
HTTP请求
  │
  ├── ① DispatcherServlet 接收请求
  │
  ├── ② HandlerMapping 查找处理器
  │     根据 URL 找到对应的 @Controller / @RequestMapping 方法
  │     → 返回 HandlerExecutionChain（Handler + 拦截器链）
  │
  ├── ③ HandlerAdapter 执行处理器
  │     适配不同的 Handler（Controller 方法、HttpRequestHandler 等）
  │     → 调用目标方法，参数绑定（@RequestParam/@PathVariable/@RequestBody）
  │     → 返回 ModelAndView（或 @ResponseBody 时直接写 JSON）
  │
  ├── ④ ViewResolver 解析视图
  │     根据视图名找到具体视图（JSP/Thymeleaf/FreeMarker）
  │     → @RestController 时跳过这一步
  │
  └── ⑤ 响应返回给客户端
```

> 🧠 **为什么要搞 HandlerMapping + HandlerAdapter 两层？** HandlerMapping 负责"找到谁"，HandlerAdapter 负责"怎么调"——适配不同的处理器类型。这是适配器模式 + 策略模式的经典应用。

---

## 二、核心组件

| 组件 | 作用 | 类比 |
|------|------|------|
| **DispatcherServlet** | 前端总控制器，统一接收所有请求 | 前台 |
| **HandlerMapping** | 根据 URL 找对应的 Controller 方法 | 导航——你找哪个部门 |
| **HandlerAdapter** | 调用 Handler，绑定参数 | 接线员——帮你接通具体的人 |
| **ViewResolver** | 逻辑视图名 → 物理视图文件 | 门牌号转实际地址 |
| **HandlerInterceptor** | 请求前/后/完成后拦截 | 安检——进门查、出门查 |

---

## 三、@RestController vs @Controller

```java
@Controller
public class PageController {
    @RequestMapping("/hello")
    public String hello() {
        return "hello";  // 返回视图名 → ViewResolver 解析 → 渲染 HTML
    }
}

@RestController  // = @Controller + @ResponseBody
public class ApiController {
    @GetMapping("/api/user")
    public User getUser() {
        return new User();  // 返回对象 → Jackson 序列化 → JSON
    }
}
```

---

## 四、拦截器 vs 过滤器（面试高频对比）

| | 过滤器（Filter） | 拦截器（Interceptor） |
|---|---|---|
| 规范 | **Servlet 规范**（Java EE） | **Spring 框架** |
| 作用范围 | 所有请求（包括静态资源） | 只拦截进入 SpringMVC 的请求 |
| 能力 | 只能拿到 request/response | 可以拿到 Handler（哪个方法）、ModelAndView |
| 执行顺序 | 过滤器 → Servlet → 拦截器 | 在外层之后 |
| 适用场景 | 编码设置、XSS 防护、跨域 | 登录校验、权限检查、日志记录 |

```java
// 过滤器
public class MyFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        // 前置
        chain.doFilter(req, res);  // 放行
        // 后置
    }
}

// 拦截器
public class MyInterceptor implements HandlerInterceptor {
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        return true;  // true=放行  false=拦截
    }
    public void postHandle(req, res, handler, modelAndView) { }  // Controller 后、视图前
    public void afterCompletion(req, res, handler, ex) { }        // 视图渲染后
}
```

---

## 5. SpringMVC 参数绑定

| 注解 | 作用 | 示例 |
|------|------|------|
| `@RequestParam` | 绑定 URL 查询参数 / 表单 | `?name=xx` |
| `@PathVariable` | 绑定 URL 路径变量 | `/user/{id}` |
| `@RequestBody` | 绑定 JSON 请求体 → Java 对象 | `{ "name": "xx" }` |
| `@RequestHeader` | 绑定请求头 | `Authorization: Bearer xxx` |

---

## 常见面试问题清单

| #   | 问题                                         | 回答要点                                                                                           |
| --- | ------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| Q1  | **SpringMVC 请求处理流程？**                      | DispatcherServlet → HandlerMapping(找) → HandlerAdapter(调) → Controller → ViewResolver(视图) → 响应 |
| Q2  | **@RestController vs @Controller？**        | @RestController = @Controller + @ResponseBody（直接返回 JSON，不走视图）                                  |
| Q3  | **拦截器 vs 过滤器？**                            | 过滤器(Servlet规范,所有请求,只能拿req/res)；拦截器(Spring,只MVC请求,能拿Handler)                                    |
| Q4  | **HandlerMapping 和 HandlerAdapter 为什么分开？** | 适配器模式——Mapping 负责找，Adapter 负责调。支持不同类型的 Handler                                                 |
| Q5  | **参数绑定有哪些注解？**                             | @RequestParam(查询参数)、@PathVariable(路径)、@RequestBody(JSON)、@RequestHeader(请求头)                   |

---

## 面试满分话术

- **核心定位说明**：Spring MVC 是 Spring 对传统 MVC 模式的实现与扩展，传统 MVC 分模型、视图、控制器三层，Spring 体系中进一步细化为业务逻辑在 service 层、数据访问在 repository 层、Web 层由 Controller 层控制，该分层是 Spring 全家桶的事实架构约定。
- **Web 层运行机制**：Spring MVC 采用**前端控制器模式**，
- 核心角色 **DispatcherServlet** 统一接收所有请求，按请求**映射规则**分发至 Controller 处理；
- Controller 执行后返回 **ModelAndView** 对象，
- 由 **ViewResolver** 解析为 JSP、Thymeleaf 等页面，最终渲染成 HTML 返回浏览器。
- **前后端分离适配**：若 Controller 标注 @ResponseBody 或使用 @RestController，Spring MVC 会通过 **HttpMessageConverter** 将对象转换成 Json、Xml 等格式返回前端。
- **底层与优势**：Spring MVC 是对 servlet API 的高级封装，底层运行在 servlet 容器之上，通过 DispatcherServlet 将请求分发、参数绑定等流程标准化、自动化，简化 Web 开发。
> **面试官：SpringMVC 一个请求怎么处理的？**

> 一切从 DispatcherServlet 出发。收到请求后，先通过 HandlerMapping 找到对应的 Controller 方法（返回 HandlerExecutionChain 包含拦截器链）。然后 HandlerAdapter 执行处理器——完成参数绑定（@RequestParam/@PathVariable/@RequestBody）、调用目标方法、返回 ModelAndView。如果有视图名，ViewResolver 解析出实际视图渲染；如果是 @RestController 则直接序列化 JSON 写回响应。
>
> 拦截器和过滤器对比是常考点：过滤器是 Servlet 规范，所有请求都过；拦截器是 Spring 的，只拦截 MVC 请求，且能拿到 Handler 信息和 ModelAndView。

---

## 记忆口诀

```
前端总控 Dispatcher，请求统一它来收。
HandlerMapping 找处理器，Adapter 执行绑参数。
视图解析 ViewResolver，JSON 返回 REST 做。
过滤器在外拦截里，Servlet 标准 Spring 框架。
```

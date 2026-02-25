# TOPIC 1.1 — WHAT IS SPRING MVC: PHILOSOPHY & DESIGN GOALS

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What is Spring MVC?

Spring MVC is a **web framework built on the Servlet API**, part of the broader Spring Framework. It implements the **Front Controller pattern** where a single servlet — `DispatcherServlet` — receives all HTTP requests and delegates them to appropriate handlers.

It is NOT a standalone framework. It is deeply integrated with the **Spring IoC container**, meaning every component — controllers, resolvers, converters — is a **Spring-managed bean**.

---

### The Core Philosophy

Spring MVC was built around **4 foundational philosophies**:

**1. Separation of Concerns**
The framework enforces a strict boundary between request handling (Controller), business logic (Service), data (Model), and rendering (View). No layer bleeds into another by design.

**2. Convention over Configuration with full Override Control**
Spring MVC provides intelligent defaults for everything — default HandlerMappings, default ViewResolvers, default MessageConverters. But every single default is **replaceable**. You are never locked in.

**3. POJO-Based Programming Model**
Controllers are plain Java classes. No framework interface is required. A method becomes a handler purely through annotation metadata (`@RequestMapping`). The framework discovers and invokes it via **reflection at runtime**.

**4. Testability as a First-Class Concern**
Because controllers are POJOs with no servlet dependencies injected directly, they can be unit-tested without a running container. MockMvc further enables full integration testing without deploying to Tomcat.

---

### Design Goals — What Problems Spring MVC Solved

Before Spring MVC, the dominant model was **Struts 1.x**, which required:
- Controllers to extend `Action` class (tight coupling)
- XML configuration for every mapping
- ActionForm objects for every form (redundant, verbose)
- No clean separation of validation, binding, rendering

Spring MVC solved this by:

| Problem in Struts Era | Spring MVC Solution |
|---|---|
| Extend framework classes | Pure POJOs + annotations |
| XML-only mapping | Annotation-driven mapping |
| No flexible view support | ViewResolver chain |
| No content negotiation | ContentNegotiatingViewResolver + MessageConverters |
| No clean exception handling | @ExceptionHandler + @ControllerAdvice |
| Hard to test | MockMvc + POJO controllers |

---

### Where Spring MVC Lives in the Ecosystem

```
Java EE Servlet Container (Tomcat / Jetty / Undertow)
        │
        ▼
  Servlet API (javax.servlet / jakarta.servlet)
        │
        ▼
  Spring MVC (spring-webmvc module)
        │
        ├── DispatcherServlet (Front Controller)
        ├── Spring IoC Container (ApplicationContext)
        ├── AOP Infrastructure (Proxies, Interceptors)
        └── Your Application Code (Controllers, Services)
```

Spring MVC sits **between the Servlet container and your application code**. It translates raw `HttpServletRequest` objects into rich method arguments and converts your return values back into HTTP responses.

---

### spring-webmvc Module — What's Inside

The artifact `spring-webmvc` contains:
- `DispatcherServlet`
- All `HandlerMapping` implementations
- All `HandlerAdapter` implementations
- All `ViewResolver` implementations
- All `HttpMessageConverter` implementations
- `@Controller`, `@RequestMapping`, and all MVC annotations
- MockMvc test infrastructure

It depends on `spring-web` which contains lower-level HTTP abstractions like `HttpEntity`, `MediaType`, `HttpHeaders`.

---

### Runtime Architecture — The Big Picture

At runtime, Spring MVC operates as follows:

```
HTTP Request
     │
     ▼
Servlet Container (Tomcat)
     │  matches URL to DispatcherServlet via web.xml or ServletRegistration
     ▼
DispatcherServlet.service()
     │
     ├── consults HandlerMapping → finds Handler + Interceptors
     ├── consults HandlerAdapter → invokes Handler Method
     │       │
     │       ├── resolves method arguments (ArgumentResolvers)
     │       ├── executes your @RequestMapping method
     │       └── processes return value (ReturnValueHandlers)
     │
     ├── ViewResolver (if view name returned)
     └── HttpMessageConverter (if @ResponseBody used)
          │
          ▼
     HTTP Response written to client
```

Every step in this chain involves **Spring-managed beans**. This is the critical architectural insight — DispatcherServlet does not hardcode any behavior. It **delegates everything to beans it discovers in the WebApplicationContext**.

---

### Internal Architecture — Reflection & Proxy Usage

When your application starts, Spring MVC performs the following at **initialization time** (not per-request):

1. Scans all `@Controller` beans
2. Reflects over all methods annotated with `@RequestMapping`
3. Builds a **mapping registry** — a map of `RequestMappingInfo → HandlerMethod`
4. `HandlerMethod` wraps the controller bean reference + `java.lang.reflect.Method`

At **request time**:
1. Looks up `HandlerMethod` from registry
2. Calls `method.invoke(controllerInstance, resolvedArgs)` via reflection
3. If the controller is proxied (AOP), invokes through the **CGLIB proxy**

This means:
- Mapping resolution cost = **O(1) lookup** (pre-built at startup)
- Method invocation cost = **reflection invoke** (small but non-zero overhead)
- If `@Transactional` or AOP advice exists on controller — **CGLIB proxy** wraps it

---

### Thread-Per-Request Model

Spring MVC inherits the **Servlet thread model**:

- Tomcat maintains a thread pool (default 200 threads)
- Each HTTP request gets **one thread** for its entire lifecycle
- That thread executes: HandlerMapping → HandlerAdapter → View rendering → Response write
- Thread is returned to pool only after response is fully committed

Implication: **Blocking operations block the thread**. Database calls, external API calls, file I/O — all hold the thread. This is why async support (`DeferredResult`, `Callable`) was introduced — to release the container thread while waiting.

`RequestContextHolder` uses a `ThreadLocal<RequestAttributes>` to make the current `HttpServletRequest` available anywhere in the call stack without passing it explicitly. This is powerful but creates a trap — if you spawn a new thread manually, it will NOT have the request context.

---

### Memory & Performance Implications

- The **mapping registry** is built at startup and held in memory for the JVM lifetime. Large numbers of `@RequestMapping` methods increase startup time marginally but have zero per-request cost.
- **`ModelAndView` objects** are created per request and GC'd after response. Keep them lightweight.
- **HttpMessageConverter** serialization (Jackson) is the most CPU-intensive part of a typical REST request. Jackson's `ObjectMapper` should always be a **singleton** — Spring ensures this.
- **ViewResolver chain** is traversed on every view-returning request. Order matters for performance.

---

### Enterprise Pitfalls at This Level

**Pitfall 1 — Treating Spring MVC as Spring Boot**
Spring MVC requires manual servlet registration, context configuration, and dependency wiring. Spring Boot auto-configures all of this. Confusing the two leads to missing configurations in plain Spring MVC setups.

**Pitfall 2 — Controller as a God Object**
Spring MVC makes it easy to put everything in the controller. Resist. Controllers must only handle HTTP concerns — argument extraction, response building. Business logic belongs in the service layer.

**Pitfall 3 — Thread-Safety of Controllers**
Controllers are **singleton beans**. Instance variables on controllers are shared across all requests and all threads. This is a critical concurrency bug source.

**Pitfall 4 — Assuming DispatcherServlet handles everything**
Requests to static resources, error pages handled by container, and requests to other servlets bypass DispatcherServlet entirely. Many developers assume Spring MVC handles all traffic.

---

### Common Misconceptions

| Misconception | Reality |
|---|---|
| Spring MVC = Spring Boot | Spring Boot auto-configures Spring MVC. They are separate. |
| DispatcherServlet IS the application | DispatcherServlet is one servlet. It delegates to beans. |
| @Controller is required | Only @Component + @RequestMapping method is technically needed |
| Spring MVC is stateless | The framework is. Your session-scoped beans may not be. |
| All requests go through DispatcherServlet | Only requests matching its URL mapping do |

---

## 2️⃣ CODE EXAMPLES

### XML Configuration (Traditional)

**web.xml**
```xml
<web-app>

  <!-- Root ApplicationContext — services, repositories -->
  <listener>
    <listener-class>
      org.springframework.web.context.ContextLoaderListener
    </listener-class>
  </listener>

  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/root-context.xml</param-value>
  </context-param>

  <!-- DispatcherServlet — Web ApplicationContext -->
  <servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>
      org.springframework.web.servlet.DispatcherServlet
    </servlet-class>
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>/WEB-INF/dispatcher-servlet.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
  </servlet>

  <servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>

</web-app>
```

**dispatcher-servlet.xml**
```xml
<beans>
  <!-- Enables @RequestMapping, @Controller scanning -->
  <mvc:annotation-driven/>

  <context:component-scan base-package="com.example.web"/>

  <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="prefix" value="/WEB-INF/views/"/>
    <property name="suffix" value=".jsp"/>
  </bean>
</beans>
```

---

### Java Config (Modern — No XML)

```java
// Replaces web.xml — registered by Servlet container via SpringServletContainerInitializer
public class AppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        // Root context: Services, Repositories, Security
        return new Class[]{RootConfig.class};
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        // Web context: Controllers, ViewResolvers, MVC config
        return new Class[]{WebConfig.class};
    }

    @Override
    protected String[] getServletMappings() {
        return new String[]{"/"};  // DispatcherServlet handles ALL requests
    }
}
```

```java
@Configuration
@EnableWebMvc  // Registers all default MVC infrastructure beans
@ComponentScan("com.example.web")
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public ViewResolver viewResolver() {
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");
        return resolver;
    }
}
```

```java
// Simplest possible controller — POJO, no framework dependency
@Controller
public class HomeController {

    @GetMapping("/")
    public String home(Model model) {
        model.addAttribute("message", "Spring MVC is running");
        return "home";  // Resolved to /WEB-INF/views/home.jsp
    }
}
```

---

### Edge Case — Controller Without @Controller

```java
// This WORKS if registered as a bean manually — @Controller is just @Component + stereotype
@Component
public class MinimalController {

    @RequestMapping("/minimal")
    @ResponseBody
    public String handle() {
        return "Works without @Controller stereotype";
    }
}
```

**What happens internally:** `RequestMappingHandlerMapping` scans beans for `@RequestMapping` methods. It checks for `@Controller` OR `@RequestMapping` at class level. A `@Component` bean with `@RequestMapping` methods is detected. This is rarely used but valid.

---

### Edge Case — Instance Variable Concurrency Trap

```java
@Controller
public class UnsafeController {

    // DANGEROUS — shared across all threads
    private int requestCount = 0;

    @GetMapping("/count")
    @ResponseBody
    public String count() {
        requestCount++;  // Race condition — not thread-safe
        return "Count: " + requestCount;
    }
}
```

```java
@Controller
public class SafeController {

    // Thread-safe — AtomicInteger or use stateless design
    private final AtomicInteger requestCount = new AtomicInteger(0);

    @GetMapping("/count")
    @ResponseBody
    public String count() {
        return "Count: " + requestCount.incrementAndGet();
    }
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
Which component is responsible for receiving ALL incoming HTTP requests in Spring MVC?

A) ContextLoaderListener
B) DispatcherServlet
C) HandlerAdapter
D) InternalResourceViewResolver

**Answer: B**
ContextLoaderListener only bootstraps the root context. HandlerAdapter executes handlers. ViewResolver renders views. Only DispatcherServlet is the Front Controller that receives requests.

---

**Q2 — Select All That Apply**
Which statements about Spring MVC controllers are TRUE?

A) Controllers must extend a Spring framework class
B) Controllers are singleton beans by default
C) Controllers are thread-safe by default
D) Instance variables on controllers are shared across requests
E) @Controller is just a specialization of @Component

**Answer: B, D, E**
A is false — POJOs work. C is false — singleton does not mean thread-safe. D is the dangerous consequence of B. E is true — @Controller adds stereotype semantics.

---

**Q3 — Scenario**
A developer places a `List<String> requestData` as an instance variable on a `@Controller` class and adds items to it in a `@PostMapping` method. The app is deployed to a server handling 100 concurrent users. What will happen?

A) Each request gets its own list — no issue
B) All requests share the same list — race conditions and data corruption
C) Spring creates a new controller instance per request
D) Spring throws a BeanCreationException at startup

**Answer: B**
Controllers are singletons. The list is shared. Multiple threads calling `add()` simultaneously produces race conditions.

---

**Q4 — True/False**
`AbstractAnnotationConfigDispatcherServletInitializer` completely replaces `web.xml` in a Servlet 3.0+ environment.

**Answer: True**
Servlet 3.0 introduced `ServletContainerInitializer`. Spring's `SpringServletContainerInitializer` detects `WebApplicationInitializer` implementations (which `AbstractAnnotationConfigDispatcherServletInitializer` implements) and registers the DispatcherServlet programmatically.

---

**Q5 — Code Prediction**
```java
@Controller
public class TestController {
    @GetMapping("/test")
    public String handle() {
        return "result";
    }
}
```
No `ViewResolver` is configured. What happens when `/test` is requested?

A) Returns string "result" as response body
B) Throws NoSuchBeanDefinitionException at startup
C) Returns 500 — no ViewResolver can resolve "result"
D) Spring uses a default ViewResolver automatically

**Answer: D — partially.**
Spring MVC registers a default `InternalResourceViewResolver` with no prefix/suffix if none is configured. It will attempt to forward to `/result` which likely results in a 404 — but no exception at startup. The trap here is assuming it's a 500 — it's actually a 404 from the servlet container.

---

**Q6 — Ordering (Drag and Drop)**
Order these events in Spring MVC startup sequence:

- Servlet container starts
- DispatcherServlet.init() called
- @RequestMapping methods scanned and registry built
- ContextLoaderListener creates root ApplicationContext
- WebApplicationContext created by DispatcherServlet

**Correct Order:**
1. Servlet container starts
2. ContextLoaderListener creates root ApplicationContext
3. DispatcherServlet.init() called
4. WebApplicationContext created by DispatcherServlet
5. @RequestMapping methods scanned and registry built

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — "Spring MVC is thread-safe"**
The *framework infrastructure* is thread-safe. Your *controller code* may not be. Examiners exploit this distinction constantly. The question "Are Spring MVC controllers thread-safe?" has the answer NO — because of singleton scope + mutable instance variables.

**Trap 2 — @Controller vs @Component**
Examiners ask: "What is the minimum requirement to make a class a handler?" Many say @Controller. The real answer is: the class must be a Spring bean AND have @RequestMapping methods. @Component satisfies the first requirement. @Controller adds no technical behavior beyond @Component for RequestMappingHandlerMapping detection purposes.

**Trap 3 — DispatcherServlet handles ALL requests**
Only requests matching the servlet's URL pattern are handled. If mapped to `/api/*`, requests to `/static/image.png` bypass it. Many questions show a `404` scenario and ask why — answer is often that the request didn't match DispatcherServlet's mapping.

**Trap 4 — ContextLoaderListener is required**
It is NOT required. You can put everything in the DispatcherServlet's WebApplicationContext. ContextLoaderListener is a best practice for separation, not a hard requirement.

---

## 5️⃣ SUMMARY SHEET

```
SPRING MVC PHILOSOPHY
─────────────────────────────────────────────────────
Pattern         : Front Controller (DispatcherServlet)
Programming     : POJO-based, annotation-driven
Container       : Servlet API (Tomcat/Jetty/Undertow)
Thread Model    : Thread-per-request (blocking by default)
Context         : Two-level ApplicationContext hierarchy
Config Styles   : XML / Annotation / Java Config / Boot Auto-config

KEY ARCHITECTURAL FACTS
─────────────────────────────────────────────────────
• DispatcherServlet delegates to beans — nothing is hardcoded
• Mapping registry built ONCE at startup via reflection
• Per-request: reflection invoke + argument resolution
• Controllers: SINGLETON scope → NOT inherently thread-safe
• ThreadLocal (RequestContextHolder) stores request per thread
• @Controller = @Component + MVC stereotype (no extra magic)

STARTUP SEQUENCE
─────────────────────────────────────────────────────
Container Start
    → ContextLoaderListener → Root ApplicationContext
    → DispatcherServlet.init() → WebApplicationContext
    → HandlerMapping scans @RequestMapping → builds registry
    → Special beans initialized (resolvers, adapters, converters)

REQUEST LIFECYCLE (HIGH LEVEL)
─────────────────────────────────────────────────────
Request → DispatcherServlet
    → HandlerMapping (find handler)
    → HandlerAdapter (invoke handler)
        → ArgumentResolvers (bind method params)
        → Your method executes
        → ReturnValueHandlers (process return)
    → ViewResolver OR MessageConverter
    → Response written

COMMON INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "DispatcherServlet is the Front Controller of Spring MVC"
• "Controllers are singletons — never use mutable instance variables"
• "Mapping registry is built at startup — zero per-request lookup cost"
• "Spring MVC sits on top of Servlet API — not a replacement"
• "@Controller adds no technical behavior beyond @Component for mapping"
• "ContextLoaderListener creates parent context — optional but best practice"
```

---

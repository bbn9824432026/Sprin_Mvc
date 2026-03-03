# TOPIC 1.5 — SPRING MVC vs SPRING BOOT vs SPRING WEBFLUX: POSITIONING

---

## 1️⃣ CONCEPTUAL EXPLANATION

### Why This Topic Matters

This is one of the **most misunderstood areas** in Spring interviews and certifications. Candidates conflate these three things constantly. The confusion costs marks because questions test precise understanding of what each layer provides, where responsibilities lie, and what trade-offs exist.

The three are **not alternatives to each other** in a simple sense. They operate at different layers of abstraction and serve different purposes.

---

### The Relationship — Layered View

```
┌─────────────────────────────────────────────────────────┐
│                    SPRING BOOT                          │
│  Auto-configuration, embedded server, starter POMs,    │
│  production-ready features, opinionated defaults        │
│                                                         │
│  ┌─────────────────────┐  ┌──────────────────────────┐ │
│  │    SPRING MVC       │  │    SPRING WEBFLUX        │ │
│  │  Servlet-based      │  │  Reactive, non-blocking  │ │
│  │  Blocking I/O       │  │  Project Reactor based   │ │
│  │  Thread-per-request │  │  Event loop model        │ │
│  └─────────────────────┘  └──────────────────────────┘ │
│                                                         │
│  Both run ON Spring Boot OR standalone                  │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                  SPRING FRAMEWORK                       │
│  Core IoC container, AOP, Data, MVC, WebFlux           │
│  The foundation everything builds on                    │
└─────────────────────────────────────────────────────────┘
```

**Spring Boot is a configuration tool. Spring MVC and WebFlux are web frameworks. Spring Framework is the foundation.**

---

### Spring MVC — Precise Definition

Spring MVC is a **Servlet-based synchronous web framework** built on the Java Servlet API.

Core characteristics:

**1. Blocking I/O Model**
Every request occupies one thread from start to finish. If the handler calls a database, that thread waits. This is the traditional Java EE model.

**2. Servlet Container Required**
Requires a Servlet container — Tomcat, Jetty, Undertow. The container manages thread pools, HTTP parsing, connection management.

**3. Synchronous by Default**
Controller methods execute synchronously. The return value is the response. There is no reactive stream — just a direct method call and return.

**4. Mature and Comprehensive**
Spring MVC has been production-tested since 2003. Every enterprise pattern is solved. Every edge case is handled. The ecosystem is vast.

**5. Module: spring-webmvc**
The JAR is `spring-webmvc`. It depends on `spring-web` for lower-level HTTP abstractions.

---

### Spring Boot — Precise Definition

Spring Boot is an **opinionated configuration and packaging framework** built ON TOP of Spring Framework.

It provides:

**1. Auto-Configuration**
Based on what's on the classpath, Spring Boot automatically configures beans. If `spring-webmvc` is on the classpath, it auto-configures `DispatcherServlet`, `HandlerMappings`, `ViewResolvers`, `MessageConverters`, etc. — things you'd configure manually in plain Spring MVC.

**2. Embedded Server**
Spring Boot embeds Tomcat (or Jetty/Undertow) directly in the JAR. No WAR deployment. The application starts with `main()`. The server is inside the app, not the app inside the server.

**3. Starter POMs**
`spring-boot-starter-web` pulls in `spring-webmvc` + Tomcat + Jackson + Validation. One dependency replaces 10+.

**4. Production-Ready Features**
Actuator, health checks, metrics, externalized configuration (`application.properties`), profiles.

**5. What Spring Boot Does NOT Do**
It does NOT change Spring MVC's behavior. `DispatcherServlet` works identically in Boot vs plain Spring MVC. `@RequestMapping`, `@ExceptionHandler`, `HandlerInterceptor` — all identical. Boot just removes the configuration boilerplate.

---

### Spring WebFlux — Precise Definition

Spring WebFlux is a **reactive, non-blocking web framework** introduced in Spring 5. It is a parallel web stack to Spring MVC — not an upgrade of it.

Core characteristics:

**1. Reactive Programming Model**
Based on **Project Reactor** (`Mono<T>`, `Flux<T>`). Handlers return publishers, not values. The framework subscribes to publishers and writes results asynchronously.

**2. Non-Blocking I/O**
Does NOT require a thread per request. Uses an **event loop** (Netty's model). A small number of threads handle thousands of concurrent connections by not blocking on I/O.

**3. Does NOT Require Servlet API**
Can run on Netty (non-servlet), Undertow, or Servlet 3.1+ containers. On Netty, there is no `HttpServletRequest` — only reactive `ServerRequest`.

**4. Two Programming Models**
- **Annotated Controllers** — same `@Controller`, `@RequestMapping` annotations, but return `Mono<String>` instead of `String`
- **Functional Endpoints** — `RouterFunction` + `HandlerFunction` — fully functional, no annotations

**5. Module: spring-webflux**
Separate JAR — `spring-webflux`. Cannot mix with `spring-webmvc` in the same application context.

---

### The Threading Model Comparison — Deep Dive

This is the most important technical distinction:

**Spring MVC — Thread Per Request**
```
Request 1 → Thread A → [DB query: 200ms wait] → Thread A → Response
Request 2 → Thread B → [DB query: 200ms wait] → Thread B → Response
Request 3 → Thread C → [DB query: 200ms wait] → Thread C → Response
...
Request 201 → QUEUED — no threads available (pool exhausted at 200)

Thread pool: 200 threads
1000 concurrent requests → 800 queued, latency spikes
```

**Spring WebFlux — Event Loop**
```
Request 1 → EventLoop Thread → [DB query initiated, non-blocking] → 
            Thread FREE to handle other requests
            [DB result arrives as event] → Thread → Response

Request 2 → Same EventLoop Thread → [external call, non-blocking] →
            Thread FREE again

EventLoop threads: 8 (= number of CPU cores)
1000 concurrent requests → all handled, minimal queuing
Condition: ALL I/O must be non-blocking (R2DBC, WebClient, Reactive Mongo)
```

**The critical nuance:** WebFlux's advantage evaporates if you use blocking I/O (JDBC, traditional REST calls). Mixing blocking code in a reactive pipeline stalls the event loop thread — worse than just using Spring MVC.

---

### When to Use Spring MVC vs WebFlux

| Factor | Choose Spring MVC | Choose Spring WebFlux |
|---|---|---|
| Team familiarity | Traditional Java team | Reactive programming experience |
| I/O characteristics | CPU-bound, moderate concurrency | High concurrency, I/O-bound, streaming |
| Database | JPA/JDBC (blocking) | R2DBC, MongoDB reactive, Cassandra reactive |
| External APIs | RestTemplate (blocking OK) | WebClient (non-blocking) |
| Existing codebase | Existing Spring MVC app | Greenfield reactive system |
| Debugging complexity | Straightforward stack traces | Complex — async stack traces |
| Library ecosystem | Mature, everything supported | Some libraries lack reactive support |
| Concurrency model | Simple — thread-per-request | Complex — backpressure, publisher chains |

**Enterprise reality:** 90% of Spring applications use Spring MVC. WebFlux is appropriate for specific high-concurrency, I/O-intensive scenarios. The Spring team itself states: "If you have a Spring MVC application that works fine, there is no need to change."

---

### Spring Boot + Spring MVC — What Auto-Configuration Does

When `spring-boot-starter-web` is on classpath, `DispatcherServletAutoConfiguration` runs:

```
SpringApplication.run()
        │
        ▼
Reads spring.factories / AutoConfiguration.imports
        │
        ▼
DispatcherServletAutoConfiguration
  → Registers DispatcherServlet bean
  → Maps it to "/"
  → Creates DispatcherServletRegistrationBean

WebMvcAutoConfiguration
  → Configures ContentNegotiatingViewResolver
  → Configures BeanNameViewResolver
  → Configures static resource handling (/static, /public, /resources, /META-INF/resources)
  → Configures HttpMessageConverters (Jackson if on classpath)
  → Configures Formatter/Converter beans
  → Configures LocaleResolver
  → Configures index.html welcome page support

EmbeddedWebServerFactoryCustomizerAutoConfiguration
  → Configures embedded Tomcat
  → Reads server.port, server.servlet.context-path, etc.
```

**Everything WebMvcAutoConfiguration does is what you'd manually write in `WebMvcConfigurer` and `web.xml` in plain Spring MVC.**

---

### @EnableWebMvc — The Critical Interaction With Boot

This is a **heavily tested trap**:

```java
// WRONG in Spring Boot — disables Boot's auto-configuration
@SpringBootApplication
@EnableWebMvc  // THIS DISABLES WebMvcAutoConfiguration
public class MyApp { ... }
```

`@EnableWebMvc` imports `DelegatingWebMvcConfiguration`. Spring Boot's `WebMvcAutoConfiguration` has:

```java
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
public class WebMvcAutoConfiguration { ... }
```

`DelegatingWebMvcConfiguration` extends `WebMvcConfigurationSupport`. So when `@EnableWebMvc` is present, `WebMvcAutoConfiguration` backs off — **all Spring Boot MVC auto-configuration is disabled**.

**Correct approach in Spring Boot:** implement `WebMvcConfigurer` WITHOUT `@EnableWebMvc`.

---

### The Annotated Controller — MVC vs WebFlux Comparison

```java
// Spring MVC Controller
@RestController
public class ProductController {
    
    @GetMapping("/products/{id}")
    public Product getProduct(@PathVariable Long id) {
        return productService.findById(id); // Blocking — holds thread
        // Returns Product directly
    }
    
    @GetMapping("/products")
    public List<Product> getAllProducts() {
        return productService.findAll(); // Blocking call
    }
}

// Spring WebFlux Annotated Controller (same annotations, different return types)
@RestController
public class ReactiveProductController {
    
    @GetMapping("/products/{id}")
    public Mono<Product> getProduct(@PathVariable Long id) {
        return reactiveProductService.findById(id); // Non-blocking
        // Returns Mono — publisher of 0 or 1 item
        // Thread released while DB query runs
    }
    
    @GetMapping("/products")
    public Flux<Product> getAllProducts() {
        return reactiveProductService.findAll(); // Non-blocking stream
        // Returns Flux — publisher of 0..N items
        // Can stream items as they arrive
    }
}
```

---

### Shared Infrastructure — What MVC and WebFlux Share

Both use the same:
- `@Controller`, `@RestController`, `@RequestMapping` family
- `@ExceptionHandler`, `@ControllerAdvice`
- `@RequestBody`, `@ResponseBody`, `@PathVariable`, `@RequestParam`
- `HttpMessageConverter` (MVC) ↔ `HttpMessageReader/Writer` (WebFlux — parallel abstraction)
- Spring's IoC container, AOP, validation, security integration

What they do NOT share:
- `HandlerMapping` implementations (different class hierarchy)
- `HandlerAdapter` implementations (different)
- `DispatcherServlet` (MVC) vs `DispatcherHandler` (WebFlux)
- Filter (MVC — Servlet) vs `WebFilter` (WebFlux — reactive)
- `HttpServletRequest` (MVC) vs `ServerRequest` (WebFlux functional)

---

### Spring Boot + WebFlux — Auto-Configuration

When `spring-boot-starter-webflux` is on classpath:
- `ReactiveWebServerFactoryAutoConfiguration` → configures Netty by default
- `WebFluxAutoConfiguration` → configures `DispatcherHandler`, reactive codecs, etc.
- No `DispatcherServlet` — replaced by `DispatcherHandler`

**You cannot have both `spring-boot-starter-web` (MVC) and `spring-boot-starter-webflux` on the classpath simultaneously.** If both are present, Spring Boot defaults to MVC. WebFlux loses.

---

### Performance Characteristics — Precise Numbers

**Spring MVC (Tomcat default)**
- Thread pool: 200 threads (configurable via `server.tomcat.threads.max`)
- Max concurrent blocking requests: ~200
- Memory per thread: ~512KB stack → 200 threads ≈ 100MB just for stacks
- Best for: < 1000 concurrent users, any I/O pattern

**Spring WebFlux (Netty default)**
- Event loop threads: `Runtime.getRuntime().availableProcessors() * 2` (typically 8-16)
- Max concurrent requests: theoretically unlimited (bounded by memory, not threads)
- Memory: much lower per connection
- Best for: > 10,000 concurrent connections, streaming, SSE, WebSocket

**Reality check:** A well-tuned Spring MVC app handles 50,000 req/sec with fast handlers. WebFlux advantage appears only when I/O wait is the bottleneck.

---

### Functional Endpoints — WebFlux Only

WebFlux introduces a completely different programming model with no annotations:

```java
// Router — defines URL → handler mappings
@Configuration
public class ProductRouter {
    
    @Bean
    public RouterFunction<ServerResponse> routes(ProductHandler handler) {
        return RouterFunctions.route()
            .GET("/products/{id}", handler::getProduct)
            .GET("/products", handler::getAllProducts)
            .POST("/products", handler::createProduct)
            .build();
    }
}

// Handler — pure functions
@Component
public class ProductHandler {
    
    public Mono<ServerResponse> getProduct(ServerRequest request) {
        Long id = Long.valueOf(request.pathVariable("id"));
        return productService.findById(id)
            .flatMap(product -> ServerResponse.ok()
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue(product))
            .switchIfEmpty(ServerResponse.notFound().build());
    }
}
```

This has no equivalent in Spring MVC. Spring MVC has no `RouterFunction` support in the traditional sense.

---

### Summary Comparison Table

| Dimension | Spring MVC | Spring WebFlux | Spring Boot |
|---|---|---|---|
| Nature | Web framework | Web framework | Configuration tool |
| I/O Model | Blocking (Servlet) | Non-blocking (Reactive) |Wraps either |
| Thread Model | Thread-per-request | Event loop | N/A |
| API Foundation | Servlet API | Project Reactor | Spring Framework |
| DispatcherServlet | Yes | No (DispatcherHandler) | Auto-configures either |
| Default Server | Tomcat | Netty | Tomcat (MVC) / Netty (WebFlux) |
| Return Types | POJO, ResponseEntity | Mono, Flux, POJO | N/A |
| Filter Type | javax/jakarta Filter | WebFilter | N/A |
| Introduced | Spring 2.5 (2007) | Spring 5 (2017) | 2014 |
| Maturity | Very High | High | Very High |

---

## 2️⃣ CODE EXAMPLES

### Plain Spring MVC — Zero Boot

```java
// No Spring Boot — manual everything
public class AppInitializer 
    extends AbstractAnnotationConfigDispatcherServletInitializer {

    protected Class<?>[] getRootConfigClasses() {
        return new Class[]{ServiceConfig.class};
    }
    protected Class<?>[] getServletConfigClasses() {
        return new Class[]{WebConfig.class};
    }
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }
}

@Configuration
@EnableWebMvc
@ComponentScan("com.example")
public class WebConfig implements WebMvcConfigurer {
    // MANUAL: Must configure everything
    @Bean
    public ViewResolver viewResolver() {
        InternalResourceViewResolver r = new InternalResourceViewResolver();
        r.setPrefix("/WEB-INF/views/");
        r.setSuffix(".jsp");
        return r;
    }
    // Must manually configure Jackson, static resources, etc.
}
```

---

### Spring Boot + Spring MVC — Auto-Configured

```java
// Spring Boot — Boot handles all of the above automatically
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
        // DispatcherServlet: auto-configured
        // Tomcat: embedded and started
        // Jackson: auto-configured
        // Static resources: auto-configured
        // ViewResolvers: auto-configured
    }
}

// Just write controllers — everything else is handled
@RestController
public class ProductController {
    @GetMapping("/products")
    public List<Product> getAll() {
        return List.of(new Product("Widget", 9.99));
    }
}
```

---

### Spring Boot — Customizing Auto-Configuration

```java
// Extend auto-configuration without disabling it
// DO NOT use @EnableWebMvc in Spring Boot
@Configuration
public class WebConfig implements WebMvcConfigurer {

    // Override specific defaults
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoggingInterceptor());
    }

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://example.com");
    }

    @Override
    public void configureMessageConverters(
            List<HttpMessageConverter<?>> converters) {
        // ADD converters — Boot's converters still active
        // Use extendMessageConverters() to modify Boot's defaults
    }
    // Boot's WebMvcAutoConfiguration still fully active
}
```

---

### WebFlux — Annotated Style With Reactor

```java
@SpringBootApplication
public class ReactiveApp {
    public static void main(String[] args) {
        SpringApplication.run(ReactiveApp.class, args);
        // Netty started instead of Tomcat
        // DispatcherHandler registered (not DispatcherServlet)
        // Reactive codecs registered
    }
}

// pom.xml: spring-boot-starter-webflux (NOT spring-boot-starter-web)

@RestController
@RequestMapping("/api")
public class ReactiveController {

    private final ReactiveProductRepository repository;

    // Mono — 0 or 1 result
    @GetMapping("/products/{id}")
    public Mono<ResponseEntity<Product>> getById(@PathVariable String id) {
        return repository.findById(id)
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }

    // Flux — 0..N results, can stream
    @GetMapping(value = "/products", 
                produces = MediaType.APPLICATION_NDJSON_VALUE)
    public Flux<Product> streamAll() {
        return repository.findAll()
            .delayElements(Duration.ofMillis(100)); // Simulate stream
    }

    // Mono<Void> — write-only operation
    @PostMapping("/products")
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<Product> create(@RequestBody Mono<Product> productMono) {
        return productMono.flatMap(repository::save);
    }
}
```

---

### WebFlux Functional Style

```java
@Configuration
public class ProductRouterConfig {

    @Bean
    public RouterFunction<ServerResponse> productRoutes(
            ProductHandler handler) {
        return RouterFunctions.route()
            .path("/api/products", builder -> builder
                .GET("/{id}", handler::findById)
                .GET("", handler::findAll)
                .POST("", handler::create)
                .PUT("/{id}", handler::update)
                .DELETE("/{id}", handler::delete)
            )
            .build();
    }
}

@Component
public class ProductHandler {

    private final ReactiveProductRepository repository;

    public Mono<ServerResponse> findById(ServerRequest request) {
        String id = request.pathVariable("id");
        return repository.findById(id)
            .flatMap(p -> ServerResponse.ok()
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue(p))
            .switchIfEmpty(ServerResponse.notFound().build());
    }

    public Mono<ServerResponse> findAll(ServerRequest request) {
        return ServerResponse.ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(repository.findAll(), Product.class);
    }

    public Mono<ServerResponse> create(ServerRequest request) {
        return request.bodyToMono(Product.class)
            .flatMap(repository::save)
            .flatMap(saved -> ServerResponse
                .created(URI.create("/api/products/" + saved.getId()))
                .bodyValue(saved));
    }
}
```

---

### Edge Case — Both Starters on Classpath

```xml
<!-- pom.xml — PROBLEMATIC -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

```java
// Result: Spring Boot uses SPRING MVC (Tomcat)
// WebFlux loses — Spring Boot's tie-breaking rule:
// If both present, servlet-based stack (MVC) wins
// WebFlux is still available for WebClient usage
// But the web server is Tomcat/MVC, not Netty/WebFlux

// To force WebFlux when both present:
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(App.class);
        app.setWebApplicationType(WebApplicationType.REACTIVE);
        app.run(args);
    }
}
```

---

### Edge Case — @EnableWebMvc Disabling Boot Auto-Config

```java
// WRONG — disables Boot MVC auto-configuration
@SpringBootApplication
@EnableWebMvc
public class BadConfig { }

// Effect:
// - WebMvcAutoConfiguration backs off (ConditionalOnMissingBean triggers)
// - Jackson MessageConverter: NOT auto-configured
// - Static resources: NOT auto-configured
// - ContentNegotiatingViewResolver: NOT auto-configured
// - You must configure EVERYTHING manually

// CORRECT — extend Boot's configuration
@Configuration
public class GoodConfig implements WebMvcConfigurer {
    // Only override what you need — Boot handles the rest
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
Which statement correctly describes the relationship between Spring Boot and Spring MVC?

A) Spring Boot replaces Spring MVC with a simpler web framework
B) Spring Boot is a web framework that competes with Spring MVC
C) Spring Boot auto-configures Spring MVC but does not change its behavior
D) Spring MVC requires Spring Boot to function

**Answer: C**
Spring Boot auto-configures `DispatcherServlet`, message converters, view resolvers etc. — things you'd configure manually in plain Spring MVC. The `DispatcherServlet` itself behaves identically. Spring MVC works perfectly fine without Spring Boot (plain WAR deployment).

---

**Q2 — Select All That Apply**
Which of the following are TRUE when `@EnableWebMvc` is added to a Spring Boot application?

A) Jackson auto-configuration is disabled
B) Static resource handling is disabled
C) DispatcherServlet is unregistered
D) WebMvcAutoConfiguration backs off due to `@ConditionalOnMissingBean`
E) The application fails to start

**Answer: A, B, D**
`@EnableWebMvc` imports `DelegatingWebMvcConfiguration` which extends `WebMvcConfigurationSupport`. `WebMvcAutoConfiguration` has `@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)` — so it backs off. Jackson and static resource auto-config (part of `WebMvcAutoConfiguration`) are lost. DispatcherServlet remains registered (separate auto-configuration). App starts but is missing Boot's MVC defaults.

---

**Q3 — MCQ**
What is the default embedded server for Spring Boot with `spring-boot-starter-webflux`?

A) Tomcat
B) Jetty
C) Netty
D) Undertow

**Answer: C**
`spring-boot-starter-webflux` defaults to **Netty** (via `reactor-netty`). This is non-servlet, event-loop based. `spring-boot-starter-web` (MVC) defaults to Tomcat.

---

**Q4 — Scenario**
A developer adds both `spring-boot-starter-web` and `spring-boot-starter-webflux` to a Spring Boot application without any additional configuration. What happens?

A) Application fails to start — conflicting starters
B) WebFlux takes precedence — Netty starts
C) Spring MVC takes precedence — Tomcat starts
D) Both run on different ports

**Answer: C**
When both starters are present, Spring Boot defaults to the servlet-based stack (Spring MVC + Tomcat). `WebApplicationType` resolves to `SERVLET`. WebFlux is still usable for `WebClient` but the server is Tomcat/MVC.

---

**Q5 — Code Prediction**
```java
@RestController
public class TestController {
    @GetMapping("/test")
    public Mono<String> test() {
        return Mono.just("hello");
    }
}
```
This controller is in a **Spring MVC** application (not WebFlux). What happens?

A) Compilation error — Mono not supported in Spring MVC
B) Runtime error — Spring MVC cannot process Mono return type
C) Works correctly — Spring MVC supports Mono via async support
D) Returns empty response

**Answer: C**
Spring MVC supports `Mono<T>` and `Flux<T>` return types via `ReactiveTypeHandler` (introduced in Spring 5). It adapts reactor types to `DeferredResult` internally. The response is the resolved value "hello". This is NOT the same as WebFlux — it's MVC handling reactive return types on a Servlet container.

---

**Q6 — True/False**
Spring WebFlux can replace JDBC with no code changes to improve performance.

**Answer: False**
JDBC is inherently blocking. Using JDBC in a WebFlux application blocks the event loop thread, degrading performance worse than just using Spring MVC. WebFlux requires reactive database drivers (R2DBC for relational, reactive MongoDB driver, etc.) to realize its non-blocking benefits.

---

**Q7 — MCQ**
Which interface is the WebFlux equivalent of Spring MVC's `DispatcherServlet`?

A) `ReactiveDispatcherServlet`
B) `DispatcherHandler`
C) `WebHandler`
D) `RouterFunction`

**Answer: B**
`DispatcherHandler` is WebFlux's central dispatcher. It implements `WebHandler` and serves the same orchestration role as `DispatcherServlet` in MVC, but operates on reactive types (`ServerWebExchange`) rather than Servlet API objects.

---

**Q8 — Select All That Apply**
Which annotations/features are shared between Spring MVC and Spring WebFlux?

A) `@Controller`
B) `@RequestMapping`
C) `HandlerInterceptor`
D) `@ExceptionHandler`
E) `@ControllerAdvice`
F) `javax.servlet.Filter`

**Answer: A, B, D, E**
`@Controller`, `@RequestMapping`, `@ExceptionHandler`, `@ControllerAdvice` work in both. `HandlerInterceptor` is MVC-specific — WebFlux uses `WebFilter`. `javax.servlet.Filter` is Servlet-only — WebFlux uses `WebFilter` (reactive).

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — "Spring Boot is required for Spring MVC"**
Absolutely false. Spring MVC predates Spring Boot by 7 years. Plain Spring MVC WAR deployments are valid and common in enterprise. Spring Boot is optional convenience.

**Trap 2 — "@EnableWebMvc is needed in Spring Boot"**
The opposite is true. `@EnableWebMvc` in Spring Boot disables Boot's MVC auto-configuration. The correct approach is `implements WebMvcConfigurer` without `@EnableWebMvc`. This is a top-3 Spring Boot MVC mistake.

**Trap 3 — "WebFlux is always faster"**
WebFlux is faster only for I/O-bound workloads with high concurrency AND when all I/O is non-blocking. For CPU-bound work, CRUD apps with JDBC, or moderate concurrency, Spring MVC performs equally well or better. Certification questions test this nuance.

**Trap 4 — "Mono/Flux can't be used in Spring MVC"**
Spring MVC (5+) supports `Mono` and `Flux` return types. They're adapted to async Servlet processing internally. The application is still running on Tomcat with blocking semantics overall — but reactive return types work. This surprises candidates who think Mono/Flux = WebFlux only.

**Trap 5 — "Both starters = both frameworks running"**
Both starters on classpath does NOT mean two servers run. Spring Boot picks ONE web application type. MVC wins by default. This is a tie-breaking rule, not an error.

**Trap 6 — "WebFlux requires Netty"**
WebFlux can run on Servlet 3.1+ containers (Tomcat, Jetty) — it just loses some non-blocking benefits in that case. Netty is the default for Boot+WebFlux but not mandatory. On Tomcat, WebFlux uses async Servlet support.

---

## 5️⃣ SUMMARY SHEET

```
THREE-WAY DISTINCTION
─────────────────────────────────────────────────────
Spring Framework  = Foundation (IoC, AOP, Data, Web)
Spring MVC        = Servlet-based synchronous web framework
Spring WebFlux    = Reactive non-blocking web framework
Spring Boot       = Auto-configuration + embedded server tool
                    (wraps either MVC or WebFlux)

SPRING MVC CHARACTERISTICS
─────────────────────────────────────────────────────
• Servlet API — javax/jakarta.servlet
• Thread-per-request — blocking I/O
• DispatcherServlet — front controller
• Default server: Tomcat
• Return types: POJO, ResponseEntity, ModelAndView
• Filters: javax/jakarta.servlet.Filter
• Mature — since Spring 2.5 (2007)

SPRING WEBFLUX CHARACTERISTICS
─────────────────────────────────────────────────────
• Project Reactor — Mono<T>, Flux<T>
• Event loop — non-blocking I/O
• DispatcherHandler — front handler
• Default server: Netty
• Return types: Mono<T>, Flux<T>
• Filters: WebFilter (reactive)
• Introduced: Spring 5 (2017)

SPRING BOOT + SPRING MVC INTERACTION
─────────────────────────────────────────────────────
Boot auto-configures: DispatcherServlet, Jackson, ViewResolvers,
  static resources, ContentNegotiation, LocaleResolver

@EnableWebMvc in Boot → WebMvcAutoConfiguration BACKS OFF
  → Loses: Jackson, static resources, content negotiation defaults
  → NEVER use @EnableWebMvc in Spring Boot

Correct Boot customization: implements WebMvcConfigurer (no @EnableWebMvc)

BOTH STARTERS CONFLICT RESOLUTION
─────────────────────────────────────────────────────
spring-boot-starter-web + spring-boot-starter-webflux
→ Spring MVC wins (Tomcat)
→ To force WebFlux: app.setWebApplicationType(REACTIVE)

SHARED BETWEEN MVC AND WEBFLUX
─────────────────────────────────────────────────────
@Controller, @RestController, @RequestMapping family
@ExceptionHandler, @ControllerAdvice
@RequestBody, @ResponseBody, @PathVariable, @RequestParam
Spring IoC container, AOP, Validation, Security integration

NOT SHARED
─────────────────────────────────────────────────────
MVC: DispatcherServlet, HandlerInterceptor, javax.servlet.Filter
WebFlux: DispatcherHandler, WebFilter, ServerRequest/ServerResponse

WHEN TO CHOOSE WHAT
─────────────────────────────────────────────────────
Spring MVC:    JDBC, JPA, team knows blocking, moderate concurrency
Spring WebFlux: R2DBC, reactive Mongo, SSE, WebSocket, >10K concurrent
Spring Boot:   ALWAYS — unless deploying WAR to external container

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "Spring Boot auto-configures Spring MVC — it doesn't replace it"
• "@EnableWebMvc disables Spring Boot's MVC auto-configuration"
• "WebFlux needs fully non-blocking I/O — JDBC in WebFlux blocks the event loop"
• "Mono/Flux return types work in Spring MVC — adapted via DeferredResult"
• "Both starters present — Spring MVC wins by default"
• "WebFlux uses DispatcherHandler, not DispatcherServlet"
• "Spring MVC predates Spring Boot by 7 years"
• "WebFlux can run on Tomcat — Netty is default, not mandatory"
```

---

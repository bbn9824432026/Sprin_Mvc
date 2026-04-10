# TOPIC 3.7 — HANDLER INTERCEPTORS: PRE, POST, AFTER COMPLETION

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What HandlerInterceptor Really Is

`HandlerInterceptor` is Spring MVC's mechanism for **cross-cutting concerns at the handler level** — sitting between the DispatcherServlet's request routing and the actual handler invocation. It is architecturally distinct from Servlet Filters (which operate at the Servlet container level, before Spring MVC) and from AOP advice (which operates at the method level within Spring beans). Understanding where interceptors sit in the full stack, what they can and cannot do, and how their three-method contract behaves under all conditions is the core of this topic.

---

### Interceptors vs Filters vs AOP — The Three Levels

```
HTTP Request
     │
     ▼
┌─────────────────────────────────────────────────────┐
│  SERVLET FILTER CHAIN (container level)             │
│  DelegatingFilterProxy → Spring Security Filter     │
│  CharacterEncodingFilter                            │
│  OncePerRequestFilter implementations               │
│  Access: raw HttpServletRequest/Response            │
│  No Spring context: beans via DelegatingFilterProxy │
│  Fires: for ALL URL patterns matched                │
└─────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────┐
│  DISPATCHERSERVLET                                  │
│                                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │  HANDLER INTERCEPTORS (Spring MVC level)      │  │
│  │  HandlerMapping → HandlerExecutionChain       │  │
│  │  preHandle() → handler invocation → postHandle│  │
│  │  Access: HttpServletRequest, handler object   │  │
│  │  Spring beans available via @Autowired        │  │
│  │  Fires: only for handlers matched by mapping  │  │
│  └───────────────────────────────────────────────┘  │
│                                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │  AOP ADVICE (bean level)                      │  │
│  │  @Before, @After, @Around on @Controller beans│  │
│  │  Access: method arguments, return value       │  │
│  │  Full Spring context available                │  │
│  │  Fires: per-method on proxied beans           │  │
│  └───────────────────────────────────────────────┘  │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Key differences:**

```
Feature              Filter          Interceptor        AOP
─────────────────────────────────────────────────────────────
Level                Container       Spring MVC         Spring Bean
Spring context       No (direct)     YES                YES
Accesses handler     NO              YES (HandlerMethod) YES (method)
URL filtering        Pattern-based   Pattern-based      Pointcut-based
@Autowired works     Via Proxy       YES (it's a bean)  YES
Execution scope      All requests    Mapped requests    Method calls
Can stop chain       YES (doFilter)  YES (false return)  @Around
Response access      YES             YES                 YES
```

---

### HandlerInterceptor Interface — Precise Contract

```java
public interface HandlerInterceptor {

    // Called BEFORE handler method execution
    // Return true: continue processing
    // Return false: stop processing (afterCompletion called for prior)
    default boolean preHandle(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler  // HandlerMethod, HttpRequestHandler, etc.
    ) throws Exception {
        return true;
    }

    // Called AFTER handler execution, BEFORE view rendering
    // Only called if preHandle returned true for this interceptor
    // NOT called if exception was thrown by handler
    // NOT called if async processing started
    // modelAndView is null for @ResponseBody endpoints
    default void postHandle(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler,
        @Nullable ModelAndView modelAndView  // null for @ResponseBody
    ) throws Exception {
    }

    // Called AFTER complete request processing (including rendering)
    // Called EVEN IF exception occurred
    // Called EVEN IF preHandle of a LATER interceptor returned false
    // NOT called if THIS interceptor's preHandle returned false
    // Exceptions here are SWALLOWED (logged only)
    default void afterCompletion(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler,
        @Nullable Exception ex  // null if no exception
    ) throws Exception {
    }
}
```

---

### AsyncHandlerInterceptor — For Async Requests

```java
// Extends HandlerInterceptor for async request handling
public interface AsyncHandlerInterceptor extends HandlerInterceptor {

    // Called instead of postHandle/afterCompletion when
    // async handling starts (Callable/DeferredResult returned)
    // Called on the request-processing thread when async starts
    // afterCompletion called on the async dispatch thread
    default void afterConcurrentHandlingStarted(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler
    ) throws Exception {
    }
}
```

**Async interceptor lifecycle:**
```
First dispatch (Callable returned):
  preHandle()                           ← called on Servlet thread
  handler.invoke() → Callable started
  afterConcurrentHandlingStarted()      ← called on Servlet thread
  [Servlet thread released]

  [Worker thread executes Callable]

Second dispatch (result ready):
  preHandle()                           ← called AGAIN on new Servlet thread
  handler.invoke() → processes result
  postHandle()                          ← called on new Servlet thread
  afterCompletion()                     ← called on new Servlet thread
```

---

### Interceptor Execution Order — The Complete Rules

```
Three interceptors: A, B, C (registered in this order)

NORMAL REQUEST (all preHandle return true):
  A.preHandle()      → true
  B.preHandle()      → true
  C.preHandle()      → true
  [handler executes]
  C.postHandle()     ← REVERSE ORDER
  B.postHandle()     ← REVERSE ORDER
  A.postHandle()     ← REVERSE ORDER
  [view renders]
  C.afterCompletion() ← REVERSE ORDER
  B.afterCompletion() ← REVERSE ORDER
  A.afterCompletion() ← REVERSE ORDER

B.preHandle() RETURNS FALSE:
  A.preHandle()      → true  (interceptorIndex=0)
  B.preHandle()      → false (STOP — afterCompletion for A)
  [handler NOT executed]
  [postHandle NOT called for any]
  A.afterCompletion() ← only A (index 0..interceptorIndex=0)
  B.afterCompletion() ← NOT called (B returned false)
  C.afterCompletion() ← NOT called (never ran preHandle)

EXCEPTION IN HANDLER:
  A.preHandle()      → true
  B.preHandle()      → true
  C.preHandle()      → true
  [handler throws exception → stored as dispatchException]
  [postHandle NOT called for any — exception occurred]
  [exception handling: processHandlerException]
  C.afterCompletion(ex) ← REVERSE ORDER with exception
  B.afterCompletion(ex)
  A.afterCompletion(ex)

EXCEPTION IN postHandle:
  A.preHandle()
  B.preHandle()
  [handler executes]
  C.postHandle()
  B.postHandle() → throws exception → stored as dispatchException
  [A.postHandle() NOT called — exception stops postHandle chain]
  C.afterCompletion(ex)
  B.afterCompletion(ex)
  A.afterCompletion(ex)
```

---

### MappedInterceptor — URL Pattern Filtering

When interceptors are registered with path patterns via `addInterceptors()`:

```java
@Override
public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(new LoggingInterceptor())
            .addPathPatterns("/**")
            .excludePathPatterns("/static/**", "/health");

    registry.addInterceptor(new AuthInterceptor())
            .addPathPatterns("/api/**", "/admin/**")
            .excludePathPatterns("/api/public/**");
}
```

The interceptors are wrapped in `MappedInterceptor`:

```java
// MappedInterceptor.matches() — called during HandlerExecutionChain assembly
public boolean matches(HttpServletRequest request) {
    String lookupPath = getLookupPathForRequest(request);

    // Check include patterns first
    if (this.includePatterns != null) {
        boolean match = false;
        for (PathPattern pattern : this.includePatterns) {
            if (pattern.matches(
                    PathPatternParser.defaultInstance
                        .parse(lookupPath))) {
                match = true;
                break;
            }
        }
        if (!match) return false;
    }

    // Then check exclude patterns
    if (this.excludePatterns != null) {
        for (PathPattern pattern : this.excludePatterns) {
            if (pattern.matches(
                    PathPatternParser.defaultInstance
                        .parse(lookupPath))) {
                return false; // EXCLUDED
            }
        }
    }

    return true;
}
```

**URL pattern rules for interceptors:**
```
/**       → ALL paths (including /products/42/reviews)
/api/**   → /api/ and any sub-path
/api/*    → /api/products but NOT /api/products/42
/products → EXACT /products only
Pattern precedence: include patterns checked, then exclude patterns
Exclude always wins over include for same URL
```

---

### The handler Parameter — What It Is

```java
// In preHandle(), postHandle(), afterCompletion():
Object handler  // This is the actual handler object

// Common types:
if (handler instanceof HandlerMethod hm) {
    // Most common: @RequestMapping method handler
    Class<?> controllerClass = hm.getBeanType();
    Method method = hm.getMethod();
    String beanName = hm.getBean().toString();

    // Access annotations on method
    ResponseBody rb = hm.getMethodAnnotation(ResponseBody.class);
    RequestMapping rm = hm.getMethodAnnotation(RequestMapping.class);

    // Access annotations on controller class
    RestController rc = hm.getBeanType()
        .getAnnotation(RestController.class);
}

if (handler instanceof ResourceHttpRequestHandler rhrh) {
    // Static resource handler
    List<Resource> locations = rhrh.getLocations();
}

if (handler instanceof HttpRequestHandler hrh) {
    // Simple HttpRequestHandler (e.g., default servlet handler)
}
```

---

### What preHandle() Can and Cannot Do

```java
@Override
public boolean preHandle(HttpServletRequest request,
                          HttpServletResponse response,
                          Object handler) throws Exception {

    // CAN DO:
    // 1. Read request headers, params, URI
    String authHeader = request.getHeader("Authorization");
    String uri = request.getRequestURI();

    // 2. Set request attributes (available in handler)
    request.setAttribute("startTime", System.nanoTime());
    request.setAttribute("traceId", generateTraceId());

    // 3. Write to response and return false (short-circuit)
    if (authHeader == null) {
        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.getWriter().write(
            "{\"error\":\"Unauthorized\"}");
        return false; // Stop processing
    }

    // 4. Modify request (via wrapper — advanced)
    // 5. Log, metrics, tracing

    // CANNOT DO (in terms of meaningful impact):
    // 6. Modify response body after handler wrote it
    //    (postHandle is too late for @ResponseBody)
    // 7. Access method arguments of the handler
    //    (not resolved yet — that happens in ha.handle())
    // 8. Access the return value of the handler
    //    (not returned yet)

    return true;
}
```

---

### What postHandle() Can and Cannot Do

```java
@Override
public void postHandle(HttpServletRequest request,
                        HttpServletResponse response,
                        Object handler,
                        @Nullable ModelAndView modelAndView)
        throws Exception {

    // CAN DO for VIEW-BASED responses:
    // 1. Add model attributes (view not rendered yet)
    if (modelAndView != null) {
        modelAndView.addObject("globalVersion", "2.0");
        modelAndView.addObject("requestId",
            request.getAttribute("requestId"));
    }

    // 2. Change view name (redirect, forward)
    if (modelAndView != null &&
            !modelAndView.getViewName().startsWith("redirect:")) {
        // Add security headers to non-redirect responses
        response.setHeader("X-Content-Type-Options",
            "nosniff");
    }

    // 3. Read timing from request attributes
    Long startTime = (Long) request.getAttribute("startTime");
    if (startTime != null) {
        long elapsed = System.nanoTime() - startTime;
        // Log or record metric (response not written yet for view)
    }

    // CANNOT DO for @ResponseBody responses:
    // modelAndView IS NULL — response already written
    // Cannot modify JSON/XML body
    // Cannot change HTTP status
    // Cannot add headers after body written (may be committed)

    // ALWAYS: postHandle is called BEFORE view rendering
    // Changes to modelAndView ARE reflected in view output
}
```

---

### What afterCompletion() Should Do

```java
@Override
public void afterCompletion(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler,
                              @Nullable Exception ex)
        throws Exception {

    // GUARANTEED to run (for interceptors whose preHandle succeeded)
    // Use for:

    // 1. Resource cleanup
    SomeThreadLocalResource resource =
        (SomeThreadLocalResource)
        request.getAttribute("resource");
    if (resource != null) {
        resource.close();
    }

    // 2. Performance logging
    Long startTime = (Long) request.getAttribute("startTime");
    if (startTime != null) {
        long elapsed = System.nanoTime() - startTime;
        metricsService.recordRequestDuration(
            request.getRequestURI(),
            request.getMethod(),
            response.getStatus(),
            elapsed);
    }

    // 3. Exception-aware cleanup
    if (ex != null) {
        // Request failed — log with exception details
        log.warn("Request failed: {} {}",
            request.getMethod(),
            request.getRequestURI(), ex);
    } else {
        // Request succeeded
        log.debug("Request completed: {} {} {}",
            request.getMethod(),
            request.getRequestURI(),
            response.getStatus());
    }

    // 4. Clear security context (if not using Spring Security)
    SecurityContextHolder.clearContext();

    // EXCEPTIONS HERE ARE SWALLOWED:
    // Throwing an exception in afterCompletion is logged
    // but does NOT affect the response
    // The response has already been committed
    // Use try-catch if cleanup can fail
    try {
        cleanupOperation();
    } catch (Exception e) {
        log.error("Cleanup failed", e);
        // Do NOT re-throw — would be swallowed anyway
    }
}
```

---

### Interceptor Registration — All Forms

```java
// FORM 1: WebMvcConfigurer.addInterceptors()
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {

        // Basic registration — applies to all requests
        registry.addInterceptor(new LoggingInterceptor());

        // With path patterns
        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/api/**", "/admin/**")
                .excludePathPatterns(
                    "/api/public/**",
                    "/api/auth/**"
                );

        // With explicit ordering
        registry.addInterceptor(new TracingInterceptor())
                .addPathPatterns("/**")
                .order(Ordered.HIGHEST_PRECEDENCE);
                // Runs before other interceptors

        // Spring beans can be @Autowired into interceptors
        // when registered as beans
        registry.addInterceptor(rateLimitInterceptor());

        // Built-in interceptors
        LocaleChangeInterceptor lci = new LocaleChangeInterceptor();
        lci.setParamName("lang");
        registry.addInterceptor(lci);
    }

    @Bean
    public RateLimitInterceptor rateLimitInterceptor() {
        return new RateLimitInterceptor(rateLimitService());
    }
}
```

**XML configuration:**
```xml
<mvc:interceptors>
    <!-- Global interceptor (no mapping = all requests) -->
    <bean class="com.example.LoggingInterceptor"/>

    <!-- Mapped interceptor -->
    <mvc:interceptor>
        <mvc:mapping path="/api/**"/>
        <mvc:exclude-mapping path="/api/public/**"/>
        <bean class="com.example.AuthInterceptor"/>
    </mvc:interceptor>
</mvc:interceptors>
```

---

### Built-In Interceptors — The Important Ones

```java
// 1. LocaleChangeInterceptor
// Changes locale based on request parameter
LocaleChangeInterceptor lci = new LocaleChangeInterceptor();
lci.setParamName("lang");  // ?lang=fr changes locale
lci.setHttpMethods("GET", "POST");  // Limit to these methods
// Requires SessionLocaleResolver or CookieLocaleResolver
// AcceptHeaderLocaleResolver → UnsupportedOperationException

// 2. ThemeChangeInterceptor (deprecated in Spring 6)
ThemeChangeInterceptor tci = new ThemeChangeInterceptor();
tci.setParamName("theme");  // ?theme=dark

// 3. UserRoleAuthorizationInterceptor
UserRoleAuthorizationInterceptor auth =
    new UserRoleAuthorizationInterceptor();
auth.setAuthorizedRoles("ADMIN", "MANAGER");
// Checks request.isUserInRole()

// 4. WebContentInterceptor
WebContentInterceptor wci = new WebContentInterceptor();
wci.setCacheSeconds(3600);  // Set Cache-Control headers
wci.addCacheMapping(
    CacheControl.maxAge(Duration.ofHours(1)),
    "/static/**"
);
wci.addCacheMapping(
    CacheControl.noStore(),
    "/api/**"
);
```

---

### Interceptors and HandlerMethod — Annotation-Based Logic

```java
// Interceptor that checks @RequiresRole annotation on handler method
@Component
public class RoleBasedInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) throws IOException {

        // Only process @RequestMapping handlers
        if (!(handler instanceof HandlerMethod hm)) {
            return true; // Pass through for other handler types
        }

        // Check method-level annotation
        RequiresRole roleAnnotation =
            hm.getMethodAnnotation(RequiresRole.class);

        // Check class-level annotation if no method-level
        if (roleAnnotation == null) {
            roleAnnotation = AnnotationUtils.findAnnotation(
                hm.getBeanType(), RequiresRole.class);
        }

        // No annotation → allow through
        if (roleAnnotation == null) {
            return true;
        }

        // Check if user has required role
        String requiredRole = roleAnnotation.value();
        if (!request.isUserInRole(requiredRole)) {
            response.setStatus(HttpStatus.FORBIDDEN.value());
            response.setContentType(
                MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write(
                "{\"error\":\"Insufficient privileges\"}");
            return false;
        }

        return true;
    }
}

// Custom annotation
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface RequiresRole {
    String value();
}
```

---

### Request-Scoped Data Sharing via Request Attributes

```java
// Pattern: preHandle sets data, handler uses it, afterCompletion cleans up
@Component
public class RequestContextInterceptor
    implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) {
        // Generate and set trace ID
        String traceId = UUID.randomUUID().toString();
        request.setAttribute("traceId", traceId);

        // Set in MDC for logging
        MDC.put("traceId", traceId);
        MDC.put("uri", request.getRequestURI());
        MDC.put("method", request.getMethod());

        // Record start time
        request.setAttribute("startTime",
            System.currentTimeMillis());

        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                  HttpServletResponse response,
                                  Object handler,
                                  Exception ex) {
        // ALWAYS clean up MDC — thread pool reuse risk
        MDC.remove("traceId");
        MDC.remove("uri");
        MDC.remove("method");

        // Log completion
        Long start = (Long) request.getAttribute("startTime");
        if (start != null) {
            long elapsed = System.currentTimeMillis() - start;
            log.info("Request: {} {} → {} ({}ms)",
                request.getMethod(),
                request.getRequestURI(),
                response.getStatus(),
                elapsed);
        }
    }
}

// Handler method can access the trace ID:
@GetMapping("/products")
public List<Product> getProducts(HttpServletRequest request) {
    String traceId = (String) request.getAttribute("traceId");
    log.debug("Processing request with trace: {}", traceId);
    return productService.findAll();
}
```

---

### Rate Limiting Interceptor Pattern

```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {

    private final RateLimiter rateLimiter;
    private final Cache<String, AtomicInteger> requestCounts;

    public RateLimitInterceptor(
            RateLimiterConfig config) {
        this.rateLimiter = new RateLimiter(config);
        // Using Caffeine or Guava cache for rate tracking
        this.requestCounts = Caffeine.newBuilder()
            .expireAfterWrite(1, TimeUnit.MINUTES)
            .build();
    }

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) throws IOException {

        String clientId = extractClientId(request);
        int limit = getLimit(handler);

        AtomicInteger count = requestCounts.get(
            clientId, k -> new AtomicInteger(0));

        int current = count.incrementAndGet();

        // Set rate limit headers
        response.setHeader("X-Rate-Limit-Limit",
            String.valueOf(limit));
        response.setHeader("X-Rate-Limit-Remaining",
            String.valueOf(Math.max(0, limit - current)));

        if (current > limit) {
            response.setStatus(
                HttpStatus.TOO_MANY_REQUESTS.value());
            response.setContentType(
                MediaType.APPLICATION_JSON_VALUE);
            response.setHeader("Retry-After", "60");
            response.getWriter().write(
                "{\"error\":\"Rate limit exceeded\"}");
            return false;
        }

        return true;
    }

    private String extractClientId(HttpServletRequest request) {
        // API key from header, or IP address
        String apiKey = request.getHeader("X-API-Key");
        return apiKey != null ? apiKey :
            request.getRemoteAddr();
    }

    private int getLimit(Object handler) {
        if (handler instanceof HandlerMethod hm) {
            RateLimit annotation =
                hm.getMethodAnnotation(RateLimit.class);
            if (annotation != null) {
                return annotation.value();
            }
        }
        return 100; // Default: 100 requests/minute
    }
}
```

---

## 2️⃣ CODE EXAMPLES

### Complete Interceptor Implementation Suite

```java
// ─── 1. AUTHENTICATION INTERCEPTOR ──────────────────────────
@Component
public class JwtAuthInterceptor implements HandlerInterceptor {

    @Autowired
    private JwtTokenValidator jwtValidator;

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) throws IOException {

        // Skip authentication for non-controller handlers
        if (!(handler instanceof HandlerMethod)) {
            return true;
        }

        // Skip if @Public annotation present
        HandlerMethod hm = (HandlerMethod) handler;
        if (hm.hasMethodAnnotation(Public.class) ||
                hm.getBeanType().isAnnotationPresent(Public.class)) {
            return true;
        }

        // Extract JWT from Authorization header
        String authHeader = request.getHeader("Authorization");
        if (authHeader == null ||
                !authHeader.startsWith("Bearer ")) {
            sendError(response, HttpStatus.UNAUTHORIZED,
                "Missing authorization token");
            return false;
        }

        String token = authHeader.substring(7);
        try {
            Claims claims = jwtValidator.validate(token);
            // Store in request for downstream use
            request.setAttribute("userId",
                claims.getSubject());
            request.setAttribute("userRoles",
                claims.get("roles"));
            return true;
        } catch (JwtException ex) {
            sendError(response, HttpStatus.UNAUTHORIZED,
                "Invalid or expired token");
            return false;
        }
    }

    private void sendError(HttpServletResponse response,
            HttpStatus status, String message)
            throws IOException {
        response.setStatus(status.value());
        response.setContentType(
            MediaType.APPLICATION_JSON_VALUE);
        response.getWriter().write(
            "{\"error\":\"" + message + "\"}");
    }
}

// ─── 2. PERFORMANCE MONITORING INTERCEPTOR ──────────────────
@Component
public class PerformanceInterceptor
    implements HandlerInterceptor {

    @Autowired
    private MeterRegistry meterRegistry;

    private static final String START_TIME_KEY = "reqStart";
    private static final String TIMER_SAMPLE_KEY = "timerSample";

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) {
        request.setAttribute(START_TIME_KEY,
            System.nanoTime());
        request.setAttribute(TIMER_SAMPLE_KEY,
            Timer.start(meterRegistry));
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                  HttpServletResponse response,
                                  Object handler,
                                  Exception ex) {
        Timer.Sample sample = (Timer.Sample)
            request.getAttribute(TIMER_SAMPLE_KEY);

        if (sample != null && handler instanceof HandlerMethod hm) {
            String controllerName = hm.getBeanType()
                .getSimpleName();
            String methodName = hm.getMethod().getName();

            sample.stop(Timer.builder("http.server.requests")
                .tag("controller", controllerName)
                .tag("method", methodName)
                .tag("status",
                    String.valueOf(response.getStatus()))
                .tag("exception",
                    ex != null ? ex.getClass().getSimpleName()
                               : "none")
                .register(meterRegistry));
        }
    }
}

// ─── 3. AUDIT LOGGING INTERCEPTOR ───────────────────────────
@Component
public class AuditInterceptor implements HandlerInterceptor {

    @Autowired
    private AuditService auditService;

    @Override
    public void afterCompletion(HttpServletRequest request,
                                  HttpServletResponse response,
                                  Object handler,
                                  Exception ex) {
        // Only audit mutating operations
        String method = request.getMethod();
        if (!method.equals("POST") &&
                !method.equals("PUT") &&
                !method.equals("DELETE") &&
                !method.equals("PATCH")) {
            return;
        }

        AuditEvent event = AuditEvent.builder()
            .userId((String) request.getAttribute("userId"))
            .action(method)
            .resource(request.getRequestURI())
            .status(response.getStatus())
            .timestamp(Instant.now())
            .success(ex == null &&
                response.getStatus() < 400)
            .build();

        // Async to avoid blocking the response
        auditService.logAsync(event);
    }
}
```

---

### Interceptor Registration with Order Control

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Autowired private JwtAuthInterceptor jwtAuth;
    @Autowired private PerformanceInterceptor perf;
    @Autowired private AuditInterceptor audit;
    @Autowired private RateLimitInterceptor rateLimit;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {

        // ORDER MATTERS: execution in registration order for preHandle
        // Reverse order for postHandle and afterCompletion

        // 1. Performance/Tracing — FIRST (outermost timing)
        registry.addInterceptor(perf)
                .addPathPatterns("/**")
                .order(1);

        // 2. Rate Limiting — before auth
        registry.addInterceptor(rateLimit)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/health")
                .order(2);

        // 3. Authentication — after rate limiting
        registry.addInterceptor(jwtAuth)
                .addPathPatterns("/api/**", "/admin/**")
                .excludePathPatterns(
                    "/api/public/**",
                    "/api/auth/login",
                    "/api/auth/refresh"
                )
                .order(3);

        // 4. Audit — last in preHandle (has auth info from attr)
        registry.addInterceptor(audit)
                .addPathPatterns("/api/**")
                .order(4);

        // Built-in: Locale change
        LocaleChangeInterceptor locale =
            new LocaleChangeInterceptor();
        locale.setParamName("lang");
        registry.addInterceptor(locale)
                .addPathPatterns("/**")
                .order(10);
    }
}
```

---

### Testing Interceptors with MockMvc

```java
@ExtendWith(SpringExtension.class)
@WebMvcTest(controllers = ProductController.class)
class InterceptorTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private ProductService productService;

    @Test
    void testAuthInterceptorBlocksWithoutToken() throws Exception {
        mockMvc.perform(get("/api/products"))
            .andExpect(status().isUnauthorized())
            .andExpect(jsonPath("$.error")
                .value("Missing authorization token"));
    }

    @Test
    void testAuthInterceptorAllowsWithValidToken() throws Exception {
        when(productService.findAll())
            .thenReturn(List.of(new Product(1L, "Widget")));

        mockMvc.perform(get("/api/products")
                .header("Authorization",
                    "Bearer " + validJwtToken()))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$").isArray());
    }

    @Test
    void testRateLimitHeadersPresent() throws Exception {
        mockMvc.perform(get("/api/products")
                .header("Authorization",
                    "Bearer " + validJwtToken()))
            .andExpect(status().isOk())
            .andExpect(header().exists("X-Rate-Limit-Limit"))
            .andExpect(header().exists(
                "X-Rate-Limit-Remaining"));
    }
}
```

---

### HandlerInterceptorAdapter — Legacy (Deprecated in Spring 5.3)

```java
// Pre-Spring 5.3: extend HandlerInterceptorAdapter
// to avoid implementing unused methods
public class LegacyInterceptor
    extends HandlerInterceptorAdapter {
    // Only override what you need
    // preHandle, postHandle, afterCompletion all have default impls
    @Override
    public boolean preHandle(...) { return true; }
}

// Spring 5.3+: use HandlerInterceptor directly
// All three methods have default implementations (return true / no-op)
public class ModernInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(...) { return true; }
    // postHandle and afterCompletion are default no-ops
    // HandlerInterceptorAdapter is deprecated
}
```

---

### Edge Case — Interceptor With preHandle Writing Response

```java
@Component
public class CachingInterceptor implements HandlerInterceptor {

    @Autowired
    private ResponseCache cache;

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) throws IOException {

        // Only cache GET requests
        if (!HttpMethod.GET.matches(request.getMethod())) {
            return true;
        }

        String cacheKey = request.getRequestURI() +
            "?" + request.getQueryString();
        CachedResponse cached = cache.get(cacheKey);

        if (cached != null) {
            // Write cached response directly
            response.setStatus(HttpStatus.OK.value());
            response.setContentType(cached.getContentType());
            response.setHeader("X-Cache", "HIT");
            response.setHeader("Cache-Control",
                "max-age=300");
            response.getWriter().write(cached.getBody());

            // Return FALSE — skip handler execution
            // Handler method NOT called
            // postHandle NOT called
            // afterCompletion IS called for interceptors[0..this-1]
            return false;
        }

        request.setAttribute("cacheKey", cacheKey);
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                  HttpServletResponse response,
                                  Object handler,
                                  Exception ex) {
        // Store response in cache (if this ran, response succeeded)
        String cacheKey = (String)
            request.getAttribute("cacheKey");
        if (cacheKey != null && ex == null &&
                response.getStatus() == 200) {
            // Note: can't easily read response body here
            // Use ContentCachingResponseWrapper if needed
        }
    }
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
Five interceptors (A→E) are registered. All preHandle() return `true`. Interceptor C's `postHandle()` throws an unchecked exception. Which afterCompletion() methods are called?

A) No afterCompletion() — exception in postHandle stops all cleanup
B) E.afterCompletion(), D.afterCompletion(), C.afterCompletion() — from point of failure
C) E.afterCompletion(), D.afterCompletion(), C.afterCompletion(), B.afterCompletion(), A.afterCompletion()
D) A.afterCompletion() only — reverse stops at the interceptor that threw

**Answer: C**
`postHandle()` exceptions are caught, stored as `dispatchException`, and processing continues to `processDispatchResult()`. `triggerAfterCompletion()` is called for ALL interceptors from `interceptorIndex` (E=4) down to 0 (A). ALL five afterCompletion() methods are called with the exception. The exception in `postHandle()` does NOT stop `afterCompletion()` execution.

---

**Q2 — Select All That Apply**
Which statements correctly describe `postHandle()` limitations?

A) `postHandle()` receives `null` for `modelAndView` when the handler uses `@ResponseBody`
B) `postHandle()` can modify the JSON body of a `@ResponseBody` response
C) `postHandle()` runs before view rendering
D) `postHandle()` is not called when an exception is thrown in the handler
E) `postHandle()` is called in REVERSE order of interceptor registration

**Answer: A, C, D, E**
B is wrong — by the time `postHandle()` runs, `@ResponseBody` has ALREADY written to the response stream. The response may be committed. Modifying it is not possible. A, C, D, E are all correct: null modelAndView for `@ResponseBody`, before rendering, skipped on exception, reverse order.

---

**Q3 — Code Prediction**
```java
// Interceptors registered in order: [Timing, Auth, Audit]
// Auth.preHandle() returns false
```
Which methods are called?

A) Timing.pre, Auth.pre, Timing.after, Auth.after
B) Timing.pre, Auth.pre, Timing.after
C) Timing.pre, Auth.pre(false) → Timing.after
D) Timing.pre, Auth.pre(false) → Auth.after, Timing.after

**Answer: C**
`applyPreHandle()` records `interceptorIndex=0` (Timing succeeded at index 0). When Auth at index 1 returns false, `triggerAfterCompletion()` iterates from `interceptorIndex=0` down to 0. Only `Timing.afterCompletion()` is called. Auth returned false → Auth's afterCompletion NOT called. Audit never ran preHandle → Audit's afterCompletion NOT called.

---

**Q4 — MCQ**
A `Callable<Product>` is returned by the handler. Interceptor `TracingInterceptor` implements `AsyncHandlerInterceptor`. Which sequence is correct for the full async lifecycle?

A) preHandle → handle → postHandle → afterCompletion (all on one thread)
B) First dispatch: preHandle → handle → afterConcurrentHandlingStarted; Second dispatch: preHandle → handle → postHandle → afterCompletion
C) First dispatch: preHandle → handle; Second dispatch: postHandle → afterCompletion
D) preHandle → handle → afterConcurrentHandlingStarted → postHandle → afterCompletion

**Answer: B**
Async lifecycle with `AsyncHandlerInterceptor`: First dispatch (Servlet thread): `preHandle()`, handler executes (Callable created), `afterConcurrentHandlingStarted()`. Servlet thread released. Second dispatch (result ready, new Servlet thread): `preHandle()` called AGAIN, handler processes result, `postHandle()`, `afterCompletion()`.

---

**Q5 — True/False**
An interceptor added via `addInterceptors()` without any path patterns applies to ALL requests including static resource requests handled by `ResourceHttpRequestHandler`.

**Answer: True**
When no path patterns are specified, the interceptor is registered as a raw `HandlerInterceptor` (not `MappedInterceptor`). In `AbstractHandlerMapping.getHandlerExecutionChain()`, non-mapped interceptors are added to EVERY handler execution chain regardless of the handler type. This includes `ResourceHttpRequestHandler` for `/static/**`. If you want to exclude static resources, add `.excludePathPatterns("/static/**")`.

---

**Q6 — MCQ**
`afterCompletion()` is guaranteed to run for an interceptor. Which of the following will NOT cause it to be skipped?

A) The interceptor's own `preHandle()` returned `false`
B) A later interceptor's `preHandle()` returned `false`
C) An exception was thrown in the handler method
D) The response was committed before `afterCompletion()` ran

**Answer: B, C, D**
Only A causes this interceptor's `afterCompletion()` to be skipped — when ITS OWN `preHandle()` returned false. B: a later interceptor returning false still calls `afterCompletion()` for THIS interceptor (which succeeded preHandle). C: exceptions trigger `afterCompletion()` with the exception as parameter. D: response being committed has no effect on `afterCompletion()` execution.

---

**Q7 — Select All That Apply**
What can be legitimately accessed via the `handler` parameter in an interceptor's `preHandle()` method?

A) The URL pattern that matched the request
B) The `@RequestMapping` annotation on the handler method
C) The return value of the handler method
D) The controller class that contains the handler method
E) Custom annotations on the handler method

**Answer: A, B, D, E**
For `HandlerMethod` handlers: A (via `request.getAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE)`), B (`hm.getMethodAnnotation(RequestMapping.class)`), D (`hm.getBeanType()`), E (any annotation via `hm.getMethodAnnotation()`). C is wrong — `preHandle()` runs BEFORE handler invocation, so there is no return value yet.

---

**Q8 — Scenario**
An interceptor's `preHandle()` sets `request.setAttribute("authUser", user)`. The handler method signature is `public Product getProduct(HttpServletRequest request, @PathVariable Long id)`. How does the handler access the user set by the interceptor?

A) Not possible — interceptors and handlers run in different contexts
B) Via `request.getAttribute("authUser")` inside the handler
C) Via `@RequestAttribute("authUser") User user` parameter
D) Via `@SessionAttribute("authUser") User user` parameter

**Answer: B and C**
B: `request.getAttribute("authUser")` works since it's the SAME request object. C: `@RequestAttribute("authUser")` is handled by `RequestAttributeMethodArgumentResolver` which calls `request.getAttribute("authUser")` — exactly the same mechanism, just cleaner. Both work. D is wrong — session attributes are stored in `HttpSession`, not request attributes.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — `postHandle()` is useless for `@ResponseBody` responses**
The most important interceptor trap. For REST endpoints, the response body is written INSIDE `ha.handle()` (Phase 6). `postHandle()` runs AFTER Phase 6. The response is already written, possibly committed. `modelAndView` is `null`. Adding headers in `postHandle()` may throw `IllegalStateException` if response is committed. Use `ResponseBodyAdvice` to intercept `@ResponseBody` writing, or add headers in `preHandle()`.

**Trap 2 — `afterCompletion()` exceptions are silently swallowed**
Code in `afterCompletion()` that throws will have its exception logged at DEBUG/WARN level but NOT propagated. The response is already committed at this point. Developers relying on exceptions from `afterCompletion()` for error detection will miss them. Always wrap `afterCompletion()` cleanup code in try-catch.

**Trap 3 — interceptorIndex tracks SUCCESSFUL preHandles only**
`interceptorIndex` is set to the index of the LAST interceptor whose `preHandle()` returned `true`. When B's preHandle returns false, `interceptorIndex` holds A's index (0), not B's (1). `triggerAfterCompletion()` iterates from `interceptorIndex` DOWN to 0, NOT from B's index. B itself (returned false) → afterCompletion NOT called.

**Trap 4 — Interceptors added without path patterns run for EVERYTHING**
No path pattern = MappedInterceptor not created = applies to ALL handlers including `ResourceHttpRequestHandler` for CSS/JS/images. This is a performance issue — authentication interceptors shouldn't run for static resources. Always specify `.addPathPatterns("/api/**")` or `.excludePathPatterns("/static/**")`.

**Trap 5 — Async requests: `preHandle()` called TWICE**
For `Callable<T>` or `DeferredResult<T>`, the interceptor's `preHandle()` is called on BOTH the initial dispatch AND the async result dispatch. If your `preHandle()` has side effects (incrementing counters, modifying state), those effects happen twice. Use `DispatcherType.ASYNC` check to skip on second pass: `request.getDispatcherType() == DispatcherType.ASYNC`.

**Trap 6 — Interceptor `handler` parameter is not always `HandlerMethod`**
For static resource requests, `handler` is `ResourceHttpRequestHandler`. For BeanNameUrlHandlerMapping handlers, it's the bean instance. For `DefaultServletHttpRequestHandler`, it's a different type. Always check `handler instanceof HandlerMethod` before casting. Failing to check → `ClassCastException` in production for static resource requests.

**Trap 7 — `HandlerInterceptorAdapter` is deprecated in Spring 5.3**
Spring 5.3 deprecated `HandlerInterceptorAdapter`. The reason: all `HandlerInterceptor` methods have default implementations. Extending an adapter class was only needed when Java didn't have default interface methods. Modern code implements `HandlerInterceptor` directly. Some exam questions present code using `HandlerInterceptorAdapter` — it still works but is deprecated.

---

## 5️⃣ SUMMARY SHEET

```
HANDLERINTERCEPTOR CONTRACT
─────────────────────────────────────────────────────
preHandle(req, res, handler) → boolean
  return true:  continue processing
  return false: STOP — afterCompletion for prior interceptors

postHandle(req, res, handler, mav)
  Called AFTER handler, BEFORE view rendering
  mav = null for @ResponseBody (response already written)
  NOT called if: preHandle returned false, exception in handler,
                 async started, HTTP 304

afterCompletion(req, res, handler, ex)
  Called AFTER everything (including rendering)
  ALWAYS called for interceptors whose preHandle succeeded
  ex = null on success, exception on failure
  Exceptions here: SWALLOWED (logged only)

EXECUTION ORDER
─────────────────────────────────────────────────────
preHandle:       forward order (A, B, C)
postHandle:      REVERSE order (C, B, A)
afterCompletion: REVERSE order (C, B, A)

interceptorIndex = index of LAST successful preHandle
afterCompletion only runs for [0..interceptorIndex]

WHEN EACH IS SKIPPED
─────────────────────────────────────────────────────
postHandle skipped:
  → Any preHandle returned false
  → Exception in ha.handle()
  → Async processing started
  → HTTP 304 Not Modified

afterCompletion skipped (for that interceptor):
  → Its OWN preHandle returned false
  → It never ran preHandle (later interceptor failed earlier)

ASYNC INTERCEPTOR (AsyncHandlerInterceptor)
─────────────────────────────────────────────────────
afterConcurrentHandlingStarted():
  Called on first dispatch when async starts
  Called INSTEAD OF postHandle/afterCompletion on first dispatch

Second dispatch (async result):
  preHandle() called AGAIN (check DispatcherType.ASYNC to skip)
  postHandle() called
  afterCompletion() called

HANDLER TYPE CHECKING
─────────────────────────────────────────────────────
if (handler instanceof HandlerMethod hm):
  hm.getBeanType()        → controller class
  hm.getMethod()          → the method
  hm.getBean()            → controller bean instance
  hm.getMethodAnnotation(Ann.class) → annotation on method
  hm.getBeanType().getAnnotation()  → annotation on class

For static resources: handler instanceof ResourceHttpRequestHandler
ALWAYS check type before casting

URL PATTERN FILTERING
─────────────────────────────────────────────────────
.addPathPatterns("/**")       → all paths
.addPathPatterns("/api/**")   → /api/ and all sub-paths
.excludePathPatterns("/static/**") → exclude static
No path patterns → applies to ALL handlers (including static resources)
Exclude always wins over include for same URL

FILTER vs INTERCEPTOR vs AOP
─────────────────────────────────────────────────────
Filter:       Container level, no Spring context direct, ALL URLs
Interceptor:  Spring MVC level, Spring beans available, mapped URLs
AOP:          Bean level, Spring proxy, method-level pointcuts

POSTHANDLE AND @RESPONSEBODY
─────────────────────────────────────────────────────
Response written in ha.handle() Phase 6
postHandle() runs after Phase 6 — response ALREADY written
modelAndView IS NULL for @ResponseBody
Adding headers in postHandle → possible IllegalStateException
FIX: Use ResponseBodyAdvice for @ResponseBody interception
     Use preHandle() for headers that don't depend on response

AFTERCOMPLETION FOR RESOURCE CLEANUP
─────────────────────────────────────────────────────
Guaranteed cleanup point (like try-finally)
Clear MDC (logging context)
Record metrics/timing
Release thread-local resources
Check ex parameter for error-aware cleanup

REGISTRATION ORDER = EXECUTION ORDER (preHandle)
─────────────────────────────────────────────────────
registry.addInterceptor(a).order(1)
registry.addInterceptor(b).order(2)
registry.addInterceptor(c).order(3)
preHandle:       a → b → c
postHandle:      c → b → a
afterCompletion: c → b → a

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "postHandle() is useless for @ResponseBody — response already written"
• "afterCompletion() exceptions are swallowed — always use try-catch inside"
• "interceptorIndex tracks successful preHandles — determines afterCompletion scope"
• "Async: preHandle called TWICE — check DispatcherType.ASYNC to skip second"
• "No path patterns = interceptor applies to ALL handlers including static resources"
• "handler instanceof HandlerMethod — always check before casting"
• "HandlerInterceptorAdapter deprecated Spring 5.3 — implement HandlerInterceptor directly"
• "preHandle false: afterCompletion for prior interceptors, NOT for the false-returning one"
• "postHandle order: REVERSE of preHandle registration order"
• "Use request.setAttribute() in preHandle to pass data to handler and afterCompletion"
```

---

**PART 3 — CONTROLLERS AND REQUEST MAPPING COMPLETE** ✅

```
3.1 @Controller and @RestController: Differences and Internals    ✅
3.2 @RequestMapping: URL Patterns, Methods, Params, Headers        ✅
3.3 Method Arguments: Complete Resolution Reference                ✅
3.4 Method Return Values: Complete Handler Reference               ✅
3.5 Data Binding: @ModelAttribute, WebDataBinder, Validation       ✅
3.6 Exception Handling: @ExceptionHandler, @ControllerAdvice       ✅
3.7 Handler Interceptors: Pre, Post, After Completion              ✅
```

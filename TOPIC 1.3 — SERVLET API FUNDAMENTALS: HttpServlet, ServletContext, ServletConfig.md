# TOPIC 1.3 — SERVLET API FUNDAMENTALS: HttpServlet, ServletContext, ServletConfig

---

## 1️⃣ CONCEPTUAL EXPLANATION

### Why Servlet API Matters for Spring MVC Mastery

Spring MVC is **not magic**. It is a sophisticated wrapper around the Servlet API. Every `@RequestMapping` method ultimately receives its data from `HttpServletRequest`. Every response is ultimately written to `HttpServletResponse`. Every Spring MVC concept — scopes, filters, listeners, initialization — maps directly to a Servlet API concept.

If you don't understand the Servlet API deeply, you will misunderstand Spring MVC at every level. Certification questions frequently test whether you know **where Spring ends and the Servlet API begins**.

---

### The Servlet Specification — What It Is

The Servlet specification is a **Java EE / Jakarta EE standard** that defines:
- How HTTP requests are received and processed by Java applications
- The lifecycle of servlet objects
- How servlet containers (Tomcat, Jetty, Undertow) must behave
- Scoping rules for request, session, and application data
- The filter and listener mechanisms

Current version: **Servlet 6.0** (Jakarta EE 10). Spring 6.x requires Jakarta EE 9+ (namespace `jakarta.servlet.*`). Spring 5.x used `javax.servlet.*`.

This namespace change is a **certification trap** — code using `javax.servlet` won't compile with Spring 6.

---

### The Servlet Container — Role & Responsibilities

The container (Tomcat) is responsible for:

```
1. Listening on a port (default 8080)
2. Accepting TCP connections
3. Parsing raw HTTP bytes into HttpServletRequest objects
4. Managing thread pool — assigning threads to requests
5. Finding the correct Servlet for the request URL
6. Calling servlet.service(request, response)
7. Writing HttpServletResponse back to the client
8. Managing Servlet lifecycle (init, service, destroy)
9. Managing Session lifecycle
10. Managing ServletContext (one per web application)
```

Spring MVC operates **inside step 6**. The container hands Spring a request object and expects a response object to be populated. Everything Spring does is within this boundary.

---

### HttpServlet — The Foundation

`HttpServlet` extends `GenericServlet` extends `Servlet`. The hierarchy:

```
Servlet (interface)
    └── GenericServlet (abstract)
            └── HttpServlet (abstract)
                    └── DispatcherServlet (Spring)
                            └── FrameworkServlet (Spring)
                                    └── HttpServletBean (Spring)
```

`HttpServlet.service()` routes to method-specific handlers:

```java
// Inside HttpServlet — simplified
protected void service(HttpServletRequest req, HttpServletResponse resp) {
    String method = req.getMethod();
    if ("GET".equals(method)) {
        doGet(req, resp);
    } else if ("POST".equals(method)) {
        doPost(req, resp);
    } else if ("PUT".equals(method)) {
        doPut(req, resp);
    } else if ("DELETE".equals(method)) {
        doDelete(req, resp);
    }
    // ... etc
}
```

`DispatcherServlet` overrides `doService()` (Spring's abstraction layer) which is called by `doGet`, `doPost`, etc. in `FrameworkServlet`. This means **all HTTP methods funnel into one method** in Spring — `DispatcherServlet.doDispatch()`.

---

### DispatcherServlet Inheritance Chain — Detailed

```
HttpServletBean
│  • Reads <init-param> from web.xml into bean properties
│  • Calls initServletBean() on init
│
FrameworkServlet
│  • Creates / retrieves WebApplicationContext
│  • Overrides doGet, doPost, doPut, doDelete, doPatch, doOptions, doTrace
│  • All HTTP methods call → processRequest(request, response)
│  • processRequest() sets up LocaleContext and RequestAttributes (ThreadLocal)
│  • Then calls doService() — abstract in FrameworkServlet
│
DispatcherServlet
   • Implements doService()
   • Sets special attributes on request (WebApplicationContext, LocaleResolver, etc.)
   • Calls doDispatch() — the core dispatching logic
```

This chain is critical. When a POST request arrives:
1. Tomcat calls `FrameworkServlet.doPost()`
2. `doPost()` calls `processRequest()`
3. `processRequest()` initializes ThreadLocals, calls `doService()`
4. `doService()` sets request attributes, calls `doDispatch()`
5. `doDispatch()` does the actual handler lookup and invocation

---

### HttpServletRequest — What Spring Extracts From It

`HttpServletRequest` carries everything about the incoming HTTP request:

```java
// URL information
request.getRequestURI()        // /api/products/42
request.getRequestURL()        // http://localhost:8080/api/products/42
request.getContextPath()       // /myapp (if deployed with context path)
request.getServletPath()       // /api/products/42
request.getPathInfo()          // null (or remaining path)
request.getQueryString()       // page=1&size=10

// HTTP method
request.getMethod()            // GET, POST, PUT, DELETE...

// Headers
request.getHeader("Accept")    // application/json
request.getHeader("Content-Type") // application/json
request.getHeaderNames()       // Enumeration of all header names

// Parameters (query string + form body)
request.getParameter("page")   // "1"
request.getParameterMap()      // Map<String, String[]>

// Body
request.getInputStream()       // Raw body as InputStream (read ONCE)
request.getReader()            // Body as BufferedReader (read ONCE)

// Session
request.getSession()           // Creates session if not exists
request.getSession(false)      // Returns null if no session exists

// Attributes (request-scoped storage)
request.setAttribute("key", value)
request.getAttribute("key")

// Security
request.getUserPrincipal()
request.isUserInRole("ADMIN")
```

**Critical trap:** `getInputStream()` and `getReader()` can each be called **only once**. After the body is read, it is consumed. This is why Spring's `@RequestBody` works only once per request. Filters that read the body must use `ContentCachingRequestWrapper` to allow re-reading.

---

### HttpServletResponse — What Spring Writes To It

```java
// Status
response.setStatus(200)
response.sendError(404, "Not Found")

// Headers
response.setHeader("Content-Type", "application/json")
response.setHeader("Cache-Control", "no-cache")
response.addCookie(new Cookie("sessionId", "abc123"))

// Body
response.getWriter()           // Write text content
response.getOutputStream()     // Write binary content
// Again — use one or the other, not both

// Redirect
response.sendRedirect("/new-url")  // Sends 302
```

**Critical:** Once `response.getWriter().flush()` or `response.getOutputStream().flush()` is called, the response is **committed**. After commit:
- You cannot change the status code
- You cannot add headers
- You cannot forward to another resource

Spring's exception handling must happen **before** the response is committed. This is why exception resolvers are invoked early in the chain.

---

### ServletContext — Application-Wide Scope

`ServletContext` represents **the entire web application**. There is exactly **one** `ServletContext` per web application per JVM.

```java
// Lifecycle: created when app deploys, destroyed when app undeploys

// What it provides:
servletContext.getContextPath()           // "/myapp"
servletContext.getRealPath("/WEB-INF")    // Filesystem path
servletContext.getResourceAsStream("/WEB-INF/config.xml")
servletContext.getAttribute("key")       // Application-scoped attributes
servletContext.setAttribute("key", val)  // Store application-scoped data
servletContext.getInitParameter("dbUrl") // From <context-param> in web.xml
servletContext.log("message")            // Container logging
```

**Spring's use of ServletContext:**
- `WebApplicationContext` is stored as an attribute in `ServletContext`
- Key: `WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE`
- This is how `ContextLoaderListener` makes the root context available to `DispatcherServlet`
- `ServletContext` is injected into beans that implement `ServletContextAware`

---

### ServletConfig — Servlet-Specific Configuration

`ServletConfig` is per-servlet (unlike `ServletContext` which is per-application).

```xml
<!-- web.xml -->
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/dispatcher-servlet.xml</param-value>
    </init-param>
    <init-param>
        <param-name>throwExceptionIfNoHandlerFound</param-name>
        <param-value>true</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
```

```java
// Accessed via:
servletConfig.getInitParameter("contextConfigLocation")
servletConfig.getServletName()     // "dispatcher"
servletConfig.getServletContext()  // Access to ServletContext from here
```

`HttpServletBean` (grandparent of DispatcherServlet) reads `ServletConfig` init params and maps them to bean properties using Spring's `BeanWrapper`. This is how `contextConfigLocation` init-param becomes `DispatcherServlet.setContextConfigLocation()`.

---

### Servlet Lifecycle — Precise Sequence

```
1. Container starts
        │
        ▼
2. ServletContext created (one per app)
        │
        ▼
3. ServletContextListeners notified → contextInitialized()
   (ContextLoaderListener runs here → creates Root ApplicationContext)
        │
        ▼
4. Filters initialized
        │
        ▼
5. Servlets initialized (if load-on-startup >= 0)
   Servlet.init(ServletConfig) called
   → HttpServletBean.init()
   → FrameworkServlet.initServletBean()
   → DispatcherServlet creates WebApplicationContext
   → HandlerMappings, HandlerAdapters, ViewResolvers initialized
        │
        ▼
6. Application serves requests
   For each request:
   → Filter chain executes (pre-processing)
   → Servlet.service() called
   → Filter chain executes (post-processing)
        │
        ▼
7. Application shuts down
   → Servlet.destroy() called
   → Filters destroyed
   → ServletContextListeners notified → contextDestroyed()
   → ServletContext destroyed
```

---

### Request Scope vs Session Scope vs Application Scope

This maps directly to Servlet API:

| Scope | Servlet API | Spring Annotation | Lifetime |
|---|---|---|---|
| Request | `HttpServletRequest.setAttribute()` | `@RequestScope` | One HTTP request |
| Session | `HttpSession.setAttribute()` | `@SessionScope` | User session (timeout-based) |
| Application | `ServletContext.setAttribute()` | `@ApplicationScope` | App lifetime |
| Flash | Via Session, cleared after next request | `RedirectAttributes` | One redirect cycle |

Spring's scoped proxies use `ThreadLocal` + Servlet API scopes under the hood.

---

### Servlet 3.0+ — Programmatic Registration

Servlet 3.0 (2009) introduced the ability to register servlets, filters, and listeners **programmatically** without `web.xml`. Spring leverages this via `ServletContainerInitializer`:

```java
// Spring's hook into Servlet 3.0 startup
// META-INF/services/javax.servlet.ServletContainerInitializer contains:
// org.springframework.web.SpringServletContainerInitializer

public class SpringServletContainerInitializer implements ServletContainerInitializer {
    @Override
    public void onStartup(Set<Class<?>> webAppInitializerClasses, 
                          ServletContext servletContext) {
        // Finds all WebApplicationInitializer implementations
        // Calls their onStartup() methods
        // This is how AbstractAnnotationConfigDispatcherServletInitializer works
    }
}
```

This is the **foundation of zero-XML Spring MVC configuration**. The container finds Spring's `ServletContainerInitializer` via Java `ServiceLoader`, which then discovers your `WebApplicationInitializer` subclass.

---

### Async Servlet Support (Servlet 3.0+)

Servlet 3.0 introduced async processing — the ability to release the container thread while processing continues on another thread:

```java
// Traditional (blocking)
// Container thread handles entire request — held until response written

// Async (non-blocking container thread)
AsyncContext asyncContext = request.startAsync();
asyncContext.start(() -> {
    // runs on different thread
    // container thread is RELEASED back to pool
    doLongOperation();
    asyncContext.complete(); // signals response is ready
});
```

Spring MVC's `DeferredResult` and `Callable` support maps to this Servlet 3.0 async mechanism. This will be covered in depth in Topic 4.11.

---

### Filters vs Servlets — Execution Model

```
Incoming Request
       │
       ▼
  Filter 1.doFilter() ──→ chain.doFilter() ──┐
                                              │
                                         Filter 2.doFilter() ──→ chain.doFilter() ──┐
                                                                                     │
                                                                              DispatcherServlet.service()
                                                                                     │
                                                                              (Spring MVC processing)
                                                                                     │
                                                                              Response written
                                                                                     │
                                         Filter 2 (post-processing) ←───────────────┘
  Filter 1 (post-processing) ←──────────────────────────────────────────────────────┘
       │
       ▼
  Response sent to client
```

Filters execute **outside** Spring's context. They have no access to Spring beans by default (unless using `DelegatingFilterProxy` — which bridges Servlet filter chain into Spring bean). This is how Spring Security operates.

---

### Memory & Performance Implications

- `ServletContext` is shared — never store large mutable objects in it
- `HttpSession` is per-user — in clustered environments, session data is serialized/replicated. Keep sessions small.
- `HttpServletRequest` attributes are fast (HashMap internally) — preferred for request-scoped data sharing between filters and servlets
- Reading `request.getInputStream()` twice causes `IllegalStateException` — always use `ContentCachingRequestWrapper` when body re-reading is needed
- `response.getWriter()` vs `response.getOutputStream()` — calling both on same response causes `IllegalStateException`

---

## 2️⃣ CODE EXAMPLES

### Servlet Lifecycle — Complete Example

```java
// A raw servlet (for comparison — Spring replaces this pattern)
public class LegacyServlet extends HttpServlet {

    private DataSource dataSource; // initialized in init()

    @Override
    public void init(ServletConfig config) throws ServletException {
        super.init(config); // MUST call super — loads ServletConfig
        String dbUrl = config.getInitParameter("dbUrl");
        // Initialize resources here — called ONCE
        this.dataSource = createDataSource(dbUrl);
    }

    @Override
    protected void doGet(HttpServletRequest req, 
                         HttpServletResponse resp) throws IOException {
        // Called per request — must be thread-safe
        String id = req.getParameter("id");
        resp.setContentType("text/html");
        resp.getWriter().write("<h1>Item: " + id + "</h1>");
    }

    @Override
    public void destroy() {
        // Cleanup — called ONCE before undeployment
        closeDataSource(dataSource);
    }
}
```

---

### Accessing Servlet API in Spring MVC Controllers

```java
@Controller
public class ServletAwareController {

    // Option 1 — Inject directly as method parameter (preferred)
    @GetMapping("/info")
    public String info(HttpServletRequest request,
                       HttpServletResponse response,
                       HttpSession session) {
        
        String userAgent = request.getHeader("User-Agent");
        String sessionId = session.getId();
        response.setHeader("X-Custom", "value");
        
        return "info";
    }

    // Option 2 — Inject ServletContext via Spring
    @Autowired
    private ServletContext servletContext;

    @GetMapping("/context")
    @ResponseBody
    public String contextInfo() {
        return servletContext.getContextPath();
    }

    // Option 3 — Use RequestContextHolder (anywhere in call stack)
    @GetMapping("/anywhere")
    @ResponseBody
    public String anywhere() {
        HttpServletRequest request = ((ServletRequestAttributes)
            RequestContextHolder.currentRequestAttributes()).getRequest();
        return request.getRemoteAddr();
    }
}
```

---

### ContentCachingRequestWrapper — Re-Reading Body

```java
// Filter that logs request body without consuming it for Spring
@Component
public class LoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) 
                                    throws ServletException, IOException {

        // Wrap request to allow body re-reading
        ContentCachingRequestWrapper wrappedRequest = 
            new ContentCachingRequestWrapper(request);

        chain.doFilter(wrappedRequest, response);  // Spring reads body here

        // AFTER chain — body is now cached in wrapper
        byte[] body = wrappedRequest.getContentAsByteArray();
        log.info("Request body: {}", new String(body, StandardCharsets.UTF_8));
        // Body is still available for any subsequent reads
    }
}
```

---

### Programmatic Servlet Registration (Zero XML)

```java
public class MyWebInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class[]{RootConfig.class};
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[]{WebConfig.class};
    }

    @Override
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }

    // Customizing the DispatcherServlet registration
    @Override
    protected void customizeRegistration(ServletRegistration.Dynamic registration) {
        // Maps to Servlet 3.0 programmatic registration
        registration.setInitParameter("throwExceptionIfNoHandlerFound", "true");
        registration.setMultipartConfig(
            new MultipartConfigElement("/tmp", 5_000_000, 10_000_000, 0)
        );
    }

    // Adding filters programmatically
    @Override
    protected Filter[] getServletFilters() {
        CharacterEncodingFilter encodingFilter = new CharacterEncodingFilter();
        encodingFilter.setEncoding("UTF-8");
        encodingFilter.setForceEncoding(true);
        return new Filter[]{encodingFilter};
    }
}
```

---

### Session Scope — Servlet API vs Spring Abstraction

```java
// Raw Servlet API approach
@GetMapping("/cart/add")
public String addToCart(HttpSession session, @RequestParam Long productId) {
    @SuppressWarnings("unchecked")
    List<Long> cart = (List<Long>) session.getAttribute("cart");
    if (cart == null) {
        cart = new ArrayList<>();
        session.setAttribute("cart", cart);
    }
    cart.add(productId);
    return "redirect:/cart";
}

// Spring abstraction approach — @SessionScope bean
@Component
@SessionScope  // Lives for the duration of user's HTTP session
public class ShoppingCart {
    private List<Long> items = new ArrayList<>();
    public void addItem(Long id) { items.add(id); }
    public List<Long> getItems() { return items; }
}

@Controller
public class CartController {
    @Autowired
    private ShoppingCart cart;  // Spring injects session-scoped proxy
    
    @PostMapping("/cart/add")
    public String add(@RequestParam Long productId) {
        cart.addItem(productId);  // Operates on current user's session instance
        return "redirect:/cart";
    }
}
```

---

### Edge Case — Calling Both getWriter() and getOutputStream()

```java
@GetMapping("/bad")
public void bad(HttpServletResponse response) throws IOException {
    response.getWriter().write("text");
    response.getOutputStream().write("bytes".getBytes()); 
    // THROWS: IllegalStateException: getWriter() has already been called
}

// Spring's MessageConverters handle this correctly:
// Text converters use getWriter()
// Binary converters use getOutputStream()
// Spring never mixes them for the same response
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
At what point in the Servlet lifecycle is `ContextLoaderListener.contextInitialized()` called?

A) When the first HTTP request arrives
B) When `DispatcherServlet.init()` is called
C) When `ServletContext` is created during application deployment
D) When the JVM starts

**Answer: C**
`ServletContextListener.contextInitialized()` is called immediately after `ServletContext` creation, before any servlet initialization and before any requests are processed. This is why the root `ApplicationContext` (with services and repositories) is available before `DispatcherServlet` initializes.

---

**Q2 — Select All That Apply**
Which of the following are TRUE about `HttpServletRequest.getInputStream()`?

A) Can be called multiple times safely
B) Can only be called once — subsequent calls throw IllegalStateException
C) Cannot be used if getReader() has already been called
D) Spring's @RequestBody reads from this stream
E) ContentCachingRequestWrapper allows re-reading the body

**Answer: C, D, E**
A is wrong — second call on same stream reads nothing (stream exhausted), and calling after `getReader()` throws `IllegalStateException`. B is partially wrong — it's not that it throws on second call, but that the stream is exhausted. C is correct — using both throws `IllegalStateException`. D is correct — `RequestResponseBodyMethodProcessor` reads from `getInputStream()`. E is correct.

---

**Q3 — Scenario**
A developer writes a Servlet Filter that reads `request.getInputStream()` to log the body, then calls `chain.doFilter(request, response)`. Spring's `@RequestBody` in the controller receives an empty object. Why?

A) Filters cannot access request body
B) `getInputStream()` was consumed in the filter — Spring reads an empty stream
C) Spring Security blocked the request
D) `@RequestBody` requires `Content-Type: application/json` header

**Answer: B**
The `InputStream` is a one-time readable stream. Once the filter consumed it, the stream position is at EOF. Spring's `HttpMessageConverter` reads from the same stream and gets nothing. Fix: use `ContentCachingRequestWrapper`.

---

**Q4 — MCQ**
Where does Spring's `WebApplicationContext` (root context) get stored so that `DispatcherServlet` can find it?

A) In a static field of `ContextLoaderListener`
B) As an attribute of `HttpSession`
C) As an attribute of `ServletContext`
D) In a `ThreadLocal` variable

**Answer: C**
`ContextLoaderListener` stores the root `ApplicationContext` in `ServletContext` under the key `WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE`. `DispatcherServlet` retrieves it from `ServletContext` during initialization to set it as the parent context.

---

**Q5 — Code Output**
```java
@GetMapping("/test")
public void test(HttpServletResponse response) throws IOException {
    response.setStatus(200);
    response.getWriter().write("Hello");
    response.sendRedirect("/other");
}
```
What happens?

A) Response sends "Hello" then redirects
B) `IllegalStateException` — response already committed when sendRedirect called
C) Redirect to /other with empty body
D) Both actions execute — client sees redirect

**Answer: B**
After `getWriter().write("Hello")` and assuming the response buffer is flushed (or if `sendRedirect` is called after commit), `IllegalStateException` is thrown. Even if not yet committed, `sendRedirect` resets the response — but mixing write and redirect is always an error in practice. The response should do one or the other.

---

**Q6 — MCQ**
In Spring 6.x, what is the correct import for `HttpServletRequest`?

A) `javax.servlet.http.HttpServletRequest`
B) `jakarta.servlet.http.HttpServletRequest`
C) `org.springframework.web.HttpServletRequest`
D) `org.springframework.servlet.http.HttpServletRequest`

**Answer: B**
Spring 6 requires Jakarta EE 9+, which uses the `jakarta.*` namespace. The `javax.servlet.*` namespace was used up to Spring 5.x. This is a common compilation error when migrating from Spring 5 to Spring 6.

---

**Q7 — Ordering**
Order the Servlet lifecycle events from first to last during application startup:

- DispatcherServlet.init() called
- First HTTP request processed
- Filter.init() called
- ServletContextListener.contextInitialized() called
- ServletContext created

**Correct Order:**
1. ServletContext created
2. ServletContextListener.contextInitialized() called
3. Filter.init() called
4. DispatcherServlet.init() called
5. First HTTP request processed

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — javax vs jakarta**
Spring 5 → `javax.servlet`. Spring 6 → `jakarta.servlet`. Examiners present code with `javax.servlet` imports in a Spring 6 context — the code **does not compile**. Always check the import package in questions about Servlet API classes.

**Trap 2 — load-on-startup behavior**
`load-on-startup` with value `-1` or omitted means the servlet initializes on **first request** — not at startup. Value `0` or higher means initialization at startup, with lower values initializing first. Spring Boot sets this to `1` by default. In plain Spring MVC without `load-on-startup=1`, the first request pays the initialization cost — a common production surprise.

**Trap 3 — ServletContext vs ApplicationContext**
`ServletContext` is a Servlet API concept — application-wide data store, one per web app. `ApplicationContext` is a Spring concept — IoC container, bean factory. They are stored in the same JVM but are fundamentally different objects. Confusing them is a common error.

**Trap 4 — Session creation**
`request.getSession()` CREATES a session if none exists. `request.getSession(false)` returns `null` if no session exists. Many security vulnerabilities come from inadvertently creating sessions. Spring Security is careful to use `getSession(false)` before authentication.

**Trap 5 — Filter vs Interceptor access to Spring beans**
A raw `Filter` has no Spring context — it cannot `@Autowire` beans. To use Spring beans in a Filter, you must either:
- Use `DelegatingFilterProxy` (delegates to a Spring-managed bean)
- Use `WebApplicationContextUtils.getWebApplicationContext(servletContext)` manually
This is why Spring Security uses `DelegatingFilterProxy` as the bridge.

**Trap 6 — destroy() guarantee**
`Servlet.destroy()` is NOT guaranteed to be called in all cases. If the JVM crashes or is killed with `SIGKILL`, `destroy()` does not execute. Do not rely on it for critical resource cleanup — use JVM shutdown hooks or database-level cleanup instead.

---

## 5️⃣ SUMMARY SHEET

```
SERVLET API HIERARCHY
─────────────────────────────────────────────────────
Servlet → GenericServlet → HttpServlet
                                └── HttpServletBean (Spring)
                                        └── FrameworkServlet (Spring)
                                                └── DispatcherServlet (Spring)

SERVLET LIFECYCLE
─────────────────────────────────────────────────────
ServletContext created
    → ServletContextListeners (ContextLoaderListener → Root AppContext)
    → Filter.init()
    → Servlet.init() [if load-on-startup >= 0]
    → Requests served (Filter chain → Servlet.service())
    → Servlet.destroy()
    → Filter.destroy()
    → ServletContextListeners.contextDestroyed()
    → ServletContext destroyed

THREE SCOPES — SERVLET API
─────────────────────────────────────────────────────
Request   → HttpServletRequest.setAttribute/getAttribute  (1 request)
Session   → HttpSession.setAttribute/getAttribute         (user session)
App       → ServletContext.setAttribute/getAttribute      (app lifetime)

KEY CLASSES & THEIR ROLES
─────────────────────────────────────────────────────
HttpServletRequest  → Incoming HTTP request data (read-only from app perspective)
HttpServletResponse → Outgoing HTTP response (write headers, body, status)
HttpSession         → Per-user server-side state storage
ServletContext      → Application-wide config and shared state
ServletConfig       → Per-servlet init parameters

CRITICAL RESTRICTIONS
─────────────────────────────────────────────────────
getInputStream()    → Read ONCE — use ContentCachingRequestWrapper to re-read
getWriter()         → Cannot mix with getOutputStream() on same response
sendRedirect()      → Cannot call after response committed
response headers    → Cannot modify after response committed

SPRING 5 vs SPRING 6 NAMESPACE
─────────────────────────────────────────────────────
Spring 5: javax.servlet.http.HttpServletRequest
Spring 6: jakarta.servlet.http.HttpServletRequest
           ↑ This change breaks compilation — common migration trap

SERVLET 3.0+ KEY FEATURES
─────────────────────────────────────────────────────
• Programmatic servlet/filter registration (no web.xml needed)
• Async processing (request.startAsync())
• ServletContainerInitializer (Spring's zero-XML hook)
• Annotation-based config (@WebServlet, @WebFilter, @WebListener)

FILTER vs INTERCEPTOR ACCESS TO SPRING
─────────────────────────────────────────────────────
Raw Filter         → NO Spring context — cannot @Autowire
DelegatingFilterProxy → Bridges Filter to Spring bean (Spring Security uses this)
Interceptor        → IS a Spring bean — full @Autowire support

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "DispatcherServlet extends HttpServlet — Spring MVC IS a Servlet"
• "ServletContext is one per web app — ApplicationContext is Spring's IoC container"
• "getInputStream() is one-time readable — use ContentCachingRequestWrapper to re-read"
• "load-on-startup omitted → servlet initializes on first request, not at startup"
• "Spring 6 requires jakarta.servlet — javax.servlet causes compile errors"
• "Filters cannot @Autowire Spring beans — use DelegatingFilterProxy"
• "ContextLoaderListener runs before DispatcherServlet.init()"
• "request.getSession() creates session — getSession(false) returns null if none"
```

---

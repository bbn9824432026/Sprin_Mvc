# TOPIC 1.4 — DISPATCHERSERVLET: THE FRONT CONTROLLER

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What Is the Front Controller Pattern?

The **Front Controller** is a design pattern where a **single entry point** receives all requests for a web application and delegates them to appropriate handlers. Without this pattern, each URL would map to its own servlet — leading to duplicated cross-cutting logic (logging, security, encoding, exception handling) in every servlet.

Spring MVC's `DispatcherServlet` IS this front controller. Every request that matches its URL mapping flows through it. It owns the entire request processing pipeline.

---

### DispatcherServlet's Core Responsibility

`DispatcherServlet` does **not process requests itself**. Its sole responsibility is **orchestration** — finding the right components and delegating to them in the correct order. This is the Strategy pattern applied at scale.

```
DispatcherServlet knows WHAT needs to happen.
Special beans know HOW to do it.
```

Every behavior is **pluggable**. Every default is **replaceable**. This is the architectural genius of Spring MVC.

---

### The Special Beans — DispatcherServlet's Delegates

`DispatcherServlet` looks for these specific bean types in the `WebApplicationContext`. These are called **special beans**:

| Bean Type | Interface | Default Implementation | Role |
|---|---|---|---|
| HandlerMapping | `HandlerMapping` | `RequestMappingHandlerMapping` | Maps request to handler |
| HandlerAdapter | `HandlerAdapter` | `RequestMappingHandlerAdapter` | Invokes the handler |
| HandlerExceptionResolver | `HandlerExceptionResolver` | `ExceptionHandlerExceptionResolver` | Handles exceptions |
| ViewResolver | `ViewResolver` | `InternalResourceViewResolver` | Resolves view names |
| LocaleResolver | `LocaleResolver` | `AcceptHeaderLocaleResolver` | Resolves locale |
| ThemeResolver | `ThemeResolver` | `FixedThemeResolver` | Resolves theme |
| MultipartResolver | `MultipartResolver` | None (must configure) | Handles file uploads |
| FlashMapManager | `FlashMapManager` | `SessionFlashMapManager` | Manages redirect flash data |

**Critical architectural point:** At startup, `DispatcherServlet.initStrategies()` initializes ALL of these. It first looks for beans of these types in the `WebApplicationContext`. If not found, it falls back to defaults defined in `DispatcherServlet.properties` file (bundled inside `spring-webmvc.jar`).

---

### DispatcherServlet.properties — The Default Strategy File

Inside `spring-webmvc.jar` at path:
`org/springframework/web/servlet/DispatcherServlet.properties`

```properties
# Default HandlerMappings (in order)
org.springframework.web.servlet.HandlerMapping=\
  org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
  org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
  org.springframework.web.servlet.function.support.RouterFunctionMapping

# Default HandlerAdapters
org.springframework.web.servlet.HandlerAdapter=\
  org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
  org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
  org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter,\
  org.springframework.web.servlet.function.support.HandlerFunctionAdapter

# Default HandlerExceptionResolvers
org.springframework.web.servlet.HandlerExceptionResolver=\
  org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
  org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
  org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

# Default ViewResolver
org.springframework.web.servlet.ViewResolver=\
  org.springframework.web.servlet.view.InternalResourceViewResolver
```

This file is the **ground truth** of Spring MVC defaults. When `@EnableWebMvc` is used, this file is bypassed and a more opinionated set of defaults is registered instead.

---

### initStrategies() — What Runs at Startup

```java
// Inside DispatcherServlet — called during init()
protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);        // 1. File upload support
    initLocaleResolver(context);           // 2. Locale detection
    initThemeResolver(context);            // 3. Theme resolution
    initHandlerMappings(context);          // 4. URL → Handler mapping
    initHandlerAdapters(context);          // 5. Handler invocation
    initHandlerExceptionResolvers(context);// 6. Exception handling
    initRequestToViewNameTranslator(context); // 7. Default view name
    initViewResolvers(context);            // 8. View name → View
    initFlashMapManager(context);          // 9. Flash attributes
}
```

Each `initX()` method:
1. Looks for bean(s) of the required type in WebApplicationContext
2. If found — uses them (sorted by `Ordered` / `@Order` if multiple)
3. If NOT found — reads default from `DispatcherServlet.properties`

This runs **once** at servlet initialization. The resulting strategy objects are stored as instance fields of `DispatcherServlet` and reused for every request.

---

### doDispatch() — The Heart of Spring MVC

This is the most important method to understand. Every request flows through here:

```java
// Simplified but accurate representation of doDispatch()
protected void doDispatch(HttpServletRequest request, 
                          HttpServletResponse response) throws Exception {

    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;

    try {
        ModelAndView mv = null;
        Exception dispatchException = null;

        try {
            // STEP 1: Check if multipart (file upload)
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);

            // STEP 2: Find handler (controller method)
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null) {
                noHandlerFound(processedRequest, response); // 404
                return;
            }

            // STEP 3: Find handler adapter
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // STEP 4: Handle Last-Modified (HTTP caching)
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return; // Interceptor said stop
            }

            // STEP 5: Execute interceptors preHandle()
            // (included in applyPreHandle above)

            // STEP 6: Actually invoke the handler (your @RequestMapping method)
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            // STEP 7: Apply default view name if none set
            applyDefaultViewName(processedRequest, mv);

            // STEP 8: Execute interceptors postHandle()
            mappedHandler.applyPostHandle(processedRequest, response, mv);

        } catch (Exception ex) {
            dispatchException = ex;
        }

        // STEP 9: Resolve view or handle exception
        processDispatchResult(processedRequest, response, 
                              mappedHandler, mv, dispatchException);

    } finally {
        // STEP 10: Execute interceptors afterCompletion()
        if (mappedHandler != null) {
            mappedHandler.triggerAfterCompletion(
                processedRequest, response, null);
        }
        // STEP 11: Clean up multipart resources
        if (multipartRequestParsed) {
            cleanupMultipart(processedRequest);
        }
    }
}
```

---

### getHandler() — How Handler Is Found

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) {
    if (this.handlerMappings != null) {
        for (HandlerMapping mapping : this.handlerMappings) {
            HandlerExecutionChain handler = mapping.getHandler(request);
            if (handler != null) {
                return handler; // First match wins
            }
        }
    }
    return null;
}
```

`HandlerMapping` list is **ordered**. First match wins. Default order:
1. `RequestMappingHandlerMapping` (handles `@RequestMapping`)
2. `BeanNameUrlHandlerMapping` (handles bean name = URL)
3. `RouterFunctionMapping` (handles functional endpoints)

`getHandler()` returns a `HandlerExecutionChain` — which is **handler + interceptors**. The interceptors applicable to this URL are assembled here.

---

### getHandlerAdapter() — Matching Adapter to Handler

```java
protected HandlerAdapter getHandlerAdapter(Object handler) {
    for (HandlerAdapter adapter : this.handlerAdapters) {
        if (adapter.supports(handler)) {
            return adapter;
        }
    }
    throw new ServletException("No adapter for handler: " + handler);
}
```

Each `HandlerAdapter.supports(handler)` checks the handler type:
- `RequestMappingHandlerAdapter.supports()` → returns true for `HandlerMethod` instances
- `HttpRequestHandlerAdapter.supports()` → returns true for `HttpRequestHandler` instances
- `SimpleControllerHandlerAdapter.supports()` → returns true for `Controller` interface implementations

---

### processDispatchResult() — View Rendering or Exception Handling

```java
private void processDispatchResult(HttpServletRequest request,
                                   HttpServletResponse response,
                                   HandlerExecutionChain mappedHandler,
                                   ModelAndView mv,
                                   Exception exception) throws Exception {

    if (exception != null) {
        // Exception occurred — try to resolve it
        if (exception instanceof ModelAndViewDefiningException) {
            mv = ((ModelAndViewDefiningException) exception).getModelAndView();
        } else {
            Object handler = mappedHandler != null ? 
                             mappedHandler.getHandler() : null;
            mv = processHandlerException(request, response, handler, exception);
        }
    }

    // Render the view if we have a ModelAndView
    if (mv != null && !mv.wasCleared()) {
        render(mv, request, response);
    }
}
```

---

### render() — View Resolution and Rendering

```java
protected void render(ModelAndView mv, 
                      HttpServletRequest request,
                      HttpServletResponse response) throws Exception {

    // Resolve locale for i18n
    Locale locale = this.localeResolver.resolveLocale(request);
    response.setLocale(locale);

    View view;
    String viewName = mv.getViewName();

    if (viewName != null) {
        // Resolve view name to View object
        view = resolveViewName(viewName, mv.getModelInternal(), 
                               locale, request);
        if (view == null) {
            throw new ServletException("No view found for name: " + viewName);
        }
    } else {
        // View object was returned directly
        view = mv.getView();
    }

    // Actually render the view
    view.render(mv.getModelInternal(), request, response);
}
```

---

### Complete Request Lifecycle — Every Step

```
1. CLIENT sends HTTP GET /products/42
        │
2. TOMCAT accepts TCP connection, parses HTTP, creates
   HttpServletRequest + HttpServletResponse
        │
3. TOMCAT selects DispatcherServlet (URL pattern match: "/")
        │
4. FILTER CHAIN executes (CharacterEncodingFilter, SecurityFilter, etc.)
        │
5. DispatcherServlet.service() → FrameworkServlet.processRequest()
        │
6. FrameworkServlet sets up ThreadLocals:
   - LocaleContextHolder.setLocaleContext()
   - RequestContextHolder.setRequestAttributes()
        │
7. DispatcherServlet.doService()
   Sets request attributes:
   - WEB_APPLICATION_CONTEXT_ATTRIBUTE
   - LOCALE_RESOLVER_ATTRIBUTE
   - THEME_RESOLVER_ATTRIBUTE
   - FLASH_MAP_ATTRIBUTE
        │
8. DispatcherServlet.doDispatch()
        │
9. checkMultipart() — wraps request if Content-Type: multipart/form-data
        │
10. getHandler() — iterates HandlerMappings
    RequestMappingHandlerMapping matches /products/42 → HandlerMethod
    Returns HandlerExecutionChain(HandlerMethod + [interceptors])
        │
11. getHandlerAdapter() — finds RequestMappingHandlerAdapter
    (supports HandlerMethod → true)
        │
12. HandlerInterceptor.preHandle() called for each interceptor
    If any returns false → processing STOPS, response already handled
        │
13. RequestMappingHandlerAdapter.handle()
        │
    a. InvocableHandlerMethod prepares arguments:
       - @PathVariable "42" → Long id = 42L
       - @RequestParam values bound
       - @RequestBody deserialized via HttpMessageConverter
       - Model, HttpServletRequest injected
    b. @ModelAttribute methods executed (if any)
    c. @InitBinder methods registered (if any)
    d. YOUR method invoked via reflection:
       productController.getProduct(42L, model)
    e. Return value processed:
       - String "product/detail" → stored in ModelAndView
       - OR @ResponseBody → written to response immediately
        │
14. applyDefaultViewName() — if no view name, uses URL as view name
        │
15. HandlerInterceptor.postHandle() called (reverse order)
    (NOT called if exception occurred OR @ResponseBody wrote response)
        │
16. processDispatchResult()
    If exception: HandlerExceptionResolvers consulted
    If ModelAndView: render() called
        │
17. render()
    - LocaleResolver.resolveLocale()
    - ViewResolver chain consulted → resolves "product/detail"
      InternalResourceViewResolver → /WEB-INF/views/product/detail.jsp
    - View.render(model, request, response)
      JSP rendered, HTML written to response
        │
18. HandlerInterceptor.afterCompletion() called (reverse order)
    Called EVEN IF exception occurred
        │
19. RequestContextHolder and LocaleContextHolder ThreadLocals CLEARED
        │
20. FrameworkServlet publishes ServletRequestHandledEvent
        │
21. FILTER CHAIN post-processing
        │
22. TOMCAT commits response, sends bytes to client
        │
23. Thread returned to pool
```

---

### Thread-Local Setup and Teardown — Critical Detail

In `FrameworkServlet.processRequest()`:

```java
protected final void processRequest(HttpServletRequest request, 
                                    HttpServletResponse response) {

    // Save previous context (for nested dispatches)
    LocaleContext previousLocaleContext = 
        LocaleContextHolder.getLocaleContext();
    RequestAttributes previousAttributes = 
        RequestContextHolder.getRequestAttributes();

    // SET UP ThreadLocals for this request
    LocaleContextHolder.setLocaleContext(buildLocaleContext(request));
    RequestContextHolder.setRequestAttributes(
        new ServletRequestAttributes(request, response));

    try {
        doService(request, response); // → doDispatch()
    } finally {
        // ALWAYS restore previous state — even on exception
        LocaleContextHolder.resetLocaleContext();
        RequestContextHolder.resetRequestAttributes();

        // Notify ServletRequestAttributes that request is complete
        // This triggers @RequestScope bean destruction
        ((ServletRequestAttributes) previousAttributes)
            .requestCompleted();
    }
}
```

The `finally` block is critical — ThreadLocals are **always** cleaned up regardless of exceptions. Memory leaks in thread pools occur when ThreadLocals aren't cleaned. Spring is careful about this.

---

### noHandlerFound() — The 404 Path

```java
protected void noHandlerFound(HttpServletRequest request, 
                               HttpServletResponse response) throws Exception {

    if (this.throwExceptionIfNoHandlerFound) {
        throw new NoHandlerFoundException(
            request.getMethod(), 
            getRequestUri(request),
            new ServletServerHttpRequest(request).getHeaders()
        );
        // This exception then goes through HandlerExceptionResolvers
        // → Can be caught by @ExceptionHandler(NoHandlerFoundException.class)
    } else {
        // Default behavior — delegate to container's default servlet
        response.sendError(HttpServletResponse.SC_NOT_FOUND);
        // Container handles 404 — Spring has no more involvement
    }
}
```

**Critical trap:** By default, `throwExceptionIfNoHandlerFound` is `false`. This means 404s bypass Spring's exception handling chain entirely — they go to the container. To handle 404s with `@ExceptionHandler`, you must set this to `true` AND disable the default servlet handler.

---

### Multiple DispatcherServlets — When and Why

You can register multiple `DispatcherServlet` instances with different URL mappings:

```java
// API servlet — handles /api/*
public class ApiWebInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
    protected String[] getServletMappings() { return new String[]{"/api/*"}; }
    protected Class<?>[] getServletConfigClasses() { return new Class[]{ApiConfig.class}; }
}

// Admin servlet — handles /admin/*
public class AdminWebInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
    protected String[] getServletMappings() { return new String[]{"/admin/*"}; }
    protected Class<?>[] getServletConfigClasses() { return new Class[]{AdminConfig.class}; }
}
```

Each `DispatcherServlet` gets its **own** `WebApplicationContext`. Both share the same root `ApplicationContext` (services, repositories). This is an enterprise pattern for isolating API and admin concerns.

---

### DispatcherServlet and Static Resources Problem

When `DispatcherServlet` is mapped to `/` (default servlet), it intercepts ALL requests — including requests for CSS, JS, images. Static resource requests find no handler → 404.

Solutions:
1. `<mvc:resources>` / `WebMvcConfigurer.addResourceHandlers()` — tells Spring to serve statics
2. `<mvc:default-servlet-handler/>` — delegates unmatched requests to container's default servlet
3. Map `DispatcherServlet` to `/app/*` instead of `/` — statics bypass Spring entirely

---

### Memory & Performance Implications

- `initStrategies()` runs ONCE — all strategy beans are held as instance fields
- Per-request cost: `getHandler()` is an O(n) walk of HandlerMappings list (tiny, typically 3 items)
- `HandlerExecutionChain` created per request — lightweight wrapper, GC'd quickly
- `ModelAndView` created per request — also lightweight, GC'd after render
- `doDispatch()` has zero synchronization — pure delegation, highly concurrent
- The **only shared mutable state** is the strategy bean references — all immutable after init

---

## 2️⃣ CODE EXAMPLES

### Observing initStrategies via Custom HandlerMapping

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    // Registering a custom HandlerMapping
    // DispatcherServlet will discover this bean and add to its list
    @Bean
    public HandlerMapping customHandlerMapping() {
        SimpleUrlHandlerMapping mapping = new SimpleUrlHandlerMapping();
        mapping.setOrder(Ordered.HIGHEST_PRECEDENCE); // Checked first
        Map<String, Object> urlMap = new HashMap<>();
        urlMap.put("/health", new HealthHandler());
        mapping.setUrlMap(urlMap);
        return mapping;
    }
}

// A raw HttpRequestHandler — handled by HttpRequestHandlerAdapter
public class HealthHandler implements HttpRequestHandler {
    @Override
    public void handleRequest(HttpServletRequest request,
                              HttpServletResponse response) throws IOException {
        response.setStatus(200);
        response.getWriter().write("UP");
    }
}
```

---

### throwExceptionIfNoHandlerFound — Enable 404 as Exception

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureDefaultServletHandling(
            DefaultServletHandlerConfigurer configurer) {
        // DO NOT enable default servlet handler
        // If enabled, 404s go to container — not to Spring exception handlers
    }
}

// In WebInitializer:
@Override
protected void customizeRegistration(ServletRegistration.Dynamic registration) {
    registration.setInitParameter(
        "throwExceptionIfNoHandlerFound", "true");
}

// Now this works:
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(NoHandlerFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    @ResponseBody
    public ErrorResponse handleNotFound(NoHandlerFoundException ex) {
        return new ErrorResponse(404, "Resource not found: " + ex.getRequestURL());
    }
}
```

---

### Accessing WebApplicationContext from Request

```java
// DispatcherServlet sets this attribute on every request
@GetMapping("/debug")
public String debug(HttpServletRequest request) {
    
    WebApplicationContext wac = (WebApplicationContext) request.getAttribute(
        DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE);
    
    // Can inspect the context — useful in tests/debug
    String[] beanNames = wac.getBeanDefinitionNames();
    return "debug";
}
```

---

### Custom HandlerAdapter — For Legacy Controllers

```java
// Supporting a legacy interface-based controller pattern
public interface LegacyController {
    String handleRequest(HttpServletRequest req, HttpServletResponse resp);
}

// Custom adapter that supports LegacyController
@Component
public class LegacyControllerAdapter implements HandlerAdapter {

    @Override
    public boolean supports(Object handler) {
        return handler instanceof LegacyController;
    }

    @Override
    public ModelAndView handle(HttpServletRequest request,
                               HttpServletResponse response,
                               Object handler) throws Exception {
        String viewName = ((LegacyController) handler)
            .handleRequest(request, response);
        return new ModelAndView(viewName);
    }

    @Override
    public long getLastModified(HttpServletRequest request, Object handler) {
        return -1; // No caching support
    }
}
```

---

### Observing Request Lifecycle via Interceptor

```java
@Component
public class TimingInterceptor implements HandlerInterceptor {

    private static final String START_TIME = "requestStartTime";

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) {
        // STEP 12 in lifecycle — before handler execution
        request.setAttribute(START_TIME, System.currentTimeMillis());
        System.out.println("preHandle: " + request.getRequestURI());
        return true; // Continue processing
    }

    @Override
    public void postHandle(HttpServletRequest request,
                           HttpServletResponse response,
                           Object handler,
                           ModelAndView modelAndView) {
        // STEP 15 — after handler, before view rendering
        // modelAndView is null if @ResponseBody was used
        System.out.println("postHandle — view: " + 
            (modelAndView != null ? modelAndView.getViewName() : "none (ResponseBody)"));
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler,
                                Exception ex) {
        // STEP 18 — after complete request including view rendering
        // Called EVEN IF exception occurred
        long duration = System.currentTimeMillis() - 
            (Long) request.getAttribute(START_TIME);
        System.out.println("afterCompletion: " + duration + "ms, ex=" + ex);
    }
}
```

---

### Edge Case — postHandle NOT Called on Exception

```java
@Controller
public class EdgeCaseController {

    @GetMapping("/fail")
    public String fail() {
        throw new RuntimeException("Something went wrong");
        // postHandle() is NOT called
        // afterCompletion() IS called (with the exception)
        // processDispatchResult() handles the exception
    }
}

// Execution order for /fail:
// preHandle() → handler throws exception → 
// postHandle() SKIPPED →
// processDispatchResult() → HandlerExceptionResolvers →
// afterCompletion() (exception passed as parameter)
```

---

### Edge Case — postHandle NOT Called for @ResponseBody

```java
@RestController
public class RestEdgeCase {

    @GetMapping("/data")
    public Product getData() {
        return new Product("Widget", 9.99);
        // Jackson writes JSON to response during ha.handle()
        // When postHandle() is called, response is ALREADY WRITTEN
        // modelAndView parameter in postHandle() will be NULL
        // You cannot modify the response in postHandle() for REST endpoints
    }
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
In what order does `DispatcherServlet` consult `HandlerMapping` implementations to find a handler?

A) Alphabetical order by class name
B) Random order — all are checked and best match selected
C) Ordered by the `Ordered` interface or `@Order` annotation — lowest value first
D) Registration order in the ApplicationContext

**Answer: C**
`HandlerMapping` beans are sorted by `AnnotationAwareOrderComparator`. Lowest order value = highest priority = checked first. First mapping that returns a non-null `HandlerExecutionChain` wins. This is why `RequestMappingHandlerMapping` (order=0) takes priority over `BeanNameUrlHandlerMapping` (order=2) by default.

---

**Q2 — Select All That Apply**
Which of the following correctly describe `HandlerExecutionChain`?

A) Contains the handler (controller method) and applicable interceptors
B) Created once at startup and reused for all requests
C) Created per request
D) Returned by HandlerMapping.getHandler()
E) Contains the ModelAndView to be rendered

**Answer: A, C, D**
B is wrong — it's created per request (handler is looked up each time). E is wrong — ModelAndView is a separate object created by HandlerAdapter after execution.

---

**Q3 — Scenario**
`DispatcherServlet` is mapped to `/`. A request comes in for `/images/logo.png`. No handler is found. `throwExceptionIfNoHandlerFound` is `false` (default). What happens?

A) Spring returns HTTP 404 via `@ExceptionHandler`
B) Spring logs a warning and returns empty 200 response
C) `response.sendError(404)` is called — container handles the 404
D) Spring throws `NoHandlerFoundException` which propagates to container

**Answer: C**
With default settings, `noHandlerFound()` calls `response.sendError(404)`. The Servlet container then handles the error — typically showing its own 404 page or delegating to an error page configured in web.xml. Spring's exception handling chain is NOT invoked.

---

**Q4 — Ordering (Drag and Drop)**
Order these steps in `doDispatch()` execution:

- `postHandle()` interceptors called
- `afterCompletion()` interceptors called
- Handler found via `getHandler()`
- `preHandle()` interceptors called
- HandlerAdapter invokes handler method
- `processDispatchResult()` — view rendered or exception handled

**Correct Order:**
1. Handler found via `getHandler()`
2. `preHandle()` interceptors called
3. HandlerAdapter invokes handler method
4. `postHandle()` interceptors called
5. `processDispatchResult()` — view rendered or exception handled
6. `afterCompletion()` interceptors called

---

**Q5 — Code Prediction**
```java
public class MyInterceptor implements HandlerInterceptor {
    public boolean preHandle(...) {
        response.sendRedirect("/login");
        return false;
    }
    public void postHandle(...) { System.out.println("postHandle"); }
    public void afterCompletion(...) { System.out.println("afterCompletion"); }
}
```
What is printed when this interceptor's `preHandle` runs?

A) "postHandle" then "afterCompletion"
B) "afterCompletion" only
C) Nothing
D) "postHandle" only

**Answer: C**
When `preHandle()` returns `false`, `DispatcherServlet` stops processing immediately. `postHandle()` is NOT called. `afterCompletion()` is only called for interceptors that already successfully ran `preHandle()` — if this is the first interceptor, nothing further runs. No output.

---

**Q6 — True/False**
`DispatcherServlet.initStrategies()` is called on every HTTP request to re-initialize the strategy beans.

**Answer: False**
`initStrategies()` is called ONCE during `DispatcherServlet.init()` at startup. The strategy beans (HandlerMappings, HandlerAdapters, etc.) are stored as instance fields and reused for every request. This is what makes Spring MVC performant — zero initialization cost per request.

---

**Q7 — Tricky MCQ**
A `@RestController` method returns a `Product` object. At what point in `doDispatch()` is the JSON written to `HttpServletResponse`?

A) During `processDispatchResult()` when `render()` is called
B) During `postHandle()` by the interceptor
C) During `ha.handle()` — inside `RequestMappingHandlerAdapter`
D) After `afterCompletion()` by the MessageConverter

**Answer: C**
For `@ResponseBody`, `RequestMappingHandlerAdapter` processes the return value via `HttpMessageConverter` and writes directly to the response **during** `ha.handle()`. By the time `postHandle()` is called, the response is already written. This is why `postHandle()` cannot modify the response for REST controllers, and `modelAndView` is `null` in `postHandle()` for `@ResponseBody` methods.

---

**Q8 — MCQ**
What happens if no `HandlerAdapter` supports the handler returned by `HandlerMapping`?

A) DispatcherServlet returns HTTP 400
B) DispatcherServlet skips to the next HandlerMapping
C) `ServletException` is thrown: "No adapter for handler"
D) The handler is invoked directly via reflection

**Answer: C**
`getHandlerAdapter()` iterates all adapters and calls `supports()`. If none match, it throws `ServletException("No adapter for handler [" + handler + "]")`. This is an application configuration error — it means a handler type was registered without a corresponding adapter.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — postHandle() and @ResponseBody**
`postHandle()` is called AFTER handler execution but BEFORE view rendering. For `@ResponseBody` endpoints, the response is written during handler execution — so `postHandle()` receives a `null` ModelAndView and cannot modify the response. Many candidates assume interceptors can modify REST responses in `postHandle()` — they cannot.

**Trap 2 — afterCompletion() always runs (almost)**
`afterCompletion()` runs even when exceptions occur. BUT — it only runs for interceptors whose `preHandle()` returned `true`. If `preHandle()` of interceptor #2 returns `false`, interceptors #3+ never have `afterCompletion()` called. Only interceptors #1 and the failed #2? No — only those that successfully completed `preHandle()`. This is a precise rule examiners test.

**Trap 3 — Default 404 handling**
The default `throwExceptionIfNoHandlerFound=false` means `@ExceptionHandler(NoHandlerFoundException.class)` NEVER fires for 404s. This surprises many developers who expect global exception handlers to catch all errors. You must explicitly enable `throwExceptionIfNoHandlerFound=true`.

**Trap 4 — Multiple DispatcherServlets share root context**
Each DispatcherServlet has its OWN WebApplicationContext but shares the SAME root ApplicationContext. This means `@Service` beans are shared, but `@Controller` beans are per-DispatcherServlet. If a controller is accidentally defined in root context, it may not be visible to the DispatcherServlet's WebApplicationContext.

**Trap 5 — initStrategies ordering**
`initHandlerMappings()` collects ALL `HandlerMapping` beans and sorts them. If you define a custom `HandlerMapping` without setting its order, it gets `Integer.MAX_VALUE` (lowest priority) by default and is checked last. Missing `setOrder()` on custom strategies is a common misconfiguration.

**Trap 6 — DispatcherServlet.properties fallback**
When `@EnableWebMvc` is NOT used AND no explicit beans are defined, Spring falls back to `DispatcherServlet.properties`. When `@EnableWebMvc` IS used, `DelegatingWebMvcConfiguration` registers a completely different, more comprehensive set of defaults. The two sets of defaults are NOT the same. Examiners test behavior differences between `@EnableWebMvc` present vs absent.

---

## 5️⃣ SUMMARY SHEET

```
DISPATCHERSERVLET ROLE
─────────────────────────────────────────────────────
Pattern     : Front Controller — single entry point
Responsibility : Orchestration only — delegates to special beans
Extends     : HttpServlet (via HttpServletBean → FrameworkServlet)
Mapped to   : Usually "/" (all requests) or specific patterns

SPECIAL BEANS (Initialized Once at Startup)
─────────────────────────────────────────────────────
HandlerMapping              → Maps URL to Handler
HandlerAdapter              → Invokes Handler
HandlerExceptionResolver    → Handles Exceptions
ViewResolver                → Resolves View Names
LocaleResolver              → Detects Locale
ThemeResolver               → Detects Theme
MultipartResolver           → Parses File Uploads
FlashMapManager             → Manages Redirect Flash Data

initStrategies() ORDER
─────────────────────────────────────────────────────
1. MultipartResolver      5. HandlerAdapters
2. LocaleResolver         6. HandlerExceptionResolvers
3. ThemeResolver          7. RequestToViewNameTranslator
4. HandlerMappings        8. ViewResolvers
                          9. FlashMapManager

doDispatch() EXECUTION ORDER
─────────────────────────────────────────────────────
1.  checkMultipart()
2.  getHandler() → HandlerExecutionChain
3.  getHandlerAdapter()
4.  preHandle() interceptors [stop if false returned]
5.  ha.handle() → ModelAndView (or response written for @ResponseBody)
6.  applyDefaultViewName()
7.  postHandle() interceptors [SKIPPED if exception or @ResponseBody wrote response? NO — postHandle always called unless exception]
8.  processDispatchResult() → render() or exception handling
9.  afterCompletion() [ALWAYS — for interceptors whose preHandle succeeded]
10. ThreadLocals cleared

CRITICAL BEHAVIORAL RULES
─────────────────────────────────────────────────────
• preHandle() returns false   → processing stops, postHandle SKIPPED,
                                afterCompletion called for already-run interceptors only
• @ResponseBody               → response written in ha.handle(),
                                postHandle gets null ModelAndView
• No handler found (default)  → response.sendError(404), bypasses @ExceptionHandler
• No handler found (configured) → NoHandlerFoundException → @ExceptionHandler CAN catch
• Exception in handler        → postHandle SKIPPED, afterCompletion still runs

404 HANDLING TRAP
─────────────────────────────────────────────────────
Default: throwExceptionIfNoHandlerFound=false
  → sendError(404) → container handles → @ExceptionHandler NOT called

Fix: throwExceptionIfNoHandlerFound=true + disable default servlet handler
  → NoHandlerFoundException thrown → @ExceptionHandler CAN handle

THREADLOCALS SET PER REQUEST (cleared in finally)
─────────────────────────────────────────────────────
RequestContextHolder    → stores ServletRequestAttributes
LocaleContextHolder     → stores LocaleContext
Both cleared in finally block — no leak in thread pools

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "DispatcherServlet orchestrates — it never processes requests itself"
• "initStrategies() runs once at startup — zero per-request init cost"
• "postHandle() gets null ModelAndView for @ResponseBody endpoints"
• "afterCompletion() always runs for successful preHandle() interceptors"
• "Default 404 bypasses Spring exception handling — needs throwExceptionIfNoHandlerFound=true"
• "Each DispatcherServlet has own WebApplicationContext but shares root context"
• "HandlerExecutionChain = handler + interceptors — created per request"
• "getHandler() returns first match — HandlerMappings are ordered"
```

---

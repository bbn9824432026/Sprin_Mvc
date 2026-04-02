# TOPIC 2.1 — DISPATCHERSERVLET LIFECYCLE: init() TO destroy()

---

## 1️⃣ CONCEPTUAL EXPLANATION

### Why the Lifecycle Matters at This Depth

Most developers know DispatcherServlet "starts up and handles requests." Certification and senior interviews go far deeper — they test the **exact method call chain**, what happens at each phase, what can go wrong, how Spring Boot changes the lifecycle, and what the precise threading model is during each phase. This topic covers every phase with surgical precision.

---

### The Inheritance Chain — Lifecycle Methods Distributed Across Classes

The lifecycle is not in one class. It is **distributed** across the four-class inheritance hierarchy:

```
HttpServletBean
    ├── init(ServletConfig)          ← Entry point from Servlet container
    ├── initBeanWrapper()            ← Maps init-params to bean properties
    └── initServletBean()            ← Abstract — delegates up

FrameworkServlet
    ├── initServletBean()            ← Creates/retrieves WebApplicationContext
    ├── initWebApplicationContext()  ← Core context wiring
    ├── onRefresh()                  ← Called after context refresh
    ├── processRequest()             ← Per-request entry point
    ├── doGet/doPost/etc.            ← Overrides HttpServlet methods
    └── destroy()                    ← Closes WebApplicationContext

DispatcherServlet
    ├── onRefresh()                  ← Calls initStrategies()
    ├── initStrategies()             ← Initialises all special beans
    ├── doService()                  ← Sets request attributes
    └── doDispatch()                 ← Core dispatching logic
```

Understanding which class owns which method is a **direct exam question**.

---

### Phase 1 — Instantiation

The Servlet container (Tomcat) instantiates `DispatcherServlet` using **reflection** — calling the no-arg constructor.

```java
// DispatcherServlet has two constructors:

// 1. No-arg — used by Servlet container (web.xml / programmatic registration)
public DispatcherServlet() {
    super();
    // Sets detectAllHandlerMappings = true (default)
    // Sets detectAllHandlerAdapters = true (default)
    // Sets detectAllViewResolvers = true (default)
    setDispatchOptionsRequest(true);
}

// 2. Context-accepting — used by Spring Boot
public DispatcherServlet(WebApplicationContext webApplicationContext) {
    super(webApplicationContext);
    // Receives pre-built WebApplicationContext from Spring Boot
    // Context is NOT refreshed here — just stored
    setDispatchOptionsRequest(true);
}
```

**In Spring Boot:** `DispatcherServletAutoConfiguration` creates `DispatcherServlet` as a Spring bean using constructor 2, passing the existing `ApplicationContext`. This is why Boot doesn't need a separate child context — the same context is passed in.

**In plain Spring MVC:** Container calls constructor 1. No context yet. Context creation happens in `init()`.

---

### Phase 2 — init(ServletConfig) — HttpServletBean

```java
// HttpServletBean.init() — first Spring method called after instantiation
@Override
public final void init() throws ServletException {

    // STEP 1: Read all <init-param> entries from web.xml
    PropertyValues pvs = new ServletConfigPropertyValues(
        getServletConfig(),  // Access to init-params
        this.requiredProperties  // Params that MUST be present
    );

    // STEP 2: Wrap DispatcherServlet in a BeanWrapper
    BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
    ResourceLoader resourceLoader = 
        new ServletContextResourceLoader(getServletContext());
    bw.registerCustomEditor(Resource.class, 
        new ResourceEditor(resourceLoader, getEnvironment()));

    // STEP 3: Apply init-params as bean properties
    // This is how contextConfigLocation init-param becomes
    // DispatcherServlet.setContextConfigLocation() call
    initBeanWrapper(bw);
    bw.setPropertyValues(pvs, true);
    // After this: all <init-param> values are set on DispatcherServlet

    // STEP 4: Delegate to subclass
    initServletBean();
}
```

**Key insight:** `HttpServletBean.init()` uses Spring's `BeanWrapper` to map servlet init-params to Java bean properties. This means every `<init-param>` in web.xml becomes a setter call on `DispatcherServlet`. For example:

```xml
<init-param>
    <param-name>throwExceptionIfNoHandlerFound</param-name>
    <param-value>true</param-value>
</init-param>
```

Maps to: `dispatcherServlet.setThrowExceptionIfNoHandlerFound(true)`

---

### Phase 3 — initServletBean() — FrameworkServlet

```java
// FrameworkServlet.initServletBean()
@Override
protected final void initServletBean() throws ServletException {

    getServletContext().log("Initializing Spring " + 
        getClass().getSimpleName() + " '" + getServletName() + "'");

    long startTime = System.currentTimeMillis();

    try {
        // THE critical call — creates or retrieves WebApplicationContext
        this.webApplicationContext = initWebApplicationContext();

        // Hook for subclasses (DispatcherServlet doesn't use this)
        initFrameworkServlet();

    } catch (ServletException | RuntimeException ex) {
        logger.error("Context initialization failed", ex);
        throw ex;
        // If this throws, Servlet container marks servlet as UNAVAILABLE
        // All subsequent requests to this servlet: HTTP 503
    }

    long elapsed = System.currentTimeMillis() - startTime;
    getServletContext().log("Completed initialization in " + elapsed + " ms");
}
```

**Critical:** If `initWebApplicationContext()` throws any exception, `DispatcherServlet` is permanently unavailable. The container marks it as failed. Subsequent requests receive HTTP 503 (Service Unavailable) or HTTP 500 depending on the container.

---

### Phase 4 — initWebApplicationContext() — FrameworkServlet

This is the most complex initialisation method:

```java
protected WebApplicationContext initWebApplicationContext() {

    // STEP 1: Find root context (from ContextLoaderListener)
    WebApplicationContext rootContext =
        WebApplicationContextUtils.getWebApplicationContext(
            getServletContext()
        );
    // Returns null if no ContextLoaderListener — that's OK

    WebApplicationContext wac = null;

    // STEP 2: Context passed via constructor? (Spring Boot path)
    if (this.webApplicationContext != null) {
        wac = this.webApplicationContext;
        if (wac instanceof ConfigurableWebApplicationContext cwac) {
            if (!cwac.isActive()) {
                // Context not yet refreshed
                if (cwac.getParent() == null) {
                    cwac.setParent(rootContext);
                }
                configureAndRefreshWebApplicationContext(cwac);
                // Refreshes the context — instantiates beans
            }
            // If already active (Spring Boot) — skip refresh
        }
    }

    // STEP 3: Look for context in ServletContext by attribute name
    // (Used when sharing context between multiple DispatcherServlets)
    if (wac == null) {
        wac = findWebApplicationContext();
        // Looks for contextAttribute init-param
    }

    // STEP 4: Create new context (plain Spring MVC path)
    if (wac == null) {
        wac = createWebApplicationContext(rootContext);
        // Creates AnnotationConfigWebApplicationContext or XmlWebApplicationContext
        // Sets rootContext as parent
        // Calls configureAndRefreshWebApplicationContext()
        // → context.refresh() → all beans instantiated
    }

    // STEP 5: Handle case where context was pre-refreshed
    // (Spring Boot passes pre-built, already-refreshed context)
    if (!this.refreshEventReceived) {
        // If context was not refreshed here, initStrategies manually
        synchronized (this.onRefreshMonitor) {
            onRefresh(wac);
        }
    }
    // If context was refreshed here, ContextRefreshedEvent triggers onRefresh()

    // STEP 6: Publish context to ServletContext
    if (this.publishContext) {
        String attrName = getServletContextAttributeName();
        getServletContext().setAttribute(attrName, wac);
    }

    return wac;
}
```

---

### Phase 5 — onRefresh() and initStrategies() — DispatcherServlet

`onRefresh()` is called either:
- Via `ContextRefreshedEvent` listener (when context refreshed during init)
- Directly from `initWebApplicationContext()` (when context pre-refreshed)

```java
// DispatcherServlet.onRefresh()
@Override
protected void onRefresh(ApplicationContext context) {
    initStrategies(context);
}

// DispatcherServlet.initStrategies() — full method
protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);
    initLocaleResolver(context);
    initThemeResolver(context);
    initHandlerMappings(context);
    initHandlerAdapters(context);
    initHandlerExceptionResolvers(context);
    initRequestToViewNameTranslator(context);
    initViewResolvers(context);
    initFlashMapManager(context);
}
```

Each `initX()` follows this exact pattern:

```java
private void initHandlerMappings(ApplicationContext context) {

    this.handlerMappings = null;

    if (this.detectAllHandlerMappings) {
        // Find ALL HandlerMapping beans in context AND ancestors
        Map<String, HandlerMapping> matchingBeans =
            BeanFactoryUtils.beansOfTypeIncludingAncestors(
                context,
                HandlerMapping.class,
                true,   // include non-singletons
                false   // don't allow eager init
            );

        if (!matchingBeans.isEmpty()) {
            this.handlerMappings = new ArrayList<>(matchingBeans.values());
            // Sort by order — lowest value = highest priority
            AnnotationAwareOrderComparator.sort(this.handlerMappings);
        }
    } else {
        // detectAllHandlerMappings=false — only look for bean named "handlerMapping"
        try {
            HandlerMapping hm = context.getBean(
                "handlerMapping", HandlerMapping.class);
            this.handlerMappings = Collections.singletonList(hm);
        } catch (NoSuchBeanDefinitionException ex) {
            // ignored
        }
    }

    // Fallback to default from DispatcherServlet.properties
    if (this.handlerMappings == null) {
        this.handlerMappings = getDefaultStrategies(
            context, HandlerMapping.class);
    }
}
```

**`detectAllHandlerMappings=true` (default):** Collects ALL `HandlerMapping` beans from context and ancestors (root context too). This is why you can define `HandlerMapping` beans in either root or child context.

**`detectAllHandlerMappings=false`:** Only looks for a bean named exactly `"handlerMapping"`. Used for very controlled configurations.

---

### Phase 6 — Request Processing — FrameworkServlet

Every HTTP method (GET, POST, PUT, DELETE, PATCH, OPTIONS, TRACE) is overridden in `FrameworkServlet` to funnel into one method:

```java
// FrameworkServlet overrides ALL HTTP methods:
@Override
protected final void doGet(HttpServletRequest request, 
                            HttpServletResponse response)
        throws ServletException, IOException {
    processRequest(request, response);
}

@Override
protected final void doPost(HttpServletRequest request,
                             HttpServletResponse response)
        throws ServletException, IOException {
    processRequest(request, response);
}
// Same for doPut, doDelete, doOptions, doTrace, doPatch

// The single funnel:
protected final void processRequest(HttpServletRequest request,
                                     HttpServletResponse response)
        throws ServletException, IOException {

    long startTime = System.currentTimeMillis();
    Throwable failureCause = null;

    // STEP 1: Save current ThreadLocal state
    // (Important for nested dispatch scenarios)
    LocaleContext previousLocaleContext = 
        LocaleContextHolder.getLocaleContext();
    RequestAttributes previousAttributes = 
        RequestContextHolder.getRequestAttributes();

    // STEP 2: Build new ThreadLocal state for this request
    LocaleContext localeContext = buildLocaleContext(request);
    ServletRequestAttributes requestAttributes = 
        buildRequestAttributes(request, response, previousAttributes);

    // STEP 3: Set async manager on request
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
    asyncManager.registerCallableInterceptor(
        FrameworkServlet.class.getName(),
        new RequestBindingInterceptor()
    );

    // STEP 4: Initialize ThreadLocals
    initContextHolders(request, localeContext, requestAttributes);

    try {
        // STEP 5: Delegate to DispatcherServlet.doService()
        doService(request, response);

    } catch (ServletException | IOException ex) {
        failureCause = ex;
        throw ex;
    } catch (Throwable ex) {
        failureCause = ex;
        throw new NestedServletException("Request processing failed", ex);
    } finally {
        // STEP 6: ALWAYS restore previous ThreadLocal state
        resetContextHolders(request, previousLocaleContext, previousAttributes);

        // STEP 7: Notify request attributes (triggers @RequestScope destruction)
        if (requestAttributes != null) {
            requestAttributes.requestCompleted();
        }

        // STEP 8: Log request completion
        logResult(request, response, failureCause, asyncManager);

        // STEP 9: Publish ServletRequestHandledEvent
        publishRequestHandledEvent(request, response, startTime, failureCause);
    }
}
```

**Critical details:**
1. `previousLocaleContext` and `previousAttributes` are saved — supports **nested dispatches** (INCLUDE, FORWARD) where a second `processRequest` call happens within the first
2. `requestAttributes.requestCompleted()` in `finally` triggers destruction of `@RequestScope` beans — this is how request-scoped beans are cleaned up
3. `publishRequestHandledEvent` fires `ServletRequestHandledEvent` — used for metrics, auditing

---

### Phase 7 — doService() — DispatcherServlet

```java
// DispatcherServlet.doService()
@Override
protected void doService(HttpServletRequest request,
                          HttpServletResponse response) throws Exception {

    logRequest(request);

    // STEP 1: Save attributes snapshot for INCLUDE dispatches
    // (If this is a forward/include, existing attributes are preserved)
    Map<String, Object> attributesSnapshot = null;
    if (WebUtils.isIncludeRequest(request)) {
        attributesSnapshot = new HashMap<>();
        Enumeration<?> attrNames = request.getAttributeNames();
        while (attrNames.hasMoreElements()) {
            String attrName = (String) attrNames.nextElement();
            if (this.cleanupAfterInclude || 
                attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
                attributesSnapshot.put(attrName, 
                    request.getAttribute(attrName));
            }
        }
    }

    // STEP 2: Set DispatcherServlet-specific request attributes
    // These are available to controllers, interceptors, views
    request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, 
        getWebApplicationContext());
    request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, 
        this.localeResolver);
    request.setAttribute(THEME_RESOLVER_ATTRIBUTE, 
        this.themeResolver);
    request.setAttribute(THEME_SOURCE_ATTRIBUTE, 
        getThemeSource());

    // STEP 3: Handle flash map (redirect attributes)
    if (this.flashMapManager != null) {
        FlashMap inputFlashMap = this.flashMapManager
            .retrieveAndUpdate(request, response);
        if (inputFlashMap != null) {
            request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, 
                Collections.unmodifiableMap(inputFlashMap));
        }
        request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, 
            new FlashMap());
        request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, 
            this.flashMapManager);
    }

    // STEP 4: Set request path (parsed once, reused)
    RequestPath previousRequestPath = null;
    if (this.parseRequestPath) {
        previousRequestPath = (RequestPath) request.getAttribute(
            ServletRequestPathUtils.PATH_ATTRIBUTE);
        ServletRequestPathUtils.parseAndCache(request);
    }

    try {
        // STEP 5: THE core dispatch call
        doDispatch(request, response);
    } finally {
        // STEP 6: Restore attributes for INCLUDE dispatches
        if (attributesSnapshot != null) {
            restoreAttributesAfterInclude(request, attributesSnapshot);
        }
        // STEP 7: Restore previous request path
        if (this.parseRequestPath) {
            ServletRequestPathUtils.setParsedRequestPath(
                previousRequestPath, request);
        }
    }
}
```

**Key attributes set by doService():**

| Attribute Key | Value | Purpose |
|---|---|---|
| `WEB_APPLICATION_CONTEXT_ATTRIBUTE` | `WebApplicationContext` | Available to controllers |
| `LOCALE_RESOLVER_ATTRIBUTE` | `LocaleResolver` | Used by view rendering |
| `THEME_RESOLVER_ATTRIBUTE` | `ThemeResolver` | Used by themed views |
| `INPUT_FLASH_MAP_ATTRIBUTE` | Incoming flash data | From previous redirect |
| `OUTPUT_FLASH_MAP_ATTRIBUTE` | Empty FlashMap | Populated by controllers |

---

### Phase 8 — destroy() — FrameworkServlet

```java
// FrameworkServlet.destroy()
@Override
public void destroy() {
    getServletContext().log("Destroying Spring FrameworkServlet '" + 
        getServletName() + "'");

    // Close WebApplicationContext if it was created here
    // (Not closed if context was passed in via constructor — Spring Boot manages it)
    if (this.webApplicationContext instanceof ConfigurableApplicationContext cac
            && !this.webApplicationContextInjected) {
        cac.close();
        // Triggers: @PreDestroy methods
        //           DisposableBean.destroy()
        //           destroyMethod from @Bean
        //           All singleton beans destroyed in reverse creation order
    }
}
```

**`webApplicationContextInjected` flag:**
- `true` when context was passed via constructor (Spring Boot)
- `false` when context was created during init() (plain Spring MVC)

In Spring Boot, `DispatcherServlet.destroy()` does NOT close the `ApplicationContext` — Spring Boot's `SpringApplication` owns the context and closes it during JVM shutdown.

In plain Spring MVC, `destroy()` closes the child `WebApplicationContext`. The root context is closed by `ContextLoaderListener.contextDestroyed()` separately.

---

### ThreadLocal Setup and Teardown — Memory Leak Prevention

This is architecturally critical:

```
Per-Request ThreadLocal Management:
─────────────────────────────────────────────────────────────
Thread from Tomcat pool assigned to request
        │
processRequest() called
        │
├── Save previous state:
│   previousLocaleContext = LocaleContextHolder.getLocaleContext()
│   previousAttributes = RequestContextHolder.getRequestAttributes()
│
├── Set new state:
│   LocaleContextHolder.setLocaleContext(new SimpleLocaleContext(locale))
│   RequestContextHolder.setRequestAttributes(new ServletRequestAttributes(req, res))
│
├── doService() → doDispatch() → your controller code
│   (ThreadLocals accessible anywhere in call stack)
│
└── finally block — ALWAYS executes:
    LocaleContextHolder.setLocaleContext(previousLocaleContext)
    RequestContextHolder.setRequestAttributes(previousAttributes)
    requestAttributes.requestCompleted()
    // @RequestScope beans destroyed here

Thread returned to Tomcat pool — ThreadLocals CLEAN
```

**Why this matters:** Thread pool threads are reused. If ThreadLocals are not cleaned, the next request on the same thread sees stale data from the previous request. Spring's `finally` block guarantees cleanup.

---

### Spring Boot Lifecycle Differences

In Spring Boot, the lifecycle is fundamentally different:

```
PLAIN SPRING MVC LIFECYCLE:
─────────────────────────────────────────────────────────
Tomcat starts → Finds DispatcherServlet in web.xml
→ Instantiates via no-arg constructor
→ Calls init(ServletConfig)
→ HttpServletBean reads init-params
→ FrameworkServlet creates WebApplicationContext
→ context.refresh() — beans created
→ initStrategies()
→ Ready

SPRING BOOT LIFECYCLE:
─────────────────────────────────────────────────────────
SpringApplication.run()
→ Creates ApplicationContext (includes DispatcherServlet as bean)
→ context.refresh() — ALL beans created (including DispatcherServlet)
→ EmbeddedWebServerFactory creates Tomcat
→ DispatcherServletRegistrationBean registers DispatcherServlet with Tomcat
→ Tomcat starts, calls DispatcherServlet.init(ServletConfig)
→ HttpServletBean reads init-params (from DispatcherServletRegistrationBean)
→ FrameworkServlet.initWebApplicationContext()
   → Finds pre-built context (passed via constructor)
   → Context already refreshed — skips refresh
   → Calls onRefresh() → initStrategies()
→ Ready

KEY DIFFERENCE: In Boot, context.refresh() happens BEFORE Tomcat starts.
In plain MVC, context.refresh() happens AFTER Tomcat starts (during init()).
```

---

### DispatcherServlet Configuration Properties

All configurable via init-params (web.xml) or `DispatcherServletRegistrationBean` (Boot):

| Property | Default | Effect |
|---|---|---|
| `contextClass` | `XmlWebApplicationContext` | Context implementation class |
| `contextConfigLocation` | `/WEB-INF/{name}-servlet.xml` | Config file location |
| `namespace` | `{servlet-name}-servlet` | Context namespace |
| `detectAllHandlerMappings` | `true` | Collect all or named-only |
| `detectAllHandlerAdapters` | `true` | Collect all or named-only |
| `detectAllViewResolvers` | `true` | Collect all or named-only |
| `throwExceptionIfNoHandlerFound` | `false` | 404 as exception |
| `cleanupAfterInclude` | `true` | Restore attrs after INCLUDE |
| `publishContext` | `true` | Store context in ServletContext |
| `publishEvents` | `true` | Fire ServletRequestHandledEvent |
| `dispatchOptionsRequest` | `true` | Handle OPTIONS requests |
| `dispatchTraceRequest` | `false` | Handle TRACE requests |

---

### Memory & Performance Implications

```
STARTUP PHASE:
─────────────────────────────────────────────────────────
context.refresh() cost: proportional to number of beans
initStrategies() cost:  O(n) bean lookups — n = number of special beans
HandlerMapping scan:    Reflects all @Controller/@RequestMapping — O(methods)
All of this: ONE-TIME cost at startup

PER-REQUEST PHASE:
─────────────────────────────────────────────────────────
ThreadLocal set/reset:  O(1) — negligible
Handler lookup:         O(1) map lookup — pre-built registry
HandlerAdapter.handle(): Reflection invoke — small overhead
Argument resolution:    Per-argument resolver chain walk — O(resolvers)
View resolution:        O(resolvers) chain walk
JSON serialization:     Dominant cost for REST — Jackson ObjectMapper (singleton)

MEMORY FOOTPRINT:
─────────────────────────────────────────────────────────
WebApplicationContext:  Heap — all singleton beans live here for app lifetime
Per-request objects:    ModelAndView, RequestAttributes — GC'd after response
ThreadLocals:           Per thread — 200 threads × small refs = negligible
```

---

## 2️⃣ CODE EXAMPLES

### Observing Lifecycle via Custom DispatcherServlet

```java
// Subclassing DispatcherServlet to observe lifecycle
public class ObservableDispatcherServlet extends DispatcherServlet {

    @Override
    protected void initStrategies(ApplicationContext context) {
        System.out.println("=== initStrategies() called ===");
        System.out.println("Context: " + context.getClass().getSimpleName());
        System.out.println("Bean count: " + context.getBeanDefinitionCount());
        super.initStrategies(context);
        System.out.println("=== initStrategies() complete ===");
    }

    @Override
    protected void doService(HttpServletRequest request,
                              HttpServletResponse response) throws Exception {
        System.out.println("=== doService() for: " + 
            request.getMethod() + " " + request.getRequestURI());
        super.doService(request, response);
    }

    @Override
    public void destroy() {
        System.out.println("=== DispatcherServlet destroying ===");
        super.destroy();
    }
}
```

---

### Custom HandlerMapping — Registered via detectAll

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    // DispatcherServlet's initHandlerMappings uses
    // BeanFactoryUtils.beansOfTypeIncludingAncestors(HandlerMapping.class)
    // — finds THIS bean automatically
    @Bean
    @Order(Ordered.HIGHEST_PRECEDENCE) // Checked FIRST
    public HandlerMapping healthCheckMapping() {
        SimpleUrlHandlerMapping mapping = new SimpleUrlHandlerMapping();
        mapping.setOrder(Ordered.HIGHEST_PRECEDENCE);
        Map<String, HttpRequestHandler> urlMap = new HashMap<>();
        urlMap.put("/health", (req, res) -> {
            res.setStatus(200);
            res.getWriter().write("{\"status\":\"UP\"}");
        });
        mapping.setUrlMap(urlMap);
        return mapping;
    }
}
```

---

### Accessing ThreadLocals Set by processRequest()

```java
@Service
public class RequestContextService {

    // Called from service layer — no HttpServletRequest parameter needed
    public String getCurrentRequestUri() {
        // RequestContextHolder is populated by FrameworkServlet.processRequest()
        RequestAttributes attrs = RequestContextHolder.getRequestAttributes();
        if (attrs instanceof ServletRequestAttributes sra) {
            return sra.getRequest().getRequestURI();
        }
        return "no-request-context";
    }

    public String getCurrentLocale() {
        // LocaleContextHolder is also set by processRequest()
        Locale locale = LocaleContextHolder.getLocale();
        return locale.toString();
    }

    // DANGER: Spawning new thread loses ThreadLocals
    public void asyncOperation() {
        Locale currentLocale = LocaleContextHolder.getLocale(); // capture now

        new Thread(() -> {
            // LocaleContextHolder.getLocale() here returns DEFAULT locale
            // ThreadLocal is not inherited by child threads
            // Must pass captured values explicitly
            System.out.println("In new thread, locale: " + currentLocale);
        }).start();
    }
}
```

---

### Listening to ServletRequestHandledEvent

```java
// processRequest() publishes this event after EVERY request
@Component
public class RequestMetricsListener 
    implements ApplicationListener<ServletRequestHandledEvent> {

    private final AtomicLong requestCount = new AtomicLong();
    private final AtomicLong errorCount = new AtomicLong();

    @Override
    public void onApplicationEvent(ServletRequestHandledEvent event) {
        requestCount.incrementAndGet();

        if (event.getFailureCause() != null) {
            errorCount.incrementAndGet();
            log.error("Request failed: {} {}, cause: {}",
                event.getMethod(),
                event.getRequestUrl(),
                event.getFailureCause().getMessage()
            );
        }

        log.debug("Request: {} {} completed in {}ms, status: {}",
            event.getMethod(),
            event.getRequestUrl(),
            event.getProcessingTimeMillis(),
            event.getStatusCode()
        );
    }
}
```

---

### Spring Boot — Customising DispatcherServlet Registration

```java
// Spring Boot — customise how DispatcherServlet is registered
@Configuration
public class DispatcherConfig {

    @Bean
    public DispatcherServletRegistrationBean dispatcherServletRegistration(
            DispatcherServlet dispatcherServlet) {

        DispatcherServletRegistrationBean registration =
            new DispatcherServletRegistrationBean(dispatcherServlet, "/");

        registration.setLoadOnStartup(1);
        registration.setName("dispatcher");

        // Equivalent of <init-param> in web.xml
        registration.addInitParameter(
            "throwExceptionIfNoHandlerFound", "true");
        registration.addInitParameter(
            "enableLoggingRequestDetails", "true");

        return registration;
    }
}
```

---

### Edge Case — detectAllHandlerMappings=false

```java
// Plain Spring MVC — limit HandlerMapping detection
public class SelectiveWebInitializer
    extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected void customizeRegistration(
            ServletRegistration.Dynamic registration) {
        // Only look for bean named exactly "handlerMapping"
        // Ignores all other HandlerMapping beans
        registration.setInitParameter(
            "detectAllHandlerMappings", "false");
    }
}

@Configuration
@EnableWebMvc
public class WebConfig {

    // With detectAllHandlerMappings=false, ONLY this bean is used
    // Must be named "handlerMapping" exactly
    @Bean(name = "handlerMapping")
    public HandlerMapping handlerMapping() {
        RequestMappingHandlerMapping mapping =
            new RequestMappingHandlerMapping();
        mapping.setOrder(0);
        return mapping;
    }

    // This bean is IGNORED — different name
    @Bean
    public HandlerMapping anotherMapping() {
        return new BeanNameUrlHandlerMapping();
    }
}
```

---

### Edge Case — destroy() and Context Ownership

```java
// Plain Spring MVC — DispatcherServlet OWNS and closes child context
public class PlainMvcServlet extends DispatcherServlet {
    // destroy() calls childContext.close()
    // All @PreDestroy methods on @Controller, @Component in child context called
}

// Spring Boot — DispatcherServlet does NOT own context
// context was injected (webApplicationContextInjected=true)
// destroy() skips close() — SpringApplication closes context on shutdown

// Demonstrating the flag:
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        ConfigurableApplicationContext ctx =
            SpringApplication.run(App.class, args);

        DispatcherServlet ds = ctx.getBean(DispatcherServlet.class);
        // ds.webApplicationContextInjected == true
        // ds.destroy() will NOT call ctx.close()
        // ctx.close() is called by SpringApplication shutdown hook
    }
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
In which class is the `init(ServletConfig)` method that Spring MVC overrides as the entry point for `DispatcherServlet` initialisation?

A) `DispatcherServlet`
B) `FrameworkServlet`
C) `HttpServletBean`
D) `AbstractApplicationContext`

**Answer: C**
`HttpServletBean` overrides `HttpServlet.init(ServletConfig)`. It reads init-params via `BeanWrapper`, then calls `initServletBean()`. `FrameworkServlet` implements `initServletBean()`. `DispatcherServlet` never directly overrides `init()`.

---

**Q2 — Select All That Apply**
Which of the following are set as `HttpServletRequest` attributes by `DispatcherServlet.doService()`?

A) `WebApplicationContext`
B) `LocaleResolver`
C) `HandlerMapping`
D) `FlashMap` (output)
E) `ThemeResolver`

**Answer: A, B, D, E**
`HandlerMapping` is NOT set as a request attribute — it's an instance field of `DispatcherServlet`. `WebApplicationContext`, `LocaleResolver`, `ThemeResolver`, and both input and output `FlashMap` are all set as request attributes by `doService()`.

---

**Q3 — Ordering**
Order these lifecycle phases from first to last:

- `initStrategies()` called
- `doService()` called (first request)
- `init(ServletConfig)` called by Tomcat
- `DispatcherServlet` instantiated via no-arg constructor
- `context.refresh()` — beans created
- `initWebApplicationContext()` called

**Correct Order:**
1. `DispatcherServlet` instantiated via no-arg constructor
2. `init(ServletConfig)` called by Tomcat
3. `initWebApplicationContext()` called
4. `context.refresh()` — beans created
5. `initStrategies()` called
6. `doService()` called (first request)

---

**Q4 — MCQ**
What happens to the child `WebApplicationContext` when `DispatcherServlet.destroy()` is called in Spring Boot?

A) The context is closed — all beans destroyed
B) The context is NOT closed — Spring Boot's `SpringApplication` manages context closure
C) The context is refreshed one final time before closure
D) The context is transferred to another servlet

**Answer: B**
In Spring Boot, `DispatcherServlet` is created with a pre-built context passed via constructor. `webApplicationContextInjected=true`. `destroy()` checks this flag and skips `context.close()`. The `ApplicationContext` is closed by `SpringApplication`'s JVM shutdown hook.

---

**Q5 — Code Prediction**
```java
@Service
public class MyService {
    public String getUri() {
        return ((ServletRequestAttributes)
            RequestContextHolder.currentRequestAttributes())
            .getRequest().getRequestURI();
    }
}

@Controller
public class MyController {
    @Autowired MyService myService;

    @GetMapping("/test")
    public void test() {
        new Thread(() -> {
            System.out.println(myService.getUri()); // Line X
        }).start();
    }
}
```
What happens at Line X?

A) Prints the current request URI correctly
B) Prints null
C) Throws `IllegalStateException` — no request bound to thread
D) Throws `NullPointerException`

**Answer: C**
`RequestContextHolder` uses a `ThreadLocal`. The new `Thread` is NOT the request thread — it has no `ThreadLocal` data. `RequestContextHolder.currentRequestAttributes()` throws `IllegalStateException: No thread-bound request found`. The ThreadLocal is set by `processRequest()` on the Servlet container thread only.

---

**Q6 — True/False**
`FrameworkServlet.processRequest()` publishes a `ServletRequestHandledEvent` even when the request results in an exception.

**Answer: True**
`publishRequestHandledEvent()` is called in the `finally` block of `processRequest()`. It always executes regardless of whether an exception occurred. The event carries the failure cause if an exception was thrown. This is used for metrics and monitoring.

---

**Q7 — MCQ**
With `detectAllHandlerMappings=true` (default), from where does `initHandlerMappings()` collect `HandlerMapping` beans?

A) Only from the child `WebApplicationContext`
B) Only from the root `ApplicationContext`
C) From the child context AND ancestors (including root context)
D) From `DispatcherServlet.properties` only

**Answer: C**
`initHandlerMappings()` uses `BeanFactoryUtils.beansOfTypeIncludingAncestors()` which searches the child context AND walks up the parent chain. `HandlerMapping` beans can be defined in either root or child context and will be found. This is the meaning of "including ancestors".

---

**Q8 — Scenario**
A `DispatcherServlet` is configured with `publishEvents=false`. A developer has an `ApplicationListener<ServletRequestHandledEvent>` registered. What happens?

A) The listener is called but receives empty events
B) `BeanCreationException` — listener cannot be registered
C) The listener is never called — no events published
D) The listener is called once at startup only

**Answer: C**
`publishEvents=false` disables `publishRequestHandledEvent()`. The `finally` block in `processRequest()` still runs but skips event publication. The listener receives no events. `publishEvents=true` is the default.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — "init() is in DispatcherServlet"**
`init(ServletConfig)` is in `HttpServletBean`. `initServletBean()` is in `FrameworkServlet`. `initStrategies()` is in `DispatcherServlet`. These are three different classes. Exam questions ask which class owns which lifecycle method. Confusing them costs marks.

**Trap 2 — Spring Boot context refresh timing**
In plain Spring MVC: Tomcat starts → `init()` → `context.refresh()` (context created AFTER Tomcat).
In Spring Boot: `context.refresh()` (including all beans) → THEN Tomcat starts.
This means in Spring Boot, ALL beans are ready before the first request is possible. In plain MVC, there's a window after Tomcat starts but before `DispatcherServlet.init()` completes where requests could arrive (though they'd get 503).

**Trap 3 — destroy() and context ownership**
In plain Spring MVC: `destroy()` closes the child context. In Spring Boot: `destroy()` does NOT close the context. The `webApplicationContextInjected` flag controls this. Getting this wrong leads to thinking Spring Boot causes double-close or premature close of the context.

**Trap 4 — ThreadLocal inheritance**
Child threads spawned from the request thread do NOT inherit `ThreadLocals`. `RequestContextHolder` and `LocaleContextHolder` are empty in child threads. This is a runtime bug — no compile error, no immediate exception. The exception only occurs when the child thread tries to access `RequestContextHolder.currentRequestAttributes()`.

**Trap 5 — requestCompleted() triggers @RequestScope destruction**
`requestAttributes.requestCompleted()` in the `finally` block is what triggers `@RequestScope` bean destruction. If this isn't called (impossible in normal flow, but relevant in tests or custom implementations), request-scoped beans leak. Understanding this links request scope lifecycle to the `processRequest()` finally block.

**Trap 6 — detectAllHandlerMappings=false**
With `detectAllHandlerMappings=false`, Spring looks for a bean named EXACTLY `"handlerMapping"`. All other `HandlerMapping` beans are silently ignored. A common misconfiguration: defining a `HandlerMapping` bean with a different name and wondering why it's never consulted.

**Trap 7 — OPTIONS and TRACE handling**
`dispatchOptionsRequest=true` (default since Spring 4.3) means OPTIONS requests go through `doDispatch()` and can be handled by `@RequestMapping`. `dispatchTraceRequest=false` (default) means TRACE requests are handled by `HttpServlet.doTrace()` directly — not by Spring MVC. Many assume all HTTP methods are dispatched equally.

---

## 5️⃣ SUMMARY SHEET

```
LIFECYCLE METHOD OWNERSHIP
─────────────────────────────────────────────────────
HttpServletBean:
  init(ServletConfig)        ← Entry from Servlet container
  initBeanWrapper()          ← Registers custom editors
  [reads init-params via BeanWrapper → sets bean properties]

FrameworkServlet:
  initServletBean()          ← Called by HttpServletBean.init()
  initWebApplicationContext() ← Creates/retrieves child WebApplicationContext
  onRefresh()                ← Called after context refresh (abstract in FS)
  processRequest()           ← ALL HTTP methods funnel here
  doGet/doPost/...           ← Override HttpServlet → call processRequest()
  destroy()                  ← Closes child context (if owned)

DispatcherServlet:
  onRefresh()                ← Calls initStrategies()
  initStrategies()           ← Initialises all special beans
  doService()                ← Sets request attributes → calls doDispatch()
  doDispatch()               ← Core dispatching logic

INIT SEQUENCE (Plain Spring MVC)
─────────────────────────────────────────────────────
1.  Tomcat instantiates DispatcherServlet (no-arg constructor)
2.  Tomcat calls init(ServletConfig)
3.  HttpServletBean reads init-params via BeanWrapper
4.  initServletBean() called
5.  initWebApplicationContext() — creates child context
6.  context.refresh() — instantiates all @Controller, ViewResolver, etc. beans
7.  ContextRefreshedEvent triggers onRefresh()
8.  initStrategies() — HandlerMappings, HandlerAdapters etc. stored as fields
9.  Child context stored in ServletContext
10. DispatcherServlet READY

INIT SEQUENCE (Spring Boot)
─────────────────────────────────────────────────────
1.  SpringApplication creates single ApplicationContext
2.  context.refresh() — ALL beans created (incl. DispatcherServlet bean)
3.  EmbeddedWebServerFactory creates Tomcat
4.  DispatcherServletRegistrationBean registers DS with Tomcat
5.  Tomcat starts
6.  Tomcat calls DispatcherServlet.init(ServletConfig)
7.  initWebApplicationContext() finds pre-built context (no refresh)
8.  initStrategies() called
9.  READY
Key: context.refresh() BEFORE Tomcat starts in Boot (reverse of plain MVC)

PER-REQUEST FLOW (processRequest)
─────────────────────────────────────────────────────
Save previous ThreadLocals (for nested dispatch)
Set new ThreadLocals:
  LocaleContextHolder.setLocaleContext()
  RequestContextHolder.setRequestAttributes()
→ doService() → doDispatch()
finally (ALWAYS):
  Restore previous ThreadLocals
  requestAttributes.requestCompleted() ← @RequestScope beans destroyed here
  publishRequestHandledEvent()         ← metrics/monitoring

doService() REQUEST ATTRIBUTES SET
─────────────────────────────────────────────────────
WEB_APPLICATION_CONTEXT_ATTRIBUTE  -> WebApplicationContext
LOCALE_RESOLVER_ATTRIBUTE          -> LocaleResolver
THEME_RESOLVER_ATTRIBUTE           -> ThemeResolver
INPUT_FLASH_MAP_ATTRIBUTE          -> Incoming flash data (from redirect)
OUTPUT_FLASH_MAP_ATTRIBUTE         -> New empty FlashMap
FLASH_MAP_MANAGER_ATTRIBUTE        -> FlashMapManager

KEY INIT-PARAMS / CONFIGURATION PROPERTIES
─────────────────────────────────────────────────────
detectAllHandlerMappings   default=true  (false=only bean named "handlerMapping")
throwExceptionIfNoHandlerFound default=false (true=404 as exception)
publishEvents              default=true  (false=no ServletRequestHandledEvent)
dispatchOptionsRequest     default=true  (OPTIONS goes through doDispatch)
dispatchTraceRequest       default=false (TRACE handled by HttpServlet directly)
publishContext             default=true  (context stored in ServletContext)

THREADLOCAL TRAP
─────────────────────────────────────────────────────
Set on:  request thread only (by processRequest())
Cleared: always in finally block (no leak)
Child threads: DO NOT inherit ThreadLocals
  -> RequestContextHolder.currentRequestAttributes() throws IllegalStateException
  -> Must capture and pass context values explicitly

destroy() BEHAVIOUR
─────────────────────────────────────────────────────
Plain MVC: webApplicationContextInjected=false
  -> destroy() calls childContext.close()
  -> All child beans @PreDestroy called

Spring Boot: webApplicationContextInjected=true
  -> destroy() SKIPS close()
  -> SpringApplication shutdown hook closes context

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "init() is in HttpServletBean — not DispatcherServlet"
• "initStrategies() runs once — stores special beans as instance fields"
• "processRequest() is the single funnel for ALL HTTP methods"
• "ThreadLocals always cleaned in finally block — no thread pool leaks"
• "requestCompleted() in finally triggers @RequestScope bean destruction"
• "Spring Boot: context.refresh() before Tomcat starts (reverse of plain MVC)"
• "destroy() skips context.close() in Spring Boot — webApplicationContextInjected=true"
• "Child threads don't inherit ThreadLocals — RequestContextHolder empty in new threads"
• "detectAllHandlerMappings=false: ONLY bean named 'handlerMapping' used"
• "ServletRequestHandledEvent published after EVERY request — even failed ones"
```

---

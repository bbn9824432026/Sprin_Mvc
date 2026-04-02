# TOPIC 2.3 — SPECIAL BEANS IN DISPATCHERSERVLET

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What "Special Beans" Means

DispatcherServlet does not hardcode any behavior. Every decision it makes — how to find a handler, how to invoke it, how to resolve a view, how to handle exceptions — is **delegated to beans it discovers in the WebApplicationContext**. These beans are called "special beans" because DispatcherServlet looks for them by **type** (not by name, with one exception) and uses them as its strategy objects.

This is the **Strategy pattern** applied at the framework level. DispatcherServlet is the context. Each special bean type is a pluggable strategy. You replace a strategy by registering a bean of the appropriate type.

Every special bean has:
1. A defined interface it must implement
2. A default implementation used when no custom bean is found
3. An ordering mechanism when multiple beans of the same type exist
4. Specific behavior contracts DispatcherServlet relies on

---

### The Complete Special Bean Catalogue

```
Special Bean Type              Default Implementation
─────────────────────────────────────────────────────────────────────
HandlerMapping              →  RequestMappingHandlerMapping
                               BeanNameUrlHandlerMapping
HandlerAdapter              →  RequestMappingHandlerAdapter
                               HttpRequestHandlerAdapter
                               SimpleControllerHandlerAdapter
HandlerExceptionResolver    →  ExceptionHandlerExceptionResolver
                               ResponseStatusExceptionResolver
                               DefaultHandlerExceptionResolver
ViewResolver                →  InternalResourceViewResolver
LocaleResolver              →  AcceptHeaderLocaleResolver
LocaleContextResolver       →  AcceptHeaderLocaleResolver (extends this)
ThemeResolver               →  FixedThemeResolver
MultipartResolver           →  None (must be explicitly configured)
FlashMapManager             →  SessionFlashMapManager
RequestToViewNameTranslator →  DefaultRequestToViewNameTranslator
```

---

### 1. HandlerMapping — Full Analysis

**Interface:** `org.springframework.web.servlet.HandlerMapping`

```java
public interface HandlerMapping {
    // Constant for best-matching pattern attribute on request
    String BEST_MATCHING_HANDLER_ATTRIBUTE = HandlerMapping.class.getName() 
        + ".bestMatchingHandler";
    String LOOKUP_PATH = HandlerMapping.class.getName() + ".lookupPath";
    String PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE = HandlerMapping.class.getName() 
        + ".pathWithinHandlerMapping";
    String BEST_MATCHING_PATTERN_ATTRIBUTE = HandlerMapping.class.getName() 
        + ".bestMatchingPattern";
    String INTROSPECT_TYPE_LEVEL_MAPPING = HandlerMapping.class.getName() 
        + ".introspectTypeLevelMapping";
    String URI_TEMPLATE_VARIABLES_ATTRIBUTE = HandlerMapping.class.getName() 
        + ".uriTemplateVariables";
    String MATRIX_VARIABLES_ATTRIBUTE = HandlerMapping.class.getName() 
        + ".matrixVariables";
    String PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE = HandlerMapping.class.getName() 
        + ".producibleMediaTypes";

    // The ONE method every HandlerMapping must implement:
    @Nullable
    HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
}
```

**What `getHandler()` must return:**
- `HandlerExecutionChain` containing the handler object AND the applicable interceptors
- `null` if this mapping cannot handle the request (DispatcherServlet tries next mapping)

**Default HandlerMappings registered by `@EnableWebMvc` (in order):**

```
Order 0:  RequestMappingHandlerMapping
          → Handles @RequestMapping annotated methods
          → Builds registry: RequestMappingInfo → HandlerMethod
          → Registry built at startup, O(1) lookup per request

Order 1:  BeanNameUrlHandlerMapping  
          → Maps URL to bean name
          → Bean named "/products" maps to GET /products
          → Legacy pattern rarely used today

Order 2:  RouterFunctionMapping
          → Handles functional endpoints (RouterFunction)
          → WebFlux-style routing in Spring MVC

Order Integer.MAX_VALUE-1: SimpleUrlHandlerMapping (for resources)
          → Handles /static/**, /webjars/** etc.
          → Registered by addResourceHandlers()

Order Integer.MAX_VALUE: DefaultServletHttpRequestHandler mapping
          → Lowest priority — last resort
          → Passes to container's default servlet
```

**RequestMappingHandlerMapping internals:**

```
At startup (afterPropertiesSet()):
  → Scans all @Controller beans in WebApplicationContext
  → For each bean: reflects over all methods
  → For each @RequestMapping method: creates RequestMappingInfo
  → Stores in mappingRegistry: Map<RequestMappingInfo, HandlerMethod>

RequestMappingInfo encapsulates:
  → URL patterns (Ant / URI templates / Regex)
  → HTTP methods (GET, POST, etc.)
  → Params conditions (params="active=true")
  → Headers conditions (headers="X-Version=2")
  → Consumes conditions (consumes="application/json")
  → Produces conditions (produces="application/json")

Per request:
  → Extracts request path
  → Finds all matching RequestMappingInfos
  → Ranks by specificity
  → Returns best match as HandlerMethod
  → Extracts URI template variables → sets as request attribute
     (URI_TEMPLATE_VARIABLES_ATTRIBUTE)
```

---

### 2. HandlerAdapter — Full Analysis

**Interface:** `org.springframework.web.servlet.HandlerAdapter`

```java
public interface HandlerAdapter {
    // Can this adapter invoke the given handler?
    boolean supports(Object handler);

    // Invoke the handler and return ModelAndView (null if response written)
    @Nullable
    ModelAndView handle(HttpServletRequest request,
                        HttpServletResponse response,
                        Object handler) throws Exception;

    // For HTTP caching — Last-Modified header support
    // Return -1 if not supported
    long getLastModified(HttpServletRequest request, Object handler);
}
```

**Why adapters exist:** `HandlerMapping` can return different types of handler objects:
- `HandlerMethod` (from `RequestMappingHandlerMapping`)
- `HttpRequestHandler` (for resource handlers)
- `Controller` interface implementations (legacy)
- `HandlerFunction` (for functional endpoints)

Each handler type needs a different invocation mechanism. The Adapter pattern allows DispatcherServlet to invoke any handler type without knowing which type it is.

**`RequestMappingHandlerAdapter` — the most important:**

```
handle(request, response, handlerMethod):
  │
  ├── Finds applicable @InitBinder methods → sets up WebDataBinder
  ├── Finds applicable @ModelAttribute methods → executes them → adds to model
  ├── Creates ServletInvocableHandlerMethod from HandlerMethod
  │
  ├── Resolves method arguments (HandlerMethodArgumentResolver chain):
  │   For each method parameter:
  │   → @PathVariable    → RequestMappingHandlerMapping set URI vars on request
  │                        → UriTemplateVariablesHandlerMethodArgumentResolver
  │   → @RequestParam    → ServletRequestMethodArgumentResolver
  │   → @RequestBody     → RequestResponseBodyMethodProcessor
  │                        → reads request.getInputStream()
  │                        → finds matching HttpMessageConverter
  │                        → deserialises to method param type
  │   → @RequestHeader   → RequestHeaderMethodArgumentResolver
  │   → @CookieValue     → ServletCookieValueMethodArgumentResolver
  │   → Model            → ModelMethodProcessor (injects ModelAndViewContainer)
  │   → HttpServletRequest → ServletRequestMethodArgumentResolver
  │   → BindingResult    → ErrorsMethodArgumentResolver
  │   → etc. (26+ built-in resolvers)
  │
  ├── Invokes method via reflection: method.invoke(bean, resolvedArgs)
  │
  └── Handles return value (HandlerMethodReturnValueHandler chain):
      → String           → ViewNameMethodReturnValueHandler → sets view name
      → ModelAndView     → ModelAndViewMethodReturnValueHandler
      → @ResponseBody    → RequestResponseBodyMethodProcessor
                           → finds matching HttpMessageConverter
                           → serialises return value
                           → writes to response
      → ResponseEntity   → HttpEntityMethodProcessor
      → void             → if response written: done; else: default view name
```

**`HttpRequestHandlerAdapter`:**
```java
// Simplest adapter — just delegates directly
public ModelAndView handle(HttpServletRequest req, 
                            HttpServletResponse res, 
                            Object handler) throws Exception {
    ((HttpRequestHandler) handler).handleRequest(req, res);
    return null; // Response already written by handler
}
```

**`SimpleControllerHandlerAdapter`:**
```java
// For legacy Controller interface implementations
public ModelAndView handle(HttpServletRequest req,
                            HttpServletResponse res,
                            Object handler) throws Exception {
    return ((Controller) handler).handleRequest(req, res);
}
```

---

### 3. HandlerExceptionResolver — Full Analysis

**Interface:** `org.springframework.web.servlet.HandlerExceptionResolver`

```java
public interface HandlerExceptionResolver {
    @Nullable
    ModelAndView resolveException(HttpServletRequest request,
                                   HttpServletResponse response,
                                   @Nullable Object handler,
                                   Exception ex);
    // Returns null → this resolver cannot handle it, try next
    // Returns empty ModelAndView → exception handled, no view to render
    // Returns ModelAndView with view → render this view
}
```

**Default resolver chain (in order):**

```
Order 1: ExceptionHandlerExceptionResolver
  → Handles @ExceptionHandler methods
  → Searches current controller class FIRST
  → Then searches @ControllerAdvice beans (in @Order order)
  → Uses same argument/return value resolution as regular handlers
  → Supports @ResponseBody, ResponseEntity in @ExceptionHandler methods

Order 2: ResponseStatusExceptionResolver
  → Handles exceptions annotated with @ResponseStatus
  → Also handles ResponseStatusException thrown directly
  → Sets HTTP status code and reason phrase on response
  → Does NOT support @ResponseBody in exception handling

Order 3: DefaultHandlerExceptionResolver
  → Handles standard Spring MVC exceptions:
    MethodArgumentNotValidException → 400
    HttpRequestMethodNotSupportedException → 405
    HttpMediaTypeNotSupportedException → 415
    HttpMediaTypeNotAcceptableException → 406
    MissingPathVariableException → 500
    MissingServletRequestParameterException → 400
    MissingServletRequestPartException → 400
    BindException → 400
    NoHandlerFoundException → 404
    AsyncRequestTimeoutException → 503
    → Sets appropriate HTTP status but returns EMPTY ModelAndView
    → No response body — just status code
```

**Resolution algorithm in DispatcherServlet:**

```java
// processHandlerException() in DispatcherServlet
protected ModelAndView processHandlerException(
        HttpServletRequest request, HttpServletResponse response,
        Object handler, Exception ex) throws Exception {

    // Walk the resolver chain
    if (this.handlerExceptionResolvers != null) {
        for (HandlerExceptionResolver resolver : 
                this.handlerExceptionResolvers) {
            ModelAndView exMv = resolver.resolveException(
                request, response, handler, ex);
            if (exMv != null) {
                if (exMv.isEmpty()) {
                    // Resolver handled it but no view
                    // Response already written (e.g. @ResponseBody exception handler)
                    return null;
                }
                // Has view name → render it
                if (!exMv.hasView()) {
                    exMv.setViewName(
                        getDefaultViewName(request));
                }
                return exMv;
            }
            // null returned → try next resolver
        }
    }
    // No resolver handled it → re-throw
    throw ex;
}
```

---

### 4. ViewResolver — Full Analysis

**Interface:** `org.springframework.web.servlet.ViewResolver`

```java
public interface ViewResolver {
    @Nullable
    View resolveViewName(String viewName, Locale locale) throws Exception;
    // Returns null → this resolver cannot resolve it, try next
    // Returns View → use this View for rendering
}
```

**Default:** `InternalResourceViewResolver` (only registered when no other is configured)

**ViewResolver chain — how it works:**

```
DispatcherServlet.render(mv, request, response):
  │
  ├── viewName = mv.getViewName()  (e.g., "product/list")
  │
  └── resolveViewName(viewName, locale):
        Iterates viewResolvers (ordered):
        │
        ├── ContentNegotiatingViewResolver (if configured, order=Ordered.HIGHEST_PRECEDENCE)
        │     → Delegates to all other resolvers
        │     → Selects best view based on Accept header / content negotiation
        │     → Returns View that matches what client accepts
        │
        ├── BeanNameViewResolver
        │     → Looks for bean named "product/list" in context
        │     → If found AND implements View → return it
        │     → Used for custom View implementations
        │
        ├── InternalResourceViewResolver (typically last)
        │     → Prepends prefix: /WEB-INF/views/
        │     → Appends suffix: .jsp
        │     → Returns InternalResourceView("/WEB-INF/views/product/list.jsp")
        │     → Never returns null — always claims it can resolve
        │     → This is why it must be LAST in the chain
        │
        └── null returned → ServletException: no view found
```

**Critical: `InternalResourceViewResolver` always returns non-null.** It creates an `InternalResourceView` for any view name without checking if the JSP file actually exists. The 404 from a missing JSP only occurs when the view tries to render (forward to non-existent resource). This is why `InternalResourceViewResolver` must be the last resolver.

**View interface:**
```java
public interface View {
    String RESPONSE_STATUS_ATTRIBUTE = View.class.getName() + ".responseStatus";
    String PATH_VARIABLES = View.class.getName() + ".pathVariables";
    String SELECTED_CONTENT_TYPE = View.class.getName() + ".selectedContentType";

    @Nullable
    default String getContentType() { return null; }

    void render(@Nullable Map<String, ?> model,
                HttpServletRequest request,
                HttpServletResponse response) throws Exception;
}
```

---

### 5. LocaleResolver — Full Analysis

**Interface:** `org.springframework.web.servlet.LocaleResolver`

```java
public interface LocaleResolver {
    Locale resolveLocale(HttpServletRequest request);
    void setLocale(HttpServletRequest request,
                   @Nullable HttpServletResponse response,
                   @Nullable Locale locale);
}
```

**Implementations:**

```
AcceptHeaderLocaleResolver (DEFAULT)
  → Reads Accept-Language HTTP header
  → Returns Locale from header
  → setLocale() throws UnsupportedOperationException
    (cannot change Accept header — it comes from client)
  → No locale persistence across requests

SessionLocaleResolver
  → Stores/retrieves Locale in HttpSession
  → setLocale() persists to session
  → Survives page navigation within same session
  → Use with LocaleChangeInterceptor for ?lang=fr switching

CookieLocaleResolver
  → Stores/retrieves Locale in Cookie
  → setLocale() writes cookie to response
  → Persists across browser sessions
  → Use with LocaleChangeInterceptor

FixedLocaleResolver
  → Always returns same configured Locale
  → setLocale() throws UnsupportedOperationException
  → Useful for single-locale applications

LocaleContextResolver (extends LocaleResolver)
  → Also provides TimeZone
  → SessionLocaleResolver and CookieLocaleResolver implement this
```

**LocaleContextHolder:**
```java
// Set by FrameworkServlet.processRequest() using resolved locale
LocaleContextHolder.setLocaleContext(
    buildLocaleContext(request)
    // → calls localeResolver.resolveLocale(request)
);
// Available throughout request via:
Locale locale = LocaleContextHolder.getLocale();
```

---

### 6. MultipartResolver — Full Analysis

**Interface:** `org.springframework.web.multipart.MultipartResolver`

```java
public interface MultipartResolver {
    boolean isMultipart(HttpServletRequest request);
    MultipartHttpServletRequest resolveMultipart(HttpServletRequest request)
        throws MultipartException;
    void cleanupMultipart(MultipartHttpServletRequest request);
}
```

**Default:** NONE. Must be explicitly configured. If not configured, file uploads throw `MultipartException`.

**Bean name is critical:** Must be named exactly **`"multipartResolver"`**:

```java
// initMultipartResolver() uses NAME lookup, not type lookup:
private void initMultipartResolver(ApplicationContext context) {
    try {
        this.multipartResolver = context.getBean(
            MULTIPART_RESOLVER_BEAN_NAME,  // = "multipartResolver"
            MultipartResolver.class);
    } catch (NoSuchBeanDefinitionException ex) {
        this.multipartResolver = null; // No multipart support
    }
}
```

**This is the ONLY special bean looked up by NAME rather than type.**

**Implementations:**

```
StandardServletMultipartResolver (preferred since Servlet 3.0)
  → Uses Servlet 3.0 Part API (javax/jakarta.servlet.http.Part)
  → No additional library needed
  → Must configure MultipartConfigElement on the servlet registration
  → Bean: @Bean("multipartResolver") StandardServletMultipartResolver

CommonsMultipartResolver (legacy)
  → Uses Apache Commons FileUpload library
  → Works with Servlet 2.x
  → More configuration options (max sizes, temp dir, encoding)
  → Not recommended for new projects
```

**File upload processing flow:**

```
Request arrives with Content-Type: multipart/form-data
        │
        ▼
DispatcherServlet.doDispatch()
        │
        ▼
checkMultipart(request):
  → multipartResolver.isMultipart(request)?
    → Content-Type starts with "multipart/"? → YES
  → multipartResolver.resolveMultipart(request)
    → Returns MultipartHttpServletRequest wrapper
    → Files parsed and available via getFile(), getParts()
        │
        ▼
Normal handler processing
  → @RequestParam MultipartFile file → resolved by ArgumentResolver
  → @RequestPart MultipartFile file → resolved by RequestPartMethodArgumentResolver
        │
        ▼
doDispatch() finally:
  → cleanupMultipart(request)
  → Temp files deleted
```

---

### 7. FlashMapManager — Full Analysis

**Interface:** `org.springframework.web.servlet.FlashMapManager`

```java
public interface FlashMapManager {
    @Nullable
    FlashMap retrieveAndUpdate(HttpServletRequest request,
                               HttpServletResponse response);

    void saveOutputFlashMap(FlashMap flashMap,
                            HttpServletRequest request,
                            HttpServletResponse response);
}
```

**Default:** `SessionFlashMapManager`

**Purpose:** Flash attributes survive exactly ONE redirect. They are stored BEFORE the redirect and removed AFTER the next request reads them.

**How RedirectAttributes works internally:**

```
Controller sets RedirectAttributes:
  redirectAttributes.addFlashAttribute("message", "Saved!");
  return "redirect:/success";
        │
        ▼
DispatcherServlet.processDispatchResult()
  → render() → RedirectView.renderMergedOutputModel()
  → RequestContextUtils.saveOutputFlashMap(flashMap, request, response)
  → FlashMapManager.saveOutputFlashMap()
  → Stores flashMap in HttpSession
  → Sets targetRequestPath = "/success"
  → Sets expirationTime = now + 3 minutes (default)
        │
        ▼
HTTP 302 redirect sent to client
        │
        ▼
Client makes GET /success request (new request)
        │
        ▼
DispatcherServlet.doService()
  → flashMapManager.retrieveAndUpdate(request, response)
  → Finds flash map in session matching /success path
  → Removes from session (one-time use)
  → Sets as INPUT_FLASH_MAP_ATTRIBUTE on request
        │
        ▼
Controller reads flash attributes:
  @GetMapping("/success")
  public String success(@ModelAttribute("message") String msg) {
      // msg = "Saved!" — from flash map
  }
  // OR via Model (Spring auto-adds flash map contents to Model)
```

**`FlashMap` structure:**
```java
public final class FlashMap extends HashMap<String, Object> 
    implements Comparable<FlashMap> {
    
    private String targetRequestPath;
    // Ensures flash map only consumed by correct URL
    
    private final MultiValueMap<String, String> targetRequestParams;
    // Also matches query params for precise targeting
    
    private long expirationTime = -1;
    // Prevents stale flash attributes from being consumed
}
```

---

### 8. RequestToViewNameTranslator — Full Analysis

**Interface:** `org.springframework.web.servlet.RequestToViewNameTranslator`

```java
public interface RequestToViewNameTranslator {
    @Nullable
    String getViewName(HttpServletRequest request) throws Exception;
}
```

**Default:** `DefaultRequestToViewNameTranslator`

**Purpose:** When a controller method returns `void` or `null` (no explicit view name), this translator derives a view name from the request URL.

```java
// DefaultRequestToViewNameTranslator algorithm:
// GET /products/list → view name "products/list"
// GET /products/list.html → view name "products/list" (strips extension)
// GET /app/products/list → view name "products/list" (strips context path)

// Configuration options:
translator.setPrefix("");       // Prepend to derived name
translator.setSuffix("");       // Append to derived name
translator.setSeparator("/");   // URL separator char
translator.setStripLeadingSlash(true);
translator.setStripTrailingSlash(true);
translator.setStripExtension(true);
```

**When it's used:**

```java
@Controller
public class ProductController {

    // Returns void — no explicit view name
    // DefaultRequestToViewNameTranslator derives "products/list"
    // from request URI /products/list
    @GetMapping("/products/list")
    public void list(Model model) {
        model.addAttribute("products", productService.findAll());
        // View: /WEB-INF/views/products/list.jsp (with InternalResourceViewResolver)
    }
}
```

---

### Special Bean Interaction — Complete Flow Diagram

```
Request: POST /api/orders
         Content-Type: multipart/form-data
         Accept: application/json
         
                    │
                    ▼
         MultipartResolver.isMultipart() → YES
         resolveMultipart() → MultipartHttpServletRequest
                    │
                    ▼
         HandlerMapping chain:
         RequestMappingHandlerMapping.getHandler()
           → Looks up POST /api/orders in registry
           → Finds OrderController.createOrder(MultipartFile, ...)
           → Assembles HandlerExecutionChain + interceptors
                    │
                    ▼
         HandlerAdapter chain:
         RequestMappingHandlerAdapter.supports(HandlerMethod) → TRUE
         handle():
           → @ModelAttribute methods execute
           → ArgumentResolvers resolve parameters
           → method.invoke() → OrderController.createOrder()
           → ReturnValueHandlers: @ResponseBody → Jackson → JSON written
                    │
                    ▼
         (No ViewResolver needed — response already written)
                    │
                    ▼
         HandlerInterceptor.afterCompletion()
                    │
                    ▼
         MultipartResolver.cleanupMultipart() → temp files deleted
                    │
                    ▼
         LocaleContextHolder / RequestContextHolder cleared
                    │
                    ▼
         ServletRequestHandledEvent published
```

---

### detectAll Flags — Controlling Special Bean Discovery

```java
// DispatcherServlet properties:
private boolean detectAllHandlerMappings = true;
private boolean detectAllHandlerAdapters = true;
private boolean detectAllHandlerExceptionResolvers = true;
private boolean detectAllViewResolvers = true;

// When true (default): uses BeanFactoryUtils.beansOfTypeIncludingAncestors()
// → Finds ALL beans of the type in context + parent context

// When false: uses context.getBean("handlerMapping", HandlerMapping.class)
// → Finds ONLY the bean with that specific name

// Only MultipartResolver always uses name lookup ("multipartResolver")
// regardless of any detectAll flag
```

---

## 2️⃣ CODE EXAMPLES

### Registering All Special Beans Explicitly (Java Config)

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    // ── HandlerMapping (auto-registered by @EnableWebMvc) ──────────────────
    // RequestMappingHandlerMapping is registered automatically
    // To customise:
    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        configurer.setUseTrailingSlashMatch(false);
        configurer.setUseSuffixPatternMatch(false); // deprecated but tested
    }

    // ── MultipartResolver (MUST be named "multipartResolver") ──────────────
    @Bean(name = "multipartResolver") // Name is CRITICAL
    public MultipartResolver multipartResolver() {
        return new StandardServletMultipartResolver();
        // Also need MultipartConfigElement on servlet registration
    }

    // ── LocaleResolver ─────────────────────────────────────────────────────
    @Bean
    public LocaleResolver localeResolver() {
        SessionLocaleResolver resolver = new SessionLocaleResolver();
        resolver.setDefaultLocale(Locale.ENGLISH);
        return resolver;
    }

    // ── LocaleChangeInterceptor (works with SessionLocaleResolver) ──────────
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        LocaleChangeInterceptor lci = new LocaleChangeInterceptor();
        lci.setParamName("lang"); // ?lang=fr changes locale
        registry.addInterceptor(lci);
    }

    // ── ViewResolver chain ──────────────────────────────────────────────────
    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        // Order 1: BeanNameViewResolver (for custom View beans)
        registry.beanName();

        // Order 2: JSP resolver (always last — never returns null)
        registry.jsp("/WEB-INF/views/", ".jsp");
    }

    // ── FlashMapManager (default: SessionFlashMapManager) ──────────────────
    // Usually not needed to override — default works well
    @Bean
    public FlashMapManager flashMapManager() {
        return new SessionFlashMapManager();
    }

    // ── RequestToViewNameTranslator ─────────────────────────────────────────
    @Bean
    public RequestToViewNameTranslator viewNameTranslator() {
        DefaultRequestToViewNameTranslator translator =
            new DefaultRequestToViewNameTranslator();
        translator.setStripExtension(true);
        translator.setStripLeadingSlash(true);
        return translator;
    }
}
```

---

### Custom HandlerMapping — URL Prefix to Controller Package

```java
// Custom HandlerMapping that maps /v1/** to v1 controllers
// and /v2/** to v2 controllers
@Component
public class VersionedHandlerMapping
    extends RequestMappingHandlerMapping {

    @Override
    protected boolean isHandler(Class<?> beanType) {
        // Only consider controllers in com.example.v1 or v2
        return super.isHandler(beanType) &&
               (beanType.getPackageName().contains(".v1") ||
                beanType.getPackageName().contains(".v2"));
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE; // Check before default mapping
    }
}

// DispatcherServlet finds this via detectAllHandlerMappings=true
// No explicit registration needed — @Component is sufficient
```

---

### Custom HandlerExceptionResolver

```java
// Custom resolver that handles domain-specific exceptions
@Component
@Order(0) // Before all default resolvers
public class DomainExceptionResolver implements HandlerExceptionResolver {

    @Override
    public ModelAndView resolveException(HttpServletRequest request,
                                          HttpServletResponse response,
                                          Object handler,
                                          Exception ex) {

        if (ex instanceof ProductNotFoundException pnfe) {
            response.setStatus(HttpStatus.NOT_FOUND.value());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            try {
                response.getWriter().write(
                    "{\"error\":\"Product not found\",\"id\":" +
                    pnfe.getProductId() + "}"
                );
            } catch (IOException e) {
                // log
            }
            return new ModelAndView(); // Empty MaV = handled, no view
        }

        return null; // Not handled — try next resolver
    }
}
```

---

### Custom ViewResolver — Database-Driven Templates

```java
// ViewResolver that loads templates from database
@Component
@Order(1) // Before InternalResourceViewResolver
public class DatabaseViewResolver implements ViewResolver {

    @Autowired
    private TemplateRepository templateRepository;

    @Override
    public View resolveViewName(String viewName, Locale locale)
            throws Exception {

        Optional<Template> template =
            templateRepository.findByName(viewName);

        if (template.isPresent()) {
            return new DatabaseTemplateView(template.get());
        }

        return null; // Not found — try next resolver (InternalResourceViewResolver)
    }
}
```

---

### Edge Case — Wrong MultipartResolver Bean Name

```java
// WRONG — bean not found by DispatcherServlet
// initMultipartResolver looks for name "multipartResolver" exactly
@Bean(name = "fileUploadResolver") // WRONG NAME
public MultipartResolver multipartResolver() {
    return new StandardServletMultipartResolver();
}
// Result: multipartResolver field = null in DispatcherServlet
// File upload requests throw MultipartException or files not parsed

// CORRECT
@Bean(name = "multipartResolver") // EXACT name required
public MultipartResolver multipartResolver() {
    return new StandardServletMultipartResolver();
}
```

---

### Accessing Special Beans at Runtime

```java
@Controller
public class DiagnosticsController {

    // DispatcherServlet sets WebApplicationContext as request attribute
    // We can access it and inspect special beans
    @GetMapping("/diag/beans")
    @ResponseBody
    public Map<String, Object> diagnostics(HttpServletRequest request) {

        WebApplicationContext wac = (WebApplicationContext)
            request.getAttribute(
                DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE);

        Map<String, Object> info = new LinkedHashMap<>();

        // HandlerMappings
        info.put("handlerMappings",
            wac.getBeansOfType(HandlerMapping.class)
               .keySet());

        // HandlerAdapters
        info.put("handlerAdapters",
            wac.getBeansOfType(HandlerAdapter.class)
               .keySet());

        // ViewResolvers
        info.put("viewResolvers",
            wac.getBeansOfType(ViewResolver.class)
               .keySet());

        // LocaleResolver
        info.put("localeResolver",
            wac.getBeansOfType(LocaleResolver.class)
               .keySet());

        // MultipartResolver — name-based lookup
        boolean hasMultipart = wac.containsBean("multipartResolver");
        info.put("multipartResolver", hasMultipart);

        return info;
    }
}
```

---

### FlashAttributes Complete Example

```java
@Controller
@RequestMapping("/orders")
public class OrderController {

    @PostMapping
    public String createOrder(@Valid @ModelAttribute Order order,
                               BindingResult result,
                               RedirectAttributes redirectAttrs) {
        if (result.hasErrors()) {
            return "order/form";
        }

        Order saved = orderService.save(order);

        // Flash attribute — survives ONE redirect
        // Stored in session by SessionFlashMapManager
        redirectAttrs.addFlashAttribute("successMessage",
            "Order #" + saved.getId() + " created successfully!");

        // Regular redirect attribute — appears in URL query string
        redirectAttrs.addAttribute("orderId", saved.getId());

        // URL: /orders/42?orderId=42 (orderId in URL)
        // "successMessage" in flash — NOT in URL
        return "redirect:/orders/{orderId}";
    }

    @GetMapping("/{id}")
    public String viewOrder(@PathVariable Long id,
                             Model model) {
        // Flash attribute automatically added to Model by Spring
        // "successMessage" is in model if redirected from createOrder
        model.addAttribute("order", orderService.findById(id));
        return "order/view";
        // In JSP: ${successMessage} shows the message
    }
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
Which special bean in DispatcherServlet is discovered by NAME rather than by TYPE?

A) `HandlerMapping`
B) `ViewResolver`
C) `MultipartResolver`
D) `HandlerAdapter`

**Answer: C**
`MultipartResolver` is the ONLY special bean looked up by the fixed name `"multipartResolver"`. All other special beans are discovered by TYPE using `BeanFactoryUtils.beansOfTypeIncludingAncestors()` (when `detectAll*=true`). If you name your `MultipartResolver` bean anything else, DispatcherServlet won't find it.

---

**Q2 — Select All That Apply**
A `HandlerExceptionResolver.resolveException()` returns an **empty** `ModelAndView` (not null). Which of the following are TRUE?

A) DispatcherServlet tries the next resolver in the chain
B) DispatcherServlet considers the exception handled
C) No view is rendered
D) The response has already been written by the resolver
E) DispatcherServlet re-throws the exception

**Answer: B, C, D**
An empty `ModelAndView` (created by `new ModelAndView()`) signals "exception handled, no view needed." DispatcherServlet stops the resolver chain (B), does not render any view (C), and assumes the resolver already wrote the response (D). Returning `null` would try the next resolver (A). Re-throwing only happens when no resolver handles it (E).

---

**Q3 — Scenario**
`InternalResourceViewResolver` is configured with prefix `/WEB-INF/views/` and suffix `.jsp`. A controller returns view name `"product/list"`. The file `/WEB-INF/views/product/list.jsp` does NOT exist. What happens?

A) `ViewResolutionException` thrown at handler return time
B) HTTP 500 — `InternalResourceViewResolver` can't find the JSP
C) `InternalResourceViewResolver` returns `null` — next resolver tried
D) HTTP 404 — the forward to non-existent JSP resource fails

**Answer: D**
`InternalResourceViewResolver` NEVER returns null — it always creates an `InternalResourceView`. The view resolves fine. The 404 only occurs when `InternalResourceView.render()` calls `request.getDispatcher("/WEB-INF/views/product/list.jsp").forward(req, res)`. The Servlet container's dispatcher cannot find the JSP → HTTP 404 from the container's error handling.

---

**Q4 — MCQ**
`DefaultRequestToViewNameTranslator` is used. A controller has this method:

```java
@GetMapping("/account/settings")
public void settings(Model model) {
    model.addAttribute("user", currentUser());
}
```

What view name is derived?

A) `"/account/settings"`
B) `"account/settings"`
C) `"settings"`
D) `"account_settings"`

**Answer: B**
`DefaultRequestToViewNameTranslator` strips the leading slash from the request URI `/account/settings` → `"account/settings"`. The `stripLeadingSlash=true` by default. No extension to strip. Result: `"account/settings"`. With `InternalResourceViewResolver`: `/WEB-INF/views/account/settings.jsp`.

---

**Q5 — Select All That Apply**
Which of the following are correctly matched special bean interfaces and their primary purpose?

A) `HandlerMapping` → Maps HTTP requests to handler objects
B) `ViewResolver` → Converts view names to `View` objects for rendering
C) `FlashMapManager` → Manages temporary attributes across redirects via session
D) `MultipartResolver` → Parses binary file bodies from any request
E) `LocaleResolver` → Resolves `Locale` for the current request

**Answer: A, B, C, E**
D is partially wrong — `MultipartResolver` only parses requests with `Content-Type: multipart/form-data`. It does NOT parse binary content from arbitrary requests. Regular binary body parsing is done by `HttpMessageConverter`.

---

**Q6 — Ordering**
When a POST request arrives with `Content-Type: multipart/form-data`, order these DispatcherServlet operations:

- `HandlerAdapter.handle()` invoked
- `MultipartResolver.cleanupMultipart()` called
- `HandlerMapping.getHandler()` returns `HandlerExecutionChain`
- `MultipartResolver.resolveMultipart()` wraps request
- `HandlerInterceptor.preHandle()` called
- `HandlerInterceptor.afterCompletion()` called

**Correct Order:**
1. `MultipartResolver.resolveMultipart()` wraps request
2. `HandlerMapping.getHandler()` returns `HandlerExecutionChain`
3. `HandlerInterceptor.preHandle()` called
4. `HandlerAdapter.handle()` invoked
5. `HandlerInterceptor.afterCompletion()` called
6. `MultipartResolver.cleanupMultipart()` called

---

**Q7 — True/False**
`SessionFlashMapManager` stores flash attributes in `HttpSession`. If the session expires before the redirect target is accessed, the flash attributes are lost.

**Answer: True**
Flash attributes are stored in `HttpSession` by `SessionFlashMapManager`. If the session expires (or is invalidated) before the redirected request retrieves them, the flash map is gone. This is a real reliability concern in high-latency redirect scenarios. `FlashMap` also has an expiration time (default 3 minutes) to prevent accumulation of unused flash attributes.

---

**Q8 — MCQ**
A `HandlerExceptionResolver` returns `null` for every exception. What does DispatcherServlet do?

A) Returns HTTP 200 with empty body
B) Logs the exception and returns HTTP 500
C) Re-throws the exception — Servlet container handles it
D) Calls `response.sendError(500)` and returns

**Answer: C**
If all `HandlerExceptionResolver`s return `null`, `processHandlerException()` re-throws the original exception. The exception propagates up through `doDispatch()` → `doService()` → `processRequest()` → out of `DispatcherServlet.service()`. The Servlet container receives the unhandled exception and typically generates an HTTP 500 response.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — MultipartResolver bean name**
Every other special bean is looked up by TYPE. MultipartResolver is looked up by NAME: `"multipartResolver"`. Name it anything else → no multipart support → `MultipartException` at runtime. This is the #1 trap in multipart configuration questions.

**Trap 2 — InternalResourceViewResolver never returns null**
All other ViewResolvers can return null (cannot handle this view name — try next). `InternalResourceViewResolver` ALWAYS returns a view. This means: it must be LAST in the chain. If placed first, it intercepts all view names, preventing Thymeleaf/FreeMarker/etc. resolvers from ever being consulted.

**Trap 3 — Empty ModelAndView vs null from HandlerExceptionResolver**
`null` = "I can't handle this exception — try next resolver."
`new ModelAndView()` (empty) = "I handled it — no view needed."
`new ModelAndView("errorView")` = "I handled it — render this view."
Returning empty `ModelAndView` when the resolver forgot to write the response body results in a blank HTTP 200 response with no body — a silent failure.

**Trap 4 — HandlerAdapter.supports() determines which adapter is used**
If a `HandlerMethod` is returned by `HandlerMapping` but NO `HandlerAdapter` has `supports(handler) == true`, `ServletException: No adapter for handler` is thrown. This happens when a custom handler type is returned by a `HandlerMapping` but no corresponding adapter is registered. Exam questions present custom handler types and ask what error occurs.

**Trap 5 — FlashAttributes and multiple redirects**
Flash attributes survive exactly ONE redirect. If you redirect twice (A → B → C), flash attributes set on A are available on B but NOT on C. If you need them on C, you must explicitly re-save them as flash attributes from B's handler.

**Trap 6 — LocaleResolver.setLocale() not always supported**
`AcceptHeaderLocaleResolver.setLocale()` throws `UnsupportedOperationException`. If you configure `LocaleChangeInterceptor` (which calls `setLocale()`) with `AcceptHeaderLocaleResolver`, you get `UnsupportedOperationException` at runtime when locale change is attempted. Must use `SessionLocaleResolver` or `CookieLocaleResolver` with `LocaleChangeInterceptor`.

**Trap 7 — detectAllHandlerMappings=false and bean naming**
With `detectAllHandlerMappings=false`, only a bean named exactly `"handlerMapping"` is used. `"requestMappingHandlerMapping"` (the auto-registered name from `@EnableWebMvc`) is IGNORED. Application has no handler mapping → every request gets 404/500. Exam questions present this scenario and ask why all requests fail.

---

## 5️⃣ SUMMARY SHEET

```
SPECIAL BEANS — COMPLETE CATALOGUE
─────────────────────────────────────────────────────
HandlerMapping              Default: RequestMappingHandlerMapping
                                     BeanNameUrlHandlerMapping
HandlerAdapter              Default: RequestMappingHandlerAdapter
                                     HttpRequestHandlerAdapter
                                     SimpleControllerHandlerAdapter
HandlerExceptionResolver    Default: ExceptionHandlerExceptionResolver (order 1)
                                     ResponseStatusExceptionResolver (order 2)
                                     DefaultHandlerExceptionResolver (order 3)
ViewResolver                Default: InternalResourceViewResolver (last resort)
LocaleResolver              Default: AcceptHeaderLocaleResolver
ThemeResolver               Default: FixedThemeResolver
MultipartResolver           Default: NONE — must configure explicitly
FlashMapManager             Default: SessionFlashMapManager
RequestToViewNameTranslator Default: DefaultRequestToViewNameTranslator

DISCOVERY METHOD
─────────────────────────────────────────────────────
ALL special beans: discovered by TYPE
  → BeanFactoryUtils.beansOfTypeIncludingAncestors()
  → Searches child context AND root context

EXCEPTION: MultipartResolver — discovered by NAME "multipartResolver"
  → context.getBean("multipartResolver", MultipartResolver.class)
  → Wrong name = no multipart support (silently)

HANDLERMAPPING ORDER (default @EnableWebMvc)
─────────────────────────────────────────────────────
Order 0:            RequestMappingHandlerMapping  (@RequestMapping methods)
Order 1:            BeanNameUrlHandlerMapping     (bean name = URL)
Order 2:            RouterFunctionMapping         (functional endpoints)
Order MAX-1:        SimpleUrlHandlerMapping       (static resources)
Order MAX:          DefaultServletHandlerMapping  (container default servlet)

HANDLERADAPTER — supports() method determines which is used
─────────────────────────────────────────────────────
RequestMappingHandlerAdapter  → supports HandlerMethod (from @RequestMapping)
HttpRequestHandlerAdapter     → supports HttpRequestHandler
SimpleControllerHandlerAdapter → supports Controller interface

EXCEPTION RESOLVER CHAIN ORDER
─────────────────────────────────────────────────────
1. ExceptionHandlerExceptionResolver → @ExceptionHandler (local → @ControllerAdvice)
2. ResponseStatusExceptionResolver   → @ResponseStatus on exception class
3. DefaultHandlerExceptionResolver   → Standard Spring MVC exceptions → status only

resolveException() return values:
  null        → not handled, try next resolver
  empty MaV   → handled, response already written (no view rendered)
  MaV+view    → handled, render this view

VIEWRESOLVER CHAIN RULES
─────────────────────────────────────────────────────
Resolvers tried in order — first non-null wins
InternalResourceViewResolver NEVER returns null — must be LAST
ContentNegotiatingViewResolver delegates to ALL others — usually FIRST
BeanNameViewResolver: looks for bean named = viewName — before JSP resolver

LOCALERESOLVER COMPARISON
─────────────────────────────────────────────────────
AcceptHeaderLocaleResolver  → HTTP header only, setLocale() THROWS
SessionLocaleResolver       → Session storage, setLocale() works
CookieLocaleResolver        → Cookie storage, setLocale() works
FixedLocaleResolver         → Always same locale, setLocale() THROWS

LocaleChangeInterceptor requires SessionLocaleResolver or CookieLocaleResolver

MULTIPARTRESOLVER — CRITICAL FACTS
─────────────────────────────────────────────────────
Bean name MUST be: "multipartResolver"
Default: NONE — file uploads fail without explicit configuration
Cleanup: always called in doDispatch() finally — temp files deleted
StandardServletMultipartResolver → Servlet 3.0+ Part API (preferred)
CommonsMultipartResolver → Apache Commons FileUpload (legacy)

FLASHMAPMANAGER FLOW
─────────────────────────────────────────────────────
Before redirect:  saveOutputFlashMap() → stores in HttpSession
After redirect:   retrieveAndUpdate() → retrieves + REMOVES from session
Lifetime:         ONE request after redirect (removed immediately after)
Default expiry:   3 minutes (prevents stale flash accumulation)
Multiple redirects: flash attributes survive ONLY ONE redirect

REQUESTTOVIEWNAMETRANSLATOR
─────────────────────────────────────────────────────
Used when: controller returns void or null (no explicit view name)
Algorithm: strips leading slash, strips extension, uses URI as view name
GET /account/settings → void → view name: "account/settings"

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "MultipartResolver is the ONLY special bean looked up by name — 'multipartResolver'"
• "InternalResourceViewResolver never returns null — must always be last in chain"
• "HandlerAdapter.supports() determines which adapter invokes which handler type"
• "Empty ModelAndView from HandlerExceptionResolver = handled, no view rendered"
• "AcceptHeaderLocaleResolver.setLocale() throws — use Session/Cookie resolver with LocaleChangeInterceptor"
• "Flash attributes survive exactly ONE redirect — stored/removed from session"
• "detectAll=false: only bean named 'handlerMapping' used — others silently ignored"
• "ExceptionHandlerExceptionResolver checks local @Controller first, then @ControllerAdvice"
• "DefaultHandlerExceptionResolver sets status code only — no response body"
• "@PostConstruct fires before proxy — this.getClass() shows raw class, not proxy"
```

---

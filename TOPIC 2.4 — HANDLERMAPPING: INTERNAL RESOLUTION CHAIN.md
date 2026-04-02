# TOPIC 2.4 — HANDLERMAPPING: INTERNAL RESOLUTION CHAIN

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What HandlerMapping Really Is

`HandlerMapping` is the **URL-to-handler translation layer** of Spring MVC. It answers one question: *"Given this HTTP request, which handler object should process it?"* The answer is not just a handler — it is a `HandlerExecutionChain` containing the handler AND all applicable interceptors.

Understanding HandlerMapping at depth means understanding: how the registry is built, how URL matching works internally, how ambiguity is resolved, how interceptors are assembled, and what happens when no match is found. Every one of these is a certification question source.

---

### The HandlerMapping Interface — Revisited Precisely

```java
public interface HandlerMapping {

    // Request attributes set by HandlerMapping implementations:
    String BEST_MATCHING_HANDLER_ATTRIBUTE =
        HandlerMapping.class.getName() + ".bestMatchingHandler";

    String LOOKUP_PATH =
        HandlerMapping.class.getName() + ".lookupPath";

    String PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE =
        HandlerMapping.class.getName() + ".pathWithinHandlerMapping";

    String BEST_MATCHING_PATTERN_ATTRIBUTE =
        HandlerMapping.class.getName() + ".bestMatchingPattern";

    String URI_TEMPLATE_VARIABLES_ATTRIBUTE =
        HandlerMapping.class.getName() + ".uriTemplateVariables";

    String MATRIX_VARIABLES_ATTRIBUTE =
        HandlerMapping.class.getName() + ".matrixVariables";

    String PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE =
        HandlerMapping.class.getName() + ".producibleMediaTypes";

    // The contract:
    @Nullable
    HandlerExecutionChain getHandler(HttpServletRequest request)
        throws Exception;
}
```

**Every `HandlerMapping` implementation sets several request attributes** when it finds a match. These attributes are later used by argument resolvers, view resolvers, and interceptors. Understanding which attributes are set is critical for understanding how `@PathVariable`, matrix variables, and URL patterns work.

---

### The HandlerMapping Hierarchy

```
HandlerMapping (interface)
    │
    ├── AbstractHandlerMapping
    │       │ Provides: interceptor management, CORS handling,
    │       │           getHandler() template method,
    │       │           HandlerExecutionChain assembly
    │       │
    │       ├── AbstractUrlHandlerMapping
    │       │       │ Provides: URL-to-handler map registration
    │       │       │           Ant pattern matching on URLs
    │       │       │
    │       │       ├── SimpleUrlHandlerMapping
    │       │       │       Maps specific URL patterns to handlers
    │       │       │       Used for static resources, custom handlers
    │       │       │
    │       │       └── BeanNameUrlHandlerMapping
    │       │               Maps bean names starting with "/" to handlers
    │       │               Legacy pattern — bean named "/products" handles /products
    │       │
    │       └── AbstractHandlerMethodMapping
    │               │ Provides: method-level mapping registry
    │               │           RequestMappingInfo → HandlerMethod registry
    │               │           Ambiguity resolution
    │               │
    │               └── RequestMappingHandlerMapping
    │                       THE primary HandlerMapping
    │                       Processes @RequestMapping/@GetMapping etc.
    │                       Scans @Controller beans at startup
    │
    └── RouterFunctionMapping
            Handles functional RouterFunction endpoints
            No inheritance from AbstractHandlerMapping
```

---

### AbstractHandlerMapping — The Template Method Pattern

`AbstractHandlerMapping` implements `getHandler()` as a template method:

```java
// AbstractHandlerMapping.getHandler() — the template
@Override
@Nullable
public final HandlerExecutionChain getHandler(
        HttpServletRequest request) throws Exception {

    // STEP 1: Delegate to subclass to find the actual handler
    Object handler = getHandlerInternal(request);
    // → RequestMappingHandlerMapping overrides this
    // → Returns HandlerMethod or null

    // STEP 2: If no handler, use default handler (if configured)
    if (handler == null) {
        handler = getDefaultHandler();
        // defaultHandler can be set for catch-all handling
    }

    // STEP 3: If still no handler, return null
    // DispatcherServlet will call noHandlerFound()
    if (handler == null) {
        return null;
    }

    // STEP 4: Resolve handler bean name to actual bean instance
    if (handler instanceof String handlerName) {
        handler = obtainApplicationContext().getBean(handlerName);
    }

    // STEP 5: Look up or build HandlerExecutionChain
    HandlerExecutionChain executionChain =
        getHandlerExecutionChain(handler, request);

    // STEP 6: Handle CORS pre-flight requests
    if (hasCorsConfigurationSource(handler) ||
        CorsUtils.isPreFlightRequest(request)) {
        CorsConfiguration config = getCorsConfiguration(handler, request);
        if (getCorsConfigurationSource() != null) {
            CorsConfiguration globalConfig =
                getCorsConfigurationSource()
                    .getCorsConfiguration(request);
            config = (globalConfig != null ?
                globalConfig.combine(config) : config);
        }
        if (config != null) {
            config.validateAllowCredentials();
        }
        executionChain = getCorsHandlerExecutionChain(
            request, executionChain, config);
    }

    return executionChain;
}
```

---

### getHandlerExecutionChain() — Assembling Interceptors

```java
// AbstractHandlerMapping.getHandlerExecutionChain()
protected HandlerExecutionChain getHandlerExecutionChain(
        Object handler, HttpServletRequest request) {

    // Create chain with handler
    HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain hec ?
        hec : new HandlerExecutionChain(handler));

    // Add applicable interceptors
    for (HandlerInterceptor interceptor : this.adaptedInterceptors) {

        if (interceptor instanceof MappedInterceptor mappedInterceptor) {
            // MappedInterceptor has URL patterns
            if (mappedInterceptor.matches(request)) {
                // URL matches this interceptor's patterns
                chain.addInterceptor(mappedInterceptor.getInterceptor());
            }
        } else {
            // Regular interceptor — applies to ALL requests
            chain.addInterceptor(interceptor);
        }
    }

    return chain;
}
```

**This is why interceptors added via `addInterceptors()` appear in the chain.** The `WebMvcConfigurer.addInterceptors()` method ultimately populates `AbstractHandlerMapping.adaptedInterceptors`. When `getHandlerExecutionChain()` runs, it selects applicable ones.

**`MappedInterceptor`** wraps an interceptor with include/exclude URL patterns:
- Only added to chain if request URL matches include pattern AND doesn't match exclude pattern
- URL matching uses `PathMatcher` (AntPathMatcher by default)

---

### RequestMappingHandlerMapping — Complete Internal Analysis

This is the most important `HandlerMapping`. It powers all `@RequestMapping` methods.

#### Startup Phase — Building the Registry

```java
// AbstractHandlerMethodMapping.afterPropertiesSet()
// Called by Spring after bean initialization
@Override
public void afterPropertiesSet() {
    initHandlerMethods();
}

protected void initHandlerMethods() {
    // Scan all beans in the ApplicationContext
    for (String beanName : getCandidateBeanNames()) {
        if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
            processCandidateBean(beanName);
        }
    }
    handlerMethodsInitialized(getHandlerMethods());
}

protected void processCandidateBean(String beanName) {
    Class<?> beanType = null;
    try {
        beanType = obtainApplicationContext().getType(beanName);
    } catch (Throwable ex) { /* skip */ }

    if (beanType != null && isHandler(beanType)) {
        detectHandlerMethods(beanName);
    }
}

// RequestMappingHandlerMapping.isHandler():
@Override
protected boolean isHandler(Class<?> beanType) {
    return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
            AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
    // @Controller includes @RequestMapping as meta-annotation via @RestController
    // Also true for @Component if it has class-level @RequestMapping
}
```

#### Method-Level Scanning

```java
protected void detectHandlerMethods(Object handler) {
    Class<?> handlerType = (handler instanceof String beanName ?
        obtainApplicationContext().getType(beanName) :
        handler.getClass());

    if (handlerType != null) {
        Class<?> userType = ClassUtils.getUserClass(handlerType);
        // getUserClass: unwraps CGLIB proxy to get original class

        // Scan all methods — including inherited
        Map<Method, T> methods = MethodIntrospector.selectMethods(
            userType,
            (MethodIntrospector.MetadataLookup<T>) method -> {
                try {
                    return getMappingForMethod(method, userType);
                    // Returns RequestMappingInfo or null
                } catch (Throwable ex) {
                    throw new IllegalStateException(/*...*/);
                }
            });

        // Register each method
        methods.forEach((method, mapping) -> {
            Method invocableMethod =
                AopUtils.selectInvocableMethod(method, userType);
            registerHandlerMethod(handler, invocableMethod, mapping);
        });
    }
}
```

#### getMappingForMethod() — RequestMappingInfo Construction

```java
// RequestMappingHandlerMapping.getMappingForMethod()
@Override
@Nullable
protected RequestMappingInfo getMappingForMethod(
        Method method, Class<?> handlerType) {

    // Check method for @RequestMapping
    RequestMappingInfo info = createRequestMappingInfo(method);

    if (info != null) {
        // Check class for @RequestMapping (type-level)
        RequestMappingInfo typeInfo =
            createRequestMappingInfo(handlerType);

        if (typeInfo != null) {
            // COMBINE: class-level + method-level
            // URL: class prefix + method suffix
            // Methods: intersection of class and method methods
            // Headers, Params, Consumes, Produces: intersection
            info = typeInfo.combine(info);
        }

        // Set prefix from PathPrefix configuration
        String prefix = getPathPrefix(handlerType);
        if (prefix != null) {
            info = RequestMappingInfo.paths(prefix)
                .options(this.config).build().combine(info);
        }
    }

    return info;
}
```

#### The Mapping Registry

```java
// MappingRegistry — the heart of RequestMappingHandlerMapping
class MappingRegistry {

    // Primary registry
    private final Map<T, MappingRegistration<T>> registry = new HashMap<>();

    // Per-URL index for fast lookup
    private final MultiValueMap<String, T> pathLookup =
        new LinkedMultiValueMap<>();

    // For ambiguity detection
    private final Map<String, List<HandlerMethod>> nameLookup =
        new ConcurrentHashMap<>();

    // Cross-Origin config per handler method
    private final Map<HandlerMethod, CorsConfiguration> corsLookup =
        new ConcurrentHashMap<>();

    // READ-WRITE LOCK for concurrent access
    private final ReentrantReadWriteLock readWriteLock =
        new ReentrantReadWriteLock();
    // → Multiple threads can READ simultaneously
    // → Only one thread can WRITE (register new mappings)
    // → Registration happens at startup — reads are lock-free at runtime
}
```

---

### Per-Request Lookup — How getHandlerInternal() Works

```java
// AbstractHandlerMethodMapping.getHandlerInternal()
@Override
@Nullable
protected HandlerMethod getHandlerInternal(
        HttpServletRequest request) throws Exception {

    // STEP 1: Extract the lookup path from request
    String lookupPath = initLookupPath(request);
    // For /api/products/42 → lookupPath = "/api/products/42"
    // Strips context path, servlet path

    // Acquire READ lock — concurrent requests can all read simultaneously
    this.mappingRegistry.acquireReadLock();
    try {
        // STEP 2: Find the best matching handler method
        HandlerMethod handlerMethod = lookupHandlerMethod(
            lookupPath, request);

        // STEP 3: If found, ensure we use the actual bean instance
        // (not just the bean name stored in registry)
        return (handlerMethod != null ?
            handlerMethod.createWithResolvedBean() : null);

    } finally {
        this.mappingRegistry.releaseReadLock();
    }
}
```

#### lookupHandlerMethod() — The Core Matching Algorithm

```java
@Nullable
protected HandlerMethod lookupHandlerMethod(
        String lookupPath, HttpServletRequest request) throws Exception {

    List<Match> matches = new ArrayList<>();

    // STEP 1: Direct URL matches (fast path — no pattern matching)
    List<T> directPathMatches =
        this.mappingRegistry.getMappingsByDirectPath(lookupPath);
    // e.g., /products/list → exact match if registered

    if (directPathMatches != null) {
        addMatchingMappings(directPathMatches, matches, request);
    }

    // STEP 2: If no direct matches, try ALL pattern mappings
    if (matches.isEmpty()) {
        addMatchingMappings(
            this.mappingRegistry.getRegistrations().keySet(),
            matches,
            request
        );
        // This tries every registered RequestMappingInfo
        // O(n) in worst case — but typically fast due to pattern indexing
    }

    // STEP 3: No matches found
    if (matches.isEmpty()) {
        return handleNoMatch(
            this.mappingRegistry.getRegistrations().keySet(),
            lookupPath, request);
        // May throw exception or return null
    }

    // STEP 4: Sort matches by specificity — BEST MATCH WINS
    Match bestMatch = matches.get(0);
    if (matches.size() > 1) {
        Comparator<Match> comparator =
            new MatchComparator(getMappingComparator(request));
        matches.sort(comparator);
        bestMatch = matches.get(0);

        // STEP 5: Ambiguity check — are top two matches equally specific?
        Match secondBestMatch = matches.get(1);
        if (comparator.compare(bestMatch, secondBestMatch) == 0) {
            // Two equally-specific matches — AMBIGUOUS
            Method m1 = bestMatch.getHandlerMethod().getMethod();
            Method m2 = secondBestMatch.getHandlerMethod().getMethod();
            String uri = request.getRequestURI();
            throw new IllegalStateException(
                "Ambiguous handler methods mapped for '" + uri + "': {" +
                m1 + ", " + m2 + "}");
        }
    }

    // STEP 6: Set request attributes from best match
    request.setAttribute(BEST_MATCHING_HANDLER_ATTRIBUTE,
        bestMatch.getHandlerMethod());
    handleMatch(bestMatch.mapping, lookupPath, request);
    // → Sets URI_TEMPLATE_VARIABLES_ATTRIBUTE
    // → Sets BEST_MATCHING_PATTERN_ATTRIBUTE
    // → Sets MATRIX_VARIABLES_ATTRIBUTE
    // → Sets PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE

    return bestMatch.getHandlerMethod();
}
```

---

### RequestMappingInfo — What It Encapsulates

```java
public final class RequestMappingInfo {

    // Conditions that ALL must match for this mapping to apply:

    PathPatternsRequestCondition pathPatternsCondition;
    // → URL pattern matching (AntPath, URI templates, regex segments)
    // → /products/{id}, /products/*, /products/**

    RequestMethodsRequestCondition methodsCondition;
    // → HTTP methods: GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS

    ParamsRequestCondition paramsCondition;
    // → Query parameters: params="active=true", params="!deleted"

    HeadersRequestCondition headersCondition;
    // → HTTP headers: headers="X-API-Version=2"

    ConsumesRequestCondition consumesCondition;
    // → Request Content-Type: consumes="application/json"

    ProducesRequestCondition producesCondition;
    // → Response Accept: produces="application/json"

    RequestConditionHolder customConditionHolder;
    // → Custom RequestCondition implementations

    // MATCHING: ALL conditions must match
    // COMBINING: class-level + method-level combined
    // SPECIFICITY: more specific conditions win over less specific

    @Nullable
    RequestMappingInfo getMatchingCondition(HttpServletRequest request) {
        // Returns null if any condition doesn't match
        // Returns new RequestMappingInfo with matched conditions if all match
    }

    int compareTo(RequestMappingInfo other, HttpServletRequest request) {
        // Specificity comparison:
        // More specific URL > less specific
        // More specific HTTP method > no method
        // More specific produces > no produces
        // More specific params > no params
        // etc.
    }
}
```

---

### URL Pattern Matching — Deep Internals

Spring MVC supports three URL matching strategies:

#### 1. PathPattern (Modern — Default since Spring 5.3)

```java
// org.springframework.web.util.pattern.PathPattern
// Compiled at startup — very fast matching

Pattern         Matches                     Does NOT match
/products       /products                   /products/
/products/      /products/                  /products
/products/{id}  /products/42                /products/
                /products/abc               /products/42/detail
/products/*     /products/list              /products/list/detail
                /products/42                /products/
/products/**    /products/                  (not applicable — matches all)
                /products/list
                /products/list/detail
                /products/42/reviews/5
```

**PathPattern vs AntPathMatcher:**
- `PathPattern`: compiled at startup, character-by-character matching, O(n) where n=path length
- `AntPathMatcher`: string-based regex-like matching, slower but more flexible
- Spring 5.3+ uses `PathPattern` by default for `RequestMappingHandlerMapping`

#### 2. URI Template Variables

```java
// {id} captures the path segment
@GetMapping("/products/{id}")
public Product get(@PathVariable Long id) { ... }
// /products/42 → id=42
// /products/abc → id="abc" (type conversion happens in ArgumentResolver)

// {id:.+} captures including dots
@GetMapping("/files/{filename:.+}")
public Resource getFile(@PathVariable String filename) { ... }
// /files/report.pdf → filename="report.pdf"
// Without :.+ → filename="report" (dots truncate by default)

// Multiple variables
@GetMapping("/users/{userId}/orders/{orderId}")
public Order getOrder(@PathVariable Long userId,
                       @PathVariable Long orderId) { ... }

// Optional segment — Spring 5+
@GetMapping({"/products/{id}", "/products"})
// Two mappings — handles both with and without ID
```

#### 3. Regex Segments (Advanced)

```java
// {varName:regex} — must match regex
@GetMapping("/products/{id:\\d+}")
// /products/42 → MATCHES (digits only)
// /products/abc → NO MATCH (falls through to next mapping)

@GetMapping("/api/{version:v[0-9]+}/products")
// /api/v1/products → MATCHES, version="v1"
// /api/beta/products → NO MATCH
```

---

### Specificity Ranking — How Best Match Is Selected

When multiple mappings match a request, Spring selects the MOST SPECIFIC one:

```
Specificity Rules (in priority order):

1. URL pattern specificity:
   /products/42     >  /products/{id}   >  /products/*   >  /products/**
   (literal)           (template)          (single *)       (double **)

2. HTTP method specified > no HTTP method
   @GetMapping("/x")  >  @RequestMapping("/x")

3. params condition specified > no params
   params="type=premium"  >  no params condition

4. headers condition specified > no headers
   headers="X-API-V=2"  >  no headers condition

5. consumes condition specified > no consumes
   consumes="application/json"  >  no consumes

6. produces condition specified > no produces
   produces="application/json"  >  no produces

7. Method count (fewer mappings for same URL = more specific)

EXAMPLE:
@GetMapping("/products/{id}")          ← matches GET /products/42
@RequestMapping("/products/{id}")      ← also matches GET /products/42
Winner: @GetMapping — HTTP method specified wins over generic @RequestMapping
```

---

### Ambiguous Mapping Detection — When It Fires

```java
// Ambiguous mapping: two methods with IDENTICAL specificity for same request
// This is detected at RUNTIME (not startup) — first request to the URL triggers it

// CASE 1: Identical URL + method
@GetMapping("/products")
public List<Product> getAllProducts() { ... }

@GetMapping("/products")
public List<Product> getProductsV2() { ... }
// THROWS: IllegalStateException at startup during afterPropertiesSet()
// "Ambiguous mapping: cannot map 'productController' method...
//  to {GET /products}: There is already 'productController' bean method..."
// DETECTED AT STARTUP via MappingRegistry

// CASE 2: Apparently different but actually same pattern
@GetMapping("/products/{id}")
public Product getById(@PathVariable Long id) { ... }

@GetMapping("/products/{productId}")
public Product getByProductId(@PathVariable Long productId) { ... }
// THROWS: Same as Case 1 — {id} and {productId} are identical patterns
// Only variable name differs — the pattern is the same

// CASE 3: Not actually ambiguous — different conditions
@GetMapping(value="/products/{id}", produces="application/json")
public Product getJson(@PathVariable Long id) { ... }

@GetMapping(value="/products/{id}", produces="application/xml")
public Product getXml(@PathVariable Long id) { ... }
// VALID — produces condition differentiates them
// GET /products/42 with Accept: application/json → first method
// GET /products/42 with Accept: application/xml  → second method
```

---

### handleNoMatch() — When No Mapping Found

```java
// Called when lookupHandlerMethod() finds no matches
@Override
protected HandlerMethod handleNoMatch(
        Set<RequestMappingInfo> infos,
        String lookupPath,
        HttpServletRequest request) throws ServletException {

    PartialMatchHelper helper = new PartialMatchHelper(infos, request);

    // Is there a mapping for this URL but wrong HTTP method?
    if (helper.hasMethodsMismatch()) {
        Set<String> methods = helper.getAllowedMethods();
        if (HttpMethod.OPTIONS.matches(request.getMethod())) {
            // OPTIONS request — return allowed methods
            HttpOptionsHandler handler = new HttpOptionsHandler(methods);
            return new HandlerMethod(handler, HTTP_OPTIONS_HANDLE_METHOD);
        }
        throw new HttpRequestMethodNotSupportedException(
            request.getMethod(), methods);
        // → ResponseStatusExceptionResolver → 405 Method Not Allowed
    }

    // Is there a mapping for this URL but wrong consumes?
    if (helper.hasConsumesMismatch()) {
        Set<MediaType> mediaTypes = helper.getConsumableMediaTypes();
        throw new HttpMediaTypeNotSupportedException(
            request.getContentType(), new ArrayList<>(mediaTypes));
        // → 415 Unsupported Media Type
    }

    // Is there a mapping for this URL but wrong produces?
    if (helper.hasProducesMismatch()) {
        Set<MediaType> mediaTypes = helper.getProducibleMediaTypes();
        throw new HttpMediaTypeNotAcceptableException(
            new ArrayList<>(mediaTypes));
        // → 406 Not Acceptable
    }

    // Is there a mapping for this URL but wrong params/headers?
    if (helper.hasParamsMismatch()) {
        throw new UnsatisfiedServletRequestParameterException(
            helper.getParamConditions(),
            request.getParameterMap());
        // → 400 Bad Request
    }

    // No match at all
    return null;
    // → DispatcherServlet.noHandlerFound() → 404
}
```

**This is critical:** Spring gives specific error codes based on WHY matching failed:
- URL matches but wrong method → **405**
- URL matches but wrong Content-Type → **415**
- URL matches but wrong Accept → **406**
- URL matches but wrong params → **400**
- No URL match at all → **404**

---

### BeanNameUrlHandlerMapping — Internal Analysis

```java
// Legacy but still registered by default
public class BeanNameUrlHandlerMapping
    extends AbstractDetectingUrlHandlerMapping {

    @Override
    protected String[] determineUrlsForHandler(String beanName) {
        List<String> urls = new ArrayList<>();

        // Bean name starts with "/" → it's a URL mapping
        if (beanName.startsWith("/")) {
            urls.add(beanName);
        }

        // Also check aliases
        String[] aliases = obtainApplicationContext()
            .getAliases(beanName);
        for (String alias : aliases) {
            if (alias.startsWith("/")) {
                urls.add(alias);
            }
        }

        return StringUtils.toStringArray(urls);
    }
}

// Usage (rare today):
@Component("/products")  // Bean name IS the URL
public class ProductHandler implements HttpRequestHandler {
    @Override
    public void handleRequest(HttpServletRequest req,
                               HttpServletResponse res) throws IOException {
        res.getWriter().write("Products list");
    }
}
// GET /products → BeanNameUrlHandlerMapping finds "/products" bean
// HttpRequestHandlerAdapter invokes it
```

---

### SimpleUrlHandlerMapping — For Static Resources

```java
// Used internally for static resource handling
// Registered by WebMvcConfigurer.addResourceHandlers()

// Internally creates:
SimpleUrlHandlerMapping resourceMapping = new SimpleUrlHandlerMapping();
resourceMapping.setOrder(Integer.MAX_VALUE - 1);

Map<String, ResourceHttpRequestHandler> urlMap = new HashMap<>();
urlMap.put("/static/**", resourceHandler);
urlMap.put("/webjars/**", webjarHandler);

resourceMapping.setUrlMap(urlMap);
// URL patterns mapped directly to handler objects (not controller methods)
```

---

### HandlerExecutionChain — What It Contains

```java
public class HandlerExecutionChain {

    private final Object handler;
    // The actual handler: HandlerMethod, HttpRequestHandler, Controller, etc.

    private final List<HandlerInterceptor> interceptorList =
        new ArrayList<>();
    // Interceptors applicable to this request
    // In the ORDER they should execute preHandle()

    private int interceptorIndex = -1;
    // Tracks how far into the interceptor chain we got
    // Used by triggerAfterCompletion() to only run afterCompletion()
    // for interceptors whose preHandle() succeeded

    // Key methods:
    boolean applyPreHandle(HttpServletRequest req,
                            HttpServletResponse res) throws Exception {
        for (int i = 0; i < getInterceptors().length; i++) {
            HandlerInterceptor interceptor = getInterceptors()[i];
            if (!interceptor.preHandle(req, res, this.handler)) {
                // preHandle returned false → STOP
                triggerAfterCompletion(req, res, null);
                return false;
            }
            this.interceptorIndex = i;
            // Record how far we got — for afterCompletion cleanup
        }
        return true;
    }

    void applyPostHandle(HttpServletRequest req,
                          HttpServletResponse res,
                          @Nullable ModelAndView mv) throws Exception {
        for (int i = getInterceptors().length - 1; i >= 0; i--) {
            // REVERSE ORDER for postHandle
            HandlerInterceptor interceptor = getInterceptors()[i];
            interceptor.postHandle(req, res, this.handler, mv);
        }
    }

    void triggerAfterCompletion(HttpServletRequest req,
                                  HttpServletResponse res,
                                  @Nullable Exception ex) {
        for (int i = this.interceptorIndex; i >= 0; i--) {
            // REVERSE ORDER, only up to interceptorIndex
            // (only interceptors whose preHandle ran)
            HandlerInterceptor interceptor = getInterceptors()[i];
            try {
                interceptor.afterCompletion(req, res, this.handler, ex);
            } catch (Throwable ex2) {
                logger.error("...", ex2);
                // afterCompletion exceptions are SWALLOWED
                // Never propagated — too late in the cycle
            }
        }
    }
}
```

---

## 2️⃣ CODE EXAMPLES

### Custom RequestCondition — API Version Routing

```java
// Custom condition that matches X-API-Version header
public class ApiVersionCondition
    implements RequestCondition<ApiVersionCondition> {

    private final int apiVersion;

    public ApiVersionCondition(int version) {
        this.apiVersion = version;
    }

    @Override
    public ApiVersionCondition combine(ApiVersionCondition other) {
        // Method-level overrides class-level
        return new ApiVersionCondition(other.apiVersion);
    }

    @Override
    @Nullable
    public ApiVersionCondition getMatchingCondition(
            HttpServletRequest request) {
        String versionHeader = request.getHeader("X-API-Version");
        if (versionHeader != null) {
            int requestedVersion = Integer.parseInt(versionHeader);
            if (requestedVersion >= this.apiVersion) {
                return this; // Match
            }
        }
        return null; // No match
    }

    @Override
    public int compareTo(ApiVersionCondition other,
                          HttpServletRequest request) {
        // Higher version number = more specific = wins
        return other.apiVersion - this.apiVersion;
    }
}

// Custom HandlerMapping that uses this condition
@Component
public class VersionedRequestMappingHandlerMapping
    extends RequestMappingHandlerMapping {

    @Override
    protected RequestCondition<?> getCustomTypeCondition(
            Class<?> handlerType) {
        ApiVersion apiVersion =
            AnnotationUtils.findAnnotation(handlerType, ApiVersion.class);
        return apiVersion != null ?
            new ApiVersionCondition(apiVersion.value()) : null;
    }

    @Override
    protected RequestCondition<?> getCustomMethodCondition(
            Method method) {
        ApiVersion apiVersion =
            AnnotationUtils.findAnnotation(method, ApiVersion.class);
        return apiVersion != null ?
            new ApiVersionCondition(apiVersion.value()) : null;
    }
}

// Custom annotation
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface ApiVersion {
    int value();
}

// Usage
@RestController
@RequestMapping("/products")
@ApiVersion(1)
public class ProductControllerV1 {

    @GetMapping("/{id}")
    public Product getV1(@PathVariable Long id) {
        return productService.findById(id);
    }
}

@RestController
@RequestMapping("/products")
@ApiVersion(2)
public class ProductControllerV2 {

    @GetMapping("/{id}")
    public ProductDTO getV2(@PathVariable Long id) {
        return productService.findByIdAsDTO(id);
    }
}
// X-API-Version: 1 → routes to V1
// X-API-Version: 2 → routes to V2 (more specific — wins)
```

---

### Inspecting Registered Mappings at Runtime

```java
@RestController
@RequestMapping("/system")
public class MappingInspector {

    @Autowired
    private RequestMappingHandlerMapping handlerMapping;

    @GetMapping("/mappings")
    public Map<String, Object> mappings() {
        Map<RequestMappingInfo, HandlerMethod> map =
            handlerMapping.getHandlerMethods();

        Map<String, Object> result = new LinkedHashMap<>();
        map.forEach((info, method) -> {
            result.put(
                info.toString(),
                method.getShortLogMessage()
            );
        });
        return result;
    }

    @GetMapping("/handler")
    public String findHandler(HttpServletRequest request) throws Exception {
        HandlerExecutionChain chain =
            handlerMapping.getHandler(request);
        if (chain == null) return "No handler found";

        return "Handler: " + chain.getHandler() +
               ", Interceptors: " + chain.getInterceptorList().size();
    }
}
```

---

### URL Pattern Matching Edge Cases

```java
@RestController
@RequestMapping("/api")
public class PatternController {

    // CASE 1: Literal vs template — literal wins
    @GetMapping("/products/featured")    // Matches /api/products/featured
    public String featured() { return "featured"; }

    @GetMapping("/products/{id}")        // Matches /api/products/42
    public String byId(@PathVariable String id) { return "id=" + id; }
    // GET /api/products/featured → "featured" wins (literal more specific)
    // GET /api/products/42 → byId wins

    // CASE 2: Single wildcard vs double wildcard
    @GetMapping("/files/*")              // Matches /api/files/report
    public String singleWildcard() { return "single"; }

    @GetMapping("/files/**")             // Matches /api/files/a/b/c
    public String doubleWildcard() { return "double"; }
    // GET /api/files/report → "single" wins (* more specific than **)
    // GET /api/files/a/b/c → "double" wins (* doesn't match slashes)

    // CASE 3: Regex restriction
    @GetMapping("/products/{id:\\d+}")   // ONLY digits
    public String numericId(@PathVariable Long id) { return "numeric"; }

    @GetMapping("/products/{slug:[a-z-]+}") // ONLY lowercase + hyphens
    public String slugId(@PathVariable String slug) { return "slug"; }
    // GET /api/products/42 → numeric (matches \d+)
    // GET /api/products/my-product → slug (matches [a-z-]+)
    // GET /api/products/Product123 → 404 (matches neither)

    // CASE 4: Matrix variables (must enable)
    @GetMapping("/users/{userId}")
    public String withMatrix(
        @PathVariable String userId,
        @MatrixVariable(name="role", pathVar="userId") String role) {
        return userId + " role=" + role;
    }
    // GET /api/users/42;role=admin → userId=42, role=admin
}
```

---

### Programmatic Handler Registration (Runtime)

```java
// Register new handler methods dynamically at runtime
// Useful for plugin architectures
@Component
public class DynamicHandlerRegistrar {

    @Autowired
    private RequestMappingHandlerMapping handlerMapping;

    public void registerHandler(Object handler,
                                  String urlPattern,
                                  RequestMethod method) {
        // Build RequestMappingInfo programmatically
        RequestMappingInfo info = RequestMappingInfo
            .paths(urlPattern)
            .methods(method)
            .options(handlerMapping.getBuilderConfiguration())
            .build();

        // Find the actual method to register
        Method handlerMethod;
        try {
            handlerMethod = handler.getClass()
                .getMethod("handle", HttpServletRequest.class);
        } catch (NoSuchMethodException e) {
            throw new IllegalArgumentException("No handle() method", e);
        }

        // Register — acquires write lock on MappingRegistry
        handlerMapping.registerMapping(info, handler, handlerMethod);
    }

    public void unregisterHandler(String urlPattern,
                                    RequestMethod method) {
        RequestMappingInfo info = RequestMappingInfo
            .paths(urlPattern)
            .methods(method)
            .options(handlerMapping.getBuilderConfiguration())
            .build();

        handlerMapping.unregisterMapping(info);
    }
}
```

---

### Edge Case — Class-Level + Method-Level Combining

```java
@RestController
@RequestMapping(
    value = "/api/v1",
    consumes = "application/json",
    produces = "application/json"
)
public class ApiController {

    // Combined result:
    // URL:      /api/v1/products
    // Method:   POST
    // Consumes: application/json (inherited from class)
    // Produces: application/json (inherited from class)
    @PostMapping("/products")
    public Product create(@RequestBody Product product) {
        return productService.save(product);
    }

    // Combined result:
    // URL:      /api/v1/products/export
    // Method:   GET
    // Consumes: */* (OVERRIDES class-level — method-level wins for consumes)
    // Produces: text/csv (OVERRIDES class-level — method-level wins)
    @GetMapping(value = "/products/export",
                produces = "text/csv")
    public ResponseEntity<Resource> export() {
        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType("text/csv"))
            .body(exportService.exportToCsv());
    }
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
`RequestMappingHandlerMapping` builds its registry during which lifecycle phase?

A) During `context.refresh()` Phase 5 — `invokeBeanFactoryPostProcessors()`
B) During `context.refresh()` Phase 11 — `finishBeanFactoryInitialization()`, specifically in `afterPropertiesSet()`
C) On first HTTP request — lazy initialisation
D) During `DispatcherServlet.initStrategies()`

**Answer: B**
`RequestMappingHandlerMapping` is a `InitializingBean`. Its `afterPropertiesSet()` calls `initHandlerMethods()` which scans all `@Controller` beans. This happens during Phase 11 when the bean is instantiated and initialised. `initStrategies()` (D) only STORES the already-built mapping object — it doesn't trigger registry building.

---

**Q2 — Select All That Apply**
A `GET /products/42` request arrives. Which `RequestMappingInfo` attributes are set on the `HttpServletRequest` by `RequestMappingHandlerMapping` when a match is found?

A) `URI_TEMPLATE_VARIABLES_ATTRIBUTE` with `{id: "42"}`
B) `BEST_MATCHING_PATTERN_ATTRIBUTE` with `/products/{id}`
C) `HANDLER_METHOD_ATTRIBUTE` with the resolved `HandlerMethod`
D) `BEST_MATCHING_HANDLER_ATTRIBUTE` with the resolved `HandlerMethod`
E) `PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE`

**Answer: A, B, D, E**
`URI_TEMPLATE_VARIABLES_ATTRIBUTE` stores extracted path variables (A). `BEST_MATCHING_PATTERN_ATTRIBUTE` stores the matched pattern (B). `BEST_MATCHING_HANDLER_ATTRIBUTE` stores the `HandlerMethod` (D). `PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE` stores what the handler can produce (E). `HANDLER_METHOD_ATTRIBUTE` (C) is not a standard constant — distractor.

---

**Q3 — Code Prediction**
```java
@RestController
@RequestMapping("/api")
public class TestController {

    @GetMapping("/items/{id}")
    public String byId(@PathVariable String id) {
        return "id=" + id;
    }

    @GetMapping("/items/featured")
    public String featured() {
        return "featured";
    }
}
```
`GET /api/items/featured` — which method is called?

A) `byId()` with `id="featured"`
B) `featured()`
C) `IllegalStateException` — ambiguous mapping
D) `404 Not Found`

**Answer: B**
Both mappings match `/api/items/featured`. `featured()` maps to the literal `/items/featured`. `byId()` maps to the template `/items/{id}`. Literal paths are MORE SPECIFIC than URI templates. Spring's specificity ranking selects `featured()`. This is a fundamental specificity rule.

---

**Q4 — MCQ**
A `POST /products` request arrives. A mapping exists for `GET /products`. No mapping exists for `POST /products`. What exception does `handleNoMatch()` throw?

A) `NoHandlerFoundException`
B) `HttpRequestMethodNotSupportedException`
C) `HttpMediaTypeNotSupportedException`
D) `MethodArgumentNotValidException`

**Answer: B**
`handleNoMatch()` detects that a mapping exists for the URL `/products` but with a different HTTP method. It throws `HttpRequestMethodNotSupportedException` with the allowed methods (GET in this case). `DefaultHandlerExceptionResolver` converts this to HTTP 405 Method Not Allowed with an `Allow: GET` header.

---

**Q5 — Select All That Apply**
Which of the following cause `IllegalStateException: Ambiguous handler methods` to be thrown?

A) Two methods with `@GetMapping("/products/{id}")` and `@GetMapping("/products/{productId}")`
B) Two methods with `@GetMapping(value="/products/{id}", produces="application/json")` and `@GetMapping(value="/products/{id}", produces="application/xml")`
C) Two methods with `@GetMapping("/products")` in the same controller
D) Two methods with `@GetMapping("/products")` in different controllers
E) Two methods with `@GetMapping("/products/*")` and `@GetMapping("/products/{id}")`

**Answer: A, C, D**
A: `{id}` and `{productId}` are identical URL patterns — ambiguous. C: Two identical mappings in same controller — ambiguous. D: Two identical mappings in different controllers — still ambiguous — Spring's registry is global. B: Different `produces` conditions differentiate them — NOT ambiguous. E: `/*` and `/{id}` have different specificity — `/{id}` wins — NOT ambiguous.

---

**Q6 — True/False**
`HandlerExecutionChain.triggerAfterCompletion()` calls `afterCompletion()` on ALL registered interceptors, including those whose `preHandle()` returned `false`.

**Answer: False**
`triggerAfterCompletion()` only calls `afterCompletion()` for interceptors whose `preHandle()` successfully returned `true`. The `interceptorIndex` field tracks how far the `preHandle` chain got. Iteration in `triggerAfterCompletion()` goes from `interceptorIndex` down to 0 — interceptors that never had `preHandle()` called (because an earlier one returned false) never have `afterCompletion()` called.

---

**Q7 — Scenario**
A developer registers two `HandlerMapping` beans: one at order `0` (custom) and one at order `1` (the default `RequestMappingHandlerMapping`). The custom mapping returns a handler for `/api/**`. A request arrives for `/api/products`. The custom handler processes it successfully. What happens?

A) Both handlers are invoked — results merged
B) Only the custom handler is invoked — first match wins
C) `RequestMappingHandlerMapping` is always consulted first regardless of order
D) `IllegalStateException` — multiple handlers found

**Answer: B**
`DispatcherServlet.getHandler()` iterates `HandlerMapping` list in order. First `HandlerMapping` that returns non-null wins. The custom mapping at order 0 returns a handler → `RequestMappingHandlerMapping` is never consulted for this request. This is the "first match wins" rule.

---

**Q8 — MCQ**
`@GetMapping(value="/products/{id:\\d+}")` is registered. A request arrives for `GET /products/abc`. What does Spring MVC return?

A) The handler is invoked with `id="abc"`
B) `400 Bad Request` — regex validation failed
C) `handleNoMatch()` is called — 404 if no other mapping matches
D) `IllegalArgumentException` — type conversion fails

**Answer: C**
The regex `\\d+` means digits only. `/products/abc` does NOT match this pattern. `RequestMappingHandlerMapping` does not find a match. `handleNoMatch()` is called. If no other mapping covers `/products/abc`, `null` is returned, DispatcherServlet calls `noHandlerFound()` → 404. The regex is evaluated at URL MATCHING time, not at argument resolution time.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — "Ambiguity is detected at startup"**
Ambiguity detection in `RequestMappingHandlerMapping` is PARTIALLY at startup. Exact duplicate mappings (same URL + method) ARE detected at startup during `registerHandlerMethod()`. But some ambiguities involving complex condition matching are detected at request time — specifically when two mappings have equal specificity for the same request. Exam questions sometimes say "throws at startup" for ambiguities that actually throw at request time.

**Trap 2 — URI template variables vs literal — always literal wins**
`/products/featured` vs `/products/{id}` — literal ALWAYS wins regardless of order of registration. This is a specificity rule, not a first-registered rule. Many developers assume the order of `@GetMapping` declarations matters — it does NOT for specificity.

**Trap 3 — `{id}` and `{productId}` are identical patterns**
The VARIABLE NAME in URI templates is irrelevant to pattern matching. `/products/{id}` and `/products/{productId}` are the SAME pattern. Registering both causes `IllegalStateException`. Only the regex constraint (if different) would differentiate them: `/products/{id:\\d+}` vs `/products/{slug:[a-z]+}`.

**Trap 4 — afterCompletion swallows exceptions**
Inside `triggerAfterCompletion()`, each `afterCompletion()` call is wrapped in try-catch. Exceptions from `afterCompletion()` are logged but NOT propagated. This means even if `afterCompletion()` throws a `RuntimeException`, the original response is still committed. Developers relying on `afterCompletion()` exception propagation for error handling are surprised when it doesn't work.

**Trap 5 — handleNoMatch() gives specific status codes**
Many developers assume "no mapping found = 404". Wrong. `handleNoMatch()` returns:
- URL found, wrong method → **405** (with Allow header)
- URL found, wrong Content-Type → **415**
- URL found, wrong Accept → **406**
- URL found, wrong params → **400**
- URL not found at all → **404**

Exam questions present specific scenarios and ask which status code is returned.

**Trap 6 — MappedInterceptor URL matching uses PathMatcher**
Interceptors added with `.addPathPatterns("/api/**")` use `PathMatcher` (AntPathMatcher by default, or PathPattern if configured). The pattern `/api/**` matches `/api/products` and `/api/products/42` but NOT `/api` (without trailing content if configured strictly). Exclusion patterns `.excludePathPatterns("/api/public/**")` run AFTER inclusion — exclusion wins.

**Trap 7 — detectAllHandlerMappings and ancestor contexts**
With `detectAllHandlerMappings=true` (default), `BeanFactoryUtils.beansOfTypeIncludingAncestors()` searches BOTH child context AND root context. `HandlerMapping` beans in the root context ARE found and used. This is unexpected — you'd expect root context to not have web-layer beans. But technically valid. A `HandlerMapping` accidentally placed in root context will be found and used.

---

## 5️⃣ SUMMARY SHEET

```
HANDLERMAPPING HIERARCHY
─────────────────────────────────────────────────────
AbstractHandlerMapping (template getHandler())
  ├── AbstractUrlHandlerMapping
  │     ├── SimpleUrlHandlerMapping   (URL → handler object)
  │     └── BeanNameUrlHandlerMapping (bean name "/" → handler)
  └── AbstractHandlerMethodMapping
        └── RequestMappingHandlerMapping  ← PRIMARY
              (@RequestMapping → HandlerMethod registry)

HANDLERMAPPING CHAIN ORDER (default @EnableWebMvc)
─────────────────────────────────────────────────────
Order 0:    RequestMappingHandlerMapping  (@RequestMapping)
Order 1:    BeanNameUrlHandlerMapping     (bean name = URL)
Order 2:    RouterFunctionMapping         (functional)
Order MAX-1: SimpleUrlHandlerMapping     (static resources)
Order MAX:  DefaultServletHandlerMapping  (container default)
RULE: First non-null return WINS — remaining mappings not consulted

REGISTRY BUILDING (RequestMappingHandlerMapping)
─────────────────────────────────────────────────────
Phase: afterPropertiesSet() during Phase 11 of context.refresh()
Scans: ALL @Controller beans in context + ancestors
Per bean: reflects over methods, finds @RequestMapping
Creates: RequestMappingInfo per method
Combines: class-level @RequestMapping + method-level @RequestMapping
Stores: Map<RequestMappingInfo, HandlerMethod> with read-write lock

PER-REQUEST LOOKUP ALGORITHM
─────────────────────────────────────────────────────
1. Extract lookupPath from request
2. Acquire READ lock (concurrent reads OK)
3. Fast path: direct URL lookup (exact match)
4. Slow path: iterate all patterns (O(n))
5. Sort matches by SPECIFICITY
6. Ambiguity check: top two equally specific → IllegalStateException
7. Set request attributes: URI_TEMPLATE_VARIABLES, BEST_MATCHING_PATTERN, etc.
8. Release READ lock
9. Return HandlerMethod

SPECIFICITY RANKING (highest to lowest)
─────────────────────────────────────────────────────
/products/featured   > /products/{id}  > /products/*  > /products/**
@GetMapping("/x")    > @RequestMapping("/x")  (method specified wins)
params="type=X"      > no params condition
headers="X-V=2"      > no headers condition
consumes="app/json"  > no consumes condition
produces="app/json"  > no produces condition

HANDLERNOMATCH STATUS CODES
─────────────────────────────────────────────────────
URL exists, wrong HTTP method   → 405 Method Not Allowed
URL exists, wrong Content-Type  → 415 Unsupported Media Type
URL exists, wrong Accept        → 406 Not Acceptable
URL exists, wrong params/headers → 400 Bad Request
URL not found at all            → 404 Not Found (via noHandlerFound())

URI TEMPLATE RULES
─────────────────────────────────────────────────────
{id}        = any single path segment (no slashes)
{id:.+}     = any path including dots
{id:\\d+}   = digits only (regex)
{a}/{b}     = two separate segments
/**         = any path including slashes
{id} vs {productId} = IDENTICAL PATTERN — causes ambiguous mapping

HANDLEREXECUTIONCHAIN ASSEMBLY
─────────────────────────────────────────────────────
AbstractHandlerMapping.getHandlerExecutionChain():
  Iterates adaptedInterceptors:
  → MappedInterceptor: included only if URL matches include/exclude patterns
  → Regular HandlerInterceptor: always included
  Added in registration order → preHandle executes in that order
  postHandle/afterCompletion execute in REVERSE order

afterCompletion():
  → Only called for interceptors whose preHandle() returned true
  → Exceptions SWALLOWED (logged only — never propagated)
  → interceptorIndex tracks furthest successful preHandle

AMBIGUOUS MAPPING DETECTION
─────────────────────────────────────────────────────
Exact duplicates          → detected at STARTUP (registerHandlerMethod)
Equal-specificity matches → detected at REQUEST TIME (lookupHandlerMethod)
{id} vs {name}            → identical patterns → ambiguous at startup
Different produces        → NOT ambiguous (conditions differentiate)

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "HandlerMapping returns HandlerExecutionChain — handler + applicable interceptors"
• "First HandlerMapping to return non-null wins — chain not exhausted"
• "Literal /products/featured beats template /products/{id} — always"
• "{id} and {productId} are identical patterns — causes ambiguous mapping"
• "handleNoMatch() gives 405/415/406/400 BEFORE giving 404"
• "afterCompletion() swallows exceptions — never propagated"
• "afterCompletion() only runs for interceptors whose preHandle() succeeded"
• "Registry built in afterPropertiesSet() Phase 11 — not in initStrategies()"
• "detectAllHandlerMappings searches root context too — unexpected but valid"
• "MappedInterceptor URL matching: include patterns checked, then exclude patterns"
```

---

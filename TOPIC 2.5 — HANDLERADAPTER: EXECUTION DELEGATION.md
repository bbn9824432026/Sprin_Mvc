# TOPIC 2.5 — HANDLERADAPTER: EXECUTION DELEGATION

---

## 1️⃣ CONCEPTUAL EXPLANATION

### Why HandlerAdapter Exists — The Adapter Pattern at Scale

`DispatcherServlet` does not know what type of handler object `HandlerMapping` returns. The handler could be a `HandlerMethod` (from `@RequestMapping`), an `HttpRequestHandler` (for resources), a legacy `Controller` interface implementation, or a `HandlerFunction` (functional endpoints). Each requires a completely different invocation mechanism.

Without the Adapter pattern, `DispatcherServlet` would need an `instanceof` chain growing with every new handler type — violating Open/Closed principle and coupling the dispatcher to handler implementations.

`HandlerAdapter` solves this cleanly:

```
DispatcherServlet asks:
  "Who can handle this handler object?"

HandlerAdapter chain answers:
  RequestMappingHandlerAdapter: "I can — it's a HandlerMethod"
  → supports(handler) = (handler instanceof HandlerMethod)

  HttpRequestHandlerAdapter: "I can — it's an HttpRequestHandler"
  → supports(handler) = (handler instanceof HttpRequestHandler)

  SimpleControllerHandlerAdapter: "I can — it's a Controller"
  → supports(handler) = (handler instanceof Controller)
```

DispatcherServlet uses the FIRST adapter that returns `true` from `supports()`. Then calls `handle()` on it.

---

### The HandlerAdapter Interface — Precise Contract

```java
public interface HandlerAdapter {

    // Can this adapter invoke the given handler?
    // Called by DispatcherServlet to select correct adapter
    boolean supports(Object handler);

    // Invoke the handler — return ModelAndView or null
    // null return means: response already written (e.g., @ResponseBody)
    @Nullable
    ModelAndView handle(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler
    ) throws Exception;

    // For HTTP cache support — Last-Modified header
    // Return -1 to indicate not supported
    // Deprecated since Spring 5.3 — use CacheControl instead
    @SuppressWarnings("deprecation")
    long getLastModified(HttpServletRequest request, Object handler);
}
```

---

### The Four Default HandlerAdapters

```
Adapter                          Supports                     Order
─────────────────────────────────────────────────────────────────────
RequestMappingHandlerAdapter   → HandlerMethod               Ordered.LOWEST_PRECEDENCE - 10
                                 (from @RequestMapping)

HttpRequestHandlerAdapter      → HttpRequestHandler           Ordered.LOWEST_PRECEDENCE - 10
                                 (static resources, custom)

SimpleControllerHandlerAdapter → Controller interface          Ordered.LOWEST_PRECEDENCE - 10
                                 (legacy Spring MVC pattern)

HandlerFunctionAdapter         → HandlerFunction               Ordered.LOWEST_PRECEDENCE - 10
                                 (functional WebMvc endpoints)
```

All have the same order value — `DispatcherServlet` finds them in the order they appear in the list, which is the order they were registered.

---

### RequestMappingHandlerAdapter — Complete Architecture

This is the most complex class in Spring MVC. Its `handle()` method orchestrates the entire controller method invocation pipeline.

```java
public class RequestMappingHandlerAdapter
    extends AbstractMessageConverterMethodExceptionHandler
    implements HandlerAdapter, InitializingBean, BeanFactoryAware {

    // Core components — all initialised in afterPropertiesSet()

    // Argument resolution chain
    private List<HandlerMethodArgumentResolver> argumentResolvers;
    private List<HandlerMethodArgumentResolver> initBinderArgumentResolvers;

    // Return value handling chain
    private List<HandlerMethodReturnValueHandler> returnValueHandlers;

    // Message converters — used by argument resolvers & return value handlers
    private List<HttpMessageConverter<?>> messageConverters;

    // Session/model management
    private SessionAttributeStore sessionAttributeStore;
    private ParameterNameDiscoverer parameterNameDiscoverer;

    // Binding initialisation
    private WebBindingInitializer webBindingInitializer;

    // Async support
    private AsyncTaskExecutor taskExecutor;
    private Long asyncRequestTimeout;

    // @ControllerAdvice beans — collected at startup
    private List<ControllerAdviceBean> controllerAdviceBeans;
}
```

#### afterPropertiesSet() — Complete Initialisation

```java
@Override
public void afterPropertiesSet() {

    // STEP 1: Process @ControllerAdvice beans
    // Find all @ControllerAdvice beans in ApplicationContext
    // Extract @InitBinder methods → global binders
    // Extract @ModelAttribute methods → global model populators
    // Extract @ExceptionHandler methods → global exception handlers
    initControllerAdviceCache();

    // STEP 2: Build argument resolver list
    if (this.argumentResolvers == null) {
        List<HandlerMethodArgumentResolver> resolvers =
            getDefaultArgumentResolvers();
        this.argumentResolvers = new HandlerMethodArgumentResolverComposite()
            .addResolvers(resolvers);
    }

    // STEP 3: Build @InitBinder argument resolvers
    // (Subset — not all argument types valid in @InitBinder)
    if (this.initBinderArgumentResolvers == null) {
        List<HandlerMethodArgumentResolver> resolvers =
            getDefaultInitBinderArgumentResolvers();
        this.initBinderArgumentResolvers =
            new HandlerMethodArgumentResolverComposite()
                .addResolvers(resolvers);
    }

    // STEP 4: Build return value handler list
    if (this.returnValueHandlers == null) {
        List<HandlerMethodReturnValueHandler> handlers =
            getDefaultReturnValueHandlers();
        this.returnValueHandlers =
            new HandlerMethodReturnValueHandlerComposite()
                .addHandlers(handlers);
    }
}
```

---

### handle() — The Complete Execution Flow

```java
// RequestMappingHandlerAdapter.handle()
@Override
@Nullable
public final ModelAndView handle(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler) throws Exception {

    HandlerMethod handlerMethod = (HandlerMethod) handler;

    // Check for async processing already in progress
    checkRequest(request);

    // SessionAttributes management
    // Lock session for @SessionAttributes processing
    // (prevents concurrent modifications to session attrs)
    if (synchronizeOnSession) {
        HttpSession session = request.getSession(false);
        if (session != null) {
            Object mutex = WebUtils.getSessionMutex(session);
            synchronized (mutex) {
                return invokeHandlerMethod(request, response, handlerMethod);
            }
        }
    }

    return invokeHandlerMethod(request, response, handlerMethod);
}
```

#### invokeHandlerMethod() — The Core

```java
@Nullable
protected ModelAndView invokeHandlerMethod(
        HttpServletRequest request,
        HttpServletResponse response,
        HandlerMethod handlerMethod) throws Exception {

    // STEP 1: Wrap request/response in Servlet-specific wrappers
    ServletWebRequest webRequest =
        new ServletWebRequest(request, response);

    try {
        // STEP 2: Create WebDataBinderFactory
        // Collects @InitBinder methods from:
        //   a) Current @Controller class
        //   b) Applicable @ControllerAdvice beans (ordered by @Order)
        WebDataBinderFactory binderFactory =
            getDataBinderFactory(handlerMethod);

        // STEP 3: Create ModelFactory
        // Collects @ModelAttribute methods from:
        //   a) Current @Controller class
        //   b) Applicable @ControllerAdvice beans
        ModelFactory modelFactory =
            getModelFactory(handlerMethod, binderFactory);

        // STEP 4: Create the invocable handler method
        ServletInvocableHandlerMethod invocableMethod =
            createInvocableHandlerMethod(handlerMethod);
        if (this.argumentResolvers != null) {
            invocableMethod.setHandlerMethodArgumentResolvers(
                this.argumentResolvers);
        }
        if (this.returnValueHandlers != null) {
            invocableMethod.setHandlerMethodReturnValueHandlers(
                this.returnValueHandlers);
        }
        invocableMethod.setDataBinderFactory(binderFactory);
        invocableMethod.setParameterNameDiscoverer(
            this.parameterNameDiscoverer);

        // STEP 5: Create ModelAndViewContainer
        // Holds the Model + ViewName + response-written flag
        ModelAndViewContainer mavContainer =
            new ModelAndViewContainer();

        // Copy flash attributes into model
        mavContainer.addAllAttributes(
            RequestContextUtils.getInputFlashMap(request));

        // STEP 6: Execute @ModelAttribute methods
        // These populate the model BEFORE the handler method runs
        modelFactory.initModel(webRequest, mavContainer, invocableMethod);

        // STEP 7: Set default view name from translator
        // (Used if handler returns void/null)
        mavContainer.setIgnoreDefaultModelOnRedirect(
            this.ignoreDefaultModelOnRedirect);

        // STEP 8: Check for async result from previous partial processing
        AsyncWebRequest asyncWebRequest =
            WebAsyncUtils.createAsyncWebRequest(request, response);
        asyncWebRequest.setTimeout(this.asyncRequestTimeout);

        WebAsyncManager asyncManager =
            WebAsyncUtils.getAsyncManager(request);
        asyncManager.setTaskExecutor(this.taskExecutor);
        asyncManager.setAsyncWebRequest(asyncWebRequest);
        asyncManager.registerCallableInterceptors(
            this.callableInterceptors);
        asyncManager.registerDeferredResultInterceptors(
            this.deferredResultInterceptors);

        if (asyncManager.hasConcurrentResult()) {
            // Async result available — process it
            Object result = asyncManager.getConcurrentResult();
            mavContainer = (ModelAndViewContainer)
                asyncManager.getConcurrentResultContext()[0];
            asyncManager.clearConcurrentResult();
            invocableMethod = invocableMethod
                .wrapConcurrentResult(result);
        }

        // STEP 9: INVOKE THE HANDLER METHOD
        invocableMethod.invokeAndHandle(webRequest, mavContainer);

        // STEP 10: Return null if async started
        if (asyncManager.isConcurrentHandlingStarted()) {
            return null;
        }

        // STEP 11: Build ModelAndView from container
        return getModelAndView(mavContainer, modelFactory, webRequest);

    } finally {
        webRequest.requestCompleted();
    }
}
```

---

### ServletInvocableHandlerMethod — The Actual Invocation

This class handles the actual method invocation with argument resolution and return value handling:

```java
// ServletInvocableHandlerMethod.invokeAndHandle()
public void invokeAndHandle(
        ServletWebRequest webRequest,
        ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {

    // STEP 1: Resolve arguments and invoke the method
    Object returnValue = invokeForRequest(webRequest, mavContainer,
        providedArgs);

    // STEP 2: Set response status from @ResponseStatus if present
    setResponseStatus(webRequest);

    // STEP 3: Check if we should handle return value
    if (returnValue == null) {
        if (isRequestNotModified(webRequest) ||
            getResponseStatus() != null ||
            mavContainer.isRequestHandled()) {
            disableContentCachingIfNecessary(webRequest);
            mavContainer.setRequestHandled(true);
            return; // Done — no return value processing
        }
    } else if (StringUtils.hasText(getResponseStatusReason())) {
        mavContainer.setRequestHandled(true);
        return;
    }

    // STEP 4: Mark container as NOT request-handled
    // (Return value handler will mark it handled if @ResponseBody)
    mavContainer.setRequestHandled(false);
    Assert.state(this.returnValueHandlers != null, "...");

    // STEP 5: Handle the return value
    try {
        this.returnValueHandlers.handleReturnValue(
            returnValue,
            getReturnValueType(returnValue),
            mavContainer,
            webRequest
        );
    } catch (Exception ex) {
        throw ex;
    }
}
```

#### invokeForRequest() — Argument Resolution + Reflection Call

```java
// InvocableHandlerMethod.invokeForRequest()
@Nullable
public Object invokeForRequest(
        NativeWebRequest request,
        @Nullable ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {

    // RESOLVE ALL METHOD ARGUMENTS
    Object[] args = getMethodArgumentValues(
        request, mavContainer, providedArgs);

    // INVOKE via reflection
    return doInvoke(args);
}

protected Object[] getMethodArgumentValues(
        NativeWebRequest request,
        @Nullable ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {

    MethodParameter[] parameters = getMethodParameters();

    if (ObjectUtils.isEmpty(parameters)) {
        return EMPTY_ARGS;
    }

    Object[] args = new Object[parameters.length];

    for (int i = 0; i < parameters.length; i++) {
        MethodParameter parameter = parameters[i];
        parameter.initParameterNameDiscovery(
            this.parameterNameDiscoverer);

        // Check if argument was provided externally
        args[i] = findProvidedArgument(parameter, providedArgs);
        if (args[i] != null) {
            continue;
        }

        // Find a resolver that supports this parameter
        if (!this.resolvers.supportsParameter(parameter)) {
            throw new IllegalStateException(formatArgumentError(parameter,
                "No suitable resolver"));
        }

        // Resolve the argument
        try {
            args[i] = this.resolvers.resolveArgument(
                parameter, mavContainer, request,
                this.dataBinderFactory);
        } catch (Exception ex) {
            throw ex;
        }
    }

    return args;
}
```

---

### HandlerMethodArgumentResolver — The Complete Chain

The argument resolver chain is **ordered** — first resolver that `supportsParameter()` returns `true` wins.

```java
public interface HandlerMethodArgumentResolver {

    // Does this resolver handle this parameter type/annotation?
    boolean supportsParameter(MethodParameter parameter);

    // Resolve the actual argument value from the request
    @Nullable
    Object resolveArgument(
        MethodParameter parameter,
        @Nullable ModelAndViewContainer mavContainer,
        NativeWebRequest webRequest,
        @Nullable WebDataBinderFactory binderFactory
    ) throws Exception;
}
```

**The complete default resolver chain (in order):**

```
GROUP 1: Annotation-based resolvers

  RequestParamMethodArgumentResolver (supports=false)
    → @RequestParam on simple types (String, int, etc.)
    → Also handles MultipartFile, Part

  RequestParamMapMethodArgumentResolver
    → @RequestParam on Map (gets ALL params as map)

  PathVariableMethodArgumentResolver
    → @PathVariable on simple types
    → Reads from URI_TEMPLATE_VARIABLES_ATTRIBUTE

  PathVariableMapMethodArgumentResolver
    → @PathVariable on Map (gets ALL path vars as map)

  MatrixVariableMethodArgumentResolver
    → @MatrixVariable on simple types
    → Reads from MATRIX_VARIABLES_ATTRIBUTE

  MatrixVariableMapMethodArgumentResolver
    → @MatrixVariable on Map

  ServletModelAttributeMethodProcessor (supports=false)
    → @ModelAttribute on non-simple types
    → Data binding + validation

  RequestResponseBodyMethodProcessor
    → @RequestBody
    → Reads request body via HttpMessageConverter chain
    → Deserialises JSON/XML to Java object

  RequestPartMethodArgumentResolver
    → @RequestPart (file upload + body combined)

  RequestHeaderMethodArgumentResolver
    → @RequestHeader on simple types

  RequestHeaderMapMethodArgumentResolver
    → @RequestHeader on Map/HttpHeaders

  ServletCookieValueMethodArgumentResolver
    → @CookieValue

  ExpressionValueMethodArgumentResolver
    → @Value (Spring EL expressions)

  SessionAttributeMethodArgumentResolver
    → @SessionAttribute

  RequestAttributeMethodArgumentResolver
    → @RequestAttribute

GROUP 2: Non-annotation type-based resolvers

  ServletRequestMethodArgumentResolver
    → HttpServletRequest, WebRequest, NativeWebRequest
    → InputStream, Reader
    → HttpMethod, Locale, TimeZone, ZoneId
    → Principal

  ServletResponseMethodArgumentResolver
    → HttpServletResponse, OutputStream, Writer

  HttpEntityMethodProcessor
    → HttpEntity<T>, RequestEntity<T>
    → Reads body via HttpMessageConverter

  RedirectAttributesMethodArgumentResolver
    → RedirectAttributes

  ModelMethodProcessor
    → Model, ModelMap

  MapMethodProcessor
    → Map (as model)

  ErrorsMethodArgumentResolver
    → Errors, BindingResult
    → Must come AFTER the model attribute it corresponds to

  SessionStatusMethodArgumentResolver
    → SessionStatus

  UriComponentsBuilderMethodArgumentResolver
    → UriComponentsBuilder, ServletUriComponentsBuilder

GROUP 3: Custom resolvers
    → Added via WebMvcConfigurer.addArgumentResolvers()
    → Added AFTER all defaults (cannot replace defaults)

GROUP 4: Final catch-all resolvers

  RequestParamMethodArgumentResolver (supports=true)
    → Simple types WITHOUT @RequestParam annotation
    → Last resort for unresolved simple parameters

  ServletModelAttributeMethodProcessor (supports=true)
    → Non-simple types WITHOUT @ModelAttribute annotation
    → Last resort for unresolved complex parameters
```

**The "supports=false/true" dual registration pattern:**
`RequestParamMethodArgumentResolver` is registered TWICE:
1. First registration: `supports=false` → only handles `@RequestParam` annotated params
2. Second registration (at end): `supports=true` → handles ANY simple type parameter even without annotation

This means `public String method(String name)` works — `name` is resolved from request param `name` even without `@RequestParam`.

---

### HandlerMethodReturnValueHandler — The Complete Chain

```java
public interface HandlerMethodReturnValueHandler {

    boolean supportsReturnType(MethodParameter returnType);

    void handleReturnValue(
        @Nullable Object returnValue,
        MethodParameter returnType,
        ModelAndViewContainer mavContainer,
        NativeWebRequest webRequest
    ) throws Exception;
}
```

**Complete default return value handler chain (in order):**

```
ModelAndViewMethodReturnValueHandler
  → Returns ModelAndView directly

ModelMethodProcessor
  → Returns Model (unusual — treated as model contribution)

ViewMethodReturnValueHandler
  → Returns View object directly

ResponseBodyEmitterReturnValueHandler
  → Returns ResponseBodyEmitter, SseEmitter, StreamingResponseBody
  → Async streaming

StreamingResponseBodyReturnValueHandler
  → Returns StreamingResponseBody
  → Streams to response directly

HttpEntityMethodProcessor
  → Returns HttpEntity<T>, ResponseEntity<T>
  → Sets status, headers, writes body via MessageConverter

HttpHeadersReturnValueHandler
  → Returns HttpHeaders
  → Writes headers only, no body

CallableMethodReturnValueHandler
  → Returns Callable<T>
  → Async processing via task executor

DeferredResultMethodReturnValueHandler
  → Returns DeferredResult<T>, ListenableFuture<T>
  → Async long-polling

AsyncTaskMethodReturnValueHandler
  → Returns WebAsyncTask<T>

ReactiveTypeReturnValueHandler
  → Returns Mono<T>, Flux<T> (Spring 5 MVC + Reactor)
  → Adapts to DeferredResult for Servlet async

RequestResponseBodyMethodProcessor
  → @ResponseBody or @RestController class
  → Serialises return value via HttpMessageConverter chain
  → Sets Content-Type header
  → Marks mavContainer.requestHandled = true
  → (No view will be rendered)

ViewNameMethodReturnValueHandler
  → Returns String → sets as view name in mavContainer

MapMethodProcessor
  → Returns Map → added to model

ViewMethodReturnValueHandler (void/null)
  → No return value → use default view name

  Custom return value handlers:
  → Added via WebMvcConfigurer.addReturnValueHandlers()
  → Added AFTER all defaults

  ServletModelAttributeMethodProcessor (last catch-all)
  → Non-simple return types without annotation
```

---

### ModelAndViewContainer — The Central State Object

`ModelAndViewContainer` is the mutable state object passed through the entire handler invocation pipeline:

```java
public class ModelAndViewContainer {

    // Was the response fully handled? (e.g., @ResponseBody wrote it)
    // If true: no view rendering, no further model processing
    private boolean requestHandled = false;

    // The view — either a String (name) or a View object
    @Nullable
    private Object view;

    // The model — all attributes accumulated during processing
    private final ModelMap defaultModel = new BindingAwareModelMap();

    // Redirect model — separate model for redirect scenarios
    @Nullable
    private ModelMap redirectModel;

    // Should redirect use default model or redirect model?
    private boolean redirectModelScenario = false;

    // HTTP status to set
    @Nullable
    private HttpStatus status;

    // Attributes that should NOT be treated as data binding targets
    private final Set<String> noBinding = new HashSet<>();

    // Attributes that have been data-bound and validated
    private final Set<String> bindingDisabled = new HashSet<>();

    // @SessionAttributes: names collected for session storage
    private final SessionStatus sessionStatus = new SimpleSessionStatus();

    // Should default model be ignored on redirect?
    private boolean ignoreDefaultModelOnRedirect = false;
}
```

**Critical `requestHandled` flag:**
- Set to `true` by `RequestResponseBodyMethodProcessor` when `@ResponseBody` return value is written
- When `true`: `invokeHandlerMethod()` returns `null` from `getModelAndView()`
- `DispatcherServlet` receives `null` ModelAndView → skips ViewResolver + View.render()
- This is the mechanism that prevents view rendering for REST endpoints

---

### @InitBinder — WebDataBinderFactory Internals

```java
// WebDataBinderFactory creates WebDataBinder for each bind operation
// @InitBinder methods customise the binder before binding occurs

// Execution order for @InitBinder:
// 1. @ControllerAdvice @InitBinder methods (global)
// 2. @Controller-local @InitBinder methods

// What @InitBinder can do:
@InitBinder
public void initBinder(WebDataBinder binder) {

    // Register custom PropertyEditor for type conversion
    binder.registerCustomEditor(Date.class,
        new CustomDateEditor(new SimpleDateFormat("yyyy-MM-dd"), false));

    // Register custom Converter
    binder.addCustomFormatter(new DateTimeFormatter());

    // Set allowed/disallowed fields (security!)
    binder.setAllowedFields("name", "email", "address");
    // Prevents binding to fields like "admin", "role" (mass assignment attacks)

    binder.setDisallowedFields("id", "createdAt", "role");

    // Set required fields
    binder.setRequiredFields("name", "email");

    // Set validator
    binder.setValidator(new UserValidator());
}

// @InitBinder can be scoped to specific model attributes:
@InitBinder("product")
public void initProductBinder(WebDataBinder binder) {
    // Only runs when binding "product" model attribute
    binder.setDisallowedFields("id");
}
```

---

### @ModelAttribute Methods — Execution Timing and Order

```java
// @ModelAttribute methods ALWAYS execute before the handler method
// Even for @ResponseBody endpoints — model just isn't used

@Controller
@RequestMapping("/orders")
public class OrderController {

    // Executes BEFORE every handler method in this controller
    // Populates model with category list for ALL order pages
    @ModelAttribute("categories")
    public List<Category> populateCategories() {
        return categoryService.findAll(); // DB call on EVERY request to this controller
    }

    // Order of execution:
    // 1. @ControllerAdvice @ModelAttribute methods (global)
    // 2. Controller @ModelAttribute methods (local)
    // 3. Handler method itself

    @GetMapping("/{id}")
    public String view(@PathVariable Long id, Model model) {
        // "categories" already in model from populateCategories()
        model.addAttribute("order", orderService.findById(id));
        return "order/view";
    }
}
```

---

### HttpRequestHandlerAdapter — Static Resources

```java
public class HttpRequestHandlerAdapter implements HandlerAdapter {

    @Override
    public boolean supports(Object handler) {
        return (handler instanceof HttpRequestHandler);
    }

    @Override
    @Nullable
    public ModelAndView handle(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler) throws Exception {
        ((HttpRequestHandler) handler).handleRequest(request, response);
        return null; // Response written by handler — no view needed
    }

    @Override
    public long getLastModified(HttpServletRequest request,
                                 Object handler) {
        if (handler instanceof LastModified lm) {
            return lm.getLastModified(request);
        }
        return -1L;
    }
}

// ResourceHttpRequestHandler (static resources) implements HttpRequestHandler
// It is invoked by HttpRequestHandlerAdapter
// Serves files from classpath:/static/, /META-INF/resources/, etc.
```

---

### SimpleControllerHandlerAdapter — Legacy Interface

```java
public class SimpleControllerHandlerAdapter implements HandlerAdapter {

    @Override
    public boolean supports(Object handler) {
        return (handler instanceof Controller);
        // org.springframework.web.servlet.mvc.Controller
        // NOT @Controller annotation
    }

    @Override
    @Nullable
    public ModelAndView handle(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler) throws Exception {
        return ((Controller) handler).handleRequest(request, response);
        // Returns ModelAndView directly — legacy pattern
    }
}

// Legacy usage (Spring 1.x style — avoid in new code):
@Component("/products")
public class ProductController implements Controller {
    @Override
    public ModelAndView handleRequest(HttpServletRequest request,
                                       HttpServletResponse response) {
        ModelAndView mav = new ModelAndView("product/list");
        mav.addObject("products", productService.findAll());
        return mav;
    }
}
```

---

### The MessageConverter Role in RequestMappingHandlerAdapter

`RequestResponseBodyMethodProcessor` handles both:
- `@RequestBody` argument resolution (reads request body)
- `@ResponseBody` return value handling (writes response body)

Both use the `HttpMessageConverter` chain:

```
@RequestBody resolution:
  Request has Content-Type: application/json
  requestBody type is Product
  → Iterate converters:
  → MappingJackson2HttpMessageConverter.canRead(Product.class, application/json)?
  → YES → readWithMessageConverters() → Jackson deserialise → Product instance

@ResponseBody handling:
  Return value is Product
  Request has Accept: application/json
  → Iterate converters:
  → MappingJackson2HttpMessageConverter.canWrite(Product.class, application/json)?
  → YES → writeWithMessageConverters() → Jackson serialise → write to response
  → Sets Content-Type: application/json on response
  → Sets mavContainer.requestHandled = true
```

**Content negotiation determines which converter is selected for @ResponseBody.** This links HandlerAdapter to ViewResolver's counterpart for REST: HttpMessageConverter.

---

## 2️⃣ CODE EXAMPLES

### Custom HandlerMethodArgumentResolver

```java
// Custom resolver for @CurrentUser annotation
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface CurrentUser {}

@Component
public class CurrentUserArgumentResolver
    implements HandlerMethodArgumentResolver {

    @Autowired
    private UserService userService;

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(CurrentUser.class);
    }

    @Override
    @Nullable
    public Object resolveArgument(
            MethodParameter parameter,
            @Nullable ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest,
            @Nullable WebDataBinderFactory binderFactory) {

        HttpServletRequest request =
            webRequest.getNativeRequest(HttpServletRequest.class);

        // Extract from security context or session
        Principal principal = request.getUserPrincipal();
        if (principal == null) {
            if (parameter.isOptional()) return null;
            throw new IllegalStateException("No authenticated user");
        }

        return userService.findByUsername(principal.getName());
    }
}

// Registration
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private CurrentUserArgumentResolver currentUserResolver;

    @Override
    public void addArgumentResolvers(
            List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(currentUserResolver);
        // Added AFTER all defaults — cannot override built-in resolvers
    }
}

// Usage
@GetMapping("/profile")
public String profile(@CurrentUser User user, Model model) {
    model.addAttribute("user", user);
    return "profile";
}
```

---

### Custom HandlerMethodReturnValueHandler

```java
// Handler for custom @ApiResponse annotation
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ApiResponse {}

@Component
public class ApiResponseReturnValueHandler
    implements HandlerMethodReturnValueHandler {

    @Autowired
    private ObjectMapper objectMapper;

    @Override
    public boolean supportsReturnType(MethodParameter returnType) {
        return returnType.hasMethodAnnotation(ApiResponse.class);
    }

    @Override
    public void handleReturnValue(
            @Nullable Object returnValue,
            MethodParameter returnType,
            ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest) throws Exception {

        // Wrap response in standard API envelope
        Map<String, Object> envelope = new LinkedHashMap<>();
        envelope.put("status", "success");
        envelope.put("data", returnValue);
        envelope.put("timestamp", Instant.now().toString());

        HttpServletResponse response =
            webRequest.getNativeResponse(HttpServletResponse.class);
        response.setContentType("application/json");
        response.setStatus(200);
        objectMapper.writeValue(response.getWriter(), envelope);

        // Critical: mark request as handled
        // Otherwise Spring will try to find a view
        mavContainer.setRequestHandled(true);
    }
}

// Registration
@Override
public void addReturnValueHandlers(
        List<HandlerMethodReturnValueHandler> handlers) {
    handlers.add(new ApiResponseReturnValueHandler(objectMapper));
}

// Usage
@GetMapping("/products")
@ApiResponse
public List<Product> getProducts() {
    return productService.findAll();
    // Response: {"status":"success","data":[...],"timestamp":"..."}
}
```

---

### Observing Argument Resolution Pipeline

```java
// Custom resolver that logs resolution for debugging
@Component
public class LoggingArgumentResolver
    implements HandlerMethodArgumentResolver {

    @Autowired
    private HandlerMethodArgumentResolverComposite delegate;

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        // Never claims to support — just for logging
        return false;
        // Won't actually be called for resolution
    }

    // To truly intercept, use a decorator approach:
}

// Better approach — use HandlerInterceptor to observe post-binding state:
@Component
public class BindingObserverInterceptor implements HandlerInterceptor {

    @Override
    public void postHandle(HttpServletRequest request,
                           HttpServletResponse response,
                           Object handler,
                           ModelAndView modelAndView) {
        if (handler instanceof HandlerMethod hm) {
            System.out.println("Handler: " + hm.getShortLogMessage());
            System.out.println("Model attributes: " +
                (modelAndView != null ?
                    modelAndView.getModel().keySet() : "none"));
        }
    }
}
```

---

### Custom HandlerAdapter for Scripted Controllers

```java
// Hypothetical: adapter for Groovy script controllers
public class GroovyScriptHandlerAdapter implements HandlerAdapter {

    @Override
    public boolean supports(Object handler) {
        return handler instanceof GroovyScriptController;
    }

    @Override
    @Nullable
    public ModelAndView handle(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler) throws Exception {

        GroovyScriptController scriptController =
            (GroovyScriptController) handler;

        // Execute Groovy script
        Map<String, Object> binding = new HashMap<>();
        binding.put("request", request);
        binding.put("response", response);

        Object result = scriptController.execute(binding);

        if (result instanceof String viewName) {
            return new ModelAndView(viewName);
        } else if (result instanceof ModelAndView mav) {
            return mav;
        }

        return null; // Script wrote response directly
    }

    @Override
    public long getLastModified(HttpServletRequest request,
                                 Object handler) {
        return -1L; // No caching support
    }
}

// Registration
@Bean
public HandlerAdapter groovyHandlerAdapter() {
    return new GroovyScriptHandlerAdapter();
}
// DispatcherServlet's detectAllHandlerAdapters=true finds this automatically
```

---

### Edge Case — @InitBinder and Mass Assignment Prevention

```java
@Controller
public class UserController {

    // SECURITY: Prevent mass assignment attacks
    // Without this, POST body could set "admin=true", "role=ADMIN"
    @InitBinder("user")
    public void restrictUserBinding(WebDataBinder binder) {
        // ONLY allow these fields from form/request binding
        binder.setAllowedFields(
            "firstName", "lastName", "email", "phoneNumber");
        // "id", "role", "admin", "createdAt" are now blocked
        // Silently ignored if present in request
    }

    @PostMapping("/users/{id}")
    public String updateUser(
            @PathVariable Long id,
            @ModelAttribute("user") @Valid User user,
            BindingResult result) {
        // user.getRole() will be null/default
        // even if request contained role=ADMIN
        if (result.hasErrors()) return "user/edit";
        userService.update(id, user);
        return "redirect:/users/" + id;
    }
}
```

---

### Edge Case — mavContainer.setRequestHandled() Critical Usage

```java
@RestController
public class StreamController {

    @GetMapping("/stream")
    public void stream(HttpServletResponse response) throws IOException {
        // Writing directly to response
        response.setContentType("text/plain");
        response.setStatus(200);
        PrintWriter writer = response.getWriter();
        for (int i = 0; i < 100; i++) {
            writer.println("Line " + i);
            writer.flush();
        }
        // IMPORTANT: Return type is void
        // Spring MVC handles void return by checking if response is committed
        // Response IS committed (writer.flush() called) → Spring skips view rendering
        // mavContainer.requestHandled = true is set implicitly via response-committed check
    }

    @GetMapping("/custom-response")
    public void customResponse(
            HttpServletRequest request,
            HttpServletResponse response) throws IOException {
        // Custom binary response
        response.setContentType("application/octet-stream");
        response.setStatus(200);
        byte[] data = generateBinaryData();
        response.getOutputStream().write(data);
        // void return → Spring checks response committed → skips view resolution
    }
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
`RequestMappingHandlerAdapter.invokeHandlerMethod()` builds a `ModelAndViewContainer`. What does `mavContainer.setRequestHandled(true)` cause?

A) The response is immediately committed to the client
B) `invokeHandlerMethod()` returns `null` — DispatcherServlet skips ViewResolver and View rendering
C) The model is cleared and session attributes are saved
D) The response status is set to 200 OK

**Answer: B**
When `mavContainer.isRequestHandled() == true`, `getModelAndView()` returns `null`. `DispatcherServlet.processDispatchResult()` receives `null` ModelAndView and skips `render()`. This is the mechanism that prevents view rendering for `@ResponseBody` / `@RestController` endpoints. `RequestResponseBodyMethodProcessor` sets `requestHandled=true` after writing the body.

---

**Q2 — Select All That Apply**
Which of the following are processed by `RequestMappingHandlerAdapter.afterPropertiesSet()` at startup?

A) Scans all `@Controller` beans to build URL registry
B) Discovers all `@ControllerAdvice` beans and extracts their `@InitBinder` and `@ModelAttribute` methods
C) Registers all default `HandlerMethodArgumentResolver` implementations
D) Registers all default `HandlerMethodReturnValueHandler` implementations
E) Creates the `WebDataBinder` for each `@ModelAttribute` parameter

**Answer: B, C, D**
A is done by `RequestMappingHandlerMapping.afterPropertiesSet()` — different class. E happens per-request during `invokeHandlerMethod()` — not at startup. B, C, D are all done during `RequestMappingHandlerAdapter.afterPropertiesSet()`.

---

**Q3 — Ordering**
Order these steps when processing a `POST /products` request with `@RequestBody Product product`:

- `WebDataBinderFactory` created from `@InitBinder` methods
- `RequestResponseBodyMethodProcessor.resolveArgument()` reads JSON
- Handler method invoked via reflection
- `@ModelAttribute` methods executed
- `HttpMessageConverter` deserialises JSON to `Product`
- `mavContainer.setRequestHandled(true)` set

**Correct Order:**
1. `WebDataBinderFactory` created from `@InitBinder` methods
2. `@ModelAttribute` methods executed
3. `RequestResponseBodyMethodProcessor.resolveArgument()` reads JSON
4. `HttpMessageConverter` deserialises JSON to `Product`
5. Handler method invoked via reflection
6. `mavContainer.setRequestHandled(true)` set

---

**Q4 — MCQ**
`RequestParamMethodArgumentResolver` is registered TWICE in the default argument resolver chain. What is the purpose of the second registration with `useDefaultResolution=true`?

A) To handle `@RequestParam` with `required=false`
B) To handle simple type parameters without any annotation
C) To handle `@RequestParam` on `Map` types
D) To serve as a fallback for failed first-pass resolution

**Answer: B**
The first registration (`useDefaultResolution=false`) handles parameters explicitly annotated with `@RequestParam`. The second registration (`useDefaultResolution=true`) at the END of the chain handles simple type parameters (String, int, long, etc.) WITHOUT any annotation. This is why `public String method(String name)` works — `name` is resolved from request param `name` even without explicit `@RequestParam`.

---

**Q5 — True/False**
Custom `HandlerMethodArgumentResolver` implementations added via `WebMvcConfigurer.addArgumentResolvers()` are inserted BEFORE the default resolvers and can override default resolution behaviour.

**Answer: False**
Custom resolvers added via `addArgumentResolvers()` are appended AFTER all default resolvers. They cannot override built-in resolution. If you need to override a default resolver (e.g., custom `@RequestParam` handling), you must replace the entire `RequestMappingHandlerAdapter`'s resolver list directly — not use `addArgumentResolvers()`. This is a common trap — developers expect their custom resolver to intercept before built-ins but it never gets called because a default resolver claims the parameter first.

---

**Q6 — Code Prediction**
```java
@RestController
public class TestController {

    @GetMapping("/test")
    public Product getProduct() {
        return new Product("Widget", 9.99);
    }
}

// WebConfig:
@Override
public void configureMessageConverters(
        List<HttpMessageConverter<?>> converters) {
    // Empty — no converters added
}
```
`GET /test` with `Accept: application/json`. What happens?

A) Returns JSON `{"name":"Widget","price":9.99}`
B) HTTP 406 Not Acceptable
C) HTTP 500 — NullPointerException in serialisation
D) Returns empty response body

**Answer: B**
`configureMessageConverters()` with empty list REPLACES all defaults. No `MappingJackson2HttpMessageConverter` registered. `RequestResponseBodyMethodProcessor` cannot find a converter that can write `Product` as `application/json`. Throws `HttpMediaTypeNotAcceptableException` → HTTP 406.

---

**Q7 — MCQ**
Where in the execution chain are `@ModelAttribute` methods from `@ControllerAdvice` executed relative to controller-local `@ModelAttribute` methods?

A) `@ControllerAdvice` methods execute AFTER controller-local methods
B) `@ControllerAdvice` methods execute BEFORE controller-local methods
C) Both execute in parallel — order is non-deterministic
D) `@ControllerAdvice` methods only execute if the controller has no local `@ModelAttribute` methods

**Answer: B**
`ModelFactory.initModel()` executes `@ModelAttribute` methods in this order: (1) global methods from `@ControllerAdvice` beans (sorted by `@Order`), then (2) controller-local `@ModelAttribute` methods. Global always before local. Both always execute (D is wrong).

---

**Q8 — Scenario**
A controller method returns `ResponseEntity<Product>` with HTTP 201 and a `Location` header. Which `HandlerMethodReturnValueHandler` processes this, and what does it do?

A) `ViewNameMethodReturnValueHandler` — sets view name from entity
B) `HttpEntityMethodProcessor` — extracts status, headers, body; writes body via `HttpMessageConverter`; sets `mavContainer.requestHandled=true`
C) `RequestResponseBodyMethodProcessor` — ignores status and headers, only serialises body
D) `ModelAndViewMethodReturnValueHandler` — creates ModelAndView from entity

**Answer: B**
`HttpEntityMethodProcessor` handles `HttpEntity<T>` and `ResponseEntity<T>`. It:
1. Extracts HTTP status (201 Created)
2. Extracts and sets response headers (Location)
3. Serialises body via `HttpMessageConverter` chain
4. Sets `mavContainer.setRequestHandled(true)` — no view rendering

`RequestResponseBodyMethodProcessor` handles `@ResponseBody` on method/class level — not `ResponseEntity`.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — `addArgumentResolvers()` APPENDS, never prepends**
Custom argument resolvers go at the END of the chain. If `@RequestParam String name` is on a parameter, the built-in `RequestParamMethodArgumentResolver` claims it first — your custom resolver never runs. To truly replace built-in resolution, you must set the full resolver list on `RequestMappingHandlerAdapter` directly (via `@Bean` override) — not use `addArgumentResolvers()`.

**Trap 2 — `@ModelAttribute` methods run even for `@ResponseBody` endpoints**
`modelFactory.initModel()` is called in `invokeHandlerMethod()` BEFORE argument resolution. It always runs all applicable `@ModelAttribute` methods. If your `@ModelAttribute` method makes a DB call and the handler method has `@ResponseBody`, that DB call still happens — and the result is stored in a model that's immediately thrown away. Significant hidden performance cost.

**Trap 3 — `mavContainer.setRequestHandled(true)` is the REST/View split**
The single boolean flag `requestHandled` in `ModelAndViewContainer` is what determines whether a request gets view rendering or not. Any return value handler that writes the response directly MUST set this to `true`. If a custom return value handler writes the response but forgets `setRequestHandled(true)`, Spring will also try to render a view — resulting in `IllegalStateException: response already committed`.

**Trap 4 — void return and response-committed check**
A controller method returning `void` doesn't automatically mean "no view." Spring checks if the response is already committed. If NOT committed, `DefaultRequestToViewNameTranslator` derives a view name and attempts rendering. If the method wrote to `response.getWriter()` but never committed (no flush), Spring will still try to render a view — causing double-write confusion.

**Trap 5 — `@InitBinder` runs per binding operation, not per request**
If a controller has two `@ModelAttribute` parameters, `@InitBinder` runs TWICE — once for each binding operation. If `@InitBinder` has side effects (logging, counters), they fire per-binding, not per-request. Also: `@InitBinder("product")` with a name qualifier only runs for the `"product"` model attribute — not for all bindings.

**Trap 6 — `synchronizeOnSession` in RequestMappingHandlerAdapter**
`synchronizeOnSession=false` by default. When `true`, the handler method is executed inside `synchronized(session)` block — preventing concurrent requests from the same session from running simultaneously. Many developers don't know this exists and implement their own session locking. The flag solves concurrent form submission problems but adds contention for concurrent tabs.

**Trap 7 — `@ResponseBody` on class vs method**
`@RestController` = `@Controller` + `@ResponseBody` on the class. `RequestResponseBodyMethodProcessor.supportsReturnType()` checks for `@ResponseBody` on METHOD OR CLASS. Class-level `@ResponseBody` (via `@RestController`) applies to ALL methods including void-returning ones. A void method in a `@RestController` → `requestHandled=true` not set via `@ResponseBody` path, but response-committed check handles it if response was written.

---

## 5️⃣ SUMMARY SHEET

```
HANDLERADAPTER — INTERFACE CONTRACT
─────────────────────────────────────────────────────
supports(handler)  → determines which adapter is used
handle(req, res, handler) → invokes handler, returns ModelAndView or null
  null return = response already written (no view rendering needed)
getLastModified() → HTTP cache support (deprecated in 5.3)

FOUR DEFAULT ADAPTERS
─────────────────────────────────────────────────────
RequestMappingHandlerAdapter  → HandlerMethod (from @RequestMapping)
HttpRequestHandlerAdapter     → HttpRequestHandler (resources, custom)
SimpleControllerHandlerAdapter → Controller interface (legacy)
HandlerFunctionAdapter        → HandlerFunction (functional endpoints)

REQUESTMAPPINGHANDLERADAPTER EXECUTION FLOW
─────────────────────────────────────────────────────
handle()
  → invokeHandlerMethod()
      1. Create WebDataBinderFactory  (@InitBinder methods collected)
      2. Create ModelFactory          (@ModelAttribute methods collected)
      3. Create ServletInvocableHandlerMethod
      4. Create ModelAndViewContainer (mavContainer)
      5. Copy flash attributes to model
      6. Execute @ModelAttribute methods  ← ALWAYS, even for @ResponseBody
      7. invokeAndHandle():
           a. resolveArgument() for each parameter (ArgumentResolver chain)
           b. method.invoke(bean, resolvedArgs)  ← REFLECTION
           c. handleReturnValue() (ReturnValueHandler chain)
      8. getModelAndView() → null if mavContainer.requestHandled=true

KEY STATE OBJECT: ModelAndViewContainer
─────────────────────────────────────────────────────
requestHandled=true  → No ViewResolver, no View.render()
                     → Set by RequestResponseBodyMethodProcessor
                     → Set by HttpEntityMethodProcessor
requestHandled=false → Proceed to ViewResolver + View.render()
view field           → String (name) or View object
defaultModel         → BindingAwareModelMap (all model attributes)

ARGUMENT RESOLVER CHAIN (critical subset)
─────────────────────────────────────────────────────
@RequestParam       → RequestParamMethodArgumentResolver
@PathVariable       → PathVariableMethodArgumentResolver
@RequestBody        → RequestResponseBodyMethodProcessor (uses MessageConverter)
@ModelAttribute     → ServletModelAttributeMethodProcessor (data binding)
@RequestHeader      → RequestHeaderMethodArgumentResolver
@CookieValue        → ServletCookieValueMethodArgumentResolver
Model/ModelMap      → ModelMethodProcessor
HttpServletRequest  → ServletRequestMethodArgumentResolver
BindingResult       → ErrorsMethodArgumentResolver
RedirectAttributes  → RedirectAttributesMethodArgumentResolver

DUAL REGISTRATION TRICK
─────────────────────────────────────────────────────
RequestParamMethodArgumentResolver registered TWICE:
  1st (useDefaultResolution=false) → only @RequestParam annotated
  Last (useDefaultResolution=true) → any simple type WITHOUT annotation
  → Allows public void method(String name) without @RequestParam

Custom resolvers via addArgumentResolvers() → APPENDED AFTER defaults
Cannot override built-in resolvers via addArgumentResolvers()

RETURN VALUE HANDLER CHAIN (critical subset)
─────────────────────────────────────────────────────
ModelAndView        → ModelAndViewMethodReturnValueHandler
ResponseEntity<T>   → HttpEntityMethodProcessor  (sets status+headers+body)
@ResponseBody       → RequestResponseBodyMethodProcessor (MessageConverter)
String              → ViewNameMethodReturnValueHandler (sets view name)
void                → check response committed → default view name
Callable<T>         → CallableMethodReturnValueHandler (async)
Mono<T>/Flux<T>     → ReactiveTypeReturnValueHandler (via DeferredResult)
SseEmitter          → ResponseBodyEmitterReturnValueHandler (streaming)

@INITBINDER EXECUTION
─────────────────────────────────────────────────────
Timing:   Per data-binding operation (not per request)
Order:    @ControllerAdvice first, then controller-local
Scoping:  @InitBinder("name") → only for that model attribute
Usage:    PropertyEditors, allowed/disallowed fields, validators

@MODELATTRIBUTE METHOD EXECUTION
─────────────────────────────────────────────────────
Timing:   BEFORE every handler method — ALWAYS (even @ResponseBody)
Order:    @ControllerAdvice methods FIRST, then controller-local
Runs on:  Every request to the controller (hidden DB call trap)

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "supports() selects which adapter invokes which handler type"
• "null return from handle() = response written = no view rendering"
• "mavContainer.requestHandled=true = skip ViewResolver entirely"
• "@ModelAttribute methods run even for @ResponseBody endpoints — hidden cost"
• "addArgumentResolvers() appends after defaults — cannot override built-ins"
• "RequestParamMethodArgumentResolver registered twice — second handles no-annotation simple types"
• "@InitBinder runs per binding operation — not once per request"
• "HttpEntityMethodProcessor handles ResponseEntity — extracts status, headers, body"
• "synchronizeOnSession=true serialises concurrent requests from same session"
• "Custom ReturnValueHandler MUST call mavContainer.setRequestHandled(true) if writing response"
```

---

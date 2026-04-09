# TOPIC 3.6 — EXCEPTION HANDLING: @ExceptionHandler, @ControllerAdvice, HandlerExceptionResolver

---

## 1️⃣ CONCEPTUAL EXPLANATION

### Why Exception Handling Deserves Deep Study

Exception handling in Spring MVC is architecturally more complex than it appears. There are THREE completely separate mechanisms operating at different levels, with precise ordering rules, interaction patterns, and subtle behaviours that are heavily tested. Mastery requires understanding not just which annotation to use, but exactly when each mechanism fires, what it can and cannot handle, and how they interact when multiple mechanisms are configured simultaneously.

---

### The Three-Layer Exception Handling Architecture

```
LAYER 1: HandlerExceptionResolver chain
         (DispatcherServlet level — processHandlerException())
         ├── ExceptionHandlerExceptionResolver  (order=0)
         │     → @ExceptionHandler methods in controllers
         │     → @ExceptionHandler in @ControllerAdvice
         ├── ResponseStatusExceptionResolver    (order=1)
         │     → @ResponseStatus on exception class
         │     → ResponseStatusException thrown directly
         └── DefaultHandlerExceptionResolver   (order=2)
               → Standard Spring MVC exceptions
               → Sets HTTP status codes

LAYER 2: Servlet container error handling
         (after DispatcherServlet — when Layer 1 fails)
         → web.xml <error-page> or
         → ErrorPageRegistrar in Spring Boot

LAYER 3: Spring Boot BasicErrorController
         (Spring Boot only)
         → Catches /error dispatch from container
         → Renders error response (JSON or HTML)
         → Based on ErrorAttributes

All three can coexist — understanding which fires when is critical.
```

---

### Layer 1: ExceptionHandlerExceptionResolver — Complete Internals

```java
public class ExceptionHandlerExceptionResolver
    extends AbstractMessageConverterMethodExceptionHandler
    implements HandlerExceptionResolver, ApplicationContextAware,
               InitializingBean {

    // Per-controller cache: ControllerClass → ExceptionHandlerMethodResolver
    private final Map<Class<?>, ExceptionHandlerMethodResolver>
        exceptionHandlerCache = new ConcurrentHashMap<>(64);

    // @ControllerAdvice beans ordered by @Order
    private final Map<ControllerAdviceBean,
        ExceptionHandlerMethodResolver>
        exceptionHandlerAdviceCache = new LinkedHashMap<>();

    @Override
    public void afterPropertiesSet() {
        // STARTUP: scan all @ControllerAdvice beans
        // Extract @ExceptionHandler methods
        // Build ExceptionHandlerMethodResolver per advice
        // Store in exceptionHandlerAdviceCache (ordered)
        initExceptionHandlerAdviceCache();
    }

    @Override
    @Nullable
    public ModelAndView resolveException(
            HttpServletRequest request,
            HttpServletResponse response,
            @Nullable Object handler,
            Exception exception) {

        // handler is the HandlerMethod (controller method that threw)
        // or null if exception occurred outside handler execution

        if (shouldApplyTo(request, handler)) {
            prepareResponse(exception, response);
            ModelAndView result = doResolveHandlerMethodException(
                request, response, (HandlerMethod) handler, exception);
            if (result != null) {
                if (result.isEmpty() && handler != null &&
                        handler instanceof HandlerMethod &&
                        producesTextHtml(request, response)) {
                    // Special: return non-empty MaV for text/html
                }
            }
            return result;
        }
        return null;
    }
}
```

**`doResolveHandlerMethodException()` — the core lookup:**

```java
@Nullable
protected ModelAndView doResolveHandlerMethodException(
        HttpServletRequest request,
        HttpServletResponse response,
        @Nullable HandlerMethod handlerMethod,
        Exception exception) {

    // STEP 1: Find @ExceptionHandler method
    ServletInvocableHandlerMethod exceptionHandlerMethod =
        getExceptionHandlerMethod(handlerMethod, exception);

    if (exceptionHandlerMethod == null) {
        return null; // Not handled — try next resolver
    }

    // STEP 2: Set up argument/return value handlers
    // Same infrastructure as regular handler invocation
    exceptionHandlerMethod.setHandlerMethodArgumentResolvers(
        this.argumentResolvers);
    exceptionHandlerMethod.setHandlerMethodReturnValueHandlers(
        this.returnValueHandlers);

    // STEP 3: Create container (like regular handler invocation)
    ServletWebRequest webRequest =
        new ServletWebRequest(request, response);
    ModelAndViewContainer mavContainer =
        new ModelAndViewContainer();

    // STEP 4: Store exception in model (accessible in view)
    ArrayList<Throwable> exceptions = new ArrayList<>();
    try {
        // Walk exception cause chain
        Throwable exToExpose = exception;
        while (exToExpose != null) {
            exceptions.add(exToExpose);
            Throwable cause = exToExpose.getCause();
            exToExpose = (cause != exToExpose ? cause : null);
        }
        Object[] arguments = new Object[exceptions.size() + 1];
        exceptions.toArray(arguments);
        arguments[arguments.length - 1] = handlerMethod;

        // STEP 5: INVOKE the @ExceptionHandler method
        exceptionHandlerMethod.invokeAndHandle(
            webRequest, mavContainer, arguments);

    } catch (Throwable invocationEx) {
        // Exception in the @ExceptionHandler itself
        // Log and return null (propagates original exception)
        if (invocationEx != exception && ...) {
            logger.warn("Failure in @ExceptionHandler: ...",
                invocationEx);
        }
        return null;
    }

    // STEP 6: Return null if response written (@ResponseBody)
    if (mavContainer.isRequestHandled()) {
        return new ModelAndView();
        // EMPTY ModelAndView = handled, no view
    } else {
        // Has view name → render error view
        ModelAndView mav = new ModelAndView();
        mav.addAllObjects(mavContainer.getModel());
        mav.setViewName(mavContainer.getViewName());
        return mav;
    }
}
```

---

### ExceptionHandlerMethodResolver — The Lookup Algorithm

```java
// For a given exception, finds the appropriate @ExceptionHandler method
public class ExceptionHandlerMethodResolver {

    // exception type → method that handles it
    private final Map<Class<? extends Throwable>, Method>
        mappedMethods = new LinkedHashMap<>();

    // After cache: exception type → resolved method (with inheritance)
    private final Map<Class<? extends Throwable>, Method>
        exceptionLookupCache = new ConcurrentHashMap<>(16);

    // Build at startup: scan class for @ExceptionHandler methods
    public ExceptionHandlerMethodResolver(Class<?> handlerType) {
        for (Method method : MethodIntrospector.selectMethods(
                handlerType, EXCEPTION_HANDLER_METHODS)) {
            ExceptionHandler ann =
                AnnotationUtils.findAnnotation(method,
                    ExceptionHandler.class);
            // Register each exception type listed in annotation
            for (Class<? extends Throwable> exType :
                    ann.value()) {
                addExceptionMapping(exType, method);
            }
        }
    }

    // Resolve at runtime
    @Nullable
    public Method resolveMethodByThrowable(Throwable exception) {
        Method method = resolveMethodByExceptionType(
            exception.getClass());
        if (method == null) {
            Throwable cause = exception.getCause();
            if (cause != null) {
                method = resolveMethodByExceptionType(
                    cause.getClass());
            }
        }
        return method;
    }

    @Nullable
    private Method resolveMethodByExceptionType(
            Class<? extends Throwable> exceptionType) {

        Method method =
            this.exceptionLookupCache.get(exceptionType);

        if (method == null) {
            method = getMappedMethod(exceptionType);
            this.exceptionLookupCache.put(
                exceptionType,
                method != null ? method : NO_MATCHING_EXCEPTION_HANDLER);
        }
        return (method != NO_MATCHING_EXCEPTION_HANDLER ?
            method : null);
    }

    @Nullable
    private Method getMappedMethod(
            Class<? extends Throwable> exceptionType) {

        List<Class<? extends Throwable>> matches =
            new ArrayList<>();

        // Find ALL registered exception types that are
        // assignable from the thrown exception type
        for (Class<? extends Throwable> mappedException :
                this.mappedMethods.keySet()) {
            if (mappedException.isAssignableFrom(exceptionType)) {
                matches.add(mappedException);
            }
        }

        if (!matches.isEmpty()) {
            // Sort by SPECIFICITY: closest in hierarchy wins
            // IOException before Exception
            matches.sort(
                new ExceptionDepthComparator(exceptionType));
            return this.mappedMethods.get(matches.get(0));
        }

        return null;
    }
}
```

**Lookup priority:**
```
1. Controller-local @ExceptionHandler (FIRST checked)
   → ExceptionHandlerMethodResolver for THIS controller class
   → Most specific exception type wins

2. @ControllerAdvice @ExceptionHandler (SECOND checked)
   → Iterates exceptionHandlerAdviceCache (in @Order order)
   → For each ControllerAdvice: check if applicable to this handler
   → For each applicable ControllerAdvice: find matching handler

3. Exception cause chain checked
   → If SomeWrapper(cause=ProductNotFoundException) thrown
   → First tries SomeWrapper
   → Then tries ProductNotFoundException

4. None found → return null → try next HandlerExceptionResolver
```

---

### @ExceptionHandler — Complete Specification

```java
// @ExceptionHandler method signature — supported parameter types:
@ExceptionHandler(ProductNotFoundException.class)
public ResponseEntity<ErrorResponse> handleNotFound(
    ProductNotFoundException ex,  // The exception itself
    HttpServletRequest request,    // Current request
    HttpServletResponse response,  // Current response
    WebRequest webRequest,         // Spring abstraction
    HttpSession session,           // Session (use carefully in error handling)
    Model model,                   // Model (for view-based error handling)
    Locale locale,                 // For i18n error messages
    // Note: @PathVariable, @RequestParam NOT available
    // Note: @ModelAttribute NOT available
    // Note: BindingResult NOT available
    // Full list is subset of regular handler params
) {
    // Can return all same types as regular handlers:
    // ResponseEntity<T>, @ResponseBody, ModelAndView, String (view name)
    ErrorResponse error = new ErrorResponse(
        HttpStatus.NOT_FOUND.value(),
        "Product not found: " + ex.getId(),
        request.getRequestURI()
    );
    return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
}

// Multiple exception types in one handler
@ExceptionHandler({
    ProductNotFoundException.class,
    CategoryNotFoundException.class
})
public ResponseEntity<ErrorResponse> handleNotFound(
    RuntimeException ex,  // Common supertype
    WebRequest request) {
    return ResponseEntity.status(HttpStatus.NOT_FOUND)
        .body(new ErrorResponse(404, ex.getMessage()));
}

// Exception hierarchy — @ExceptionHandler hierarchy works
@ExceptionHandler(Exception.class)  // Catch-all
public ResponseEntity<ErrorResponse> handleAll(Exception ex) {
    return ResponseEntity.internalServerError()
        .body(new ErrorResponse(500, "Internal error"));
}
// If ProductNotFoundException extends RuntimeException extends Exception:
//   ProductNotFoundException → handled by most specific handler
//   Not ProductNotFoundException → falls through to Exception handler

// @ResponseStatus on @ExceptionHandler
@ExceptionHandler(ValidationException.class)
@ResponseStatus(HttpStatus.BAD_REQUEST)
@ResponseBody
public ErrorResponse handleValidation(ValidationException ex) {
    return new ErrorResponse(400, ex.getMessage());
    // @ResponseStatus sets the HTTP status
    // @ResponseBody serialises return value
    // Equivalent to ResponseEntity.status(400).body(...)
}
```

---

### @ControllerAdvice — Architecture and Scoping

```java
// @ControllerAdvice meta-annotation:
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component  // ← It's a Spring bean
public @interface ControllerAdvice {

    // Scope by base packages
    @AliasFor("basePackages")
    String[] value() default {};

    @AliasFor("value")
    String[] basePackages() default {};

    // Scope by specific classes in these packages
    Class<?>[] basePackageClasses() default {};

    // Scope by assignable type (controllers that are subtypes)
    Class<?>[] assignableTypes() default {};

    // Scope by annotation on controller class
    Class<? extends Annotation>[] annotations() default {};
}

// Examples:
@ControllerAdvice  // Applies to ALL controllers
@ControllerAdvice("com.example.api")  // Only api package controllers
@ControllerAdvice(assignableTypes=ProductController.class)  // Specific controller
@ControllerAdvice(annotations=RestController.class)  // Only @RestController classes
@ControllerAdvice(basePackageClasses=ProductController.class)  // Package of that class
```

**`@ControllerAdvice` applicability check:**

```java
// ControllerAdviceBean.isApplicableToBeanType() — called for each handler
public boolean isApplicableToBeanType(@Nullable Class<?> beanType) {
    return HandlerTypePredicate.forAnnotation(/* annotation types */)
        .and(HandlerTypePredicate.forBasePackage(/* packages */))
        .and(HandlerTypePredicate.forAssignableType(/* types */))
        .test(beanType);
    // All conditions must match (AND logic)
    // Empty conditions = matches ALL (no restriction)
}
```

---

### @RestControllerAdvice — The Composed Annotation

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@ControllerAdvice
@ResponseBody  // ← All methods return body (not view names)
public @interface RestControllerAdvice {
    // Same attributes as @ControllerAdvice
}

// @RestControllerAdvice = @ControllerAdvice + @ResponseBody
// All @ExceptionHandler methods in @RestControllerAdvice
// are treated as @ResponseBody — JSON/XML responses

// Equivalent:
@ControllerAdvice
@ResponseBody
public class GlobalExceptionHandler { ... }
// vs
@RestControllerAdvice
public class GlobalExceptionHandler { ... }
```

---

### Layer 2: ResponseStatusExceptionResolver

```java
// Handles two patterns:

// PATTERN 1: Exception class annotated with @ResponseStatus
@ResponseStatus(
    code = HttpStatus.NOT_FOUND,  // or value=
    reason = "Product not found"  // Optional: adds WWW-Authenticate-like header
)
public class ProductNotFoundException extends RuntimeException {
    // When this exception is thrown and reaches
    // ResponseStatusExceptionResolver:
    // → response.sendError(404, "Product not found")
    // → Bypasses Spring exception handling for error view
    // → Servlet container handles error page
}

// PATTERN 2: Throw ResponseStatusException directly
throw new ResponseStatusException(
    HttpStatus.NOT_FOUND,
    "Product not found: " + id,
    originalCause  // optional
);
// → response.setStatus(404)
// → response body: Spring's default error format (or container default)

// IMPORTANT: ResponseStatusExceptionResolver is ORDER 2
// ExceptionHandlerExceptionResolver (ORDER 0) runs FIRST
// If you have @ExceptionHandler for ProductNotFoundException,
// IT runs, not ResponseStatusExceptionResolver
```

**`reason` attribute trap:**
```java
@ResponseStatus(code=HttpStatus.NOT_FOUND, reason="Product not found")
// This calls response.sendError(404, "Product not found")
// sendError() COMMITS the response immediately
// AND marks it as an error dispatch
// The reason string goes to container's error page
// Spring's normal response writing is bypassed
// This is DIFFERENT from just setting status 404

// Without reason:
@ResponseStatus(HttpStatus.NOT_FOUND)
// response.setStatus(404) only
// Response NOT committed
// Normal response body can still be written
```

---

### Layer 3: DefaultHandlerExceptionResolver

Handles Spring MVC's own internal exceptions. These are thrown by the framework itself, not by application code:

```java
// Exception → HTTP Status mapping:
HttpRequestMethodNotSupportedException     → 405 Method Not Allowed
                                              + Allow header
HttpMediaTypeNotSupportedException         → 415 Unsupported Media Type
HttpMediaTypeNotAcceptableException        → 406 Not Acceptable
MissingPathVariableException               → 500 Internal Server Error
MissingServletRequestParameterException    → 400 Bad Request
MissingServletRequestPartException         → 400 Bad Request
ServletRequestBindingException             → 400 Bad Request
MethodArgumentTypeMismatchException        → 400 Bad Request
NoHandlerFoundException                    → 404 Not Found
                                              (if throwExceptionIfNoHandlerFound=true)
AsyncRequestTimeoutException               → 503 Service Unavailable
BindException                              → 400 Bad Request
MethodArgumentNotValidException            → 400 Bad Request

// DefaultHandlerExceptionResolver:
// 1. Calls response.sendError(statusCode) OR response.setStatus(statusCode)
// 2. Returns EMPTY ModelAndView (no error view, no body by default)
// 3. You can override these with @ExceptionHandler — they run FIRST
```

---

### Complete Resolution Sequence — Worked Example

```java
// Given this exception thrown in a controller:
throw new ProductNotFoundException(42L);

// And this setup:
@Controller
public class ProductController {
    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<Error> handleLocal(ProductNotFoundException ex) {
        return ResponseEntity.notFound().build();
    }
}

@RestControllerAdvice
@Order(1)
public class GlobalAdvice1 {
    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<Error> handleGlobal1(ProductNotFoundException ex) {
        return ResponseEntity.status(404).body(new Error("global1"));
    }
}

@RestControllerAdvice
@Order(2)
public class GlobalAdvice2 {
    @ExceptionHandler(Exception.class)
    public ResponseEntity<Error> handleAll(Exception ex) {
        return ResponseEntity.status(500).body(new Error("global2"));
    }
}

// RESOLUTION SEQUENCE:
// 1. ExceptionHandlerExceptionResolver.resolveException() called
// 2. getExceptionHandlerMethod(handlerMethod=ProductController, exception=ProductNotFoundException)
// 3. FIRST: Check ProductController's own ExceptionHandlerMethodResolver
//    → finds handleLocal() → SELECTED
// 4. handleLocal() invoked → returns ResponseEntity.notFound()
// 5. Response written → returns empty ModelAndView
// 6. GlobalAdvice1 and GlobalAdvice2 NEVER consulted

// If ProductController had NO local @ExceptionHandler:
// 3. ProductController has no matching handler
// 4. Check GlobalAdvice1 (order=1):
//    → isApplicableToBeanType(ProductController.class)? YES (no scope restriction)
//    → ExceptionHandlerMethodResolver for GlobalAdvice1:
//      findMethod(ProductNotFoundException.class) → handleGlobal1() FOUND
// 5. handleGlobal1() invoked
// 6. GlobalAdvice2 NEVER consulted (GlobalAdvice1 handled it)

// If NO ExceptionHandler found anywhere:
// ExceptionHandlerExceptionResolver returns null
// ResponseStatusExceptionResolver.resolveException() called
//   → Is exception annotated with @ResponseStatus? → check
// DefaultHandlerExceptionResolver.resolveException() called
//   → Is it a standard Spring MVC exception? → check
// All resolvers return null
// Exception re-thrown to FrameworkServlet
// HTTP 500 from container
```

---

### @ExceptionHandler Argument Resolution — What's Available

```java
// Supported argument types in @ExceptionHandler methods:
// (Subset of regular handler method arguments)

@ExceptionHandler(SomeException.class)
public ResponseEntity<Error> handler(
    SomeException ex,              // The exception (or supertype)
    Throwable throwable,           // Any throwable
    HandlerMethod handlerMethod,   // The handler that threw (if applicable)
    HttpServletRequest request,    // Request
    HttpServletResponse response,  // Response
    HttpSession session,           // Session
    WebRequest webRequest,         // Spring WebRequest
    NativeWebRequest nativeReq,    // NativeWebRequest
    UriComponentsBuilder ucb,      // URL building
    Locale locale,                 // Locale
    Principal principal            // Security principal
) { ... }

// NOT available (unlike regular handlers):
// @PathVariable, @RequestParam, @RequestHeader, @CookieValue
// Model, ModelMap, @ModelAttribute
// BindingResult
// These are REQUEST-time annotations — exception handlers run after
```

---

### Problem Hierarchy — Exception Matching Depth

```java
// Exception hierarchy:
RuntimeException
    └── AppException
          ├── NotFoundException
          │     ├── ProductNotFoundException
          │     └── OrderNotFoundException
          └── ValidationException

// Handler registrations:
@ExceptionHandler(AppException.class)
void handleApp(AppException ex) { ... }

@ExceptionHandler(NotFoundException.class)
void handleNotFound(NotFoundException ex) { ... }

// Thrown: ProductNotFoundException
// ExceptionDepthComparator:
//   AppException → depth from ProductNotFoundException = 3
//   NotFoundException → depth from ProductNotFoundException = 2
// WINNER: handleNotFound (depth=2, closer in hierarchy)

// The MOST SPECIFIC (closest in hierarchy) handler wins
// This is per-handler-registry (controller or ControllerAdvice)
// A local controller handler is always tried BEFORE ControllerAdvice handlers
```

---

### @ResponseBody in @ExceptionHandler — The Return Path

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // Return type is ResponseEntity → HttpEntityMethodProcessor
    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(
            ProductNotFoundException ex,
            HttpServletRequest request) {
        ApiError error = ApiError.builder()
            .status(HttpStatus.NOT_FOUND.value())
            .message(ex.getMessage())
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .build();
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    // INTERNAL PROCESSING:
    // @RestControllerAdvice has @ResponseBody
    // @ExceptionHandler method treated as @ResponseBody method
    // InvocableHandlerMethod.invokeAndHandle() processes return value
    // HttpEntityMethodProcessor:
    //   → Sets 404 status
    //   → Jackson serialises ApiError
    //   → mavContainer.setRequestHandled(true)
    // Empty ModelAndView returned to processHandlerException()
    // → No error view rendered
    // → afterCompletion() called
}
```

---

### Building a Production-Grade Exception Handling System

```
RECOMMENDED ARCHITECTURE:

1. Domain exceptions (application-specific):
   ProductNotFoundException extends ResourceNotFoundException
   ValidationException extends AppException

2. Local @ExceptionHandler only for truly controller-specific:
   @Controller
   ProductController {
     @ExceptionHandler(InventoryException.class)
     // only relevant in product context
   }

3. @RestControllerAdvice for REST APIs:
   @RestControllerAdvice
   @Order(Ordered.HIGHEST_PRECEDENCE)  // Run before generic handlers
   GlobalRestExceptionHandler {
     @ExceptionHandler(ResourceNotFoundException.class) → 404
     @ExceptionHandler(ValidationException.class) → 422
     @ExceptionHandler(MethodArgumentNotValidException.class) → 400
     @ExceptionHandler(HttpRequestMethodNotSupportedException.class) → 405
   }

4. Final catch-all:
   @RestControllerAdvice
   @Order(Ordered.LOWEST_PRECEDENCE)
   FallbackExceptionHandler {
     @ExceptionHandler(Exception.class) → 500
   }

5. Spring Boot ErrorController for errors outside Spring MVC
```

---

## 2️⃣ CODE EXAMPLES

### Complete Production Exception Handler

```java
// Exception hierarchy
public class AppException extends RuntimeException {
    private final HttpStatus status;
    private final String errorCode;

    public AppException(HttpStatus status, String errorCode,
                         String message) {
        super(message);
        this.status = status;
        this.errorCode = errorCode;
    }
    // getters
}

public class ResourceNotFoundException extends AppException {
    public ResourceNotFoundException(String resource, Object id) {
        super(HttpStatus.NOT_FOUND,
              "RESOURCE_NOT_FOUND",
              resource + " not found with id: " + id);
    }
}

public class BusinessValidationException extends AppException {
    private final Map<String, String> violations;

    public BusinessValidationException(
            Map<String, String> violations) {
        super(HttpStatus.UNPROCESSABLE_ENTITY,
              "VALIDATION_FAILED",
              "Business validation failed");
        this.violations = violations;
    }
}

// Standard error response format
public record ApiError(
    int status,
    String error,
    String errorCode,
    String message,
    String path,
    Instant timestamp,
    Map<String, String> fieldErrors
) {
    public static ApiError of(HttpStatus status, String errorCode,
                               String message, String path) {
        return new ApiError(status.value(), status.getReasonPhrase(),
            errorCode, message, path, Instant.now(), null);
    }

    public static ApiError withFieldErrors(HttpStatus status,
            String errorCode, String message, String path,
            Map<String, String> fieldErrors) {
        return new ApiError(status.value(), status.getReasonPhrase(),
            errorCode, message, path, Instant.now(), fieldErrors);
    }
}

// Global exception handler
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    // Application-specific exceptions
    @ExceptionHandler(AppException.class)
    public ResponseEntity<ApiError> handleAppException(
            AppException ex,
            HttpServletRequest request) {

        log.warn("Application exception: {} - {}",
            ex.getErrorCode(), ex.getMessage());

        return ResponseEntity
            .status(ex.getStatus())
            .body(ApiError.of(
                ex.getStatus(),
                ex.getErrorCode(),
                ex.getMessage(),
                request.getRequestURI()));
    }

    // JSR-303 validation failures (@Valid @RequestBody)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValidation(
            MethodArgumentNotValidException ex,
            HttpServletRequest request) {

        Map<String, String> fieldErrors =
            new LinkedHashMap<>();
        ex.getBindingResult()
            .getFieldErrors()
            .forEach(e -> fieldErrors.put(
                e.getField(),
                e.getDefaultMessage()));

        log.debug("Validation failed: {}", fieldErrors);

        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(ApiError.withFieldErrors(
                HttpStatus.BAD_REQUEST,
                "VALIDATION_FAILED",
                "Request validation failed",
                request.getRequestURI(),
                fieldErrors));
    }

    // HTTP method not supported
    @ExceptionHandler(HttpRequestMethodNotSupportedException.class)
    public ResponseEntity<ApiError> handleMethodNotAllowed(
            HttpRequestMethodNotSupportedException ex,
            HttpServletRequest request) {

        return ResponseEntity
            .status(HttpStatus.METHOD_NOT_ALLOWED)
            .allow(ex.getSupportedHttpMethods()
                .toArray(new HttpMethod[0]))
            .body(ApiError.of(
                HttpStatus.METHOD_NOT_ALLOWED,
                "METHOD_NOT_ALLOWED",
                ex.getMessage(),
                request.getRequestURI()));
    }

    // Content type not supported
    @ExceptionHandler(HttpMediaTypeNotSupportedException.class)
    public ResponseEntity<ApiError> handleUnsupportedMediaType(
            HttpMediaTypeNotSupportedException ex,
            HttpServletRequest request) {

        return ResponseEntity
            .status(HttpStatus.UNSUPPORTED_MEDIA_TYPE)
            .body(ApiError.of(
                HttpStatus.UNSUPPORTED_MEDIA_TYPE,
                "UNSUPPORTED_MEDIA_TYPE",
                "Supported types: " + ex.getSupportedMediaTypes(),
                request.getRequestURI()));
    }

    // JSON parse failure
    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ResponseEntity<ApiError> handleUnreadable(
            HttpMessageNotReadableException ex,
            HttpServletRequest request) {

        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(ApiError.of(
                HttpStatus.BAD_REQUEST,
                "MALFORMED_JSON",
                "Request body is not readable",
                request.getRequestURI()));
    }

    // Catch-all — MUST be last
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleUnexpected(
            Exception ex,
            HttpServletRequest request) {

        // Log the full stack trace for unexpected errors
        log.error("Unexpected error processing {}",
            request.getRequestURI(), ex);

        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ApiError.of(
                HttpStatus.INTERNAL_SERVER_ERROR,
                "INTERNAL_ERROR",
                "An unexpected error occurred",
                request.getRequestURI()));
    }
}
```

---

### @ExceptionHandler in Controller — Local Override

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @Autowired
    private OrderService orderService;

    @GetMapping("/{id}")
    public Order getOrder(@PathVariable Long id) {
        return orderService.findById(id)
            .orElseThrow(() ->
                new OrderNotFoundException(id));
    }

    @PostMapping
    public Order createOrder(
            @Valid @RequestBody CreateOrderRequest req) {
        return orderService.create(req);
    }

    // LOCAL exception handler — takes priority over GlobalExceptionHandler
    // for exceptions thrown in THIS controller's methods
    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<Map<String, String>> handleOrderNotFound(
            OrderNotFoundException ex) {
        // Custom response format specific to Order API
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(Map.of(
                "error", "ORDER_NOT_FOUND",
                "orderId", String.valueOf(ex.getOrderId()),
                "message", "Order does not exist"
            ));
    }

    // Handles inventory errors specific to order creation
    @ExceptionHandler(InsufficientInventoryException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public Map<String, Object> handleInventory(
            InsufficientInventoryException ex) {
        return Map.of(
            "error", "INSUFFICIENT_INVENTORY",
            "productId", ex.getProductId(),
            "requested", ex.getRequested(),
            "available", ex.getAvailable()
        );
    }
}
```

---

### Custom HandlerExceptionResolver

```java
// Implementing HandlerExceptionResolver directly
// (lower-level than @ExceptionHandler — for framework-level control)
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class SecurityExceptionResolver
    implements HandlerExceptionResolver {

    @Autowired
    private ObjectMapper objectMapper;

    @Override
    @Nullable
    public ModelAndView resolveException(
            HttpServletRequest request,
            HttpServletResponse response,
            @Nullable Object handler,
            Exception ex) {

        if (!(ex instanceof SecurityException)) {
            return null; // Not our exception
        }

        SecurityException secEx = (SecurityException) ex;

        try {
            response.setStatus(
                HttpStatus.FORBIDDEN.value());
            response.setContentType(
                MediaType.APPLICATION_JSON_VALUE);

            Map<String, Object> error = Map.of(
                "status", 403,
                "error", "FORBIDDEN",
                "message", secEx.getMessage()
            );

            objectMapper.writeValue(
                response.getWriter(), error);

        } catch (IOException e) {
            log.error("Error writing security error", e);
        }

        // Empty ModelAndView = handled, no view rendering
        return new ModelAndView();
    }
}
```

---

### ResponseStatusException — Spring 5+ Pattern

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    @GetMapping("/{id}")
    public Product getProduct(@PathVariable Long id) {
        return productService.findById(id)
            .orElseThrow(() ->
                // Inline exception without creating exception class
                new ResponseStatusException(
                    HttpStatus.NOT_FOUND,
                    "Product not found: " + id)
            );
        // ResponseStatusExceptionResolver handles this:
        // response.setStatus(404)
        // message set in response
    }

    @GetMapping("/{id}/inventory")
    public InventoryInfo getInventory(@PathVariable Long id) {
        if (!productService.exists(id)) {
            throw new ResponseStatusException(
                HttpStatus.NOT_FOUND,
                "Product " + id + " not found",
                new EntityNotFoundException("product", id)
                // Cause wrapped
            );
        }
        return inventoryService.getForProduct(id);
    }

    // ResponseStatusException caught by ExceptionHandlerExceptionResolver
    // FIRST (if @ExceptionHandler registered for it)
    // Then ResponseStatusExceptionResolver
    @ExceptionHandler(ResponseStatusException.class)
    public ResponseEntity<ApiError> handleResponseStatus(
            ResponseStatusException ex,
            HttpServletRequest request) {
        // Custom handling — overrides ResponseStatusExceptionResolver
        return ResponseEntity
            .status(ex.getStatusCode())
            .body(ApiError.of(
                (HttpStatus) ex.getStatusCode(),
                "RESOURCE_ERROR",
                ex.getReason(),
                request.getRequestURI()));
    }
}
```

---

### Exception Handling with @ControllerAdvice Scoping

```java
// Only handles exceptions from API controllers
@RestControllerAdvice(
    annotations = RestController.class,
    basePackages = "com.example.api"
)
@Order(1)
public class ApiExceptionHandler {

    @ExceptionHandler(AppException.class)
    public ResponseEntity<ApiError> handle(AppException ex,
            HttpServletRequest req) {
        return ResponseEntity.status(ex.getStatus())
            .body(ApiError.of(ex.getStatus(),
                ex.getErrorCode(), ex.getMessage(),
                req.getRequestURI()));
    }
}

// Only handles exceptions from MVC controllers (view-based)
@ControllerAdvice(
    annotations = Controller.class,
    basePackages = "com.example.web"
)
@Order(2)
public class MvcExceptionHandler {

    @ExceptionHandler(AppException.class)
    public ModelAndView handle(AppException ex) {
        ModelAndView mav = new ModelAndView("error/app-error");
        mav.addObject("errorCode", ex.getErrorCode());
        mav.addObject("message", ex.getMessage());
        mav.setStatus(ex.getStatus());
        return mav;
    }
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
A `ProductNotFoundException` is thrown. The controller has a local `@ExceptionHandler(RuntimeException.class)`. `GlobalAdvice` has `@ExceptionHandler(ProductNotFoundException.class)`. Which handler is invoked?

A) `GlobalAdvice` handler — more specific exception type
B) Controller local handler — controller-local always checked first
C) Both handlers invoked — Spring MVC chains exception handlers
D) `DefaultHandlerExceptionResolver` — handles `RuntimeException`

**Answer: B**
`ExceptionHandlerExceptionResolver` checks the **controller-local** `ExceptionHandlerMethodResolver` FIRST. The controller has `RuntimeException` handler — `ProductNotFoundException` IS-A `RuntimeException`. It matches. The controller-local handler runs. `GlobalAdvice` is never consulted. Local always beats global regardless of specificity.

---

**Q2 — Select All That Apply**
`@ResponseStatus(code=HttpStatus.NOT_FOUND, reason="Not Found")` on an exception class causes which behaviour when `ResponseStatusExceptionResolver` handles it?

A) `response.sendError(404, "Not Found")` is called
B) `response.setStatus(404)` is called  
C) The response is committed immediately
D) Spring can still write a response body after this
E) `@ExceptionHandler` for this exception runs INSTEAD if present

**Answer: A, C, E**
`reason` attribute → `response.sendError()` (not `setStatus()`). `sendError()` commits the response immediately (C). After `sendError()`, writing to the response body throws `IllegalStateException` (D is wrong). `ExceptionHandlerExceptionResolver` runs BEFORE `ResponseStatusExceptionResolver` — if a matching `@ExceptionHandler` exists, it runs instead (E is correct).

---

**Q3 — Ordering**
Order the exception resolution steps when `ProductNotFoundException` is thrown:

- `ResponseStatusExceptionResolver.resolveException()` called
- `ExceptionHandlerExceptionResolver` checks `@ControllerAdvice` beans
- `ExceptionHandlerExceptionResolver` checks controller-local `@ExceptionHandler`
- `DefaultHandlerExceptionResolver.resolveException()` called
- Exception re-thrown → container returns HTTP 500
- `ExceptionHandlerExceptionResolver.resolveException()` called

**Correct Order:**
1. `ExceptionHandlerExceptionResolver.resolveException()` called (order=0)
2. `ExceptionHandlerExceptionResolver` checks controller-local `@ExceptionHandler`
3. `ExceptionHandlerExceptionResolver` checks `@ControllerAdvice` beans (if no local)
4. `ResponseStatusExceptionResolver.resolveException()` called (order=1, if EHER returned null)
5. `DefaultHandlerExceptionResolver.resolveException()` called (order=2, if RSR returned null)
6. Exception re-thrown → container returns HTTP 500 (if all returned null)

---

**Q4 — MCQ**
`@ControllerAdvice(annotations=RestController.class)` is on a global exception handler. A `ProductNotFoundException` is thrown from a `@Controller` (not `@RestController`) method. What happens?

A) The `@ControllerAdvice` handler is invoked — exception type matching takes priority
B) The `@ControllerAdvice` handler is NOT invoked — annotation scope excludes `@Controller`
C) `@ControllerAdvice` scope is only a suggestion — Spring still applies it
D) `IllegalStateException` — scoped `@ControllerAdvice` cannot be used for non-REST controllers

**Answer: B**
`ControllerAdviceBean.isApplicableToBeanType()` checks if the handler's class (the `@Controller` class) has the `@RestController` annotation. It does not. The advice is NOT applicable. `ExceptionHandlerExceptionResolver` skips this `ControllerAdviceBean`. Scope filtering is enforced strictly.

---

**Q5 — True/False**
A `@ExceptionHandler` method in a `@RestControllerAdvice` can return a `String` as a view name to render an error view.

**Answer: False**
`@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`. The class-level `@ResponseBody` means ALL methods (including `@ExceptionHandler` methods) are treated as `@ResponseBody`. A `String` return → `StringHttpMessageConverter` writes it as `text/plain` to the response body. It is NOT treated as a view name. To return a view name from a `@ControllerAdvice` exception handler, use `@ControllerAdvice` WITHOUT `@ResponseBody`, and return `ModelAndView` or `String` without `@ResponseBody`.

---

**Q6 — Code Prediction**
```java
@RestControllerAdvice
public class Advice {
    @ExceptionHandler(Exception.class)
    public ResponseEntity<String> handleAll(Exception ex) {
        return ResponseEntity.status(500).body("error");
    }
}

@RestController
public class Ctrl {
    @GetMapping("/test")
    public String test() {
        throw new NullPointerException("null!");
    }

    @ExceptionHandler(RuntimeException.class)
    public ResponseEntity<String> handleRuntime(
            RuntimeException ex) {
        return ResponseEntity.status(500).body("local");
    }
}
```
`GET /test` — what response body is returned?

A) `"error"` — global `Exception` handler is more general
B) `"local"` — controller-local handler checked first
C) HTTP 500 with no body — no matching handler
D) `"error"` — `@RestControllerAdvice` takes priority over local handlers

**Answer: B**
Controller-local `@ExceptionHandler` is ALWAYS checked before `@ControllerAdvice`. `Ctrl` has `@ExceptionHandler(RuntimeException.class)`. `NullPointerException` IS-A `RuntimeException`. Controller-local handler matches. Response body: `"local"`. Global advice never consulted.

---

**Q7 — Select All That Apply**
Which argument types are available in `@ExceptionHandler` methods?

A) `@PathVariable` annotated parameters
B) The exception itself (typed to exception class or supertype)
C) `HttpServletRequest`
D) `Model`
E) `WebRequest`
F) `BindingResult`

**Answer: B, C, D, E**
A (`@PathVariable`) is NOT available — `@ExceptionHandler` runs after URL matching has already happened but the URL variable extraction context is not the same as regular handler invocation. F (`BindingResult`) is also NOT available — it's a request-binding concept, not an exception-handling concept. B, C, D, E are all supported parameter types in `@ExceptionHandler` methods.

---

**Q8 — MCQ**
`DefaultHandlerExceptionResolver` handles `HttpRequestMethodNotSupportedException`. It calls `response.sendError(405)` and returns an empty `ModelAndView`. A developer has `@ExceptionHandler(HttpRequestMethodNotSupportedException.class)` in a `@ControllerAdvice`. What actually handles the exception?

A) `DefaultHandlerExceptionResolver` — it has lower order number
B) The `@ControllerAdvice` `@ExceptionHandler` — `ExceptionHandlerExceptionResolver` runs first
C) Both are invoked — the last one's response wins
D) `DefaultHandlerExceptionResolver` — it's built-in and cannot be overridden

**Answer: B**
`ExceptionHandlerExceptionResolver` has order=0. `DefaultHandlerExceptionResolver` has order=2. Spring walks the resolver chain in order. `ExceptionHandlerExceptionResolver` finds the matching `@ExceptionHandler` in `@ControllerAdvice` → invokes it → returns empty `ModelAndView` → chain stops. `DefaultHandlerExceptionResolver` is NEVER reached. This is why you can override all default exception mappings with `@ExceptionHandler`.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — Controller-local ALWAYS beats @ControllerAdvice**
The most important ordering rule. Regardless of the exception type specificity, regardless of `@Order` on the advice, controller-local `@ExceptionHandler` is ALWAYS checked first. A controller with `@ExceptionHandler(RuntimeException.class)` will handle `NullPointerException` before a `@ControllerAdvice` with `@ExceptionHandler(NullPointerException.class)`. Local beats global.

**Trap 2 — `@ResponseStatus(reason=...)` commits the response**
`@ResponseStatus(code=NOT_FOUND)` without `reason` → `response.setStatus(404)` → response NOT committed → can still write body.
`@ResponseStatus(code=NOT_FOUND, reason="Not found")` → `response.sendError(404, "Not found")` → response COMMITTED → cannot write body → Spring's normal exception body handling bypassed → container's error page shows.
Many developers add `reason` thinking it sets a message in the JSON response. It doesn't — it goes to the container's error page.

**Trap 3 — `@ExceptionHandler` in `@RestControllerAdvice` cannot return view names**
Class-level `@ResponseBody` (via `@RestControllerAdvice`) applies to ALL methods including `@ExceptionHandler`. `String` return → text body. If you need to return HTML error pages from `@ControllerAdvice`, use `@ControllerAdvice` (without `@ResponseBody`) and return `ModelAndView` or `String`.

**Trap 4 — `ExceptionHandlerExceptionResolver` handles exception CAUSE chain**
If `WrapperException(cause=ProductNotFoundException)` is thrown, and there's `@ExceptionHandler(ProductNotFoundException.class)` but no handler for `WrapperException`, Spring WILL invoke the `ProductNotFoundException` handler by traversing the cause chain. Many developers don't know this and assume only the thrown exception type matters.

**Trap 5 — `@ExceptionHandler` exception type in annotation vs parameter**
```java
@ExceptionHandler  // No exception types specified in annotation
public void handle(ProductNotFoundException ex) {
    // Spring infers exception types from parameter type
}
// vs
@ExceptionHandler(ProductNotFoundException.class)
public void handle(Exception ex) {
    // Parameter is Exception supertype — annotation specifies what to catch
}
```
If `@ExceptionHandler` has no `value`, Spring infers from method parameter type. If `value` is specified, the annotation wins. The method parameter type only needs to be assignable from the handled exception — it can be a supertype.

**Trap 6 — Multiple `@ControllerAdvice` with same order causes non-deterministic behavior**
If two `@RestControllerAdvice` classes both handle the same exception type and have the same `@Order` (or no `@Order`), the one that gets invoked is unpredictable. Always assign unique `@Order` values to `@ControllerAdvice` beans that handle overlapping exception types. `HIGHEST_PRECEDENCE` for most specific, `LOWEST_PRECEDENCE` for catch-all.

**Trap 7 — `@ExceptionHandler` cannot handle exceptions thrown in `@ModelAttribute` methods differently than handler exceptions**
Exceptions thrown in `@ModelAttribute` methods (which run before the handler) ARE caught by `ExceptionHandlerExceptionResolver`. The `handler` parameter in `resolveException()` will be the `HandlerMethod` (the controller method), not the `@ModelAttribute` method. This is often surprising — you expected the `@ExceptionHandler` to know the exception came from model population, but it doesn't distinguish.

---

## 5️⃣ SUMMARY SHEET

```
THREE-LAYER EXCEPTION HANDLING ARCHITECTURE
─────────────────────────────────────────────────────
Layer 1: HandlerExceptionResolver chain (Spring MVC)
  ExceptionHandlerExceptionResolver (order=0)
    → @ExceptionHandler in controller (LOCAL — FIRST)
    → @ExceptionHandler in @ControllerAdvice (GLOBAL — after local)
  ResponseStatusExceptionResolver (order=1)
    → @ResponseStatus on exception class
    → ResponseStatusException thrown directly
  DefaultHandlerExceptionResolver (order=2)
    → Standard Spring MVC exceptions
    → Sets status code, returns empty MaV, no body

Layer 2: Servlet container error handling
  → When Layer 1 fails (all resolvers return null)
  → web.xml <error-page>

Layer 3: Spring Boot BasicErrorController
  → Catches /error dispatch
  → Renders structured error response

EXCEPTION HANDLER LOOKUP ORDER (within ExceptionHandlerExceptionResolver)
─────────────────────────────────────────────────────
1. Controller-local @ExceptionHandler — ALWAYS FIRST
   → ExceptionHandlerMethodResolver for controller class
   → Most specific exception type (by depth) wins

2. @ControllerAdvice @ExceptionHandler — ONLY if local fails
   → Iterate in @Order order
   → isApplicableToBeanType() check (scope filtering)
   → Most specific exception type within that advice wins

3. Cause chain traversal
   → If thrown exception not matched → check ex.getCause()
   → Recursive until match found or null cause

RESOLUTION RETURN VALUES
─────────────────────────────────────────────────────
null           → not handled, try next resolver
empty MaV      → handled, @ResponseBody wrote response, no view
MaV with view  → handled, render this error view

@EXCEPTIONHANDLER PARAMETER TYPES
─────────────────────────────────────────────────────
AVAILABLE:  Exception/Throwable (the exception), HandlerMethod,
            HttpServletRequest, HttpServletResponse, HttpSession,
            WebRequest, NativeWebRequest, Principal, Locale,
            UriComponentsBuilder, Model (for view-based advice)

NOT AVAILABLE: @PathVariable, @RequestParam, @RequestHeader,
               @CookieValue, @ModelAttribute, BindingResult

@RESPONSESTATUS BEHAVIOUR
─────────────────────────────────────────────────────
@ResponseStatus(HttpStatus.NOT_FOUND)
  → response.setStatus(404)
  → Response NOT committed
  → Body can still be written

@ResponseStatus(code=NOT_FOUND, reason="Not Found")
  → response.sendError(404, "Not Found")
  → Response COMMITTED immediately
  → Body cannot be written
  → Container handles error page
  → @ExceptionHandler runs INSTEAD if present (EHER order=0)

@CONTROLLERADVICE SCOPING
─────────────────────────────────────────────────────
No scope     → applies to ALL controllers
annotations  → only controllers with specified annotation
basePackages → only controllers in specified packages
assignableTypes → only specified controller types

@RestControllerAdvice = @ControllerAdvice + @ResponseBody
  → All @ExceptionHandler returns written as body (not view names)

DEFAULTHANDLEREXCEPTIONRESOLVER MAPPINGS
─────────────────────────────────────────────────────
HttpRequestMethodNotSupportedException → 405
HttpMediaTypeNotSupportedException     → 415
HttpMediaTypeNotAcceptableException    → 406
MissingServletRequestParameterException → 400
MethodArgumentTypeMismatchException     → 400
MethodArgumentNotValidException         → 400
BindException                           → 400
NoHandlerFoundException                 → 404
AsyncRequestTimeoutException            → 503
→ All can be overridden with @ExceptionHandler

MULTI-ADVICE ORDERING
─────────────────────────────────────────────────────
@Order(Ordered.HIGHEST_PRECEDENCE) → runs first
@Order(1) → runs before @Order(2)
No @Order → unpredictable — ASSIGN ORDER EXPLICITLY
Same order + overlapping exceptions → non-deterministic

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "Controller-local @ExceptionHandler ALWAYS beats @ControllerAdvice"
• "@ResponseStatus(reason=...) calls sendError() → commits response"
• "@RestControllerAdvice @ExceptionHandler: String return = text body, NOT view name"
• "ExceptionHandlerExceptionResolver traverses exception cause chain"
• "ExceptionHandlerExceptionResolver order=0, runs BEFORE ResponseStatus (order=1)"
• "All DefaultHandlerExceptionResolver mappings can be overridden with @ExceptionHandler"
• "@ControllerAdvice scope: empty = all; annotations= specific; assignableTypes= specific"
• "@ExceptionHandler inference: no annotation value → parameter type determines exception"
• "Exception in @ExceptionHandler itself: logged, null returned, original exception propagates"
• "Multiple @ControllerAdvice same order + overlapping types → non-deterministic"
```

---

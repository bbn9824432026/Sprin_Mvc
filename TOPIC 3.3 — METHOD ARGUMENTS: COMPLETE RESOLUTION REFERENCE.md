# TOPIC 3.3 — METHOD ARGUMENTS: COMPLETE RESOLUTION REFERENCE

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What Argument Resolution Really Is

When Spring MVC invokes a controller method, it must supply values for every parameter. Those values come from **many different sources**: the URL, request body, HTTP headers, cookies, session, model, the Spring container itself. The mechanism that maps each parameter to its source is **argument resolution** — one of the most feature-rich and complex systems in Spring MVC.

Understanding argument resolution at depth means knowing: which resolver handles which parameter type, what happens when multiple resolvers could claim the same parameter, what happens when resolution fails, how type conversion integrates, and how the dual-registration pattern works for fallback resolution.

---

### The Resolution Pipeline — How It Works

```
HandlerMethod has parameters: [Long id, Product product, Model model, ...]
        │
        ▼
InvocableHandlerMethod.getMethodArgumentValues()
        │
        ├── For each MethodParameter:
        │     │
        │     ├── resolvers.supportsParameter(parameter)?
        │     │     → Walk resolver chain in ORDER
        │     │     → First resolver returning true WINS
        │     │     → No resolver → IllegalStateException
        │     │
        │     └── resolvers.resolveArgument(parameter, mavContainer,
        │               webRequest, binderFactory)
        │           → Calls winning resolver's resolveArgument()
        │           → Returns Object (type conversion applied if needed)
        │           → Null OK for optional params
        │
        └── Returns Object[] args — one per parameter
              │
              ▼
        method.invoke(handlerBean, args)
```

---

### HandlerMethodArgumentResolverComposite — The Facade

```java
public class HandlerMethodArgumentResolverComposite
    implements HandlerMethodArgumentResolver {

    private final List<HandlerMethodArgumentResolver> argumentResolvers =
        new LinkedList<>();

    // Cache: MethodParameter → winning resolver
    // Prevents re-walking the chain on every request for same parameter
    private final Map<MethodParameter, HandlerMethodArgumentResolver>
        argumentResolverCache = new ConcurrentHashMap<>(256);

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return getArgumentResolver(parameter) != null;
    }

    @Override
    @Nullable
    public Object resolveArgument(
            MethodParameter parameter,
            @Nullable ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest,
            @Nullable WebDataBinderFactory binderFactory)
            throws Exception {

        HandlerMethodArgumentResolver resolver =
            getArgumentResolver(parameter);
        if (resolver == null) {
            throw new IllegalArgumentException(
                "Unsupported parameter type [" +
                parameter.getParameterType().getName() + "]");
        }
        return resolver.resolveArgument(
            parameter, mavContainer, webRequest, binderFactory);
    }

    @Nullable
    private HandlerMethodArgumentResolver getArgumentResolver(
            MethodParameter parameter) {

        // CHECK CACHE FIRST — O(1)
        HandlerMethodArgumentResolver result =
            this.argumentResolverCache.get(parameter);

        if (result == null) {
            // WALK CHAIN — O(n) first time only
            for (HandlerMethodArgumentResolver resolver :
                    this.argumentResolvers) {
                if (resolver.supportsParameter(parameter)) {
                    result = resolver;
                    // CACHE the winning resolver
                    this.argumentResolverCache.put(
                        parameter, result);
                    break;
                }
            }
        }
        return result;
    }
}
```

**The cache is critical for performance.** The resolver chain has 30+ resolvers. Walking it for every parameter on every request would be expensive. The cache ensures each distinct `MethodParameter` (unique per controller method) only walks the chain ONCE. Subsequent requests use the cached resolver directly — O(1).

---

### The Complete Resolver Chain — All 30+ Resolvers in Order

```
── ANNOTATION-BASED RESOLVERS (checked FIRST) ──────────────────

[1] RequestParamMethodArgumentResolver (useDefaultResolution=false)
    supportsParameter:
      → @RequestParam on simple type or MultipartFile/Part
      → NOT on Map (handled by [3])
    resolveArgument:
      → Reads request.getParameter(name) or request.getPart(name)
      → Applies type conversion
      → required=true + missing + no defaultValue → MissingServletRequestParameterException

[2] RequestParamMapMethodArgumentResolver
    supportsParameter:
      → @RequestParam on Map<String,String> or MultiValueMap
    resolveArgument:
      → Returns request.getParameterMap() as Map
      → Returns request.getParameterMap() as MultiValueMap<String,String>

[3] PathVariableMethodArgumentResolver
    supportsParameter:
      → @PathVariable on simple type
    resolveArgument:
      → Reads URI_TEMPLATE_VARIABLES_ATTRIBUTE from request
      → Gets value for variable name
      → Applies type conversion

[4] PathVariableMapMethodArgumentResolver
    supportsParameter:
      → @PathVariable on Map<String,String>
    resolveArgument:
      → Returns entire URI_TEMPLATE_VARIABLES_ATTRIBUTE map

[5] MatrixVariableMethodArgumentResolver
    supportsParameter:
      → @MatrixVariable on non-Map
    resolveArgument:
      → Reads MATRIX_VARIABLES_ATTRIBUTE
      → Finds variable in specified path variable segment

[6] MatrixVariableMapMethodArgumentResolver
    supportsParameter:
      → @MatrixVariable on Map

[7] ServletModelAttributeMethodProcessor (supports=false)
    supportsParameter:
      → @ModelAttribute on non-simple types
      → Also handles non-annotated complex types (as fallback — see [29])
    resolveArgument:
      → Creates instance (default constructor or @ModelAttribute factory)
      → WebDataBinder.bind() — populates from request params
      → Validates if @Valid/@Validated present
      → Stores in model + session if @SessionAttributes
      → Puts BindingResult in model AFTER the bound object

[8] RequestResponseBodyMethodProcessor (as argument resolver)
    supportsParameter:
      → @RequestBody
    resolveArgument:
      → Reads request.getInputStream() — ONE TIME ONLY
      → Content-Type → selects HttpMessageConverter
      → Deserialises to parameter type
      → Validates if @Valid/@Validated present
      → No BindingResult in model (unlike @ModelAttribute)

[9] RequestPartMethodArgumentResolver
    supportsParameter:
      → @RequestPart (multipart file + body combined)
    resolveArgument:
      → Gets MultipartFile or parsed body from multipart part

[10] RequestHeaderMethodArgumentResolver
    supportsParameter:
      → @RequestHeader on non-Map
    resolveArgument:
      → request.getHeader(name)
      → Type conversion (String → target type)

[11] RequestHeaderMapMethodArgumentResolver
    supportsParameter:
      → @RequestHeader on Map or HttpHeaders

[12] ServletCookieValueMethodArgumentResolver
    supportsParameter:
      → @CookieValue on non-Map
    resolveArgument:
      → Reads Cookie[] from request
      → Finds cookie by name

[13] ExpressionValueMethodArgumentResolver
    supportsParameter:
      → @Value
    resolveArgument:
      → Evaluates Spring EL expression or ${} placeholder
      → Uses Environment + ConversionService

[14] SessionAttributeMethodArgumentResolver
    supportsParameter:
      → @SessionAttribute
    resolveArgument:
      → request.getSession().getAttribute(name)
      → required=true + missing → exception

[15] RequestAttributeMethodArgumentResolver
    supportsParameter:
      → @RequestAttribute
    resolveArgument:
      → request.getAttribute(name)

── TYPE-BASED RESOLVERS (checked after annotation resolvers) ────

[16] ServletRequestMethodArgumentResolver
    supportsParameter: resolves many types by type-matching:
      → HttpServletRequest, ServletRequest, WebRequest, NativeWebRequest
      → MultipartRequest, MultipartHttpServletRequest (if multipart)
      → HttpSession (CREATES session if needed)
      → PushBuilder (HTTP/2)
      → Principal (request.getUserPrincipal())
      → InputStream, Reader (request body)
      → HttpMethod (request.getMethod())
      → Locale (from LocaleContextHolder)
      → TimeZone, ZoneId (from LocaleContextHolder)

[17] ServletResponseMethodArgumentResolver
    supportsParameter:
      → HttpServletResponse, ServletResponse
      → OutputStream, Writer (response body)
      → Sets mavContainer.requestHandled=true (response being written directly)

[18] HttpEntityMethodProcessor (as argument resolver)
    supportsParameter:
      → HttpEntity<T>, RequestEntity<T>
    resolveArgument:
      → Reads entire request including headers + body
      → Returns HttpEntity<T> with headers and deserialized body

[19] RedirectAttributesMethodArgumentResolver
    supportsParameter:
      → RedirectAttributes (interface)
    resolveArgument:
      → Returns FlashMap-backed RedirectAttributes
      → Flash attributes stored here survive redirect

[20] ModelMethodProcessor
    supportsParameter:
      → Model (interface), ModelMap
    resolveArgument:
      → Returns mavContainer.getModel() (the current model)
      → Changes to model visible in view

[21] MapMethodProcessor
    supportsParameter:
      → Map (when not annotated — returns model map)
    resolveArgument:
      → Returns mavContainer.getModel() as Map

[22] ErrorsMethodArgumentResolver
    supportsParameter:
      → Errors (interface), BindingResult
    resolveArgument:
      → Finds BindingResult in model for PRECEDING @ModelAttribute param
      → MUST come immediately after @ModelAttribute in parameter list
      → If no preceding @ModelAttribute → IllegalStateException

[23] SessionStatusMethodArgumentResolver
    supportsParameter:
      → SessionStatus
    resolveArgument:
      → Returns SessionStatus from mavContainer
      → sessionStatus.setComplete() clears @SessionAttributes

[24] UriComponentsBuilderMethodArgumentResolver
    supportsParameter:
      → UriComponentsBuilder, ServletUriComponentsBuilder
    resolveArgument:
      → Builds UriComponentsBuilder from current request context
      → Used for building URLs relative to current request

[25] PrincipalMethodArgumentResolver
    supportsParameter:
      → Principal (when NOT already handled by [16])
      → Specifically for custom Principal implementations

── ASYNC / REACTIVE RESOLVERS ───────────────────────────────────

[26] ContinuationHandlerMethodArgumentResolver (Kotlin only)
    → Kotlin coroutines support

── CATCH-ALL RESOLVERS (LAST in chain) ──────────────────────────

[27] RequestParamMethodArgumentResolver (useDefaultResolution=true)
    supportsParameter:
      → ANY simple type parameter WITHOUT @RequestParam
      → String, int, long, double, boolean, enum, etc.
    resolveArgument:
      → Uses parameter NAME as request param name
      → public void method(String name) → from request.getParameter("name")

[28] ServletModelAttributeMethodProcessor (supports=true)
    supportsParameter:
      → ANY non-simple type parameter WITHOUT @ModelAttribute
    resolveArgument:
      → Same as [7] but for unannotated complex types
      → Last resort for complex types
```

---

### Group 1: @RequestParam — Full Analysis

```java
// @RequestParam full API
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequestParam {
    String value() default "";    // Parameter name (alias for name)
    String name() default "";     // Parameter name
    boolean required() default true;  // MissingServletRequestParameterException if absent
    String defaultValue() default ValueConstants.DEFAULT_NONE;
    // defaultValue: if set → required becomes effectively false
}
```

**Resolution scenarios:**

```java
@GetMapping("/products")
public void examples(
    // 1. Named, required (default)
    @RequestParam("category") String category,
    // GET /products?category=books → category="books"
    // GET /products → MissingServletRequestParameterException → 400

    // 2. Optional with default
    @RequestParam(value="page", defaultValue="0") int page,
    // GET /products → page=0 (default)
    // GET /products?page=2 → page=2
    // Note: defaultValue makes required irrelevant

    // 3. Optional with Optional<T>
    @RequestParam("sort") Optional<String> sort,
    // GET /products → sort=Optional.empty()
    // GET /products?sort=name → sort=Optional.of("name")

    // 4. Required=false
    @RequestParam(value="filter", required=false) String filter,
    // GET /products → filter=null
    // GET /products?filter=active → filter="active"

    // 5. All params as Map
    @RequestParam Map<String, String> allParams,
    // GET /products?a=1&b=2 → {a:1, b:2}
    // Only first value per key — use MultiValueMap for multiple

    // 6. Multi-value parameter
    @RequestParam List<String> tags,
    // GET /products?tags=a&tags=b&tags=c → ["a","b","c"]

    // 7. Without annotation — catch-all resolver
    String name,
    // GET /products?name=widget → name="widget" (no @RequestParam needed)
    // Handled by RequestParamMethodArgumentResolver(useDefaultResolution=true)
    // required=true by default when annotation absent
) { ... }
```

---

### Group 2: @PathVariable — Full Analysis

```java
// URI template variables
@GetMapping("/products/{id}/reviews/{reviewId}")
public void examples(
    // 1. Named path variable
    @PathVariable("id") Long id,
    // Reads from URI_TEMPLATE_VARIABLES_ATTRIBUTE["id"]
    // Type conversion: String "42" → Long 42

    // 2. Name inferred from parameter name
    @PathVariable Long reviewId,
    // Variable name = parameter name "reviewId"
    // Only works if parameter name available (debug info compiled in)

    // 3. Required path variable (default)
    @PathVariable(required=true) Long id,
    // Missing → MethodArgumentConversionNotSupportedException if wrong type

    // 4. Optional path variable (unusual — URL structure implies presence)
    @PathVariable(required=false) Long optionalId,

    // 5. All path variables as Map
    @PathVariable Map<String, String> pathVars,
    // → {"id": "42", "reviewId": "7"}

    // 6. Type conversion failure
    // GET /products/abc/reviews/7 → id="abc" → Long conversion fails
    // → MethodArgumentTypeMismatchException → 400
) { ... }
```

**How URI template variables are extracted:**

```
RequestMappingHandlerMapping finds match:
  Pattern: /products/{id}/reviews/{reviewId}
  Request:  /products/42/reviews/7

  UriTemplateVariables = {"id": "42", "reviewId": "7"}
  Set as request attribute: URI_TEMPLATE_VARIABLES_ATTRIBUTE

PathVariableMethodArgumentResolver.resolveArgument():
  → reads attribute from request
  → gets value for parameter's "id" or "reviewId" key
  → ConversionService.convert("42", Long.class) → 42L
```

---

### Group 3: @RequestBody — Full Analysis

```java
// @RequestBody — reads entire request body
@PostMapping("/products")
public Product create(
    @RequestBody Product product
    // RequestResponseBodyMethodProcessor.resolveArgument():
    // 1. Checks Content-Type header
    // 2. Walks HttpMessageConverter chain
    // 3. Finds MappingJackson2HttpMessageConverter for application/json
    // 4. Reads request.getInputStream() — ONE TIME ONLY
    // 5. Jackson deserialises → Product instance
    // 6. Stream consumed — cannot read again
) { ... }

// With validation
@PostMapping("/products")
public Product createValidated(
    @Valid @RequestBody Product product,
    // @Valid triggers JSR-303 validation AFTER deserialization
    // Validation failure → MethodArgumentNotValidException → 400
    // NOT propagated to BindingResult (unlike @ModelAttribute)

    BindingResult result  // THIS IS WRONG for @RequestBody
    // BindingResult after @RequestBody is IGNORED
    // Exception is thrown regardless of BindingResult presence
    // CORRECT: use @ExceptionHandler(MethodArgumentNotValidException.class)
) { ... }

// HttpEntity — includes headers
@PostMapping("/products")
public Product createViaEntity(
    HttpEntity<Product> entity
    // entity.getBody() → deserialized Product
    // entity.getHeaders() → all request headers
    // Handled by HttpEntityMethodProcessor, not RequestBody resolver
) { ... }
```

---

### Group 4: @ModelAttribute — Full Analysis

```java
// @ModelAttribute — data binding from request parameters
@PostMapping("/products")
public String create(
    @ModelAttribute("product") Product product,
    // ModelAttributeMethodProcessor.resolveArgument():
    // 1. Look for "product" in model (from @ModelAttribute method or @SessionAttributes)
    // 2. If not found: create new instance via default constructor
    // 3. WebDataBinder.bind():
    //    → Gets request parameters
    //    → Maps to Product fields by name
    //    → product.setName(request.getParameter("name"))
    //    → product.setPrice(Double.parseDouble(request.getParameter("price")))
    //    → Type conversion via ConversionService
    // 4. Validation if @Valid/@Validated
    // 5. Adds Product to model as "product"
    // 6. Adds BindingResult to model as "org.springframework.validation.BindingResult.product"
    //    IMMEDIATELY AFTER the product (adjacent in model)

    BindingResult result  // CORRECT for @ModelAttribute
    // BindingResult for @ModelAttribute IS populated
    // MUST be immediately after the @ModelAttribute parameter
    // If result.hasErrors() → return to form view
) { ... }
```

**@ModelAttribute resolution priority:**

```
1. Already in model? (from @ModelAttribute method)
   → Use existing instance and bind on top

2. In @SessionAttributes?
   → Retrieve from session, bind on top

3. Neither found → Create new instance:
   → Default constructor
   → OR @ModelAttribute factory method (see below)
   → Bind request params onto new instance
```

---

### Group 5: Request Infrastructure Objects

```java
@GetMapping("/info")
public void accessEverything(
    // Servlet objects — injected directly
    HttpServletRequest request,
    HttpServletResponse response,
    HttpSession session,        // Creates if not exists

    // Spring abstractions
    WebRequest webRequest,      // Abstraction over HttpServletRequest
    NativeWebRequest nativeReq, // WebRequest with getNativeRequest()

    // Request metadata
    HttpMethod method,          // HttpMethod.GET
    Locale locale,              // From LocaleContextHolder
    TimeZone tz,                // From LocaleContextHolder
    ZoneId zoneId,              // From LocaleContextHolder

    // Security
    Principal principal,        // request.getUserPrincipal()

    // I/O streams
    InputStream inputStream,    // request.getInputStream()
    Reader reader,              // request.getReader()
    OutputStream outputStream,  // response.getOutputStream()
    Writer writer,              // response.getWriter()

    // Model and binding
    Model model,                // mavContainer.getModel()
    ModelMap modelMap,          // Same as Model
    RedirectAttributes redirectAttrs,  // For POST-Redirect-GET
    SessionStatus sessionStatus,       // Clear @SessionAttributes

    // URL building
    UriComponentsBuilder ucb    // For building URLs
) { ... }
```

---

### Type Conversion — How String → Target Type Works

Request parameters are always **Strings**. The target parameter type may be `Long`, `Date`, `enum`, etc. Spring MVC uses `ConversionService` to bridge this:

```
String "42" → Long 42:
  ConversionService.canConvert(String.class, Long.class) → true
  ConversionService.convert("42", Long.class) → 42L

String "2024-03-15" → LocalDate:
  → No built-in converter by default
  → Register: @Bean ConversionServiceFactoryBean with custom converters
  → OR: @DateTimeFormat(iso=ISO.DATE) on parameter

String "ACTIVE" → Status enum:
  → StringToEnumConverterFactory
  → Status.valueOf("ACTIVE") → Status.ACTIVE
  → Case-sensitive! "active" → ConversionFailedException

Type conversion failure:
  → MethodArgumentTypeMismatchException (wraps ConversionFailedException)
  → DefaultHandlerExceptionResolver → HTTP 400

Null for primitive types:
  → @RequestParam int id when id not present and required=false
  → NPE during unboxing null → NullPointerException
  → Use Integer (boxed) or provide defaultValue="0"
```

---

### WebDataBinder — For @ModelAttribute Binding

```java
// WebDataBinder internals for @ModelAttribute
WebDataBinder binder = binderFactory.createBinder(
    webRequest,
    targetObject,   // Product instance
    objectName      // "product"
);

// Binding process:
binder.bind(new MutablePropertyValues(
    webRequest.getParameterMap()));
// → BeanWrapper.setPropertyValue("name", "Widget")
// → BeanWrapper.setPropertyValue("price", "9.99")
// → Type conversion via ConversionService
// → Nested object binding: "address.street" → product.address.street
// → Collection binding: "tags[0]" → product.tags.get(0)

// After binding, validation if @Valid present:
SmartValidator validator = binder.getValidator();
validator.validate(targetObject, binder.getBindingResult());

// BindingResult stored in model:
mavContainer.getModel().put(
    BindingResult.MODEL_KEY_PREFIX + objectName,
    binder.getBindingResult());
// KEY = "org.springframework.validation.BindingResult.product"
```

---

### @SessionAttributes and @SessionAttribute — Distinction

```java
// CLASS-LEVEL: declares which MODEL attributes should be stored in session
@Controller
@SessionAttributes({"cart", "user"})  // Store these in session
public class CheckoutController {

    @GetMapping("/checkout")
    public String checkout(
            Model model,
            SessionStatus sessionStatus) {
        // "cart" from model → automatically stored in session
        // Next request: "cart" retrieved from session into model
        return "checkout";
    }

    @PostMapping("/checkout/complete")
    public String complete(SessionStatus status) {
        status.setComplete();  // Clears "cart" and "user" from session
        return "redirect:/home";
    }
}

// METHOD PARAMETER: read a SPECIFIC session attribute
@GetMapping("/dashboard")
public String dashboard(
    @SessionAttribute("user") User user,  // Direct session read
    // Different from @SessionAttributes class annotation
    // This reads DIRECTLY from session — not via model
    // required=true by default
    @SessionAttribute(name="preferences", required=false)
        UserPreferences prefs
) { ... }
```

---

### BindingResult — Critical Rules

```java
// RULE 1: BindingResult must IMMEDIATELY follow its @ModelAttribute
@PostMapping("/form")
public String submit(
    @ModelAttribute("user") User user,
    BindingResult userResult,      // ← CORRECT position
    @ModelAttribute("address") Address address,
    BindingResult addressResult    // ← CORRECT position
) { ... }

// WRONG: BindingResult separated from @ModelAttribute
@PostMapping("/form")
public String wrong(
    @ModelAttribute("user") User user,
    String extraParam,             // ← WRONG — breaks binding
    BindingResult result           // ErrorsMethodArgumentResolver FAILS
    // IllegalStateException: "An Errors/BindingResult argument is expected
    //  to be declared immediately after the model attribute"
) { ... }

// RULE 2: @RequestBody with BindingResult — BindingResult ignored
@PostMapping("/api")
public String api(
    @Valid @RequestBody Product product,
    BindingResult result  // ← This is actually processed differently
    // For @RequestBody: if valid → method runs, result.hasErrors()=false
    // For @RequestBody: if invalid → MethodArgumentNotValidException THROWN
    //                                BEFORE method runs (result never used)
    // Better approach: remove BindingResult, use @ExceptionHandler
) { ... }
```

---

### @Value in Controller Parameters

```java
@RestController
public class ConfigController {

    // @Value in controller PARAMETERS (not fields)
    @GetMapping("/config")
    public String config(
        @Value("${app.version}") String version,
        // ExpressionValueMethodArgumentResolver
        // Resolves at request time (but value is cached after first resolution)

        @Value("#{systemProperties['user.timezone']}") String tz,
        // Spring EL expression — evaluated per request
        // Can access beans: @Value("#{myBean.property}")

        @Value("${app.max-items:100}") int maxItems
        // Default value syntax: ${key:default}
    ) {
        return version + " tz=" + tz + " max=" + maxItems;
    }
}

// NOTE: @Value on method parameters is per-request evaluation
// @Value on fields is evaluated ONCE at bean creation
// Method parameter @Value is rarely needed — prefer field injection
```

---

### Argument Resolution Failure Scenarios

```java
// FAILURE 1: No resolver claims parameter
// (Custom type without annotation + not simple type)
@GetMapping("/test")
public void test(ComplexCustomType obj) {
    // Walk resolver chain — no match
    // → IllegalStateException: "No suitable resolver for argument [0]
    //    of type 'ComplexCustomType'"
    // Fix: add @ModelAttribute or implement custom ArgumentResolver
}

// FAILURE 2: @RequestParam required missing
@GetMapping("/search")
public void search(@RequestParam String q) {
    // GET /search (no q param)
    // → MissingServletRequestParameterException
    // → DefaultHandlerExceptionResolver → 400
}

// FAILURE 3: Type conversion failure
@GetMapping("/products/{id}")
public void get(@PathVariable Long id) {
    // GET /products/abc
    // String "abc" → Long fails
    // → MethodArgumentTypeMismatchException → 400
}

// FAILURE 4: @RequestBody deserialization failure
@PostMapping("/products")
public void create(@RequestBody Product product) {
    // POST with body: "this is not json"
    // Content-Type: application/json
    // Jackson parse fails
    // → HttpMessageNotReadableException → 400
}

// FAILURE 5: @Valid validation failure for @RequestBody
@PostMapping("/products")
public void createValid(@Valid @RequestBody Product product) {
    // Valid JSON but Product.name = null (required)
    // → MethodArgumentNotValidException → 400
    // @ExceptionHandler needed to handle gracefully
}
```

---

## 2️⃣ CODE EXAMPLES

### Complete Parameter Resolution Showcase

```java
@RestController
@RequestMapping("/showcase")
public class ArgumentShowcaseController {

    // ── URL Parameters ──────────────────────────────────────────────

    @GetMapping("/path/{id}/sub/{name}")
    public Map<String, Object> pathVariables(
            @PathVariable Long id,
            @PathVariable String name,
            @PathVariable Map<String, String> allVars) {
        return Map.of(
            "id", id,
            "name", name,
            "allVars", allVars
        );
        // GET /showcase/path/42/sub/widget
        // → {id:42, name:"widget", allVars:{id:"42",name:"widget"}}
    }

    @GetMapping("/params")
    public Map<String, Object> requestParams(
            @RequestParam("q") String query,
            @RequestParam(value="page", defaultValue="0") int page,
            @RequestParam(required=false) String filter,
            @RequestParam Optional<String> sort,
            @RequestParam(value="tags") List<String> tags,
            @RequestParam MultiValueMap<String, String> allParams) {
        return Map.of(
            "q", query,
            "page", page,
            "filter", filter,
            "sort", sort,
            "tags", tags,
            "allParams", allParams
        );
    }

    // ── HTTP Headers ────────────────────────────────────────────────

    @GetMapping("/headers")
    public Map<String, String> headers(
            @RequestHeader("Authorization") String auth,
            @RequestHeader(value="X-Custom", required=false)
                String custom,
            @RequestHeader HttpHeaders allHeaders,
            @RequestHeader Map<String, String> headerMap) {
        return Map.of(
            "auth", auth,
            "custom", custom != null ? custom : "absent",
            "contentType",
                allHeaders.getContentType() != null ?
                    allHeaders.getContentType().toString() : "none"
        );
    }

    // ── Cookies ─────────────────────────────────────────────────────

    @GetMapping("/cookies")
    public Map<String, String> cookies(
            @CookieValue("JSESSIONID") String sessionId,
            @CookieValue(value="pref", required=false,
                         defaultValue="en") String preference) {
        return Map.of(
            "sessionId", sessionId,
            "pref", preference
        );
    }

    // ── Request Body ────────────────────────────────────────────────

    @PostMapping(value="/body",
                 consumes=MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<Map<String, Object>> requestBody(
            @Valid @RequestBody ProductDTO dto,
            @RequestHeader("X-Trace-Id") String traceId) {
        return ResponseEntity.ok(Map.of(
            "received", dto,
            "trace", traceId
        ));
    }

    // ── Infrastructure Objects ──────────────────────────────────────

    @GetMapping("/infra")
    public void infrastructure(
            HttpServletRequest request,
            HttpServletResponse response,
            HttpSession session,
            Locale locale,
            HttpMethod method,
            Principal principal,
            Model model) {
        model.addAttribute("uri", request.getRequestURI());
        model.addAttribute("locale", locale.getLanguage());
        model.addAttribute("method", method.name());
        model.addAttribute("user",
            principal != null ? principal.getName() : "anonymous");
        response.setHeader("X-Processed", "true");
    }
}
```

---

### Custom HandlerMethodArgumentResolver — Pagination

```java
// Custom annotation
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface PageableDefault {
    int page() default 0;
    int size() default 20;
    String sort() default "id";
}

// Custom data object
public record PageRequest(int page, int size, String sort) {}

// Custom resolver
@Component
public class PageableArgumentResolver
    implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.getParameterType() == PageRequest.class &&
               parameter.hasParameterAnnotation(PageableDefault.class);
    }

    @Override
    public Object resolveArgument(
            MethodParameter parameter,
            @Nullable ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest,
            @Nullable WebDataBinderFactory binderFactory) {

        PageableDefault defaults =
            parameter.getParameterAnnotation(PageableDefault.class);

        int page = parseInt(webRequest.getParameter("page"),
            defaults.page());
        int size = parseInt(webRequest.getParameter("size"),
            defaults.size());
        String sort = webRequest.getParameter("sort") != null ?
            webRequest.getParameter("sort") : defaults.sort();

        // Validate bounds
        size = Math.min(size, 100); // Max 100 per page
        page = Math.max(page, 0);   // Min page 0

        return new PageRequest(page, size, sort);
    }

    private int parseInt(String value, int defaultValue) {
        if (value == null) return defaultValue;
        try { return Integer.parseInt(value); }
        catch (NumberFormatException e) { return defaultValue; }
    }
}

// Registration
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    @Autowired
    private PageableArgumentResolver pageableResolver;

    @Override
    public void addArgumentResolvers(
            List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(pageableResolver);
        // Added after all defaults
        // supportsParameter() is called AFTER all built-in resolvers
    }
}

// Usage
@GetMapping("/products")
public List<Product> getProducts(
        @PageableDefault(size=10, sort="name") PageRequest pageable) {
    // GET /products → PageRequest(0, 10, "name")
    // GET /products?page=2&size=5 → PageRequest(2, 5, "name")
    // GET /products?sort=price → PageRequest(0, 10, "price")
    return productService.findAll(pageable);
}
```

---

### @ModelAttribute With BindingResult — Full Form Pattern

```java
@Controller
@RequestMapping("/users")
@SessionAttributes("registrationData")
public class RegistrationController {

    @ModelAttribute("registrationData")
    public RegistrationForm initRegistrationForm() {
        // Called BEFORE handler methods
        // Puts empty form in model + session (via @SessionAttributes)
        return new RegistrationForm();
    }

    @GetMapping("/register")
    public String showForm(Model model) {
        // "registrationData" already in model from @ModelAttribute method
        return "user/register";
    }

    @PostMapping("/register")
    public String processForm(
            @Valid @ModelAttribute("registrationData")
                RegistrationForm form,
            // 1. Look in session for "registrationData" (found - from GET)
            // 2. Bind POST params on top of session object
            // 3. Validate with @Valid
            // 4. Add to model

            BindingResult result,
            // MUST immediately follow @ModelAttribute
            // Populated with validation/binding errors

            SessionStatus sessionStatus,
            RedirectAttributes redirectAttrs) {

        if (result.hasErrors()) {
            // Validation failed — show form again with errors
            // "registrationData" in model has user's invalid input
            return "user/register";
        }

        userService.register(form);

        // Clear session (remove "registrationData" from session)
        sessionStatus.setComplete();

        // Flash message for redirect
        redirectAttrs.addFlashAttribute("success",
            "Registration complete!");

        return "redirect:/users/login";
    }
}
```

---

### Edge Case — Argument Resolution Order and Precedence

```java
@RestController
public class ResolutionOrderDemo {

    // DEMO: Three resolvers could potentially claim 'name'
    // 1. @RequestParam String name → [1] RequestParamMethodArgumentResolver
    // 2. @PathVariable String name → [3] PathVariableMethodArgumentResolver  
    // 3. No annotation String name → [27] catch-all RequestParam resolver
    // The ANNOTATION determines which resolver is used

    @GetMapping("/test/{name}")
    public String demo(
            @PathVariable String name,
            // → PathVariableMethodArgumentResolver (has @PathVariable)
            // Reads from URI template variables: name="widget"

            @RequestParam(required=false) String name2,
            // → RequestParamMethodArgumentResolver (has @RequestParam)
            // Reads from query string: ?name2=value

            String name3
            // → catch-all RequestParamMethodArgumentResolver
            // Reads from query string as ?name3=value
    ) {
        return name + " " + name2 + " " + name3;
    }
    // GET /test/widget?name2=foo&name3=bar
    // → "widget foo bar"
}
```

---

### Edge Case — BindingResult MUST Be Adjacent

```java
@Controller
public class BindingOrderController {

    // CORRECT: BindingResult immediately follows its @ModelAttribute
    @PostMapping("/correct")
    public String correct(
            @ModelAttribute User user,
            BindingResult userResult,
            @ModelAttribute Address address,
            BindingResult addressResult) {
        // userResult corresponds to user
        // addressResult corresponds to address
        return "view";
    }

    // WRONG: Extra parameter between @ModelAttribute and BindingResult
    @PostMapping("/wrong")
    public String wrong(
            @ModelAttribute User user,
            @RequestParam String extra,  // ← breaks adjacency
            BindingResult result) {
        // IllegalStateException at RUNTIME:
        // ErrorsMethodArgumentResolver cannot find preceding model attribute
        return "view";
    }

    // VALID: HttpServletRequest between pairs is OK
    // (not a model attribute — doesn't break the pair)
    @PostMapping("/valid")
    public String valid(
            @ModelAttribute User user,
            BindingResult userResult,    // ← correctly adjacent
            HttpServletRequest request,  // ← not a model attribute
            @ModelAttribute Address addr,
            BindingResult addrResult) {  // ← correctly adjacent
        return "view";
    }
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
`HandlerMethodArgumentResolverComposite` caches which resolver handles which `MethodParameter`. When is this cache populated?

A) At application startup when handler methods are registered
B) On the first request that invokes a method with that parameter type
C) During `afterPropertiesSet()` of `RequestMappingHandlerAdapter`
D) Each request re-walks the resolver chain — no caching

**Answer: B**
The cache is populated lazily — on the first request that resolves a given `MethodParameter`. `getArgumentResolver()` checks the cache first; on miss, walks the chain and stores the winning resolver. Subsequent requests for the same method's parameters find the resolver in O(1) from the cache. The cache is not pre-populated at startup.

---

**Q2 — Select All That Apply**
Which of the following are TRUE about `@RequestParam` without `required=false` or `defaultValue`?

A) Parameter is required — `MissingServletRequestParameterException` if absent
B) If parameter is a primitive type and absent, `NullPointerException` during unboxing
C) If parameter is `Optional<String>`, no exception when absent
D) `required=true` is the default — can be omitted
E) `defaultValue` makes the `required` attribute irrelevant

**Answer: A, C, D, E**
B is wrong — `MissingServletRequestParameterException` is thrown BEFORE unboxing; there's no unboxing of null. However, if you use `required=false` with a primitive type, you GET null which causes NPE during unboxing — but with default `required=true`, the exception is thrown first. C is correct — `Optional<String>` type causes Spring to treat the parameter as optional regardless of `required`. D and E are both correct.

---

**Q3 — Code Prediction**
```java
@Controller
public class FormController {

    @PostMapping("/submit")
    public String submit(
            @ModelAttribute User user,
            String extra,
            BindingResult result) {
        return "view";
    }
}
```
`POST /submit?name=John&extra=data` is called. What happens?

A) Method executes normally — user bound, extra="data", result empty
B) `IllegalStateException` — BindingResult is not adjacent to `@ModelAttribute`
C) `MissingServletRequestParameterException` — `extra` not annotated
D) `IllegalStateException` at startup — invalid method signature

**Answer: B**
`ErrorsMethodArgumentResolver.resolveArgument()` looks for the PRECEDING model attribute in the parameter list. The preceding parameter is `String extra` — not a model attribute. The resolver cannot find the corresponding `@ModelAttribute` parameter. Throws `IllegalStateException: "An Errors/BindingResult argument is expected to be declared immediately after the model attribute, the @RequestBody or the @RequestPart arguments"`. This occurs at RUNTIME on first request, not at startup.

---

**Q4 — MCQ**
A controller method has `@Valid @RequestBody Product product, BindingResult result`. Product fails validation. What happens?

A) `result.hasErrors()` returns `true`, method executes with errors
B) `MethodArgumentNotValidException` thrown BEFORE method executes
C) `BindingException` thrown, method executes with populated BindingResult
D) `HttpMessageNotReadableException` thrown — JSON parse failure

**Answer: B**
For `@RequestBody` with `@Valid`: `RequestResponseBodyMethodProcessor.resolveArgument()` deserialises the body, then runs validation. If validation fails → `MethodArgumentNotValidException` is thrown IMMEDIATELY — before the method executes. The `BindingResult` parameter presence does NOT suppress the exception (unlike `@ModelAttribute` where `BindingResult` presence suppresses the validation exception).

---

**Q5 — Select All That Apply**
Which parameter types are resolved by `ServletRequestMethodArgumentResolver` WITHOUT any annotation?

A) `HttpServletRequest`
B) `Principal`
C) `HttpMethod`
D) `InputStream`
E) `RedirectAttributes`

**Answer: A, C, D**
`HttpServletRequest` (A), `HttpMethod` (C), `InputStream` (D) are all handled by `ServletRequestMethodArgumentResolver` by type-matching without annotation. `Principal` (B) is primarily handled by `PrincipalMethodArgumentResolver` (a separate resolver). `RedirectAttributes` (E) is handled by `RedirectAttributesMethodArgumentResolver`.

---

**Q6 — True/False**
A controller method parameter of type `Map<String, String>` without any annotation is resolved by `MapMethodProcessor` as the current model map.

**Answer: True**
`MapMethodProcessor.supportsParameter()` returns `true` for `Map` type without any annotation. It injects the model map (`mavContainer.getModel()`). If the developer intended to receive all request parameters, they should use `@RequestParam Map<String, String>` — without the annotation, they get the MODEL, not the request parameters. This is a classic trap — unannotated `Map` = model map, not request params.

---

**Q7 — MCQ**
`@PathVariable` annotation is on a `Long` parameter. The URL template is `/products/{id}`. Request arrives for `GET /products/abc`. What exception and HTTP status?

A) `NumberFormatException` — HTTP 500 (unhandled)
B) `MethodArgumentTypeMismatchException` — HTTP 400
C) `MethodArgumentConversionNotSupportedException` — HTTP 400
D) `PathVariableNotFoundException` — HTTP 404

**Answer: B**
`PathVariableMethodArgumentResolver` extracts `"abc"` from URI variables, then calls `ConversionService.convert("abc", Long.class)`. Conversion fails → `MethodArgumentTypeMismatchException` is thrown. `DefaultHandlerExceptionResolver` handles this → HTTP 400 Bad Request.

---

**Q8 — Ordering**
Order the argument resolution steps for `@Valid @ModelAttribute("product") Product product`:

- `BindingResult` added to model
- Validation run (JSR-303)
- `Product` instance added to model as "product"
- WebDataBinder created
- `Product` instance found or created
- Request params bound to `Product` fields

**Correct Order:**
1. `Product` instance found or created (from model/session/new)
2. `WebDataBinder` created (with @InitBinder applied)
3. Request params bound to `Product` fields
4. Validation run (JSR-303 — @Valid triggers this)
5. `Product` instance added to model as "product"
6. `BindingResult` added to model (immediately after product)

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — Unannotated `Map` parameter = model map, not request params**
`Map<String, String>` without annotation → `MapMethodProcessor` → injects model map. `@RequestParam Map<String, String>` → `RequestParamMapMethodArgumentResolver` → injects request parameters. The annotation completely changes the source. This catches developers who remove `@RequestParam` thinking it's optional for `Map`.

**Trap 2 — `BindingResult` for `@RequestBody` doesn't suppress exceptions**
For `@ModelAttribute`, adding `BindingResult` immediately after suppresses `BindException` — validation errors go into the result, method executes. For `@RequestBody` with `@Valid`, `BindingResult` presence does NOT suppress `MethodArgumentNotValidException`. The exception is thrown regardless. Handle it with `@ExceptionHandler`.

**Trap 3 — `BindingResult` must be IMMEDIATELY adjacent**
Even one non-model-attribute parameter between `@ModelAttribute` and `BindingResult` breaks the adjacency rule. `ErrorsMethodArgumentResolver` looks at the IMMEDIATELY preceding parameter. If it's not a model attribute → `IllegalStateException`. The error occurs at runtime (first request), not at startup.

**Trap 4 — `@RequestParam` required=false with primitive type**
`@RequestParam(required=false) int id` — when `id` is absent, Spring tries to inject `null` for `int`. Unboxing `null` → `NullPointerException`. Always use boxed types (`Integer`) for optional numeric parameters, or provide a `defaultValue="0"`.

**Trap 5 — Custom resolvers added via `addArgumentResolvers()` cannot override built-ins**
Custom resolvers are appended AFTER all 28 built-in resolvers. If a built-in resolver claims the parameter first (e.g., `@RequestParam` is present), your custom resolver's `supportsParameter()` is never called. To intercept before built-ins, you must replace the entire `argumentResolvers` list on `RequestMappingHandlerAdapter`.

**Trap 6 — `@Value` on method parameter vs field**
`@Value` on a FIELD is evaluated ONCE during bean creation. `@Value` on a METHOD PARAMETER is evaluated on EVERY request via `ExpressionValueMethodArgumentResolver`. For constant config values, field injection is more efficient. SpEL expressions in method parameters can access beans and are evaluated per-request — useful but potentially expensive.

**Trap 7 — Resolver cache is per `MethodParameter` identity**
`MethodParameter` identity includes: declaring class + method + parameter index. The cache maps each unique `MethodParameter` to its resolver. Two different controller classes with the same parameter signature get separate cache entries. The cache grows with the number of distinct handler methods registered — a concern for applications with thousands of endpoints.

---

## 5️⃣ SUMMARY SHEET

```
ARGUMENT RESOLVER CHAIN — KEY RESOLVERS IN ORDER
─────────────────────────────────────────────────────
[1]  RequestParamMethodArgumentResolver  → @RequestParam (simple types)
[2]  RequestParamMapMethodArgumentResolver → @RequestParam Map
[3]  PathVariableMethodArgumentResolver  → @PathVariable (simple types)
[4]  PathVariableMapMethodArgumentResolver → @PathVariable Map
[7]  ServletModelAttributeMethodProcessor → @ModelAttribute (complex)
[8]  RequestResponseBodyMethodProcessor  → @RequestBody
[10] RequestHeaderMethodArgumentResolver → @RequestHeader
[12] ServletCookieValueMethodArgumentResolver → @CookieValue
[13] ExpressionValueMethodArgumentResolver → @Value
[14] SessionAttributeMethodArgumentResolver → @SessionAttribute
[16] ServletRequestMethodArgumentResolver → HttpServletRequest, Locale, etc.
[17] ServletResponseMethodArgumentResolver → HttpServletResponse, Writer
[19] RedirectAttributesMethodArgumentResolver → RedirectAttributes
[20] ModelMethodProcessor → Model, ModelMap
[22] ErrorsMethodArgumentResolver → BindingResult (must be adjacent!)
[27] RequestParamMethodArgumentResolver (useDefaultResolution=true)
     → simple types WITHOUT @RequestParam (catch-all)
[28] ServletModelAttributeMethodProcessor (supports=true)
     → complex types WITHOUT @ModelAttribute (catch-all)

CACHING: MethodParameter → winning resolver (ConcurrentHashMap)
First request: O(n) chain walk. Subsequent: O(1) cache hit.

@REQUESTPARAM RULES
─────────────────────────────────────────────────────
required=true (default): MissingServletRequestParameterException if absent
required=false: null if absent
defaultValue: makes required irrelevant
Optional<T>: Optional.empty() if absent (implicit required=false)
primitive + required=false → NPE on unboxing null → use boxed type
List<String>: ?tags=a&tags=b → ["a","b"]
Map<String,String>: all params as map (no @RequestParam = MODEL map)

@PATHVARIABLE RULES
─────────────────────────────────────────────────────
Type conversion via ConversionService
Failure → MethodArgumentTypeMismatchException → 400
Map: all URI variables as map
{id:\\d+}: regex constraint → no match = 404 (not 400)

@REQUESTBODY RULES
─────────────────────────────────────────────────────
Reads getInputStream() — ONE TIME ONLY
Content-Type → selects HttpMessageConverter
@Valid: MethodArgumentNotValidException on failure
BindingResult after @RequestBody: does NOT suppress exception
JSON parse failure: HttpMessageNotReadableException → 400

@MODELATTRIBUTE RULES
─────────────────────────────────────────────────────
Priority: model attribute → session (@SessionAttributes) → new instance
Binds request params via WebDataBinder
@Valid: validation run, errors in BindingResult (does NOT throw if BindingResult present)
BindingResult: MUST be immediately adjacent (next parameter)
BindingResult gap: IllegalStateException at runtime

UNANNOTATED PARAMETER FALLBACKS
─────────────────────────────────────────────────────
Simple type (String, int, etc.): @RequestParam catch-all resolver
  → reads from request.getParameter(name)
Complex type: @ModelAttribute catch-all resolver
  → creates instance, binds params
Map (no annotation): MapMethodProcessor → model map (NOT request params!)
  → use @RequestParam Map for request params

TYPE CONVERSION FAILURES
─────────────────────────────────────────────────────
@RequestParam: MethodArgumentTypeMismatchException → 400
@PathVariable: MethodArgumentTypeMismatchException → 400
@ModelAttribute field: FieldError in BindingResult (not an exception)
@RequestBody JSON: HttpMessageNotReadableException → 400

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "Argument resolver cache: first request walks chain O(n), after O(1)"
• "BindingResult must immediately follow its @ModelAttribute — any gap = IllegalStateException"
• "@RequestBody + @Valid: MethodArgumentNotValidException thrown regardless of BindingResult"
• "Unannotated Map parameter = model map (MapMethodProcessor), NOT request params"
• "Optional<String> with @RequestParam: Optional.empty() when absent — no exception"
• "primitive + required=false: NPE on unboxing null — use boxed Integer/Long"
• "Custom resolvers added via addArgumentResolvers() go AFTER all built-ins"
• "@Value on method param: evaluated per request; @Value on field: evaluated once"
• "@PathVariable regex: no match = 404 (URL-level), not 400 (binding-level)"
• "RequestParamMethodArgumentResolver registered twice: once for annotated, once for catch-all"
```

---

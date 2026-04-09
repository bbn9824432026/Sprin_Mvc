# TOPIC 3.1 — @Controller AND @RestController: DIFFERENCES AND INTERNALS

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What @Controller and @RestController Really Are

Most developers treat `@Controller` and `@RestController` as "the annotation that makes a class handle web requests." This surface-level understanding fails under examination. At the architecture level, these annotations are **stereotype markers** that trigger three completely separate mechanisms: bean registration, request mapping discovery, and response body handling. Understanding all three mechanisms and how they interact is what mastery requires.

---

### @Controller — Complete Internal Analysis

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component          // ← THIS is the only active annotation
public @interface Controller {
    @AliasFor(annotation = Component.class)
    String value() default "";
}
```

**`@Controller` is `@Component` with a stereotype name.** Technically, Spring's component scan discovers it because of `@Component`. The `@Controller` stereotype itself adds:

1. **Bean registration** — `@Component` triggers `ClassPathScanningCandidateComponentProvider`
2. **Handler detection** — `RequestMappingHandlerMapping.isHandler()` checks for `@Controller` OR `@RequestMapping`
3. **No response body handling** — return values go through the ViewResolver pipeline

```java
// RequestMappingHandlerMapping.isHandler() — the detection logic
@Override
protected boolean isHandler(Class<?> beanType) {
    return (AnnotatedElementUtils.hasAnnotation(
                beanType, Controller.class) ||
            AnnotatedElementUtils.hasAnnotation(
                beanType, RequestMapping.class));
}
```

This means: a class annotated with only `@Component` + `@RequestMapping` (no `@Controller`) IS detected as a handler. `@Controller` is NOT strictly required for handler detection — only for component scanning as a bean.

---

### @RestController — Complete Internal Analysis

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller             // ← Inherits @Component via @Controller
@ResponseBody           // ← THE key addition
public @interface RestController {
    @AliasFor(annotation = Controller.class)
    String value() default "";
}
```

**`@RestController` = `@Controller` + `@ResponseBody` at the class level.**

The `@ResponseBody` here is a **class-level annotation**. `RequestResponseBodyMethodProcessor` checks for `@ResponseBody` on both the method AND the class:

```java
// RequestResponseBodyMethodProcessor.supportsReturnType()
@Override
public boolean supportsReturnType(MethodParameter returnType) {
    return (AnnotatedElementUtils.hasAnnotation(
                returnType.getContainingClass(),
                ResponseBody.class) ||
            returnType.hasMethodAnnotation(ResponseBody.class));
    // TRUE if @RestController (class-level @ResponseBody)
    // TRUE if @ResponseBody on method
    // FALSE otherwise
}
```

This single `supportsReturnType()` returning `true` causes:
- Return value serialised by `HttpMessageConverter` (Jackson)
- `mavContainer.setRequestHandled(true)`
- ViewResolver chain NEVER called
- No view rendering

---

### The Annotation Metadata Propagation Chain

Understanding how `@ResponseBody` gets from class annotation to per-request processing requires understanding Spring's annotation metadata resolution:

```
@RestController on ProductController
        │
        ▼
AnnotatedElementUtils.hasAnnotation(ProductController.class, ResponseBody.class)
        │
        ├── Checks class-level annotations
        ├── Finds @RestController
        ├── @RestController is meta-annotated with @ResponseBody
        └── Returns TRUE — annotation found via meta-annotation chain

Per-request (RequestResponseBodyMethodProcessor.supportsReturnType()):
        │
        ▼
returnType.getContainingClass() = ProductController.class
AnnotatedElementUtils.hasAnnotation(ProductController.class, ResponseBody.class)
        → TRUE (via meta-annotation chain through @RestController)
        → This method's return value will be serialised via MessageConverter
        → mavContainer.setRequestHandled(true)
```

This meta-annotation resolution runs on EVERY handler method invocation. It is O(1) due to caching in `AnnotatedElementUtils`.

---

### Return Value Processing — The Split

The entire difference between `@Controller` and `@RestController` at runtime comes down to which `HandlerMethodReturnValueHandler` processes the return value:

```
@Controller method returning String "product/list":
        │
        ▼
HandlerMethodReturnValueHandlerComposite.handleReturnValue()
        │
        ├── RequestResponseBodyMethodProcessor.supportsReturnType()
        │     → @ResponseBody on class/method? NO (@Controller only)
        │     → returns false — skip
        │
        ├── ViewNameMethodReturnValueHandler.supportsReturnType()
        │     → Is return type CharSequence? YES (String is CharSequence)
        │     → returns true — SELECTED
        │
        └── ViewNameMethodReturnValueHandler.handleReturnValue()
              → mavContainer.setViewName("product/list")
              → mavContainer.requestHandled = false
              → ViewResolver chain will be called

─────────────────────────────────────────────────────────────────────

@RestController method returning Product object:
        │
        ▼
HandlerMethodReturnValueHandlerComposite.handleReturnValue()
        │
        ├── RequestResponseBodyMethodProcessor.supportsReturnType()
        │     → @ResponseBody on class? YES (@RestController has @ResponseBody)
        │     → returns true — SELECTED
        │
        └── RequestResponseBodyMethodProcessor.handleReturnValue()
              → Finds HttpMessageConverter for (Product, application/json)
              → Jackson serialises Product to JSON
              → Writes JSON to response
              → response.getWriter().flush()
              → mavContainer.setRequestHandled(true)
              → ViewResolver chain NEVER called
```

---

### What Happens to Model in @RestController

A critical and frequently tested point:

```java
@RestController
public class ProductController {

    @ModelAttribute("globalData")
    public Map<String, String> addGlobalData() {
        // This STILL executes for every request
        // Even though mavContainer.requestHandled=true
        // Even though model is never used for view rendering
        return Map.of("env", "production");
    }

    @GetMapping("/products")
    public List<Product> getAll() {
        // getAll() executes AFTER addGlobalData()
        // model has "globalData" in it
        // But mavContainer.requestHandled=true
        // model is NEVER rendered — completely discarded
        return productService.findAll();
    }
}
```

`@ModelAttribute` methods in `@RestController` classes are wasted work. They execute, populate the model, and then the model is thrown away. This is a performance trap in practice — developers put expensive `@ModelAttribute` methods on `@RestController` classes and wonder why performance is poor.

---

### Content Type Negotiation — How It Affects Both

For `@Controller` with `@ResponseBody` (or `@RestController`), content type negotiation determines which `HttpMessageConverter` is selected:

```
Request: GET /products
         Accept: application/json

        ↓
RequestResponseBodyMethodProcessor.writeWithMessageConverters()
        ↓
ContentNegotiationManager determines acceptable media types:
  From Accept header: [application/json, */*]
        ↓
Iterate HttpMessageConverter list:
  StringHttpMessageConverter: canWrite(List<Product>, application/json)? NO
  ByteArrayHttpMessageConverter: NO
  ResourceHttpMessageConverter: NO
  MappingJackson2HttpMessageConverter:
    canWrite(List<Product>, application/json)? YES ← SELECTED
        ↓
MappingJackson2HttpMessageConverter.write():
  → Jackson ObjectMapper.writeValue(response.getOutputStream(), productList)
  → response.setContentType("application/json")
  → response flush

Request: GET /products
         Accept: application/xml

        ↓
MappingJackson2XmlHttpMessageConverter:
  canWrite(List<Product>, application/xml)? YES (if Jackson XML on classpath)
  OR
Jaxb2RootElementHttpMessageConverter:
  canWrite(List<Product>, application/xml)? YES (if @XmlRootElement on Product)
```

---

### @ResponseStatus on Class vs Method

```java
@RestController
@RequestMapping("/api/v1/products")
@ResponseStatus(HttpStatus.OK)  // Class-level: all methods default to 200
public class ProductController {

    // Returns 200 (from class-level @ResponseStatus + default)
    @GetMapping
    public List<Product> getAll() {
        return productService.findAll();
    }

    // Returns 201 — method-level OVERRIDES class-level
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Product create(@RequestBody Product product) {
        return productService.save(product);
    }

    // Returns 204 — no body
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        productService.delete(id);
        // void return + NO_CONTENT + @RestController
        // → requestHandled=true (NO_CONTENT means no body)
        // → no view rendering
    }
}
```

`@ResponseStatus` is processed by `ServletInvocableHandlerMethod.setResponseStatus()`:

```java
private void setResponseStatus(ServletWebRequest webRequest)
        throws IOException {

    HttpStatus status = getResponseStatus();
    if (status == null) {
        return;
    }

    HttpServletResponse response =
        webRequest.getResponse();
    if (this.responseStatusReason != null) {
        response.sendError(status.value(),
            this.responseStatusReason);
    } else {
        response.setStatus(status.value());
    }

    // Store for ModelAndView
    webRequest.getRequest().setAttribute(
        View.RESPONSE_STATUS_ATTRIBUTE, status);
}
```

---

### Proxy Considerations — CGLIB vs Interface

`@Controller` and `@RestController` classes are typically CGLIB-proxied when `@Transactional`, `@Cacheable`, or other AOP advice applies:

```
ProductController (original class)
        ↓
ProductController$$SpringCGLIB$$0 (CGLIB subclass proxy)
        ↓
Spring stores proxy in ApplicationContext
        ↓
RequestMappingHandlerMapping.isHandler():
  beanType = ProductController$$SpringCGLIB$$0
  ClassUtils.getUserClass(beanType) = ProductController (unwrapped)
  AnnotatedElementUtils.hasAnnotation(ProductController, Controller) = true
  ↓
detectHandlerMethods():
  userType = ClassUtils.getUserClass(beanType) = ProductController
  Reflects over ProductController methods (not proxy)
  Registers HandlerMethod with proxy as bean but ProductController methods
        ↓
Per request:
  HandlerMethod.getBean() = proxy ($$SpringCGLIB$$0)
  method.invoke(proxy, args)
  → AOP advice fires (transaction starts, cache checked)
  → Original method executes
```

**Critical:** `@RequestMapping` methods on controllers are discovered from the **original class** (via `ClassUtils.getUserClass()`), but invoked on the **proxy**. This ensures AOP interceptors run.

---

### @Controller with @ResponseBody — The Hybrid

It is valid and common to have `@Controller` with some methods using `@ResponseBody`:

```java
@Controller
@RequestMapping("/products")
public class ProductController {

    // View-based — returns JSP view
    @GetMapping
    public String list(Model model) {
        model.addAttribute("products", productService.findAll());
        return "product/list";
        // → ViewResolver chain → InternalResourceViewResolver
        // → /WEB-INF/views/product/list.jsp
    }

    // REST-like — returns JSON for AJAX
    @GetMapping("/search")
    @ResponseBody  // Method-level — this method only
    public List<Product> search(
            @RequestParam String q) {
        return productService.search(q);
        // → RequestResponseBodyMethodProcessor
        // → Jackson serialises to JSON
    }

    // View with status code
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public String create(@Valid @ModelAttribute Product product,
                          BindingResult result) {
        if (result.hasErrors()) return "product/form";
        productService.save(product);
        return "redirect:/products";
    }
}
```

This hybrid approach is valid. `@ResponseBody` at method level works in `@Controller` classes. The return value handler selection is per-method — `supportsReturnType()` is called for each method's return value separately.

---

### @ResponseBody and Void Return Type

```java
@RestController
public class FileController {

    // void + @RestController = no body, requestHandled=true
    // But HOW does requestHandled get set?
    @GetMapping("/download")
    public void download(HttpServletResponse response)
            throws IOException {
        response.setContentType("application/octet-stream");
        response.setHeader("Content-Disposition",
            "attachment; filename=data.bin");

        byte[] data = dataService.getData();
        response.getOutputStream().write(data);
        // Response written directly

        // For void return in @RestController:
        // RequestResponseBodyMethodProcessor.supportsReturnType():
        //   → @ResponseBody on class? YES
        //   → supports = true
        // handleReturnValue() with null returnValue:
        //   → returnValue is null (void)
        //   → mavContainer.setRequestHandled(true) — SET HERE
        //   → No serialisation attempted
    }
}
```

**Void in `@RestController`:** `RequestResponseBodyMethodProcessor` still handles it. `handleReturnValue()` receives `null` as `returnValue`. It calls `mavContainer.setRequestHandled(true)` to prevent view rendering. It does NOT write to the response (nothing to write). The response was written directly in the method body.

---

### @Controller Without Any RequestMapping

```java
// Valid but unusual — class-level @RequestMapping
@Controller
public class AdminController {

    // No class-level @RequestMapping
    // All mappings at method level — absolute paths

    @GetMapping("/admin/users")
    public String users(Model model) { ... }

    @GetMapping("/admin/settings")
    public String settings(Model model) { ... }
}

// Also valid — no @RequestMapping on controller at all
// (registered as Spring bean but no handler methods)
@Controller
public class SupportBean {
    // No @RequestMapping methods
    // isHandler() returns true (has @Controller)
    // detectHandlerMethods() finds no methods
    // No mappings registered
    // Just a Spring bean with controller stereotype
}
```

---

### HTTP Message Converters — Default Chain

For `@RestController`, the default converters registered by `@EnableWebMvc` (in order):

```
1.  ByteArrayHttpMessageConverter
    → byte[] ↔ application/octet-stream, */*

2.  StringHttpMessageConverter
    → String ↔ text/plain, */*
    → DEFAULT charset: ISO-8859-1 (not UTF-8!)
    → Override: StringHttpMessageConverter(StandardCharsets.UTF_8)

3.  ResourceHttpMessageConverter
    → Resource ↔ */*
    → Used for file downloads

4.  ResourceRegionHttpMessageConverter
    → ResourceRegion ↔ application/octet-stream

5.  SourceHttpMessageConverter
    → Source (XML) ↔ application/xml, text/xml, application/*+xml

6.  AllEncompassingFormHttpMessageConverter
    → MultiValueMap ↔ application/x-www-form-urlencoded, multipart/form-data

7.  MappingJackson2HttpMessageConverter (if Jackson on classpath)
    → Object ↔ application/json, application/*+json
    → Uses singleton ObjectMapper
    → canWrite(): object is serialisable by Jackson

8.  MappingJackson2XmlHttpMessageConverter (if Jackson XML on classpath)
    → Object ↔ application/xml, text/xml, application/*+xml

9.  Jaxb2RootElementHttpMessageConverter (if JAXB on classpath)
    → @XmlRootElement classes ↔ application/xml, text/xml
```

**StringHttpMessageConverter charset trap:** Default is ISO-8859-1. Non-ASCII characters in String responses may be corrupted. Fix:

```java
@Override
public void extendMessageConverters(
        List<HttpMessageConverter<?>> converters) {
    converters.removeIf(c ->
        c instanceof StringHttpMessageConverter);
    converters.add(0,
        new StringHttpMessageConverter(StandardCharsets.UTF_8));
}
```

---

### @RequestBody — The Complement to @ResponseBody

```java
@RestController
@RequestMapping("/products")
public class ProductController {

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Product create(@RequestBody Product product) {
        // @RequestBody resolution:
        // RequestResponseBodyMethodProcessor.resolveArgument()
        //   → reads request.getInputStream()
        //   → Content-Type: application/json
        //   → MappingJackson2HttpMessageConverter.read(Product.class, input)
        //   → Jackson deserialises JSON → Product instance
        //   → Stream consumed — cannot read again
        return productService.save(product);
    }

    // WRONG — @RequestBody + @Valid
    @PutMapping("/{id}")
    public Product update(
            @PathVariable Long id,
            @Valid @RequestBody Product product,  // Validation applied
            BindingResult result) {
        // PROBLEM: @RequestBody + BindingResult is NOT like @ModelAttribute + BindingResult
        // If validation fails for @RequestBody:
        //   MethodArgumentNotValidException thrown BEFORE handler runs
        //   BindingResult is ignored / not populated the same way
        // CORRECT approach: use BindingResult only with @ModelAttribute
        // For @RequestBody validation: let MethodArgumentNotValidException propagate
        //   and handle with @ExceptionHandler
        if (result.hasErrors()) return null; // This line may never reach
        return productService.update(id, product);
    }
}
```

---

### Runtime Class — What the Bean Actually Is

```java
// With @Transactional — bean is a CGLIB proxy
@RestController
@Transactional
public class ProductController {
    ...
}

// In application context:
ProductController bean = context.getBean(ProductController.class);
System.out.println(bean.getClass().getName());
// Output: com.example.ProductController$$SpringCGLIB$$0

// This.getClass() inside the controller:
@GetMapping("/class")
@ResponseBody
public String className() {
    return this.getClass().getName();
    // Returns: com.example.ProductController$$SpringCGLIB$$0
    // NOT: com.example.ProductController
}
```

---

## 2️⃣ CODE EXAMPLES

### @Controller vs @RestController — Side-by-Side

```java
// VIEW-BASED CONTROLLER
@Controller
@RequestMapping("/mvc/products")
public class ProductViewController {

    @Autowired
    private ProductService productService;

    // Returns view name → ViewResolver chain
    @GetMapping
    public String list(Model model) {
        model.addAttribute("products",
            productService.findAll());
        return "product/list";
        // → /WEB-INF/views/product/list.jsp (InternalResourceViewResolver)
    }

    // Returns ModelAndView directly
    @GetMapping("/{id}")
    public ModelAndView detail(@PathVariable Long id) {
        Product product = productService.findById(id)
            .orElseThrow(() ->
                new ProductNotFoundException(id));
        ModelAndView mav = new ModelAndView("product/detail");
        mav.addObject("product", product);
        return mav;
    }

    // Form submission — redirect on success
    @PostMapping
    public String create(
            @Valid @ModelAttribute("product") Product product,
            BindingResult result,
            RedirectAttributes redirectAttrs) {
        if (result.hasErrors()) {
            return "product/form"; // Re-show form with errors
        }
        Product saved = productService.save(product);
        redirectAttrs.addFlashAttribute("success",
            "Product created: " + saved.getName());
        return "redirect:/mvc/products";
    }
}

// REST CONTROLLER
@RestController
@RequestMapping("/api/products")
public class ProductRestController {

    @Autowired
    private ProductService productService;

    // Returns JSON — no ViewResolver
    @GetMapping
    public List<Product> list() {
        return productService.findAll();
        // → MappingJackson2HttpMessageConverter
        // → Content-Type: application/json
    }

    // Returns JSON with status 200
    @GetMapping("/{id}")
    public ResponseEntity<Product> detail(
            @PathVariable Long id) {
        return productService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
        // HttpEntityMethodProcessor handles ResponseEntity
    }

    // Returns JSON with status 201
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Product create(@RequestBody Product product) {
        return productService.save(product);
    }

    // Returns 204 — no body
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        productService.delete(id);
    }

    // Returns 200 with updated resource
    @PutMapping("/{id}")
    public Product update(
            @PathVariable Long id,
            @RequestBody Product product) {
        return productService.update(id, product);
    }
}
```

---

### @ResponseBody on Individual Methods in @Controller

```java
@Controller
@RequestMapping("/products")
public class HybridController {

    // View-based: returns JSP
    @GetMapping
    public String list(Model model) {
        model.addAttribute("products",
            productService.findAll());
        return "product/list";
    }

    // AJAX endpoint: returns JSON
    @GetMapping("/{id}/json")
    @ResponseBody  // Only this method returns JSON
    public Product getJson(@PathVariable Long id) {
        return productService.findById(id)
            .orElseThrow(() ->
                new ResponseStatusException(
                    HttpStatus.NOT_FOUND));
    }

    // AJAX endpoint: returns JSON via ResponseEntity
    @PostMapping("/validate")
    @ResponseBody
    public ResponseEntity<Map<String, Object>> validate(
            @Valid @RequestBody Product product,
            Errors errors) {
        if (errors.hasErrors()) {
            Map<String, Object> errorMap =
                new LinkedHashMap<>();
            errors.getFieldErrors().forEach(e ->
                errorMap.put(e.getField(),
                    e.getDefaultMessage()));
            return ResponseEntity.badRequest().body(
                Map.of("errors", errorMap));
        }
        return ResponseEntity.ok(
            Map.of("valid", true));
    }
}
```

---

### ResponseBodyAdvice — Wrapping All Responses

```java
// Wraps ALL @ResponseBody and @RestController responses
// in a standard API envelope
@ControllerAdvice
public class ApiResponseWrapper
    implements ResponseBodyAdvice<Object> {

    @Override
    public boolean supports(
            MethodParameter returnType,
            Class<? extends HttpMessageConverter<?>> converterType) {

        // Apply only to RestController methods
        // and not to error responses
        return returnType.getContainingClass()
            .isAnnotationPresent(RestController.class) &&
            !returnType.getContainingClass()
                .isAnnotationPresent(NoWrap.class);
    }

    @Override
    public Object beforeBodyWrite(
            @Nullable Object body,
            MethodParameter returnType,
            MediaType selectedContentType,
            Class<? extends HttpMessageConverter<?>> converterType,
            ServerHttpRequest request,
            ServerHttpResponse response) {

        // Don't wrap if already wrapped
        if (body instanceof ApiResponse) {
            return body;
        }

        // Don't wrap error responses
        if (response.getHeaders()
                .getContentType() == MediaType.APPLICATION_PROBLEM_JSON) {
            return body;
        }

        return ApiResponse.success(body);
        // {"status":"success","data":<original body>,"timestamp":"..."}
    }
}

// Result: every @RestController method's JSON is automatically wrapped
// GET /api/products →
// {"status":"success","data":[{"id":1,"name":"Widget"}],"timestamp":"2024-..."}
```

---

### Custom HttpMessageConverter

```java
// Custom CSV converter for List<Product>
public class ProductCsvHttpMessageConverter
    extends AbstractHttpMessageConverter<List<Product>> {

    public ProductCsvHttpMessageConverter() {
        // Register supported media type
        super(new MediaType("text", "csv"));
    }

    @Override
    protected boolean supports(Class<?> clazz) {
        // Only handles List — check is approximate
        // Full generic check done in canRead/canWrite
        return List.class.isAssignableFrom(clazz);
    }

    @Override
    protected List<Product> readInternal(
            Class<? extends List<Product>> clazz,
            HttpInputMessage inputMessage) throws IOException {
        // Parse CSV → List<Product>
        List<Product> products = new ArrayList<>();
        BufferedReader reader = new BufferedReader(
            new InputStreamReader(
                inputMessage.getBody(),
                StandardCharsets.UTF_8));
        String line;
        while ((line = reader.readLine()) != null) {
            String[] parts = line.split(",");
            products.add(new Product(
                Long.parseLong(parts[0]),
                parts[1],
                Double.parseDouble(parts[2])));
        }
        return products;
    }

    @Override
    protected void writeInternal(
            List<Product> products,
            HttpOutputMessage outputMessage) throws IOException {
        // Serialise List<Product> → CSV
        PrintWriter writer = new PrintWriter(
            new OutputStreamWriter(
                outputMessage.getBody(),
                StandardCharsets.UTF_8));
        for (Product p : products) {
            writer.println(p.getId() + "," +
                p.getName() + "," + p.getPrice());
        }
        writer.flush();
    }
}

// Register via extendMessageConverters
@Override
public void extendMessageConverters(
        List<HttpMessageConverter<?>> converters) {
    converters.add(new ProductCsvHttpMessageConverter());
}

// Usage — content negotiation selects CSV when Accept: text/csv
@RestController
@RequestMapping("/api/products")
public class ProductController {

    @GetMapping(produces = {"application/json", "text/csv"})
    public List<Product> getAll() {
        return productService.findAll();
        // Accept: application/json → Jackson → JSON
        // Accept: text/csv → ProductCsvHttpMessageConverter → CSV
    }
}
```

---

### Edge Case — @RequestBody with @Valid and Exception Handling

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Order create(@Valid @RequestBody OrderRequest request) {
        // If validation fails:
        // MethodArgumentNotValidException thrown
        // BEFORE this method body executes
        // BindingResult NOT available here (unlike @ModelAttribute)
        return orderService.create(request);
    }
}

// Handle validation failures globally
@RestControllerAdvice
public class ValidationExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, Object> handleValidation(
            MethodArgumentNotValidException ex) {
        Map<String, Object> errors = new LinkedHashMap<>();
        errors.put("status", 400);
        errors.put("message", "Validation failed");

        Map<String, String> fieldErrors = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(e ->
            fieldErrors.put(e.getField(), e.getDefaultMessage()));
        errors.put("errors", fieldErrors);

        return errors;
    }

    @ExceptionHandler(HttpMessageNotReadableException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, String> handleBadJson(
            HttpMessageNotReadableException ex) {
        return Map.of("error", "Malformed JSON request body");
    }
}
```

---

### Edge Case — Detecting @RestController at Runtime

```java
@Component
public class ControllerInspector
    implements ApplicationListener<ContextRefreshedEvent> {

    @Autowired
    private ApplicationContext context;

    @Override
    public void onApplicationEvent(
            ContextRefreshedEvent event) {

        // Find all @RestController beans
        Map<String, Object> restControllers =
            context.getBeansWithAnnotation(RestController.class);

        // Find all @Controller beans (includes @RestController)
        Map<String, Object> controllers =
            context.getBeansWithAnnotation(Controller.class);

        // Pure @Controller (not @RestController)
        Set<String> pureControllers = new HashSet<>(
            controllers.keySet());
        pureControllers.removeAll(restControllers.keySet());

        System.out.println("@RestControllers: " +
            restControllers.keySet());
        System.out.println("Pure @Controllers: " +
            pureControllers);

        // For each @RestController, show its request mappings
        restControllers.forEach((name, bean) -> {
            Class<?> type = AopUtils.getTargetClass(bean);
            RequestMapping classMapping =
                AnnotationUtils.findAnnotation(type,
                    RequestMapping.class);
            System.out.println(name + " → " +
                (classMapping != null ?
                    Arrays.toString(classMapping.value()) :
                    "no class-level mapping"));
        });
    }
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
What is the MINIMUM annotation set required for a class to be detected as a handler by `RequestMappingHandlerMapping` AND registered as a Spring bean via component scanning?

A) `@Controller` only
B) `@Component` only
C) `@Component` + `@RequestMapping` (class or method level)
D) `@Controller` + `@RequestMapping`

**Answer: A**
`@Controller` includes `@Component` (for bean registration via component scan) AND is detected by `RequestMappingHandlerMapping.isHandler()` (checks for `@Controller` annotation). A class with only `@Component` IS a Spring bean but `isHandler()` returns false unless it also has `@RequestMapping`. So `@Controller` alone satisfies both requirements. `@Component` + `@RequestMapping` also works but `@Controller` alone is sufficient and cleaner.

---

**Q2 — Select All That Apply**
Which of the following correctly describe the runtime difference between `@Controller` and `@RestController`?

A) `@RestController` registers both a controller AND a `HandlerAdapter`
B) `@RestController` causes `RequestResponseBodyMethodProcessor` to handle return values
C) `@Controller` methods returning `String` go through `ViewResolver`
D) `@RestController` methods returning `String` go through `ViewResolver`
E) Both are CGLIB-proxied under the same conditions (e.g., `@Transactional`)

**Answer: B, C, E**
A is wrong — no additional `HandlerAdapter` is registered. D is wrong — `@RestController` has class-level `@ResponseBody` so `RequestResponseBodyMethodProcessor` handles String returns too (writes to response as text, NOT as view name). B is correct — class-level `@ResponseBody` makes RRPBMP handle all returns. C is correct — `@Controller` String returns go through `ViewNameMethodReturnValueHandler` → ViewResolver. E is correct — both are CGLIB-proxied under the same AOP conditions.

---

**Q3 — Code Prediction**
```java
@RestController
@RequestMapping("/api")
public class TestController {

    @ModelAttribute("meta")
    public Map<String, String> addMeta() {
        System.out.println("addMeta() called");
        return Map.of("version", "v1");
    }

    @GetMapping("/data")
    public Map<String, String> getData() {
        System.out.println("getData() called");
        return Map.of("result", "ok");
    }
}
```
`GET /api/data` — what is printed and what is returned?

A) Only `"getData() called"` printed; `{"result":"ok"}` returned
B) Both `"addMeta() called"` and `"getData() called"` printed; `{"result":"ok"}` returned
C) Both printed; `{"meta":{"version":"v1"},"result":"ok"}` returned (merged JSON)
D) `"addMeta() called"` printed; error thrown

**Answer: B**
`@ModelAttribute` methods ALWAYS execute before the handler — even in `@RestController`. Both print. But the `meta` model attribute is NEVER included in the JSON response. `getData()` returns its own `Map` directly as `@ResponseBody`. The model attribute is discarded. JSON response: `{"result":"ok"}` only. C is wrong because `@ResponseBody` serialises the RETURN VALUE of the handler method, not the entire model.

---

**Q4 — MCQ**
A `@RestController` method returns `void`. What does `RequestResponseBodyMethodProcessor.handleReturnValue()` do?

A) Throws `IllegalStateException` — void is not a valid return type for `@RestController`
B) Sets `mavContainer.setRequestHandled(true)` — no body written, view rendering skipped
C) Serialises `null` as the JSON value `null`
D) Falls through to `ViewNameMethodReturnValueHandler` — view derived from URL

**Answer: B**
`RequestResponseBodyMethodProcessor.supportsReturnType()` returns `true` for `@RestController` even for `void` return type. `handleReturnValue()` receives `null` as `returnValue`. When `returnValue == null`, it calls `mavContainer.setRequestHandled(true)` without writing anything to the response. The method was expected to write to the response directly (or the response body is empty). No view rendering attempted.

---

**Q5 — True/False**
In a `@RestController`, a method returning `String` sends the string value as plain text in the response body (Content-Type: text/plain), not as a view name.

**Answer: True**
`@RestController` has class-level `@ResponseBody`. `RequestResponseBodyMethodProcessor.supportsReturnType()` returns `true`. `StringHttpMessageConverter` handles `String` return types with `text/plain` (or `text/*` per Accept header). The string is written directly to the response body. `ViewNameMethodReturnValueHandler` is NEVER consulted because `RequestResponseBodyMethodProcessor` claims the return value first.

---

**Q6 — MCQ**
`StringHttpMessageConverter` has a default charset of ISO-8859-1. A `@RestController` method returns a `String` containing the character `"é"`. What happens?

A) `é` is correctly encoded as UTF-8 in the response
B) `é` may be corrupted/replaced — ISO-8859-1 cannot represent all Unicode characters correctly in all scenarios
C) Spring automatically detects the charset from the string content
D) `HttpMediaTypeNotAcceptableException` thrown — charset conflict

**Answer: B**
The default `StringHttpMessageConverter` uses ISO-8859-1. `é` is representable in ISO-8859-1 (code point 0xE9) but many other characters are not. More importantly, if the client expects UTF-8 and the response is ISO-8859-1, multi-byte characters will be corrupted. The fix: register `StringHttpMessageConverter(StandardCharsets.UTF_8)` via `extendMessageConverters()`.

---

**Q7 — Select All That Apply**
Which of the following are processed by `RequestResponseBodyMethodProcessor`?

A) `@RequestBody` parameter resolution
B) `@ResponseBody` return value handling
C) `@ModelAttribute` return value handling
D) `ResponseEntity<T>` return value handling
E) `HttpEntity<T>` parameter resolution

**Answer: A, B, E**
`RequestResponseBodyMethodProcessor` handles `@RequestBody` (argument resolution) AND `@ResponseBody` (return value). It implements BOTH `HandlerMethodArgumentResolver` AND `HandlerMethodReturnValueHandler`. C is handled by `ServletModelAttributeMethodProcessor`. D is handled by `HttpEntityMethodProcessor`. E (`HttpEntity<T>` as parameter) is also handled by `HttpEntityMethodProcessor`, not `RRBMP`.

---

**Q8 — Scenario**
A `@Controller` class has method A returning `String "product/list"` and method B annotated with `@ResponseBody` returning `Product`. For method B, `@ModelAttribute` methods in the controller still execute. The developer assumes they don't because B returns JSON. What actually happens and why?

A) Developer is right — `@ModelAttribute` methods are skipped for `@ResponseBody` methods
B) `@ModelAttribute` methods execute for ALL methods regardless of `@ResponseBody` — model is built and then discarded for `@ResponseBody` methods
C) `@ModelAttribute` methods execute but their results are added to the JSON response
D) `@ModelAttribute` methods only execute if they share the same `@RequestMapping` path

**Answer: B**
`ModelFactory.initModel()` runs ALL applicable `@ModelAttribute` methods BEFORE the handler method — regardless of whether the handler is `@ResponseBody`. This is by design — the model may be used by error views or other processing. For `@ResponseBody` methods, the model is populated but then `mavContainer.setRequestHandled(true)` causes it to be discarded. The work was done for nothing. This is the `@ModelAttribute` performance trap for mixed controllers.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — `@Controller` vs `@RestController` is NOT about registration**
Both are registered as Spring beans via component scanning (`@Component` inheritance). The difference is NOT in how they're registered — it's in how their return values are processed. `@RestController` adds class-level `@ResponseBody`. That's the entire difference. Questions that frame the distinction as "registration mechanism" are wrong.

**Trap 2 — `@RestController` String return = text body, NOT view name**
The most common misconception. `@Controller` returning `String` → view name. `@RestController` returning `String` → response body text (Content-Type: text/plain or as negotiated). This trips developers who put a `@RestController` method returning a redirect view name expecting a redirect — they get literal string `"redirect:/success"` as text in the response body.

**Trap 3 — `@ModelAttribute` methods execute even in `@RestController`**
Critical performance trap. Many production `@RestController` classes have `@ModelAttribute` methods that make DB calls. Those calls happen on EVERY request regardless of `@ResponseBody`. The result is discarded. The fix: move expensive `@ModelAttribute` methods to `@ControllerAdvice` scoped only to view-based controllers, or remove them from `@RestController` classes.

**Trap 4 — `void` return in `@RestController` is not an error**
`void` is a valid return type in `@RestController`. `RequestResponseBodyMethodProcessor` handles it by setting `requestHandled=true` with null body. No error. The method is expected to write to the response directly OR return an empty body. Exam questions sometimes present `void` `@RestController` methods and ask "what HTTP status is returned" — answer: 200 OK (default), unless `@ResponseStatus` specifies otherwise.

**Trap 5 — `@RequestBody` streams can only be read ONCE**
`HttpServletRequest.getInputStream()` is a stream — readable once. `RequestResponseBodyMethodProcessor.resolveArgument()` reads it. After reading, it's consumed. If any code (Filter, interceptor, another resolver) tries to read the stream again, it gets empty bytes. For logging request bodies, use `ContentCachingRequestWrapper` which buffers and allows re-reading.

**Trap 6 — StringHttpMessageConverter charset is ISO-8859-1**
Default charset is ISO-8859-1, not UTF-8. Non-ASCII characters in String responses sent via `StringHttpMessageConverter` may be corrupted if client expects UTF-8. This is a production bug that only manifests with non-ASCII content. Spring Boot auto-configures UTF-8, but plain Spring MVC requires explicit configuration.

**Trap 7 — CGLIB proxy class name appears in `this.getClass()`**
Inside a proxied `@Controller` method, `this.getClass()` returns `ProductController$$SpringCGLIB$$0`. This breaks code that uses `this.getClass().getSimpleName()` for logging or reflection. Use `AopUtils.getTargetClass(this)` to get the original class, or `ClassUtils.getUserClass(this)` as used by Spring internally.

---

## 5️⃣ SUMMARY SHEET

```
@CONTROLLER vs @RESTCONTROLLER — COMPLETE COMPARISON
─────────────────────────────────────────────────────
Feature              @Controller              @RestController
─────────────────────────────────────────────────────
Meta-annotation      @Component               @Controller + @ResponseBody
Component scan       ✓ (via @Component)       ✓ (via @Controller → @Component)
isHandler() detected ✓ (has @Controller)      ✓ (has @Controller via meta)
ResponseBody         NO (default)             YES (class-level @ResponseBody)
String return        View name → ViewResolver  Text body → StringHttpMsgConverter
Object return        Model attribute (+ view)  JSON/XML → Jackson
ViewResolver called  YES (for view returns)   NO (requestHandled=true)
@ModelAttribute runs YES                       YES (but model discarded!)
CGLIB proxied        Same conditions as any bean (if AOP applies)

RETURN VALUE HANDLER SELECTION
─────────────────────────────────────────────────────
@Controller String:
  RequestResponseBodyMethodProcessor.supports() → false (no @ResponseBody)
  ViewNameMethodReturnValueHandler.supports()   → true  ← SELECTED
  → mavContainer.setViewName("product/list")
  → ViewResolver chain called

@RestController String:
  RequestResponseBodyMethodProcessor.supports() → true  ← SELECTED
  (class-level @ResponseBody via @RestController)
  → StringHttpMessageConverter writes to response
  → mavContainer.setRequestHandled(true)

@Controller + @ResponseBody method:
  Same as @RestController for THAT METHOD ONLY

REQUESTRESPONSEBODYMETHODPROCESSOR SELECTION CONDITION
─────────────────────────────────────────────────────
supportsReturnType():
  AnnotatedElementUtils.hasAnnotation(containingClass, ResponseBody) = true
  OR returnType.hasMethodAnnotation(ResponseBody) = true
→ Both class-level AND method-level @ResponseBody detected

VOID RETURN IN @RESTCONTROLLER
─────────────────────────────────────────────────────
→ RRPBMP still selected (class has @ResponseBody)
→ handleReturnValue(null, ...) called
→ mavContainer.setRequestHandled(true) — NO body written
→ HTTP 200 (default) or @ResponseStatus value
→ No error thrown

@MODELATTRIBUTE IN @RESTCONTROLLER
─────────────────────────────────────────────────────
ALWAYS executes → Model populated → Model DISCARDED
Performance trap: DB calls in @ModelAttribute wasted for REST endpoints

HTTP MESSAGE CONVERTER DEFAULT CHAIN
─────────────────────────────────────────────────────
ByteArrayHttpMessageConverter     → byte[]
StringHttpMessageConverter        → String (charset: ISO-8859-1 default!)
ResourceHttpMessageConverter      → Resource
SourceHttpMessageConverter        → XML Source
AllEncompassingFormHttpMessageConverter → form data
MappingJackson2HttpMessageConverter → Object (JSON)
MappingJackson2XmlHttpMessageConverter → Object (XML)
Jaxb2RootElementHttpMessageConverter → @XmlRootElement (XML)

StringHttpMessageConverter charset trap:
  Default = ISO-8859-1 (NOT UTF-8)
  Fix: extendMessageConverters() with StringHttpMessageConverter(UTF_8)

@REQUESTBODY STREAM CONSUMPTION
─────────────────────────────────────────────────────
getInputStream() readable ONCE
After RRPBMP reads it → consumed → cannot re-read
For logging/filtering: use ContentCachingRequestWrapper

PROXY BEHAVIOUR
─────────────────────────────────────────────────────
@Transactional on @Controller/@RestController → CGLIB proxy
this.getClass() → returns proxy class ($$SpringCGLIB$$0)
ClassUtils.getUserClass(this) → returns original class
AopUtils.getTargetClass(this) → returns original class
Methods discovered on original class, invoked on proxy

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "@RestController = @Controller + @ResponseBody at class level"
• "@ResponseBody is the only functional difference between the two"
• "@Controller returning String → view name; @RestController → text body"
• "@ModelAttribute methods run in @RestController — model silently discarded"
• "void return in @RestController is valid — requestHandled=true, no body"
• "StringHttpMessageConverter default charset is ISO-8859-1, not UTF-8"
• "RequestResponseBodyMethodProcessor handles both @RequestBody and @ResponseBody"
• "@RequestBody stream is one-time readable — ContentCachingRequestWrapper for re-read"
• "CGLIB proxy: this.getClass() shows proxy — use AopUtils.getTargetClass() for original"
• "isHandler() checks for @Controller OR @RequestMapping — @Controller alone sufficient"
```

---

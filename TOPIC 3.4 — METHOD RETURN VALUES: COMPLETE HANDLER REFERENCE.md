# TOPIC 3.4 — METHOD RETURN VALUES: COMPLETE HANDLER REFERENCE

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What Return Value Handling Really Is

When a controller method executes and returns a value, Spring MVC must decide what to do with it. That value could be a view name, a domain object, a `ResponseEntity`, a reactive type, a stream, or nothing at all. Each type demands completely different processing. The **`HandlerMethodReturnValueHandler`** chain is what makes this work — a symmetrical counterpart to argument resolution.

Understanding this system at depth means knowing: which handler processes which return type, what `mavContainer.requestHandled` means for each, how content negotiation integrates, how `ResponseEntity` differs from `@ResponseBody`, and what the exact processing sequence is inside `RequestResponseBodyMethodProcessor`.

---

### The Return Value Handler Interface

```java
public interface HandlerMethodReturnValueHandler {

    // Does this handler process this return type?
    boolean supportsReturnType(MethodParameter returnType);

    // Process the return value
    void handleReturnValue(
        @Nullable Object returnValue,
        MethodParameter returnType,
        ModelAndViewContainer mavContainer,
        NativeWebRequest webRequest
    ) throws Exception;
}
```

**Key contract:** If a handler sets `mavContainer.setRequestHandled(true)`, DispatcherServlet skips ViewResolver and view rendering entirely. If it sets a view name or View object in `mavContainer`, rendering proceeds. This single boolean is the entire REST vs view-based split.

---

### HandlerMethodReturnValueHandlerComposite — The Facade

```java
// Same caching pattern as argument resolvers
public class HandlerMethodReturnValueHandlerComposite
    implements HandlerMethodReturnValueHandler {

    private final List<HandlerMethodReturnValueHandler> returnValueHandlers =
        new ArrayList<>();

    // Cache: MethodParameter (return type) → winning handler
    private final Map<MethodParameter, HandlerMethodReturnValueHandler>
        returnValueHandlerCache = new ConcurrentHashMap<>(64);

    @Override
    public void handleReturnValue(
            @Nullable Object returnValue,
            MethodParameter returnType,
            ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest) throws Exception {

        HandlerMethodReturnValueHandler handler =
            selectHandler(returnValue, returnType);

        if (handler == null) {
            throw new IllegalArgumentException(
                "Unknown return value type: " +
                returnType.getParameterType().getName());
        }

        handler.handleReturnValue(
            returnValue, returnType, mavContainer, webRequest);
    }

    @Nullable
    private HandlerMethodReturnValueHandler selectHandler(
            @Nullable Object value,
            MethodParameter returnType) {

        // Special case: async return types
        boolean isAsyncValue = isAsyncReturnValue(value, returnType);

        for (HandlerMethodReturnValueHandler handler :
                this.returnValueHandlers) {
            if (isAsyncValue &&
                !(handler instanceof AsyncHandlerMethodReturnValueHandler)) {
                continue;
            }
            if (handler.supportsReturnType(returnType)) {
                return handler;
            }
        }
        return null;
    }
}
```

---

### The Complete Return Value Handler Chain — All Handlers in Order

```
[1]  ModelAndViewMethodReturnValueHandler
     supportsReturnType: ModelAndView
     handleReturnValue:
       → mavContainer.setView(mv.getView() or mv.getViewName())
       → mavContainer.addAllAttributes(mv.getModel())
       → mavContainer.setStatus(mv.getStatus())
       → requestHandled = false (view rendering proceeds)

[2]  ModelMethodProcessor
     supportsReturnType: Model
     handleReturnValue:
       → mavContainer.addAllAttributes(model)
       → requestHandled = false

[3]  ViewMethodReturnValueHandler
     supportsReturnType: View (interface implementations)
     handleReturnValue:
       → mavContainer.setView(view)
       → requestHandled = false

[4]  ResponseBodyEmitterReturnValueHandler
     supportsReturnType:
       → ResponseBodyEmitter (and subclasses)
       → SseEmitter
       → StreamingResponseBody
       → Also: Flux<T> when acceptedTypes contain text/event-stream (SSE)
     handleReturnValue:
       → Starts async processing
       → Writes emitted objects via MessageConverter
       → Completes when emitter completes

[5]  StreamingResponseBodyReturnValueHandler
     supportsReturnType: StreamingResponseBody
     handleReturnValue:
       → Starts async processing
       → Calls StreamingResponseBody.writeTo(OutputStream) on async thread
       → requestHandled = true (response written directly)

[6]  HttpEntityMethodProcessor
     supportsReturnType:
       → HttpEntity<T> (and subclass ResponseEntity<T>)
     handleReturnValue:
       → Extracts HttpStatus from ResponseEntity
       → Sets status on response
       → Extracts HttpHeaders, sets all headers on response
       → Writes body via HttpMessageConverter chain
         (same content negotiation as @ResponseBody)
       → requestHandled = true

[7]  HttpHeadersReturnValueHandler
     supportsReturnType: HttpHeaders
     handleReturnValue:
       → Sets all headers on response
       → No body written
       → requestHandled = true

[8]  CallableMethodReturnValueHandler
     supportsReturnType: Callable<T>
     handleReturnValue:
       → asyncManager.startCallableProcessing(callable)
       → Submits to TaskExecutor thread pool
       → Current thread released
       → Result dispatch triggers new request
       → requestHandled = false (on main thread — async handles it)

[9]  DeferredResultMethodReturnValueHandler
     supportsReturnType:
       → DeferredResult<T>
       → ListenableFuture<T> (deprecated)
       → CompletableFuture<T>
     handleReturnValue:
       → asyncManager.startDeferredResultProcessing(deferredResult)
       → No thread pool — result set externally
       → requestHandled = false

[10] AsyncTaskMethodReturnValueHandler
     supportsReturnType: WebAsyncTask<T>
     handleReturnValue:
       → asyncManager.startCallableProcessing(webAsyncTask.getCallable())
       → Supports timeout callback and completion callback

[11] ReactiveTypeReturnValueHandler
     supportsReturnType: Mono<T>, Flux<T> (Project Reactor)
     handleReturnValue:
       → Subscribes to reactive type
       → Adapts to DeferredResult or ResponseBodyEmitter
       → For Flux<T> with text/event-stream → SseEmitter path
       → For Mono<T> → DeferredResult path

[12] RequestResponseBodyMethodProcessor (as return value handler)
     supportsReturnType:
       → @ResponseBody on method
       → @ResponseBody on containing class (@RestController)
     handleReturnValue:
       → Content negotiation → selects HttpMessageConverter
       → Writes to response via converter
       → requestHandled = true

[13] ViewNameMethodReturnValueHandler
     supportsReturnType: CharSequence (includes String)
     handleReturnValue:
       → mavContainer.setViewName(viewName)
       → If starts with "redirect:" → RedirectView created
       → requestHandled = false (ViewResolver chain called)

[14] MapMethodProcessor
     supportsReturnType: Map (not annotated)
     handleReturnValue:
       → mavContainer.addAllAttributes(map)
       → requestHandled = false

[15] ViewMethodReturnValueHandler (void/null)
     supportsReturnType: void
     handleReturnValue (void):
       → If @ResponseBody (class-level):
           mavContainer.setRequestHandled(true)
       → If response already committed:
           mavContainer.setRequestHandled(true)
       → Otherwise:
           requestHandled = false (default view name derived from URL)

[Custom handlers added via addReturnValueHandlers()]
     → Added AFTER all defaults
     → Cannot replace built-in handlers
```

---

### ModelAndView — The Classic Return Type

```java
// ModelAndView: explicit control over both model and view
@GetMapping("/products")
public ModelAndView list() {
    ModelAndView mav = new ModelAndView();

    // Set view — three ways:
    mav.setViewName("product/list");     // Logical view name
    // OR
    mav.setView(new InternalResourceView("/WEB-INF/views/product/list.jsp"));
    // OR
    mav.setViewName("redirect:/products");

    // Set model attributes
    mav.addObject("products", productService.findAll());
    mav.addObject("count", productService.count());

    // Set HTTP status (Spring 5+)
    mav.setStatus(HttpStatus.OK);

    return mav;
}

// ModelAndViewMethodReturnValueHandler.handleReturnValue():
//   mavContainer.setView(mav.getViewName()) or mav.getView()
//   mavContainer.addAllAttributes(mav.getModelMap())
//   mavContainer.setStatus(mav.getStatus())
//   requestHandled = false → ViewResolver chain runs
```

---

### String Return — The Most Common View-Based Return

```java
// In @Controller (NOT @RestController):
// String → ViewNameMethodReturnValueHandler → view name

@GetMapping("/products")
public String list(Model model) {
    model.addAttribute("products", productService.findAll());

    // These are all interpreted as view names:
    return "product/list";         // → InternalResourceView or Thymeleaf
    return "redirect:/products";   // → RedirectView (HTTP 302)
    return "forward:/products";    // → forward to /products (same request)
}

// In @RestController:
// String → RequestResponseBodyMethodProcessor → text/plain body
// "product/list" written as literal string to response
// NOT a view name — critical distinction
```

**`ViewNameMethodReturnValueHandler.supportsReturnType()`:**
```java
@Override
public boolean supportsReturnType(MethodParameter returnType) {
    Class<?> paramType = returnType.getParameterType();
    return (void.class == paramType ||
            CharSequence.class.isAssignableFrom(paramType));
}
```

It checks `CharSequence` — String, StringBuilder, StringBuffer all qualify. But `RequestResponseBodyMethodProcessor` checks for `@ResponseBody` FIRST (it's earlier in the chain). In `@RestController`, RRPBMP claims String returns before `ViewNameMethodReturnValueHandler`.

---

### ResponseEntity — The Most Powerful Return Type

```java
// ResponseEntity gives full control: status + headers + body
@GetMapping("/products/{id}")
public ResponseEntity<Product> getById(@PathVariable Long id) {

    return productService.findById(id)
        .map(product -> {
            return ResponseEntity
                .ok()                         // 200 OK
                .contentType(MediaType.APPLICATION_JSON)
                .eTag(product.getEtag())
                .lastModified(product.getLastModified())
                .body(product);
                // HttpEntityMethodProcessor:
                // 1. response.setStatus(200)
                // 2. response.setContentType("application/json")
                // 3. response.setHeader("ETag", product.getEtag())
                // 4. Jackson serialises product → response body
                // 5. mavContainer.requestHandled = true
        })
        .orElse(ResponseEntity.notFound().build());
        // 404 with no body
}

// ResponseEntity builder patterns:
ResponseEntity.ok(body)                     // 200 with body
ResponseEntity.created(location).body(body) // 201 with Location header
ResponseEntity.noContent().build()          // 204 no body
ResponseEntity.badRequest().body(error)     // 400 with body
ResponseEntity.notFound().build()           // 404 no body
ResponseEntity.status(HttpStatus.ACCEPTED)  // Any status
    .header("X-Custom", "value")
    .body(data)

// Wildcard for error handling:
ResponseEntity<?> response = ...  // Body type not known at compile time
```

**`HttpEntityMethodProcessor` vs `RequestResponseBodyMethodProcessor`:**

```
ResponseEntity<T> → HttpEntityMethodProcessor
  1. Extract status from ResponseEntity
  2. Extract headers from ResponseEntity
  3. Set status on HttpServletResponse
  4. Set each header on HttpServletResponse
  5. Write body via HttpMessageConverter (same as @ResponseBody)
  6. mavContainer.requestHandled = true

@ResponseBody method → RequestResponseBodyMethodProcessor
  1. Content negotiation (no explicit status/headers)
  2. Default status: 200 (or @ResponseStatus value)
  3. No custom headers possible
  4. Write body via HttpMessageConverter
  5. mavContainer.requestHandled = true

Key difference: ResponseEntity controls status + headers programmatically
               @ResponseBody uses @ResponseStatus for status (annotation only)
```

---

### void Return — Context-Dependent Behaviour

```java
// void return — THREE different behaviours depending on context

// CASE 1: @RestController method — requestHandled=true, no view
@RestController
public class RestCtrl {
    @GetMapping("/data")
    public void getData(HttpServletResponse response)
            throws IOException {
        // Write directly to response
        response.getWriter().write("data");
        // void + @RestController → RRPBMP handles it
        // mavContainer.requestHandled = true
        // No view rendering
    }
}

// CASE 2: @Controller method — response committed, no view
@Controller
public class MvcCtrl {
    @GetMapping("/download")
    public void download(HttpServletResponse response)
            throws IOException {
        response.setContentType("application/pdf");
        response.getOutputStream().write(pdfData);
        // void + @Controller + committed response
        // Spring detects committed response
        // mavContainer.requestHandled = true
    }

    // CASE 3: @Controller void method — response NOT committed
    @GetMapping("/setup")
    public void setup(Model model) {
        model.addAttribute("data", service.getData());
        // void + @Controller + uncommitted response
        // ViewNameMethodReturnValueHandler:
        //   mavContainer.requestHandled = false
        // DefaultRequestToViewNameTranslator:
        //   derives "setup" from /setup URL
        // InternalResourceViewResolver:
        //   resolves to /WEB-INF/views/setup.jsp
    }
}
```

---

### Async Return Types — Deep Analysis

#### Callable<T>

```java
@GetMapping("/slow-data")
public Callable<List<Product>> slowData() {
    // Returns immediately — Servlet thread released
    return () -> {
        // Runs on TaskExecutor thread pool (NOT Servlet thread)
        Thread.sleep(3000);
        return productService.findAll();
        // When complete:
        // asyncManager dispatches new request to DispatcherServlet
        // Second pass: result is List<Product>
        // @ResponseBody handling applies to the result
        // JSON written to response
    };
    // Time to return: ~0ms (Callable created, not executed)
}
```

#### DeferredResult<T>

```java
@GetMapping("/long-poll")
public DeferredResult<String> longPoll() {
    DeferredResult<String> result =
        new DeferredResult<>(
            30_000L,               // timeout: 30 seconds
            "timeout"              // timeout default value
        );

    // Register for event — no thread holding this request
    eventBus.register(event -> {
        result.setResult(event.getData());
        // → Triggers async dispatch
        // → Second pass processes "string result" as @ResponseBody
    });

    return result;
    // Servlet thread released immediately
    // Tomcat thread pool NOT occupied waiting
    // Up to 30 seconds can elapse with NO thread held
}
```

#### CompletableFuture<T>

```java
@GetMapping("/async-compute")
public CompletableFuture<Product> asyncCompute(
        @PathVariable Long id) {
    // DeferredResultMethodReturnValueHandler handles CompletableFuture
    return CompletableFuture.supplyAsync(() ->
        productService.findById(id)
            .orElseThrow(),
        taskExecutor
    );
    // Adapted to DeferredResult internally
    // Completes when future resolves
}
```

#### Mono<T> — Spring MVC + Reactor

```java
@GetMapping("/reactive")
public Mono<Product> reactive(@PathVariable Long id) {
    // ReactiveTypeReturnValueHandler:
    // → Subscribes to Mono
    // → Adapts to DeferredResult
    // → When Mono completes → result dispatched via DeferredResult path
    return webClient.get()
        .uri("/external/{id}", id)
        .retrieve()
        .bodyToMono(Product.class);
    // Non-blocking HTTP call
    // But still runs on Servlet container (blocking eventually)
}

// Flux for SSE streaming
@GetMapping(value="/stream",
            produces=MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<Product> stream() {
    // ReactiveTypeReturnValueHandler:
    // → Accept: text/event-stream detected
    // → Routes to ResponseBodyEmitter (SseEmitter) path
    // → Each element emitted as SSE event
    return Flux.interval(Duration.ofSeconds(1))
        .map(i -> productService.findRandom());
}
```

---

### Streaming Return Types

```java
// StreamingResponseBody — stream large files
@GetMapping("/export")
public ResponseEntity<StreamingResponseBody> exportLargeData() {
    StreamingResponseBody body = outputStream -> {
        // This runs on async thread
        // Can write large amounts of data without holding Servlet thread
        try (var data = dataService.streamAll()) {
            data.forEach(item -> {
                try {
                    outputStream.write(
                        (item.toCsv() + "\n").getBytes());
                } catch (IOException e) {
                    throw new UncheckedIOException(e);
                }
            });
        }
    };

    return ResponseEntity.ok()
        .contentType(MediaType.parseMediaType("text/csv"))
        .header("Content-Disposition",
            "attachment; filename=export.csv")
        .body(body);
}

// SseEmitter — Server-Sent Events
@GetMapping(value="/events",
            produces=MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter sseStream() {
    SseEmitter emitter = new SseEmitter(60_000L); // 60s timeout

    // Runs on background thread
    taskExecutor.execute(() -> {
        try {
            for (int i = 0; i < 10; i++) {
                emitter.send(SseEmitter.event()
                    .id(String.valueOf(i))
                    .name("update")
                    .data(productService.getUpdate(i)));
                Thread.sleep(1000);
            }
            emitter.complete();
        } catch (Exception e) {
            emitter.completeWithError(e);
        }
    });

    return emitter;
    // Connection held open
    // Events pushed as data becomes available
}
```

---

### HttpHeaders Return Type

```java
// Returns ONLY headers — no body
@RequestMapping(value="/resources/{id}",
                method=RequestMethod.HEAD)
public HttpHeaders headResource(@PathVariable Long id) {
    Resource resource = storageService.getResource(id);

    HttpHeaders headers = new HttpHeaders();
    headers.setContentLength(resource.contentLength());
    headers.setContentType(
        MediaType.APPLICATION_OCTET_STREAM);
    headers.setLastModified(resource.lastModified());
    headers.setETag(resource.getEtag());

    return headers;
    // HttpHeadersReturnValueHandler:
    // → Sets each header on response
    // → response body is empty
    // → mavContainer.requestHandled = true
}
```

---

### Content Negotiation in Return Value Processing

For handlers that write to the response body (`RequestResponseBodyMethodProcessor`, `HttpEntityMethodProcessor`), content negotiation determines which `HttpMessageConverter` is selected:

```java
// writeWithMessageConverters() inside AbstractMessageConverterMethodProcessor

protected <T> void writeWithMessageConverters(
        @Nullable T value,
        MethodParameter returnType,
        ServletServerHttpRequest inputMessage,
        ServletServerHttpResponse outputMessage)
        throws IOException, HttpMediaTypeNotAcceptableException,
               HttpMessageNotWritableException {

    // STEP 1: Determine requested media types
    List<MediaType> acceptableTypes =
        getAcceptableMediaTypes(
            (HttpServletRequest) inputMessage.getNativeRequest());
    // From Accept header, content negotiation strategy

    // STEP 2: Determine producible media types
    List<MediaType> producibleTypes =
        getProducibleMediaTypes(
            (HttpServletRequest) inputMessage.getNativeRequest(),
            valueType,
            targetType);
    // From: @RequestMapping(produces=...) if set
    // OR: all converters' supported types for this value type

    // STEP 3: Find compatible types
    List<MediaType> mediaTypesToUse = new ArrayList<>();
    for (MediaType requestedType : acceptableTypes) {
        for (MediaType producibleType : producibleTypes) {
            if (requestedType.isCompatibleWith(producibleType)) {
                mediaTypesToUse.add(
                    getMostSpecificMediaType(
                        requestedType, producibleType));
            }
        }
    }

    // STEP 4: Sort by specificity
    MediaType.sortBySpecificityAndQuality(mediaTypesToUse);

    // STEP 5: Select first compatible type
    for (MediaType mediaType : mediaTypesToUse) {
        if (mediaType.isConcrete()) {
            selectedContentType = mediaType;
            break;
        }
    }

    // STEP 6: Find converter that can write this type
    for (HttpMessageConverter<?> converter :
            this.messageConverters) {
        if (converter.canWrite(valueType, selectedContentType)) {
            // Found! Invoke ResponseBodyAdvice.beforeBodyWrite()
            // Then converter.write(value, selectedContentType, outputMessage)
            break;
        }
    }

    // No converter found:
    throw new HttpMediaTypeNotAcceptableException(producibleTypes);
    // → 406 Not Acceptable
}
```

---

### ResponseBodyAdvice — Intercepting Body Writing

```java
public interface ResponseBodyAdvice<T> {
    // Should this advice apply to this return type + converter?
    boolean supports(
        MethodParameter returnType,
        Class<? extends HttpMessageConverter<?>> converterType);

    // Modify body before writing
    @Nullable
    T beforeBodyWrite(
        @Nullable T body,
        MethodParameter returnType,
        MediaType selectedContentType,
        Class<? extends HttpMessageConverter<?>> selectedConverterType,
        ServerHttpRequest request,
        ServerHttpResponse response);
}

// Called INSIDE writeWithMessageConverters()
// BEFORE converter.write() — can modify body or response
// Set on @ControllerAdvice class or configured on RequestMappingHandlerAdapter
```

---

## 2️⃣ CODE EXAMPLES

### All Return Types Side-by-Side

```java
@RestController
@RequestMapping("/demo")
public class ReturnTypeShowcase {

    // [1] String → text/plain (NOT view name in @RestController)
    @GetMapping("/string")
    public String asString() {
        return "Hello World";
        // Response: "Hello World", Content-Type: text/plain
    }

    // [2] Domain object → JSON
    @GetMapping("/object")
    public Product asObject() {
        return productService.findFirst();
        // Response: {"id":1,"name":"Widget",...}
        // Content-Type: application/json
    }

    // [3] Collection → JSON array
    @GetMapping("/list")
    public List<Product> asList() {
        return productService.findAll();
        // Response: [{"id":1,...},{"id":2,...}]
    }

    // [4] ResponseEntity — full control
    @GetMapping("/entity/{id}")
    public ResponseEntity<Product> asEntity(
            @PathVariable Long id) {
        return productService.findById(id)
            .map(p -> ResponseEntity.ok()
                .lastModified(p.getModifiedAt())
                .eTag(p.getVersion().toString())
                .body(p))
            .orElse(ResponseEntity.notFound().build());
    }

    // [5] void — write directly
    @GetMapping("/raw")
    public void raw(HttpServletResponse response)
            throws IOException {
        response.setContentType("text/plain");
        response.getWriter().write("raw output");
    }

    // [6] Callable — async
    @GetMapping("/async")
    public Callable<Product> async() {
        return () -> expensiveService.compute();
        // Servlet thread released immediately
    }

    // [7] DeferredResult — external completion
    @GetMapping("/deferred")
    public DeferredResult<String> deferred() {
        DeferredResult<String> result =
            new DeferredResult<>(5000L, "timeout");
        scheduler.schedule(
            () -> result.setResult("completed"),
            2, TimeUnit.SECONDS);
        return result;
    }

    // [8] CompletableFuture
    @GetMapping("/future/{id}")
    public CompletableFuture<Product> future(
            @PathVariable Long id) {
        return CompletableFuture.supplyAsync(
            () -> productService.findById(id).orElseThrow(),
            executor);
    }

    // [9] Mono (Reactor)
    @GetMapping("/mono/{id}")
    public Mono<Product> mono(@PathVariable Long id) {
        return productService.findByIdReactive(id);
    }

    // [10] Flux as SSE
    @GetMapping(value="/flux",
                produces=MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<Product> flux() {
        return productService.streamAll();
    }

    // [11] StreamingResponseBody
    @GetMapping("/stream")
    public ResponseEntity<StreamingResponseBody> stream() {
        return ResponseEntity.ok()
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .body(out -> largeDataService.writeTo(out));
    }

    // [12] SseEmitter
    @GetMapping(value="/sse",
                produces=MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter sse() {
        SseEmitter emitter = new SseEmitter();
        eventService.subscribe(emitter);
        return emitter;
    }
}
```

---

### ResponseEntity — Advanced Patterns

```java
@RestController
@RequestMapping("/api/products")
public class ProductApiController {

    // Conditional GET with ETag/Last-Modified
    @GetMapping("/{id}")
    public ResponseEntity<Product> getWithCaching(
            @PathVariable Long id,
            WebRequest webRequest) {

        Product product = productService.findById(id)
            .orElseThrow(() ->
                new ResponseStatusException(
                    HttpStatus.NOT_FOUND));

        // Spring checks If-None-Match / If-Modified-Since
        if (webRequest.checkNotModified(
                product.getEtag(),
                product.getLastModified().toEpochMilli())) {
            // Returns 304 Not Modified with no body
            return null; // Spring handles 304 response
        }

        return ResponseEntity.ok()
            .eTag(product.getEtag())
            .lastModified(product.getLastModified())
            .cacheControl(CacheControl.maxAge(
                Duration.ofMinutes(5)))
            .body(product);
    }

    // Created with Location header
    @PostMapping
    public ResponseEntity<Product> create(
            @Valid @RequestBody Product product,
            UriComponentsBuilder ucb) {

        Product saved = productService.save(product);

        URI location = ucb.path("/api/products/{id}")
            .buildAndExpand(saved.getId())
            .toUri();

        return ResponseEntity
            .created(location)   // 201 Created
            .body(saved);
        // Response: 201, Location: /api/products/42, body: {saved product}
    }

    // No content
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(
            @PathVariable Long id) {
        productService.delete(id);
        return ResponseEntity.noContent().build();
        // 204 No Content, no body
    }

    // Partial content (range requests)
    @GetMapping("/{id}/data")
    public ResponseEntity<byte[]> getData(
            @PathVariable Long id,
            @RequestHeader(value="Range",
                           required=false) String range) {
        byte[] data = productService.getData(id);

        if (range != null) {
            // Parse range, return partial content
            return ResponseEntity.status(
                    HttpStatus.PARTIAL_CONTENT)
                .header("Content-Range", "bytes 0-99/1000")
                .body(Arrays.copyOf(data, 100));
        }
        return ResponseEntity.ok().body(data);
    }
}
```

---

### Async Patterns — Timeout and Error Handling

```java
@RestController
@RequestMapping("/api/async")
public class AsyncController {

    @Autowired
    private ThreadPoolTaskExecutor taskExecutor;

    // Callable with timeout
    @GetMapping("/callable-timeout")
    public WebAsyncTask<String> callableWithTimeout() {
        Callable<String> callable =
            () -> slowService.compute();

        // WebAsyncTask allows timeout + callback configuration
        WebAsyncTask<String> asyncTask =
            new WebAsyncTask<>(5000L, callable);
            // 5 second timeout

        asyncTask.onTimeout(() -> {
            log.warn("Async task timed out");
            return "timeout";
            // Returned as the result on timeout
        });

        asyncTask.onError(ex -> {
            log.error("Async task error", ex);
            return "error: " + ex.getMessage();
        });

        asyncTask.onCompletion(() ->
            log.info("Async task completed"));

        return asyncTask;
    }

    // DeferredResult with structured timeout handling
    @GetMapping("/deferred-error")
    public DeferredResult<ResponseEntity<String>> deferredError() {
        DeferredResult<ResponseEntity<String>> result =
            new DeferredResult<>(10_000L);

        result.onTimeout(() ->
            result.setErrorResult(
                ResponseEntity.status(
                    HttpStatus.REQUEST_TIMEOUT)
                .body("Request timed out")));

        result.onError(ex ->
            result.setErrorResult(
                ResponseEntity.internalServerError()
                    .body("Error: " + ex.getMessage())));

        taskExecutor.execute(() -> {
            try {
                String data = externalService.fetch();
                result.setResult(ResponseEntity.ok(data));
            } catch (Exception e) {
                result.setErrorResult(e);
            }
        });

        return result;
    }
}
```

---

### ResponseBodyAdvice — Global Response Wrapping

```java
// Applied to all @RestController methods
@ControllerAdvice
public class StandardResponseAdvice
    implements ResponseBodyAdvice<Object> {

    @Override
    public boolean supports(
            MethodParameter returnType,
            Class<? extends HttpMessageConverter<?>> converterType) {

        // Apply only to JSON converters
        if (!MappingJackson2HttpMessageConverter.class
                .isAssignableFrom(converterType)) {
            return false;
        }

        // Skip if method is in error handling controllers
        // or already returns StandardResponse
        Class<?> returnClass = returnType.getParameterType();
        return !StandardResponse.class
            .isAssignableFrom(returnClass) &&
               !ErrorResponse.class
            .isAssignableFrom(returnClass);
    }

    @Override
    public Object beforeBodyWrite(
            Object body,
            MethodParameter returnType,
            MediaType selectedContentType,
            Class<? extends HttpMessageConverter<?>> converterType,
            ServerHttpRequest request,
            ServerHttpResponse response) {

        if (body == null) {
            return StandardResponse.empty();
        }

        // Modify response headers
        response.getHeaders()
            .add("X-Response-Time",
                String.valueOf(System.currentTimeMillis()));

        return StandardResponse.of(body);
        // {"success":true,"data":<original body>,"timestamp":...}
    }
}

// StandardResponse record
public record StandardResponse<T>(
    boolean success,
    T data,
    String timestamp
) {
    public static <T> StandardResponse<T> of(T data) {
        return new StandardResponse<>(true, data,
            Instant.now().toString());
    }
    public static StandardResponse<Void> empty() {
        return new StandardResponse<>(true, null,
            Instant.now().toString());
    }
}
```

---

### Edge Case — Map Return Without Annotation

```java
@Controller  // NOT @RestController
public class MapReturnController {

    // Map without annotation → MapMethodProcessor
    // → Adds to MODEL, not JSON response
    // → View rendered with these model attributes
    @GetMapping("/view-map")
    public Map<String, Object> viewMap() {
        return Map.of("products", productService.findAll(),
                      "count", productService.count());
        // MapMethodProcessor adds to mavContainer model
        // ViewNameMethodReturnValueHandler does NOT run
        // Default view name derived: "view-map"
        // InternalResourceViewResolver: /WEB-INF/views/view-map.jsp
    }

    // @ResponseBody Map → JSON (not model)
    @GetMapping("/json-map")
    @ResponseBody
    public Map<String, Object> jsonMap() {
        return Map.of("products", productService.findAll());
        // RequestResponseBodyMethodProcessor selected first
        // Jackson serialises Map → JSON
        // NOT added to model
    }
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
A `@Controller` (not `@RestController`) method returns `Map<String, Object>` without `@ResponseBody`. What does Spring do with the return value?

A) Serialises it as JSON via `MappingJackson2HttpMessageConverter`
B) Adds it to the model and derives view name from URL
C) Throws `IllegalArgumentException` — `Map` is not a valid return type
D) Returns it as a JSON response body

**Answer: B**
`MapMethodProcessor.supportsReturnType()` returns `true` for `Map` type in a non-`@ResponseBody` context. `handleReturnValue()` calls `mavContainer.addAllAttributes(map)`. `requestHandled` remains `false`. `DefaultRequestToViewNameTranslator` derives view name from URL. `ViewResolver` is called. Map contents are available in the view as model attributes.

---

**Q2 — Select All That Apply**
Which return types result in `mavContainer.setRequestHandled(true)` being called, thereby skipping ViewResolver?

A) `String` in `@Controller` method
B) `ResponseEntity<T>`
C) `Callable<T>`
D) `void` in `@RestController` method
E) `ModelAndView`

**Answer: B, D**
A: `ViewNameMethodReturnValueHandler` sets view name — requestHandled=false. B: `HttpEntityMethodProcessor` writes response — requestHandled=true. C: `CallableMethodReturnValueHandler` starts async — requestHandled=false on main thread (result processed on second dispatch). D: `RequestResponseBodyMethodProcessor` handles void in `@RestController` — requestHandled=true. E: `ModelAndViewMethodReturnValueHandler` sets view — requestHandled=false.

---

**Q3 — Code Prediction**
```java
@RestController
public class TestCtrl {

    @GetMapping("/data")
    public String data() {
        return "product/list";
    }
}
```
`GET /data` with `Accept: text/html`. What is the response?

A) Redirects to `/product/list`
B) Renders `/WEB-INF/views/product/list.jsp`
C) Returns literal text `"product/list"` with `Content-Type: text/plain`
D) HTTP 406 — `StringHttpMessageConverter` cannot produce `text/html`

**Answer: C**
`@RestController` has class-level `@ResponseBody`. `RequestResponseBodyMethodProcessor` is selected (before `ViewNameMethodReturnValueHandler`). `StringHttpMessageConverter` writes `"product/list"` to the response. It produces `text/plain` and `*/*`. `Accept: text/html` is compatible with `text/plain` (or */* wildcard match). Response: `"product/list"` as `text/plain`. NOT a view name.

---

**Q4 — MCQ**
`DeferredResultMethodReturnValueHandler` handles which of the following return types?

A) `Callable<T>` only
B) `DeferredResult<T>` and `CompletableFuture<T>`
C) `DeferredResult<T>`, `CompletableFuture<T>`, and `Mono<T>`
D) `DeferredResult<T>` only

**Answer: B**
`DeferredResultMethodReturnValueHandler.supportsReturnType()` returns `true` for `DeferredResult`, `ListenableFuture` (deprecated), and `CompletableFuture`. `Mono<T>` is handled by `ReactiveTypeReturnValueHandler` (a separate handler). `Callable<T>` is handled by `CallableMethodReturnValueHandler`.

---

**Q5 — True/False**
A `@Controller` method returning `void` with an uncommitted response always triggers view rendering with a view name derived from the request URL.

**Answer: True**
`ViewNameMethodReturnValueHandler.supportsReturnType()` returns `true` for `void`. When `handleReturnValue()` is called with a `void` method in `@Controller` (not `@RestController`) where response is not committed, it leaves `mavContainer.requestHandled=false`. `applyDefaultViewName()` in `doDispatch()` then calls `DefaultRequestToViewNameTranslator.getViewName()` to derive the view name from the URL. ViewResolver is called. If no view exists, runtime error during rendering.

---

**Q6 — MCQ**
`ResponseEntity<Product>` is returned with `HttpStatus.CREATED` and a `Location` header. What is the processing sequence in `HttpEntityMethodProcessor`?

A) Write body → set status → set headers
B) Set headers → set status → write body
C) Set status → set headers → write body (Jackson)
D) Set status → write body (Jackson) → set headers

**Answer: C**
`HttpEntityMethodProcessor.handleReturnValue()` processes in this order: (1) extract and set HTTP status via `response.setStatus()`, (2) extract and set all headers from `HttpHeaders`, (3) write body via `writeWithMessageConverters()` which selects `MappingJackson2HttpMessageConverter` and serialises the `Product`. Headers must be set BEFORE body writing because once `getOutputStream().write()` is called, the response may be committed and headers can no longer be changed.

---

**Q7 — Select All That Apply**
Which return types initiate async processing and release the Servlet container thread before the response is complete?

A) `Callable<T>`
B) `DeferredResult<T>`
C) `ResponseEntity<T>`
D) `SseEmitter`
E) `CompletableFuture<T>`

**Answer: A, B, D, E**
`Callable<T>` (A): submitted to `TaskExecutor`, Servlet thread released. `DeferredResult<T>` (B): no thread holds the request, result set externally. `ResponseEntity<T>` (C): synchronous — response written immediately on Servlet thread. `SseEmitter` (D): connection kept open but Servlet thread released (async started). `CompletableFuture<T>` (E): adapted to `DeferredResult`, Servlet thread released.

---

**Q8 — Scenario**
`RequestResponseBodyMethodProcessor` processes a `@ResponseBody` method returning `Product`. `Accept: application/json`. Jackson is on the classpath. But the developer called `configureMessageConverters()` with only `StringHttpMessageConverter`. What happens?

A) Jackson is used — it's always available
B) `Product` serialised as its `toString()` via `StringHttpMessageConverter`
C) `HttpMediaTypeNotAcceptableException` — no converter can write `Product` as `application/json`
D) Spring falls back to XML serialisation

**Answer: C**
`configureMessageConverters()` REPLACES all default converters. Only `StringHttpMessageConverter` remains. `writeWithMessageConverters()` checks: can `StringHttpMessageConverter` write `Product` as `application/json`? No — `StringHttpMessageConverter` only handles `String`. No converter matches. `HttpMediaTypeNotAcceptableException` thrown → HTTP 406.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — `String` return in `@RestController` is NOT a view name**
In `@Controller`: `String` → `ViewNameMethodReturnValueHandler` → view name → ViewResolver.
In `@RestController`: `String` → `RequestResponseBodyMethodProcessor` → text body → `StringHttpMessageConverter`.
The return value `"redirect:/home"` in a `@RestController` method sends the literal string `"redirect:/home"` as the response body. Not a redirect. This is the most common `@RestController` misuse.

**Trap 2 — `Callable<T>` releases the Servlet thread — `postHandle()` timing**
When `Callable<T>` is returned, DispatcherServlet exits at Phase 7. `postHandle()` is NOT called on the first dispatch. It IS called on the second dispatch (when async result arrives). Interceptors that rely on `postHandle()` for request-level cleanup will run on a DIFFERENT request cycle than the original handler invocation.

**Trap 3 — `void` in `@Controller` with uncommitted response = view rendering**
Developers write `void` methods that set model attributes and expect the method to "just return" with no view. But uncommitted `void` in `@Controller` → view name derived from URL → ViewResolver called → JSP rendered. The response appears to have extra content or throws an error if no JSP exists. Always return `void` in `@Controller` ONLY if you're writing directly to the response.

**Trap 4 — `HttpEntityMethodProcessor` vs `RequestResponseBodyMethodProcessor`**
`ResponseEntity<T>` → `HttpEntityMethodProcessor`.
`@ResponseBody` method → `RequestResponseBodyMethodProcessor`.
Both use `writeWithMessageConverters()` for body writing, but `HttpEntityMethodProcessor` additionally extracts and sets the HTTP status and response headers from the entity. `@ResponseBody` alone cannot set custom headers (use `@ResponseStatus` for status only).

**Trap 5 — `Map` without annotation = model map in `@Controller`**
`MapMethodProcessor` handles unannotated `Map` returns in `@Controller` — adds to model. `RequestResponseBodyMethodProcessor` handles `Map` in `@RestController` (class-level `@ResponseBody`) — serialises as JSON. The annotation context completely changes what `Map` means. Removing `@RestController` from a REST controller and replacing with `@Controller` causes all `Map` return values to add to model instead of becoming JSON responses.

**Trap 6 — `ResponseBodyAdvice.beforeBodyWrite()` can return `null`**
Returning `null` from `beforeBodyWrite()` means "write nothing." This can suppress the response body. If the developer expects the original body but accidentally returns `null` — blank HTTP 200 response. Always return `body` (or modified body) unless intentionally suppressing.

**Trap 7 — `DeferredResult` timeout value vs default value**
`new DeferredResult<>(5000L)` — 5 second timeout. On timeout: `AsyncRequestTimeoutException` → `DefaultHandlerExceptionResolver` → HTTP 503.
`new DeferredResult<>(5000L, "timeout-response")` — timeout returns "timeout-response" as the result (processed normally as return value). The second constructor arg is the RESULT on timeout, not an error. Many assume timeout always means error — with a default value it doesn't.

---

## 5️⃣ SUMMARY SHEET

```
RETURN VALUE HANDLER CHAIN — KEY HANDLERS IN ORDER
─────────────────────────────────────────────────────
[1]  ModelAndViewMethodReturnValueHandler
       ModelAndView → sets view + model, requestHandled=false

[3]  ViewMethodReturnValueHandler
       View object → sets View, requestHandled=false

[4]  ResponseBodyEmitterReturnValueHandler
       ResponseBodyEmitter, SseEmitter → async streaming

[5]  StreamingResponseBodyReturnValueHandler
       StreamingResponseBody → async streaming, requestHandled=true

[6]  HttpEntityMethodProcessor
       ResponseEntity<T>, HttpEntity<T>
       → sets status + headers + writes body, requestHandled=true

[7]  HttpHeadersReturnValueHandler
       HttpHeaders → sets headers only, no body, requestHandled=true

[8]  CallableMethodReturnValueHandler
       Callable<T> → starts async via TaskExecutor

[9]  DeferredResultMethodReturnValueHandler
       DeferredResult<T>, CompletableFuture<T> → external async

[11] ReactiveTypeReturnValueHandler
       Mono<T> → DeferredResult path
       Flux<T> + SSE → SseEmitter path

[12] RequestResponseBodyMethodProcessor
       @ResponseBody method or @RestController class
       → content negotiation → MessageConverter → requestHandled=true

[13] ViewNameMethodReturnValueHandler
       CharSequence (String, etc.) in @Controller
       → sets view name, requestHandled=false

[14] MapMethodProcessor
       Map without annotation → adds to model, requestHandled=false

requestHandled=true  → ViewResolver skipped
requestHandled=false → ViewResolver called

RESPONSESTATUS CODE MATRIX
─────────────────────────────────────────────────────
@ResponseBody method (no annotation)          → 200 OK (default)
@ResponseStatus(HttpStatus.CREATED)           → 201 Created
@ResponseStatus(HttpStatus.NO_CONTENT)        → 204 No Content
ResponseEntity.ok(body)                       → 200 OK
ResponseEntity.created(uri)                   → 201 Created
ResponseEntity.noContent().build()            → 204 No Content
ResponseEntity.notFound().build()             → 404 Not Found
ResponseEntity.status(HttpStatus.ACCEPTED)    → 202 Accepted

HTTPENTITYMETHODPROCESSOR vs REQUESTRESPONSEBODYMETHODPROCESSOR
─────────────────────────────────────────────────────
ResponseEntity<T> → HttpEntityMethodProcessor
  → Extracts status, sets on response
  → Extracts headers, sets on response
  → Writes body via MessageConverter
  → requestHandled=true

@ResponseBody → RequestResponseBodyMethodProcessor
  → Default 200 (or @ResponseStatus value)
  → No custom headers (except via ResponseBodyAdvice)
  → Writes body via MessageConverter
  → requestHandled=true

ASYNC RETURN TYPES
─────────────────────────────────────────────────────
Callable<T>          → TaskExecutor thread, second dispatch
DeferredResult<T>    → External completion, second dispatch
CompletableFuture<T> → Adapted to DeferredResult
WebAsyncTask<T>      → Callable with timeout + callbacks
Mono<T>              → Adapted to DeferredResult
Flux<T> + SSE        → SseEmitter path, streaming
SseEmitter           → Connection held, events pushed
StreamingResponseBody → Async stream to OutputStream

VOID RETURN BEHAVIOURS
─────────────────────────────────────────────────────
@RestController void     → requestHandled=true (no view)
@Controller void + committed response → requestHandled=true
@Controller void + uncommitted → viewName derived from URL → view rendered

CONTENT NEGOTIATION IN BODY WRITING
─────────────────────────────────────────────────────
1. Get acceptable media types from Accept header
2. Get producible media types from @RequestMapping(produces) or converters
3. Find compatible intersection
4. Sort by specificity+quality
5. Find HttpMessageConverter supporting selected type
6. ResponseBodyAdvice.beforeBodyWrite() called
7. converter.write(body, mediaType, response)
8. No converter found → HttpMediaTypeNotAcceptableException → 406

STRING IN @RESTCONTROLLER
─────────────────────────────────────────────────────
@RestController → class-level @ResponseBody
String → RequestResponseBodyMethodProcessor (wins over ViewNameHandler)
→ StringHttpMessageConverter → text/plain body
"redirect:/home" → literal text "redirect:/home" in response
NOT a redirect — use return ResponseEntity.status(302).location(uri).build()

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "ResponseEntity sets status+headers+body; @ResponseBody only writes body"
• "String in @RestController → text body (NOT view name)"
• "void @Controller + uncommitted response → view derived from URL"
• "Callable releases Servlet thread; postHandle on second dispatch"
• "DeferredResult timeout default value → treated as normal result, not error"
• "Map without @ResponseBody in @Controller → adds to model map"
• "mavContainer.requestHandled=true → ViewResolver never called"
• "HttpMediaTypeNotAcceptableException if no converter matches selected type → 406"
• "ResponseBodyAdvice.beforeBodyWrite: returning null suppresses body entirely"
• "CompletableFuture adapted to DeferredResult by DeferredResultMethodReturnValueHandler"
```

---

# TOPIC 2.7 — VIEW RESOLUTION AND RENDERING PIPELINE

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What View Resolution Really Is

View resolution is the process of converting a **logical view name** (a String like `"product/list"`) into a **`View` object** capable of rendering a response. It sits entirely within Phase 10 of `doDispatch()` — inside `processDispatchResult()` → `render()`. Understanding it deeply means understanding: how the resolver chain works, how `ContentNegotiatingViewResolver` delegates, what `View.render()` does internally, how redirects work at the byte level, and how the entire pipeline is bypassed for REST.

---

### The Full View Layer Architecture

```
ModelAndView (view name + model map)
        │
        ▼
DispatcherServlet.render()
        │
        ├── localeResolver.resolveLocale() → Locale
        ├── resolveViewName(viewName, model, locale, request)
        │       │
        │       └── ViewResolver chain (ordered)
        │             ├── ContentNegotiatingViewResolver (delegates to all)
        │             ├── BeanNameViewResolver
        │             ├── ThymeleafViewResolver (if Thymeleaf on classpath)
        │             ├── FreeMarkerViewResolver
        │             ├── InternalResourceViewResolver (always last)
        │             └── Returns first non-null View
        │
        └── view.render(model, request, response)
              │
              ├── AbstractView.render() — base implementation
              │     ├── createMergedOutputModel() — merges static + dynamic attributes
              │     ├── prepareResponse() — sets Content-Type, charset
              │     └── renderMergedOutputModel() — concrete rendering
              │
              ├── InternalResourceView — JSP forward
              ├── RedirectView — HTTP 302 redirect
              ├── MappingJackson2JsonView — JSON rendering
              └── AbstractThymeleafView — Thymeleaf processing
```

---

### ViewResolver Interface — Precise Contract

```java
public interface ViewResolver {

    @Nullable
    View resolveViewName(String viewName, Locale locale)
        throws Exception;
    // Returns null: this resolver cannot handle this view name
    //               try next resolver in chain
    // Returns View: use this View for rendering
    // Throws:       resolution failed (distinct from returning null)

    // NOTE: No ordering mechanism in the interface itself
    // Ordering via @Order, Ordered, or configureViewResolvers() position
}
```

**The chain iteration (inside `DispatcherServlet.resolveViewName()`):**

```java
@Nullable
protected View resolveViewName(String viewName,
        @Nullable Map<String, Object> model,
        Locale locale,
        HttpServletRequest request) throws Exception {

    if (this.viewResolvers != null) {
        for (ViewResolver viewResolver : this.viewResolvers) {
            View view = viewResolver.resolveViewName(
                viewName, locale);
            if (view != null) {
                return view;
                // FIRST non-null wins
                // All remaining resolvers skipped
            }
        }
    }
    return null;
    // No resolver could handle this view name
    // → render() throws ServletException
}
```

---

### AbstractCachingViewResolver — Caching Layer

Most `ViewResolver` implementations extend `AbstractCachingViewResolver`:

```java
public abstract class AbstractCachingViewResolver
    implements ViewResolver {

    // Cache: Map<cacheKey(viewName+locale), View>
    // Default capacity: 1024
    private final Map<Object, View> viewAccessCache =
        new ConcurrentHashMap<>(DEFAULT_CACHE_LIMIT);

    @Override
    @Nullable
    public View resolveViewName(String viewName, Locale locale)
            throws Exception {

        if (!isCache()) {
            // Caching disabled — create new View every request
            return createView(viewName, locale);
        }

        Object cacheKey = getCacheKey(viewName, locale);
        View view = this.viewAccessCache.get(cacheKey);

        if (view == null) {
            synchronized (this.viewCreationCache) {
                // Double-checked locking
                view = this.viewCreationCache.get(cacheKey);
                if (view == null) {
                    // Create the View object
                    view = createView(viewName, locale);
                    // Store result (including UNRESOLVED_VIEW sentinel)
                    if (view != null) {
                        this.viewAccessCache.put(cacheKey, view);
                    }
                }
            }
        }

        // UNRESOLVED_VIEW is a sentinel meaning "this view name
        // was already tried and cannot be resolved"
        return (view != UNRESOLVED_VIEW ? view : null);
    }
}
```

**Critical implications:**
- `View` objects are created ONCE and cached (unless `cache=false`)
- `InternalResourceView` for `/WEB-INF/views/product/list.jsp` is ONE cached instance
- Caching means `createView()` reflection/file scanning only happens on first access
- `setCacheLimit(0)` disables caching — useful in development (template changes reflect immediately)
- **Thread-safety:** `View.render()` MUST be thread-safe — multiple concurrent requests share same `View` instance

---

### InternalResourceViewResolver — Internal Analysis

The most commonly used resolver in traditional Spring MVC:

```java
public class InternalResourceViewResolver
    extends UrlBasedViewResolver {

    // viewClass = InternalResourceView or JstlView (if JSTL detected)

    @Override
    protected Class<?> requiredViewClass() {
        return InternalResourceView.class;
    }

    @Override
    protected AbstractUrlBasedView instantiateView() {
        return (getViewClass() == InternalResourceView.class ?
            new InternalResourceView() :
            (AbstractUrlBasedView) BeanUtils.instantiateClass(
                getViewClass()));
    }

    @Override
    protected boolean canHandle(String viewName, Locale locale) {
        // ALWAYS returns true — never returns null
        // This is why it MUST be last in the chain
        return true;
    }
}
```

**`InternalResourceView.render()` — what actually happens:**

```java
// InternalResourceView.renderMergedOutputModel()
@Override
protected void renderMergedOutputModel(
        Map<String, Object> model,
        HttpServletRequest request,
        HttpServletResponse response) throws Exception {

    // STEP 1: Expose model attributes as request attributes
    exposeModelAsRequestAttributes(model, request);
    // Each model entry: request.setAttribute(name, value)
    // This is how ${product} in JSP gets its value
    // It reads from request attributes

    // STEP 2: Expose path variables as request attributes
    exposePathVariables(request);

    // STEP 3: Expose spring macros helper (JSTL, Spring tag support)
    if (this.exposeContextBeansAsAttributes) {
        exposeHelpers(request);
    }

    // STEP 4: Determine resource to forward to
    String dispatcherPath = prepareForRendering(request, response);
    // = prefix + viewName + suffix
    // e.g., /WEB-INF/views/product/list.jsp

    // STEP 5: Get RequestDispatcher for the JSP
    RequestDispatcher rd = getRequestDispatcher(request, dispatcherPath);
    // = request.getRequestDispatcher("/WEB-INF/views/product/list.jsp")

    if (rd == null) {
        throw new ServletException(
            "Could not get RequestDispatcher for [" +
            dispatcherPath + "]");
    }

    // STEP 6: Include vs Forward decision
    if (useInclude(request, response)) {
        response.setContentType(getContentType());
        rd.include(request, response);
        // INCLUDE: outer response not committed
        // Current response writer still usable after include
    } else {
        // Standard forward — most common path
        rd.forward(request, response);
        // FORWARD: new request dispatch to JSP
        // JSP executes, writes to response
        // Response is committed after forward
    }
}
```

**Key: Model → Request Attributes → JSP EL**
```
model.put("products", productList)
        ↓
request.setAttribute("products", productList)
        ↓
${products} in JSP reads request attribute "products"
        ↓
<c:forEach items="${products}" var="p">
```

---

### RedirectView — Internal Analysis

```java
public class RedirectView extends AbstractUrlBasedView
    implements SmartView {

    private boolean contextRelative = false;
    // false: redirect URL used as-is
    // true: URL is relative to context path

    private boolean http10Compatible = true;
    // true: HTTP 302 (legacy compatibility)
    // false: HTTP 303 (POST-Redirect-GET pattern)

    private boolean exposeModelAttributes = true;
    // true: model attributes added as query params
    // false: clean redirect URL

    @Override
    protected void renderMergedOutputModel(
            Map<String, Object> model,
            HttpServletRequest request,
            HttpServletResponse response) throws IOException {

        // STEP 1: Build the target URL
        String targetUrl = createTargetUrl(model, request);
        // For "redirect:/products":
        //   targetUrl = "/products"
        // If exposeModelAttributes=true:
        //   Model attributes not in URI template
        //   → appended as query params
        //   → "redirect:/products?sortBy=name"

        // STEP 2: Save flash map BEFORE redirect
        // Flash attributes from RedirectAttributes are saved here
        RequestContextUtils.saveOutputFlashMap(
            targetUrl, request, response);
        // → SessionFlashMapManager stores flash in HttpSession
        // → Sets targetRequestPath = "/products"

        // STEP 3: Send redirect
        sendRedirect(request, response, targetUrl,
            this.http10Compatible);
    }

    protected void sendRedirect(HttpServletRequest request,
                                  HttpServletResponse response,
                                  String targetUrl,
                                  boolean http10Compatible)
            throws IOException {

        // Encode URL (session tracking if needed)
        String encodedURL = (isRemoteHost(targetUrl) ?
            targetUrl :
            response.encodeRedirectURL(targetUrl));

        if (http10Compatible) {
            // HTTP 302 Found (default)
            HttpStatus attributeStatusCode = (HttpStatus)
                request.getAttribute(
                    View.RESPONSE_STATUS_ATTRIBUTE);

            if (this.statusCode != null) {
                response.setStatus(this.statusCode.value());
                response.setHeader("Location", encodedURL);
            } else if (attributeStatusCode != null) {
                response.setStatus(attributeStatusCode.value());
                response.setHeader("Location", encodedURL);
            } else {
                // Standard 302
                response.sendRedirect(encodedURL);
            }
        } else {
            // HTTP 303 See Other (PRG pattern)
            HttpStatus statusCode = getHttp11StatusCode(
                request, response, targetUrl);
            response.setStatus(statusCode.value());
            response.setHeader("Location", encodedURL);
        }
    }
}
```

**`redirect:` prefix processing:**
```java
// UrlBasedViewResolver.createView() handles redirect: prefix
@Override
protected View createView(String viewName, Locale locale)
        throws Exception {

    // Handle redirect: prefix
    if (viewName.startsWith(REDIRECT_URL_PREFIX)) {
        // "redirect:/products"
        String redirectUrl = viewName.substring(
            REDIRECT_URL_PREFIX.length());
        // redirectUrl = "/products"

        RedirectView view = new RedirectView(redirectUrl,
            isRedirectContextRelative(),
            isRedirectHttp10Compatible());
        view.setApplicationContext(getApplicationContext());
        return applyLifecycleMethods(REDIRECT_URL_PREFIX, view);
    }

    // Handle forward: prefix
    if (viewName.startsWith(FORWARD_URL_PREFIX)) {
        String forwardUrl = viewName.substring(
            FORWARD_URL_PREFIX.length());
        InternalResourceView view =
            new InternalResourceView(forwardUrl);
        return applyLifecycleMethods(FORWARD_URL_PREFIX, view);
    }

    // Normal view name — delegate to parent
    return super.createView(viewName, locale);
}
```

---

### ContentNegotiatingViewResolver — Deep Analysis

The most architecturally interesting `ViewResolver`. It doesn't resolve views itself — it delegates to all other resolvers and selects the best match based on content negotiation:

```java
public class ContentNegotiatingViewResolver
    extends WebApplicationObjectSupport
    implements ViewResolver, Ordered, InitializingBean {

    @Nullable
    private ContentNegotiationManager contentNegotiationManager;
    // Determines requested media types from:
    // 1. URL extension (/products.json → application/json)
    // 2. Query param (/products?format=json → application/json)
    // 3. Accept header (Accept: application/json)

    @Nullable
    private List<View> defaultViews;
    // Views always included in candidate list

    @Override
    @Nullable
    public View resolveViewName(String viewName, Locale locale)
            throws Exception {

        // STEP 1: Get requested media types
        List<MediaType> requestedMediaTypes =
            getMediaTypes(
                ((ServletRequestAttributes) RequestContextHolder
                    .getRequestAttributes()).getRequest());

        if (requestedMediaTypes != null) {

            // STEP 2: Get ALL candidate views from ALL other resolvers
            List<View> candidateViews =
                getCandidateViews(viewName, locale,
                    requestedMediaTypes);

            // STEP 3: Select best view matching requested media types
            View bestView = getBestView(
                candidateViews, requestedMediaTypes,
                RequestContextHolder.getRequestAttributes());

            if (bestView != null) {
                return bestView;
            }
        }

        // STEP 4: Return default view or null
        // ...
        return null;
    }

    private List<View> getCandidateViews(
            String viewName,
            Locale locale,
            List<MediaType> requestedMediaTypes)
            throws Exception {

        List<View> candidateViews = new ArrayList<>();

        // Ask EVERY registered ViewResolver for a View
        if (this.viewResolvers != null) {
            Assert.state(this.contentNegotiationManager != null,
                "No ContentNegotiationManager set");
            for (ViewResolver viewResolver : this.viewResolvers) {
                View view =
                    viewResolver.resolveViewName(viewName, locale);
                if (view != null) {
                    candidateViews.add(view);
                }
                // Also ask for view with media-type-specific suffix
                for (MediaType requestedMediaType :
                        requestedMediaTypes) {
                    List<String> extensions =
                        this.contentNegotiationManager
                            .resolveFileExtensions(requestedMediaType);
                    for (String extension : extensions) {
                        String viewNameWithExtension =
                            viewName + '.' + extension;
                        view = viewResolver.resolveViewName(
                            viewNameWithExtension, locale);
                        if (view != null) {
                            candidateViews.add(view);
                        }
                    }
                }
            }
        }

        // Add default views
        if (!CollectionUtils.isEmpty(this.defaultViews)) {
            candidateViews.addAll(this.defaultViews);
        }

        return candidateViews;
    }

    private View getBestView(List<View> candidateViews,
            List<MediaType> requestedMediaTypes,
            RequestAttributes attrs) {

        // For each requested media type (in client preference order):
        for (MediaType mediaType : requestedMediaTypes) {
            // Find first candidate that can produce this media type
            for (View candidateView : candidateViews) {
                if (StringUtils.hasText(
                        candidateView.getContentType())) {
                    MediaType candidateContentType =
                        MediaType.parseMediaType(
                            candidateView.getContentType());
                    if (mediaType.isCompatibleWith(
                            candidateContentType)) {
                        mediaType = mediaType.removeQualityValue();
                        // Set selected content type on request
                        attrs.setAttribute(
                            View.SELECTED_CONTENT_TYPE,
                            mediaType,
                            RequestAttributes.SCOPE_REQUEST);
                        return candidateView;
                    }
                }
            }
        }
        return null;
    }
}
```

---

### View Interface — Complete Contract

```java
public interface View {

    // Request attributes set by Spring before calling render()
    String RESPONSE_STATUS_ATTRIBUTE =
        View.class.getName() + ".responseStatus";
    String PATH_VARIABLES =
        View.class.getName() + ".pathVariables";
    String SELECTED_CONTENT_TYPE =
        View.class.getName() + ".selectedContentType";

    // What Content-Type does this view produce?
    // null means View doesn't declare its content type
    @Nullable
    default String getContentType() { return null; }

    // Render the model into the response
    void render(@Nullable Map<String, ?> model,
                HttpServletRequest request,
                HttpServletResponse response) throws Exception;
}
```

---

### AbstractView — Base Template Implementation

```java
public abstract class AbstractView extends WebApplicationObjectSupport
    implements View, BeanNameAware {

    // Static attributes — set at config time, merged into every model
    private final Map<String, Object> staticAttributes =
        new LinkedHashMap<>();

    // Should path variables be exposed as model attributes?
    private boolean exposePathVariables = true;

    // Should all request attributes be available in view?
    private boolean exposeContextBeansAsAttributes = false;

    @Override
    public void render(@Nullable Map<String, ?> model,
                        HttpServletRequest request,
                        HttpServletResponse response) throws Exception {

        if (logger.isDebugEnabled()) {
            logger.debug("View " + formatViewName() +
                ", model " + (model != null ? model : Collections.emptyMap()));
        }

        // STEP 1: Build complete model
        Map<String, Object> mergedModel =
            createMergedOutputModel(model, request, response);
        // Merges: static attributes (from config)
        //       + path variables
        //       + dynamic model (from controller)
        // Dynamic attributes OVERRIDE static (same-name conflict)

        // STEP 2: Set response Content-Type
        prepareResponse(request, response);

        // STEP 3: Render (subclass-specific)
        renderMergedOutputModel(mergedModel, request, response);
    }

    protected Map<String, Object> createMergedOutputModel(
            @Nullable Map<String, ?> model,
            HttpServletRequest request,
            HttpServletResponse response) {

        // RequestContext for Spring tag library support
        @SuppressWarnings("unchecked")
        Map<String, Object> pathVars = (this.exposePathVariables ?
            (Map<String, Object>) request.getAttribute(
                View.PATH_VARIABLES) : null);

        int size = this.staticAttributes.size();
        size += (model != null ? model.size() : 0);
        size += (pathVars != null ? pathVars.size() : 0);

        Map<String, Object> mergedModel =
            CollectionUtils.newLinkedHashMap(size);

        // Merge in priority order (later overwrites earlier):
        mergedModel.putAll(this.staticAttributes); // lowest priority
        if (pathVars != null) {
            mergedModel.putAll(pathVars);           // medium priority
        }
        if (model != null) {
            mergedModel.putAll(model);              // highest priority
        }

        return mergedModel;
    }
}
```

---

### Thymeleaf Integration — How It Plugs Into the Pipeline

Thymeleaf integrates as a standard `ViewResolver`:

```
ThymeleafViewResolver (implements ViewResolver)
        │
        ▼
resolveViewName("product/list", locale)
        │
        ├── Checks if template exists: /templates/product/list.html
        ├── Creates ThymeleafView or AjaxThymeleafView
        └── Returns cached View

ThymeleafView.render(model, request, response)
        │
        ├── Gets Thymeleaf ITemplateEngine from ApplicationContext
        ├── Creates IContext with model data + locale + request/response
        ├── Processes template: evaluates th:text, th:each, etc.
        ├── Writes processed HTML to response OutputStream
        └── Sets Content-Type: text/html; charset=UTF-8
```

**No `RequestDispatcher.forward()` is used for Thymeleaf.** The template engine writes directly to the response. This is fundamentally different from JSP which requires a Servlet container forward.

---

### JSON View — MappingJackson2JsonView

Unlike `@ResponseBody` (which uses `HttpMessageConverter`), `MappingJackson2JsonView` is a proper `View` implementation used in content negotiation scenarios:

```java
public class MappingJackson2JsonView extends AbstractJackson2View {

    // Default: all model attributes serialised
    // Can configure specific attributes to include
    private Set<String> modelKeys;

    @Override
    protected Object filterModel(Map<String, Object> model) {
        Map<String, Object> result = new HashMap<>(model.size());

        Set<String> filter = extractKeysFromModel(model);

        for (Map.Entry<String, Object> entry : model.entrySet()) {
            // Exclude BindingResult objects (not serialisable)
            if (!(entry.getValue() instanceof BindingResult) &&
                    (filter == null ||
                     filter.contains(entry.getKey()))) {
                result.put(entry.getKey(), entry.getValue());
            }
        }
        return result;
    }

    @Override
    protected void renderMergedOutputModel(
            Map<String, Object> model,
            HttpServletRequest request,
            HttpServletResponse response) throws Exception {

        // Set Content-Type
        response.setContentType(getContentType());
        // "application/json"

        // Filter model to serialisable content
        Object value = filterModel(model);

        // Write JSON
        ObjectMapper objectMapper = getObjectMapper();
        JsonEncoding encoding = getJsonEncoding(getContentType());

        ObjectWriter objectWriter = objectMapper.writer();
        JsonGenerator generator = objectMapper.getFactory()
            .createGenerator(response.getOutputStream(), encoding);

        objectWriter.writeValue(generator, value);
        generator.flush();
    }
}
```

**When is `MappingJackson2JsonView` used vs `MappingJackson2HttpMessageConverter`?**

```
@ResponseBody / @RestController:
  → MessageConverter path (in HandlerAdapter, NOT ViewResolver)
  → No ViewResolver involved
  → Direct serialisation, no model wrapping

ContentNegotiatingViewResolver + MappingJackson2JsonView:
  → ViewResolver path (in DispatcherServlet.render())
  → Model is the FULL ModelMap (all attributes)
  → All serialisable model attributes included in JSON
  → Used for view-based controllers that also serve JSON
```

---

### View Rendering — Thread Safety Requirement

```
View objects are CACHED and SHARED across concurrent requests
        │
        ▼
view.render() called CONCURRENTLY by multiple threads
        │
        ▼
MUST be thread-safe:
  ✓ InternalResourceView    — safe (RequestDispatcher is per-request)
  ✓ RedirectView            — safe (no mutable state used in render)
  ✓ MappingJackson2JsonView — safe if ObjectMapper is thread-safe (it is)
  ✓ ThymeleafView           — safe (IContext is per-render, engine is shared)
  ✗ Any custom View that stores per-request state in instance fields — UNSAFE
```

---

### Locale in View Rendering

```java
// DispatcherServlet.render() — locale resolution
Locale locale = (this.localeResolver != null ?
    this.localeResolver.resolveLocale(request) :
    request.getLocale());
response.setLocale(locale);

// Locale is passed to:
// 1. ViewResolver.resolveViewName(viewName, locale)
//    → Enables locale-specific views: "products_fr", "products_de"
//    → AbstractCachingViewResolver cache key includes locale
//    → Each locale gets its own cached View

// 2. View.render() has access to locale via response.getLocale()
//    → JSP: uses locale for formatting
//    → Thymeleaf: uses locale for message resolution (#{key})
//    → MessageSource: uses locale for i18n messages
```

---

### The Complete View Class Hierarchy

```
View (interface)
    └── AbstractView
          │
          ├── AbstractUrlBasedView
          │     ├── InternalResourceView        (JSP forward)
          │     │     └── JstlView              (JSP + JSTL support)
          │     ├── RedirectView               (HTTP redirect)
          │     └── AbstractTemplateView
          │           ├── FreeMarkerView        (FreeMarker templates)
          │           └── ThymeleafView         (Thymeleaf)
          │
          ├── AbstractJackson2View
          │     └── MappingJackson2JsonView     (JSON via Jackson)
          │     └── MappingJackson2XmlView      (XML via Jackson)
          │
          └── AbstractPdfView                  (PDF generation)
              AbstractXlsView                  (Excel generation)
              MarshallingView                  (XML via Marshaller)
              ScriptTemplateView               (Mustache, etc.)
```

---

### When Is View Resolution Completely Bypassed?

```
BYPASS CONDITIONS:
────────────────────────────────────────────────────────────────
1. @ResponseBody or @RestController
   → mavContainer.requestHandled=true
   → ha.handle() returns null
   → processDispatchResult() receives null mv
   → render() NEVER called

2. ResponseEntity<T> returned
   → HttpEntityMethodProcessor writes response
   → mavContainer.requestHandled=true
   → Same bypass

3. HttpServletResponse written directly in controller
   → If response committed: Spring detects and skips rendering
   → If not committed: Spring may still attempt view rendering

4. preHandle() returns false
   → Phases 5-10 partially skipped
   → render() never called

5. Async started (Callable/DeferredResult)
   → doDispatch() returns at Phase 7
   → render() on first dispatch never called
   → render() on second dispatch (async result) may be called

6. HTTP 304 Not Modified
   → Phase 4 returns
   → render() never called

7. void return with committed response
   → mavContainer.requestHandled=true
   → Same bypass as case 1
```

---

## 2️⃣ CODE EXAMPLES

### Complete ViewResolver Chain Configuration

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {

        // 1. ContentNegotiatingViewResolver — FIRST
        // (configured separately — cannot set order via registry)
        // Added automatically when configured with enableContentNegotiation()

        // 2. BeanNameViewResolver — resolves by bean name
        registry.beanName();
        // Checks if context has a bean named = viewName

        // 3. JSP resolver — LAST (never returns null)
        registry.jsp("/WEB-INF/views/", ".jsp");
    }

    // ContentNegotiatingViewResolver must be a @Bean to control it
    @Bean
    public ContentNegotiatingViewResolver contentNegotiatingViewResolver(
            ContentNegotiationManager manager) {
        ContentNegotiatingViewResolver resolver =
            new ContentNegotiatingViewResolver();
        resolver.setContentNegotiationManager(manager);
        resolver.setOrder(Ordered.HIGHEST_PRECEDENCE);
        resolver.setDefaultViews(List.of(
            new MappingJackson2JsonView()
        ));
        return resolver;
    }

    @Override
    public void configureContentNegotiation(
            ContentNegotiationConfigurer configurer) {
        configurer
            .favorParameter(true)          // ?format=json
            .parameterName("format")
            .ignoreAcceptHeader(false)     // Also check Accept header
            .defaultContentType(MediaType.TEXT_HTML)
            .mediaType("json", MediaType.APPLICATION_JSON)
            .mediaType("xml", MediaType.APPLICATION_XML)
            .mediaType("html", MediaType.TEXT_HTML);
    }
}
```

---

### Custom View — PDF Generation

```java
// Custom view that generates PDF using iText
public class ProductListPdfView extends AbstractPdfView {

    @Override
    protected void buildPdfDocument(
            Map<String, Object> model,
            Document document,
            PdfWriter writer,
            HttpServletRequest request,
            HttpServletResponse response) throws Exception {

        @SuppressWarnings("unchecked")
        List<Product> products =
            (List<Product>) model.get("products");

        // PdfPTable for product listing
        PdfPTable table = new PdfPTable(3);
        table.addCell("ID");
        table.addCell("Name");
        table.addCell("Price");

        for (Product product : products) {
            table.addCell(String.valueOf(product.getId()));
            table.addCell(product.getName());
            table.addCell(String.valueOf(product.getPrice()));
        }

        document.add(table);
        // Response headers set automatically:
        // Content-Type: application/pdf
    }
}

// Register as a named bean — used by BeanNameViewResolver
@Bean(name = "productListPdf")
public View productListPdfView() {
    return new ProductListPdfView();
}

// Controller — returns view name
@GetMapping(value = "/products",
            produces = "application/pdf")
public String exportPdf(Model model) {
    model.addAttribute("products", productService.findAll());
    return "productListPdf";
    // → BeanNameViewResolver finds bean named "productListPdf"
}
```

---

### RedirectAttributes and Flash Map — Complete Example

```java
@Controller
@RequestMapping("/checkout")
public class CheckoutController {

    @PostMapping("/complete")
    public String completeCheckout(
            @Valid @ModelAttribute Order order,
            BindingResult result,
            RedirectAttributes redirectAttrs,
            HttpServletRequest request) {

        if (result.hasErrors()) {
            return "checkout/form";
        }

        Order saved = orderService.place(order);

        // Flash attribute — stored in session before redirect
        // Available as model attribute on next request
        redirectAttrs.addFlashAttribute("successOrder", saved);
        redirectAttrs.addFlashAttribute("message",
            "Order placed successfully!");

        // Regular redirect attribute — becomes query param
        // /checkout/confirm?orderId=42
        redirectAttrs.addAttribute("orderId", saved.getId());

        // RedirectView will:
        // 1. Call RequestContextUtils.saveOutputFlashMap()
        //    → SessionFlashMapManager stores flash in session
        // 2. Send HTTP 302 to /checkout/confirm?orderId=42
        return "redirect:/checkout/confirm/{orderId}";
    }

    @GetMapping("/confirm/{orderId}")
    public String confirm(
            @PathVariable Long orderId,
            // Flash attributes automatically available as @ModelAttribute
            @ModelAttribute("successOrder") Order order,
            Model model) {
        // Flash map contents automatically merged into Model
        // by RequestMappingHandlerAdapter.invokeHandlerMethod()
        // (mavContainer.addAllAttributes(flashMap))
        model.addAttribute("orderId", orderId);
        return "checkout/confirm";
    }
}
```

---

### Thymeleaf Configuration — ViewResolver Setup

```java
// Spring Boot auto-configures this — manual for plain Spring MVC
@Configuration
public class ThymeleafConfig {

    @Bean
    public SpringTemplateEngine templateEngine(
            SpringResourceTemplateResolver resolver) {
        SpringTemplateEngine engine = new SpringTemplateEngine();
        engine.setTemplateResolver(resolver);
        engine.setEnableSpringELCompiler(true);
        return engine;
    }

    @Bean
    public SpringResourceTemplateResolver templateResolver() {
        SpringResourceTemplateResolver resolver =
            new SpringResourceTemplateResolver();
        resolver.setApplicationContext(applicationContext);
        resolver.setPrefix("classpath:/templates/");
        resolver.setSuffix(".html");
        resolver.setTemplateMode(TemplateMode.HTML);
        // DISABLE caching in development:
        resolver.setCacheable(false); // true in production
        resolver.setCharacterEncoding("UTF-8");
        return resolver;
    }

    @Bean
    public ThymeleafViewResolver thymeleafViewResolver(
            SpringTemplateEngine engine) {
        ThymeleafViewResolver resolver =
            new ThymeleafViewResolver();
        resolver.setTemplateEngine(engine);
        resolver.setOrder(1); // Before InternalResourceViewResolver
        resolver.setCharacterEncoding("UTF-8");
        // Thymeleaf can return null for unknown view names
        // (unlike InternalResourceViewResolver which never returns null)
        return resolver;
    }
}
```

---

### Edge Case — Static Attributes vs Model Attributes

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public InternalResourceViewResolver viewResolver() {
        InternalResourceViewResolver resolver =
            new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");

        // Static attributes — same for ALL requests
        // Merged into every model before rendering
        // Lower priority than model attributes (model wins on conflict)
        Map<String, Object> staticAttrs = new HashMap<>();
        staticAttrs.put("appName", "My Application");
        staticAttrs.put("appVersion", "2.0.1");
        staticAttrs.put("supportEmail", "help@example.com");
        resolver.setAttributesMap(staticAttrs);

        return resolver;
    }
}

// In JSP: ${appName} always available regardless of controller
// In JSP: ${product} from model attribute — per-request

// If controller does:
// model.addAttribute("appName", "Override") → controller value wins
// because model is merged AFTER static attributes
```

---

### Edge Case — ViewResolver Order and "Never Returns Null"

```java
// WRONG: InternalResourceViewResolver before Thymeleaf
@Override
public void configureViewResolvers(ViewResolverRegistry registry) {
    // WRONG ORDER:
    registry.jsp("/WEB-INF/views/", ".jsp");  // NEVER returns null
    registry.enableContentNegotiation();
    // Thymeleaf resolver never consulted — JSP resolver wins always
}

// CORRECT: InternalResourceViewResolver last
@Override
public void configureViewResolvers(ViewResolverRegistry registry) {
    registry.enableContentNegotiation();      // First (delegates to all)
    registry.beanName();                       // Second
    // Thymeleaf auto-added if on classpath  // Third
    registry.jsp("/WEB-INF/views/", ".jsp");  // LAST
}

// Even more explicit:
@Bean
public ViewResolver internalResourceViewResolver() {
    InternalResourceViewResolver r =
        new InternalResourceViewResolver();
    r.setPrefix("/WEB-INF/views/");
    r.setSuffix(".jsp");
    r.setOrder(Integer.MAX_VALUE); // EXPLICIT: last
    return r;
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
`AbstractCachingViewResolver` uses a concurrent cache of `View` objects. What is the implication for custom `View` implementations that are cached and reused?

A) Custom `View.render()` must be declared `synchronized`
B) Custom `View.render()` must be thread-safe — it is called concurrently by multiple request threads sharing the same `View` instance
C) `AbstractCachingViewResolver` creates a new `View` per request to avoid concurrency issues
D) `View` objects are stored in ThreadLocal — no thread-safety requirement

**Answer: B**
The cache stores ONE `View` instance per `viewName + locale` key. Multiple concurrent requests for the same view share the SAME `View` instance. `render()` is called concurrently. Custom views must not store per-request state in instance fields. All per-request data must come from method parameters (`model`, `request`, `response`).

---

**Q2 — Select All That Apply**
`InternalResourceViewResolver` has which of these characteristics?

A) Returns `null` when the JSP file does not exist
B) Always returns a non-null `View` for any view name
C) Must be the LAST resolver in the chain
D) Uses `RequestDispatcher.forward()` to render JSPs
E) Caches `View` objects — one per view name + locale

**Answer: B, C, D, E**
A is wrong — `InternalResourceViewResolver` always returns non-null regardless of JSP existence. The 404 only happens at render time when `forward()` fails. B, C (consequence of B), D (forward mechanism), E (AbstractCachingViewResolver inheritance) are all correct.

---

**Q3 — Ordering**
Order these operations in `InternalResourceView.renderMergedOutputModel()`:

- `RequestDispatcher.forward()` called
- Model attributes set as request attributes via `request.setAttribute()`
- Content-Type set on response
- `RequestDispatcher` obtained from request
- Path variables exposed as request attributes

**Correct Order:**
1. Model attributes set as request attributes via `request.setAttribute()`
2. Path variables exposed as request attributes
3. Content-Type set on response
4. `RequestDispatcher` obtained from request
5. `RequestDispatcher.forward()` called

---

**Q4 — MCQ**
`ContentNegotiatingViewResolver` has `order=Integer.MIN_VALUE` (highest priority). A request for `/products` arrives with `Accept: application/json`. `ContentNegotiatingViewResolver` collects candidates from all other resolvers. Which `View` does it select?

A) The first `View` from the first resolver that returned non-null
B) The `View` whose `getContentType()` is compatible with `application/json`
C) The `InternalResourceView` because it is always in the candidate list
D) `null` — content negotiation fails and falls through to `InternalResourceViewResolver`

**Answer: B**
`getBestView()` iterates requested media types (from Accept header) and finds the first candidate `View` whose `getContentType()` is compatible. `MappingJackson2JsonView` returns `"application/json"` from `getContentType()`. It is selected. `InternalResourceView` returns `null` from `getContentType()` → not selected for JSON.

---

**Q5 — True/False**
When a controller returns `"redirect:/products"`, the `RedirectView` saves flash attributes to the session BEFORE sending the HTTP 302 response.

**Answer: True**
`RedirectView.renderMergedOutputModel()` calls `RequestContextUtils.saveOutputFlashMap()` (Step 2) BEFORE `sendRedirect()` (Step 3). Flash attributes must be saved to the session before the redirect is sent, because after the redirect the client makes a new request and the flash map must already be in the session when that request arrives.

---

**Q6 — MCQ**
A developer configures both `ThymeleafViewResolver` (order=1) and `InternalResourceViewResolver` (order=2). The controller returns view name `"dashboard"`. There is no Thymeleaf template `dashboard.html` but there IS a JSP `/WEB-INF/views/dashboard.jsp`. What happens?

A) `ThymeleafViewResolver` throws `TemplateInputException` and the request fails
B) `ThymeleafViewResolver` returns `null` — `InternalResourceViewResolver` resolves it — JSP rendered
C) `InternalResourceViewResolver` is never consulted — `ThymeleafViewResolver` always wins
D) Both resolvers attempt rendering simultaneously — conflict error

**Answer: B**
Unlike `InternalResourceViewResolver`, `ThymeleafViewResolver` CAN return `null` when the template does not exist. It checks for template existence before returning a View. If not found → returns `null` → chain continues to `InternalResourceViewResolver` → finds JSP → renders it. This is correct chaining behaviour.

---

**Q7 — Code Prediction**
```java
@Controller
public class DemoController {

    @GetMapping("/demo")
    public void demo(HttpServletResponse response)
            throws IOException {
        response.getWriter().write("Hello");
        // No flush(), no commit
    }
}
```
What view is rendered after the method returns?

A) No view — `void` return means response handled
B) View name `"demo"` derived from URL by `DefaultRequestToViewNameTranslator`
C) HTTP 500 — cannot render view after writing to response
D) `InternalResourceView` for `/WEB-INF/views/demo.jsp`

**Answer: B and D are linked — D is the actual result**
`void` return → `mavContainer.requestHandled` is NOT set to true (no `@ResponseBody`). `RequestResponseBodyMethodProcessor` is not involved. Spring creates a `ModelAndView` with no view name. `applyDefaultViewName()` → `DefaultRequestToViewNameTranslator` → `"demo"`. `InternalResourceViewResolver` resolves `"demo"` → `/WEB-INF/views/demo.jsp`. `RequestDispatcher.forward()` called. But the response writer was already used → `IllegalStateException` if using `forward()` after `getWriter()` with content. Answer: **B**, which leads to a runtime exception during rendering. The correct answer is B (view name derived) which then likely causes a rendering error.

---

**Q8 — Select All That Apply**
Which of the following completely bypass the `ViewResolver` chain?

A) `@ResponseBody` on controller method
B) `ResponseEntity<T>` return type
C) `ModelAndView` return type with view name `"product/list"`
D) Controller method writing directly to `HttpServletResponse.getOutputStream()`
E) `"redirect:/products"` returned from controller
F) `void` return with uncommitted response

**Answer: A, B**
C uses `ViewResolver` (resolves `"product/list"`). D may bypass rendering if response is committed but the chain is still attempted. E uses `RedirectView` which IS a `View` — goes through `ViewResolver` chain (but `UrlBasedViewResolver` intercepts the `redirect:` prefix before resolver chain). F uses `ViewResolver` to derive view from URL. A and B are the clean bypasses via `mavContainer.requestHandled=true`.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — InternalResourceViewResolver returning null**
It NEVER does. The most important fact about this resolver. Every exam question involving multiple resolvers hinges on this. If it's not last, ALL views get resolved to JSP — Thymeleaf, JSON views, bean-name views are all bypassed. The rule: InternalResourceViewResolver must be `order=Integer.MAX_VALUE` or the last entry in `configureViewResolvers()`.

**Trap 2 — View caching and thread safety**
`View` objects are singletons — one per `viewName + locale`. They are called by multiple request threads simultaneously. A custom `View` that stores per-request state (current user, request data) in instance fields has a race condition. This is identical to the controller singleton thread-safety trap but for views. The fix: only use method parameters in `render()`.

**Trap 3 — redirect: flash attributes timing**
`saveOutputFlashMap()` runs BEFORE `sendRedirect()`. This is the correct order — flash must be in session before client makes the next request. If you're debugging and see flash attributes sometimes missing, the issue is NOT timing — it's usually the target URL not matching (flash map has `targetRequestPath`) or session expiration.

**Trap 4 — void return does NOT mean response handled**
`void` return from a controller method does NOT set `mavContainer.requestHandled=true` unless `@ResponseBody` is on the method/class. A `void` method in a `@Controller` (not `@RestController`) with uncommitted response → Spring tries to render a view derived from the URL. This catches developers who write `response.getWriter().write("data")` thinking that stops view rendering.

**Trap 5 — ContentNegotiatingViewResolver and the resolver list**
`ContentNegotiatingViewResolver` collects candidates from ALL other resolvers. It asks them for views. But it does NOT use the standard resolver chain order. It asks ALL resolvers, collects ALL views, then picks the best based on content type. Setting order=HIGHEST_PRECEDENCE on `ContentNegotiatingViewResolver` means it runs first — but it DELEGATES internally to all others. If all others return null, CNVR returns null too and the chain continues.

**Trap 6 — Model attributes merged priority**
In `AbstractView.createMergedOutputModel()`, the merge order is: static attributes (lowest) → path variables → dynamic model (highest). A controller's `model.addAttribute("appName", "X")` overrides a static attribute named `"appName"` set on the resolver. This is intentional — dynamic data wins. Questions ask which value wins when same key exists in both.

**Trap 7 — MappingJackson2JsonView vs @ResponseBody path**
Both produce JSON but via completely different pipelines. `@ResponseBody` uses `HttpMessageConverter` in `RequestMappingHandlerAdapter` — completely bypasses ViewResolver. `MappingJackson2JsonView` IS a View — goes through the ViewResolver chain, serialises the entire model map (not just the return value). The serialised content is different: `@ResponseBody` serialises only the return value; `MappingJackson2JsonView` serialises ALL model attributes as a JSON object.

---

## 5️⃣ SUMMARY SHEET

```
VIEW RESOLUTION CHAIN
─────────────────────────────────────────────────────
DispatcherServlet.render()
  → resolveLocale() → set on response
  → resolveViewName(name, locale) — chain iteration
    → First non-null View wins
  → view.render(model, request, response)

RESOLVER CHAIN RULES
─────────────────────────────────────────────────────
Returns null:     "I can't resolve this — try next"
Returns View:     "Use this — chain stops"
Throws:           Resolution error — propagates up

InternalResourceViewResolver: NEVER returns null
  → MUST be last in chain (Integer.MAX_VALUE order)

ThymeleafViewResolver: returns null if template missing
BeanNameViewResolver: returns null if no bean with that name
ContentNegotiatingViewResolver: delegates to ALL resolvers,
  selects best by content negotiation, MUST be first

ABSTRACTCACHINGVIEWRESOLVER CACHING
─────────────────────────────────────────────────────
Cache key: viewName + locale
One View instance per key → thread-safe render() required
setCacheLimit(0): disables caching (dev mode)
View objects are SHARED across concurrent requests

INTERNALRESOURCEVIEW RENDERING
─────────────────────────────────────────────────────
1. exposeModelAsRequestAttributes() → request.setAttribute() per model entry
2. exposePathVariables() → path vars as request attributes
3. prepareResponse() → Content-Type set
4. getRequestDispatcher(path) → path = prefix + name + suffix
5. rd.forward(request, response) → JSP executes
→ JSP EL reads from request attributes
→ ${product} = request.getAttribute("product")

REDIRECTVIEW
─────────────────────────────────────────────────────
createTargetUrl() → builds redirect URL
  + exposeModelAttributes=true → non-path-variable attrs as query params
saveOutputFlashMap() → BEFORE redirect (flash in session first)
sendRedirect():
  http10Compatible=true  → HTTP 302 Found (default)
  http10Compatible=false → HTTP 303 See Other

redirect: prefix → intercepted by UrlBasedViewResolver
  → creates RedirectView (never goes to InternalResourceViewResolver)

ABSTRACTVIEW.createMergedOutputModel() PRIORITY
─────────────────────────────────────────────────────
1. staticAttributes (config time — lowest)
2. path variables
3. dynamic model attributes (controller — HIGHEST, wins on conflict)

VIEW BYPASS CONDITIONS (ViewResolver never called)
─────────────────────────────────────────────────────
@ResponseBody / @RestController    → mavContainer.requestHandled=true
ResponseEntity<T>                  → mavContainer.requestHandled=true
preHandle() returned false         → doDispatch() returns early
Async started                      → doDispatch() returns early
HTTP 304 Not Modified              → doDispatch() returns early

void return WITHOUT @ResponseBody → ViewResolver IS called
  → DefaultRequestToViewNameTranslator derives view name from URL

JSON via @ResponseBody:     MessageConverter path (no ViewResolver)
JSON via MappingJackson2JsonView: ViewResolver path (full model serialised)

CONTENT NEGOTIATION VIEW SELECTION
─────────────────────────────────────────────────────
ContentNegotiatingViewResolver:
  1. Determine requested media types (extension → param → Accept header)
  2. Ask ALL resolvers for candidate Views
  3. Select View whose getContentType() matches requested media type
  4. InternalResourceView.getContentType() = null (not considered for JSON)
  5. MappingJackson2JsonView.getContentType() = "application/json" ✓

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "InternalResourceViewResolver never returns null — always last in chain"
• "View objects are cached and shared — render() must be thread-safe"
• "Flash attributes saved to session BEFORE HTTP 302 is sent"
• "void controller return → ViewResolver still called (no requestHandled=true)"
• "@ResponseBody bypasses ViewResolver entirely — MessageConverter path"
• "MappingJackson2JsonView serialises full model; @ResponseBody serialises return value only"
• "ContentNegotiatingViewResolver delegates to all resolvers — not a leaf resolver"
• "Model attributes override static attributes (same key) — dynamic wins"
• "JSP reads model via request.setAttribute() — not directly from model map"
• "ThymeleafViewResolver can return null — safe to place before InternalResourceViewResolver"
```

---

**PART 2 — DISPATCHERSERVLET INTERNALS COMPLETE** ✅

```
2.1 DispatcherServlet Lifecycle: init() to destroy()           ✅
2.2 WebApplicationContext Bootstrapping                        ✅
2.3 Special Beans in DispatcherServlet                         ✅
2.4 HandlerMapping: Internal Resolution Chain                  ✅
2.5 HandlerAdapter: Execution Delegation                       ✅
2.6 doDispatch(): The Core Request Processing Loop             ✅
2.7 View Resolution and Rendering Pipeline                     ✅
```


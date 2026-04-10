# TOPIC 4.4 — STATIC RESOURCES, CACHING, AND RESOURCE HANDLING

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What Static Resource Handling Is in Spring MVC

Static resource handling is the mechanism by which Spring MVC serves files — CSS, JavaScript, images, fonts, HTML — from the classpath, filesystem, or other locations. It is NOT simply "Spring passes files through." It involves a complete pipeline: `ResourceHttpRequestHandler`, a `ResourceResolver` chain, a `ResourceTransformer` chain, content negotiation, HTTP caching headers, and version-based cache busting. Understanding this pipeline at depth reveals significant performance engineering capability.

---

### The Static Resource Pipeline Architecture

```
HTTP Request: GET /static/css/app.css
        │
        ▼
DispatcherServlet
        │
        ▼
RequestMappingHandlerMapping → null (no @RequestMapping for this URL)
        │
        ▼
SimpleUrlHandlerMapping (order=Integer.MAX_VALUE-1)
  → Maps /static/** → ResourceHttpRequestHandler
        │
        ▼
HttpRequestHandlerAdapter.handle()
        │
        ▼
ResourceHttpRequestHandler.handleRequest()
        │
        ├── Phase 1: Check HTTP method (GET or HEAD only)
        ├── Phase 2: ResourceResolver chain → Resource
        ├── Phase 3: Check Resource exists and is readable
        ├── Phase 4: MediaType determination
        ├── Phase 5: HTTP caching headers (Last-Modified, ETag)
        ├── Phase 6: Check If-Modified-Since / If-None-Match
        │     → 304 Not Modified if resource unchanged
        ├── Phase 7: ResourceTransformer chain
        ├── Phase 8: Write response headers (Content-Type, Content-Length)
        └── Phase 9: Write resource content to response
```

---

### addResourceHandlers() — Configuration API

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(
            ResourceHandlerRegistry registry) {

        // BASIC REGISTRATION
        registry.addResourceHandler("/static/**")
                // URL patterns that trigger this handler
                .addResourceLocations(
                    "classpath:/static/",
                    "classpath:/public/",
                    "/WEB-INF/static/",
                    "file:/var/www/resources/"
                    // Multiple locations searched in order
                )
                .setCachePeriod(3600)
                // Sets Cache-Control: max-age=3600
                // Shorthand — use setCacheControl() for full control

                .setCacheControl(
                    CacheControl.maxAge(Duration.ofDays(365))
                        .cachePublic()
                        .immutable()
                    // For versioned resources that never change
                )

                .setUseLastModified(true)
                // Serve Last-Modified header
                // Enables conditional GET (304 responses)

                .resourceChain(true)
                // Enable ResourceResolver/Transformer chain
                // true = caching enabled in chain
                // false = no caching (dev mode)
                .addResolver(new VersionResourceResolver()
                    .addContentVersionStrategy("/**"))
                .addTransformer(
                    new CssLinkResourceTransformer());
    }
}
```

---

### ResourceHttpRequestHandler — Internal Processing

```java
// ResourceHttpRequestHandler.handleRequest()
@Override
public void handleRequest(HttpServletRequest request,
                           HttpServletResponse response)
        throws ServletException, IOException {

    // STEP 1: Check method
    checkRequest(request);
    // Only GET and HEAD allowed
    // HEAD: same processing, but no body written

    // STEP 2: Resolve resource using resolver chain
    Resource resource = getResource(request);

    if (resource == null) {
        logger.debug("Resource not found");
        response.sendError(HttpServletResponse.SC_NOT_FOUND);
        return;
    }

    // STEP 3: Check for resource factory
    if (HttpMethod.OPTIONS.matches(
            request.getMethod())) {
        response.setHeader("Allow", "GET,HEAD,OPTIONS");
        return;
    }

    // STEP 4: Determine media type for the resource
    MediaType mediaType = getMediaType(request, resource);
    if (mediaType != null) {
        setHeaders(response, resource, mediaType);
    }

    // STEP 5: Handle HTTP caching headers
    if (isUseLastModified() &&
        new ServletWebRequest(request, response)
            .checkNotModified(resource.lastModified())) {
        // Returns HTTP 304 Not Modified
        // Entire pipeline short-circuits here
        logger.trace("Resource not modified");
        return;
    }

    // STEP 6: Set response headers
    setHeaders(response, resource, mediaType);

    // STEP 7: Write content
    // HEAD requests: only headers, no body
    if (HttpMethod.HEAD.matches(request.getMethod())) {
        logger.trace("HEAD request - no content");
        return;
    }

    // Apply ResourceTransformers
    // Then copy resource content to response
    ServletUtils.checkAndPrepareResponse(request, response);
    if (request.getHeader(
            HttpHeaders.RANGE) == null) {
        // Standard full-content response
        writeContent(response, resource);
    } else {
        // Partial content (range request)
        writePartialContent(response, resource);
    }
}
```

---

### ResourceResolver Chain — Finding Resources

The `ResourceResolver` chain allows Spring to resolve abstract resource names to actual files. Each resolver in the chain is tried in order:

```java
// ResourceResolver interface
public interface ResourceResolver {

    // Resolve the handler-relative request path to a Resource
    @Nullable
    Resource resolveResource(
        @Nullable HttpServletRequest request,
        String requestPath,
        List<? extends Resource> locations,
        ResourceResolverChain chain);

    // Resolve the URL for a given resource path
    // (used for URL generation in templates)
    @Nullable
    String resolveUrlPath(
        String resourcePath,
        List<? extends Resource> locations,
        ResourceResolverChain chain);
}
```

**Default resolver chain:**

```
1. CachingResourceResolver (wraps the rest for caching)
   └── cache.get(key) → found? return cached
       not found → delegate to next resolver
       put result in cache

2. PathResourceResolver (the actual file finder)
   → For each location:
       location.createRelative(requestPath) → Resource
       resource.exists() && resource.isReadable()? → return it
   → None found → return null → 404
```

**Available resolvers:**

```java
// PathResourceResolver (default — finds actual files)
new PathResourceResolver()
// Maps URL path to filesystem/classpath path

// VersionResourceResolver (adds versioning)
new VersionResourceResolver()
    .addContentVersionStrategy("/**")
    // Content hash: /static/css/app-a3b4c5d6.css
    // Hash computed from file content
    // URL changes when content changes → cache busting
    .addFixedVersionStrategy("v1.2.0", "/js/**")
    // Fixed version: /static/js/v1.2.0/app.js
    // Changes when version string changes

// EncodedResourceResolver (pre-compressed resources)
new EncodedResourceResolver()
// Serves .gz or .br files when client accepts gzip/brotli
// Client sends Accept-Encoding: gzip
// Handler checks for app.css.gz → serves compressed version
// Content-Encoding: gzip header set

// WebJarsResourceResolver (for WebJars)
new WebJarsResourceResolver()
// Maps /webjars/bootstrap/4.5.0/css/bootstrap.min.css
// to the actual file in the WebJar JAR file
// Also supports version-less paths:
// /webjars/bootstrap/css/bootstrap.min.css
// → resolves to current version in WebJar
```

---

### VersionResourceResolver — Cache Busting Deep Analysis

```java
// Content-based versioning — automatic cache busting
@Override
public void addResourceHandlers(
        ResourceHandlerRegistry registry) {
    registry.addResourceHandler("/static/**")
            .addResourceLocations("classpath:/static/")
            .resourceChain(true)
            .addResolver(new VersionResourceResolver()
                .addContentVersionStrategy("/**"));
}

// How it works:

// STARTUP: No pre-computation of hashes
// (hashes computed on first access)

// FIRST REQUEST: GET /static/css/app.css
// → PathResourceResolver finds /static/css/app.css
// → Returns Resource (no version in URL)

// WITH VERSIONED URL: GET /static/css/app-a1b2c3d4.css
// VersionResourceResolver processes:
// 1. Extracts version from URL: "a1b2c3d4"
// 2. Derives actual file: "app.css"
// 3. Loads app.css from PathResourceResolver
// 4. Computes MD5/SHA of content: "a1b2c3d4..."
// 5. Verifies extracted version matches computed hash
// 6. Returns resource

// URL GENERATION (for templates):
// ResourceUrlProvider.getForLookupPath("/static/css/app.css")
// → Computes hash of content
// → Returns "/static/css/app-a1b2c3d4.css"
// Used in Thymeleaf: th:src="@{/static/css/app.css}"
// → Auto-versioned: /static/css/app-a1b2c3d4.css

// Cache-Control headers:
// With versioned URL: Cache-Control: max-age=31536000, immutable
// Content hash = unique ID = can cache forever safely
// When content changes → hash changes → browsers fetch new URL
```

---

### HTTP Caching — Complete Header Management

```java
// ResourceHttpRequestHandler sets these caching headers:

// OPTION 1: setCachePeriod(seconds)
registry.addResourceHandler("/static/**")
        .setCachePeriod(3600); // 1 hour

// Sets: Cache-Control: max-age=3600
// This is HTTP/1.1 caching

// OPTION 2: setCacheControl(CacheControl)
// Full control over all caching headers
registry.addResourceHandler("/static/**")
        .setCacheControl(
            CacheControl.maxAge(Duration.ofDays(365))
                .cachePublic()
                .immutable()
        );
// → Cache-Control: max-age=31536000, public, immutable

// OPTION 3: No caching
registry.addResourceHandler("/api/**")
        .setCacheControl(CacheControl.noStore());
// → Cache-Control: no-store

// OPTION 4: Must revalidate
registry.addResourceHandler("/templates/**")
        .setCacheControl(
            CacheControl.maxAge(Duration.ofHours(1))
                .mustRevalidate()
        );
// → Cache-Control: max-age=3600, must-revalidate

// CacheControl builder patterns:
CacheControl.empty()          // No Cache-Control header
CacheControl.noCache()        // must-revalidate before using cache
CacheControl.noStore()        // don't cache at all
CacheControl.maxAge(Duration) // explicit max age
    .cachePublic()            // any cache may store
    .cachePrivate()           // only browser cache
    .mustRevalidate()         // stale must revalidate
    .immutable()              // content never changes (max-age permanent)
    .proxyRevalidate()        // proxy caches must revalidate
    .sMaxAge(Duration)        // proxy-specific max age
```

---

### Last-Modified and ETag — Conditional GET

```java
// ResourceHttpRequestHandler implements conditional GET:

// FIRST REQUEST:
// GET /static/css/app.css
// Response headers:
//   Last-Modified: Mon, 04 Mar 2024 10:00:00 GMT
//   Cache-Control: max-age=3600
//   Content-Type: text/css
// Response body: [full CSS content]

// SUBSEQUENT REQUEST (within cache period):
// Browser uses cached version — no HTTP request made at all

// AFTER CACHE EXPIRES (after 3600 seconds):
// Browser sends conditional GET:
// GET /static/css/app.css
// Request headers:
//   If-Modified-Since: Mon, 04 Mar 2024 10:00:00 GMT

// Handler processes:
// resource.lastModified() = 1709550000000 (file not changed)
// request.getHeader("If-Modified-Since") = Mon, 04 Mar 2024 10:00:00 GMT
// lastModified <= ifModifiedSince → NOT MODIFIED
// response.setStatus(304)
// NO BODY WRITTEN
// Saves bandwidth and server processing

// If file HAS changed:
// resource.lastModified() = 1709640000000 (newer)
// 1709640000000 > parsed(If-Modified-Since)
// → 200 OK with new content
```

---

### Content-Type Determination for Resources

```java
// MediaType determined from file extension
private MediaType getMediaType(HttpServletRequest request,
                                 Resource resource) {

    MediaType mediaType = null;

    // Check for explicitly configured media types
    String filename = resource.getFilename();

    // 1. From MediaTypeFactory (mime.types based)
    List<MediaType> mediaTypes =
        MediaTypeFactory.getMediaTypes(filename)
            .orElse(null);
    if (mediaTypes != null && !mediaTypes.isEmpty()) {
        mediaType = mediaTypes.get(0);
    }

    // 2. From content negotiation strategy
    // (for cases where extension isn't obvious)

    return mediaType;
}

// MediaTypeFactory uses:
// 1. Spring's built-in mime.types
// 2. Servlet container's mime types
// 3. Custom registrations

// Common mappings:
// .css   → text/css
// .js    → text/javascript (or application/javascript)
// .html  → text/html
// .json  → application/json
// .png   → image/png
// .jpg   → image/jpeg
// .svg   → image/svg+xml
// .woff2 → font/woff2
// .pdf   → application/pdf
```

---

### ResourceTransformer Chain

```java
// Transforms resource content AFTER resolving but BEFORE serving

public interface ResourceTransformer {
    Resource transform(
        HttpServletRequest request,
        Resource resource,
        ResourceTransformerChain transformerChain
    ) throws IOException;
}

// Built-in transformers:

// 1. CssLinkResourceTransformer
// Rewrites CSS @import and url() references to versioned URLs
// Input:  @import "base.css"; .icon { background: url("img/icon.png"); }
// Output: @import "base-a1b2c3d4.css"; .icon { background: url("img/icon-5e6f7a8b.css"); }
// Important: versioned URLs in CSS allow deep cache busting

// 2. AppCacheManifestTransformer (deprecated)
// Updated HTML5 AppCache manifests with versioned URLs
// Deprecated: AppCache is deprecated in HTML5

// Custom transformer:
public class MinifyResourceTransformer
    implements ResourceTransformer {

    @Override
    public Resource transform(
            HttpServletRequest request,
            Resource resource,
            ResourceTransformerChain chain)
            throws IOException {

        // Apply previous transformer in chain
        resource = chain.transform(request, resource);

        String filename = resource.getFilename();

        if (filename != null &&
                filename.endsWith(".js") &&
                !filename.endsWith(".min.js")) {
            // Minify JavaScript
            String content = new String(
                FileCopyUtils.copyToByteArray(
                    resource.getInputStream()),
                StandardCharsets.UTF_8);
            String minified = minifier.minify(content);
            return new TransformedResource(resource,
                minified.getBytes(StandardCharsets.UTF_8));
        }

        return resource;
    }
}
```

---

### ResourceUrlProvider — URL Generation in Templates

```java
// ResourceUrlProvider computes versioned URLs for templates
// Without versioned URLs in templates:
//   <link href="/static/css/app.css"/>
//   → always fetches same URL even after content changes

// With ResourceUrlProvider:
//   <link th:href="@{/static/css/app.css}"/>
//   → Thymeleaf calls ResourceUrlProvider.getForLookupPath()
//   → Computes content hash
//   → Returns /static/css/app-a1b2c3d4.css
//   → Browser caches forever (immutable)
//   → When app.css changes → hash changes → new URL → browser fetches

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(
            ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static/")
                .setCacheControl(
                    CacheControl.maxAge(Duration.ofDays(365))
                        .immutable())
                .resourceChain(true)
                .addResolver(new VersionResourceResolver()
                    .addContentVersionStrategy("/**"));
    }
}

// Spring Boot auto-registers ResourceUrlEncodingFilter
// which intercepts response URL encoding to version static URLs
// Works with Spring MVC's resource URL encoding

// In Thymeleaf: @{...} expressions are automatically versioned
// In JSP: use spring:url or ResourceUrlProvider directly
// In FreeMarker: use ${rc.getContextPath() + ...} with manual versioning
```

---

### Default Static Resource Locations — Spring Boot

```java
// Spring Boot auto-configures static resource handling
// Default locations (in priority order):
// 1. classpath:/META-INF/resources/
// 2. classpath:/resources/
// 3. classpath:/static/
// 4. classpath:/public/

// Mapped to: /**

// Customisation in application.properties:
// spring.web.resources.static-locations=classpath:/static/,file:/var/www/
// spring.web.resources.chain.strategy.content.enabled=true
// spring.web.resources.chain.strategy.content.paths=/**
// spring.web.resources.cache.period=86400
// spring.web.resources.cache.cachecontrol.max-age=365d
// spring.web.resources.cache.cachecontrol.cache-public=true

// Disable default handler:
// spring.web.resources.add-mappings=false
// (Use when all routing is in @Controller methods)
```

---

### WebJars — Third-Party Static Resources

```java
// WebJars package JavaScript/CSS libraries as JAR files
// Maven dependency example:
// <dependency>
//   <groupId>org.webjars</groupId>
//   <artifactId>bootstrap</artifactId>
//   <version>5.3.0</version>
// </dependency>

// Files inside bootstrap.jar:
// META-INF/resources/webjars/bootstrap/5.3.0/css/bootstrap.min.css
// META-INF/resources/webjars/bootstrap/5.3.0/js/bootstrap.bundle.min.js

@Override
public void addResourceHandlers(
        ResourceHandlerRegistry registry) {

    // Access via versioned URL
    registry.addResourceHandler("/webjars/**")
            .addResourceLocations(
                "classpath:/META-INF/resources/webjars/")
            .resourceChain(false)  // No caching in dev
            .addResolver(new WebJarsResourceResolver());
            // Resolves version-less URLs:
            // /webjars/bootstrap/css/bootstrap.min.css
            // → bootstrap/5.3.0/css/bootstrap.min.css

}

// Template usage:
// <link th:href="@{/webjars/bootstrap/css/bootstrap.min.css}"/>
// WebJarsResourceResolver adds the version automatically
// No need to hardcode version in template
```

---

### Default Servlet Handler — Fallback for Unmatched Requests

```java
// When DispatcherServlet is mapped to /
// It intercepts ALL requests including static files
// Default servlet handler forwards unmatched to container's default servlet
// (Tomcat's DefaultServlet serves files from webapp directory)

@Override
public void configureDefaultServletHandling(
        DefaultServletHandlerConfigurer configurer) {
    configurer.enable();
    // OR:
    configurer.enable("default"); // Specify servlet name if different
}

// Processing flow with default servlet handler:
// GET /favicon.ico
// → RequestMappingHandlerMapping: null (no @RequestMapping)
// → SimpleUrlHandlerMapping (resource handler): null
//   (if no /favicon.ico registration)
// → SimpleUrlHandlerMapping (default servlet): MATCH /**
// → DefaultServletHttpRequestHandler.handleRequest()
//   → RequestDispatcher("default").forward(request, response)
//   → Tomcat's DefaultServlet handles it
//   → Serves from webapp root directory

// Alternative: Register favicon explicitly
@Override
public void addResourceHandlers(
        ResourceHandlerRegistry registry) {
    registry.addResourceHandler("/favicon.ico")
            .addResourceLocations("classpath:/static/");
}
```

---

### Pre-Compressed Resources — EncodedResourceResolver

```java
// Serve pre-compressed resources to reduce bandwidth
// Instead of compressing on-the-fly, serve pre-compressed files

// File structure:
// classpath:/static/css/app.css      (original)
// classpath:/static/css/app.css.gz   (gzip compressed)
// classpath:/static/css/app.css.br   (brotli compressed)

@Override
public void addResourceHandlers(
        ResourceHandlerRegistry registry) {
    registry.addResourceHandler("/static/**")
            .addResourceLocations("classpath:/static/")
            .resourceChain(true)
            .addResolver(new EncodedResourceResolver())
            // Checks Accept-Encoding header
            // If "br": looks for resource.br → serves it
            // If "gzip": looks for resource.gz → serves it
            // Adds Content-Encoding: gzip/br header
            .addResolver(new PathResourceResolver());
            // Fallback: serve original uncompressed
}

// Request headers:
// Accept-Encoding: br, gzip, deflate

// EncodedResourceResolver:
// 1. Checks Accept-Encoding
// 2. Tries: app.css.br → found? Return with Content-Encoding: br
// 3. Tries: app.css.gz → found? Return with Content-Encoding: gzip
// 4. No compressed version → fall through to PathResourceResolver
// 5. Serves: app.css (uncompressed)

// Client decompresses automatically based on Content-Encoding header
```

---

## 2️⃣ CODE EXAMPLES

### Production-Grade Static Resource Configuration

```java
@Configuration
@EnableWebMvc
@Slf4j
public class ResourceConfig implements WebMvcConfigurer {

    @Autowired
    private Environment env;

    @Override
    public void addResourceHandlers(
            ResourceHandlerRegistry registry) {

        boolean isDev = env.acceptsProfiles(
            Profiles.of("development"));

        // ── VERSIONED ASSETS (CSS, JS) ────────────────────────────
        registry.addResourceHandler("/assets/**")
                .addResourceLocations(
                    "classpath:/static/assets/")
                .setCacheControl(
                    isDev ?
                    CacheControl.noCache() :
                    CacheControl.maxAge(
                        Duration.ofDays(365)).immutable()
                )
                .resourceChain(!isDev)
                // Caching disabled in dev for instant updates
                .addResolver(
                    new EncodedResourceResolver())
                // Pre-compressed first
                .addResolver(
                    new VersionResourceResolver()
                    .addContentVersionStrategy("/**"))
                // Content hash versioning
                .addResolver(
                    new PathResourceResolver())
                // Fallback: actual file
                .addTransformer(
                    new CssLinkResourceTransformer());
                // Rewrite CSS URLs to versioned

        // ── IMAGES (long cache, no versioning) ───────────────────
        registry.addResourceHandler("/images/**")
                .addResourceLocations(
                    "classpath:/static/images/",
                    "file:${user.home}/uploads/images/")
                // Multiple locations: classpath first, then filesystem
                .setCacheControl(
                    CacheControl.maxAge(
                        Duration.ofDays(30)).cachePublic())
                .setUseLastModified(true);
                // Enable 304 conditional GET

        // ── WEBJARS ──────────────────────────────────────────────
        registry.addResourceHandler("/webjars/**")
                .addResourceLocations(
                    "classpath:/META-INF/resources/webjars/")
                .setCacheControl(
                    CacheControl.maxAge(
                        Duration.ofDays(365)).immutable())
                .resourceChain(true)
                .addResolver(new WebJarsResourceResolver())
                .addResolver(new PathResourceResolver());

        // ── API DOCUMENTATION (no caching) ───────────────────────
        registry.addResourceHandler("/api-docs/**")
                .addResourceLocations(
                    "classpath:/static/api-docs/")
                .setCacheControl(CacheControl.noStore());
    }

    // ResourceUrlProvider bean — enables versioned URL generation
    @Bean
    public ResourceUrlProvider resourceUrlProvider() {
        ResourceUrlProvider provider =
            new ResourceUrlProvider();
        // Automatically picks up registered resource handlers
        return provider;
    }
}
```

---

### Custom ResourceResolver — Database-Backed Resources

```java
// Serve files stored in database
@Component
public class DatabaseResourceResolver
    implements ResourceResolver {

    @Autowired
    private FileStorageService fileStorageService;

    private static final String DB_PREFIX = "db:";

    @Override
    @Nullable
    public Resource resolveResource(
            @Nullable HttpServletRequest request,
            String requestPath,
            List<? extends Resource> locations,
            ResourceResolverChain chain) {

        // Check if this resolver handles this path
        if (!requestPath.startsWith(DB_PREFIX)) {
            // Not our pattern — delegate to chain
            return chain.resolveResource(
                request, requestPath, locations);
        }

        String fileId = requestPath.substring(
            DB_PREFIX.length());
        byte[] content =
            fileStorageService.getContent(fileId);

        if (content == null) {
            return null; // Not found
        }

        // Return in-memory Resource
        return new ByteArrayResource(content) {
            @Override
            public String getFilename() {
                return fileStorageService
                    .getFilename(fileId);
            }

            @Override
            public long lastModified() {
                return fileStorageService
                    .getLastModified(fileId);
            }
        };
    }

    @Override
    @Nullable
    public String resolveUrlPath(
            String resourcePath,
            List<? extends Resource> locations,
            ResourceResolverChain chain) {
        // URL path resolution for template URL generation
        return chain.resolveUrlPath(
            resourcePath, locations);
    }
}
```

---

### Range Requests — Partial Content Support

```java
// ResourceHttpRequestHandler automatically handles Range requests
// for streaming media (video, audio)

// Client sends:
// Range: bytes=0-1023
// → Response: 206 Partial Content
//   Content-Range: bytes 0-1023/102400
//   Accept-Ranges: bytes

// Range: bytes=1024-
// → Bytes from 1024 to end

// Multiple ranges:
// Range: bytes=0-499, 1000-1499
// → multipart/byteranges response

// Spring's ResourceRegionHttpMessageConverter handles this:
// → Reads only requested byte ranges
// → Streams range to response
// → Supports seek/resume for video players

// Controller serving video with range support:
@GetMapping("/media/{id}")
public ResponseEntity<Resource> streamVideo(
        @PathVariable Long id,
        @RequestHeader(value="Range", required=false)
            String rangeHeader) {

    Resource video = mediaService.getVideoResource(id);

    if (!video.exists()) {
        return ResponseEntity.notFound().build();
    }

    if (rangeHeader != null) {
        // ResourceHttpRequestHandler handles this automatically
        // for /static/** mappings, but for controller-served
        // resources, use ResourceRegion
        return ResponseEntity.status(
                HttpStatus.PARTIAL_CONTENT)
            .contentType(MediaType.parseMediaType("video/mp4"))
            .header("Accept-Ranges", "bytes")
            .body(video);
    }

    return ResponseEntity.ok()
        .contentType(MediaType.parseMediaType("video/mp4"))
        .header("Accept-Ranges", "bytes")
        .body(video);
}
```

---

### ResourceUrlEncodingFilter — Automatic URL Versioning

```java
// Spring Boot auto-configures this when resource chain enabled
// For plain Spring MVC:
@Bean
public ResourceUrlEncodingFilter resourceUrlEncodingFilter() {
    return new ResourceUrlEncodingFilter();
    // Wraps response to intercept response.encodeURL()
    // When URL is a static resource → adds version
}

// In web.xml:
// <filter>
//   <filter-name>resourceUrlEncodingFilter</filter-name>
//   <filter-class>
//     org.springframework.web.servlet.resource.ResourceUrlEncodingFilter
//   </filter-class>
// </filter>
// <filter-mapping>
//   <filter-name>resourceUrlEncodingFilter</filter-name>
//   <url-pattern>/*</url-pattern>
// </filter-mapping>

// In JSP templates:
// <spring:url value="/static/css/app.css" var="cssUrl"/>
// <link href="${cssUrl}"/>
// → /static/css/app-a1b2c3d4.css (versioned)

// In Thymeleaf (works automatically):
// <link th:href="@{/static/css/app.css}"/>
// → /static/css/app-a1b2c3d4.css
```

---

### Security Headers for Static Resources

```java
@Override
public void addResourceHandlers(
        ResourceHandlerRegistry registry) {

    // Custom transformer that adds security headers
    registry.addResourceHandler("/static/**")
            .addResourceLocations("classpath:/static/")
            .resourceChain(true)
            .addTransformer(
                new SecurityHeadersTransformer());
}

// ResourceTransformer is NOT for HTTP headers
// Use a Filter or interceptor for security headers

// Better: WebContentInterceptor for headers
@Override
public void addInterceptors(
        InterceptorRegistry registry) {
    WebContentInterceptor wci = new WebContentInterceptor();

    // Static resources: aggressive caching
    wci.addCacheMapping(
        CacheControl.maxAge(Duration.ofDays(365))
            .immutable().cachePublic(),
        "/static/**", "/webjars/**");

    // API: no caching
    wci.addCacheMapping(
        CacheControl.noStore(),
        "/api/**");

    registry.addInterceptor(wci);
}
```

---

### Edge Case — Resource Not Found vs 404 Page

```java
@Override
public void addResourceHandlers(
        ResourceHandlerRegistry registry) {
    registry.addResourceHandler("/static/**")
            .addResourceLocations("classpath:/static/");
}

// What happens when resource NOT found?
// GET /static/css/nonexistent.css
// → ResourceHttpRequestHandler.handleRequest()
// → getResource() → PathResourceResolver finds nothing → returns null
// → response.sendError(HttpServletResponse.SC_NOT_FOUND)
// → CONTAINER handles 404 error page

// IMPORTANT: This bypasses Spring's @ExceptionHandler mechanism
// The 404 from static resource NOT FOUND goes directly to container

// Container 404 handling:
// 1. web.xml <error-page> for 404 → shows custom page
// 2. Spring Boot ErrorController → /error endpoint

// If you need custom 404 for missing static resources:
// → Use a Servlet Filter to intercept 404 responses
// → OR configure throwExceptionIfNoHandlerFound
//    but this is for URL mapping misses, not resource misses

// For static resources specifically:
// ResourceHttpRequestHandler ALWAYS calls sendError(404)
// No way to convert this to a Spring exception via normal config
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
`ResourceHttpRequestHandler` has `setCacheControl(CacheControl.maxAge(Duration.ofDays(365)).immutable())`. A browser requests `GET /static/css/app-a1b2c3d4.css`. A second browser requests the same URL one week later. What happens on the second request?

A) The server sends the file with a fresh 200 response — `immutable` means always fresh
B) The browser never makes an HTTP request — it uses its cached copy for 365 days
C) The browser sends a conditional GET with `If-Modified-Since`
D) `Cache-Control: immutable` is ignored — browser always validates

**Answer: B**
`immutable` in `Cache-Control` tells browsers: "this URL's content will NEVER change — don't even bother revalidating during the max-age period." With `max-age=31536000, immutable`, the browser serves from cache for 365 days WITHOUT making any HTTP request. This is only safe with content-hashed URLs — if content changes, the URL hash changes, forcing a new request.

---

**Q2 — Select All That Apply**
`VersionResourceResolver` with `addContentVersionStrategy("/**")` is configured. Which of the following are TRUE?

A) File content hashes are computed at application startup
B) File content hashes are computed on first access, then cached
C) Versioned URLs (`app-a1b2c3d4.css`) can be cached aggressively with `immutable`
D) The version strategy changes the resource URL in templates automatically
E) When file content changes, the hash changes, forcing browsers to fetch the new URL

**Answer: B, C, E**
A is wrong — hashes are NOT computed at startup. They're computed on first access and cached. B is correct. C is correct — content hash = unique fingerprint = safe to cache immutably. D is partially true — it requires `ResourceUrlProvider` integration in templates (Thymeleaf `@{...}` handles this automatically). E is the core cache-busting mechanism.

---

**Q3 — MCQ**
`EncodedResourceResolver` is before `PathResourceResolver` in the chain. A request arrives with `Accept-Encoding: gzip`. File `app.css.gz` exists alongside `app.css`. What response is served?

A) `app.css` — `EncodedResourceResolver` only compresses on-the-fly
B) `app.css.gz` with `Content-Encoding: gzip` header
C) `app.css.gz` with `Content-Type: application/gzip` header
D) Error — compressed resources must be served by a separate endpoint

**Answer: B**
`EncodedResourceResolver` checks `Accept-Encoding` header. Finds `gzip` acceptable. Looks for `app.css.gz` → found. Returns it as a `Resource`. Sets `Content-Encoding: gzip` header (NOT `Content-Type: application/gzip` — the content type remains `text/css`). The browser decompresses transparently. `Content-Encoding` indicates the encoding of the body; `Content-Type` indicates the original media type.

---

**Q4 — Select All That Apply**
`ResourceHttpRequestHandler.handleRequest()` checks `resource.lastModified()` and the `If-Modified-Since` request header. Which behaviours are TRUE?

A) If not modified → HTTP 304 with no response body
B) If modified → HTTP 200 with new content AND new `Last-Modified` header
C) If `setUseLastModified(false)` → never sends `Last-Modified` header → browser always receives 200
D) The Last-Modified comparison uses millisecond precision
E) HTTP 304 response includes `Cache-Control` headers from the configuration

**Answer: A, B, C, E**
D is wrong — HTTP `Last-Modified` uses second-level precision (the format `Mon, 04 Mar 2024 10:00:00 GMT` has no sub-second). Spring truncates to seconds for comparison. E is correct — HTTP 304 responses MUST include the same caching headers as the original 200 (per HTTP spec), so `Cache-Control` is still sent with 304.

---

**Q5 — True/False**
`CssLinkResourceTransformer` modifies the CSS file content on disk.

**Answer: False**
`CssLinkResourceTransformer` is a `ResourceTransformer` that processes CSS content IN MEMORY at request time. It rewrites `@import` and `url()` references to point to versioned URLs. The original file on disk is NEVER modified. The transformation produces a new `TransformedResource` with modified content that is served to the browser. Optionally, if `CachingResourceResolver` is in the chain, the transformed result can be cached in memory.

---

**Q6 — MCQ**
`addResourceHandlers()` registers `/webjars/**` with `WebJarsResourceResolver`. Template references `/webjars/bootstrap/css/bootstrap.min.css` (no version in URL). How does `WebJarsResourceResolver` find the file?

A) Searches all JAR files on classpath for matching file
B) Reads WebJar version from `pom.xml` at runtime
C) Uses WebJar's `pom.properties` inside the JAR to determine the current version, then constructs the full path
D) Requires version in URL — throws exception if absent

**Answer: C**
`WebJarsResourceResolver` reads `META-INF/maven/.../pom.properties` from the WebJar JAR file to determine the current version. It then constructs the full versioned path: `bootstrap/5.3.0/css/bootstrap.min.css`. The template can use version-less URLs — the resolver inserts the correct version automatically. This means you only update the Maven dependency version, and templates automatically resolve correctly.

---

**Q7 — MCQ**
`resourceChain(true)` vs `resourceChain(false)` — what does the boolean parameter control?

A) Whether to enable the resource resolver chain at all
B) Whether the `CachingResourceResolver` wraps the chain (caching resolved resources in memory)
C) Whether `ResourceTransformer` instances are applied
D) Whether to use URL versioning

**Answer: B**
`resourceChain(boolean cacheResources)` — when `true`, wraps the resolver chain with `CachingResourceResolver`, which caches resolved `Resource` objects in a `ConcurrentHashMap`. First access walks the full chain; subsequent accesses return the cached `Resource` directly. `false` means no memory caching — full chain walked every request. Use `false` in development (see changes immediately), `true` in production.

---

**Q8 — Scenario**
A Spring Boot application has `spring.web.resources.chain.strategy.content.enabled=true`. A Thymeleaf template references `th:src="@{/css/app.css}"`. The `app.css` file has content hash `a1b2c3d4`. What URL appears in the rendered HTML?

A) `/css/app.css` — Thymeleaf doesn't version URLs
B) `/css/app.css?v=a1b2c3d4` — query parameter versioning
C) `/css/app-a1b2c3d4.css` — content-hashed URL
D) `/css/a1b2c3d4/app.css` — version as path prefix

**Answer: C**
`VersionResourceResolver` with `ContentVersionStrategy` embeds the hash in the filename: `app-{hash}.css`. Thymeleaf's `@{...}` expression calls `ResourceUrlProvider.getForLookupPath("/css/app.css")` which computes the hash and returns the versioned path `/css/app-a1b2c3d4.css`. This appears in the rendered HTML, enabling long-term caching.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — `immutable` is only safe with content-hashed URLs**
`Cache-Control: immutable` tells browsers "never revalidate this URL during max-age." If applied to non-versioned URLs like `/static/css/app.css`, content changes are invisible to cached browsers until max-age expires. Only use `immutable` with content-hashed or version-stamped URLs where changing content always changes the URL.

**Trap 2 — `resourceChain(false)` disables caching, not the chain itself**
`resourceChain(false)` means NO `CachingResourceResolver` wraps the chain — every request walks resolvers from scratch. It does NOT disable resolvers or transformers. The parameter name is misleading — it controls whether resolved resources are cached in memory, not whether the chain exists.

**Trap 3 — Static resource 404 bypasses `@ExceptionHandler`**
When `ResourceHttpRequestHandler` cannot find a file, it calls `response.sendError(404)` directly. This bypasses Spring's exception resolver chain entirely. `@ExceptionHandler(NoHandlerFoundException.class)` does NOT fire for missing static resources. The 404 goes straight to the Servlet container. Custom 404 pages for static resources must be configured in `web.xml` or Spring Boot's error handling.

**Trap 4 — `CssLinkResourceTransformer` requires `VersionResourceResolver` first**
`CssLinkResourceTransformer` rewrites CSS URLs to versioned versions. For this to work, `VersionResourceResolver` must be in the chain BEFORE it, because the transformer needs to know what versioned URL to generate. If `VersionResourceResolver` is absent, `CssLinkResourceTransformer` cannot produce versioned URLs and silently leaves URLs unchanged.

**Trap 5 — `Last-Modified` uses second-level precision**
`resource.lastModified()` returns milliseconds. HTTP `Last-Modified`/`If-Modified-Since` headers use second-level precision. Spring truncates to seconds for comparison. Files modified within the same second as the cached value may incorrectly return 304. This is an HTTP specification limitation, not a Spring bug.

**Trap 6 — Multiple locations searched in order — first match wins**
`addResourceLocations("classpath:/static/", "classpath:/public/")` — when both locations have the same filename, `classpath:/static/` wins (searched first). This is important when providing overlay or override resources. Higher-priority overrides should be listed FIRST.

**Trap 7 — `setCachePeriod()` vs `setCacheControl()` — different header generation**
`setCachePeriod(seconds)` generates `Cache-Control: max-age=N`. `setCacheControl()` gives full control: `public`, `private`, `must-revalidate`, `immutable`, etc. `setCachePeriod()` is a convenience shorthand. For production, use `setCacheControl()` with explicit settings. Mixing both on the same handler: `setCacheControl()` takes precedence.

---

## 5️⃣ SUMMARY SHEET

```
STATIC RESOURCE PIPELINE
─────────────────────────────────────────────────────
Request → SimpleUrlHandlerMapping → ResourceHttpRequestHandler
  1. Check HTTP method (GET/HEAD only)
  2. ResourceResolver chain → Resource
  3. Resource null → sendError(404) [bypasses @ExceptionHandler]
  4. Determine MediaType from file extension
  5. Check If-Modified-Since → 304 if not modified
  6. Set response headers (Content-Type, Cache-Control, Last-Modified)
  7. ResourceTransformer chain
  8. Write content to response (or 304 no-body)

RESOURCE RESOLVER CHAIN (typical order)
─────────────────────────────────────────────────────
CachingResourceResolver    → memory cache of resolved resources
EncodedResourceResolver    → serves .gz/.br for Accept-Encoding
VersionResourceResolver    → handles versioned URLs (hash/fixed)
WebJarsResourceResolver    → version-less WebJar URLs
PathResourceResolver       → actual file lookup (must be last)

resourceChain(true)  → CachingResourceResolver wraps chain
resourceChain(false) → no memory caching, full chain per request

VERSIONRESOURCERESOLVER
─────────────────────────────────────────────────────
ContentVersionStrategy:  app-{hash}.css
  hash = MD5/SHA of file content
  hash computed on first access, cached
  safe for immutable caching

FixedVersionStrategy:    /js/{version}/app.js
  version string in URL
  changes when version string changes
  manual cache busting

CACHE CONTROL PATTERNS
─────────────────────────────────────────────────────
Versioned assets (never change):
  CacheControl.maxAge(365, DAYS).immutable().cachePublic()
  → Cache-Control: max-age=31536000, public, immutable

Standard assets (revalidate):
  CacheControl.maxAge(1, HOURS).cachePublic()
  → Cache-Control: max-age=3600, public

User-specific:
  CacheControl.maxAge(10, MINUTES).cachePrivate()
  → Cache-Control: max-age=600, private

No caching:
  CacheControl.noStore()
  → Cache-Control: no-store

CONDITIONAL GET (Last-Modified)
─────────────────────────────────────────────────────
setUseLastModified(true): sends Last-Modified header
Second request with If-Modified-Since:
  resource.lastModified() <= ifModifiedSince → 304 (no body)
  resource.lastModified() > ifModifiedSince  → 200 (new content)
Precision: SECONDS (HTTP spec limitation)
304 response: includes Cache-Control headers

RESOURCE TRANSFORMER CHAIN
─────────────────────────────────────────────────────
CssLinkResourceTransformer:
  Rewrites CSS @import and url() to versioned URLs
  Requires VersionResourceResolver in chain
  Works IN MEMORY — file on disk not modified

ENCODEDRESOURCERESOLVER
─────────────────────────────────────────────────────
Accept-Encoding: gzip → looks for file.gz
Accept-Encoding: br   → looks for file.br
Serves compressed file with Content-Encoding: gzip/br
Content-Type: unchanged (text/css, not application/gzip)

SPRING BOOT DEFAULTS
─────────────────────────────────────────────────────
Default locations (priority order):
  classpath:/META-INF/resources/
  classpath:/resources/
  classpath:/static/
  classpath:/public/
Mapped to: /**
spring.web.resources.chain.strategy.content.enabled=true
  → Enables VersionResourceResolver globally

WEBJARS
─────────────────────────────────────────────────────
Version-less URL: /webjars/bootstrap/css/bootstrap.min.css
WebJarsResourceResolver reads version from pom.properties in JAR
Constructs: bootstrap/5.3.0/css/bootstrap.min.css

DEFAULT SERVLET HANDLER
─────────────────────────────────────────────────────
configureDefaultServletHandling(configurer).enable()
Fallback for URLs not matched by resource handlers
Forwards to container's default servlet (Tomcat's DefaultServlet)
Lowest priority — tried last

IMPORTANT: Static resource 404 → sendError(404) bypasses Spring
No @ExceptionHandler for missing static resources
Configure via container error pages or Spring Boot ErrorController

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "immutable is only safe with versioned URLs — never with /static/css/app.css"
• "resourceChain(false) disables memory caching, NOT the resolver chain"
• "Static 404 → sendError() → bypasses @ExceptionHandler → container handles"
• "CssLinkResourceTransformer requires VersionResourceResolver in chain before it"
• "Last-Modified comparison uses second-level precision (HTTP spec)"
• "EncodedResourceResolver serves .gz/.br with Content-Encoding, not Content-Type change"
• "ResourceHttpRequestHandler supports Range requests for video/audio streaming"
• "setCacheControl() takes precedence over setCachePeriod() on same handler"
• "Multiple locations: first match wins — put highest-priority location first"
• "WebJarsResourceResolver reads version from pom.properties in JAR — no version in URL"
```

---

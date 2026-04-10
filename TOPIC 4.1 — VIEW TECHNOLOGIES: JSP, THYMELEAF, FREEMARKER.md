# TOPIC 4.1 — VIEW TECHNOLOGIES: JSP, THYMELEAF, FREEMARKER

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What View Technologies Are in Spring MVC's Architecture

A view technology is a **rendering engine** that transforms a model (Java objects) into a response (HTML, XML, PDF, etc.). Spring MVC is deliberately view-technology agnostic — the `View` interface is the only contract. Any technology that implements `View.render(model, request, response)` integrates into the pipeline. Understanding what each technology does internally, how it integrates with Spring's `ViewResolver` chain, and the precise differences in rendering mechanics separates surface knowledge from mastery.

---

### The View Rendering Contract

```java
// Every view technology implements this:
public interface View {
    void render(
        @Nullable Map<String, ?> model,
        HttpServletRequest request,
        HttpServletResponse response
    ) throws Exception;
}

// Three phases in AbstractView.render():
// 1. createMergedOutputModel() — merge static + dynamic + path vars
// 2. prepareResponse() — set Content-Type, charset
// 3. renderMergedOutputModel() — TECHNOLOGY-SPECIFIC rendering
```

**The key difference between view technologies is in `renderMergedOutputModel()`:**

```
JSP:          rd.forward(request, response) → Servlet container processes JSP
Thymeleaf:    templateEngine.process(template, context, writer) → pure Java
FreeMarker:   template.process(model, writer) → pure Java
Mustache:     mustache.execute(model, writer) → pure Java
JSON View:    objectMapper.writeValue(outputStream, filteredModel) → Jackson
```

---

### JSP — Java Server Pages Deep Analysis

#### What JSP Is

JSP is NOT a Spring technology. It is a **Servlet specification feature** — JSP files are compiled by the Servlet container (Tomcat's Jasper compiler) into `HttpServlet` classes. Spring MVC simply forwards the request to the JSP. The container handles the rest.

#### InternalResourceViewResolver + InternalResourceView Flow

```
Controller returns "product/list"
        │
        ▼
InternalResourceViewResolver resolves
  prefix = "/WEB-INF/views/"
  suffix = ".jsp"
  → creates InternalResourceView("/WEB-INF/views/product/list.jsp")
        │
        ▼
InternalResourceView.renderMergedOutputModel()
  Step 1: exposeModelAsRequestAttributes(model, request)
    → for each model entry:
         request.setAttribute("products", productList)
         request.setAttribute("count", 42)
    → All model data now in request scope
    → JSP EL reads request attributes: ${products}

  Step 2: getRequestDispatcher("/WEB-INF/views/product/list.jsp")
    → request.getRequestDispatcher(path)
    → returns RequestDispatcher from Servlet container

  Step 3: rd.forward(request, response)
    → NEW request dispatch within container
    → Tomcat's Jasper compiler:
        If first request: compiles product_list_jsp.java → .class
        If already compiled: uses cached .class
        If source changed: recompiles
    → Compiled JSP servlet executes
    → Writes HTML to response output stream
```

#### JSP Internal Model Access

```java
// How JSP accesses model data:
// Model attribute → request.setAttribute() → JSP EL resolves

// In controller:
model.addAttribute("product", product);  // Product POJO
model.addAttribute("user", user);        // User POJO

// In JSP (EL resolution order):
// 1. pageScope
// 2. requestScope   ← Spring puts model here via setAttribute()
// 3. sessionScope
// 4. applicationScope

${product.name}     // → requestScope["product"].getName()
${user.email}       // → requestScope["user"].getEmail()

// JSTL tags:
<c:forEach items="${products}" var="p">
    ${p.name} - ${p.price}
</c:forEach>

<c:if test="${not empty errorMessage}">
    <div class="error">${errorMessage}</div>
</c:if>

<c:url value="/products">
    <c:param name="page" value="2"/>
</c:url>
// → /contextPath/products?page=2

// Spring Form tags (require spring-webmvc JAR):
<%@ taglib prefix="form" uri="http://www.springframework.org/tags/form" %>
<form:form modelAttribute="product" action="/products" method="post">
    <form:input path="name"/>
    <form:errors path="name" cssClass="error"/>
    <form:select path="categoryId" items="${categories}"
                 itemValue="id" itemLabel="name"/>
</form:form>
```

#### JstlView — JSTL Enhancement

```java
// InternalResourceViewResolver with JSTL:
// If JSTL (jstl.jar) is on classpath AND viewClass not specified:
// InternalResourceViewResolver automatically uses JstlView

// JstlView extends InternalResourceView, adds:
// - Spring MessageSource exposed as JSTL LocalizationContext
// - So fmt:message tags use Spring's MessageSource
// - Locale from LocaleContextHolder exposed to JSTL

// Without JstlView: fmt:message uses JSTL's own resource bundle
// With JstlView: fmt:message uses Spring's MessageSource
// Spring's MessageSource is more powerful (supports properties, DB, etc.)
```

#### JSP Security — /WEB-INF/ Protection

```
/WEB-INF/ directory is NOT directly accessible from browser
  → Browser: GET /WEB-INF/views/product/list.jsp → HTTP 404
  → Only accessible via server-side forward/include

This is WHY Spring puts JSPs in /WEB-INF/views/
  → Security: raw JSPs cannot be accessed directly
  → Only through Spring MVC controller → ViewResolver → forward
  → Without going through controller: no model data → NullPointerExceptions
```

---

### Thymeleaf — Template Engine Deep Analysis

#### Architecture Overview

```
Thymeleaf is NOT a Servlet technology — it's a pure Java template engine
Key components:
  SpringTemplateEngine (processes templates)
  SpringResourceTemplateResolver (locates templates)
  ThymeleafViewResolver (Spring MVC integration)
  SpringWebMvcThymeleafRequestContext (request context)

Thymeleaf does NOT use RequestDispatcher.forward()
→ Templates are processed in Java, HTML written directly to response
→ No Servlet container involvement in template rendering
→ Works in non-web contexts too (email templates, etc.)
```

#### Template Resolution and Processing

```java
// ThymeleafViewResolver.resolveViewName()
@Override
@Nullable
protected View loadView(String viewName, Locale locale)
        throws Exception {

    // Determine template name
    String templateName = viewName;
    // e.g., "product/list" → classpath:templates/product/list.html

    // Check if template exists
    ITemplateEngine templateEngine = getTemplateEngine();

    // Create ThymeleafView
    AbstractThymeleafView view = ...;
    view.setTemplateName(templateName);
    view.setTemplateEngine(templateEngine);
    view.setLocale(locale);

    return view;
}

// AbstractThymeleafView.render()
@Override
public void render(Map<String, ?> model,
                    HttpServletRequest request,
                    HttpServletResponse response) throws Exception {

    // Set Content-Type: text/html; charset=UTF-8
    response.setContentType(getContentType());

    // Build Thymeleaf context (wraps model + request + locale)
    IContext context = buildThymeleafContext(model, request,
        response, locale);
    // Context contains:
    // - All model attributes accessible as ${variable}
    // - Request: accessible as ${#request}
    // - Session: accessible as ${#session}
    // - Servlets: accessible as ${#servletContext}
    // - Spring beans: accessible as ${@beanName.method()}

    // Process template
    templateEngine.process(
        getTemplateName(),
        context,
        response.getWriter()
    );
    // → Reads HTML template file
    // → Processes th:* attributes
    // → Writes processed HTML to response writer
}
```

#### Thymeleaf Template Syntax — Spring MVC Integration

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org"
      xmlns:sec="http://www.thymeleaf.org/extras/spring-security">
<head>
    <!-- th:replace — replace this element with fragment -->
    <th:block th:replace="~{common/head :: head}"></th:block>
</head>
<body>

<!-- MODEL ACCESS -->
<!-- Simple variable expression -->
<p th:text="${product.name}">Default Name</p>

<!-- URL expression -->
<a th:href="@{/products/{id}(id=${product.id})}">View</a>
<!-- → /products/42 (with context path prefixed) -->

<!-- Message expression (i18n from MessageSource) -->
<p th:text="#{product.price.label}">Price:</p>

<!-- Iteration -->
<tr th:each="product : ${products}">
    <td th:text="${product.id}"></td>
    <td th:text="${product.name}"></td>
    <td th:text="${#numbers.formatDecimal(product.price,1,2)}"></td>
</tr>

<!-- Conditional -->
<div th:if="${not #lists.isEmpty(products)}">
    <!-- products exist -->
</div>
<div th:unless="${product.available}">Out of stock</div>
<td th:switch="${product.status}">
    <span th:case="'ACTIVE'">Active</span>
    <span th:case="'DISCONTINUED'">Discontinued</span>
    <span th:case="*">Unknown</span>
</td>

<!-- SPRING MVC FORM INTEGRATION -->
<form th:action="@{/products}" th:object="${productForm}"
      method="post">
    <!-- th:field generates id, name, value attributes -->
    <input th:field="*{name}" type="text"/>
    <!-- → <input id="name" name="name" value="..."/> -->

    <!-- Error display -->
    <span th:if="${#fields.hasErrors('name')}"
          th:errors="*{name}">Name error</span>

    <select th:field="*{categoryId}">
        <option th:each="cat : ${categories}"
                th:value="${cat.id}"
                th:text="${cat.name}">Category</option>
    </select>

    <button type="submit">Submit</button>
</form>

<!-- SPRING SECURITY INTEGRATION -->
<div sec:authorize="hasRole('ADMIN')">
    Admin content here
</div>
<span sec:authentication="principal.username">
    Username
</span>

<!-- SPRING BEANS -->
<p th:text="${@productService.count()}">0 products</p>

<!-- FRAGMENTS -->
<!-- Definition (in common/fragments.html): -->
<div th:fragment="productCard(product)">
    <h3 th:text="${product.name}"></h3>
</div>
<!-- Usage: -->
<div th:replace="~{common/fragments :: productCard(${product})}">
</div>

</body>
</html>
```

#### Thymeleaf Caching Configuration

```java
// SpringResourceTemplateResolver configuration
@Bean
public SpringResourceTemplateResolver templateResolver() {
    SpringResourceTemplateResolver resolver =
        new SpringResourceTemplateResolver();
    resolver.setApplicationContext(applicationContext);
    resolver.setPrefix("classpath:/templates/");
    resolver.setSuffix(".html");
    resolver.setTemplateMode(TemplateMode.HTML);
    resolver.setCharacterEncoding("UTF-8");

    // CACHING:
    // Production: cacheable=true (default)
    //   Template parsed ONCE, cached in memory
    //   Changes require application restart
    resolver.setCacheable(true);
    // Development: cacheable=false
    //   Template re-read every request
    //   Changes reflect immediately (no restart needed)
    resolver.setCacheable(false);
    resolver.setCacheTTLMs(3600000L); // 1 hour TTL if cacheable

    return resolver;
}
```

#### Thymeleaf vs JSP — Key Differences

```
THYMELEAF                        JSP
─────────────────────────────────────────────────────────────
Pure Java rendering              Compiled to Servlet by container
No server required for preview   Cannot preview without server
Natural HTML templates           Tags mixed with HTML
Works offline in browser         Requires Servlet container
Spring Boot default              Legacy default
Errors shown in browser          Errors hidden in WEB-INF
No Servlet API dependency        Servlet API required
Per-request template processing  Per-request Servlet execution
Returns null for missing files   Always returns View (never null)
XML-based syntax (th:*)          JSP tag library syntax (<%@ %>)
```

---

### FreeMarker — Template Engine Integration

#### Architecture

```
FreeMarker is another pure-Java template engine
Similar to Thymeleaf but different syntax
Spring integration via FreeMarkerViewResolver and FreeMarkerView
```

```java
// Configuration
@Bean
public FreeMarkerConfigurer freeMarkerConfigurer() {
    FreeMarkerConfigurer configurer =
        new FreeMarkerConfigurer();
    configurer.setTemplateLoaderPath(
        "classpath:/templates/freemarker/");
    configurer.setDefaultEncoding("UTF-8");

    // FreeMarker settings
    Properties settings = new Properties();
    settings.put("template_exception_handler", "rethrow");
    settings.put("default_encoding", "UTF-8");
    settings.put("number_format", "computer");
    // Avoids locale-specific number formatting issues
    configurer.setFreemarkerSettings(settings);

    return configurer;
}

@Bean
public FreeMarkerViewResolver freeMarkerViewResolver() {
    FreeMarkerViewResolver resolver =
        new FreeMarkerViewResolver();
    resolver.setPrefix("");
    resolver.setSuffix(".ftlh");  // FTL HTML extension
    resolver.setContentType("text/html;charset=UTF-8");
    resolver.setRequestContextAttribute("rc");
    // Exposes Spring's RequestContext as "rc" in templates
    resolver.setOrder(1);
    return resolver;
}
```

#### FreeMarker Template Syntax

```html
<!DOCTYPE html>
<!-- FreeMarker template (.ftlh) -->
<html>
<body>

<!-- VARIABLE OUTPUT -->
<p>${product.name}</p>
<p>${product.price?string("0.00")}</p>

<!-- CONDITIONAL -->
<#if product.available>
    <span>In Stock</span>
<#else>
    <span>Out of Stock</span>
</#if>

<!-- ITERATION -->
<#list products as product>
    <tr>
        <td>${product.id}</td>
        <td>${product.name?html}</td>
        <!-- ?html escapes HTML special characters -->
    </tr>
<#else>
    <tr><td colspan="2">No products found</td></tr>
</#list>

<!-- NULL HANDLING -->
<p>${product.description!("No description")}</p>
<!-- ! provides default if null -->
<p>${product.description??}</p>
<!-- ?? returns boolean: true if not null -->

<!-- INCLUDE -->
<#include "/templates/common/header.ftlh">

<!-- SPRING INTEGRATION -->
<!-- Access Spring MessageSource via rc (RequestContext) -->
<p>${rc.getMessage("product.name.label")}</p>

<!-- Spring form macros -->
<#import "/spring.ftl" as spring>
<form action="${rc.getContextPath()}/products" method="post">
    <@spring.formInput "product.name"/>
    <@spring.showErrors "<br>", "error"/>
    <@spring.formSingleSelect "product.categoryId", categories/>
</form>

</body>
</html>
```

---

### Choosing Between View Technologies

```
JSP:
  ✓ Zero additional dependencies
  ✓ Java EE standard — widely known
  ✓ Mature tooling (IDE support)
  ✗ Requires server for preview
  ✗ Mixed concerns (Java code in HTML historically)
  ✗ Template errors hard to debug
  ✗ Not Spring Boot default
  Use when: legacy projects, no additional dependencies desired

Thymeleaf:
  ✓ Natural HTML — works in browser without server
  ✓ Spring Boot default
  ✓ Strong Spring Security integration
  ✓ Spring Form integration
  ✓ Good error messages
  ✓ Active development
  ✗ Slightly more complex setup in plain Spring MVC
  ✗ Syntax learning curve
  Use when: new projects, Spring Boot, designer collaboration

FreeMarker:
  ✓ Simple syntax
  ✓ Powerful macros and directives
  ✓ Good performance
  ✓ Mature and stable
  ✗ Less Spring integration than Thymeleaf
  ✗ Less popular than Thymeleaf in Spring ecosystem
  Use when: email templates, document generation, content management
```

---

### ViewResolver Chain — Technology Coexistence

```java
// Multiple view technologies can coexist:
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureViewResolvers(
            ViewResolverRegistry registry) {

        // 1. Thymeleaf first (can return null if template missing)
        // Auto-registered if ThymeleafAutoConfiguration active

        // 2. FreeMarker second
        // Auto-registered if FreeMarkerAutoConfiguration active

        // 3. JSP LAST (never returns null)
        registry.jsp("/WEB-INF/views/", ".jsp");

        // Controller returns "product/list":
        // → ThymeleafViewResolver: looks for classpath:/templates/product/list.html
        //   → found? → ThymeleafView returned
        //   → not found? → null → try next
        // → FreeMarkerViewResolver: looks for /templates/freemarker/product/list.ftlh
        //   → found? → FreeMarkerView returned
        //   → not found? → null → try next
        // → InternalResourceViewResolver: /WEB-INF/views/product/list.jsp
        //   → always returns View (never null)
        //   → JSP may or may not exist — found at render time
    }
}
```

---

### ViewResolver Caching — Performance Implications

```
AbstractCachingViewResolver behavior:

Cache key: viewName + Locale (e.g., "product/list" + en_US)
Cache storage: ConcurrentHashMap
Thread safety: View objects must be thread-safe (shared)

SENTINEL value: UNRESOLVED_VIEW
  Meaning: "I tried to resolve this view name and FAILED"
  Stored in cache: yes — prevents re-trying failed resolution
  → For ThymeleafViewResolver: if template.html not found
    → caches UNRESOLVED_VIEW for "product/list"
    → subsequent requests: cache miss check, sentinel found → return null

IMPLICATION:
  In production with caching enabled:
    New template added? → Won't be found until cache cleared
    Existing template modified? → Old version served until cache cleared

Cache management:
  setCacheLimit(0)        → disable caching
  setCacheUnresolved(true) → cache failed resolutions (default for some)
  removeFromCache(name, locale) → manual eviction
```

---

### Model Attribute Access — Per Technology Comparison

```
MODEL ACCESS IN EACH TECHNOLOGY:
─────────────────────────────────────────────────────────────────
Technology  Mechanism              Example
─────────────────────────────────────────────────────────────────
JSP         request.getAttribute() ${product.name}
            (via exposeModelAsRequestAttributes)

Thymeleaf   IContext.getVariable() ${product.name}
            (model directly in context)

FreeMarker  TemplateModel          ${product.name}
            (model converted to FTL model)

All three:  Same RESULT (${product.name} works in all)
            DIFFERENT MECHANISM (how data gets to template)

JSP is SPECIAL: model data MUST be in request attributes
               because JSP EL resolves from request scope
               Spring does this in exposeModelAsRequestAttributes()

Thymeleaf/FreeMarker: model data is directly in their context objects
                      they DON'T require request attributes
                      (though Spring still sets them for consistency)
```

---

### i18n Integration — MessageSource in Views

```java
// Spring's MessageSource integration per technology:

// JSP with Spring tags:
<%@ taglib prefix="spring" uri="http://www.springframework.org/tags" %>
<spring:message code="product.name.label"/>
// Uses Spring's MessageSource directly

// JSP with JSTL (requires JstlView):
<fmt:message key="product.name.label"/>
// Uses MessageSource via JSTL LocalizationContext (if JstlView)

// Thymeleaf:
<p th:text="#{product.name.label}">Product Name</p>
// Uses Spring's MessageSource
// Locale from LocaleContextHolder

// FreeMarker:
${rc.getMessage("product.name.label")}
// rc = RequestContext (Spring's abstraction)
// Uses MessageSource

// Messages file structure:
// messages.properties (default)
// messages_fr.properties (French)
// messages_de.properties (German)
// Location: classpath root or configured in MessageSource bean
```

---

## 2️⃣ CODE EXAMPLES

### Complete JSP Configuration and Setup

```java
// web.xml or Java config:
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    // ViewResolver for JSP
    @Bean
    public InternalResourceViewResolver jspViewResolver() {
        InternalResourceViewResolver resolver =
            new InternalResourceViewResolver();
        resolver.setViewClass(JstlView.class);  // Enable JSTL+Spring MessageSource
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");
        resolver.setOrder(Integer.MAX_VALUE);  // LAST in chain

        // Static attributes available in ALL views
        Map<String, Object> attrs = new HashMap<>();
        attrs.put("appVersion", "2.0");
        attrs.put("supportUrl", "https://support.example.com");
        resolver.setAttributesMap(attrs);

        // Expose Spring beans as variables in JSP EL?
        resolver.setExposeContextBeansAsAttributes(false);
        // true would expose ALL Spring beans — security risk

        return resolver;
    }
}

// Directory structure:
// /WEB-INF/views/
//   product/
//     list.jsp
//     form.jsp
//     detail.jsp
//   common/
//     header.jspf   (JSP fragment, included)
//     footer.jspf
//   error/
//     404.jsp
//     500.jsp
```

**product/list.jsp:**
```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<%@ taglib prefix="fmt" uri="http://java.sun.com/jsp/jstl/fmt" %>
<%@ taglib prefix="spring" uri="http://www.springframework.org/tags" %>
<%@ taglib prefix="form" uri="http://www.springframework.org/tags/form" %>

<!DOCTYPE html>
<html>
<head>
    <title><spring:message code="product.list.title"/></title>
</head>
<body>
<h1><spring:message code="product.list.heading"/></h1>

<c:if test="${not empty successMessage}">
    <div class="alert-success">${successMessage}</div>
</c:if>

<c:choose>
    <c:when test="${empty products}">
        <p>No products found.</p>
    </c:when>
    <c:otherwise>
        <table>
            <tr>
                <th>ID</th><th>Name</th><th>Price</th>
            </tr>
            <c:forEach items="${products}" var="p"
                       varStatus="status">
                <tr class="${status.odd ? 'odd' : 'even'}">
                    <td>${p.id}</td>
                    <td>
                        <c:url var="detailUrl"
                               value="/products/${p.id}"/>
                        <a href="${detailUrl}">${p.name}</a>
                    </td>
                    <td>
                        <fmt:formatNumber value="${p.price}"
                            type="currency" currencyCode="USD"/>
                    </td>
                </tr>
            </c:forEach>
        </table>
    </c:otherwise>
</c:choose>

<!-- Pagination -->
<c:if test="${totalPages > 1}">
    <c:forEach begin="1" end="${totalPages}" var="page">
        <c:url var="pageUrl" value="/products">
            <c:param name="page" value="${page}"/>
        </c:url>
        <a href="${pageUrl}"
           class="${page == currentPage ? 'active' : ''}">
            ${page}
        </a>
    </c:forEach>
</c:if>
</body>
</html>
```

---

### Complete Thymeleaf Configuration and Templates

```java
// Thymeleaf configuration for plain Spring MVC
// (Spring Boot auto-configures this)
@Configuration
public class ThymeleafConfig {

    @Autowired
    private ApplicationContext applicationContext;

    @Bean
    public SpringTemplateEngine templateEngine() {
        SpringTemplateEngine engine =
            new SpringTemplateEngine();
        engine.setTemplateResolver(templateResolver());
        engine.setEnableSpringELCompiler(true);
        // SpringEL compiler improves performance for cached templates

        // Add Spring Security dialect
        engine.addDialect(
            new SpringSecurityDialect());

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
        resolver.setCharacterEncoding("UTF-8");
        resolver.setCacheable(true);
        // Development: resolver.setCacheable(false)
        return resolver;
    }

    @Bean
    public ThymeleafViewResolver thymeleafViewResolver() {
        ThymeleafViewResolver resolver =
            new ThymeleafViewResolver();
        resolver.setTemplateEngine(templateEngine());
        resolver.setCharacterEncoding("UTF-8");
        resolver.setOrder(1);
        // Returns null for missing templates → chain continues
        return resolver;
    }
}
```

**templates/product/list.html:**
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org"
      xmlns:sec="http://www.thymeleaf.org/extras/spring-security"
      lang="en">
<head>
    <meta charset="UTF-8">
    <title th:text="#{product.list.title}">Product List</title>
    <!-- Fragment inclusion for common head -->
    <th:block th:replace="~{common/layout :: head}"/>
</head>
<body>

<!-- Layout fragment inclusion -->
<div th:replace="~{common/layout :: header}"></div>

<main>
    <!-- Flash message from redirect -->
    <div th:if="${successMessage}" class="alert"
         th:text="${successMessage}">Success</div>

    <!-- Conditional content based on authentication -->
    <a sec:authorize="hasRole('ADMIN')"
       th:href="@{/products/new}">Add Product</a>

    <!-- Products table -->
    <table th:if="${not #lists.isEmpty(products)}">
        <thead>
            <tr>
                <th th:text="#{table.header.id}">ID</th>
                <th th:text="#{table.header.name}">Name</th>
                <th th:text="#{table.header.price}">Price</th>
            </tr>
        </thead>
        <tbody>
            <tr th:each="product, status : ${products}"
                th:class="${status.odd} ? 'odd' : 'even'">
                <td th:text="${product.id}">1</td>
                <td>
                    <a th:href="@{/products/{id}(id=${product.id})}"
                       th:text="${product.name}">Product</a>
                </td>
                <td th:text="${#numbers.formatDecimal(
                    product.price, 1, 2, 'POINT')}">0.00</td>
            </tr>
        </tbody>
    </table>

    <!-- Empty state -->
    <div th:unless="${not #lists.isEmpty(products)}"
         th:text="#{product.list.empty}">No products</div>
</main>

<div th:replace="~{common/layout :: footer}"></div>

</body>
</html>
```

**templates/product/form.html:**
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head><title>Product Form</title></head>
<body>

<form th:action="@{/products}" th:object="${productForm}"
      method="post" th:method="${productForm.id != null} ? 'put' : 'post'">

    <!-- Hidden method for PUT -->
    <input type="hidden" name="_method"
           th:value="${productForm.id != null} ? 'PUT' : 'POST'"/>

    <!-- CSRF token (Spring Security) -->
    <input type="hidden"
           th:name="${_csrf.parameterName}"
           th:value="${_csrf.token}"/>

    <div th:class="${#fields.hasErrors('name')} ? 'has-error' : ''">
        <label for="name" th:text="#{field.name}">Name</label>
        <input th:field="*{name}" type="text" id="name"
               class="form-control"/>
        <span th:if="${#fields.hasErrors('name')}"
              th:errors="*{name}" class="error-text">
            Error message
        </span>
    </div>

    <div th:class="${#fields.hasErrors('price')} ? 'has-error' : ''">
        <label for="price">Price</label>
        <input th:field="*{price}" type="number"
               step="0.01" id="price"/>
        <span th:if="${#fields.hasErrors('price')}"
              th:errors="*{price}" class="error-text">
        </span>
    </div>

    <div>
        <label for="categoryId">Category</label>
        <select th:field="*{categoryId}" id="categoryId">
            <option value="" th:text="#{select.category}">
                Select category
            </option>
            <option th:each="cat : ${categories}"
                    th:value="${cat.id}"
                    th:text="${cat.name}">Category</option>
        </select>
    </div>

    <!-- Global errors -->
    <div th:if="${#fields.hasAnyErrors()}">
        <ul>
            <li th:each="err : ${#fields.allErrors()}"
                th:text="${err}"></li>
        </ul>
    </div>

    <button type="submit"
            th:text="${productForm.id != null}
                     ? #{button.update} : #{button.create}">
        Submit
    </button>
</form>

</body>
</html>
```

---

### FreeMarker Setup and Template

```java
// Spring Boot FreeMarker auto-configuration (for reference):
// spring.freemarker.template-loader-path=classpath:/templates/
// spring.freemarker.suffix=.ftlh
// spring.freemarker.charset=UTF-8
// spring.freemarker.cache=true (false in dev)

// Manual configuration:
@Configuration
public class FreeMarkerConfig {

    @Bean
    public FreeMarkerConfigurer freeMarkerConfigurer() {
        FreeMarkerConfigurer configurer =
            new FreeMarkerConfigurer();
        configurer.setTemplateLoaderPath(
            "classpath:/templates/freemarker/");
        configurer.setDefaultEncoding("UTF-8");

        Properties settings = new Properties();
        settings.setProperty("default_encoding", "UTF-8");
        settings.setProperty("output_encoding", "UTF-8");
        settings.setProperty("number_format", "computer");
        settings.setProperty("date_format", "yyyy-MM-dd");
        settings.setProperty("locale", "en_US");
        configurer.setFreemarkerSettings(settings);

        return configurer;
    }

    @Bean
    public FreeMarkerViewResolver freeMarkerViewResolver() {
        FreeMarkerViewResolver resolver =
            new FreeMarkerViewResolver();
        resolver.setPrefix("fm/");
        resolver.setSuffix(".ftlh");
        resolver.setContentType("text/html;charset=UTF-8");
        resolver.setRequestContextAttribute("spring");
        resolver.setOrder(2);
        return resolver;
    }
}
```

**templates/freemarker/fm/product/list.ftlh:**
```html
<!DOCTYPE html>
<#-- Import Spring macros -->
<#import "/spring.ftl" as spring>
<html>
<head>
    <title>${spring.getMessage("product.list.title")}</title>
</head>
<body>

<h1>${spring.getMessage("product.list.heading")}</h1>

<#-- Success message from flash attributes -->
<#if successMessage?has_content>
    <div class="alert">${successMessage}</div>
</#if>

<#-- Products table -->
<#if products?has_content>
<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Price</th>
        </tr>
    </thead>
    <tbody>
    <#list products as product>
        <tr class="${product?is_odd_item?string("odd", "even")}">
            <td>${product.id}</td>
            <td>
                <a href="${spring.url("/products/" + product.id)}">
                    ${product.name?html}
                </a>
            </td>
            <td>${product.price?string("0.00")}</td>
        </tr>
    <#else>
        <tr><td colspan="3">No products</td></tr>
    </#list>
    </tbody>
</table>
<#else>
    <p>${spring.getMessage("product.list.empty")}</p>
</#if>

</body>
</html>
```

---

### Edge Case — ViewResolver Returns Null vs Throws

```java
// Thymeleaf returns null → chain continues → JSP tried
@Bean
public ThymeleafViewResolver thymeleafResolver() {
    ThymeleafViewResolver r = new ThymeleafViewResolver();
    r.setOrder(1);
    // If template not found: returns null
    // → InternalResourceViewResolver is tried next
    return r;
}

// BUT: what if both fail?
// InternalResourceViewResolver creates View for any name
// View.render() → forward() → Servlet container 404
// NOT a Spring 404 — container's 404 page shown

// Prevention: configure default error page
// web.xml:
// <error-page><error-code>404</error-code>
//   <location>/WEB-INF/views/error/404.jsp</location>
// </error-page>

// Or: validate view names before returning from controller
@GetMapping("/products/{type}")
public String getProducts(@PathVariable String type, Model model) {
    if (!VALID_TYPES.contains(type)) {
        throw new ResponseStatusException(
            HttpStatus.NOT_FOUND, "Product type not found");
    }
    // ... populate model
    return "product/" + type;
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
`InternalResourceView.renderMergedOutputModel()` calls `request.getRequestDispatcher(path).forward(request, response)`. What are the implications of this for request attributes?

A) Request attributes are cleared before the forward — model data lost
B) Request attributes are preserved through the forward — model data available in JSP
C) Only session attributes survive the forward
D) A new request is created for the forward — model must be passed explicitly

**Answer: B**
`RequestDispatcher.forward()` is an internal server-side dispatch. The SAME `HttpServletRequest` object is passed to the JSP. All attributes set via `request.setAttribute()` (including model attributes set by `exposeModelAsRequestAttributes()`) are available in the JSP. EL `${product}` resolves `request.getAttribute("product")`.

---

**Q2 — Select All That Apply**
Which are TRUE differences between Thymeleaf and JSP rendering?

A) Thymeleaf uses `RequestDispatcher.forward()`; JSP does not
B) JSP is compiled by the Servlet container; Thymeleaf is processed by a Java engine
C) Thymeleaf templates can be viewed in a browser without a server; JSP cannot
D) JSP requires `/WEB-INF/` for security; Thymeleaf templates can be anywhere in classpath
E) Thymeleaf's `ViewResolver` can return `null`; `InternalResourceViewResolver` cannot

**Answer: B, C, D, E**
A is wrong — it's the OPPOSITE: JSP uses `RequestDispatcher.forward()`, Thymeleaf uses its template engine directly. B, C, D, E are all correct.

---

**Q3 — MCQ**
`SpringTemplateEngine` has `setEnableSpringELCompiler(true)`. What does this do?

A) Enables `${...}` expressions to access Spring beans
B) Compiles Spring EL expressions to bytecode for faster execution
C) Enables `@{...}` URL expressions
D) Enables `#{...}` message expressions

**Answer: B**
`setEnableSpringELCompiler(true)` activates Thymeleaf's Spring Expression Language compiler which compiles SpEL expressions (like `${product.name}`) to bytecode. On first execution the expression is interpreted; compiled bytecode is cached. Subsequent executions are significantly faster. This is a performance optimization — it does NOT change functionality.

---

**Q4 — Select All That Apply**
Which statements about `AbstractCachingViewResolver` caching are TRUE?

A) The cache key includes both view name AND locale
B) A `View` object is created once per `viewName+locale` combination
C) `setCacheable(false)` creates a new `View` object per request
D) Failed resolutions (null returns) are also cached with a sentinel value
E) The cache is a `HashMap` — not thread-safe

**Answer: A, B, C, D**
E is wrong — the cache uses `ConcurrentHashMap` — fully thread-safe. A: yes, locale is part of the cache key (same view name may render differently per locale). B: yes, View objects are shared — they must be thread-safe. C: yes, caching disabled → `createView()` called per request. D: `UNRESOLVED_VIEW` sentinel prevents retrying failed resolutions.

---

**Q5 — True/False**
In a Spring MVC application using JSP, placing template files in `src/main/resources/templates/` (classpath) is equivalent to placing them in `src/main/webapp/WEB-INF/views/` for security purposes.

**Answer: False**
`/WEB-INF/` is protected by the Servlet specification — the container blocks direct browser access. Files in `classpath:/templates/` are accessible from the classpath but are NOT directly served as web resources UNLESS a static resource handler maps to them. However, JSP files in the classpath cannot be resolved by `InternalResourceViewResolver` because `RequestDispatcher` operates on web application paths, not classpath. For JSP: always use `webapp/WEB-INF/views/`. Thymeleaf templates CAN be in classpath.

---

**Q6 — MCQ**
A `ThymeleafViewResolver` and `InternalResourceViewResolver` are both configured. The controller returns `"dashboard"`. There is no `dashboard.html` in Thymeleaf's template path, but there IS a `dashboard.jsp` in `/WEB-INF/views/`. What is the rendering result?

A) `ThymeleafViewResolver` throws `TemplateInputException` — no fallback
B) `InternalResourceViewResolver` is ignored — Thymeleaf always wins
C) `ThymeleafViewResolver` returns `null` → `InternalResourceViewResolver` creates a View → `dashboard.jsp` is rendered
D) Both resolvers claim the view — conflict error

**Answer: C**
`ThymeleafViewResolver` checks if `dashboard.html` exists. Not found → returns `null`. `DispatcherServlet.resolveViewName()` continues iteration. `InternalResourceViewResolver.canHandle()` returns `true` (always). Creates `InternalResourceView("/WEB-INF/views/dashboard.jsp")`. `forward()` → JSP rendered.

---

**Q7 — MCQ**
In Thymeleaf, `th:text="${#fields.hasErrors('name')}"` is used in a form template. What does `#fields` represent?

A) A Spring bean named `fields`
B) The Thymeleaf `Fields` utility object for form field validation status
C) The HTML `<fields>` element
D) The `BindingResult` object directly

**Answer: B**
`#fields` is one of Thymeleaf's built-in utility objects (the `#` prefix indicates utility objects). It provides form validation support: `#fields.hasErrors('name')`, `#fields.errors('name')`, `#fields.allErrors()`, etc. It wraps the `BindingResult` that is in the model when `th:object` refers to a bound form object. It is NOT a Spring bean (A) and NOT accessed directly as `BindingResult` (D).

---

**Q8 — Select All That Apply**
Which view technologies require model data to be exposed as `HttpServletRequest` attributes to access model data in templates?

A) JSP
B) Thymeleaf
C) FreeMarker
D) `MappingJackson2JsonView`

**Answer: A**
Only JSP REQUIRES model data as request attributes because JSP EL resolves variables from `pageScope → requestScope → sessionScope → applicationScope`. Spring's `exposeModelAsRequestAttributes()` does this explicitly for JSP.

Thymeleaf uses its own `IContext` to hold model data — request attributes not required (though Spring still sets them).
FreeMarker uses its own `TemplateModel` hierarchy — request attributes not required.
`MappingJackson2JsonView` serialises the model map directly with Jackson — no HTTP attributes involved.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — JSP uses `forward()`, Thymeleaf uses direct writing**
JSP: `RequestDispatcher.forward()` → NEW internal dispatch to JSP servlet → Tomcat compiles and runs JSP.
Thymeleaf: `templateEngine.process()` → template processed in Java → HTML written directly to response writer.
Exam questions test whether you know that JSP rendering involves the Servlet container while Thymeleaf does not. A common confusion is thinking both "just render HTML."

**Trap 2 — `InternalResourceViewResolver` never returns null — always last**
This is the most critical ordering rule for view technologies. `ThymeleafViewResolver` and `FreeMarkerViewResolver` can return `null` (template not found). `InternalResourceViewResolver.canHandle()` always returns `true`. Placing JSP resolver BEFORE Thymeleaf or FreeMarker means ALL view names get resolved to JSP paths — Thymeleaf/FreeMarker are never consulted.

**Trap 3 — JSP files in `/WEB-INF/` vs classpath**
`InternalResourceViewResolver` uses `RequestDispatcher` which works with web application paths (relative to web root). JSP files MUST be in the web application (`src/main/webapp/`), specifically under `/WEB-INF/` for security. They CANNOT be in `src/main/resources/` (classpath). This is a common Spring Boot migration mistake — trying to put JSPs in the classpath where `RequestDispatcher` cannot find them.

**Trap 4 — Thymeleaf caching and development**
In production: `resolver.setCacheable(true)` — templates cached after first load. In development: must set `resolver.setCacheable(false)` or template changes won't reflect without restart. Spring Boot sets this via `spring.thymeleaf.cache=false` in dev profile. Forgetting to disable cache in development is a common frustration.

**Trap 5 — View objects are shared across threads**
`AbstractCachingViewResolver` stores ONE `View` instance per `viewName + locale`. That single instance is shared across all concurrent requests. Custom `View` implementations that store per-request state in instance fields have race conditions. All view technologies' built-in View implementations are thread-safe by design.

**Trap 6 — `JstlView` vs `InternalResourceView` for i18n**
Using `<fmt:message>` in JSP with plain `InternalResourceView` (default): the JSTL message source is used, NOT Spring's `MessageSource`. To use Spring's `MessageSource` with JSTL `fmt:message`, you must use `JstlView` (set via `resolver.setViewClass(JstlView.class)`). Spring Boot's auto-configuration uses `JstlView` when JSTL is detected.

**Trap 7 — `th:object` and BindingResult requirement**
`th:object="${productForm}"` in Thymeleaf binds to a model attribute. For `#fields.hasErrors()` to work, the `BindingResult` for that model attribute MUST be in the model (added by Spring MVC when `@ModelAttribute` with `BindingResult` is in the handler signature). Without `BindingResult` in model → `#fields.hasErrors()` throws exception at render time.

---

## 5️⃣ SUMMARY SHEET

```
VIEW RENDERING MECHANISM COMPARISON
─────────────────────────────────────────────────────
JSP:
  Renderer:      Servlet container (Tomcat Jasper)
  Mechanism:     RequestDispatcher.forward() → Jasper compiles JSP → runs
  Model access:  request.getAttribute() via exposeModelAsRequestAttributes()
  Location:      /WEB-INF/views/ (web app path, NOT classpath)
  Security:      /WEB-INF/ not accessible from browser
  ViewResolver:  InternalResourceViewResolver (NEVER returns null)
  i18n:          JstlView needed for Spring MessageSource + fmt:message

Thymeleaf:
  Renderer:      SpringTemplateEngine (pure Java)
  Mechanism:     templateEngine.process() → writes to response writer directly
  Model access:  IContext object (model passed directly)
  Location:      classpath:/templates/ (configurable)
  Security:      Classpath not web-accessible by default
  ViewResolver:  ThymeleafViewResolver (returns null if template missing)
  i18n:          #{key} uses Spring MessageSource natively

FreeMarker:
  Renderer:      FreeMarker engine (pure Java)
  Mechanism:     template.process() → writes to response writer
  Model access:  TemplateModel (FTL model objects)
  Location:      configurable (classpath)
  ViewResolver:  FreeMarkerViewResolver (returns null if missing)
  i18n:          ${spring.getMessage("key")} via RequestContext

VIEWRESOLVER CHAIN RULES
─────────────────────────────────────────────────────
InternalResourceViewResolver: ALWAYS returns non-null → MUST be last
ThymeleafViewResolver: returns null if template not found → can be first
FreeMarkerViewResolver: returns null if template not found → before JSP

Chain iteration: first non-null wins
If all return null → ServletException: no view found

ABSTRACTCACHINGVIEWRESOLVER CACHING
─────────────────────────────────────────────────────
Cache key: viewName + Locale
One View instance per key → thread-safe required
setCacheable(false) → new View per request (dev only)
UNRESOLVED_VIEW sentinel: caches failed resolutions
setCacheTTLMs(): time-based expiration

MODEL ATTRIBUTE ACCESS PATH
─────────────────────────────────────────────────────
Controller: model.addAttribute("product", p)
        ↓
AbstractView.exposeModelAsRequestAttributes():
  request.setAttribute("product", p)
        ↓
JSP:       ${product.name}  → requestScope["product"].name
Thymeleaf: ${product.name}  → IContext["product"].name
FreeMarker: ${product.name} → TemplateModel["product"].name

JSTL i18n TRAP
─────────────────────────────────────────────────────
InternalResourceView → fmt:message uses JSTL resource bundle
JstlView → fmt:message uses Spring's MessageSource
Set: resolver.setViewClass(JstlView.class) to get Spring i18n

THYMELEAF DIALECT EXTENSIONS
─────────────────────────────────────────────────────
SpringDialect: th:* attributes, ${}, @{}, #{}, *{}
SpringSecurityDialect: sec:authorize, sec:authentication
Layout Dialect: th:replace, th:insert, th:fragment

SPRING EL COMPILER IN THYMELEAF
─────────────────────────────────────────────────────
setEnableSpringELCompiler(true):
  First execution: SpEL interpreted
  Subsequent: bytecode cache used
  Significant performance improvement for cached templates

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "JSP: RequestDispatcher.forward(); Thymeleaf/FreeMarker: direct engine processing"
• "JSP templates must be in webapp/WEB-INF/ — not in classpath"
• "InternalResourceViewResolver never returns null — must always be last"
• "Thymeleaf/FreeMarker ViewResolvers return null for missing templates — safe to place first"
• "View objects cached and shared — must be thread-safe"
• "JstlView needed for Spring MessageSource integration with JSTL fmt:message"
• "Thymeleaf cacheable=false for development; true for production"
• "th:object + #fields requires BindingResult in model — else runtime error"
• "JSP /WEB-INF/ security: browser cannot directly access, only server-side forward"
• "Model data → request attributes (Spring sets this) → all view technologies can read"
```

---

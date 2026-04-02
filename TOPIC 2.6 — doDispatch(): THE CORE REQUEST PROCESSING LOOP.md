# TOPIC 2.6 — doDispatch(): THE CORE REQUEST PROCESSING LOOP

---

## 1️⃣ CONCEPTUAL EXPLANATION

### Why doDispatch() Deserves Its Own Topic

`doDispatch()` is the single most important method in Spring MVC. Every HTTP request that reaches `DispatcherServlet` passes through it. It is the **orchestration engine** — it coordinates all special beans, enforces the interceptor contract, handles exceptions, manages multipart cleanup, and delegates rendering. Understanding it line-by-line means understanding Spring MVC's entire runtime behaviour.

Certification questions, architecture interviews, and debugging sessions all converge on `doDispatch()`. Knowing the exact sequence, the exact conditions under which each step executes, and the exact behaviour when things go wrong is mandatory for mastery.

---

### doDispatch() — Complete Source-Level Walk-Through

```java
// DispatcherServlet.doDispatch()
protected void doDispatch(
        HttpServletRequest request,
        HttpServletResponse response) throws Exception {

    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;

    // Async manager — tracks whether async processing was started
    WebAsyncManager asyncManager =
        WebAsyncUtils.getAsyncManager(request);

    try {
        ModelAndView mv = null;
        Exception dispatchException = null;

        try {
            // ══════════════════════════════════════════
            // PHASE 1: MULTIPART CHECK
            // ══════════════════════════════════════════
            processedRequest = checkMultipart(request);
            multipartRequestParsed =
                (processedRequest != request);
            // If multipart: processedRequest is now a
            // MultipartHttpServletRequest wrapper
            // multipartRequestParsed=true flags cleanup needed

            // ══════════════════════════════════════════
            // PHASE 2: HANDLER LOOKUP
            // ══════════════════════════════════════════
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null) {
                noHandlerFound(processedRequest, response);
                return;
                // Either sendError(404) or throws
                // NoHandlerFoundException
            }

            // ══════════════════════════════════════════
            // PHASE 3: HANDLER ADAPTER SELECTION
            // ══════════════════════════════════════════
            HandlerAdapter ha =
                getHandlerAdapter(mappedHandler.getHandler());

            // ══════════════════════════════════════════
            // PHASE 4: LAST-MODIFIED CHECK (HTTP Caching)
            // ══════════════════════════════════════════
            String method = request.getMethod();
            boolean isGet = HttpMethod.GET.matches(method);
            if (isGet || HttpMethod.HEAD.matches(method)) {
                long lastModified =
                    ha.getLastModified(request,
                        mappedHandler.getHandler());
                if (new ServletWebRequest(request, response)
                        .checkNotModified(lastModified) &&
                        isGet) {
                    return;
                    // HTTP 304 Not Modified — skip everything
                    // Response headers already set by checkNotModified
                }
            }

            // ══════════════════════════════════════════
            // PHASE 5: PRE-HANDLE INTERCEPTORS
            // ══════════════════════════════════════════
            if (!mappedHandler.applyPreHandle(
                    processedRequest, response)) {
                return;
                // A preHandle() returned false
                // afterCompletion already triggered internally
                // by applyPreHandle() for already-run interceptors
            }

            // ══════════════════════════════════════════
            // PHASE 6: HANDLER INVOCATION
            // ══════════════════════════════════════════
            mv = ha.handle(
                processedRequest, response,
                mappedHandler.getHandler());

            // ══════════════════════════════════════════
            // PHASE 7: ASYNC CHECK
            // ══════════════════════════════════════════
            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
                // Async processing started (Callable, DeferredResult)
                // doDispatch() exits — response will be written
                // by async thread when result available
            }

            // ══════════════════════════════════════════
            // PHASE 8: DEFAULT VIEW NAME
            // ══════════════════════════════════════════
            applyDefaultViewName(processedRequest, mv);
            // If mv has no view name → derive from URL
            // via RequestToViewNameTranslator

            // ══════════════════════════════════════════
            // PHASE 9: POST-HANDLE INTERCEPTORS
            // ══════════════════════════════════════════
            mappedHandler.applyPostHandle(
                processedRequest, response, mv);
            // REVERSE ORDER of preHandle
            // NOT called if async started
            // mv may be null here (for @ResponseBody)

        } catch (Exception ex) {
            dispatchException = ex;
            // Store exception — process after finally
        } catch (Throwable err) {
            // Non-Exception Throwable (Error)
            dispatchException =
                new NestedServletException(
                    "Handler dispatch failed", err);
        }

        // ══════════════════════════════════════════════
        // PHASE 10: RESULT PROCESSING
        // (View rendering OR exception handling)
        // ══════════════════════════════════════════════
        processDispatchResult(
            processedRequest, response,
            mappedHandler, mv, dispatchException);

    } catch (Exception ex) {
        // Exception from processDispatchResult itself
        triggerAfterCompletion(
            processedRequest, response, mappedHandler, ex);
        throw ex;
    } catch (Throwable err) {
        Exception ex = new NestedServletException(
            "Handler processing failed", err);
        triggerAfterCompletion(
            processedRequest, response, mappedHandler, ex);
        throw ex;
    } finally {
        // ══════════════════════════════════════════════
        // PHASE 11: CLEANUP (ALWAYS RUNS)
        // ══════════════════════════════════════════════

        if (asyncManager.isConcurrentHandlingStarted()) {
            // Async: interceptors cleanup handled separately
            if (mappedHandler != null) {
                mappedHandler.applyAfterConcurrentHandlingStarted(
                    processedRequest, response);
            }
        } else {
            // Sync: clean up multipart request
            if (multipartRequestParsed) {
                cleanupMultipart(processedRequest);
                // Deletes temp files from file upload
            }
        }
    }
}
```

---

### Phase 1 — checkMultipart() Deep Analysis

```java
protected HttpServletRequest checkMultipart(
        HttpServletRequest request) throws MultipartException {

    if (this.multipartResolver != null &&
        this.multipartResolver.isMultipart(request)) {

        if (WebUtils.getNativeRequest(
                request, MultipartHttpServletRequest.class) != null) {
            // Already wrapped — nested dispatch scenario
            if (DispatcherType.REQUEST.equals(
                    request.getDispatcherType())) {
                logger.trace("Request already resolved to multipart " +
                    "but DispatcherType.REQUEST — re-wrapping");
            }
        } else if (hasMultipartException(request)) {
            // Previous multipart resolution failed — don't retry
            logger.debug("Multipart resolution failed for current " +
                "request before - skipping re-resolution for error dispatch");
        } else {
            try {
                return this.multipartResolver
                    .resolveMultipart(request);
                // Returns MultipartHttpServletRequest wrapper
                // Files parsed, accessible via getFile(), getParts()
            } catch (MultipartException ex) {
                if (request.getAttribute(
                        WebUtils.ERROR_EXCEPTION_ATTRIBUTE) != null) {
                    logger.debug("Multipart resolution failed for " +
                        "error dispatch", ex);
                } else {
                    throw ex;
                    // Propagates to inner catch → dispatchException
                    // Then to processDispatchResult() → exception handling
                }
            }
        }
    }
    return request; // Return original if not multipart
}
```

**Key behaviours:**
- If no `multipartResolver` configured: returns original request unchanged
- Multipart resolution failure: stored as `dispatchException`, processed in Phase 10
- Already-wrapped request (INCLUDE/FORWARD): not re-wrapped
- `multipartRequestParsed=true` flag ensures `cleanupMultipart()` runs in Phase 11

---

### Phase 2 — getHandler() Deep Analysis

```java
@Nullable
protected HandlerExecutionChain getHandler(
        HttpServletRequest request) throws Exception {

    if (this.handlerMappings != null) {
        for (HandlerMapping mapping : this.handlerMappings) {
            HandlerExecutionChain handler =
                mapping.getHandler(request);
            if (handler != null) {
                return handler;
                // First non-null wins — chain iteration stops
            }
        }
    }
    return null;
    // All mappings returned null → noHandlerFound()
}
```

**`noHandlerFound()` — what happens when null:**

```java
protected void noHandlerFound(
        HttpServletRequest request,
        HttpServletResponse response) throws Exception {

    if (this.throwExceptionIfNoHandlerFound) {
        throw new NoHandlerFoundException(
            request.getMethod(),
            getRequestUri(request),
            new ServletServerHttpRequest(request).getHeaders());
        // → Goes to dispatchException
        // → processDispatchResult() → ExceptionHandlerExceptionResolver
        // → @ExceptionHandler(NoHandlerFoundException.class) CAN catch this
    } else {
        // Default behaviour (throwExceptionIfNoHandlerFound=false)
        response.sendError(
            HttpServletResponse.SC_NOT_FOUND);
        // → Bypasses Spring exception handling entirely
        // → Container handles the 404
        // → @ExceptionHandler for 404 NEVER fires in this case
    }
}
```

---

### Phase 3 — getHandlerAdapter() Deep Analysis

```java
protected HandlerAdapter getHandlerAdapter(
        Object handler) throws ServletException {

    if (this.handlerAdapters != null) {
        for (HandlerAdapter adapter : this.handlerAdapters) {
            if (adapter.supports(handler)) {
                return adapter;
                // First adapter that supports handler type wins
            }
        }
    }

    throw new ServletException(
        "No adapter for handler [" + handler + "]: " +
        "The DispatcherServlet configuration needs to include a " +
        "HandlerAdapter that supports this handler");
    // This is NOT stored as dispatchException
    // Thrown directly from the inner try block
    // → Caught by inner catch → stored as dispatchException
    // → Processed in Phase 10
}
```

**When this throws:** Custom `HandlerMapping` returns a handler type for which no `HandlerAdapter` exists. This is the "No adapter for handler" error.

---

### Phase 4 — Last-Modified Check (HTTP Caching)

```java
// Only for GET and HEAD requests
if (isGet || HttpMethod.HEAD.matches(method)) {
    long lastModified = ha.getLastModified(request,
        mappedHandler.getHandler());

    if (new ServletWebRequest(request, response)
            .checkNotModified(lastModified) && isGet) {
        return;
        // HTTP 304 sent to client
        // ETag/Last-Modified headers set on response
        // No handler invocation
        // No interceptors after this point
    }
}
```

**`checkNotModified()` logic:**
```
Client sends: If-Modified-Since: Mon, 04 Mar 2024 10:00:00 GMT
Server's lastModified = 1709550000000 (older than client's cached time)
→ Not modified → 304

Client sends: If-None-Match: "abc123"
Server's ETag = "abc123"
→ Not modified → 304

304 response:
  - No body
  - Content-Length: 0
  - Same cache headers as original 200
  - Skips ALL interceptors (preHandle already ran, postHandle NOT called)
```

---

### Phase 5 — applyPreHandle() — Full Chain Analysis

```java
// HandlerExecutionChain.applyPreHandle()
boolean applyPreHandle(
        HttpServletRequest request,
        HttpServletResponse response) throws Exception {

    for (int i = 0; i < this.interceptorList.size(); i++) {
        HandlerInterceptor interceptor =
            this.interceptorList.get(i);

        if (!interceptor.preHandle(request, response,
                this.handler)) {
            // This interceptor returned false — STOP
            triggerAfterCompletion(request, response, null);
            // Calls afterCompletion() in reverse order
            // for interceptors[0..i-1] that returned true
            // interceptor[i] itself (returned false)
            //   → afterCompletion NOT called for it
            return false;
        }

        this.interceptorIndex = i;
        // Record successful preHandle index
    }
    return true; // All preHandles returned true
}
```

**Interceptor contract summary:**

```
preHandle() returns true  → Continue processing
preHandle() returns false → STOP — afterCompletion for prior interceptors
                            doDispatch() returns immediately (Phase 5 return)

postHandle() called        → REVERSE order, only on SUCCESS path
                             (not called if exception, not called for async)

afterCompletion() called   → REVERSE order, up to interceptorIndex
                             ALWAYS called (exception or success)
                             for interceptors[0..interceptorIndex]
                             Exceptions in afterCompletion are SWALLOWED
```

---

### Phase 6 — Handler Invocation — What ha.handle() Returns

```java
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

**Possible outcomes:**

```
Case 1: @ResponseBody / @RestController method
  → Jackson serialises return value to response
  → mavContainer.requestHandled = true
  → ha.handle() returns null
  → mv = null after Phase 6

Case 2: View-based controller (returns String or ModelAndView)
  → ha.handle() returns ModelAndView with view name + model
  → mv = ModelAndView{view="product/list", model={products=[...]}}

Case 3: void method, response not committed
  → ha.handle() returns ModelAndView with null view (no view name set)
  → applyDefaultViewName() in Phase 8 derives view from URL

Case 4: void method, response committed
  → Same as Case 3 but mavContainer.requestHandled=true
  → ha.handle() returns null
  → mv = null

Case 5: redirect:
  → ha.handle() returns ModelAndView{view="redirect:/success"}
  → render() creates RedirectView
  → Flash attributes saved before redirect

Case 6: ResponseEntity<T> returned
  → HttpEntityMethodProcessor writes status + headers + body
  → mavContainer.requestHandled = true
  → ha.handle() returns null
```

---

### Phase 7 — Async Handling Check

```java
if (asyncManager.isConcurrentHandlingStarted()) {
    return; // Exit doDispatch immediately
}
```

**When `Callable<T>` is returned:**

```
Controller returns Callable<T>
        │
        ▼
CallableMethodReturnValueHandler.handleReturnValue()
  → asyncManager.startCallableProcessing(callable)
  → Submits callable to taskExecutor thread pool
  → Sets asyncManager.concurrentHandlingStarted = true
  → Releases Servlet container thread back to pool
        │
        ▼
doDispatch() Phase 7: asyncManager.isConcurrentHandlingStarted() = true
→ return immediately (Phases 8, 9, 10 SKIPPED)
→ postHandle() NOT called
→ afterCompletion() NOT called yet
→ Servlet container thread released
        │
        ▼
Async thread completes Callable
  → Stores result
  → Dispatches NEW request to DispatcherServlet (DispatcherType.ASYNC)
        │
        ▼
Second doDispatch() call on new Servlet thread
  → Handler found again
  → asyncManager.hasConcurrentResult() = true
  → invokeHandlerMethod() processes the result
  → postHandle() called on second pass
  → afterCompletion() called on second pass
```

---

### Phase 8 — applyDefaultViewName()

```java
private void applyDefaultViewName(
        HttpServletRequest request,
        @Nullable ModelAndView mv) throws Exception {

    if (mv != null && !mv.hasView()) {
        // mv exists but no view name set
        // (controller returned void — no explicit view name)
        String defaultViewName =
            getDefaultViewName(request);
        if (defaultViewName != null) {
            mv.setViewName(defaultViewName);
            // RequestToViewNameTranslator derives from URL
            // /products/list → "products/list"
        }
    }
}
```

**This phase only matters when:** Controller returns `void` or `null` with no explicit view name.

---

### Phase 9 — applyPostHandle()

```java
// HandlerExecutionChain.applyPostHandle()
void applyPostHandle(
        HttpServletRequest request,
        HttpServletResponse response,
        @Nullable ModelAndView mv) throws Exception {

    for (int i = this.interceptorList.size() - 1; i >= 0; i--) {
        // REVERSE ORDER — opposite of preHandle
        HandlerInterceptor interceptor =
            this.interceptorList.get(i);
        interceptor.postHandle(request, response,
            this.handler, mv);
    }
}
```

**Critical limitations of postHandle():**
- `mv` is `null` for `@ResponseBody` endpoints (response already written)
- Cannot modify the response body for `@ResponseBody` (already written in Phase 6)
- CAN modify the `ModelAndView` for view-based responses
- NOT called when:
  - `preHandle()` returned false (Phase 5 returned early)
  - Async processing started (Phase 7 returned early)
  - Exception thrown in Phase 6 (goes straight to Phase 10)
  - HTTP 304 Not Modified (Phase 4 returned early)

---

### Phase 10 — processDispatchResult() — The Most Complex Phase

```java
private void processDispatchResult(
        HttpServletRequest request,
        HttpServletResponse response,
        @Nullable HandlerExecutionChain mappedHandler,
        @Nullable ModelAndView mv,
        @Nullable Exception exception) throws Exception {

    boolean errorView = false;

    // ── EXCEPTION HANDLING PATH ────────────────────────────────────
    if (exception != null) {
        if (exception instanceof ModelAndViewDefiningException mavEx) {
            // Special: exception carries its own ModelAndView
            mv = mavEx.getModelAndView();
        } else {
            Object handler = (mappedHandler != null ?
                mappedHandler.getHandler() : null);

            // Delegate to HandlerExceptionResolver chain
            mv = processHandlerException(
                request, response, handler, exception);
            // Returns ModelAndView (possibly with error view)
            // Returns null if exception fully handled
            //   (e.g., @ResponseBody @ExceptionHandler wrote response)

            errorView = (mv != null);
        }
    }

    // ── RENDERING PATH ─────────────────────────────────────────────
    if (mv != null && !mv.wasCleared()) {
        // Render view with model
        render(mv, request, response);

        if (errorView) {
            WebUtils.clearErrorRequestAttributes(request);
            // Clean up error attributes after error view rendered
        }
    } else {
        // mv is null or cleared
        // → Response was written directly (@ResponseBody, etc.)
        // → No view rendering needed
        if (logger.isTraceEnabled()) {
            logger.trace("No view rendering, null ModelAndView");
        }
    }

    // ── ASYNC CHECK ────────────────────────────────────────────────
    if (WebAsyncUtils.getAsyncManager(request)
            .isConcurrentHandlingStarted()) {
        return; // Async started during error handling — skip afterCompletion
    }

    // ── AFTER COMPLETION ───────────────────────────────────────────
    if (mappedHandler != null) {
        mappedHandler.triggerAfterCompletion(
            request, response, null);
        // null = no exception (exception already handled)
        // afterCompletion ALWAYS runs at this point
        // for interceptors[0..interceptorIndex]
    }
}
```

---

### processHandlerException() — Exception Resolution Chain

```java
// DispatcherServlet.processHandlerException()
@Nullable
protected ModelAndView processHandlerException(
        HttpServletRequest request,
        HttpServletResponse response,
        @Nullable Object handler,
        Exception ex) throws Exception {

    // Clear any ModelAndView from the original request processing
    // (prevents partial model from leaking into error response)
    request.removeAttribute(
        HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);

    ModelAndView exMv = null;

    if (this.handlerExceptionResolvers != null) {
        for (HandlerExceptionResolver resolver :
                this.handlerExceptionResolvers) {

            exMv = resolver.resolveException(
                request, response, handler, ex);

            if (exMv != null) {
                // Resolver handled it
                if (exMv.isEmpty()) {
                    return null;
                    // Empty MaV = fully handled
                    // @ResponseBody exception handler wrote response
                }
                if (!exMv.hasView()) {
                    exMv.setViewName(
                        getDefaultViewName(request));
                }
                return exMv;
                // Has view → render error view
            }
            // null → try next resolver
        }
    }

    // No resolver handled it → re-throw
    throw ex;
    // → Propagates out of processDispatchResult()
    // → Caught by outer try-catch in doDispatch()
    // → triggerAfterCompletion called with the exception
    // → Exception re-thrown to FrameworkServlet.processRequest()
    // → FrameworkServlet re-throws as NestedServletException
    // → Servlet container handles → HTTP 500
}
```

---

### render() — The View Rendering Phase

```java
// DispatcherServlet.render()
protected void render(
        ModelAndView mv,
        HttpServletRequest request,
        HttpServletResponse response) throws Exception {

    // Resolve locale for this request
    Locale locale = (this.localeResolver != null ?
        this.localeResolver.resolveLocale(request) :
        request.getLocale());
    response.setLocale(locale);

    View view;
    String viewName = mv.getViewName();

    if (viewName != null) {
        // View name → resolve to View object
        view = resolveViewName(viewName, mv.getModelInternal(),
            locale, request);

        if (view == null) {
            throw new ServletException(
                "Could not resolve view with name '" +
                viewName + "'");
        }
    } else {
        // Direct View object in ModelAndView
        view = mv.getView();
        if (view == null) {
            throw new ServletException(
                "ModelAndView [" + mv + "] neither contains " +
                "a view name nor a View object in DispatcherServlet");
        }
    }

    if (logger.isTraceEnabled()) {
        logger.trace("Rendering view [" + view + "]");
    }

    try {
        if (mv.getStatus() != null) {
            // @ResponseStatus or explicit status
            request.setAttribute(
                View.RESPONSE_STATUS_ATTRIBUTE, mv.getStatus());
            response.setStatus(mv.getStatus().value());
        }

        // RENDER — calls View.render(model, request, response)
        view.render(mv.getModelInternal(), request, response);

    } catch (Exception ex) {
        if (logger.isDebugEnabled()) {
            logger.debug("Error rendering view [" + view + "]", ex);
        }
        throw ex;
        // Propagates to outer catch in processDispatchResult
        // → triggerAfterCompletion with exception
    }
}
```

---

### Phase 11 — Finally Block — Guaranteed Cleanup

```java
finally {
    if (asyncManager.isConcurrentHandlingStarted()) {
        // ASYNC PATH: interceptors cleanup is different
        if (mappedHandler != null) {
            mappedHandler.applyAfterConcurrentHandlingStarted(
                processedRequest, response);
            // Calls AsyncHandlerInterceptor.afterConcurrentHandlingStarted()
            // NOT afterCompletion() — that fires on async result dispatch
        }
    } else {
        // SYNC PATH: clean up multipart temp files
        if (multipartRequestParsed) {
            cleanupMultipart(processedRequest);
            // StandardServletMultipartResolver:
            //   Calls request.getParts() → iterates → part.delete()
            //   Deletes temp files from disk
        }
    }
}
```

**`cleanupMultipart()` is ALWAYS called in finally** — even if:
- Handler threw an exception
- Interceptor returned false
- Exception resolver failed
- Response was already committed

Temp files are always cleaned up. This is critical for disk space management.

---

### Complete Execution Flow Diagram — All Paths

```
doDispatch(request, response)
│
├── [Phase 1] checkMultipart()
│   ├── Not multipart: processedRequest = request
│   └── Multipart: processedRequest = MultipartHttpServletRequest wrapper
│                  multipartRequestParsed = true
│
├── [Phase 2] getHandler()
│   ├── null → noHandlerFound()
│   │   ├── throwExceptionIfNoHandlerFound=true → NoHandlerFoundException
│   │   │   → dispatchException = ex → Phase 10
│   │   └── throwExceptionIfNoHandlerFound=false → sendError(404) → RETURN
│   └── HandlerExecutionChain → mappedHandler
│
├── [Phase 3] getHandlerAdapter()
│   ├── No adapter → ServletException → dispatchException → Phase 10
│   └── HandlerAdapter → ha
│
├── [Phase 4] Last-Modified check (GET/HEAD only)
│   └── Not modified → 304 → RETURN (no further processing)
│
├── [Phase 5] mappedHandler.applyPreHandle()
│   ├── preHandle=false → afterCompletion for prior interceptors → RETURN
│   └── All true → continue
│
├── [Phase 6] ha.handle() → mv (possibly null for @ResponseBody)
│   └── Exception → dispatchException → skip Phases 7-9 → Phase 10
│
├── [Phase 7] asyncManager.isConcurrentHandlingStarted()
│   └── true → applyAfterConcurrentHandlingStarted (in finally) → RETURN
│
├── [Phase 8] applyDefaultViewName() — if mv≠null and no view name
│
├── [Phase 9] mappedHandler.applyPostHandle() — REVERSE order
│   └── Exception → dispatchException → Phase 10
│
├── [Phase 10] processDispatchResult(request, response, mappedHandler, mv, dispatchException)
│   ├── If exception:
│   │   ├── processHandlerException() → resolver chain
│   │   │   ├── @ExceptionHandler found → writes response OR returns error MaV
│   │   │   └── No resolver → re-throws → triggerAfterCompletion(exception) → throws
│   │   └── Resolved mv (possibly null if fully handled)
│   ├── If mv≠null → render() → resolveViewName() → view.render()
│   └── triggerAfterCompletion(null) — ALWAYS runs here (success path)
│
└── [Phase 11] finally
    ├── Async → applyAfterConcurrentHandlingStarted()
    └── Sync → cleanupMultipart() if multipartRequestParsed
```

---

### Exception Propagation — All Paths

```
EXCEPTION IN PHASE 6 (handler invocation):
  → dispatchException = ex
  → Phases 8, 9 SKIPPED
  → Phase 10: processHandlerException() tries resolvers
    → Resolved: renders error view or null (fully handled)
               + triggerAfterCompletion(null)
    → Unresolved: re-throws
                  → caught by outer catch in doDispatch()
                  → triggerAfterCompletion(exception)
                  → re-throws to FrameworkServlet
                  → → HTTP 500

EXCEPTION IN PHASE 9 (postHandle):
  → dispatchException = ex
  → Phase 10: same as above

EXCEPTION IN PHASE 10 (render() or exception resolver):
  → Caught by outer try-catch
  → triggerAfterCompletion(exception)
  → re-thrown to FrameworkServlet → HTTP 500

EXCEPTION IN afterCompletion:
  → SWALLOWED (logged only — never propagated)
  → Does NOT affect response
  → Does NOT affect HTTP status
```

---

### Threading Model — One Full Request Cycle

```
Tomcat Thread Pool Thread #47 assigned to request:
│
├── FrameworkServlet.processRequest()
│     → Sets ThreadLocals (LocaleContextHolder, RequestContextHolder)
│     │
│     └── DispatcherServlet.doService()
│           → Sets request attributes (WebApplicationContext, etc.)
│           │
│           └── DispatcherServlet.doDispatch()
│                 → All 11 phases run on Thread #47
│                 → Thread #47 blocked for entire duration
│                 → Including DB calls, service calls, rendering
│                 │
│                 [If Callable returned — Phase 7]
│                 → Thread #47 released back to pool
│                 → Worker Thread #12 runs Callable
│                 → Async dispatch triggers new request
│                 → Thread #48 from pool handles second pass
│
├── finally: ThreadLocals cleared
└── Thread #47 returned to pool (or #48 for async)
```

---

### INCLUDE and FORWARD — Nested doDispatch()

```
REQUEST dispatch (normal):
  doDispatch() → handler invoked directly

FORWARD dispatch (RequestDispatcher.forward()):
  Second doDispatch() call on SAME thread
  → request.getDispatcherType() = FORWARD
  → doService() saves/restores request attributes
  → Full pipeline re-runs with forwarded URL

INCLUDE dispatch (RequestDispatcher.include()):
  Second doDispatch() call on SAME thread
  → request.getDispatcherType() = INCLUDE
  → doService() saves request attribute snapshot
  → Full pipeline re-runs with included URL
  → After include: original attributes restored
  → Response NOT committed by include (headers already sent by outer)

ERROR dispatch (Servlet container error handling):
  doDispatch() called for error URL
  → request.getDispatcherType() = ERROR
  → Standard error attributes available:
    javax.servlet.error.status_code
    javax.servlet.error.exception
    javax.servlet.error.request_uri
```

---

## 2️⃣ CODE EXAMPLES

### Interceptor — Demonstrating Exact doDispatch() Interaction

```java
@Component
public class TimingInterceptor implements HandlerInterceptor {

    private static final String START_TIME = "startTime";

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) {
        // Phase 5 — before ha.handle()
        request.setAttribute(START_TIME,
            System.currentTimeMillis());
        System.out.println("PRE: " + request.getRequestURI());
        return true; // Continue
    }

    @Override
    public void postHandle(HttpServletRequest request,
                            HttpServletResponse response,
                            Object handler,
                            @Nullable ModelAndView modelAndView) {
        // Phase 9 — ONLY called if:
        // 1. preHandle returned true
        // 2. No exception in ha.handle()
        // 3. No async started
        // NOTE: modelAndView is NULL for @ResponseBody endpoints
        System.out.println("POST: mv=" + modelAndView);
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                  HttpServletResponse response,
                                  Object handler,
                                  @Nullable Exception ex) {
        // Phase 10 (inside processDispatchResult) — ALWAYS (for this interceptor)
        // even if exception, even if postHandle not called
        long start = (Long) request.getAttribute(START_TIME);
        long elapsed = System.currentTimeMillis() - start;
        System.out.println("AFTER: " + elapsed + "ms, ex=" + ex);
    }
}
```

---

### Interceptor Returning false — Demonstrating Phase 5 Return

```java
@Component
public class ApiKeyInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) throws IOException {
        String apiKey = request.getHeader("X-API-Key");

        if (apiKey == null || !isValid(apiKey)) {
            // Write response directly
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.setContentType("application/json");
            response.getWriter().write(
                "{\"error\":\"Invalid or missing API key\"}");

            // Return false:
            // → doDispatch() returns immediately after applyPreHandle()
            // → ha.handle() NOT called
            // → postHandle() NOT called
            // → afterCompletion() called for interceptors[0..this-1]
            //   (interceptors that ran before this one)
            return false;
        }
        return true;
    }
}
```

---

### Observing All doDispatch() Phases via Filter

```java
// Filter wraps the entire doDispatch() call
@Component
public class RequestLifecycleFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain)
            throws ServletException, IOException {

        System.out.println("=== FILTER: Before DispatcherServlet ===");
        System.out.println("Thread: " +
            Thread.currentThread().getName());
        System.out.println("URI: " + request.getRequestURI());

        long start = System.nanoTime();
        try {
            chain.doFilter(request, response);
            // DispatcherServlet.service() → doDispatch() runs here
        } finally {
            long elapsed = System.nanoTime() - start;
            System.out.println("=== FILTER: After DispatcherServlet ===");
            System.out.println("Elapsed: " +
                TimeUnit.NANOSECONDS.toMillis(elapsed) + "ms");
            System.out.println("Status: " + response.getStatus());
            // Note: at this point, all doDispatch() phases are complete
            // ThreadLocals already cleared by processRequest() finally block
        }
    }
}
```

---

### Async Processing — Callable Return Value

```java
@RestController
public class AsyncController {

    @Autowired
    private TaskExecutor taskExecutor;

    @GetMapping("/async/data")
    public Callable<List<Product>> getDataAsync() {
        // Returns Callable — triggers async processing
        // doDispatch() Phase 7: exits immediately
        // Servlet container thread released

        return () -> {
            // Runs on separate task executor thread
            // NOT on Servlet container thread
            Thread.sleep(2000); // Simulates slow DB query
            return productService.findAll();
            // When complete: new dispatch to DispatcherServlet
            // Second doDispatch() call processes the List<Product>
            // as @ResponseBody return value
        };
    }

    @GetMapping("/async/deferred")
    public DeferredResult<String> getDeferred() {
        DeferredResult<String> result =
            new DeferredResult<>(5000L); // 5 second timeout

        // Return immediately — no thread holds this request
        taskExecutor.execute(() -> {
            try {
                Thread.sleep(1000);
                result.setResult("Completed");
                // Triggers async dispatch back to DispatcherServlet
            } catch (InterruptedException e) {
                result.setErrorResult(e);
            }
        });

        return result;
    }
}
```

---

### Exception Flow — End-to-End

```java
// Complete exception handling demonstration
@RestController
public class ExceptionDemoController {

    // This exception in ha.handle() Phase 6:
    // → dispatchException set
    // → Phases 8, 9 skipped
    // → Phase 10: processHandlerException() called
    @GetMapping("/products/{id}")
    public Product getProduct(@PathVariable Long id) {
        return productService.findById(id)
            .orElseThrow(() ->
                new ProductNotFoundException(id));
                // Unchecked — propagates out of ha.handle()
    }
}

// Exception handler — called in Phase 10
@ControllerAdvice
public class GlobalExceptionHandler {

    // ExceptionHandlerExceptionResolver finds this
    // because exception is ProductNotFoundException
    @ExceptionHandler(ProductNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    @ResponseBody
    public Map<String, Object> handleNotFound(
            ProductNotFoundException ex) {
        // This method goes through full argument resolution:
        // ProductNotFoundException → resolved as exception param
        // Writes JSON to response
        // mavContainer.requestHandled = true
        // processHandlerException returns null (empty MaV)
        // → No error view rendered
        // → afterCompletion() called in processDispatchResult

        Map<String, Object> error = new LinkedHashMap<>();
        error.put("status", 404);
        error.put("message", "Product not found: " + ex.getId());
        return error;
    }
}
```

---

### Edge Case — postHandle() Cannot Modify @ResponseBody Response

```java
// MISCONCEPTION: Modifying response in postHandle() for @RestController
@Component
public class WrongInterceptor implements HandlerInterceptor {

    @Override
    public void postHandle(HttpServletRequest request,
                            HttpServletResponse response,
                            Object handler,
                            ModelAndView modelAndView) throws IOException {
        // FOR @RestController METHODS:
        // modelAndView IS NULL here
        // Response ALREADY WRITTEN during ha.handle() (Phase 6)
        // response.getWriter().write("extra content") here → exception
        // response is COMMITTED — cannot add headers after body written

        if (modelAndView != null) {
            // This only runs for VIEW-BASED controllers
            modelAndView.addObject("globalData", "value");
            // Safe — view not rendered yet
        }

        // WRONG — may throw IllegalStateException if response committed:
        // response.setHeader("X-Custom", "value");
    }
}

// CORRECT: Use ResponseBodyAdvice for @ResponseBody modification
@ControllerAdvice
public class ResponseEnvelopeAdvice
    implements ResponseBodyAdvice<Object> {

    @Override
    public boolean supports(MethodParameter returnType,
                             Class<? extends HttpMessageConverter<?>> converterType) {
        return true; // Apply to all @ResponseBody methods
    }

    @Override
    public Object beforeBodyWrite(Object body,
                                   MethodParameter returnType,
                                   MediaType selectedContentType,
                                   Class<? extends HttpMessageConverter<?>> converterType,
                                   ServerHttpRequest request,
                                   ServerHttpResponse response) {
        // Runs INSIDE RequestResponseBodyMethodProcessor
        // BEFORE body is written — CAN modify it
        return Map.of("data", body, "timestamp",
            Instant.now().toString());
    }
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
`doDispatch()` Phase 9 — `applyPostHandle()` — is called in REVERSE interceptor order. Under which circumstances is `applyPostHandle()` NOT called at all?

A) When `preHandle()` returns `false` for any interceptor
B) When `ha.handle()` throws an exception
C) When async concurrent handling starts
D) When HTTP 304 Not Modified is returned
E) All of the above

**Answer: E**
`postHandle()` is skipped when:
- A `preHandle()` returns `false` → Phase 5 returns early before reaching Phase 9
- Exception in `ha.handle()` → `dispatchException` set → skips Phases 8, 9 → goes to Phase 10
- Async started → Phase 7 returns early before Phase 9
- HTTP 304 → Phase 4 returns early before Phase 5, 6, 9

All four conditions bypass Phase 9.

---

**Q2 — Ordering**
For a successful `GET /products` request handled by a view-based controller, order these operations:

- `afterCompletion()` called on interceptors (reverse order)
- `ViewResolver.resolveViewName()` called
- `HandlerMapping.getHandler()` returns `HandlerExecutionChain`
- `preHandle()` called on interceptors (forward order)
- `View.render()` called
- `postHandle()` called on interceptors (reverse order)
- `ha.handle()` invoked

**Correct Order:**
1. `HandlerMapping.getHandler()` returns `HandlerExecutionChain`
2. `preHandle()` called on interceptors (forward order)
3. `ha.handle()` invoked
4. `postHandle()` called on interceptors (reverse order)
5. `ViewResolver.resolveViewName()` called
6. `View.render()` called
7. `afterCompletion()` called on interceptors (reverse order)

---

**Q3 — Code Prediction**
Three interceptors registered: A, B, C (in that order). Interceptor B's `preHandle()` returns `false`. Which callbacks are invoked?

A) A.preHandle, B.preHandle, A.afterCompletion
B) A.preHandle, B.preHandle, B.afterCompletion, A.afterCompletion
C) A.preHandle, B.preHandle, C.afterCompletion, B.afterCompletion, A.afterCompletion
D) A.preHandle, B.preHandle (no afterCompletion)

**Answer: A**
When B.preHandle returns false: `applyPreHandle()` calls `triggerAfterCompletion()` with `interceptorIndex=0` (A is at index 0, last successful). `afterCompletion()` is called for interceptors[0..0] → only A. B returned false → B's `afterCompletion` is NOT called. C never had `preHandle` called → C's `afterCompletion` NOT called.

---

**Q4 — Select All That Apply**
`processDispatchResult()` is responsible for which of the following?

A) Invoking `ha.handle()` to execute the controller method
B) Calling `processHandlerException()` when `dispatchException != null`
C) Calling `render()` when a `ModelAndView` with a view name is present
D) Calling `triggerAfterCompletion()` on the success path
E) Cleaning up multipart temp files

**Answer: B, C, D**
A is Phase 6 (doDispatch's inner try). E is Phase 11 (finally block). B, C, D are all done inside `processDispatchResult()`.

---

**Q5 — True/False**
When `throwExceptionIfNoHandlerFound=false` (the default), a `NoHandlerFoundException` is thrown and can be caught by `@ExceptionHandler(NoHandlerFoundException.class)`.

**Answer: False**
With `throwExceptionIfNoHandlerFound=false`, `noHandlerFound()` calls `response.sendError(404)`. No exception is thrown. The response is already committed. `@ExceptionHandler` is NEVER invoked. To make `@ExceptionHandler` work for 404, you must set `throwExceptionIfNoHandlerFound=true` AND disable the default servlet handler.

---

**Q6 — MCQ**
A controller method returns `Callable<Product>`. After `ha.handle()` returns in Phase 6, what is the value of `mv`?

A) `ModelAndView{view="product", model={...}}`
B) `null` — the response was written asynchronously
C) `null` — `CallableMethodReturnValueHandler` starts async and `handle()` returns null
D) `ModelAndView{}` — empty ModelAndView pending async result

**Answer: C**
`CallableMethodReturnValueHandler.handleReturnValue()` calls `asyncManager.startCallableProcessing(callable)`. This sets `concurrentHandlingStarted=true` and submits to executor. `mavContainer.setRequestHandled(false)` remains, but `invokeHandlerMethod()` returns `null` (from `getModelAndView()` when async started). `ha.handle()` returns `null`. Phase 7 then returns from `doDispatch()` immediately.

---

**Q7 — Scenario**
An exception is thrown in `render()` (inside `processDispatchResult()`). What happens?

A) The exception is swallowed — response is sent with whatever was rendered so far
B) The exception propagates to the outer try-catch in `doDispatch()` → `triggerAfterCompletion(exception)` → re-thrown to FrameworkServlet → HTTP 500
C) `processHandlerException()` is called again to handle the render exception
D) `afterCompletion()` is never called because the render failed

**Answer: B**
`render()` is called inside `processDispatchResult()`. If `render()` throws, `processDispatchResult()` re-throws. The outer try-catch in `doDispatch()` catches it → calls `triggerAfterCompletion(request, response, mappedHandler, ex)` → re-throws. `afterCompletion()` IS called (D is wrong). `processHandlerException()` is NOT called again (C is wrong).

---

**Q8 — Select All That Apply**
`cleanupMultipart()` in Phase 11 (finally) is called under which conditions?

A) Only when handler invocation succeeded
B) When an exception was thrown during handler invocation
C) When `preHandle()` returned false (handler never invoked)
D) When async concurrent handling was started
E) When the response was HTTP 304 Not Modified

**Answer: B, C**
`multipartRequestParsed=true` AND `asyncManager.isConcurrentHandlingStarted()=false` triggers `cleanupMultipart()`. For async (D): async branch runs `applyAfterConcurrentHandlingStarted()` instead. For HTTP 304 (E): Phase 4 returns before multipart is parsed (or if multipart was parsed and then 304 detected, cleanup still runs). For preHandle=false (C): Phase 5 returns but `finally` still executes. For exception (B): `finally` always executes. A is wrong — cleanup is NOT limited to success path.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — postHandle() is completely bypassed for exceptions in ha.handle()**
The moment `ha.handle()` throws, `dispatchException` is set and execution jumps to Phase 10 (`processDispatchResult()`). Phases 8 and 9 are **entirely skipped**. Interceptors that rely on `postHandle()` for resource cleanup in the exception path will not run. Use `afterCompletion()` for guaranteed cleanup — it always runs (for interceptors whose preHandle succeeded).

**Trap 2 — afterCompletion() called with null exception on success path**
`triggerAfterCompletion(request, response, null)` is called at the end of `processDispatchResult()`. The `null` argument means "no exception — success path." When an exception propagates to the outer catch, `triggerAfterCompletion(request, response, mappedHandler, ex)` is called with the actual exception. Your `afterCompletion()` implementation can check the `Exception` parameter: if null = success, non-null = failure.

**Trap 3 — throwExceptionIfNoHandlerFound=false makes 404 invisible to @ExceptionHandler**
This is the single most common Spring MVC misconfiguration. With the default value (`false`), `sendError(404)` bypasses the entire Spring exception handling infrastructure. The container handles the error page. Developers write `@ExceptionHandler(NoHandlerFoundException.class)` and wonder why it never fires. Must set `throwExceptionIfNoHandlerFound=true` AND must NOT have a default servlet handler registered (which catches all URLs and returns 404 itself).

**Trap 4 — Multipart cleanup runs even when preHandle returns false**
If interceptor A.preHandle() returns false, `doDispatch()` returns immediately AFTER the `finally` block. If multipart was parsed before Phase 5 ran, `multipartRequestParsed=true`. The `finally` block still cleans up multipart temp files. This is correct and intentional — no temp file leak regardless of how early processing stops.

**Trap 5 — Async path and interceptor lifecycle**
When `Callable<T>` or `DeferredResult<T>` is returned, `doDispatch()` exits at Phase 7. The interceptor `afterCompletion()` is NOT called on the first dispatch. It is called on the SECOND dispatch (when async result is ready and DispatcherServlet processes the result). `postHandle()` is also called on the second dispatch. Developers often add cleanup logic in `afterCompletion()` and are surprised it doesn't run on the first async dispatch.

**Trap 6 — processDispatchResult() always receives dispatchException**
Even though the inner try-catch in `doDispatch()` catches exceptions, it does NOT propagate them directly. It stores them as `dispatchException` and always calls `processDispatchResult()`. The exception is processed inside `processDispatchResult()` via the `HandlerExceptionResolver` chain. This design means `afterCompletion()` is called in `processDispatchResult()` for BOTH success and handled-exception paths — only unhandled exceptions go to the outer catch.

**Trap 7 — render() is called AFTER afterCompletion in exception path**
On the SUCCESS path: render() → afterCompletion(). On the EXCEPTION path: processHandlerException() may return a `ModelAndView` for the error view → render() → afterCompletion(). Both paths render before calling `afterCompletion()`. A common misconception is that `afterCompletion()` runs immediately after the handler method — it runs AFTER everything including view rendering.

---

## 5️⃣ SUMMARY SHEET

```
doDispatch() — 11 PHASES IN ORDER
─────────────────────────────────────────────────────
Phase 1:  checkMultipart()
            → multipartRequestParsed=true if wrapped
            → Exception → dispatchException → Phase 10

Phase 2:  getHandler() → HandlerExecutionChain
            → null + throwException=false → sendError(404) → RETURN
            → null + throwException=true → NoHandlerFoundException → Phase 10

Phase 3:  getHandlerAdapter() — supports(handler)
            → No adapter → ServletException → dispatchException → Phase 10

Phase 4:  Last-Modified check (GET/HEAD only)
            → 304 Not Modified → RETURN (all phases skipped)

Phase 5:  applyPreHandle() — FORWARD order
            → false returned → triggerAfterCompletion(prior interceptors) → RETURN

Phase 6:  ha.handle() → mv (null for @ResponseBody)
            → Exception → dispatchException → SKIP 7,8,9 → Phase 10

Phase 7:  asyncManager.isConcurrentHandlingStarted()
            → true → finally runs (applyAfterConcurrentHandlingStarted) → RETURN

Phase 8:  applyDefaultViewName() — derive from URL if mv≠null, no view name

Phase 9:  applyPostHandle() — REVERSE order
            → Exception → dispatchException → Phase 10

Phase 10: processDispatchResult(req, res, mappedHandler, mv, dispatchException)
            → Exception path: processHandlerException() → resolver chain
                → resolved: render error view OR null (fully handled)
                → unresolved: re-throw → outer catch → triggerAfterCompletion(ex) → throw
            → Success path: render(mv) if mv≠null
            → ALWAYS: triggerAfterCompletion(null) on success

Phase 11: finally — ALWAYS
            → Async: applyAfterConcurrentHandlingStarted()
            → Sync: cleanupMultipart() if multipartRequestParsed

POSTHANDLE() BYPASS CONDITIONS — ALL OF:
─────────────────────────────────────────────────────
preHandle() returned false      → Phase 5 returns early
Exception in ha.handle()        → dispatchException set, Phase 9 skipped
Async started                   → Phase 7 returns early
HTTP 304 Not Modified           → Phase 4 returns early
→ Use afterCompletion() for guaranteed cleanup

AFTERCOMPLETION() GUARANTEE
─────────────────────────────────────────────────────
Called for: interceptors[0..interceptorIndex] (those whose preHandle succeeded)
Called with: null = success path (in processDispatchResult)
             exception = unhandled exception (in outer catch)
Exceptions in afterCompletion: SWALLOWED — logged only
NEVER called for interceptors whose preHandle failed or never ran

EXCEPTION FLOW SUMMARY
─────────────────────────────────────────────────────
Exception in ha.handle():
  → dispatchException → processHandlerException()
  → ExceptionHandlerExceptionResolver (@ExceptionHandler)
    → @Controller local first, then @ControllerAdvice
  → ResponseStatusExceptionResolver (@ResponseStatus on exception class)
  → DefaultHandlerExceptionResolver (standard Spring exceptions → status only)
  → Not handled → re-throws → outer catch → afterCompletion(ex) → HTTP 500

Exception in render():
  → outer catch → triggerAfterCompletion(ex) → re-throws → HTTP 500

Exception in afterCompletion():
  → SWALLOWED — logged, response not affected

noHandlerFound() BEHAVIOUR
─────────────────────────────────────────────────────
throwExceptionIfNoHandlerFound=false (DEFAULT):
  → sendError(404) → container handles → @ExceptionHandler NEVER fires

throwExceptionIfNoHandlerFound=true:
  → NoHandlerFoundException → processHandlerException() → CAN be handled

MULTIPART CLEANUP
─────────────────────────────────────────────────────
Condition: multipartRequestParsed=true AND NOT async
Timing: ALWAYS in finally — even on exception, even on preHandle=false
Purpose: Delete temp files — never leak regardless of outcome

ASYNC INTERCEPTOR LIFECYCLE
─────────────────────────────────────────────────────
First dispatch (Callable returned):
  preHandle() called ✓
  postHandle() NOT called (Phase 7 returns early)
  afterCompletion() NOT called (Phase 7 returns early)
  afterConcurrentHandlingStarted() called in finally

Second dispatch (async result ready):
  preHandle() called again ✓
  postHandle() called ✓
  afterCompletion() called ✓

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "doDispatch() has 11 phases — from multipart check to cleanup"
• "postHandle() skipped for exceptions, async, false preHandle, 304"
• "afterCompletion() always runs for successful preHandle interceptors"
• "processDispatchResult() handles both rendering and exception resolution"
• "throwExceptionIfNoHandlerFound=false: 404 bypasses @ExceptionHandler entirely"
• "multipart cleanup in finally — always runs regardless of exception"
• "mv=null from ha.handle() means @ResponseBody wrote response — no view"
• "Async first dispatch: postHandle and afterCompletion called on SECOND dispatch"
• "Exception in render() propagates to outer catch — afterCompletion called with it"
• "afterCompletion exceptions are swallowed — never propagated to response"
```

---

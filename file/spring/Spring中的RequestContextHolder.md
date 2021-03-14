### Spring中的RequestContextHolder

本质上就是一个基于ThreadLocal实现的存放请求信息的容器，再加上一般Tomcat整个请求都是在一个线程中处理完，所以可以比较方便的获取当前线程中的请求信息，不过要注意如果中途有操作是其他线程来处理逻辑，就没法取到请求信息，源码如下。

```
public abstract class RequestContextHolder {
    private static final boolean jsfPresent = ClassUtils.isPresent("javax.faces.context.FacesContext", RequestContextHolder.class.getClassLoader());
    private static final ThreadLocal<RequestAttributes> requestAttributesHolder = new NamedThreadLocal("Request attributes");
    private static final ThreadLocal<RequestAttributes> inheritableRequestAttributesHolder = new NamedInheritableThreadLocal("Request context");

    public RequestContextHolder() {
    }

    public static void resetRequestAttributes() {
        requestAttributesHolder.remove();
        inheritableRequestAttributesHolder.remove();
    }

    public static void setRequestAttributes(@Nullable RequestAttributes attributes) {
        setRequestAttributes(attributes, false);
    }

    public static void setRequestAttributes(@Nullable RequestAttributes attributes, boolean inheritable) {
        if (attributes == null) {
            resetRequestAttributes();
        } else if (inheritable) {
            inheritableRequestAttributesHolder.set(attributes);
            requestAttributesHolder.remove();
        } else {
            requestAttributesHolder.set(attributes);
            inheritableRequestAttributesHolder.remove();
        }

    }

    @Nullable
    public static RequestAttributes getRequestAttributes() {
        RequestAttributes attributes = (RequestAttributes)requestAttributesHolder.get();
        if (attributes == null) {
            attributes = (RequestAttributes)inheritableRequestAttributesHolder.get();
        }

        return attributes;
    }

    public static RequestAttributes currentRequestAttributes() throws IllegalStateException {
        RequestAttributes attributes = getRequestAttributes();
        if (attributes == null) {
            if (jsfPresent) {
                attributes = RequestContextHolder.FacesRequestAttributesFactory.getFacesRequestAttributes();
            }

            if (attributes == null) {
                throw new IllegalStateException("No thread-bound request found: Are you referring to request attributes outside of an actual web request, or processing a request outside of the originally receiving thread? If you are actually operating within a web request and still receive this message, your code is probably running outside of DispatcherServlet: In this case, use RequestContextListener or RequestContextFilter to expose the current request.");
            }
        }

        return attributes;
    }

    private static class FacesRequestAttributesFactory {
        private FacesRequestAttributesFactory() {
        }

        @Nullable
        public static RequestAttributes getFacesRequestAttributes() {
            FacesContext facesContext = FacesContext.getCurrentInstance();
            return facesContext != null ? new FacesRequestAttributes(facesContext) : null;
        }
    }
}
```

这是FrameworkServlet类中的一个方法，它也是DispatcherServlet的父类，主要是```this.initContextHolders(request, localeContext, requestAttributes);```这行，把当前的请求信息放到容器中。

```
    protected final void processRequest(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        long startTime = System.currentTimeMillis();
        Throwable failureCause = null;
        LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
        LocaleContext localeContext = this.buildLocaleContext(request);
        RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
        ServletRequestAttributes requestAttributes = this.buildRequestAttributes(request, response, previousAttributes);
        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
        asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new FrameworkServlet.RequestBindingInterceptor());
        this.initContextHolders(request, localeContext, requestAttributes);

        try {
            this.doService(request, response);
        } catch (IOException | ServletException var16) {
            failureCause = var16;
            throw var16;
        } catch (Throwable var17) {
            failureCause = var17;
            throw new NestedServletException("Request processing failed", var17);
        } finally {
            this.resetContextHolders(request, previousLocaleContext, previousAttributes);
            if (requestAttributes != null) {
                requestAttributes.requestCompleted();
            }

            this.logResult(request, response, (Throwable)failureCause, asyncManager);
            this.publishRequestHandledEvent(request, response, startTime, (Throwable)failureCause);
        }

    }
```

依赖上面的特性可以随时获取Request里中的信息，避免把HttpRequest参数逐层传递。
```
public class RequestHeaderUtil {

    /**
     * 获取request
     *
     * @return
     */
    public static HttpServletRequest getRequest() {
        ServletRequestAttributes requestAttributes = (ServletRequestAttributes) RequestContextHolder
                .getRequestAttributes();
        return requestAttributes == null ? null : requestAttributes.getRequest();
    }

    public static JSONObject getUserInfo() {
        String userInfo = getRequest().getHeader("userInfo");
        if (StringUtils.isBlank(userInfo)) {
            return new JSONObject();
        }
        try {
            return JSON.parseObject(URLDecoder.decode(userInfo, "UTF-8"));
        } catch (UnsupportedEncodingException e) {
            Log.error("+++>从header获取用户信息，进行解码时异常，Errmsg->", e);
            return new JSONObject();
        }
    }
}
```
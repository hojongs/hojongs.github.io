---
layout: post
title: "[TIL] Spring MVC: Filter vs HandlerInterceptor"
categories: [Spring MVC]
tags: [Spring MVC, TIL, Filter, HandlerInterceptor]
---

## Filter

아래 Spring framework 링크에는 몇몇 유용한 spring-web의 Filter 구현체들을 소개한다. Filter 자체에 대해서는 소개하지 않는다.

<https://docs.spring.io/spring-framework/reference/web/webmvc/filters.html>

Filter에 대한 Oracle 문서는 아래를 찾았다 (Filter는 Java Servlet에서 제공하는 interface다)

- <https://docs.oracle.com/javaee/7/tutorial/servlets006.htm>
- <https://docs.oracle.com/javaee/7/api/javax/servlet/Filter.html>
- <https://javaee.github.io/javaee-spec/javadocs/javax/servlet/Filter.html>

문서에서도 Filter를 인증, 로깅 등의 역할로 사용하도록 가이드하고 있다.

## HandlerInterceptor

- <https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet/handlermapping-interceptor.html>
- <https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/interceptors.html>
- <https://www.baeldung.com/spring-mvc-handlerinterceptor>

로깅은 HandlerInterceptor든 Filter든 어느 쪽에서도 구현할 수 있다

HandlerInterceptor 문서만 읽어봐서는 Filter와 어떻게 다르게 사용해야 하는지 이해가 잘 안됐다.

우선 HandlerInterceptor의 특징은 2가지라고 볼 수 있다

- DispatcherServlet 이후에 실행된다는 점
- Java Servlet이 아니라 Spring level에서 정의된 interface라는 점

## Filter vs HandlerInterceptor

아래 문서에 그림과 함께 둘을 비교하고 있다

<https://www.baeldung.com/spring-mvc-handlerinterceptor-vs-filter>

Filter

```java
public interface Filter {
    public void doFilter(ServletRequest request, ServletResponse response,
        FilterChain chain) throws IOException, ServletException;
}
```

HandlerInterceptor

```java
public interface HandlerInterceptor {
    default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
        throws Exception {

        return true;
    }

    default void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
        @Nullable ModelAndView modelAndView) throws Exception {
    }

    default void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler,
        @Nullable Exception ex) throws Exception {
    }
}
```

위 링크에서는 HandlerInterceptor가 handler와 modelAndView에 대한 접근이 가능하기 때문에, 좀더 상세한 구현을 할 때 사용하라고 설명하고 있따.

하지만 개인적으로 Spring에서 API server를 개발하면서 handler, modelAndView 객체에 접근할만한 use case는 아직 만나보지 못했다.

즉, Filter가 아닌 HandlerInterceptor에 구현해야만 했던 use case는 아직 없었다.

**특별한 이유가 없다면 Filter 구현을 먼저 고려해보도록 하자.**

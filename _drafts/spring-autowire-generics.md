---
layout: post
title: "Spring Autowire Generics"
summary:
categories: spring
comments: true
---

Spring has supported generic autowiring since version 3.2 and in Spring 4.0 they enhanced generic supported
so that the generic type acted as a form of @Qualifier. A good summary can be found at 
https://spring.io/blog/2013/12/03/spring-framework-4-0-and-java-generics.

You can hit an issue when trying to autowire a dependency that uses @Transactional (Note: there might be other
annotations. Any annotation that causes a proxy to be created). The problem arises if you aren't using
CGLIB because Spring will resolve a JDK dynamic proxy

// TODO: add code blocks (links to github or gists) that points to issue
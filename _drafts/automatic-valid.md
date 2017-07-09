---
layout: post
title: "Spring Automatic Validation"
categories: Spring Validation
comments: false
summary: 
---

I stumbled upon this Stack Overflow question the other day <https://stackoverflow.com/questions/44722000/how-to-spring-get-rid-of-validate-for-automatic-controller-validation> which is essentially a duplicate of 
<https://stackoverflow.com/questions/26997281/spring-mvc-valid-annotation-for-all-controllers>. 
The only answer to both of these questions, when I started to write this up, is that you cannot omit the `@Valid` annotation. 
How this has an upvote ¯\\_(ツ)\_/¯ but moving on. 
A comment on the latter question suggests that this can be accomplished by using aspects.
While aspects could likely solve this I personally think AOP would be an over-engineered solution to this particular problem. 

For the record I do not believe this is an problem nor do I believe `@Valid` is an example of code duplication as one submitter suggested.

Consider the following code

```java
@PostMapping(value = "/contact-with-valid")
public String contactWithValid(@Valid ContactRequest request, BindingResult results, Model model) {
    model.addAttribute("bindingResults", results);
    return "";
}
```

where `ContactRequest` is defined as 

```java
public class ContactRequest {

    @NotBlank
    private String name;
    
    @NotBlank
    @Email
    private String email;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }  
}
```

with a corresponding test

```java
@Test
public void contactWithValid() throws Exception {
    MvcResult result = mvc.perform(post("/contact-with-valid"))
            .andDo(print())
            .andReturn();
    Map<String, Object> model = result.getModelAndView().getModel();
    assertTrue(model.containsKey("bindingResults"));
    BindingResult results = (BindingResult) model.get("bindingResults");
    assertEquals(2, results.getErrorCount());
}
```

Before we dive into a solution lets first attempt to understand how Spring's `@Valid` annotation works.
Validation kicks in as part of the model binding process which in this specific case is handled by [ServletModelAttributeMethodProcessor's](http://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/ServletModelAttributeMethodProcessor.html) `resolveArgument` which is inherited from [ModelAttributeMethodProcessor](https://github.com/spring-projects/spring-framework/blob/master/spring-web/src/main/java/org/springframework/web/method/annotation/ModelAttributeMethodProcessor.java).
`ModelAttributeMethodProcessor.resolveArgument` in turn calls  `validateIfApplicable` to perform validation at the time of writing looks like the following

```java
/**
* Validate the model attribute if applicable.
* <p>The default implementation checks for {@code @javax.validation.Valid},
* Spring's {@link org.springframework.validation.annotation.Validated},
* and custom annotations whose name starts with "Valid".
* @param binder the DataBinder to be used
* @param parameter the method parameter declaration
*/
protected void validateIfApplicable(WebDataBinder binder, MethodParameter parameter) {
    Annotation[] annotations = parameter.getParameterAnnotations();
    for (Annotation ann : annotations) {
        Validated validatedAnn = AnnotationUtils.getAnnotation(ann, Validated.class);
        if (validatedAnn != null || ann.annotationType().getSimpleName().startsWith("Valid")) {
            Object hints = (validatedAnn != null ? validatedAnn.value() : AnnotationUtils.getValue(ann));
            Object[] validationHints = (hints instanceof Object[] ? (Object[]) hints : new Object[] {hints});
            binder.validate(validationHints);
            break;
        }
    }
}
```


<!--
// configuration setup call
WebMvcAutoConfiguration.requestMappingHandlerAdapter
WebMvcConfigurationSupport.requestMappingHandlerAdapter
RequestMappingHandlerAdapter.setCustomArgumentResolvers(getArgumentResolvers())
WebMvcConfigurationSupport.requestMappingHandlerAdapter
WebMvcConfigurationSupport.getArgumentResolvers
DelegatingWebMvcConfiguration.addArgumentResolvers / WebMvcAutoConfiguration
WebMvcConfigurerComposite.addArgumentResolvers
AutomaticControllerValidationApplication 

AutomaticControllerValidationApplication is a WebMvcConfigurer delegate

HandlerMethodArgumentResolverComposite.getArgumentResolver

WebMvcConfigurerComposite.requestMappingHandlerAdapter



TODO: more content around model binding. Who calls ModelAttributeMethodProcessor?
ModelAndViewResolverMethodReturnValueHandler?  spring-webmvc/src/main/java/org/springframework/web/servlet/mvc/method/annotation/ModelAndViewResolverMethodReturnValueHandler.java
ServletModelAttributeMethodProcessor?
spring-webmvc/src/main/java/org/springframework/web/servlet/mvc/method/annotation/ServletModelAttributeMethodProcessor.java
RequestMappingHandlerAdapter? spring-webmvc/src/main/java/org/springframework/web/servlet/mvc/method/annotation/RequestMappingHandlerAdapter.java

DispatcherServlet.doDispatch
RequestMappingHandlerAdapter.handle -> handleInternal -> invokeHandlerMethod (handlermethod is controller method, ModelFactory.initModel has invocableMethod which has HandlerMethodReturnValueHandlers of HandlerMethodArgumentResolverComposite)
ServletInvocableHandlerMethod.invokeAndHandle then invokeForRequest then getMethodARgumentValues
HandlerMethodArgumentResolverComposite.resolveArgument
ServletModelAttributeMethodProcessor.resolveArgument
-->

Now that we have some understanding regarding how Spring `@Valid` works a solution reveals itself.
We can provide our own custom `HandlerMethodArgumentResolver` by extending either `ServletModelAttributeMethodProcessor` or `ModelAttributeMethodProcessor` and overriding `validateIfApplicable`.

```java
public class AutomaticValidatonModelAttributeMethodProcessor extends ServletModelAttributeMethodProcessor {

    public AutomaticValidatonModelAttributeMethodProcessor(boolean annotationNotRequired) {
        super(annotationNotRequired);
    }

    @Override
    protected void validateIfApplicable(WebDataBinder binder, MethodParameter methodParam) {
        binder.validate(getValidationHints(methodParam));
    }

    private static Object[] getValidationHints(MethodParameter methodParam) {
        Annotation[] annotations = methodParam.getParameterAnnotations();
        for (Annotation ann : annotations) {
            Validated validatedAnn = AnnotationUtils.getAnnotation(ann, Validated.class);
            return validatedAnn != null 
                    ? validatedAnn.value()
                    : new Object[0];
        }
        return new Object[0];
    }  
}
```

Now that we have our custom HandlerMethodArgumentResolver we need to register it as an argument resolver in our application configuration.
We do that by first having our application configuration extend `WebMvcConfigurerAdapter` which will expose `addArgumentResolvers` which we can override and add our `AutomaticValidatonModelAttributeMethodProcessor`. 

A common trap that I've fallen into is to add it as an argument resolver. problem being if you use @ModelAttribute...add test to showcase issue

```java
@SpringBootApplication
public class AutomaticControllerValidationApplication extends WebMvcConfigurerAdapter {

    public static void main(String[] args) {
        SpringApplication.run(AutomaticControllerValidationApplication.class, args);
    }

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
        argumentResolvers.add(new AutomaticValidatonModelAttributeMethodProcessor(true));
    }
}
```

Our first test should still be passing so let's go ahead and add a new endpoint and coresponding test to verify validation is still triggered when `@Valid` is not present.

```java
@PostMapping(value = "/contact-without-valid")
public String contactWithoutValid(ContactRequest request, BindingResult results, Model model) {
    model.addAttribute("bindingResults", results);
    return "";
}


@Test
public void contactWithoutValid() throws Exception {
    MvcResult result = mvc.perform(post("/contact-without-valid"))
            .andDo(print())
            .andReturn();
    Map<String, Object> model = result.getModelAndView().getModel();
    assertTrue(model.containsKey("bindingResults"));
    BindingResult results = (BindingResult) model.get("bindingResults");
    assertEquals(2, results.getErrorCount());
}
```

As pointed out in the first Stack Overflow question this doesn't work when using `@RequestBody`.
Lets write a test to verify things aren't quite what they should be and confirm we have a failing test

```java
@Test
public void requestBodyContact() throws Exception {
    mvc.perform(post("/contact-request-body")
            .contentType(MediaType.APPLICATION_JSON)
            .accept(MediaType.APPLICATION_JSON)
            .content("{\"name\":\"\"}"))
            .andDo(print())
            .andExpect(jsonPath("$", hasSize(2)));
}
```

<!-- TODO: reword this?? -->
So what the heck is happening? 
Spring determines how to resolve controller method arguments via a combination of `HandlerMethodArgumentResolver.supportsParameter` and the order in which they are registered.
How to transition to RequestMappingHandlerAdapter, getDefaultArgumentResolvers, and getCustomArgumentResolvers 
mention that custom argument resolvers are registered after default and include catch all resolvers. this would include why custom servlet model attribute resolver works but why request response body doesn't

By adding the `@RequestBody` annotation our `AutomaticValidatonModelAttributeMethodProcessor` resolver no longer gets selected and instead the `RequestResponseBodyMethodProcessor` argument resolver is selected as its registered first and specifically supports resolving method arguments annotated with `@RequestBody` and handles return values from methods annotated with `@ResponseBody`.

Reviewing `RequestResponseBodyMethodProcessor` we see that it too has a `validateIfApplicable` method only this time inherited from `AbstractMessageConverterMethodArgumentResolver`.
Perfect! We just need to do the same thing we did previously. 
Let's create another custom `HandlerMethodArgumentResolver` class but this time extending `RequestResponseBodyMethodProcessor`.

```java
public class AutomaticValidationRequestResponseBodyMethodProcessor extends RequestResponseBodyMethodProcessor {

    public AutomaticValidationRequestResponseBodyMethodProcessor(List<HttpMessageConverter<?>> converters) {
        super(converters);
    }

    @Override
    protected void validateIfApplicable(WebDataBinder binder, MethodParameter methodParam) {
        binder.validate(getValidationHints(methodParam));
    }

    private static Object[] getValidationHints(MethodParameter methodParam) {
        Annotation[] annotations = methodParam.getParameterAnnotations();
        for (Annotation ann : annotations) {
            Validated validatedAnn = AnnotationUtils.getAnnotation(ann, Validated.class);
            return validatedAnn != null 
                    ? validatedAnn.value()
                    : new Object[0];
        }
        return new Object[0];
    }
}
```
Again we need to register this argument resolver within our application configuration. So let's go ahead and do that

```java
@SpringBootApplication
public class AutomaticControllerValidationApplication extends WebMvcConfigurerAdapter {

    @Autowired
    private List<HttpMessageConverter<?>> converters;
    
    public static void main(String[] args) {
        SpringApplication.run(AutomaticControllerValidationApplication.class, args);
    }

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
        argumentResolvers.add(new AutomaticValidatonModelAttributeMethodProcessor(true));
        argumentResolvers.add(new AutomaticValidationRequestResponseBodyMethodProcessor(converters));
    }
}
```
We run our test again....and it still fails!
Why does Spring using our `AutomaticValidatonModelAttributeMethodProcessor` but not using our `AutomaticValidationRequestResponseBodyMethodProcessor`?
Again it comes back to what `HandlerMethodArgumentResolver` support and when it gets registered.
Looking at the order things are registered

1. Annotation-based argument resolution
2. Type-based argument resolution
3. Custom arguments
4. Catch-all

A default `ServletModelAttributeMethodProcessor` is registered as part of the annotation-based resolvers however that only supports arguments specifically annotated with `@ModelAttribute` which is why its skipped and our custom `AutomaticValidatonModelAttributeMethodProcessor` is selected.
TODO: determine why when using @ModelAttribute our custom one is still selected instead of default.   --> it doesn't
The default `RequestResponseBodyMethodProcessor` on the other hand will always be selected thus we cannot simply use the `addARgumentResolvers` method to override the default behavior.

RequestMappingHandlerAdapter
getDefaultArgumentResolvers
getCustomArgumentResolvers


Because of the way RequestMappingHandlerAdapter registers HandlerMethodArgumentResolver






You can find the source code on GitHub here TODO: add link to repo



When using @RequestBody

<!--
TestDispatcherServlet.init
TestDispatcherServlet.initServletBean
TestDispatcherServlet.initWebApplicationContext
TestDispatcherServlet.onRefresh  
TestDispatcherServlet.initStrategies 
TestDispatcherServlet.initHandlerMappings -> Map<String, HandlerMapping> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
    AnnotationConfigEmbeddedWebApplicationContext.getBeansOfType -> DefaultListableBeanFactory concrete type AnnotationConfigEmbeddedWebApplicationContext
        -> type interface org.springframework.web.servlet.HandlerMapping
        -> type org.springframework.web.servlet.handler.MappedInterceptor
    RequestMappingHandlerMapping.setApplicationContext
    RequestMappingHandlerMapping.initApplicationContext
    RequestMappingHandlerMapping.detectMappedInterceptors
    WebMvcConfigurationSupport
TestDispatcherServlet.initHandlerAdapters -> Map<String, HandlerAdapter> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);
    AnnotationConfigEmbeddedWebApplicationContext.getBeansOfType -> DefaultListableBeanFactory concrete type 
    RequestMappingHandlerMapping.setApplicationContext
    RequestMappingHandlerMapping.initApplicationContext
    RequestMappingHandlerMapping.initServletContext
    RequestMappingHandlerAdapter.afterPropertiesSet
        RequestMappingHandlerAdapter.getDefaultArgumentResolvers
            -> resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		set initBinderArgumentResolvers -> new HandlerMethodArgumentResolverComposite().addResolvers(resolvers); where resolvers are default

AbstractApplicationContext
RequestMappingHandlerAdapter.getDefaultReturnValueHandlers
RequestResponseBodyMethodProcessor
-->

Option 1

how to custom RequestResponseBodyMethodProcessor / override validateIfApplicable

To achieve what you're trying to do you need to access the RequestMappingHandlerAdapter created by <annotation-driven/> and customize it. You can do that with a BeanPostProcessor. Sample code is shown below. You can choose whether to declare the BeanPostProcessor in XML or with annotations, whether to make it configurable with setters, etc:


Option 2

extends WebMvcConfigurationSupport

Looks like prior to WebMvcConfigurationSupport you had to use a bean post processor





https://stackoverflow.com/questions/22310127/how-to-remove-redundant-spring-mvc-method-by-providing-post-only-valid

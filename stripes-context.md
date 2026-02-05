# Stripes Framework - Deep Dive Context for Copilot

## How Stripes Routing Works (vs Spring MVC)

In Spring MVC, you define routes with annotations on methods:
```java
@GetMapping("/users/{id}")
public ResponseEntity<User> getUser(@PathVariable Long id) { ... }
```

In Stripes, the URL is bound at the **class level**, and individual methods are selected by **event name**:
```java
@UrlBinding("/users/{id}")
public class UserActionBean implements ActionBean {

    private ActionBeanContext context;
    private Long id;           // auto-bound from URL
    private User user;         // populated in handler, available to JSP

    @DefaultHandler
    public Resolution view() {
        user = userService.findById(id);
        return new ForwardResolution("/WEB-INF/jsp/user/view.jsp");
    }

    @HandlesEvent("edit")
    public Resolution edit() {
        user = userService.findById(id);
        return new ForwardResolution("/WEB-INF/jsp/user/edit.jsp");
    }

    @HandlesEvent("save")
    public Resolution save() {
        userService.save(user);
        return new RedirectResolution(UserActionBean.class).addParameter("id", user.getId());
    }

    // Getters and Setters required for all bound fields
    public ActionBeanContext getContext() { return context; }
    public void setContext(ActionBeanContext context) { this.context = context; }
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public User getUser() { return user; }
    public void setUser(User user) { this.user = user; }
}
```

### How events get triggered

From a JSP form:
```html
<stripes:form beanclass="com.example.UserActionBean">
    <stripes:text name="user.name"/>
    <stripes:submit name="save" value="Save"/>   <!-- triggers @HandlesEvent("save") -->
</stripes:form>
```

From a URL:
```
/users/42?_eventName=edit    <!-- triggers @HandlesEvent("edit") -->
/users/42                     <!-- triggers @DefaultHandler -->
```

From JavaScript (AJAX):
```javascript
$.post("/users/42", { _eventName: "save", "user.name": "New Name" });
```

---

## Parameter Binding Deep Dive

Stripes uses **convention-based parameter binding**. If the request has a parameter named `user.name`, Stripes will:
1. Find the `user` field on the ActionBean.
2. Call `getUser()` (creates one if null in some configurations).
3. Call `setName("value")` on the User object.

This works for nested objects, lists, and maps:
```
user.name           -> getUser().setName(value)
user.address.city   -> getUser().getAddress().setCity(value)
items[0].name       -> getItems().get(0).setName(value)
map['key']          -> getMap().put("key", value)
```

**Spring equivalent mental model:**
Think of it like Spring's `@ModelAttribute` binding but applied automatically to every field on the ActionBean that has a getter/setter.

---

## Configuration: web.xml (not application.properties)

Stripes is configured through the servlet filter in `web.xml`, not through annotation scanning or properties files like Spring Boot:

```xml
<filter>
    <filter-name>StripesFilter</filter-name>
    <filter-class>net.sourceforge.stripes.controller.StripesFilter</filter-class>
    <init-param>
        <param-name>ActionResolver.Packages</param-name>
        <param-value>com.example.web</param-value>  <!-- scans this package for ActionBeans -->
    </init-param>
    <init-param>
        <param-name>Interceptor.Classes</param-name>
        <param-value>
            net.sourceforge.stripes.integration.spring.SpringInterceptor,
            com.example.security.AuthInterceptor
        </param-value>
    </init-param>
</filter>
```

The `SpringInterceptor` is what enables `@SpringBean` injection into ActionBeans.

---

## Resolution Types Reference

| Resolution | Purpose | Spring Equivalent |
|---|---|---|
| `ForwardResolution("/path.jsp")` | Render a JSP | `return "viewName"` |
| `RedirectResolution(Bean.class)` | Browser redirect (PRG pattern) | `return "redirect:/url"` |
| `StreamingResolution("application/json", string)` | AJAX/JSON response | `@ResponseBody` return |
| `StreamingResolution("text/csv", inputStream)` | File download | Writing to response OutputStream |
| `JavaScriptResolution(jsCode)` | Execute JS on client | Not common in Spring |
| `ErrorResolution(404, "Not found")` | HTTP error | `ResponseEntity.status(404)` |

---

## Stripes Tag Library (JSP)

Stripes provides its own JSP tags. These are NOT JSTL, NOT Spring form tags:

```jsp
<%@ taglib prefix="stripes" uri="http://stripes.sourceforge.net/stripes.tld" %>

<stripes:form beanclass="com.example.MyActionBean">
    <stripes:text name="fieldName"/>                <!-- text input -->
    <stripes:password name="password"/>             <!-- password input -->
    <stripes:textarea name="description"/>          <!-- textarea -->
    <stripes:select name="status">                  <!-- dropdown -->
        <stripes:option value="ACTIVE">Active</stripes:option>
        <stripes:option value="INACTIVE">Inactive</stripes:option>
    </stripes:select>
    <stripes:checkbox name="active"/>               <!-- checkbox -->
    <stripes:hidden name="id"/>                     <!-- hidden field -->
    <stripes:submit name="save" value="Save"/>      <!-- submit triggers event -->
    <stripes:errors/>                               <!-- displays validation errors -->
</stripes:form>

<stripes:link beanclass="com.example.MyActionBean" event="view">
    <stripes:param name="id" value="${item.id}"/>
    View Item
</stripes:link>

<stripes:messages/>  <!-- flash/info messages -->
```

---

## Validation in Stripes

**Annotation-based (like Bean Validation but Stripes-native):**
```java
@Validate(required = true, minlength = 2, maxlength = 100)
private String name;

@Validate(required = true, mask = "^[\\w.]+@[\\w.]+$")
private String email;

@Validate(required = true, minvalue = 0, maxvalue = 150)
private Integer age;
```

**Custom validation method (runs after field-level validation passes):**
```java
@ValidationMethod(on = {"save", "update"})
public void customValidation(ValidationErrors errors) {
    if (startDate != null && endDate != null && startDate.after(endDate)) {
        errors.addGlobalError(new SimpleError("Start date must be before end date."));
    }
}
```

**Key difference from Spring:** In Spring, you'd use `@Valid` on the method parameter plus Bean Validation annotations on the DTO. In Stripes, the ActionBean itself IS the DTO, and `@Validate` is Stripes-native (not javax.validation).

---

## Interceptors (like Spring HandlerInterceptor)

```java
@Intercepts({LifecycleStage.HandlerResolution, LifecycleStage.EventHandling})
public class AuthInterceptor implements Interceptor {

    @Override
    public Resolution intercept(ExecutionContext ctx) throws Exception {
        ActionBean bean = ctx.getActionBean();
        HttpServletRequest request = bean.getContext().getRequest();

        if (requiresAuth(bean) && !isLoggedIn(request)) {
            return new RedirectResolution("/login");
        }

        return ctx.proceed();  // continue the chain
    }
}
```

Lifecycle stages in order:
1. `RequestInit` - request setup
2. `ActionBeanResolution` - find the right ActionBean
3. `HandlerResolution` - find the right handler method
4. `BindingAndValidation` - bind params and validate
5. `CustomValidation` - run @ValidationMethod methods
6. `EventHandling` - execute the handler method
7. `ResolutionExecution` - execute the returned Resolution

---

## Common Gotchas for Spring Developers

1. **No automatic JSON deserialization.** Stripes does not have `@RequestBody`. For AJAX POST with JSON body, you need to read the request InputStream manually or configure a custom TypeConverter. Most Stripes apps send data as form parameters, not JSON request bodies.

2. **ActionBeans are NOT singletons.** A new instance is created per request. Do not store request-scoped state in static fields or shared references.

3. **No component scanning for services.** Stripes only scans for ActionBeans. Your service layer should still be managed by Spring (if using Spring integration) and injected via `@SpringBean`.

4. **Getter/Setter required.** Unlike Spring's constructor injection or Lombok-powered patterns, Stripes REQUIRES traditional getter/setter pairs for every field that participates in binding.

5. **Event name from the submit button.** The `name` attribute of the submit button (not the form action) determines which handler method runs. This is different from Spring where the URL path or HTTP method determines routing.

6. **`_eventName` parameter.** For AJAX calls, you pass the event name as the `_eventName` request parameter. There is no URL-based method routing like `/users/save`.

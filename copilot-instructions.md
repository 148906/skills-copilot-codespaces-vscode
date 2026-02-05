# Copilot Instructions

## Developer Profile

I am a Java Spring / Spring Boot developer working on a legacy application built with **Java Stripes Framework** and **Knockout.js** on the frontend. I am very comfortable with Spring conventions (dependency injection, annotations like `@Autowired`, `@RestController`, `@Service`, etc.) but I need Copilot to help me bridge the gap to Stripes patterns, which differ significantly.

**Always explain Stripes concepts by drawing parallels to Spring MVC equivalents wherever possible.**

---

## Technology Stack

- **Backend:** Java Stripes Framework (NOT Spring MVC)
- **Frontend:** Knockout.js (MVVM pattern, NOT React/Angular/Vue)
- **Build:** Maven
- **Server:** Typically runs on Tomcat / Servlet container
- **Java Version:** Java 8+ (check pom.xml for exact version)

---

## Java Stripes Framework Guidelines

### Core Concepts (mapped to Spring equivalents)

| Stripes Concept | Spring Equivalent | Notes |
|---|---|---|
| `ActionBean` | `@Controller` class | The fundamental request handler in Stripes |
| `@UrlBinding` | `@RequestMapping` | Maps a URL pattern to an ActionBean |
| `@DefaultHandler` | Method with `@GetMapping("/")` | The method invoked when no specific event is named |
| `@HandlesEvent("eventName")` | `@PostMapping("/eventName")` | Routes a specific form submission or action to a method |
| `Resolution` (return type) | `ModelAndView` or `ResponseEntity` | What the handler method returns to control the response |
| `ForwardResolution` | Returning a view name / `ModelAndView` | Forwards to a JSP |
| `RedirectResolution` | `RedirectView` or `redirect:` prefix | Redirects the browser to another URL |
| `StreamingResolution` | Writing directly to `HttpServletResponse` | Used for JSON/AJAX responses |
| `ActionBeanContext` | `HttpServletRequest` + `HttpSession` wrapper | Access to request, response, session, messages |
| `@Validate` on fields | `@Valid` + `@NotNull` / `@Size` etc. | Stripes binds form params directly to ActionBean fields |
| `TypeConverter` | `@InitBinder` / `Converter` | Custom type conversion from request params |
| `Interceptor` | `HandlerInterceptor` | Cross-cutting concerns (auth, logging) |
| `@SpringBean` | `@Autowired` | Injects a Spring-managed bean INTO a Stripes ActionBean |

### ActionBean Lifecycle (important differences from Spring)

1. Stripes creates a **new ActionBean instance per request** (unlike Spring singleton controllers).
2. Request parameters are **bound directly to ActionBean fields** (setter-based). There is no `@RequestParam` or `@RequestBody` annotation.
3. Validation happens via `@Validate` annotations on fields or through `ValidationMethod` custom methods.
4. The handler method runs and returns a `Resolution`.
5. The `Resolution` determines what happens next (forward to JSP, redirect, stream JSON, etc.).

### When generating or suggesting Stripes code, follow these rules:

- ActionBeans must implement `net.sourceforge.stripes.action.ActionBean`.
- Always include `getContext()` and `setContext()` methods (or extend `BaseActionBean` if the project has one).
- Use `@UrlBinding("/some/path")` at the class level.
- Handler methods must return a `Resolution` (never `void`, `String`, or `ResponseEntity`).
- For AJAX/JSON responses, use `StreamingResolution("application/json", jsonString)`.
- Use `@Validate(required=true, ...)` on fields instead of separate validation classes.
- If the project uses `@SpringBean` for DI, do NOT use `@Autowired` directly on ActionBeans. Stripes has its own Spring integration interceptor for that.
- Do NOT suggest Spring MVC annotations (`@RequestMapping`, `@GetMapping`, `@PostMapping`, `@Controller`) on Stripes ActionBeans.

### Common Stripes Patterns

**Returning a JSP view:**
```java
return new ForwardResolution("/WEB-INF/jsp/myPage.jsp");
```

**Redirecting after form submission (POST-Redirect-GET):**
```java
return new RedirectResolution(MyActionBean.class);
```

**Returning JSON for AJAX calls:**
```java
String json = objectMapper.writeValueAsString(data);
return new StreamingResolution("application/json", json);
```

**Accessing session:**
```java
getContext().getRequest().getSession().getAttribute("key");
```

**Custom validation method:**
```java
@ValidationMethod(on = "save")
public void validateBeforeSave(ValidationErrors errors) {
    if (name == null || name.trim().isEmpty()) {
        errors.add("name", new SimpleError("Name is required."));
    }
}
```

---

## Knockout.js Frontend Guidelines

### Core Concepts

I come from a React background (or at least modern component-based frameworks), so map Knockout concepts accordingly:

| Knockout.js Concept | Modern Framework Equivalent | Notes |
|---|---|---|
| `ko.observable()` | `useState()` in React / reactive ref | Single value that auto-updates the UI on change |
| `ko.observableArray()` | `useState([])` with array state | Array with mutation tracking |
| `ko.computed()` | `useMemo()` / derived state | Automatically recalculates when dependencies change |
| `ko.pureComputed()` | `useMemo()` (lazy) | Same as computed but optimized, only evaluates when read |
| `data-bind="text: name"` | `{name}` in JSX | Binds element text to an observable |
| `data-bind="foreach: items"` | `.map()` in JSX | Iterates over an observableArray |
| `data-bind="click: handler"` | `onClick={handler}` | Event binding |
| `data-bind="visible: isShown"` | Conditional rendering `{isShown && ...}` | Toggle visibility |
| `ko.applyBindings(viewModel)` | `ReactDOM.render()` / mounting | Activates Knockout on a DOM subtree |
| ViewModel (plain JS object/class) | React Component state + logic | Holds all observables and behavior |

### When generating or suggesting Knockout.js code, follow these rules:

- Always use `ko.observable()` for values that the UI needs to react to. Plain JS variables will NOT trigger UI updates.
- Use `ko.observableArray()` for lists. Remember: calling `myArray.push()` on an observableArray triggers UI updates, but mutating an item inside it does NOT unless that item's properties are also observables.
- To read an observable's value, call it as a function: `myObservable()`. To write, pass a value: `myObservable(newValue)`. This is the most common gotcha for newcomers.
- Use `ko.computed()` for derived values, not manual subscriptions.
- Use `self = this` pattern or arrow functions in ViewModels to avoid `this` binding issues.
- For AJAX calls, prefer `$.ajax()` or `fetch()` depending on what the project already uses. After receiving data, update observables to trigger UI refresh.
- When binding to forms, use `data-bind="value: fieldName"` for two-way binding on inputs.
- Use `data-bind="with: childViewModel"` to scope bindings to a nested ViewModel.
- Use `ko.toJSON(viewModel)` to serialize the ViewModel for sending to the server.
- Do NOT suggest Angular, React, or Vue patterns. This is a Knockout.js codebase.

### Common Knockout Patterns

**ViewModel definition:**
```javascript
function MyViewModel() {
    var self = this;
    self.firstName = ko.observable("");
    self.lastName = ko.observable("");

    self.fullName = ko.computed(function() {
        return self.firstName() + " " + self.lastName();
    });

    self.save = function() {
        var data = ko.toJSON(self);
        $.ajax({
            url: "/myaction/save",
            type: "POST",
            data: data,
            contentType: "application/json",
            success: function(response) {
                // handle success
            }
        });
    };
}

ko.applyBindings(new MyViewModel());
```

**HTML binding:**
```html
<input type="text" data-bind="value: firstName" />
<span data-bind="text: fullName"></span>
<button data-bind="click: save">Save</button>
<ul data-bind="foreach: items">
    <li data-bind="text: $data.name"></li>
</ul>
```

---

## Full Stack Interaction Pattern

The typical flow between Knockout.js and Stripes is:

1. JSP page loads and includes Knockout ViewModel JS file.
2. `ko.applyBindings(new SomeViewModel())` initializes the UI.
3. User interacts with the page, observables update reactively.
4. On form submit or button click, the ViewModel makes an AJAX call (usually `$.ajax`) to a Stripes ActionBean URL.
5. The ActionBean method processes the request, calls service/DAO layers, and returns a `StreamingResolution` with JSON.
6. The Knockout ViewModel's success callback parses the JSON and updates observables, which automatically refreshes the UI.

**Copilot should always suggest code that fits this flow.** Do not suggest REST controllers, `@ResponseBody`, or Spring MVC patterns for the backend. Use Stripes ActionBeans with `StreamingResolution` for AJAX endpoints.

---

## Code Style Preferences

- Use meaningful variable and method names.
- Prefer `var self = this;` in Knockout ViewModels over arrow functions if the existing codebase uses that pattern.
- Follow existing project conventions for package naming, file organization, and indentation.
- Add comments explaining "why" not "what" when the logic is non-obvious.
- For Stripes, keep ActionBeans focused. If an ActionBean is handling too many events, suggest splitting it.

---

## What NOT To Suggest

- Do NOT suggest Spring MVC annotations on Stripes ActionBeans.
- Do NOT suggest React/Angular/Vue patterns in Knockout.js files.
- Do NOT suggest `@RestController` or `@ResponseBody`. Use `StreamingResolution` in Stripes.
- Do NOT suggest constructor injection in ActionBeans (Stripes uses setter injection via `@SpringBean`).
- Do NOT assume Spring Boot auto-configuration. Stripes has its own configuration through `StripesFilter` in `web.xml`.

# Copilot Chat Prompt Snippets

Ready-to-use prompts you can paste into GitHub Copilot Chat (Ctrl+I or the chat panel) for common tasks in your Stripes + Knockout.js codebase.

---

## Understanding Existing Code

### Explain an ActionBean
```
Explain this Stripes ActionBean. Map each part to its Spring MVC equivalent:
- What URL does it handle?
- What events/handlers does it expose?
- How does parameter binding work on the fields?
- What validations are in place?
- What does each Resolution do?
```

### Explain a Knockout ViewModel
```
Explain this Knockout.js ViewModel. For each observable and computed, tell me:
- What data does it track?
- What UI elements likely bind to it?
- How does data flow between the UI and this ViewModel?
- What AJAX calls does it make and how do the responses update the UI?
```

### Trace a full request flow
```
Trace the full request lifecycle for this feature:
1. What happens in the Knockout ViewModel when the user triggers this action?
2. What data gets sent to the server and in what format?
3. Which Stripes ActionBean and handler method receives it?
4. How does Stripes bind the request parameters to ActionBean fields?
5. What does the handler do and what Resolution does it return?
6. How does the Knockout success callback process the response?
```

---

## Writing New Code

### Create a new ActionBean
```
Create a Stripes ActionBean for [FEATURE DESCRIPTION].
Requirements:
- Use @UrlBinding for the URL path
- Include a @DefaultHandler for the initial page load (ForwardResolution to JSP)
- Include @HandlesEvent methods for: [LIST YOUR EVENTS like save, delete, search]
- For AJAX handlers, return StreamingResolution with JSON
- Inject services using @SpringBean (not @Autowired)
- Add @Validate annotations on required fields
- Include proper getters/setters for all fields
Do NOT use any Spring MVC annotations.
```

### Create a new Knockout ViewModel
```
Create a Knockout.js ViewModel for [FEATURE DESCRIPTION].
Requirements:
- Use ko.observable() for all reactive fields
- Use ko.observableArray() for any lists
- Use ko.computed() for derived values
- Include AJAX methods that call Stripes ActionBean endpoints
- Send data using form parameter format with _eventName (not JSON body)
- Use the var self = this pattern
- Include loading states (isLoading observable) and error handling
- Map Stripes nested property binding in parameter names (e.g., "booking.name")
```

### Create a CRUD feature end-to-end
```
Generate both the Stripes ActionBean and the Knockout ViewModel for a CRUD feature managing [ENTITY NAME].
Backend (Stripes):
- ActionBean with list, view, save, delete handlers
- Use StreamingResolution for all AJAX responses
- Include validation with @Validate and @ValidationMethod
Frontend (Knockout):
- ViewModel with observableArray for the list
- Form observables for create/edit
- AJAX calls matching the ActionBean events
- Loading and error state handling
Show me how the parameter names in the AJAX calls map to the ActionBean fields.
```

---

## Debugging and Fixing

### Fix a binding issue
```
This Knockout binding is not updating the UI when data changes. Check for:
1. Is the value wrapped in ko.observable()?
2. Am I reading the observable correctly (with parentheses in JS, without in data-bind)?
3. If it's an array item property, is that property itself an observable?
4. Is ko.applyBindings called correctly and only once for this DOM section?
5. Are there any typos in the data-bind attribute names?
```

### Fix an ActionBean parameter binding issue
```
The Stripes ActionBean is not receiving the values sent from the frontend. Check:
1. Do the AJAX parameter names match the ActionBean field names exactly?
2. For nested objects, is the dot notation correct (e.g., "booking.name")?
3. Does the ActionBean field have both a getter AND a setter?
4. Is the _eventName parameter being sent to route to the correct handler?
5. Is the Content-Type appropriate (should be form-encoded, not JSON)?
6. Are there @Validate annotations that might be rejecting the input silently?
```

### Debug AJAX communication
```
The AJAX call from Knockout to the Stripes backend is failing or returning unexpected results. Help me debug:
1. Check the browser Network tab - what status code and response body?
2. In the Stripes ActionBean, is the handler returning StreamingResolution with correct content type?
3. Is the JSON being serialized correctly (using ObjectMapper or Gson)?
4. Is the Knockout success callback parsing the response properly?
5. Are there Stripes interceptors that might be redirecting or blocking the request?
```

---

## Refactoring and Migration

### Modernize a JSP form to AJAX
```
This feature currently uses a traditional JSP form submission with Stripes tags.
Convert it to use AJAX:
1. Keep the existing ActionBean but change the handler to return StreamingResolution with JSON instead of ForwardResolution
2. Create a Knockout ViewModel that replaces the form behavior
3. Bind the form inputs using Knockout data-bind instead of Stripes tags
4. Add a click/submit handler that posts via $.ajax
5. Handle the JSON response by updating observables
Show both the before and after for the ActionBean and the JSP/JS.
```

### Extract a reusable Knockout component
```
This section of the ViewModel and HTML template is reused in multiple places.
Refactor it into a Knockout component:
1. Register it with ko.components.register
2. Define the params interface clearly
3. Move the template into the component definition
4. Show how to use it with <component-name params="...">
```

### Consolidate duplicate ActionBean handlers
```
These ActionBeans have similar handler methods. Suggest a refactoring approach:
1. Can a shared base ActionBean hold common logic?
2. Should common operations be moved to a service layer injected via @SpringBean?
3. Are there utility methods for common Resolution patterns (like returning JSON success/error)?
```

---

## Testing Prompts

### Write a Stripes test
```
Write a test for this Stripes ActionBean using MockServletContext and MockRoundtrip.
The test should:
1. Set up the Stripes MockServletContext
2. Create a MockRoundtrip targeting the ActionBean
3. Set request parameters matching what the frontend sends
4. Execute the roundtrip
5. Assert the handler was called, the Resolution type is correct, and the output contains expected data
```

### Test a Knockout ViewModel
```
Write a unit test for this Knockout ViewModel using Jasmine/Mocha.
The test should:
1. Create the ViewModel instance
2. Set observables to test values
3. Verify computed observables return expected results
4. Mock $.ajax to simulate AJAX calls
5. Verify observables update correctly after simulated server responses
```

---

## Quick Reference Prompts

### "How would I do X in Stripes?"
```
In Spring MVC, I would do [DESCRIBE THE SPRING WAY].
How do I achieve the same thing in Stripes? Show the Stripes-equivalent code with explanation.
```

### "How would I do X in Knockout?"
```
In React, I would do [DESCRIBE THE REACT WAY].
How do I achieve the same thing in Knockout.js? Show the Knockout-equivalent code with explanation.
```

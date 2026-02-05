# Knockout.js - Deep Dive Context for Copilot

## The Mental Model Shift

If you think in React terms, here is the fundamental difference:

- **React:** State changes trigger a re-render of the virtual DOM, which diffs and patches the real DOM.
- **Knockout:** Observables maintain a list of subscribers (DOM bindings). When an observable changes, it directly notifies each subscriber to update. There is no virtual DOM, no diffing, no re-render cycle.

This means Knockout is more like a fine-grained reactivity system (closer to SolidJS or MobX) than React's coarse-grained re-rendering.

---

## Observable Mechanics

### Reading vs Writing (the most common mistake)

```javascript
// WRONG - this replaces the observable with a plain value
viewModel.name = "Alice";

// RIGHT - this updates the observable and triggers UI refresh
viewModel.name("Alice");

// WRONG - this gives you the observable function, not its value
var n = viewModel.name;

// RIGHT - call it to unwrap the value
var n = viewModel.name();
```

### Observable vs ObservableArray

```javascript
var name = ko.observable("Alice");      // single value
var items = ko.observableArray([]);      // array with tracked mutations

// These trigger UI updates:
items.push({name: "Item 1"});
items.remove(someItem);
items.removeAll();
items.splice(0, 1);

// This does NOT trigger UI updates:
items()[0].name = "Modified";  // mutating internal property of an element

// To make inner properties reactive, they must be observables too:
function Item(data) {
    this.name = ko.observable(data.name);
    this.price = ko.observable(data.price);
}
items.push(new Item({name: "Widget", price: 9.99}));
```

### Computed Observables

```javascript
function OrderViewModel() {
    var self = this;
    self.quantity = ko.observable(1);
    self.unitPrice = ko.observable(25.00);

    // Automatically recalculates when quantity or unitPrice change
    self.totalPrice = ko.computed(function() {
        return self.quantity() * self.unitPrice();
    });

    // pureComputed is the same but sleeps when no one is listening
    self.formattedTotal = ko.pureComputed(function() {
        return "$" + self.totalPrice().toFixed(2);
    });

    // Writable computed (two-way derived value)
    self.priceWithTax = ko.pureComputed({
        read: function() {
            return self.totalPrice() * 1.1;
        },
        write: function(value) {
            self.totalPrice(value / 1.1);
        }
    });
}
```

---

## Data Binding Reference

### Text and Display

```html
<!-- One-way text display -->
<span data-bind="text: name"></span>
<span data-bind="html: richContent"></span>

<!-- CSS and style -->
<div data-bind="css: { 'active': isActive, 'error': hasError }"></div>
<div data-bind="style: { color: messageColor() }"></div>
<div data-bind="attr: { title: tooltipText, id: elementId }"></div>

<!-- Visibility -->
<div data-bind="visible: isLoggedIn"></div>
<div data-bind="hidden: isLoading"></div>

<!-- Conditional rendering (removes from DOM unlike visible) -->
<!-- ko if: isAdmin -->
<div>Admin Panel</div>
<!-- /ko -->

<!-- ko ifnot: hasData -->
<div>No data available</div>
<!-- /ko -->
```

### Form Bindings

```html
<!-- Two-way binding on input -->
<input type="text" data-bind="value: firstName" />

<!-- Real-time update (on keypress instead of blur) -->
<input type="text" data-bind="textInput: searchQuery" />

<!-- Checkbox -->
<input type="checkbox" data-bind="checked: isActive" />

<!-- Radio buttons -->
<input type="radio" value="small" data-bind="checked: selectedSize" />
<input type="radio" value="large" data-bind="checked: selectedSize" />

<!-- Dropdown -->
<select data-bind="options: availableCountries,
                   optionsText: 'name',
                   optionsValue: 'code',
                   value: selectedCountry,
                   optionsCaption: 'Choose...'"></select>

<!-- Enable/Disable -->
<button data-bind="enable: canSave">Save</button>
<input data-bind="disable: isReadOnly" />
```

### Event Bindings

```html
<!-- Click -->
<button data-bind="click: saveForm">Save</button>

<!-- Other events -->
<input data-bind="event: { blur: validateField, keypress: onKeyPress }" />

<!-- Submit (prevent default automatically) -->
<form data-bind="submit: handleSubmit">
```

### Iteration

```html
<!-- foreach -->
<ul data-bind="foreach: items">
    <li>
        <span data-bind="text: name"></span>
        <!-- $parent accesses the parent ViewModel -->
        <button data-bind="click: $parent.removeItem">Remove</button>
        <!-- $index() gives the current index -->
        <span data-bind="text: $index() + 1"></span>
        <!-- $data refers to the current item -->
        <span data-bind="text: $data"></span>
    </li>
</ul>

<!-- Containerless foreach (no wrapper element) -->
<!-- ko foreach: items -->
<div data-bind="text: name"></div>
<!-- /ko -->
```

### Scope Control

```html
<!-- 'with' changes the binding context -->
<div data-bind="with: selectedOrder">
    <!-- inside here, binding context is the selectedOrder object -->
    <span data-bind="text: orderId"></span>
    <span data-bind="text: total"></span>
</div>

<!-- 'let' introduces new variables without changing context -->
<!-- ko let: { itemCount: items().length } -->
<span data-bind="text: itemCount"></span>
<!-- /ko -->
```

---

## Component Pattern

Knockout supports reusable components (registered globally):

```javascript
ko.components.register('user-card', {
    viewModel: function(params) {
        this.name = params.name;
        this.email = params.email;
        this.isSelected = ko.observable(false);
        this.toggleSelect = function() {
            this.isSelected(!this.isSelected());
        }.bind(this);
    },
    template:
        '<div class="user-card" data-bind="css: { selected: isSelected }">' +
        '    <h3 data-bind="text: name"></h3>' +
        '    <p data-bind="text: email"></p>' +
        '    <button data-bind="click: toggleSelect">Select</button>' +
        '</div>'
});
```

Usage:
```html
<user-card params="name: userName, email: userEmail"></user-card>
```

---

## AJAX Patterns with Stripes Backend

### Loading data on page init

```javascript
function FlightSearchViewModel() {
    var self = this;
    self.flights = ko.observableArray([]);
    self.isLoading = ko.observable(false);
    self.errorMessage = ko.observable("");

    self.searchFlights = function() {
        self.isLoading(true);
        self.errorMessage("");

        $.ajax({
            url: "/flights/search",
            type: "POST",
            data: {
                _eventName: "search",
                origin: self.origin(),
                destination: self.destination(),
                departureDate: self.departureDate()
            },
            success: function(response) {
                var flightList = response.map(function(f) {
                    return new FlightModel(f);
                });
                self.flights(flightList);
            },
            error: function(xhr) {
                self.errorMessage("Failed to load flights. Please try again.");
            },
            complete: function() {
                self.isLoading(false);
            }
        });
    };
}
```

### Submitting a form

```javascript
self.saveBooking = function() {
    var payload = {
        _eventName: "save",
        "booking.passengerName": self.passengerName(),
        "booking.flightId": self.selectedFlight().id,
        "booking.seatClass": self.selectedClass()
    };

    $.ajax({
        url: "/bookings",
        type: "POST",
        data: payload,
        success: function(response) {
            if (response.success) {
                window.location.href = "/bookings/" + response.bookingId;
            } else {
                self.errors(response.errors);
            }
        }
    });
};
```

**Note the parameter naming convention:** `booking.passengerName` maps directly to the ActionBean's nested property binding. The Stripes ActionBean would have a `booking` field of type `Booking`, and Stripes will call `getBooking().setPassengerName(value)`.

---

## Debugging Tips

### Inspect observable values in browser console

```javascript
// If viewModel is accessible globally
var vm = ko.dataFor(document.getElementById("someElement"));
console.log(vm.name());                  // read a value
console.log(ko.toJSON(vm, null, 2));     // serialize entire ViewModel

// Find what an element is bound to
var context = ko.contextFor(someElement);
console.log(context.$data);    // current binding context
console.log(context.$parent);  // parent context
console.log(context.$root);    // root ViewModel
```

### Common issues to watch for

1. **Forgetting parentheses:** `text: name` in HTML binding does NOT need parentheses, but in JavaScript code you MUST use `name()` to read.
2. **Not using observables for dynamic data:** If a value should change and the UI should update, it must be an observable. Plain properties render once and never update.
3. **ObservableArray item mutations:** Changing a property on an item inside an observableArray does not trigger the array's subscribers. The item's properties must themselves be observables.
4. **Binding to non-existent properties:** Knockout silently fails. Check the browser console for binding errors.
5. **Multiple applyBindings calls:** Calling `ko.applyBindings` twice on the same DOM element causes duplicate bindings and strange behavior. Use the second argument to scope: `ko.applyBindings(vm, document.getElementById("mySection"))`.

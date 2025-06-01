# Accessible Forms

Accessible forms. Label elements. For attribute. Fieldset. Legend. Error messages. Validation. aria-invalid. aria-describedby. aria-required. Required attribute. Placeholder text. Autocomplete. Form accessibility. Hugo shortcodes. Contact form. Custom select. Custom checkbox. Screen reader forms. WCAG forms.

## Principle

Build forms so that every control has a programmatically associated label, related controls are grouped with meaningful descriptions, validation errors are announced to assistive technology and linked to their corresponding fields, and the entire form is operable by keyboard alone. Native HTML form elements provide built-in accessibility that custom components must replicate completely. In Hugo static sites, form patterns are defined in layout partials, shortcodes, and shared JavaScript so that every form across the site follows the same accessible structure without per-page configuration.

## Label Elements

### The for Attribute

The `<label>` element's `for` attribute creates a programmatic association between the label text and the form control. Screen readers announce the label when the control receives focus. Clicking the label focuses the control, enlarging the clickable area.

```html
<!-- DO: Explicit label with for attribute matching input id -->
<label for="email">Email address</label>
<input type="email" id="email" name="email">

<!-- DO: Label and input do not need to be adjacent in the DOM -->
<div class="form-group">
  <label for="username">Username</label>
  <p id="username-help" class="help-text">Letters and numbers only, 3-20 characters.</p>
  <input type="text" id="username" name="username" aria-describedby="username-help">
</div>
```

### Wrapping Label Pattern

Wrapping the input inside the `<label>` element creates an implicit association without needing `for` and `id` attributes. This pattern is useful for simple controls where the label and input are always together.

```html
<!-- DO: Implicit association by wrapping -->
<label>
  Email address
  <input type="email" name="email">
</label>

<!-- DO: Checkbox with wrapping label -->
<label>
  <input type="checkbox" name="agree">
  I agree to the terms of service
</label>

<!-- DO: Radio with wrapping label -->
<label>
  <input type="radio" name="plan" value="free">
  Free plan
</label>
```

### When Each Pattern Is Best

| Pattern | Use When | Advantage |
|---|---|---|
| `for` + `id` | Label and input are separated in the DOM | Flexible layout, works with help text between |
| Wrapping `<label>` | Label and input are always together | Simpler markup, no ID management |

### Common Label Mistakes

```html
<!-- DON'T: Missing label entirely -->
<input type="text" name="search" placeholder="Search...">
<!-- Screen reader announces: "edit text" -- no context -->

<!-- DON'T: Label without for attribute (no programmatic association) -->
<label>Email</label>
<input type="email" id="email" name="email">
<!-- Label and input look associated visually but are not connected programmatically -->

<!-- DON'T: aria-label when a visible label exists -->
<label for="email">Email address</label>
<input type="email" id="email" name="email" aria-label="Enter your email">
<!-- aria-label overrides the visible label -- screen reader ignores "Email address" -->

<!-- DON'T: Duplicate IDs -->
<label for="name">Name</label>
<input type="text" id="name" name="first_name">

<label for="name">Name</label>
<input type="text" id="name" name="last_name">
<!-- Both labels point to the first input; second input has no label -->

<!-- DO: Unique IDs for each control -->
<label for="first-name">First name</label>
<input type="text" id="first-name" name="first_name">

<label for="last-name">Last name</label>
<input type="text" id="last-name" name="last_name">
```

### Visually Hidden Labels

When design requirements omit a visible label (e.g., a search input with only a placeholder and icon), the label must still exist in the DOM for screen readers.

```html
<!-- DO: Visually hidden label for screen readers -->
<label for="search-input" class="sr-only">Search articles</label>
<input type="search" id="search-input" name="q" placeholder="Search...">

<!-- DO: aria-label as alternative (only when no visible label exists) -->
<input type="search" id="search-input" name="q" placeholder="Search..."
       aria-label="Search articles">

<!-- Prefer visible labels whenever possible -- they benefit all users -->
```

## Fieldset and Legend for Grouping

### When to Use Fieldset

`<fieldset>` groups related form controls and `<legend>` provides a label for the entire group. Screen readers announce the legend text before each control within the group, giving users context for what the group represents.

**Use fieldset and legend when:**
- Radio buttons share a common question
- Checkboxes share a common topic
- Multiple fields represent a single concept (address, date range, name)
- A form has distinct sections that need group labels

```html
<!-- Radio group: legend provides the question context -->
<fieldset>
  <legend>Preferred contact method</legend>
  <label>
    <input type="radio" name="contact" value="email">
    Email
  </label>
  <label>
    <input type="radio" name="contact" value="phone">
    Phone
  </label>
  <label>
    <input type="radio" name="contact" value="mail">
    Mail
  </label>
</fieldset>
<!-- Screen reader announces: "Preferred contact method, group"
     then for each radio: "Preferred contact method, Email, radio button" -->
```

```html
<!-- Checkbox group: legend provides topic context -->
<fieldset>
  <legend>Email notification preferences</legend>
  <label>
    <input type="checkbox" name="notifications" value="weekly">
    Weekly digest
  </label>
  <label>
    <input type="checkbox" name="notifications" value="comments">
    Comment replies
  </label>
  <label>
    <input type="checkbox" name="notifications" value="mentions">
    Mentions
  </label>
</fieldset>
```

```html
<!-- Address group: legend labels the entire set of fields -->
<fieldset>
  <legend>Shipping address</legend>

  <label for="street">Street address</label>
  <input type="text" id="street" name="street" autocomplete="street-address">

  <label for="city">City</label>
  <input type="text" id="city" name="city" autocomplete="address-level2">

  <label for="state">State</label>
  <select id="state" name="state" autocomplete="address-level1">
    <option value="">Select a state</option>
    <option value="CA">California</option>
    <option value="NY">New York</option>
  </select>

  <label for="zip">ZIP code</label>
  <input type="text" id="zip" name="zip" autocomplete="postal-code"
         inputmode="numeric" pattern="[0-9]{5}">
</fieldset>
```

### Nested Fieldsets

Fieldsets can be nested for complex forms, but keep nesting shallow to avoid confusion.

```html
<fieldset>
  <legend>Account Settings</legend>

  <label for="display-name">Display name</label>
  <input type="text" id="display-name" name="display_name">

  <fieldset>
    <legend>Privacy options</legend>
    <label>
      <input type="checkbox" name="privacy" value="profile">
      Make profile public
    </label>
    <label>
      <input type="checkbox" name="privacy" value="email">
      Show email address
    </label>
  </fieldset>
</fieldset>
```

### Styling Fieldset and Legend

```css
/* Reset default browser fieldset styling */
fieldset {
  border: 0;
  padding: 0;
  margin: 0;
  min-width: 0;  /* Fix overflow in Firefox */
}

legend {
  padding: 0;
  font-weight: 700;
  font-size: 1.125rem;
  margin-bottom: 0.5rem;
}

/* Visually hidden legend (still announced by screen readers) */
fieldset .sr-only-legend {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border-width: 0;
}
```

## Error Messages and Validation

### Associating Errors with Fields

Error messages must be programmatically linked to their form controls using `aria-describedby`. Screen readers announce the error message after the field label and role, giving the user immediate context about what is wrong.

```html
<!-- Field with error -->
<div class="form-group">
  <label for="email">Email address</label>
  <input type="email" id="email" name="email"
         aria-invalid="true"
         aria-describedby="email-error"
         value="notanemail">
  <p id="email-error" class="field-error" role="alert">
    Please enter a valid email address.
  </p>
</div>

<!-- Screen reader announces:
     "Email address, edit text, invalid entry,
      Please enter a valid email address." -->
```

### The aria-invalid Attribute

`aria-invalid` tells assistive technology that a field's current value is not valid. Screen readers announce "invalid entry" or "invalid data" when the field receives focus.

```html
<!-- Field passes validation: no aria-invalid -->
<input type="email" id="email" name="email" value="user@example.com">

<!-- Field fails validation: aria-invalid="true" -->
<input type="email" id="email" name="email"
       aria-invalid="true"
       aria-describedby="email-error"
       value="notvalid">

<!-- Field has grammar or spelling error -->
<textarea id="bio" aria-invalid="grammar" aria-describedby="bio-error">
  Their going to the store.
</textarea>
<p id="bio-error">Possible grammar error: "Their" should be "They're".</p>
```

### Error Summary Pattern

For forms with multiple errors, display an error summary at the top of the form and link each error to its field. Focus the summary after validation so users can review all errors and click through to each field.

```html
<!-- Error summary: appears at the top of the form after validation -->
<div id="error-summary" role="alert" tabindex="-1">
  <h2>There are 3 errors in this form</h2>
  <ul>
    <li><a href="#name">Name is required</a></li>
    <li><a href="#email">Email address is not valid</a></li>
    <li><a href="#message">Message must be at least 10 characters</a></li>
  </ul>
</div>

<form>
  <div class="form-group">
    <label for="name">Name</label>
    <input type="text" id="name" name="name"
           aria-invalid="true" aria-describedby="name-error" required>
    <p id="name-error" class="field-error">Name is required.</p>
  </div>

  <div class="form-group">
    <label for="email">Email address</label>
    <input type="email" id="email" name="email"
           aria-invalid="true" aria-describedby="email-error">
    <p id="email-error" class="field-error">Email address is not valid.</p>
  </div>

  <div class="form-group">
    <label for="message">Message</label>
    <textarea id="message" name="message"
              aria-invalid="true" aria-describedby="message-error"></textarea>
    <p id="message-error" class="field-error">Message must be at least 10 characters.</p>
  </div>

  <button type="submit">Send Message</button>
</form>
```

```js
// Focus the error summary after validation
function showErrorSummary(errors) {
  const summary = document.getElementById('error-summary');
  const list = summary.querySelector('ul');

  // Clear previous errors
  list.innerHTML = '';

  // Populate error list
  errors.forEach(error => {
    const li = document.createElement('li');
    const a = document.createElement('a');
    a.href = `#${error.fieldId}`;
    a.textContent = error.message;
    li.appendChild(a);
    list.appendChild(li);
  });

  // Update heading
  summary.querySelector('h2').textContent =
    `There ${errors.length === 1 ? 'is 1 error' : `are ${errors.length} errors`} in this form`;

  // Show and focus the summary
  summary.hidden = false;
  summary.focus();
}
```

### Inline Validation Timing

```js
// DO: Validate on blur (when user leaves the field)
// This avoids interrupting the user while they are still typing
field.addEventListener('blur', () => {
  validateField(field);
});

// DO: Clear errors on input (immediate feedback that the user is fixing the issue)
field.addEventListener('input', () => {
  if (field.getAttribute('aria-invalid') === 'true') {
    clearFieldError(field);
  }
});

// DON'T: Validate on every keystroke (disruptive, especially for screen readers)
field.addEventListener('input', () => {
  validateField(field);  // Fires on every character -- too noisy
});

function validateField(field) {
  const error = getFieldError(field);
  if (error) {
    field.setAttribute('aria-invalid', 'true');
    showFieldError(field, error);
  } else {
    field.removeAttribute('aria-invalid');
    clearFieldError(field);
  }
}

function showFieldError(field, message) {
  const errorId = `${field.id}-error`;
  let errorEl = document.getElementById(errorId);

  if (!errorEl) {
    errorEl = document.createElement('p');
    errorEl.id = errorId;
    errorEl.className = 'field-error';
    errorEl.setAttribute('role', 'alert');
    field.parentNode.appendChild(errorEl);
  }

  errorEl.textContent = message;
  field.setAttribute('aria-describedby', errorId);
}

function clearFieldError(field) {
  const errorId = `${field.id}-error`;
  const errorEl = document.getElementById(errorId);

  if (errorEl) {
    errorEl.textContent = '';
  }

  field.removeAttribute('aria-invalid');
}
```

### Error Message Styling

```css
/* Error message styling */
.field-error {
  color: var(--color-error, #c62828);
  font-size: 0.875rem;
  margin-top: 0.25rem;
  padding-left: 0.5rem;
  border-left: 3px solid var(--color-error, #c62828);
}

/* Error state on the input */
input[aria-invalid="true"],
textarea[aria-invalid="true"],
select[aria-invalid="true"] {
  border-color: var(--color-error, #c62828);
  box-shadow: 0 0 0 1px var(--color-error, #c62828);
}

/* Error icon (decorative -- text provides the information) */
.field-error::before {
  content: "";
  display: inline-block;
  width: 1em;
  height: 1em;
  margin-right: 0.25rem;
  background-image: url("data:image/svg+xml,...");
  vertical-align: middle;
}

/* Error summary styling */
#error-summary {
  background-color: #fce4ec;
  border: 2px solid var(--color-error, #c62828);
  border-radius: 4px;
  padding: 1rem;
  margin-bottom: 1.5rem;
}

#error-summary h2 {
  color: var(--color-error, #c62828);
  font-size: 1.125rem;
  margin-bottom: 0.5rem;
}

#error-summary a {
  color: var(--color-error, #c62828);
  text-decoration: underline;
}
```

## Required Fields

### HTML required Attribute vs aria-required

```html
<!-- Native required: browser provides built-in validation UI -->
<label for="name">Name <span aria-hidden="true">*</span></label>
<input type="text" id="name" name="name" required>
<!-- Screen reader announces: "Name, required, edit text" -->

<!-- aria-required: communicates required state without browser validation -->
<label for="name">Name <span aria-hidden="true">*</span></label>
<input type="text" id="name" name="name" aria-required="true">
<!-- Screen reader announces: "Name, required, edit text" -->
<!-- No browser validation tooltip -- you handle validation in JavaScript -->
```

### Indicating Required Fields

```html
<!-- Pattern 1: Asterisk with explanation (most common) -->
<p class="form-help" id="required-help">
  Fields marked with <span aria-hidden="true">*</span>
  <span class="sr-only">an asterisk</span> are required.
</p>

<form aria-describedby="required-help">
  <label for="name">
    Name <span aria-hidden="true">*</span>
    <span class="sr-only">(required)</span>
  </label>
  <input type="text" id="name" name="name" required>

  <label for="email">
    Email <span aria-hidden="true">*</span>
    <span class="sr-only">(required)</span>
  </label>
  <input type="email" id="email" name="email" required>

  <label for="phone">Phone (optional)</label>
  <input type="tel" id="phone" name="phone">
</form>

<!-- Pattern 2: Mark optional fields instead (when most fields are required) -->
<label for="name">Name</label>
<input type="text" id="name" name="name" required>

<label for="email">Email</label>
<input type="email" id="email" name="email" required>

<label for="phone">Phone <span class="optional">(optional)</span></label>
<input type="tel" id="phone" name="phone">
```

## Placeholder Text Is Not a Label

### Why Placeholders Fail as Labels

Placeholder text disappears when the user begins typing, removing the context for what the field expects. Users with cognitive disabilities, short-term memory issues, or who tab away and return cannot recall what the field was for. Screen readers may or may not announce placeholder text depending on the browser and screen reader combination.

```html
<!-- DON'T: Placeholder as the only label -->
<input type="email" placeholder="Email address">
<!-- When user types, they see: "user@exam..." with no label -->
<!-- Screen reader may announce: "edit text" or "Email address, edit text" (inconsistent) -->

<!-- DON'T: Placeholder repeating the label -->
<label for="email">Email</label>
<input type="email" id="email" placeholder="Email">
<!-- Redundant -- adds no value -->

<!-- DO: Label with helpful placeholder example -->
<label for="email">Email address</label>
<input type="email" id="email" name="email" placeholder="e.g., user@example.com">
<!-- Label persists; placeholder shows format example -->

<!-- DO: Label with help text instead of placeholder -->
<label for="email">Email address</label>
<input type="email" id="email" name="email" aria-describedby="email-help">
<p id="email-help" class="help-text">We will send a confirmation to this address.</p>
<!-- Help text persists alongside the label and is read by screen readers -->
```

### Placeholder Styling Concerns

```css
/* Placeholder text has low default contrast in most browsers */
/* Check that placeholder color meets 4.5:1 against the input background */

/* DON'T: Very light placeholder (fails contrast) */
::placeholder {
  color: #cccccc;    /* 1.6:1 on white -- FAILS */
}

/* DO: Sufficient contrast for placeholder text */
::placeholder {
  color: #767676;    /* 4.5:1 on white -- meets AA */
  opacity: 1;        /* Firefox reduces placeholder opacity by default */
}
```

## Form Autocomplete Attributes

### Why Autocomplete Matters

The `autocomplete` attribute tells browsers what type of data a field expects, enabling autofill. This benefits all users but is especially important for users with motor disabilities (reduces typing), cognitive disabilities (reduces memory load), and screen reader users (confirms field purpose).

WCAG 2.1 Success Criterion 1.3.5 (Identify Input Purpose, Level AA) requires `autocomplete` on fields that collect personal information.

```html
<!-- Personal information -->
<input type="text" name="name" autocomplete="name">
<input type="text" name="given_name" autocomplete="given-name">
<input type="text" name="family_name" autocomplete="family-name">
<input type="email" name="email" autocomplete="email">
<input type="tel" name="phone" autocomplete="tel">

<!-- Address -->
<input type="text" name="street" autocomplete="street-address">
<input type="text" name="city" autocomplete="address-level2">
<input type="text" name="state" autocomplete="address-level1">
<input type="text" name="zip" autocomplete="postal-code">
<input type="text" name="country" autocomplete="country-name">

<!-- Account -->
<input type="text" name="username" autocomplete="username">
<input type="password" name="password" autocomplete="new-password">
<input type="password" name="current_password" autocomplete="current-password">

<!-- Payment -->
<input type="text" name="cc_name" autocomplete="cc-name">
<input type="text" name="cc_number" autocomplete="cc-number">
<input type="text" name="cc_exp" autocomplete="cc-exp">
<input type="text" name="cc_csc" autocomplete="cc-csc">
```

## Hugo Shortcodes for Accessible Forms

### Form Field Shortcode

**layouts/shortcodes/form-field.html:**

```go-html-template
{{/*
  Usage:
  {{</* form-field
    id="email"
    label="Email address"
    type="email"
    required="true"
    autocomplete="email"
    help="We will send a confirmation to this address."
    placeholder="e.g., user@example.com"
  */>}}
*/}}
{{ $id := .Get "id" }}
{{ $label := .Get "label" }}
{{ $type := .Get "type" | default "text" }}
{{ $required := eq (.Get "required") "true" }}
{{ $autocomplete := .Get "autocomplete" }}
{{ $help := .Get "help" }}
{{ $placeholder := .Get "placeholder" }}

<div class="form-group">
  <label for="{{ $id }}">
    {{ $label }}
    {{ if $required }}
      <span aria-hidden="true">*</span>
      <span class="sr-only">(required)</span>
    {{ end }}
  </label>

  <input
    type="{{ $type }}"
    id="{{ $id }}"
    name="{{ $id }}"
    {{ if $required }}required aria-required="true"{{ end }}
    {{ with $autocomplete }}autocomplete="{{ . }}"{{ end }}
    {{ with $placeholder }}placeholder="{{ . }}"{{ end }}
    {{ with $help }}aria-describedby="{{ $id }}-help"{{ end }}>

  {{ with $help }}
    <p id="{{ $id }}-help" class="help-text">{{ . }}</p>
  {{ end }}
</div>
```

### Textarea Shortcode

**layouts/shortcodes/form-textarea.html:**

```go-html-template
{{/*
  Usage:
  {{</* form-textarea
    id="message"
    label="Your message"
    required="true"
    rows="6"
    help="Minimum 10 characters."
  */>}}
*/}}
{{ $id := .Get "id" }}
{{ $label := .Get "label" }}
{{ $required := eq (.Get "required") "true" }}
{{ $rows := .Get "rows" | default "4" }}
{{ $help := .Get "help" }}

<div class="form-group">
  <label for="{{ $id }}">
    {{ $label }}
    {{ if $required }}
      <span aria-hidden="true">*</span>
      <span class="sr-only">(required)</span>
    {{ end }}
  </label>

  <textarea
    id="{{ $id }}"
    name="{{ $id }}"
    rows="{{ $rows }}"
    {{ if $required }}required aria-required="true"{{ end }}
    {{ with $help }}aria-describedby="{{ $id }}-help"{{ end }}></textarea>

  {{ with $help }}
    <p id="{{ $id }}-help" class="help-text">{{ . }}</p>
  {{ end }}
</div>
```

### Select Shortcode

**layouts/shortcodes/form-select.html:**

```go-html-template
{{/*
  Usage:
  {{</* form-select
    id="topic"
    label="Topic"
    required="true"
    options="General inquiry,Technical support,Billing,Partnership"
    placeholder="Select a topic"
  */>}}
*/}}
{{ $id := .Get "id" }}
{{ $label := .Get "label" }}
{{ $required := eq (.Get "required") "true" }}
{{ $options := split (.Get "options") "," }}
{{ $placeholder := .Get "placeholder" }}

<div class="form-group">
  <label for="{{ $id }}">
    {{ $label }}
    {{ if $required }}
      <span aria-hidden="true">*</span>
      <span class="sr-only">(required)</span>
    {{ end }}
  </label>

  <select
    id="{{ $id }}"
    name="{{ $id }}"
    {{ if $required }}required aria-required="true"{{ end }}>
    {{ with $placeholder }}
      <option value="">{{ . }}</option>
    {{ end }}
    {{ range $options }}
      <option value="{{ . | urlize }}">{{ . }}</option>
    {{ end }}
  </select>
</div>
```

### Radio Group Shortcode

**layouts/shortcodes/form-radios.html:**

```go-html-template
{{/*
  Usage:
  {{</* form-radios
    name="contact"
    legend="Preferred contact method"
    options="Email,Phone,Mail"
    required="true"
  */>}}
*/}}
{{ $name := .Get "name" }}
{{ $legend := .Get "legend" }}
{{ $required := eq (.Get "required") "true" }}
{{ $options := split (.Get "options") "," }}

<fieldset>
  <legend>
    {{ $legend }}
    {{ if $required }}
      <span aria-hidden="true">*</span>
      <span class="sr-only">(required)</span>
    {{ end }}
  </legend>
  {{ range $index, $option := $options }}
    <label>
      <input type="radio"
             name="{{ $name }}"
             value="{{ $option | urlize }}"
             {{ if $required }}{{ if eq $index 0 }}required{{ end }}{{ end }}>
      {{ $option }}
    </label>
  {{ end }}
</fieldset>
```

## Contact Form Template with Accessibility

### Complete Hugo Contact Form Partial

**layouts/partials/contact-form.html:**

```go-html-template
{{/*
  Accessible contact form partial.
  Include in any page: {{ partial "contact-form.html" . }}
  Handles required fields, help text, error regions, and autocomplete.
*/}}
<section aria-labelledby="contact-heading">
  <h2 id="contact-heading">Contact Us</h2>

  <p id="required-fields-note">
    Fields marked with <span aria-hidden="true">*</span>
    <span class="sr-only">an asterisk</span> are required.
  </p>

  {{/* Error summary: hidden by default, shown by JavaScript after validation */}}
  <div id="contact-error-summary" role="alert" tabindex="-1" hidden>
    <h3>Please correct the following errors</h3>
    <ul></ul>
  </div>

  {{/* Success message: hidden by default, shown after successful submission */}}
  <div id="contact-success" role="status" aria-live="polite" hidden>
    <p>Thank you for your message. We will respond within 2 business days.</p>
  </div>

  <form id="contact-form" action="/api/contact" method="post" novalidate
        aria-describedby="required-fields-note">

    <div class="form-group">
      <label for="contact-name">
        Full name <span aria-hidden="true">*</span>
        <span class="sr-only">(required)</span>
      </label>
      <input type="text" id="contact-name" name="name"
             required aria-required="true"
             autocomplete="name">
    </div>

    <div class="form-group">
      <label for="contact-email">
        Email address <span aria-hidden="true">*</span>
        <span class="sr-only">(required)</span>
      </label>
      <input type="email" id="contact-email" name="email"
             required aria-required="true"
             autocomplete="email"
             aria-describedby="contact-email-help">
      <p id="contact-email-help" class="help-text">
        We will reply to this address.
      </p>
    </div>

    <div class="form-group">
      <label for="contact-phone">Phone (optional)</label>
      <input type="tel" id="contact-phone" name="phone"
             autocomplete="tel">
    </div>

    <fieldset>
      <legend>
        Reason for contact <span aria-hidden="true">*</span>
        <span class="sr-only">(required)</span>
      </legend>
      <label>
        <input type="radio" name="reason" value="general" required>
        General inquiry
      </label>
      <label>
        <input type="radio" name="reason" value="support">
        Technical support
      </label>
      <label>
        <input type="radio" name="reason" value="billing">
        Billing question
      </label>
    </fieldset>

    <div class="form-group">
      <label for="contact-message">
        Message <span aria-hidden="true">*</span>
        <span class="sr-only">(required)</span>
      </label>
      <textarea id="contact-message" name="message" rows="6"
                required aria-required="true"
                aria-describedby="contact-message-help"></textarea>
      <p id="contact-message-help" class="help-text">
        Minimum 10 characters. Be as specific as possible.
      </p>
    </div>

    <label>
      <input type="checkbox" name="consent" required aria-required="true">
      I consent to this site storing my information to respond to my inquiry.
      <span class="sr-only">(required)</span>
    </label>

    <button type="submit">Send Message</button>
  </form>
</section>
```

### Contact Form Validation JavaScript

```js
document.addEventListener('DOMContentLoaded', () => {
  const form = document.getElementById('contact-form');
  if (!form) return;

  form.addEventListener('submit', (event) => {
    event.preventDefault();
    const errors = validateContactForm(form);

    if (errors.length > 0) {
      showFormErrors(form, errors);
    } else {
      clearAllErrors(form);
      submitForm(form);
    }
  });

  // Validate on blur for individual fields
  form.querySelectorAll('input, textarea, select').forEach(field => {
    field.addEventListener('blur', () => {
      validateSingleField(field);
    });

    // Clear error on input when the user starts fixing it
    field.addEventListener('input', () => {
      if (field.getAttribute('aria-invalid') === 'true') {
        clearFieldError(field);
      }
    });
  });
});

function validateContactForm(form) {
  const errors = [];
  const fields = {
    'contact-name': { message: 'Full name is required', validate: v => v.trim().length > 0 },
    'contact-email': { message: 'Please enter a valid email address', validate: v => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v) },
    'contact-message': { message: 'Message must be at least 10 characters', validate: v => v.trim().length >= 10 },
  };

  Object.entries(fields).forEach(([id, rule]) => {
    const field = form.querySelector(`#${id}`);
    if (field && !rule.validate(field.value)) {
      errors.push({ fieldId: id, message: rule.message });
    }
  });

  // Check radio group
  const reasonSelected = form.querySelector('input[name="reason"]:checked');
  if (!reasonSelected) {
    errors.push({ fieldId: 'contact-reason', message: 'Please select a reason for contact' });
  }

  // Check consent checkbox
  const consent = form.querySelector('input[name="consent"]');
  if (consent && !consent.checked) {
    errors.push({ fieldId: 'contact-consent', message: 'You must provide consent to submit this form' });
  }

  return errors;
}

function showFormErrors(form, errors) {
  // Mark each invalid field
  errors.forEach(error => {
    const field = document.getElementById(error.fieldId);
    if (field) {
      field.setAttribute('aria-invalid', 'true');
      showFieldError(field, error.message);
    }
  });

  // Show error summary
  const summary = document.getElementById('contact-error-summary');
  const list = summary.querySelector('ul');
  list.innerHTML = '';

  errors.forEach(error => {
    const li = document.createElement('li');
    const a = document.createElement('a');
    a.href = `#${error.fieldId}`;
    a.textContent = error.message;
    li.appendChild(a);
    list.appendChild(li);
  });

  summary.querySelector('h3').textContent =
    `Please correct the following ${errors.length === 1 ? 'error' : `${errors.length} errors`}`;

  summary.hidden = false;
  summary.focus();
}

function showFieldError(field, message) {
  const errorId = `${field.id}-error`;
  let errorEl = document.getElementById(errorId);

  if (!errorEl) {
    errorEl = document.createElement('p');
    errorEl.id = errorId;
    errorEl.className = 'field-error';
    field.parentNode.appendChild(errorEl);
  }

  errorEl.textContent = message;

  // Add error to aria-describedby (preserve existing descriptions)
  const existing = field.getAttribute('aria-describedby') || '';
  if (!existing.includes(errorId)) {
    field.setAttribute('aria-describedby', `${existing} ${errorId}`.trim());
  }
}

function clearFieldError(field) {
  field.removeAttribute('aria-invalid');
  const errorId = `${field.id}-error`;
  const errorEl = document.getElementById(errorId);
  if (errorEl) {
    errorEl.textContent = '';
  }

  // Remove error from aria-describedby (keep other descriptions)
  const existing = field.getAttribute('aria-describedby') || '';
  const updated = existing.replace(errorId, '').trim();
  if (updated) {
    field.setAttribute('aria-describedby', updated);
  } else {
    field.removeAttribute('aria-describedby');
  }
}

function clearAllErrors(form) {
  form.querySelectorAll('[aria-invalid]').forEach(field => {
    clearFieldError(field);
  });

  const summary = document.getElementById('contact-error-summary');
  summary.hidden = true;
}

function validateSingleField(field) {
  // Re-validate just this one field
  if (field.hasAttribute('required') && !field.value.trim()) {
    field.setAttribute('aria-invalid', 'true');
  }
}

function submitForm(form) {
  // Submit logic here (fetch API, form action, etc.)
  const success = document.getElementById('contact-success');
  form.hidden = true;
  success.hidden = false;
}
```

## Custom Select and Checkbox Accessibility

### Custom Select (Listbox Pattern)

When a native `<select>` is insufficient (e.g., for multi-select with search), implement the ARIA listbox pattern.

```html
<!-- Custom select with listbox role -->
<div class="custom-select">
  <label id="theme-label">Choose a theme</label>
  <button
    id="theme-button"
    role="combobox"
    aria-expanded="false"
    aria-haspopup="listbox"
    aria-labelledby="theme-label"
    aria-controls="theme-listbox">
    Select a theme
  </button>

  <ul id="theme-listbox" role="listbox" aria-labelledby="theme-label" hidden>
    <li role="option" id="theme-light" aria-selected="false">Light</li>
    <li role="option" id="theme-dark" aria-selected="false">Dark</li>
    <li role="option" id="theme-system" aria-selected="false">System default</li>
  </ul>
</div>
```

```js
// Custom select keyboard behavior
const button = document.getElementById('theme-button');
const listbox = document.getElementById('theme-listbox');
const options = Array.from(listbox.querySelectorAll('[role="option"]'));

button.addEventListener('keydown', (event) => {
  switch (event.key) {
    case 'Enter':
    case ' ':
    case 'ArrowDown':
      event.preventDefault();
      openListbox();
      break;
  }
});

function openListbox() {
  button.setAttribute('aria-expanded', 'true');
  listbox.hidden = false;
  const selected = listbox.querySelector('[aria-selected="true"]') || options[0];
  selected.focus();
}

function closeListbox() {
  button.setAttribute('aria-expanded', 'false');
  listbox.hidden = true;
  button.focus();
}

listbox.addEventListener('keydown', (event) => {
  const currentIndex = options.indexOf(document.activeElement);

  switch (event.key) {
    case 'ArrowDown':
      event.preventDefault();
      if (currentIndex < options.length - 1) {
        options[currentIndex + 1].focus();
      }
      break;
    case 'ArrowUp':
      event.preventDefault();
      if (currentIndex > 0) {
        options[currentIndex - 1].focus();
      }
      break;
    case 'Enter':
    case ' ':
      event.preventDefault();
      selectOption(options[currentIndex]);
      closeListbox();
      break;
    case 'Escape':
      closeListbox();
      break;
    case 'Home':
      event.preventDefault();
      options[0].focus();
      break;
    case 'End':
      event.preventDefault();
      options[options.length - 1].focus();
      break;
  }
});

function selectOption(option) {
  options.forEach(opt => opt.setAttribute('aria-selected', 'false'));
  option.setAttribute('aria-selected', 'true');
  button.textContent = option.textContent;
}
```

### Custom Checkbox

When styling prevents using native `<input type="checkbox">`, use ARIA to replicate the behavior. Native checkboxes are always preferred.

```html
<!-- BEST: Style the native checkbox (no ARIA needed) -->
<label class="custom-checkbox">
  <input type="checkbox" name="agree">
  <span class="checkbox-visual" aria-hidden="true"></span>
  I agree to the terms
</label>
```

```css
/* Hide the native checkbox visually, keep it accessible */
.custom-checkbox input[type="checkbox"] {
  position: absolute;
  opacity: 0;
  width: 1px;
  height: 1px;
}

/* Custom visual indicator */
.custom-checkbox .checkbox-visual {
  display: inline-block;
  width: 1.25rem;
  height: 1.25rem;
  border: 2px solid #595959;
  border-radius: 3px;
  vertical-align: middle;
  margin-right: 0.5rem;
  transition: background-color 0.15s, border-color 0.15s;
}

/* Checked state */
.custom-checkbox input[type="checkbox"]:checked + .checkbox-visual {
  background-color: #0056b3;
  border-color: #0056b3;
  background-image: url("data:image/svg+xml,..."); /* Checkmark */
  background-size: 80%;
  background-position: center;
  background-repeat: no-repeat;
}

/* Focus state */
.custom-checkbox input[type="checkbox"]:focus-visible + .checkbox-visual {
  outline: 3px solid #0056b3;
  outline-offset: 2px;
}
```

**Only use ARIA checkbox if the native element is truly impossible:**

```html
<!-- ARIA checkbox: only when native is impossible -->
<div role="checkbox"
     tabindex="0"
     aria-checked="false"
     aria-labelledby="agree-label"
     id="custom-agree">
  <span class="checkbox-icon" aria-hidden="true"></span>
</div>
<span id="agree-label">I agree to the terms</span>
```

```js
// ARIA checkbox must handle Space and Enter
const checkbox = document.getElementById('custom-agree');
checkbox.addEventListener('keydown', (event) => {
  if (event.key === ' ' || event.key === 'Enter') {
    event.preventDefault();
    toggleCheckbox(checkbox);
  }
});
checkbox.addEventListener('click', () => {
  toggleCheckbox(checkbox);
});

function toggleCheckbox(el) {
  const checked = el.getAttribute('aria-checked') === 'true';
  el.setAttribute('aria-checked', String(!checked));
}
```

## Testing Accessible Forms

### Manual Testing Checklist

```
1. Labels
   [ ] Every input, select, and textarea has a programmatically associated label
   [ ] Labels are visible (not replaced by placeholder text)
   [ ] Clicking any label focuses its associated control
   [ ] Screen reader announces label text when each control receives focus

2. Required Fields
   [ ] Required fields are indicated visually (asterisk or text)
   [ ] Required fields have required or aria-required attribute
   [ ] Screen reader announces "required" for required fields
   [ ] Optional fields are clearly indicated when most fields are required

3. Error Handling
   [ ] Error messages appear adjacent to the invalid field
   [ ] Error messages are linked via aria-describedby
   [ ] Invalid fields have aria-invalid="true"
   [ ] Screen reader announces error message when field receives focus
   [ ] Error summary appears at the top of the form with links to each error
   [ ] Focus moves to error summary after form submission fails
   [ ] Errors clear when the user corrects the input

4. Grouping
   [ ] Related radio buttons are in a fieldset with a legend
   [ ] Related checkboxes are in a fieldset with a legend
   [ ] Address fields are grouped in a fieldset
   [ ] Screen reader announces legend text before each control in the group

5. Autocomplete
   [ ] Personal information fields have appropriate autocomplete values
   [ ] Browser autofill populates fields correctly
   [ ] autocomplete="off" is not used on fields that benefit from autofill

6. Keyboard
   [ ] All form controls are reachable by Tab
   [ ] Radio groups navigate with arrow keys
   [ ] Custom selects support arrow keys, Enter, Escape
   [ ] Form submits with Enter key from any text input
   [ ] Tab order follows visual order
```

## Best Practices

**DO:**
- Associate every form control with a label using `for`/`id` or wrapping `<label>` elements
- Use `<fieldset>` and `<legend>` to group related controls (radio buttons, checkboxes, address fields)
- Link error messages to fields with `aria-describedby` and mark invalid fields with `aria-invalid="true"`
- Show an error summary at the top of the form with links to each invalid field
- Focus the error summary after failed form submission
- Use the `required` attribute or `aria-required="true"` to indicate mandatory fields
- Add `autocomplete` attributes to all fields that collect personal information
- Validate on blur (not on every keystroke) and clear errors on input
- Use visible labels -- never rely on placeholder text as the sole label
- Prefer native HTML form elements over custom ARIA widgets

**DON'T:**
- Use placeholder text as the only label for a form field
- Remove or hide labels for visual design reasons without providing `.sr-only` alternatives
- Rely on color alone to indicate required fields or errors
- Use `aria-label` when a visible `<label>` already exists (it overrides the visible label)
- Use duplicate `id` attributes on form controls
- Validate on every keystroke (creates excessive noise for screen reader users)
- Omit `autocomplete` on fields that collect name, email, address, or payment information
- Skip `<fieldset>`/`<legend>` for radio and checkbox groups
- Use custom ARIA widgets when native HTML elements work
- Forget to restore `aria-describedby` references when clearing errors

## Guidelines

### Essential

- Every form control has a programmatically associated `<label>`
- Required fields have `required` or `aria-required="true"` attribute
- Error messages linked to fields via `aria-describedby`
- Invalid fields marked with `aria-invalid="true"`
- `<fieldset>` and `<legend>` wrap all radio button and checkbox groups
- `autocomplete` attributes on personal information fields (WCAG 1.3.5)
- Form is fully operable by keyboard (Tab through all fields, Enter to submit)

### Recommended

- Error summary with links to invalid fields, focused after failed submission
- Inline validation on blur with immediate error clearing on input
- Help text linked via `aria-describedby` for fields that need instructions
- Hugo shortcodes for form fields that enforce accessible structure
- Required field indicator explained at the top of the form
- Contact form partial with complete accessibility built in
- Custom checkboxes built by visually hiding the native input (not replacing with ARIA)

### Advanced

- Custom select/combobox implemented with full ARIA listbox pattern and keyboard support
- Form validation library that automatically manages `aria-invalid`, `aria-describedby`, and error summary
- Automated form accessibility testing in CI/CD with axe-core
- Server-side validation errors rendered with same accessible structure as client-side
- Multi-step forms with progress indication (`aria-current="step"`)
- File upload controls with accessible status announcements via `aria-live`

## Benefits

Labeled Controls. Programmatic label associations ensure screen reader users know the purpose of every form field before interacting with it.

Clear Error Recovery. Error messages linked to fields with `aria-describedby` and summarized at the top of the form guide users directly to problems without searching.

Grouped Context. `<fieldset>` and `<legend>` give screen reader users the context needed to understand related controls like radio buttons and address fields.

Reduced Input Effort. `autocomplete` attributes enable browser autofill, reducing typing for all users and particularly benefiting those with motor disabilities.

Consistent Form Patterns. Hugo shortcodes and partials enforce accessible form structure across every form on the site without relying on content authors to remember accessibility requirements.

Native Element Reliability. Using native HTML form elements instead of custom ARIA widgets provides built-in keyboard support, validation, and screen reader compatibility without custom JavaScript.

## Related

- [aria-labels-roles.md](./aria-labels-roles.md) - ARIA attributes for labeling and describing form controls
- [keyboard-navigation.md](./keyboard-navigation.md) - Keyboard interaction patterns for form controls and custom widgets
- [screen-reader-optimization.md](./screen-reader-optimization.md) - How screen readers process forms in browse mode and forms mode

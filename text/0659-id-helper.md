- Start Date: 2020-08-25
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/659
- Tracking: 

# {{id}} helper

## Summary

Add a new built-in template helper `{{id}}` for generating unique IDs.

See [pre-RFC issue #612](https://github.com/emberjs/rfcs/issues/612)

## Motivation

When working with HTML it is very common to need to create and reference [DOM IDs that are unique within the HTML document](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/id). Classic Ember components provide the `elementId` attribute which can be used to construct unique ids within classic components, but `elementId` is not available within Glimmer components or route templates.

There are several common use cases where a developer may need to generate a unique ID for use in a template:
1. Associating `label` and `input` elements using the label's `for` attribute and the input's `id` attribute.
2. Using WAI-ARIA attributes to improve accessibility (eg. aria-labelledby, aria-controls)
3. Integrating 3rd party libraries that attach themselves to DOM elements using DOM IDs (eg. maps, datepickers, jquery plugins, etc)

Since providing some faculty for generating unique IDs for DOM elements can reasonably be considered a requirement for most Ember apps wishing to implement an accessible UI (via labelled inputs and/or WAI-ARIA), it is reasonable for Ember to provide this functionality at the framework level. Ember already provides the `guidFor` utility in javascript, so it is reasonable for Ember to provide similar functionality within templates.

## Detailed design

Add built-in `{{id}}` template helper.

### `{{id}}`

The id helper can be invoked with no arguments. When invoked this way, the `id` helper will return a new unique id string for every invocation.

In practice this invocation style would usually be paired with a `let` block to enable re-use of the unique id generated by `{{id}}`. 

```hbs
{{#let (id) as |emailId|}}
  <label for={{emailId}}>Email address</label>
  <input id={{emailId}} type="email" />
{{/let}}

{{#let (id) as |passwordId|}}
  <label for={{passwordId}}>password</label>
  <input id={{passwordId}} type="password" />
{{/let}}
```

In the future, an inline or template version of `let` could enable a single invocation of `{{id}}` for re-use of the id within a template.

### `{{id for=object}}`

The `id` helper can be optionally invoked with a single named argument `for`. This argument accepts any object, string, number, Element, or primitive, which will be treated as a stable reference for an id, allowing the helper to return the same id value for every invocation using the same `for` value.

In class-backed templates where `this` is available, the `for` argument can be used to achieve a more ergonomic invocation style that avoids using `let` blocks.

```hbs
<label for="{{id for=this}}-email">Email address</label>
<input id="{{id for=this}}-email" type="email" />

<label for="{{id for=this}}-password">password</label>
<input id="{{id for=this}}-password" type="password" />
```

### Implementation

Ember already has a `guidFor` utility, so it makes sense to use this existing utility to implement the `{{id}}` helper. Using `guidFor` ensures that all unique ids generated by Ember use the same underlying guid implementation and avoid ID collisions.

A bare-bones implementation of the id helper might looks something like this:

```js
import { helper } from '@ember/component/helper';
import { guidFor } from '@ember/object/internals';
import { isPresent } from '@ember/utils';

export default helper(({ for }) => {
  return isPresent(for) ? guidFor(for) : guidFor();
});
```

## How we teach this

### Ember API docs: Ember.templates.helpers

The Ember API docs can be updated to include the `id` helper on the page for [Ember.templates.helpers](https://api.emberjs.com/ember/release/classes/Ember.Templates.helpers)

> Use the `{{id}}` helper to generate a unique ID string suitable for use as an ID attribute in the DOM.
> 
> ```hbs
> <input id={{id}} type="email" />
> ```
> 
> Each invocation of `{{id}}` will return a new, unique ID string. You can use the `let` helper to create an ID that can be reused within a template.
> ```hbs
> {{#let (id) as |emailId|}}
>   <label for={{emailId}}>Email address</label>
>   <input id={{emailId}} type="email" />
> {{/let}}
> ```
> 
> #### using the `for` argument
> 
> `id` can be invoked with the named argument `for`. Any object passed as `for` will be used as a stable reference for a unique ID. Whenever `id` is invoked with the same `for` argument, the same ID will be returned. Objects, strings, numbers, component/controller classes, and even DOM elements can be used as a `for` argument.
> 
> Assuming the following template has a backing class (such as a controller or component class), `{{id for=this}}` will return the same id string every time it is invoked within that template instance.
> 
> ```hbs
> <label for="{{id for=this}}-email">Email address</label>
> <input id="{{id for=this}}-email" type="email" />
> ```


### Ember Guides: Associating labels and inputs
The Ember guides currently include a section on associating labels and inputs. This section can be updated to use this new `{{id}}` helper.

[Guides: associating labels and inputs](https://guides.emberjs.com/release/components/built-in-components/#toc_ways-to-associate-labels-and-inputs)

> Every input should be associated with a label. Within HTML, there are several different ways to do this. In this section, we will show how to apply those strategies for Ember inputs.
> 
> You can nest the input inside the label:
> ```hbs
> <label>
>   Ask a question about Ember:
>   <Input type="text" @value={{this.val}} />
> </label>
> ```
> You can associate the label using for and id:
> ```hbs
> <label for={{this.myUniqueId}}>
>     Ask a question about Ember:
> </label>
> <Input id={{this.myUniqueId}} type="text" @value={{this.val}} />
> ```
> 
> In HTML, each element's id attribute must be a value that is unique within the HTML document. Ember provides the built-in `{{id}}` helper to assist you with generating unique IDs.
> ```hbs
> <label for="{{id for=this}}-question">
>     Ask a question about Ember:
> </label>
> <Input id="{{id for=this}}-question" type="text" @value={{this.val}} />
> ```
> 
> You can pass any string, number, object, or other primitive as the named argument `for`, and the `id` helper will return the same ID from every invocation using that same `for` value.
> 
> You can also invoke `id` without the `for` argument to get a new unique id from every invocation. This is helpful within template-only components, where `this` is not available.
> ```hbs
> {{#let (id) as |myId|}}
>   <label for={{myId}}>
>     Ask a question about Ember:
>   </label>
>   <Input id={{myId}} type="text" @value={{this. val}} />
> {{/let}}
> ```
> 
> The aria-label attribute enables developers to label an input element with a string that is not visually rendered, but still available to assistive technology.
> ```hbs
> <Input id="site" @value="How do text fields work?" aria-label="Ember Question"/>
> ```
> 
> While it is more appropriate to use a <label> element, the aria-label attribute can be used in instances where visible text content is not possible.

### Accessibility guides
This helper will be an important part of Ember's out-of-the-box accessibility story. Future improvements to Ember's accessibility guides will be able to use this helper when discussing how to build forms and how to work with WAI-ARIA attributes such as `aria-controls` or `aria-describedby`.


## Drawbacks

Adding new helpers increases the surface area of the framework and the code the core team commits to support long term.

Some developers may not prefer the ergonomics of using the `{{id}}` with `let` blocks with-in template-only components.

Optional usage of the `for` argument may make this slightly harder to teach and understand, and may lead to fragmented usage patterns between template-only components and class-backed templates.

There is nothing about this proposal that could not be instead implemented in an add-on.

## Alternatives

1. Do nothing; developers can use backing classes for templates that require an ID, and either use elementId in classic components or import guidFor in glimmer components or via a hand-rolled helper.

2. Introduce a keyword-style syntax that leverages a build-time AST transform to convert this:
```hbs
<label for="{{id}}-toggle">Toggle</label>
<input id="{{id}}-toggle" type="checkbox">
```
into
```hbs
{{#let (id) as |_id|}}
  <label for="{{_id}}-toggle">Toggle</label>
  <input id="{{_id}}-toggle" type="checkbox">
{{/let}}
```
This approach would eliminate the `for` argument and `{{id}}` would always return a unique id. This approach is implementable in userland or in add-ons, so it may not be appropriate to consider this alternative for the core primitive introduced into Ember.js itself. 

This approach may be slightly more ergonomic but relies on a more magical, non-standard keyword-like API that will need to be specifically taught, and will be a larger maintenance burden.

## Unresolved questions

1. **Is `id` the most appropriate name for this helper?** A few alternative names were suggested, most notably  `unique-id`. It is my belief that the name `id` is best aligned with the current set of built-in helpers, which are generally as terse as possible. Additionally, I don't believe that the word "unique" is essential to the name of this helper because within the context of HTML/DOM any ID should be assumed to be necessarily unique.

2. The named `for` argument could instead be implemented as a positional param. Would this be a preferable API? We could potentially support both named or positional params for passing a stable reference.
# Function call context for ECMAScript

The idea is this:

- `function.meta` - The function call context, initially `Object.create(null)`
- `Function.prototype.callWithContext(context, thisArg, ...args)` - Call a function with various context keys and values as included in `context`, merging them with the existing context.

Here's some code to help elaborate a little further:

```js
const three = Symbol("three")

let rootMeta = function.meta
let fooMeta
let barMeta
let bazMeta

function foo() {
    fooMeta = function.meta
    bar.callWithContext({[three]: 3}, /* `this` value */ void 0, () => {
        console.log(function.meta === fooMeta) // true - bound to `this` context
        console.log(function.meta[three]) // undefined - bound to `this` context
    })
}

function bar(init) {
    barMeta = function.meta
    console.log(barMeta[three]) // `3`
    init()
    baz()
}

function baz(init) {
    bazMeta = function.meta
    console.log(bazMeta[three]) // `3`
}

foo.callWithContext({one: 1, two: 2})

// They don't follow the standard prototype chain, for safety.
console.log(rootMeta) // {[[Prototype]]: null}

// They inherit their parent contexts, to save on memory.
console.log(fooMeta) // {[[Prototype]]: rootMeta, one: 1, two: 2}
console.log(barMeta) // {[[Prototype]]: fooMeta, one: 1, two: 2, [three]: 3}
console.log(bazMeta) // {[[Prototype]]: fooMeta, one: 1, two: 2, [three]: 3}

// They only update whenever explicitly updated, to save on memory.
console.log(rootMeta === fooMeta) // false
console.log(fooMeta === barMeta) // false
console.log(barMeta === bazMeta) // true

// They're frozen, for additional safety and some extra optimizability.
console.log(Object.isFrozen(rootMeta)) // true
console.log(Object.isFrozen(fooMeta)) // true
console.log(Object.isFrozen(barMeta)) // true

// New realms create new root metas
let newRealmMeta = new Realm().eval("function.meta")
console.log(newRealmMeta) // {[[Prototype]]: null}
console.log(Object.isFrozen(newRealmMeta)) // true

// Call contexts don't penetrate realm boundaries
console.log(newRealmMeta === rootMeta) // false
console.log({}.isPrototypeOf.call(rootMeta, newRealmMeta)) // false
```

> Alternatively, these could be [records](https://github.com/tc39/proposal-record-tuple), but that's not a requirement for this proposal.

## Implementation

```js
const apply = Function.apply.bind(Function.call)
const {create} = Object
const {apply, ownKeys, defineProperty, preventExtensions} = Reflect

let currentContext = preventExtensions(create(null))

// `function.meta`, should be called and stored at start of function
export const getFunctionMeta = () => currentContext

Function.prototype.callWithContext = function (context, thisValue, ...rest) {
    const prevContext = currentContext
    const newContext = create(prevContext)
    const keys = ownKeys(context)
    for (let i = 0; i < keys.length; i++) {
        const key = keys[i]
        defineProperty(newContext, key, {
            enumerable: false,
            configurable: false,
            writable: false,
            value: context[key]
        })
    }
    preventExtensions(newContext)
    currentContext = newContext
    try {
        return apply(this, thisValue, rest)
    } finally {
        currentContext = prevContext
    }
}
```

Here's the code example from earlier transpiled to use that (minus the realm bit).

```js
import {getFunctionMeta as getFunctionMeta$} from './polyfill.js'

const three = Symbol("three")

let rootMeta = function_meta$
let fooMeta
let barMeta
let bazMeta

function foo() {
    var function_meta$ = getFunctionMeta$()
    fooMeta = function_meta$
    bar.callWithContext({[three]: 3}, /* `this` value */ void 0, () => {
        console.log(function_meta$ === fooMeta) // true - bound to `this` context
        console.log(function_meta$[three]) // undefined - bound to `this` context
    })
}

function bar(init) {
    var function_meta$ = getFunctionMeta$()
    barMeta = function_meta$
    console.log(barMeta[three]) // `3`
    init()
    baz()
}

function baz(init) {
    var function_meta$ = getFunctionMeta$()
    bazMeta = function_meta$
    console.log(bazMeta[three]) // `3`
}

foo.callWithContext({one: 1, two: 2})

// They don't follow the standard prototype chain, for safety.
console.log(rootMeta) // {[[Prototype]]: null}

// They inherit their parent contexts, to save on memory.
console.log(fooMeta) // {[[Prototype]]: rootMeta, one: 1, two: 2}
console.log(barMeta) // {[[Prototype]]: fooMeta, one: 1, two: 2, [three]: 3}
console.log(bazMeta) // {[[Prototype]]: fooMeta, one: 1, two: 2, [three]: 3}

// They only update whenever explicitly updated, to save on memory.
console.log(rootMeta === fooMeta) // false
console.log(fooMeta === barMeta) // false
console.log(barMeta === bazMeta) // true

// They're frozen, for additional safety and some extra optimizability.
console.log(Object.isFrozen(rootMeta)) // true
console.log(Object.isFrozen(fooMeta)) // true
console.log(Object.isFrozen(barMeta)) // true
```

## Why?

Consider how context is used in virtual DOM frameworks. Now imagine if you could do that with your models as well, injecting their dependencies without having to explicitly pass them around everywhere. Think of it as to functions as `import.meta` is to modules, except in this case, instead of the host determining what variables to pass (and potentially offering hooks into that), it's all purely in ECMAScript code.

Areas where it would prove instantly useful:

- Accelerating React Hooks and similar libraries
- Getting rid of most the variable passing in models
- Simplifying dependency injection code for testing and extensibility
- Passing models into virtual DOM frameworks without property drilling and without needing to use actual difficult-to-test and error-prone globals.
- Providing a mechanism for much more easily polyfilling language proposals like [implicit cancel tokens](https://github.com/tc39/proposal-cancellation/issues/28) and [zones](https://github.com/domenic/zones/tree/eb65c6d43b452a877c24561cd64c6901e790ecf0) such that they're actually transparent to code that doesn't know about it

Note that this can't be transpiled without awareness of others using it - otherwise, you risk differing views of the world and if you consume such context and intend to allow others to set it, you have to share the same version of the module throughout or it won't work. (This adds to the list of reasons why this should exist.)

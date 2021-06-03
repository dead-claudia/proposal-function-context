# Function call context for ECMAScript

The idea is this:

- `context = new Context(defaultValue)` - Create a new context, with an optional default value
- `value = context.get()` - Get the context's associated value in the running execution context.
- `result = context.use(value, () => result)` - Invoke a function with `value` as the context's associated value in its running execution context.
- On function call, all active contexts with their associated values are copied over to that new context. This includes cross-realm calls.

Here's some code to help elaborate a little further:

```js
const oneCtx = new Context()
const twoCtx = new Context()

console.log(oneCtx.get()) // undefined
console.log(twoCtx.get()) // undefined

const fooIter = oneCtx.use(1, () => foo())
console.log(oneCtx.get()) // undefined
console.log(twoCtx.get()) // undefined

fooIter.next()
console.log(oneCtx.get()) // undefined
console.log(twoCtx.get()) // undefined

function *foo() {
    console.log(oneCtx.get()) // 1
    console.log(twoCtx.get()) // undefined
    twoCtx.use(2, () => bar())
    console.log(oneCtx.get()) // 1
    console.log(twoCtx.get()) // undefined
    yield
    console.log(oneCtx.get()) // 1
    console.log(twoCtx.get()) // undefined
}

async function bar() {
    console.log(oneCtx.get()) // 1
    console.log(twoCtx.get()) // 2
    await Promise.resolve()
    console.log(oneCtx.get()) // 1
    console.log(twoCtx.get()) // 2
    fooIter.next()
    console.log(oneCtx.get()) // 1
    console.log(twoCtx.get()) // 2
}

oneCtx.use("one", () => {
    console.log(new Realm({globals: {oneCtx}}).eval(`oneCtx.get()`)) // "one"
})
```

## Why?

Consider how context is used in virtual DOM frameworks. Now imagine if you could do that with your models as well, injecting their dependencies without having to explicitly pass them around everywhere. Think of it as to functions as `import.meta` is to modules, except in this case, instead of the host determining what variables to pass (and potentially offering hooks into that), it's all purely in ECMAScript code.

Areas where it would prove instantly useful:

- Accelerating React Hooks and similar libraries
- Getting rid of most the variable passing in models
- Simplifying dependency injection code for testing and extensibility
- Passing models into virtual DOM frameworks without property drilling and without needing to use actual difficult-to-test and error-prone globals.
- Providing a mechanism for much more easily polyfilling language proposals like [implicit cancel tokens](https://github.com/tc39/proposal-cancellation/issues/28) and [zones](https://github.com/domenic/zones/tree/eb65c6d43b452a877c24561cd64c6901e790ecf0) such that they're actually transparent to code that doesn't know about it

Note that this can't be polyfilled without transpiler support, and that transpiler support is itself not trivial. That's what makes this a proper primitive operation.

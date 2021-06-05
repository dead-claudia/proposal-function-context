# Function call context for ECMAScript

Currently, there's the following options for dependency injection:

1. Use a global
1. Use a library that implements a shared namespace
1. Use explicit parameters
1. Use library/framework functionality to proxy it implicitly

And here's the issues with that:

1. Globals aren't easily tested, and it's an error-prone mess to set them up correctly. You also have to be careful to ensure it's reset after each test, to both prevent memory leaks and prevent it from interfering with other tests.
1. This is basically globals in disguise. You might get some framework assistance for testing, but ultimately, it's the same kind of boilerplate (and often requires *more* boilerplate).
1. If you've heard of "prop drilling" in React, that's basically what this is. This of course also includes function arguments and such, too. It's much harder to screw up in testing, but it's almost as boilerplatey as globals otherwise.
1. React and a few other virtual DOM frameworks provide a context feature that's similar to what I'm seeking here, just integrated with their execution model. But these are necessarily extremely intertwined with their data models.

And in addition, I've seen a series of language proposals that can't really be polyfilled or transpiled without also needing to transpile intermediate code, which in general isn't something you can rely on even being possible (like if you're working through built-ins or other native functions).

- [Global cancel tokens that are intelligently tracked](https://github.com/tc39/proposal-cancellation/issues/28) - this needs a mechanism to ensure child cancel tokens are tracked correctly so that when their parent cancels, they cancel as well, and so they don't need nearly the effort to connect properly (for one, none of us want to have to drill this through arguments).
- [Async context](https://github.com/legendecas/proposal-async-context) - this needs a way to track async operation inheritance.

## Proposal

I suggest we explore the possibility of function context.

- Readable through function boundaries
- Variables should be visibly constant to avoid surprises
- Highly restrained to limit risk of confusion

I'll start off with an API like this, just to get the ball rolling:

- `context = new Context()` - Create a new context, with an optional default value
- `value = context.get(defaultValue)` - Get the context's associated value in the current execution context.
- `result = context.set(value, func)` - Invoke a function with `value` as the context's associated value in its execution context.
- Not yet sold on how precisely to propagate from parent to child.

Of course, I'm not tied to this API, and am fully willing to take suggestions.

## Precedent

- Obviously, there's the various context implementations in React-like frameworks
- OS environment variables function a lot like that in shell scripting languages like Bash.
- Python has [`contextvars`](https://docs.python.org/3/library/contextvars.html), which isn't exactly the same, but is very, *very* close in concept. [They have a pretty detailed rationale of their own](https://www.python.org/dev/peps/pep-0555/#rationale), which overlaps heavily with both this and [the async context strawperson](https://github.com/legendecas/proposal-async-context).
- C++ `#define`s are often used in headers specifically to set settings similarly to context as named here.

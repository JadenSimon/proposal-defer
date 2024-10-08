# ECMAScript `defer` Statement

We propose the introduction of a new control-flow statement into ECMAScript: `defer`. This statement executes code unconditionally at the end of the current scope in a last-in-first-out (LIFO) manner.

```js
defer console.log('world!');
console.log('hello, ');
// hello,
// world!
```

## Motivations
The primary motivation for `defer` is to simplify state management. Managing the lifecycle of transient state (e.g. flags and counters) and system resources (e.g. file handles, network connections) can be tricky, especially when ensuring cleanup occurs consistently across different state and/or different control flows like exceptions, early returns, or loops. `defer` provides a straightforward, minimal syntax that guarantees code execution at the right moment, reducing friction, boilerplate, and opportunities for user-error.

### Why `defer` instead of XYZ?
* Minimal syntactic overhead
    * Only 1 new keyword without any novel ordering of keywords (`defer await` vs. `await using`)
* No additional protocol or interfaces
    * Abstraction agnostic, can be used across all development environments (Node.js, Bun, Deno, web frameworks, etc.)
    * Immediate value to developers without needing the (huge) ecosystem to catch-up
* Low Cognitive Load
    * `defer` does one thing: defers execution until the end of a scope
* Naturally fits into ECMAScript's control flow without introducing inconsistencies
* Simplifies refactoring by only ever requiring re-arrangements of code

### Inspirations
`defer` has already been implemented and widely used in several popular languages:

* Go
* Zig
* Swift

The behavior in this proposal aligns nicely with Swift/Zig (LIFO, supports blocks) and is only slightly different compared to Go's function-scoped `defer`. This proposal adapts `defer` to fit naturally into ECMAScript while handling the quirks of the language.


## Behavior
A `defer` statement has an async variant: `defer await` that awaits the code execution. The following behaviors apply to both `defer` and `defer await` unless otherwise specified. 

* The expression statement or block following `defer` is unconditionally executed when the current scope exits. This can be from:
    * `return`
    * `throw`
    * `break`
    * `continue`
    * The end of each iteration of a loop
    * End of a block
* `defer` statements in the same scope contribute to a shared stack. Statements and blocks added by `defer` are evaluated in reverse-order i.e. last-in-first-out (LIFO).
    * Blocks are evaluated **as a group**, not as individual statements
    * Statements never contribute to outer scopes, only the immediate containing scope
* Blocks are evaluated with distinct scopes; they do not share declarations
    * This is OK: `defer { const x = 1; } defer { const x = 2; }` 
* Exceptions thrown by individual `defer` statements do not stop the remaining statements in the stack from executing.
    * Multiple exceptions accumulate in a `SuppressedError` construct that forms a chain of errors, much like `Error`'s `cause`
* `defer await` allows the usage of `await` within a block; `defer` does not regardless of the surrounding context.

Requiring `defer await` to use `await` was added for two reasons:
    1. Removes confusion surrounding the behavior of something like `defer foo(await bar())` (do we `await` `bar()` at scope exit?)
    2. Simplifies implementation logic, both for engines and transpilers

## Grammar
```
DeferStatement[Await]:
    defer DeferBlockOrExpressionStatement[~Await]
    [+Await] defer await DeferBlockOrExpressionStatement[+Await]

DeferBlockOrExpressionStatement[Await]:
    DeferBlock[?Await]
    DeferExpressionStatement[?Await]

DeferBlock[Await]:
    { DeferBlockBody[?Await] }

DeferBlockBody[Await]:
    StatementList[~Yield, ?Await, ~Return] opt 

DeferExpressionStatement[Await]:
   [~Await] [lookahead ≠ (] ExpressionStatement[~Yield, ~Await]
   [+Await] ExpressionStatement[~Yield, +Await]
```

`DeferBlock` is similar to a class static block except that `await` is conditionally allowed.

Note that `await` is not treated as a contextual keyword. That is, `defer await` is always invalid outside of an Await context. This is done for simplicity.

## Valid Syntax

### Synchronous Defer
```js
defer listener.dispose();
defer console.log('finally!');

defer {
    const data = finalizeData();
    logger().log('category', data);
}

// LIFO ordering
function foo() {
    defer console.log('3');
    defer {
        console.log('1');
        console.log('2');
    }
}

// Output:
// 1
// 2
// 3

// Nested functions
function outer() {
    defer console.log('outer');
    function inner() {
        defer console.log('inner');
    }
    inner();
}

// Output:
// inner
// outer

// Loops
for (let i = 0; i < 3; i++) {
    defer console.log(i);
    console.log('looped');
}

// Output:
// looped
// 0
// looped
// 1
// looped
// 2

// `yield` is _not_ a scope exit
function* gen() {
    defer console.log('done');
    yield 1;
    yield 2;
    yield 3;
}

for (const x of gen()) console.log(x);

// Output:
// 1
// 2
// 3
// done

// `throw` is allowed in blocks, this may "suppress" already thrown exceptions (same as the `using` proposal)
defer {
  throw new Error('uh oh!');
}

// Exceptions can be caught by a containing scope
try {
    defer { throw new Error('uh oh!'); }
} catch (e) {
    console.log(e.message) // uh oh!
}

// Try statements do not change the behavior of `defer` regardless 
// of whether it's in a `try`/`catch`/`finally` block. 


```

### Asynchronous Defer
```js
defer await file.close();

defer await Promise.all([
  resource1.dispose(),
  resource2.dispose(),
]);

// The block variant of `defer await` offers the most granular level of control

defer await {
    await resource1.dispose();
    // ... do something else
    await resource2.dispose();
}

// We can wait for certain things on scope exit while letting other things happen as we move on
// The same could be achieved with a separate async function declaration
defer await {
    const task = await myPendingTask;
    if (shouldCancel(task)) {
        console.log('cancelling task without waiting');
        task.cancel().catch(e => console.error('failed to cancel task', e));
    }
}
```

## Invalid Syntax

Situations introduced by `defer` that lead to SyntaxError:
```js
// We cannot return anything because `defer` statements are always executed after normal terminating control flow (`return`, `throw`, etc.)
// The same applies for `break`, `continue`, and `yield`
defer return foo;
defer { return foo; }

// `defer await` is not allowed in a non-async function or the top-level of a Script
function foo() {
    defer await;
}

// We cannot use `await` in a normal defer despite being in an async function or the top-level of a Module
async function foo() {
    defer {
        await resource.dispose();
    }
}

// No import declarations
defer {
    import foo from 'bar'
}


// Cannot appear as the sole statement of an iteration statement
for (const x of arr) defer x.dispose();

let i = 0;
while (i < 3) defer console.log(i++); 

// This is the same behavior as `while (true) const x = 0`
// While technically possible to support, doing so creates a confusing situation, does 
// `defer` execute per-iteration or in the outer scope, that also lacks practical utility.
//
//  Omitting the `defer` keyword results in the same net behavior without the ambiguity.
```

Other situations which lead to SyntaxError (it's the same behavior as `try`):
```js
// All cases below would result in `SyntaxError: Unexpected identifier 'y'` in V8

// As an expression
const x = defer y;

// In a class body
class MyClass {
    defer y; 
}
```


## Oddities
The flexibility of `defer` means it can be used in "interesting" ways that may not always have clear behavior for some readers. While the behavior in the spec is clear and consistent with the rest of ECMAScript, the novelty can leave some room for interpretation to readers. Many of these cases will likely be rarely used in practice but can still exist. `defer` itself does not create these oddities per-se, but rather adds a new mechanism to make them more apparent. This section disambiguates these cases.


### Switch Statements
`defer` statements accumulate in the `switch` statement block and will execute when leaving said block (via `break`, `return`, reaching the end of the block, etc.)

A `defer` in a case clause that is otherwise not in a separate scope adds to the containing `switch` block. 

```js
switch ('defer-me') {
    case 'defer-me': {
        defer console.log('one'); 
    }
    case 'defer-me': defer console.log('three');
    default: console.log('default');
    case 'defer-me': 
        defer console.log('two');
        break;
    case 'defer-me': defer console.log('last');
}
console.log('outside switch')

// Output:
// one
// default
// two
// three
// outside switch
```

This behavior is consistent with variable declarations in case clauses:

```js
switch ('foo') {
    case 'foo':
        let x = 1
    case 'foo': x = 2
    case 'foo': {
        let x = 'purple'
    }
    default: console.log(x) // 2
}
```

Diverging from this existing behavior is not worth it given the likely niche usefulness of `defer` within `switch` statements.
Advanced ECMAScript users are the most likely to run into this situation, and they're likely to be familiar with the existing behavior.

### Parenthesized Expressions
`defer` (without `await`) followed by a ParenthesizedExpression is interpretted as a CallExpression

```js
// Attempt to call `defer` with the result of `console.log`
defer (console.log(''));

// Normal `defer` statements
defer void (console.log(''));
defer { (console.log('')); }
```

This is because `defer ()` is already perfectly valid syntax; we cannot break backwards compatability.



## Potential Usage Concerns
This section addreses common usage concerns from adding `defer` to the language

### LIFO Evaluation
`defer` statements are evaluated in an order _opposite_ of their appeareance in source code. For someone first learning about `defer`, this can seem unnatural:
```js
defer console.log('one');
defer console.log('two');

// Output:
// two
// one
```

However, this seemingly unnatural behavior quickly becomes an asset, not a liability:
```js
const r1 = new Resource();
defer r1.dispose();

const r2 = new Resource(r1);
defer r2.dispose();
```

The LIFO behavior ensures that dependent state is always unwound before its dependencies while still providing a natural ordering of statements. This code would have safety flaws without LIFO:

```js
const r1 = new Resource();
const r2 = new Resource(r1); // `r1` would not be disposed if we fail here!
defer r2.dispose();
defer r1.dispose();
```

In practice, the LIFO behavior generally goes unnoticed by users, minimizing cognitive overhead.


### Nested Situations
In all cases, `defer` only affects the immediate containing scope. So regardless of the depth of nesting, the behavior of each individual scope is well-defined. From there, you can determine the flow of the entire program by chaining together the behavior of each scope.

```js
function bar(i) {
    defer console.log(`defer bar ${i}`);
    console.log(`bar ${i}`);
}

function foo() {
    defer bar(1)
    defer bar(2)
    console.log('foo')

    return () => {
        defer console.log('two');
        console.log('one')
    }
}

foo()()

// Output:
// foo
// bar 2
// defer bar 2
// bar 1
// defer bar 1
// one
// two
```

While there is some amount of added cognitive overhead from `defer`, many will find it easier to understand than the equivalent `try/finally` structure:

```js
function bar(i) {
    try {
        console.log(`bar ${i}`);
    } finally {
        console.log(`defer bar ${i}`);
    }
}

function foo() {
    try {
        console.log('foo');

        return () => {
            try {
                console.log('one')
            } finally {
                console.log('two')
            }
        }
    } finally {
        try {
            bar(2)
        } finally {
            bar(1)
        }
    }
}
```

## Performance Concerns
`defer` is well-suited for code generation. Conceptually, each statement can be treated as an arrow function with no parameters, pushed onto a scope-isolated execution stack that is consumed on scope exit. Static analysis works well here because `defer` statements are never conditionally added to this stack. 

For a given scope, we can construct a linked-list of `defer` statements where each node points to the _previous_ statement (likely during parsing). The first `defer` statement in a nested scope can point to the last encountered statement (if any) in containing scopes. This means you only need a pointer to the most recently encountered `defer` statement. Upon scope exit, we unwind using this pointer by iterating and executing the statements.

We can use this information to generate machine code directly by understanding that, for a given "exit" point (`return`, `break`, `continue`, `throw`, block end), the `defer` statement pointer will always be the same.

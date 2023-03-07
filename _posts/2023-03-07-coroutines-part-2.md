## Language design choices

The previous post discussed the implementation details of coroutines. This post 
will discuss the language design elements of coroutines.

### The big choice: Explicit or implicit calls

This is the choice between marking every call from one coroutine to another with a special type
vs the choice of special code to reify a coroutine. 


For example this snippet uses marking of every call:

```js
async fn foo(x, y, z) {
    baz(x)
    await bar(y)
    let task = bar(z)
    return await task
}
```

This snippet needs a special marker to create a task.

```js
async fn foo(x, y, z) {
    baz(x)
    bar(y)
    let task = suspend bar(z)
    return task.runAndWait()
}
```

The second option often can end up looking cleaner for async-like coroutines 
where most of the time we want to simply call tasks and not just use them.
The first can be nicer for generator like coroutines where we often want to 
use the object produced (possibly wrapped as an iterator) directly.

The first option allows `await` as light syntactic sugar. 
The first case could expand to something like the following
(although how much is merged into a single call to the routine depends on how the coroutine works):

```js
async fn foo(x, y, z) {
    bar(z)
    let t = bar(y)
    while (!_t.done())
        yield _t.next()
    let task = bar(z)
    while (!task.done())
        yield task.next()
    return task.result()
}
```

This is the call part of "stackless coroutines without trick".
Even if we wanted special optimization around this we know all the interesting parts are marked with `await`. 

With more stackful options the second option could be nicer to implement. Here the coroutine calls
are just normal calls and the suspended task are special so it becomes something like:

```js
async fn foo(x, y, z) {
    baz(x)
    bar(y)
    let task = suspend(async () =>  bar(z))
    return task.runAndWait()
}
```

Here the magic gets to live in functions. Suspend will set-up the stack and run and wait will 
run the code on that stack forwarding suspends to the original caller. Note here we have a major 
performance difference between the two calls to bar.

If we had types we would be able to tell which case we are in and therefore use either desugaring.
However the major performance difference would be much more hidden.

There is a third option which is both but that seems the worst of both worlds.

```js
async foo(x, y, z) {
    baz(x)
    await bar(y)
    let task = suspend bar(z)
    return await task.runAndWait()
}
```

### Typing

The differences above become even larger when we look at typing. We have two main
options for typing. I will call these the "function type" approach and the "result type"
approach.

I will explore this in an `async` context where the coroutine is a `Task` type.

#### Result typing

The result type approach most makes the "async-ness" a normal part of the type system 
and makes `async` in some ways just an implementation detail. So you may write

```ts
async fn foo(x : number): Task<string>
```

The type of `foo` is the same as if you had written

```ts
fn foo(x : number): Task<string>
```

and returned a task directly.

You may or may not need to wrap the result depending on the language. So in the language 
you may write 

```ts
async fn foo(x : number): string
```

and it becomes

```ts
fn foo(x : number): Task<string>
```

Essentially though we get a normal function out the other end and everything proceeds as normal.
In an `await` style thing `await` would convert `Task<T>` into `T`. This is more complex for a `suspend` style.
Here we would automatically strip `Task` in an `async` context unless `suspend` was called. 
With the `suspend` style we also have the question of how many `Task`s we strip and how to deal with generics
that end up as `Task`s.


#### Function typing.

With the function typing style the `async` becomes part of the type. That means a

```ts
async fn foo(x : number): string
```

is different from

```ts
fn foo(x : number): Task<string>
```

This generally requires more type system integration as it affects the functions themselves.
They also need to be considered in almost all higher order things as well directly.
If there are user defined coroutines then the set of function types has to be user defined as well.

This lends itself to the `suspend` style. Here we just let calls from `async` functions into other
`async` functions. And `suspend` converts an `async` function into a task.

`await` style is more complex here. An `await` call would let you call an `async` function and
non await calls would decay into `Task`s (or maybe we would be in the both case and we would need to call suspend). 
Perhaps we would separate `await_task` for `awaiting` a task or we could write `await task.run()` and have `async fn run(self: Task<T>): T`


#### The general function shape problem

One thing that sometimes comes up is practically we end up with multiple types of
function even if we have only a single actual type. This is sometimes talked about as "colors" of functions.

These functions split everything in 2. Thing that work with `async` and things that don't. 
This gets worse with more complex coroutines. 

One option to make this better is to somehow abstract over coroutine types. This would
let us have things like `fold` that work over multiple types. We have to be careful though
because sometimes it is wrong to suspend in the middle of a function. For example a parallel map-reduce
can't really abstract over coroutines as each part is in a different thread so can't really suspend.
Next post will talk about one way we can do this.

Fundamentally having 2 kinds of function is unavoidable for coroutines. We have multiple kinds of function. What should we do if we suspend
in the middle of a function that doesn't expect is? This really then asks why do we want `async` as a coroutine. 
Practically the answer here is we want to pick between the tradeoffs in the previous post rather than being forced into OS threads.

### Conclusions

I think it is clear that everything seems to be splitting into two major choices
"`suspend` + stackful + function typing" vs "`await` + stackless + result typing" 
although the link between the typing and `await`/`suspend` is stronger than the
"stackless/stackful" links.

Anything with "function typing" is going to be more complex at the type system level
but may be more naturally how users think. It does end up with two equivalent ways of thinking
about it as a normal object vs a special type of function. Being a normal object does
allow easily being composed like a normal object but we have access to conversions
in the "function typing" case.

It seems like for `async`, `suspend` style has much less noise, but then again
for generators it is the other way around.


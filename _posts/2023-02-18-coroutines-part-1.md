## Coroutines and their implementation

Coroutines are a useful feature in programming languages but
many implementation details end up leaking into the abstraction 
(mostly in unavoidable ways). This post explores ways in which 
they are implemented an their features

### What is a coroutine?

A coroutine is a sequential part of a program that can be suspended and
be resumed. Normally basically means that it is something that looks like
a normal function/method/procedure that has access to a `yield` operation
that suspends  it's own execution, giving a value to a caller. When the coroutine is
resumed with a value the `yield` operation results in that value. They also
still return a value at the end as usual. Note that in many cases these values are missing.

A generators is a coroutine which isn't given a value on resumption and has no result.
An `async` function which repeatedly `yields` no value to some scheduler 
before returning a result value. 

Note that often the full generality can be recovered from the weaker ones by having some place
to put the values during the control flow transfer.

This definition leaves a couple of things open. The first is how do we call
a coroutine from a normal method. Most implementations end up returning some coroutine object 
that has a `resume` operation. The second is how we can call a
coroutine directly from a coroutine. Some implementations see a call from 
a coroutine to another coroutine and just transparently treat it as a coroutine call.
Other systems make you use the normal operation and either make the callee repeatedly
yield the values yielded from the inner version or have a special operation for this repeated yield.


### Why do we want them?

They are generalized versions of generators and async. Having a common base can mean
only one implementation has to be made in the core of the compiler. There are also other uses
such as a simple ai in a game which yields a move each turn. In general they are often useful whenever
the flow of logic doesn't match the flow of time.

### How are they implemented

There are 3 different common methods. They are threads, stackless coroutines and fibers.

Fundamentally there are two decisions that need to be made. The first is how we represent the stack frame(s) and 
program counter (where we are in the co-routine).
We can either allocate a full normal stack, or allocate individual frames on the heap. The second is how we 
do a context switch. Either we use the OS or we implement it manually.

#### Thread based implementations

Thread based implementations basically spin up a new thread and rely on synchronizing between threads to 
pass the yielded value and to ensure only one thread is doing work at once. The main cost is that they are 
fairly heavyweight to create and also use a lot of resources. The advantage is that they are fast at running
straight line code and programming languages already have them. Coroutine to coroutine calls are also good.
The thread switch on the other hand is often slow. We are also unlikely to be able to optimize yields/resumes
as they are just thread synchronization.

#### Fibers

Fibers or lightweight threads are a threading model implemented outside the operating system themselves. By avoiding 
doing all the things an OS must do we save some resources. For example we can avoid the cost of context switching into kernel code.
However they are often still quite expensive. The main issue comes with the stacks. Either we allocate a full stack
or we allocate a small stack. If we allocate a small stack then we have to decide what to do when we run out of stack.
One option is to allocate a larger stack. This requires either reserving all the virtual address space before hand 
and allocating on demand (reasonably ok on 48-bit address spaces, bad on 32-bit) or having some form of linked stack structure, or
relocating the stack. All then have problems with C-abi code that doesn't understand the special stack. You may think that this is OK
and we can just switch stack to a C-abi stack before hand except there is the problem of signal handlers which want a C-stack. Another problem
is the cost of calling c-abi code is much higher and some of these functions may be hot. One option is to use a different stack from the c-stack.
This is mostly OK except you lose a register and you have to change your compiler (e.g. llvm can't just do that). Finally if you have a special stack
using the normal stack register either all code must understand the special stack or you need to switch stack when doing a coroutine to normal call.

Another issue is optimization. In general we can't optimize through coroutine yields here as after desugaring the coroutines as the code is complex.
This can be an issue for very small coroutines.

#### Stackless coroutines

With a stackless coroutine we capture a single stack frame and execution location into a single object. The advantage
of this is basically zero runtime and it is ver efficient for small coroutines. The problem comes with calling other coroutines.
We end up either with a linked list of stack frames where we traverse the list for each call yield (in the table below I have attributed this as coroutine call cost).
We can inline callee frames into the main frame to avoid some of the costs.

This form of coroutine will also allow us to do some other tricks later.

There is a way for us to avoid the traversal cost by storing a pointer the bottom frame and a function pointer to the type of the bottom frame. 
We can then jump straight to the bottom frame. Again with the inlining trick this can be made better and we could specialize frame types to do a 
switch based lookup instead of a virtual call.

#### Summary

| Method       | setup cost | static resource cost |yield cost | coroutine call cost | ability to optimize desugared code | implementation cost            |
|--------------|------------|----------------------|-----------|---------------------|------------------------------------|--------------------------------|
| thread       |   high     |   high               |  high     |      low            |              low                   |  almost zero (if threads already exist)|
| fibers with full stacks | medium        | medium |  medium   |      low            |              low                   |  low                           |
| fibers with small stacks|  low           | low   |  medium   |      low            |              low                   |  medium                        |
| stackless coroutines without trick | low | low   |  low      |      high           |             high                   |  high                          |
| stackless coroutines with trick | low |   low   |  medium   |      medium         |            medium                  |  high                          |

### Additional considerations in languages that avoid allocation.

With fibers/threads there is a massive allocation. This makes them hard to use without avoiding allocation. For stackless
coroutines as long as there is no recursion we can allocate a single object including all the frames that could be in the list. If it doesn't last long
we can keep it on the original stack frame. This is very useful for generators where the caller often keeps them on a single frame. 
On the other hand we may have to think about moving the frames though or avoid moving them. Depending on the programming language this may be easy
(i.e. it doesn't matter with GC) or very hard (rust has to do fancy stuff with `Pin` as without it everything is assumed to be movable by simple memory movements). 


### Which implementation should I use?

Basically the main trade offs depend on how long lived your coroutines are. For short-lived small coroutines, stackless coroutines are clearly best between
the optimization and the static resource costs. If you require no allocation stackless is also only option. However stackless coroutines do require more language support.
If performance is less important threads or fibers would work. With fibers you do need to think about the C-abi issues unless you avoid the c stack and that often permeate
your entire backend. 

Finally the choices may affect language design which I will loom at further in part 2.




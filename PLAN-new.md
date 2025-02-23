https://frontside.com/blog/2023-12-11-await-event-horizon/

1) Introduction

Today I will talk about Concurrency

2) Why Concurrency is hard

"Concurrency is about dealing with a lot of things at once" (Rob Pike)

- Managing efficiently concurrency, avoiding starvation  
- Dealing with race conditions, cancelation, deadlocks, resource leaks
- Orphan tasks/threads/processes

3) Let's take for instance an example with a short snippet of code using async/await

That problem is referred as the await event horizon https://frontside.com/blog/2023-12-11-await-event-horizon/
 
Explicit resource management

-> Still the same problem, it's just syntaxic sugar around try/finally, but does not
solve the issue of having primitives supporting cancelation

Abort Signals?

-> They offer some kind of way, but highly-verbose with low-guarantees: the consumer is
free to use or not the signal, so we can't use it as a primitive to ensure...

The issue

The consequence of this is that the minimum lifetime of any given async function is determined solely by the lifetime of its innermost promises. In other words, the natural lifetime of an async function is determined from the inside out.

Namely, that the maximum lifetime of a function is constrained by the lifetime of the function that calls it.

4) Structured Concurrency

We need better ways of dealing with it

The goal is to kind of follow the path of an operating system

Well-Defined Scope: Tasks are tied to their parent scope, ensuring automatic cleanup.
Hierarchy and Composition: Child tasks cannot outlive their parents, making failures easier to reason about.
Implicit Cancellation & Propagation: When a parent task is canceled, all children are canceled automatically.
Better Debugging & Readability: The execution flow remains predictable, reducing "orphan" tasks and unexpected behaviors.

5) How Effect implements Structured Concurrency

Scoped Task Management: Automatic cancellation of child tasks on parent failure.
Resource Safety: Guarantees cleanup even in error scenarios.
Fiber-Based Runtime: Uses lightweight fibers to manage tasks efficiently.

6) Why using Fibers

In JavaScript, generators can approximate fibers but lack critical features like preemptive resumption and parallel execution.
Many fiber-based runtimes (such as Effect's runtime, Kotlin Coroutines, and Ruby Fibers) leverage delimited continuations under the hood to manage task execution.

Fibers remain scoped to their parent task.
When a fiber is canceled, all of its children are also canceled (hierarchical execution).
Efficient context switching happens entirely in user space without OS thread overhead.

1. Lightweight and Efficient Execution

2. Fast Context Switching

3. Precise Control Over Execution Flow

4. Seamless Integration with Structured Concurrency

7) I hope you enjoyed the talk, and if not, I'll see you at the [insert_activity]
---
# try also 'default' to start simple
theme: dracula
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  ## Antoine Coulon, Effect Days #2
  
  Structured Concurrency, the hidden power behind Effect
# persist drawings in exports and build
drawings:
  persist: false
# use UnoCSS (experimental)
css: unocss
---

## Structured Concurrency, the hidden power behind Effect

<br>
<br>

**Antoine Coulon** @ **Effect Days** #2

---

<div class="grid grid-cols-10 gap-x-4 pt-5 pr-10 pl-10">

<div class="col-start-1 col-span-8 grid grid-cols-[3fr,2fr] mr-10">
  <div class="pb-4">
    <h1><b>Antoine Coulon</b></h1>
    <div class="leading-8 mt-8 flex flex-col">
      <p class="mt-3">Freelance Lead Software Engineer @ <b color="orange">Seanine Consulting</b></p>
      <p class="mt-3">Author <b color="orange">skott, digraph-js, effect-introduction</b></p>
      <p class="mt-3">Advocate <b color="orange">Effect</b></p>
    </div>  
  </div>
  <div class="border-l border-gray-400 border-opacity-25 !all:leading-12 !all:list-none my-auto">
  </div>

</div>

<div class="pl-20 col-start-9 col-span-10">
  <img src="/anc.jpeg" class="rounded-full w-40 margin-0-auto" />

  <div class="mt-5">
    <div class="mb-4 flex justify-between"><ri-github-line color="blue"/> <b color="opacity-30 ml-2">antoine-coulon</b></div>
    <div class="mb-4 flex justify-between"><ri-twitter-line color="blue"/> <b color="opacity-30 ml-2">c9antoine</b></div>
     <div class="mb-4 flex justify-between"><ri-linkedin-box-fill color="blue"/> <b color="opacity-30 ml-2">Antoine Coulon</b></div>
    <div class="mb-4 flex justify-between"><ri-user-3-line color="blue"/> <b color="opacity-30 ml-2">dev.to/antoinecoulon</b></div>
  </div>
</div>

</div>

<style>
  h1 {
    color: #4c7fff;
  }
  img {
    margin: 0 auto;
  }
</style>

---

## Implementing the right Concurrency model is hard

> "Concurrency is about dealing with a lot of things at once" (Rob Pike)

<br>

- Avoiding starvation, deadlocks, resource leaks, race conditions 
- Offering resource-safety through cancellation with proper finalization mechanisms
- Dealing with error management and propagation
- Providing efficient concurrency (CPU + Memory)
- APIs gaps between concurrent and non-concurrent code 


<div>

  <div class="grid grid-cols-[3fr_2fr] gap-x-4 pt-1">
    
  <div class="pt-15">
    <b>⚠️ An Event-Loop doesn't solve all concurrency issues</b>
  </div>

  <div class="flex justify-center">
    <img src="/Libuv.png" width="200" />
  </div>

  </div>

</div>

---

## Concurrency with JavaScript: few problems

> It's too easy to lose control over execution and leak resources


<div>

<div class="grid grid-cols-2 gap-x-1">

<div>

```ts
async function acquireUseRelease(work) {
  let resource = await acquire();
  try {
    // What if it never settles?
    await work(resource);
  } finally {
    // Never happens
    release(resource);
  }
}
```
</div>


<div class="pt-1">

  ```ts
  async function work(tasks) {
    try {
      // Unbounded execution
      const result = await Promise.all(tasks);
    } catch {
      // In case of Failure, other Promises are 
      // still running even though Promise.all 
      // is already settled
    }
  }
  ```
  
  </div>

  </div>

  <div class="flex justify-center">

  → Execution **control is easily lost** <br><br>
  → Zero control over **concurrent execution** <br><br>
  → Poor control over **resource management** <br><br>

  </div>




</div>




---

## Do we have builtin solutions for these problems?

<div class="grid grid-cols-2 gap-x-4 pt-5">

  <div v-click>

    Explicit resource management 


```ts
export async function acquireUseRelease(resource) {
  using resource = await acquire();
  await work(resource);
  // When leaving the scope of that function, "resource" 
  // is released following methods defined on 
  // Symbol.dispose or Symbol.asyncDispose
}
```

<div class="pt-2">

→ Automatic scope tracking and isolated cleanup following `try/finally` semantics
<br>

→ We are still bound to `work()` lifetime

</div>

  </div>

  <div v-click>

    Abort Controller API
    

  ```ts
  async function acquireUseRelease(work, signal) {
    let resource = await acquire(signal);
    try {
      await work(resource, signal);
    } finally {
      release(resource);
    }
  }
  ```

  → Good if the signal is correctly **managed** and **drilled down** within the **entire operation graph**
  <br>

  → Firing the signal is a request, **not a guaranteed cancellation**


  </div>

</div>

---

## The real solution: Structured Concurrency

> A hierarchical execution model following the path of an operating system

<div class="grid grid-cols-2 gap-x-4 pt-1">

  <div>
  <p>🔗 Resource Management: children are tied to a parent scope, ensuring automatic cleanup.</p>
  
  <p class="pt-4">⏳ Bound lifetime: by default, a child can not outlive its parent lifetime.</p>
  
  <p class="pt-4"> ❌ Cancellation & Propagation: When a parent task is cancelled, all children are cancelled automatically.</p>


  <div class="flex gap-10 pt-7">
    <img width="100" src="/kotlin.png" />
    <img width="100" src="/swift.png" />
    <img width="100" src="/java.png" />
  </div>
  
  </div>

  <div class="text-center">

```mermaid {scale: 0.8}
graph TB

A[Runtime] -->|Manages| B[Root Task]
B[Root Task] -->|forks| C[Child Task A]
B[Root Task] -->|forks| D[Child Task B]
C -->|forks| E[Child Task C]
```

  </div>

</div>

---

## 🇮🇹 🤌 Structured Concurrency with Effect

> Hold my Structured Concurrency... you don't even need to think about it


<div class="grid grid-cols-2 gap-x-4 pt-1">

<div>

```ts {all|12-17,19-21|all}
import { Console, Duration, Effect, pipe } from "effect";

const fetchUserById = (id: number) =>
  pipe(
    Effect.async((resolve) => {
      console.log(`Start fetching ${id}`);

      const timeout = setTimeout(() => {
        resolve(Effect.succeed(`User ${id}`));
      }, 1000 * id);

      return pipe(
        Effect.sync(() => clearTimeout(timeout)),
        Effect.tap(() => Console.log(
          `Timer #${id} cleared`)
        )
      );
    }),
    Effect.onInterrupt(() => 
      Console.log(`Fetch User#${id} interrupted`)
    )
  );
```


</div>

<div>

```ts {all|1-3,10-12|all}
const interruptProgram = Effect.interrupt.pipe(
  Effect.delay(Duration.seconds(2))
);

const program = pipe(
  Effect.all(
   [fetchUserById(1), fetchUserById(2), fetchUserById(3)], 
   { concurrency: "unbounded" },
  ),
  Effect.zip(interruptProgram, {
   concurrent: true,
  }),
).pipe(Effect.runFork);
```

```bash
$ Start fetching 1
$ Start fetching 2
$ Start fetching 3
$ User#1 fetched
$ Timer#2 cleared
$ User#2 fetching interrupted
$ Timer#3 cleared
$ User#3 fetching interrupted
```

</div>


</div>

--- 

## A declarative way of dealing with Concurrency

> Following the path of Structured Programming: improving quality, clarity and productivity of a program

<div class="grid grid-cols-2 gap-x-4 pt-6">

<div>

→ Concurrency is **declarative** and **explicit**

→ **Unified API** to leverage (or not) concurrency at will

→ Everything happens automatically **behind the scenes** (thanks runtime)


</div>

<div>

```ts
import { Effect } from "effect";

const userIds = Array.from(
  { length: 1000 }, (_, idx) => idx
);

const retrieveAllUsers = pipe( 
  userIds,
  // SEQUENTIAL (default)
  Effect.forEach(
    (id) => fetchUserById(id), 
    // BOUNDED CONCURRENCY
    { concurrency: 30 },
    // OR UNBOUNDED CONCURRENCY
    { concurrency: "unbounded" },
    // OR INHERITED CONCURRENCY
    { concurrency: "inherit" }
  )
);
```

</div>

</div>

---

## 👨‍👧‍👧 Automatic Supervision

> The child Fiber is automatically supervised by the parent Fiber

<div class="grid grid-cols-2 gap-x-4 pt-1">

<div>

`Effect.fork`: child is tied to the parent lifetime

```ts
const background = pipe(
  Effect.log("background"),
  Effect.repeat(Schedule.spaced("5 second"))
);

const foreground = pipe(
  Effect.log("foreground"),
  Effect.repeat({ 
    times: 2, 
    schedule: Schedule.spaced("1 second") 
  })
);

const program = Effect.gen(function* () {
  yield* Effect.fork(background);
  yield* foreground.pipe(
    Effect.onExit(() => Effect.log("Bye"))
  );
});
```

</div>

<div class="pt-5 text-center">

<div class="grid grid-cols-2 gap-x-4 pt-8">


</div>

```bash
level=INFO fiber=#0 message=foreground
level=INFO fiber=#1 message=background
level=INFO fiber=#0 message=foreground
level=INFO fiber=#0 message=foreground
level=INFO fiber=#0 message=Bye
```

<div class="pt-4 text-center">
```mermaid {scale: 0.8}
graph TB
Root[Parent Fiber #0] -->|Forks into Child Fiber #1| A[Child Fiber #1]
```
</div>

</div>




</div>

---

## 💨🏃‍➡️ Escaping the Parent Supervision

> Forking Fibers detached from parent Fibers using Scopes

<div class="grid grid-cols-2 gap-x-4 pt-1">

<div>  
  <div class="pt-5">

  ```ts {all|14-15} {lines:true}
  const program = Effect.gen(function* () {
      // ^ Effect.Effect<void, never, Scope>
    yield* Effect.forkScoped(backgroundJob)
  });

  const main = Effect.scoped(
    Effect.gen(function* () {
      // Local scope starts
      yield* Effect.fork(program)
      yield* Effect.sleep("10 seconds")
      // Local scope ends
    })
  );
  ```
  </div>

  <div class="pt-5">

  ```ts {all|1,3} {lines:true}
  const anotherProgram = (scope: Scope.Scope) =>
    Effect.gen(function* () {
      yield* Effect.forkIn(scope)(backgroundJob);
    });
  ```

  </div>

  
</div>

<div class="pt-5 text-center">

```mermaid {scale: 0.8}
graph TB
F[Fiber Runtime] -->|Manages| Root
F[Fiber Runtime] -->|Manages| App
User[Application] -->|Manages| App
App[Provided Scopes: forkIn, forkScoped] -->|Lifetime now bound to given Scopes| A 
Root[Root Fiber #0] -->|Forks into Child Fiber #1| A[Child Fiber #1]
```

</div>

</div>

---

## Fiber: the Effect primitive empowering Structured Concurrency

> A Fiber is a convenient primitive in the context of Structured Concurrency

<br>

- Lightweight: similar a virtual/green thread, minimal overhead and low handling cost.
- Non-blocking: cooperative multitasking to maximize concurrency.
- Scoped: belongs to a structured scope, ensuring proper lifecycle management.
- Stateful: can be started, suspended, resumed, or interrupted.


<div class="text-center pt-5">
```mermaid {scale: 1}
graph LR
Runtime[Effect Runtime] --> |Orchestrates| F[Fibers]
Runtime[Effect Runtime] --> |Manages| S[Scopes]
F --> |Execute| E[Effects]
F --> |Linked to| S[Scopes]
F --> |Register relationships between Parent/Children| F
```
</div>

---

## Thanks for listening!


<div class="pb-4">
    <ul class="leading-8 mt-8 flex flex-col">
    <li><b>Effect Fibers</b>: <b color="cyan">https://effect.website/docs/concurrency/fibers/</b></li>
    <li><b>ZIO Structured Concurrency</b>: <b color="cyan">https://zio.dev/reference/fiber/#structured-concurrency</b></li>
    <li><b>Kotlin Structured Concurrency</b>: <b color="cyan">https://kotlinlang.org/docs/coroutines-basics.html#structured-concurrency</b></li>
    </ul>
  </div>



<div class="flex justify-center mt-7">
  <img src="/discord-qr.png" class="w-40 margin-0-auto" />
  <img src="/effect.png" class="ml-15 rounded-full w-40 margin-0-auto mt-4" />
</div>



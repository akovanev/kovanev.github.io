---
layout: post
title: When Async Code Becomes Sync
category: blogs 
tag: Asynchronous Programming and Multithreading
---

Hopefully you know that a method marked as `async` does not need to be executed *asynchronously*. 

First, I'm going to write some code that works as expected. 

<pre><code class="language-cs">static async Task Main(string[] args)
{
    Console.WriteLine("Starts Main. Worker threads count = " +
                        ThreadPoolThreadCounter.WorkerThreads +
                        ". Managed thread Id " +
                        Thread.CurrentThread.ManagedThreadId);

    Task simulateTask = SimulateAsync();

    Console.WriteLine("Continues Main. Worker threads count = " +
                        ThreadPoolThreadCounter.WorkerThreads +
                        ". Managed thread Id " +
                        Thread.CurrentThread.ManagedThreadId);

    await simulateTask;

    Console.WriteLine("Ends Main. Worker threads count = " +
                        ThreadPoolThreadCounter.WorkerThreads +
                        ". Managed thread Id " +
                        Thread.CurrentThread.ManagedThreadId);
}

static async Task SimulateAsync()
{
    Console.WriteLine("Starts SimulateAsync. Worker threads count = " +
                        ThreadPoolThreadCounter.WorkerThreads +
                        ". Managed thread Id " +
                        Thread.CurrentThread.ManagedThreadId);

    await Task.Delay(1);

    Console.WriteLine("Ends SimulateAsync. Worker threads count = " +
                        ThreadPoolThreadCounter.WorkerThreads +
                        ". Managed thread Id " +
                        Thread.CurrentThread.ManagedThreadId);
}</code></pre>

Here is the output.
<pre><code class="nohighlight">Starts Main. Worker threads count = 0. Managed thread Id 1
Starts SimulateAsync. Worker threads count = 0. Managed thread Id 1
Continues Main. Worker threads count = 0. Managed thread Id 1
Ends SimulateAsync. Worker threads count = 1. Managed thread Id 4
Ends Main. Worker threads count = 1. Managed thread Id 4</code></pre>

Although the sequence of steps is well described in <a href="https://docs.microsoft.com/en-US/dotnet/csharp/programming-guide/concepts/async/task-asynchronous-programming-model#BKMK_WhatHappensUnderstandinganAsyncMethod">What happens in an async method
</a>, I'll put several comments here.

1. The main *Thread* with Id 1 gets `Task simulateTask = SimulateAsync()` and steps into that method.
2. The *Thread 1* continues until `await Task.Delay(1)`. At this point, the *Thread 1* could wait, but instead it wants to do something else. Because of that:
3. The *Thread 1* continues on `Main` until it meets `await simulateTask`. Since the task has not yet completed, but `await` indicates to wait, the *Thread 1* has nothing to do right now. If you have a console application, the *Thread 1* will be suspended. If this is a Web and the *Thread 1* is just handling a request, it will return to the `ThreadPool`.
4. Once `await Task.Delay` is completed, the execution engine receives the notification. The first free thread (in our case *Thread 4*) will now continue to perform `SimulateAsync`.
5. Once the *Thread 4* finished its work in `SimulateAsync`, it returns control back to the line in `Main` where the *Tread 1* should have suspended.


Now I’ll show some magic. I am going to change only one character in my code, so the run will be ending *synchronously*. 

In particular, I will set the value of delay to zero.
 <pre><code class="language-cs">//await Task.Delay(1);
await Task.Delay(0);</code></pre>

Lets then take a look at the output.
<pre><code class="nohighlight">Starts Main. Worker threads count = 0. Managed thread Id 1
Starts SimulateAsync. Worker threads count = 0. Managed thread Id 1
Ends SimulateAsync. Worker threads count = 0. Managed thread Id 1
Continues Main. Worker threads count = 0. Managed thread Id 1
Ends Main. Worker threads count = 0. Managed thread Id 1</code></pre>

As seen, the output represents the message order different from the first run. This order exactly corresponds to a *synchronous* method. Apart of that, the *Thread 1* was the only one involved into the execution. 

To undserstand what has just happened, it might be helpful to analyze

> 2 The *Thread 1* continues until `await Task.Delay(1)`.

One more time. As you remember, the *Thread 1* stepped into `SimulateAsync` and continued *until it could*. 

After I set the delay value to zero, the *Thread 1* was able to pass `await Task.Delay(0)`, as it didn't have to wait. In the same way as it passed the command on the previous step. So the *Thread 1* continued on `SimulateAsync`, completed it and got back to the `Main`, to the next line after `Task simulateTask = SimulateAsync()`. 

Doesn't it look great?

The `ThreadPoolThreadCounter` implementation details can be found in <a href="/blogs/2020/07/29/Counter-of-working-threads">Counter of Working Threads.</a>
---
layout: post
title: When Async Code Becomes Sync 
---

Hopefully you know that a method marked as <code>async</code> does not need to be executed *asynchronously*. 

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

1. The main *Thread* with Id 1 gets <code>Task simulateTask = SimulateAsync()</code> and steps into that method.
2. The *Thread 1* continues until <code>await Task.Delay(1)</code>. At this point, the *Thread 1* could wait, but instead it wants to do something else. Because of that:
3. The *Thread 1* continues on <code>Main</code> until it meets <code>await simulateTask</code>. Since the task has not yet completed, but <code>await</code> indicates to wait, the *Thread 1* has nothing to do right now. If you have a console application, the *Thread 1* will be suspended. If this is a Web and the *Thread 1* is just handling a request, it will return to the <code>ThreadPool</code>.
4. Once <code>await Task.Delay</code> is completed, the execution engine receives the notification. The first free thread (in our case *Thread 4*) will now continue to perform <code>SimulateAsync</code>.
5. Once the *Thread 4* finished its work in <code>SimulateAsync</code>, it returns control back to the line in <code>Main</code> where the *Tread 1* should have suspended.


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

> 2 The *Thread 1* continues until <code>await Task.Delay(1)</code>.

One more time. As you remember, the *Thread 1* stepped into <code>SimulateAsync</code> and continued *until it could*. 

After I set the delay value to zero, the *Thread 1* was able to pass <code>await Task.Delay(0)</code>, as it didn't have to wait. In the same way it passed the <code>Console.WriteLine</code> command on the previous step. So the *Thread 1* continued on <code>SimulateAsync</code>, completed it and got back to the <code>Main</code>, to the next line after <code>Task simulateTask = SimulateAsync()</code>. 

Doesn't it look great?

The <code>ThreadPoolThreadCounter</code> implementation details can be found in <a href="/2020/07/29/Counter-of-working-threads">Counter of Working Threads.</a>
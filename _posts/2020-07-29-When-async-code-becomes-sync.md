---
layout: post
title: When async code becomes sync 
---

As is known, a method marked as <code>async</code> does not have to be executed asynchronously. 

Nevertheless, the first example represents the typical *asynchronous* run that well described in <a href="https://docs.microsoft.com/cs-cz/dotnet/csharp/programming-guide/concepts/async/task-asynchronous-programming-model#what-happens-in-an-async-method">What happens in the asynchronous method.
</a>

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

When the main thread (with Id 1) reached <code>Task simulateTask = SimulateAsync();</code> it stepped into that method and ran until <code>await Task.Delay(1);</code>. As it was not possible to complete the command immediately, the thread was returned to the <code>Main</code> and continued until <code>await simulateTask;</code>. Having found that the task is not completed yet, the thread with Id 1 was returned to the <code>ThreadPool</code>.
Once <code>await Task.Delay</code> was finished, the execution engine was notified of that. It assigned a free background thread (with Id 4) to continue on <code>SimulateAsync</code>. That also explains why the worker (or background) threads count was increased by one.

 To show when the <code>async</code> code may run synchronously the example above will be modified so the delay is set to zero.
 <pre><code class="language-cs">//await Task.Delay(1);
await Task.Delay(0);</code></pre>

The output.
<pre><code class="nohighlight">Starts Main. Worker threads count = 0. Managed thread Id 1
Starts SimulateAsync. Worker threads count = 0. Managed thread Id 1
Ends SimulateAsync. Worker threads count = 0. Managed thread Id 1
Continues Main. Worker threads count = 0. Managed thread Id 1
Ends Main. Worker threads count = 0. Managed thread Id 1</code></pre>

The output shows the different message order which exactly corresponds to a *synchronous* run. The magic is that while a new run it appeared that <code>await Task.Delay(0);</code> may be completed immediately. The main thread proceeded to the next command, then returned from the method and then, having reached <code>await simulateTask;</code>, went on as the task was already completed. The worker threads count value during the run was constantly equal to zero, only thread with Id 1 was involved and no extra threads were taken from the <code>ThreadPool</code>.  

 The <code>ThreadPoolThreadCounter</code> implementation details can be found in <a href="/2020/07/29/Counter-of-working-threads">Counter of working threads.</a>
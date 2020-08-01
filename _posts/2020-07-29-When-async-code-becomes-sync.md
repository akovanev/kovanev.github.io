---
layout: post
title: When async code becomes sync 
---

As is known, a method marked as <code>async</code> does not have to be executed asynchronously. 

Nevertheless, the first example represents the typical order of steps for an asynchronous run that well described in <a href="https://docs.microsoft.com/cs-cz/dotnet/csharp/programming-guide/concepts/async/task-asynchronous-programming-model#what-happens-in-an-async-method">What happens in the asynchronous method.
</a>

<pre><code class="language-cs">static async Task Main(string[] args)
{
    Console.WriteLine("Starts Main. Worker threads count = " +
                      ThreadPoolThreadCounter.WorkerThreads);

    Task simulateTask = SimulateAsync();

    Console.WriteLine("Continues Main. Worker threads count = " + 
                      ThreadPoolThreadCounter.WorkerThreads);
    
    await simulateTask;

    Console.WriteLine("Ends Main. Worker threads count = " + 
                      ThreadPoolThreadCounter.WorkerThreads);
}

static async Task SimulateAsync()
{
    Console.WriteLine("Starts SimulateAsync. Worker threads count = " + 
                      ThreadPoolThreadCounter.WorkerThreads);

    await Task.Delay(1);

    Console.WriteLine("Ends SimulateAsync. Worker threads count = " +
        ThreadPoolThreadCounter.WorkerThreads);
}</code></pre>

Here is the output.
<pre><code class="nohighlight">Starts Main. Worker threads count = 0
Starts SimulateAsync. Worker threads count = 0
Continues Main. Worker threads count = 0
Ends SimulateAsync. Worker threads count = 1
Ends Main. Worker threads count = 1</code></pre>

 The thing is, after the <code>await</code> operation in <code>SimulateAsync</code> is completed, a background thread from the <code>TreadPool</code> was assigned to the task. That is why the worker thread count value returned 1.

 To show when the <code>async</code> code may run synchronously the example above should be modified so the delay is set to zero.
 <pre><code class="language-cs">//await Task.Delay(1);
await Task.Delay(0);</code></pre>

The new output shows the different message order which exactly corresponds to a synchronous run.
<pre><code class="nohighlight">Starts Main. Worker threads count = 0
Starts SimulateAsync. Worker threads count = 0
Ends SimulateAsync. Worker threads count = 0
Continues Main. Worker threads count = 0
Ends Main. Worker threads count = 0</code></pre>
 
Besides, the worker threads count value was constantly equal to zero, therefore no extra threads were taken from the <code>ThreadPool</code>.  

 The <code>ThreadPoolThreadCounter</code> implementation details can be found in <a href="/2020/07/29/Counter-of-working-threads">Counter of working threads.</a>
---
layout: post
title: When async code becomes sync 
---

<p>It is known that the method marked as <code>async</code> does not have to be executed asynchronously. That, in turn, means that not every time extra thread pool threads needed.</p>

<p>I'll create a small console application using <code>ThreadPoolThreadCounter</code> from my previous article <a href="/2020/07/29/Counter-of-working-threads">Counter of working threads</a> to show the amount of threads needed for specific runs.</p>

<pre><code class="C#">static async Task Main(string[] args)
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
        ThreadPoolThreadCounter.WorkerThreads); ;
}</code></pre>

<p>Here is the output</p>
<pre><code class="nohighlight">Starts Main. Worker threads count = 0
Starts SimulateAsync. Worker threads count = 0
Continues Main. Worker threads count = 0
Ends SimulateAsync. Worker threads count = 1
Ends Main. Worker threads count = 1</code></pre>

<p>After getting <code>await Task.Delay(1)</code> the CLR gaves control back to <code>Main</code> to continue asynchronously, which is fine. </p>
<p>Lets now reset the delay value to zero and run again.</p>
<pre><code class="C#">//await Task.Delay(1);
await Task.Delay(0);</code></pre>

<p>Here is the new output</p>
<pre><code class="nohighlight">Starts Main. Worker threads count = 0
Starts SimulateAsync. Worker threads count = 0
Ends SimulateAsync. Worker threads count = 0
Continues Main. Worker threads count = 0
Ends Main. Worker threads count = 0</code></pre>

<p>Not only the thread count value changed but also the message order. The CLR just passed through zero delay to the next line and only after <code>SimulateAsync</code> ended returned control to <code>Main</code> with the completed task.</p>



---
layout: post
title: Max degree of concurrency 
---

In the previous article <a href="/2020/07/30/Exception-handling-for-tasks-running-in-parallel">Exception handling for tasks running in parallel</a> it was suggested how to safely work with multiple tasks running in parallel. In some situations it is additionally required to limit a number of tasks that may run simultaneously. In other words, the maximum degree of concurrency should be set for a collection of tasks. 

The example below instroduces the way how to solve the issue using the <code>SemaphoreSlim</code> class. The <code>DoSimultaneousStuffAsync</code> signature is aligned with <code>DoStuffAsync</code> that expects just one integer parameter. In real life, that parameter is more likely will be generic and/or represent a custom type.

<pre><code class="language-cs">static async Task Main(string[] args)
{
    int maxDegreeOfConcurrency = 4; 
    using var throttler = new SemaphoreSlim(maxDegreeOfConcurrency);

    var doStuffTasks = Enumerable.Range(1, 20)
        .Select(x => DoSimultaneousStuffAsync(x, DoStuffAsync, throttler))
        .ToList();

    //... 
}

static async Task&lt;int&gt; DoSimultaneousStuffAsync(
    int value,
    Func&lt;int, Task&lt;int&gt;&gt; doStuffAsync,
    SemaphoreSlim throttler)
{
    await throttler.WaitAsync();
    try
    {
        Console.Write("+"); //For testing
        return await doStuffAsync(value);
    }
    finally
    {
        Console.Write("-"); //For testing
        throttler.Release();
    }
}</code></pre>

The output below represents the tasks.  
<pre><code class="nohighlight">++++-+--++-+-+-+---+++-+--+-++--++-+----</code></pre>
The plus means that the task has started, minus that it is completed. The longest sequence of pluses going in a row is 4, which is equal to the <code>maxDegreeOfConcurrency</code> value. So the solution works as expected. 
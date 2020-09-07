---
layout: post
title: Max Degree of Concurrency 
category: blogs
tag: Asynchronous Programming and Multithreading
---

In the previous article <a href="/blogs/2020/07/30/Exception-handling-for-tasks-running-in-parallel">Exception Handling for Tasks Running in Parallel</a> it was suggested how to safely work with multiple tasks running in parallel. In some situations it is additionally required to limit a number of tasks that may run simultaneously. In other words, the maximum degree of concurrency should be set for a collection of tasks. 

The example below instroduces the way how to solve the issue using the `SemaphoreSlim` class. The `DoSimultaneousStuffAsync` signature is aligned with `DoStuffAsync` that expects just one integer parameter. In real life, that parameter is more likely will be generic and/or represent a custom type.

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
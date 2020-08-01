---
layout: post
title: Max degree of concurrency 
---

In the previous article <a href="/2020/07/30/Exception-handling-for-tasks-running-in-parallel">Exception handling for tasks running in parallel</a> a general approach on how to safely work with multiple tasks running in parallel was suggested. Nevertheless, there are some situations when a number of tasks/calls/requests running simultaneously should be additionally restricted.

A simple way to solve the issue is add the <code>SemaphoreSlim</code> class object.

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
    Func&lt;int, Task&lt;int&gt;&gt; doStuff,
    SemaphoreSlim throttler)
{
    await throttler.WaitAsync();
    try
    {
        Console.Write("+"); //For testing
        return await doStuff(value);
    }
    finally
    {
        Console.Write("-"); //For testing
        throttler.Release();
    }
}</code></pre>

The output represents the tasks running simultaneously. Apparently the maximum sequence of pluses going in a row is not greater than 4, which is precisely equal to the <code>maxDegreeOfConcurrency</code> value. 
<pre><code class="nohighlight">++++-+--++-+-+-+---+++-+--+-++--++-+----</code></pre>
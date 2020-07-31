---
layout: post
title: Max degree of concurrency 
---

<p>Sometimes while calling a backend service there could be a strong restriction how many requests it can handle simultaneously. I'll modify the example from <a href="/2020/07/30/Exception-handling-for-tasks-running-in-parallel">Exception handling for tasks running in parallel</a> to show where the changes should come.</p>

<pre><code class="language-cs">static async Task Main(string[] args)
{
    <b>int maxDegreeOfConcurrency = 4; //Lets say it will be 4</b>
    <b>using var throttler = new SemaphoreSlim(maxDegreeOfConcurrency);</b>

    var doStuffTasks = Enumerable.Range(1, <b>20</b>)
        .Select(<b>x => DoSimultaneousAsync(x, DoStuffAsync, throttler)</b>)
        .ToList();

    var aggregateTask = Task.WhenAll(doStuffTasks);
    try
    {
        //ConfigureAwait can be ignored in .net core
        await aggregateTask.ConfigureAwait(false);
    }
    catch (Exception)
    {
        var errorMessages = aggregateTask.Exception
                                ?.InnerExceptions
                                .Select(x =&gt; x.Message)
                            ?? new List&lt;string&gt;();

        Console.WriteLine(string.Join(',', errorMessages));
    }

    //In .net core use IsCompletedSuccessfully instead of !IsFaulted
    foreach (var task in doStuffTasks.Where(t =&gt; !t.IsFaulted))
    {
        Console.WriteLine(await task);
    }
}

<b>static async Task&lt;int&gt; DoSimultaneousAsync(
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
}</b>

static async Task&lt;int&gt; DoStuffAsync(int value)
{
    await Task.Delay(1);

    if (value % 3 == 0)
        throw new ArgumentException(value.ToString(), nameof(value));

    return value;
}</code></pre>

<p>Here is the output</p>
<pre><code class="nohighlight">++++-+--++-+-+-+---+++-+--+-++--++-+----</code></pre>

<p>So there was no sequence of pluses going in a row more than the defined <code>maxDegreeOfConcurrency</code> value.</p>
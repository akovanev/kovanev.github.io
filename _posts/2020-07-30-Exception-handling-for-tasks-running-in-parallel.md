---
layout: post
title: Exception handling for tasks running in parallel 
---

<p>Many developers have a fear of missing something while handling exceptions working with tasks. I'll create a small example that could be easily transformed to your needs.</p>

<pre><code class="C#">static async Task Main(string[] args)
{
    var doStuffTasks = Enumerable.Range(1, 10)
        .Select(DoStuffAsync)
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
    
static async Task&lt;int&gt; DoStuffAsync(int value)
{
    await Task.Delay(1);

    if (value % 3 == 0)
        throw new ArgumentException(value.ToString(), nameof(value));

    return value;
}</code></pre>

<p>Here is the output</p>
<pre><code class="nohighlight">3 (Parameter 'value'),6 (Parameter 'value'),9 (Parameter 'value')
1
2
4
5
7
8
10</code></pre>

<p>So step by step may look like</p>
1. Create a collection of tasks 
2. Create an aggregate task for that collection using <code>Task.WhenAll</code>.
3. <code>await</code> the aggregate task inside <code>try-catch</code>. <br>
I would strongly recommend to read additionally Stephen Cleary <a href="https://blog.stephencleary.com/2016/12/eliding-async-await.html">article</a> and its exceptions section.
4. Handle the aggregate task inner exceptions within <code>catch</code> block.
5. Iterate on completed tasks. 
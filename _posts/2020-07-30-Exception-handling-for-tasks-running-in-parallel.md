---
layout: post
title: Exception handling for tasks running in parallel 
---

Many developers have a fear of missing something while handling exceptions that can arise when multiple tasks are executed in parallel.

The example below introduces a step-by-step approach how to safely work with a task collection. 

<pre><code class="language-cs">static async Task Main(string[] args)
{
    //1. Creates a collection of tasks
    var doStuffTasks = Enumerable.Range(1, 10)
        .Select(DoStuffAsync)
        .ToList();

    //2. Creates an aggregate task for all
    var aggregateTask = Task.WhenAll(doStuffTasks);
    try
    {
        //3. Awaits the aggregate task inside the try-catch block
        await aggregateTask.ConfigureAwait(false);
    }
    catch (Exception)
    {
        //4. Handles exceptions coming from tasks
        var errorMessages = aggregateTask.Exception
                                ?.InnerExceptions
                                .Select(x =&gt; x.Message)
                            ?? new List&lt;string&gt;();

        Console.WriteLine(string.Join(',', errorMessages));
    }

    //5. Iterates on completed tasks
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

<p>Here is the output.</p>
<pre><code class="nohighlight">3 (Parameter 'value'),6 (Parameter 'value'),9 (Parameter 'value')
1
2
4
5
7
8
10</code></pre>

As a note, I would strongly recommend to read  the Stephen Cleary <a href="https://blog.stephencleary.com/2016/12/eliding-async-await.html">article</a>, in particular, the exceptions section.
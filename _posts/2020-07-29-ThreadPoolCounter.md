---
layout: post
title: Counter of working threads 
---

When wanting to get the count of existing thread pool threads you may start from looking at the framework <code>ThreadPool</code> property.

<pre><code class="C#">/// &lt;summary&gt;
/// Gets the number of thread pool threads that currently exist.
/// &lt;/summary&gt;
/// &lt;remarks&gt;
/// For a thread pool implementation that may have different types of threads, the count includes all types.
/// &lt;/remarks&gt;
public static extern int ThreadCount
{
    [MethodImpl(MethodImplOptions.InternalCall)]
    get;
}
</code></pre>

<p>But the return value sometimes is greater than you may expect. I have an idea why is that so but now I just want to show my class that I am going to use in the future articles.</p>

<pre><code class="C#">public class ThreadPoolCounter
{
    private static readonly int MaxWorkerThreads;

    static ThreadPoolCounter()
    {
        ThreadPool.GetMaxThreads(out MaxWorkerThreads, out _);
    }

    public static int WorkerThreads
    {
        get
        {
            ThreadPool.GetAvailableThreads(out int workerThreads, out _);
            return MaxWorkerThreads - workerThreads;
        }
    }
}</code></pre>

So the idea of <code>WorkerThreads</code> property is return only those working threads that are currently not available for other tasks.

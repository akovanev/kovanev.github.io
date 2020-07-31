---
layout: post
title: Counter of working threads 
---

<p>When wanting to get the count of the thread pool active threads it looks natural to use the <code>ThreadCount</code> property of the <code>ThreadPool</code>.</p>

<pre><code class="language-cs">/// &lt;summary&gt;
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

<p>However the return value may be greater than expected. To be honest, I didn't find any information regarding these magic types from "the count includes all types". But as a workaround I'm going to implement a sort of helper class that is focused on only available worker threads.</p>

<pre><code class="language-cs">public class ThreadPoolThreadCounter
{
    private static readonly int MaxWorkerThreads;

    static ThreadPoolThreadCounter()
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
---
layout: post
title: Counter of Working Threads 
---

When you need to get the number of active threads, it seems natural to start with the <code>ThreadCount</code> property of the <code>ThreadPool</code>.

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

However, its current value may be greater than expected. The comment tells us about different types of threads. I couldn't find any further information on this. Fortunately, the other <code>ThreadPool</code> methods can be used as a workaround to count only the available worker threads.

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
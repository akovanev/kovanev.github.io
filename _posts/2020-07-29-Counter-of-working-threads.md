---
layout: post
title: Counter of working threads 
---

When wanting to get the count of the thread pool active threads it looks natural to use the <code>ThreadCount</code> property of the <code>ThreadPool</code>.

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

However, the return value may be greater than expected. Unfortunately, "the count includes all types" is not described well in the documentation. But as a workaround, the other <code>ThreadPool</code> methods could be used to count only available worker threads.

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
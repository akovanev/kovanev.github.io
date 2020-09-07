---
layout: post
title: Counter of Working Threads 
category: blogs
tag: Asynchronous Programming and Multithreading
---

When you need to get the number of active threads, it seems natural to start with the <code>ThreadPool.ThreadCount</code> property.
 
 According to the description, the <a href="https://docs.microsoft.com/en-us/dotnet/api/system.threading.threadpool.threadcount">ThreadPool.ThreadCount</a>
 > Gets the number of thread pool threads that currently exist.

However, that doesn't tell us anything about how many of them are active.

Looking further, you can find that the <a href="https://docs.microsoft.com/en-us/dotnet/api/system.threading.threadpool.getavailablethreads">ThreadPoolGetAvailableThreads</a>

>Retrieves the difference between the maximum number of thread pool threads returned by the GetMaxThreads(Int32, Int32) method, and the number **currently active**.

This is exactly what we need.

Now I'm going to create a simple helper class with a static property that counts the number of active worker threads.

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
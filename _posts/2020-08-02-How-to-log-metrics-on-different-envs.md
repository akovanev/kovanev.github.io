---
layout: post
title: How to log metrics on different environments
---

If an application has long-running tasks it is pretty common that there can be performance and/or memory issues. At first glance, it looks easy to add metrics as the .net framework contains a couple of very simple classes like <code>Stopwatch</code> and <code>GC</code>. The problems appeared when metrics should be logged in different ways depending on an object type, environment and so on.

The idea of this article is combine several known approaches to write the code that can be reused for work with different metrics.

<pre><code class="language-cs">public class MetricsWatcher : IDisposable
{
    private readonly Type _caller;
    private readonly string _method;
    private readonly MetricsWatcherOptions _options;
    private readonly Stopwatch _stopwatch;

    public MetricsWatcher(Type caller,
        MetricsWatcherOptions options = null,
        [CallerMemberName] string method = "")
    {
        _caller = caller;
        _method = method;
        _options = options ?? new MetricsWatcherOptions();
        _stopwatch = Stopwatch.StartNew();
    }

    public event EventHandler&lt;MetricsWatcherArgs&gt; OnCaptured;

    public void Capture(
        string message = "",
        [CallerMemberName] string method = "",
        [CallerLineNumber] int line = default)
    {
        if (OnCaptured == null) return;
       
        var args = new MetricsWatcherArgs(
            _caller.Name,
            method,
            line,
            message,
            _stopwatch.Elapsed,
            _options.WatchMemory ? GC.GetTotalMemory(false) : (long?)null);
       
        OnCaptured(_caller, args);
    }

    public void Dispose()
    {
        _stopwatch.Stop();
        Capture($"{nameof(MetricsWatcher)} disposing", _method, 0);
    }
}</code></pre>

The <code>MetricsWatcher</code> class is basically decorating the <code>Stopwatch</code> with the option to measure the memory which is currently in use. The class does not know anything about the way how its data will be processed, formatted, logged. The main repsonisibility is raise an event when  the state is considered to be captured.

Using the <code>EventHandler</code> instead of a custom delegate is not mandatory but preferable for work with the <a href="https://github.com/dotnet/reactive">Reactive Extensions</a> that, in turn, provide more flexibility on the event-based approach.

The missing implementation of <code>MetricsWatcherArgs</code>, which is used as the <code>EventHandler</code> type parameter, and <code>MetricsWatcherOptions</code> that shares just one option so far, is very straightforward.
<pre><code class="language-cs">public class MetricsWatcherArgs : EventArgs
{
    public MetricsWatcherArgs(
        string className,
        string method,
        int line,
        string message,
        TimeSpan timeSpan,
        long? totalMemory = null)
    {
        ClassName = className;
        Method = method;
        Line = line;
        Message = message;
        TimeSpan = timeSpan;
        TotalMemory = totalMemory;
    }

    public string ClassName { get; }
    public string Method { get; }
    public int Line { get; }
    public string Message { get; }
    public TimeSpan TimeSpan { get; }
    public long? TotalMemory { get; }
}

public class MetricsWatcherOptions
{
    public bool WatchMemory { get; set; }
}</code></pre>


Once the part above is understandable, the next step is introduce a factory that will be responsible for creating a watcher with the determined way of how to process the metrics. 

The *if-else* logic for setting loggers up can be added here. Bear in mind also that the code in the watcher <code>Capture</code> method will run only if there is at least one handler set. In other words, for unwanted calculations on a production environment the handler should be just not set.

<pre><code class="language-cs">public class MetricsWatcherFactory
{
    public MetricsWatcher Create(
        Type caller, 
        MetricsWatcherOptions options,
        [CallerMemberName] string method = "")
    {
        var mw = new MetricsWatcher(typeof(Program), options, method);
        var sequence = Observable.FromEventPattern&lt;MetricsWatcherArgs&gt;(
            handler => mw.OnCaptured += handler,
            handler => mw.OnCaptured -= handler);
        sequence.Subscribe(
            data => Console.WriteLine(GetMetricsString(data.EventArgs)),
            ex => Console.WriteLine(ex.Message));
        return mw;
    }

    internal string GetMetricsString(MetricsWatcherArgs mwArgs)
    {
        return $"{mwArgs.ClassName}.{mwArgs.Method}():{mwArgs.Line}" +
               $" - {mwArgs.Message} " +
               $"{mwArgs.TimeSpan.Milliseconds} ms " +
               $"{mwArgs.TotalMemory ?? 0} bytes";
    }
}</code></pre>

Finally here is a bunch of code to test all together.
<pre><code class="language-cs">static async Task Main(string[] args)
{
    await Run();
    Console.WriteLine("The End.");
}

static async Task Run()
{
    var options = new MetricsWatcherOptions { WatchMemory = true };
    var mwf = new MetricsWatcherFactory();

    using var mw = mwf.Create(typeof(Program), options);

    var array = new byte[100000];
    await Task.Delay(TimeSpan.FromSeconds(2));

    mw.Capture("Memory was allocated.");

    var array2 = new byte[100000];
    await Task.Delay(TimeSpan.FromSeconds(2));
}
</code></pre>

The output.
<pre><code class="nohighlight">Program.Run():32 - Memory was allocated. 139 ms 459640 bytes
Program.Run():0 - MetricsWatcher disposing 182 ms 567888 bytes
The End.</code></pre>

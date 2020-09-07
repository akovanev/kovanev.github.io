---
layout: post
title: How to Log Metrics on Different Environments
category: blogs
tag: Logging
---

If an application has long-running tasks then it rarely goes without obtaining some measurements. If it is not a local environment then the good question is whether the execution of the measurement code itself may affect performance or cause some other issues. 

After making a decision what should be measured, another question - how to log the data. In general, the logging strategy may vary depending not only on an environment itself but also taking into account an object type, a running method and so on.

For such tasks, it is common to create an *auxiliary class* encapsulating the logic of work with a standard framework feature. Something like the <code>StopwatchHelper</code>. While a logger is just being injected in that *auxiliary*.

There are a couple of problems this approach has. The first is how to control if the *auxiliary* method is allowed to be executed on the specific environment. The second, if the answer on the previos question was yes - should it log the data in the same way as on other environments or differently. 

One of the main questions to decide is where to put the *if-else* logic so the code would be flexible and easy to maintain. 

In the example below the *auxiliary* does not log anything. Besides, the *if-else* logic is also moved out. 
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

Apart of managing the <code>Stopwatch</code>, the <code>MetricsWatcher</code> has an option to measure the memory which is currently in use. The <code>MetricsWatcher</code> does not know anything about the way how its data will be processed, formatted, logged. The important thing it should do is raise an event when the state is considered to be captured.

Using the <code>EventHandler</code> instead of a custom delegate is not mandatory but preferable for work with the <a href="https://github.com/dotnet/reactive">Reactive Extensions</a>. That, in turn, provides more flexibility on the event-based approach.

The missing implementation of the two other classes <code>MetricsWatcherArgs</code> and <code>MetricsWatcherOptions</code> is very straightforward.
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

The *if-else* logic for setting loggers up can be added here. Bear in mind also that the code in the watcher <code>Capture</code> method will execute only if there is at least one handler set. In other words, for unwanted calculations on a production environment the handler should be just not set.

<pre><code class="language-cs">public class MetricsWatcherFactory
{
    public MetricsWatcher Create(
        Type caller, 
        MetricsWatcherOptions options,
        [CallerMemberName] string method = "")
    {
        var mw = new MetricsWatcher(caller, options, method);
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

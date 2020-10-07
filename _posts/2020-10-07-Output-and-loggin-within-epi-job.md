---
layout: post
title: Output and Logging within EPiserver Jobs
category: blogs
tag: EPiServer
---


Working with `EPiServer` on various projects for more than five years, I often have tasks to import or update content, users, prices etc. Using jobs in such tasks is a very common practice. 

Every time I've seen a list of jobs and different efforts to extract the basic functionality at some abstract level. Every time the output, logging, exception handling strategies were not ideal.

If you look at <a href="https://github.com/akovanev/EPiServer.Jobs.Extensions/">EPiServer.Jobs.Extensions</a> package you will find very simple logic, which despite this may save project hours. 

Let's suppose I have an import task. I included the job extensions in my project. Then the code may look like
<pre><code class="language-cs">
[ScheduledPlugIn(
    DisplayName = "Import job",
    Description = "Imports some stuff",
    SortIndex = 100000
)]
public class ImportJob : JobBase&lt;ImportService&gt;
{
    public ImportJob()
        : base(
            ServiceLocator.Current.GetInstance&lt;ImportService&gt;(),
            LogManager.Instance.GetLogger(nameof(ImportJob)))
    {
    }

    protected override Dictionary&lt;string, object&gt; GetExecutionParameters()
    {
        return new Dictionary&lt;string, object&gt; 
		{ {ImportService.StartDate, DateTime.Today} };
    }
}

public class ImportService : JobServiceBase
{
    public const string StartDate = "StartDate";

	//10 specifies the progress step
    public ImportService() : base(new ProgressManager(10)) {}

    public override string Execute(Dictionary&lt;string, object&gt; parameters)
    {
        LogMessage(parameters[StartDate].ToString());

        LogMessage("Import starts");

		//Here should be some logic retrieving data from middleware
        var content = Enumerable.Range(1, 100)
            .Select(x => new {name = $"name{x}", code = x})
            .ToList();

        var errorList = new List&lt;string&gt;();
        int progressId = ProgressManager.CreateNew(content.Count);
        foreach (var item in content)
        {
            try
            {
				//Here should be some import stuff. 
				//For clarity I added exceptions 
                if(item.code % 3 == 0) 
					throw new Exception(item.code.ToString());
            }
            catch (Exception)
            {
                errorList.Add(item.code.ToString());
            }

            ProgressManager.Increment(progressId);
        }
        ProgressManager.Complete(progressId);

        LogMessage("Import completed");

        return errorList.Any()
            ? $"Error codes: {string.Join(",", errorList)}"
            : "OK";
    }
}
</code></pre>

While the job is running the output and log will look like

<pre><code class="nohighlight">
2020-10-07 17:13:09,890 [81] INFO ImportJob: 10/7/2020 12:00:00 AM
2020-10-07 17:13:09,892 [81] INFO ImportJob: Import starts
2020-10-07 17:13:09,903 [81] INFO ImportJob: Execute: 10/100
2020-10-07 17:13:09,916 [81] INFO ImportJob: Execute: 20/100
2020-10-07 17:13:09,929 [81] INFO ImportJob: Execute: 30/100
2020-10-07 17:13:09,933 [81] INFO ImportJob: Execute: 40/100
2020-10-07 17:13:09,937 [81] INFO ImportJob: Execute: 50/100
2020-10-07 17:13:09,939 [81] INFO ImportJob: Execute: 60/100
2020-10-07 17:13:09,949 [81] INFO ImportJob: Execute: 70/100
2020-10-07 17:13:09,953 [81] INFO ImportJob: Execute: 80/100
2020-10-07 17:13:09,957 [81] INFO ImportJob: Execute: 90/100
2020-10-07 17:13:09,962 [81] INFO ImportJob: Execute: 100/100
2020-10-07 17:13:09,972 [81] INFO ImportJob: Import completed
</code></pre>

Here is the job result <img src="/public/imj.png">

---
layout: post
title: Integration Testing with Data Generator
category: blogs 
tag: Data Generator 
---

In the previous post about <a href="/blogs/2020/09/07/Data-generator-attributes">Data Generator Attributes</a> I described how to generate the random data based on the attributes approach. I also gave an example for the <a href="https://github.com/akovanev/DataGenerator/blob/master/Akov.DataGenerator.Demo/StudentsSampleTests/Tests/DgModels/DgStudentCollection.cs">DgStudentCollection</a> class.

Let's suppose that we have a service working with students.

<pre><code class="language-cs">interface IStudentService
{
    Task&lt;IEnumerable&lt;Student&gt;?&gt; GetAll();
}
</code></pre>

Let's aslo assume that the middleware is still under development. So the return data may have a lot of issues, including missing properties, the wrong format  etc. I'd like the service to be written in such a way that all the consistent data is proceeded to a higher level, while all the issues will be logged.

I expect that the response data to be in json, and all of the dto's will derive from the following class.

<pre><code class="language-cs">abstract class Result
{
    public virtual bool IsValid => ParsingErrors.Count == 0;
    public virtual bool HasWarnings => ParsingWarnings.Count > 0;

    public Dictionary&lt;string, string&gt; ParsingErrors { get; set; } = new Dictionary&lt;string, string&gt;();
    public Dictionary&lt;string, string&gt; ParsingWarnings { get; set; } = new Dictionary&lt;string, string&gt;();

    protected virtual void AddError(ErrorContext errorContext)
    {
        string errorKey = GetErrorKey(errorContext.Path);
        string errorValue = errorContext.Error.Message;
        ParsingErrors.Add(errorKey, errorValue);
    }

    protected virtual void AddWarning(ErrorContext errorContext)
    {
        string errorKey = GetErrorKey(errorContext.Path);
        string errorValue = errorContext.Error.Message;
        ParsingWarnings.Add(errorKey, errorValue);
    }

    protected static string GetErrorKey(string path)
        => path.Split(".").Last();

    protected static string GetErrorKeyProperty(string path)
    {
        string errorKey = GetErrorKey(path);
        int arrayIndex = errorKey.IndexOf("[", StringComparison.Ordinal);
        return arrayIndex > 0
            ? errorKey.Substring(0, arrayIndex)
            : errorKey;
    }
}
</code></pre>

Probably the `Result` is not perfect. It has dependencies of `JsonConvert`, which I would avoid in real life, but for my example I find it good enough. The main idea is to add the couple of objects to store errors and warnings on that level, and keep it simple. 

Here is the example of a dto class
<pre><code class="language-cs">class Student : Result
{
    //Other properties ...

    [JsonProperty("last_updated")]
    public DateTime LastUpdated { get; set; }

    public List&lt;Subject&gt;? Subjects { get; set; }

    public Subject? Subject { get; set; }

    // The object is valid only if all inner objects are parsed correctly.
    public override bool IsValid => base.IsValid &&
        (Subject is null || Subject.IsValid) &&
        (Subjects is null || Subjects.All(s => s.IsValid));

    // Handles errors while JsonConvert parsing (deserializing).
    [OnError]
    internal void OnError(StreamingContext context, ErrorContext errorContext)
    {
        //There is a requirement to consider datetime parsing errors as warning
        if ("last_updated" == GetErrorKeyProperty(errorContext.Path))
        {
            LastUpdated = DateTime.Today;
            AddWarning(errorContext);
        }
        else
        {
            AddError(errorContext);
        }

        //Allows to proceed without throwing an exception.
        errorContext.Handled = true;
    }
}
</code></pre>

Once the model and behaviour are defined let's create the service implementaion.

<pre><code class="language-cs">class StudentService : IStudentService
{
    private readonly ApiClient _apiClient;

    public StudentService(HttpClient httpClient)
    {
        _apiClient = new ApiClient(httpClient);
    }

    public async Task&lt;IEnumerable&lt;Student&gt;?&gt; GetAll()
    {
        var result = await _apiClient.GetAsync&lt;StudentCollection&gt;(
            "https://example.com/api/students");

        if (result?.Students is null) return null;

        Log(result.Students);

        //Only valid student objects are to return.
        return result.Students
            .Where(s => s.IsValid)
            .ToList();
    }

    //...
}

class ApiClient
{
    private readonly HttpClient _httpClient;
    private readonly JsonSerializerSettings _parsingSettings;

    public ApiClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
        _parsingSettings = new JsonSerializerSettings
        {
            DateFormatString = "dd/MM/yy", //expected format
            Culture = new CultureInfo("en-US"),
        };
    }

    public async Task&lt;T?&gt; GetAsync&lt;T&gt;(string url) where T : class
    {
        try
        {
            var response = await _httpClient.GetAsync(url);
            response.EnsureSuccessStatusCode();
            string responseAsString = await response.Content.ReadAsStringAsync();
            return JsonConvert.DeserializeObject&lt;T&gt;(responseAsString, _parsingSettings);
        }
        catch (Exception e)
        {
            Debug.WriteLine(e);
            return default;
        }
    }
}
</code></pre>

If everything is implemented right, then the `ApiClient` should not throw exceptions except the network issues. `JsonConvert.DeserializeObject<T>` is supposed to give control to the `[OnError]` methods for all the response objects that had parsing problems.

If this part is understandable then I will go ahead to creating a test solution. I will not put all the code here, focusing only on main aspects.

First I need to mock the responses with the generated data. For that I will create the `MockHttpClientFactory`.

<pre><code class="language-cs">class MockHttpClientFactory
{
    private readonly Lazy&lt;HttpClient&gt; _studentServiceHttpClient;

    public MockHttpClientFactory()
    {
        _studentServiceHttpClient = new Lazy&lt;HttpClient&gt;(() =>
        {
            var dg = new DG(
                new StudentGeneratorFactory(),
                new DataSchemeMapperConfig { UseCamelCase = true });

            //Creates DataScheme based on DgStudentCollection attributes.
            DataScheme scheme = dg.GetFromType&lt;DgStudentCollection&gt;();

            //Generates json random data.
            string jsonData = dg.GenerateJson(scheme);

            var handlerMock = new Mock&lt;HttpMessageHandler&gt;(MockBehavior.Strict);
            handlerMock.Protected()
                .Setup&lt;Task&lt;HttpResponseMessage&gt;&gt;(
                    "SendAsync",
                    ItExpr.IsAny&lt;HttpRequestMessage&gt;(),
                    ItExpr.IsAny&lt;CancellationToken&gt;()
                )
                .ReturnsAsync(new HttpResponseMessage
                {
                    StatusCode = HttpStatusCode.OK,
                    //Responds with the generated data.
                    Content = new StringContent(jsonData)
                })
                .Verifiable();

            return new HttpClient(handlerMock.Object);
        });
    }

    public HttpClient GetStudentServiceClient()
        => _studentServiceHttpClient.Value;
}

</code></pre>

Afterwards, I am ready to write a test.

<pre><code class="language-cs">public class StudentServiceTests
{
    private readonly StudentService _studentService;

    public StudentServiceTests()
    {
        var httpClient = new MockHttpClientFactory().GetStudentServiceClient();
        _studentService = new StudentService(httpClient);
    }

    [Fact]
    public async Task GetAll_RandomStudentList()
    {
        var students = (await _studentService.GetAll()).ToList();

        Assert.NotNull(students);

        //Expects only valid data returned.
        Assert.True(students.All(s => s.IsValid));

        //Due to the requirements the failure during casting to the LastUpdated 
        //is not considered to be an error, but considered to be a warning.
        //The fallback logic implies setting the today date in the case of failure.
        const string lastUpdatedJsonName = "last_updated";
        
        var studentsWithNotParsedDate = students
            .Where(s => s.HasWarnings && s.ParsingWarnings.ContainsKey(lastUpdatedJsonName))
            .ToList();
        
        DateTime expectedDate = DateTime.Today;
        
        Assert.True(studentsWithNotParsedDate.All(x => x.LastUpdated == expectedDate));
    }
}
</code></pre>

During testing, I have noticed that the code may not run so fast in the debug mode. Probably it could be found a better solution instead of using the `JsonConvert`. I will continue investigating this. 

Anyways, I believe that the approach I have shown here is clear. Having implemented it, I can be sure that both correct and inconsistent data will be handled in the safe way.

The entire code may be taken from the <a href="https://github.com/akovanev/DataGenerator/tree/master/Akov.DataGenerator.Demo">Akov.DataGenerator.Demo</a>.
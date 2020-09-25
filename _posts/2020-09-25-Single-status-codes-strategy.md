---
layout: post
title: Single Status Codes Strategy
category: blogs
tag: Microservices
---

While tons of information about using `REST` in the world of `microservices` is available on the web, I would like to focus on a very simple problem.

The more services you expose, the more similar actions you must perform to return a body and status code. If you don't have a single strategy, then the *if-else* logic may repeat in each controller, despite it is not necessary.

<a href="https://github.com/akovanev/Utils.ResultExtensions">Here</a> I wrote some classes to simplify the solution. I've already started using `C# 9` and `.NET 5`, but you can make some minor changes to my classes so they will work for you as well.

In the example below you can find the usage of the <a href="https://github.com/akovanev/Utils.ResultExtensions/blob/master/src/Result.cs">Result</a> and its extensions.
<pre><code class="language-cs">
[ApiController]
public class ApiControllerBase : ControllerBase
{
    protected ActionResult GetActionResult&lt;T&gt;(RestType restType, Result&lt;T&gt; result)
    {
        if (result is null) return new EmptyResult();

        int code = result.GetStatusCode(restType);
        object data = result.IsValid ? result.Data : result.Error.Message;
        return new JsonResult(data) { StatusCode = code };
    }
}

[Route("[controller]")]
public class WeatherForecastController : ApiControllerBase
{
	private readonly IForeCastService _foreCastService;

	public WeatherForecastController(IForeCastService foreCastService)
	{
		_foreCastService = foreCastService;
	}

	[HttpGet]
	[ProducesResponseType(typeof(IEnumerable&lt;WeatherForecast&gt;), 200)]
	[ProducesResponseType(typeof(string), 404)]
	public ActionResult Get()
	{
		Result&lt;IEnumerable&lt;WeatherForecast&gt;&gt; result = _foreCastService.GetForecastCollection();
		return GetActionResult(RestType.Get, result);
	}
}
</code></pre>

The `Get` method looks simple and, the thing is, that here you shouldn't take care about the response and status code at all. 

All the magic is concealed in the <a href="https://github.com/akovanev/Utils.ResultExtensions/blob/master/src/StatusCodeHelper.cs">StatusCodeHelper</a>. However, if the default implementation doesn't match your requirements, then you just have to write your own helper class.

The main idea behind getting a status code is based on several points

* The `REST` method.
* Whether the `Result<T>` object is valid, in other words, the `Error` property is `null`.
* Whether the `Data` property is `null` or not.

The list of questions can be expanded. It will still be possible to create and apply a single strategy across all your services though. 
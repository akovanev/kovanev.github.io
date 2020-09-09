---
layout: post
title: How to Sanitize Sensitive Data While Handling Http Requests
category: blogs
tag: Sanitizer
---

Let me introduce the [Akov.Sanitizer](https://github.com/akovanev/Sanitizer/), a super simple library that helps conceal sensitive information. 

## The Issue

If the data are bound/deserialized into some class instance, then the class can deal with sensitive information itself. However, in ASP.NET MVC and API projects, there is a gap between receiving the request and the moment when the data arrived at the controller. On this way the data can be validated, for example using fluent validation, and it is possible that they will not reach the controller at all. 

At the same time, it can be important for the project to log the request information including the payload, but having extracted all the sensitive part. At this point the binding type is not known yet, but we still have to deal with data. Of course it might be doable to find the certain type in each particular case, but mantaining such code can be very difficult.

From another side, having just data in some text format may not give us enough information to act. It would be preferable if we could know at least the name of the binding type. But, for instance, if the request contains a json, it is quite common if the name of the entry, in our case binding type, is missing. 

That, in turn, leads us to some other issues. Let's assume that we can implement a solution based on property names only. That might make perfect sense, as we have to extract the sensitive data from the properties but not from the objects. But what if we have more than one property with the same name in our solution? How to find the right one to apply its replacement pattern, if it exists?

That is challenging. The solution can be found if we agree that the names for the *sensitive* properties should inherit the same pattern for all request models.

In other words, if there is the property `Number` within the `Card` and it is sensitive, then the same named property, but within the `Room`, should be sensitive as well. If it doesn't make sense for the `Room`, then the easiest solution is to rename the first property, for instance, to the `CardNumber`.

Despite this approach imposes restrictions, it is still helpful in the context of the problem.

### The solution

If the previous part was accepted, than let's go ahead to the solution.

In the example below the card object is considered to be sanitized. The nuget package page is available [here](https://www.nuget.org/packages/Akov.Sanitizer/).
<pre><code class="language-cs">[Sanitized]
public class Card
{
    [JsonProperty("cardNumber")] 
    [ReplaceFor(typeof(PartialSanitizer), 
        propertyName: "cardNumber", 
        pattern: "4,4,*")]
    public string? Number { get; set; }

    [ReplaceFor(typeof(AsteriskSanitizer), pattern: "6,@")]
    public string? CardholderName { get; set; }

    public int? Year { get; set; }
    public int? Month { get; set; }

    [ReplaceFor(typeof(AsteriskSanitizer))]
    public string? Cvc { get; set; }

    public Address? Address { get; set; }
}
</code></pre>

Each type, which is supposed to be sanitized, should be marked with the `[Sanitized]` attribute. The `[ReplaceFor]` shows that a property requires sanitizing done by the specific sanitizer class. 

There are two supported out of the box: 

* [AsteriskSanitizer](https://github.com/akovanev/Sanitizer/blob/master/Akov.Sanitizer/Sanitizers/AsteriskSanitizer.cs)

* [PartialSanitizer](https://github.com/akovanev/Sanitizer/blob/master/Akov.Sanitizer/Sanitizers/PartialSanitizer.cs)

It is quite easy to create a custom one. For this:

1. It should be derived from the * [SanitizerBase](https://github.com/akovanev/Sanitizer/blob/master/Akov.Sanitizer/Sanitizers/SanitizerBase.cs). 

2. A custom factory extending the [SanitizerFactory](https://github.com/akovanev/Sanitizer/blob/master/Akov.Sanitizer/Sanitizers/SanitizerFactory.cs) should be added.

For the example above with the `Card` let's create a demo.

<pre><code class="language-cs">var card = new
{
    cardholderName = "Alex Kovanev",
    cardNumber = "1234567890123456",
    month = 1,
    year = 2050,
    cvc = "555",
    address = new
    {
        line1 = "street 1",
        line2 = "",
        city = "Prague",
        country = "Chechia"
    }
};

string json = JsonConvert.SerializeObject(card);

string sanitizedData = _sanitizerService.ReplaceSensitiveData(json);

Console.WriteLine(sanitizedData);
</code></pre>

The output is.
<pre><code class="nohighlight">{"CardholderName": "@@@@@@","cardNumber": "1234********3456","month":1,"year":2050,"Cvc": "***","address":{"Line1": "stre****","line2":"","city":"Prague","country":"Chechia"}}</code></pre>
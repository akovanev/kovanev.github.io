---
layout: post
title: How to Sanitize Sensitive Data While Handling Http Requests
category: blogs
tag: Sanitizer
---

[Akov.Sanitizer](https://github.com/akovanev/Sanitizer/) is a super simple library that helps conceal sensitive information. 

### The Issue

If the data is bound/deserialized into a class instance, then the class can decide what should be done with sensitive information. However, in ASP.NET MVC and Web API projects, there is a gap between receiving the request and the moment when the data arrived at the controller. On this way, the data can be validated, rejected, and it is quite possible that it will not reach the controller. 

At the same time, it can be important for a project to log every request, having preliminarily extracted all the sensitive part. As we can not be sure that  requests will reach a controller, then we should act on the middleware layer. At this point the *binding type* is not known yet. 

It would be preferable if we could know at least the name of the *binding type*. Unfortunatley, this is not a general case. For instance, if the request payload is a json, then the name of the entry, in our case the *binding type*, will mostly be missing. 

That, in turn, leads to some other issues. Let's assume that we can implement a solution based on the property names only. That might make perfect sense, as we have to extract the sensitive data from the properties but not from the objects. But if we have more than one property with the same name in different request models, then how to find the right one to apply its replacement pattern?

That is challenging. The solution can be found if we agree that the names for the *sensitive* properties should inherit the same pattern for all request models.

In other words, if there is a property called `Number` within the `Card`, and it is sensitive, then the same named property, but within the `Room` must sutisfy the same rules. As it doesn't make sense for the `Room`, then the easiest solution is to rename the `Number` property, for instance, we can name it `CardNumber`.

Despite this approach imposes restrictions, it is still helpful in the issue context.

### The solution

If the previous part is accepted, than let's go ahead to the solution.

In the example below the card object is considered to be sanitized. The nuget package is available [here](https://www.nuget.org/packages/Akov.Sanitizer/).
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

Each type, which properties define the replacement patterns using the `[ReplaceFor]`, should be marked with the `[Sanitized]` attribute. 

There are two sanitizer types for the `[ReplaceFor]` out of the box: 

* [AsteriskSanitizer](https://github.com/akovanev/Sanitizer/blob/master/Akov.Sanitizer/Sanitizers/AsteriskSanitizer.cs)

* [PartialSanitizer](https://github.com/akovanev/Sanitizer/blob/master/Akov.Sanitizer/Sanitizers/PartialSanitizer.cs)

However, if needed, it is quite easy to add a custom one. For this:

1. It should be derived from the [SanitizerBase](https://github.com/akovanev/Sanitizer/blob/master/Akov.Sanitizer/Sanitizers/SanitizerBase.cs). 

2. A custom factory extending the [SanitizerFactory](https://github.com/akovanev/Sanitizer/blob/master/Akov.Sanitizer/Sanitizers/SanitizerFactory.cs) should be added.

Now, for the example with the `Card`, let's write some demo code.

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

var _sanitizerService= new SanitizerService(new []{typeof(Card).Assembly});
string sanitizedData = _sanitizerService.ReplaceSensitiveData(json);

Console.WriteLine(sanitizedData);
</code></pre>

The output is.
<pre><code class="nohighlight">{"CardholderName": "@@@@@@","cardNumber": "1234********3456","month":1,"year":2050,"Cvc": "***","address":{"Line1": "stre****","line2":"","city":"Prague","country":"Chechia"}}</code></pre>

Just a note. Technically, since we agreed that the same-named properties should behave accordingly, it is also possible to collect them all in one place. It is one hundred percent up to you.

<pre><code class="language-cs">[Sanitized]
public class SanitizedProperties
{
    [ReplaceFor(typeof(PartialSanitizer), pattern: "4,4,*")]
    public string? CardNumber { get; set; }

    [ReplaceFor(typeof(AsteriskSanitizer), pattern: "6,@")]
    public string? CardholderName { get; set; }

    [ReplaceFor(typeof(AsteriskSanitizer))]
    public string? Cvc { get; set; }

    [ReplaceFor(typeof(PartialSanitizer), 
        propertyName:"Line1", pattern: "4,0,*")]
    public string? AddressLine1 { get; set; }
}
</code></pre>
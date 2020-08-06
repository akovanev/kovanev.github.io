---
layout: post
title: How to sanitize sensitive data
---

As soon as it becomes necessary to store/log some user data, there is a need how to extract all sensitive information correctly. 

In the example below the card entity is considered to be sanitized.
<pre><code class="language-cs">[Sanitized]
public class Card
{
    [ReplaceWith(Pattern = SanitizePatternType.LastFour)]
    public string Number { get; set; }

    [ReplaceWith(Pattern = SanitizePatternType.Asterisk)]
    public string CardholderName { get; set; }

    public int? Year { get; set; }
    public int? Month { get; set; }

    [ReplaceWith(Pattern = SanitizePatternType.Asterisk)]
    public string Cvc { get; set; }
}
</code></pre>

If it were possible to mark the <code>Card</code> class with the <code>Sanitized</code> attribute and define patterns for its specific properties, the code then would look readable and enough flexible.

Unfortunately, there is no easy way to implement a universal solution. Much depends on the input data format. If it is the json, the class name will not be sent. Therefore, if several classes have the same named property but with different patterns or even no pattern listed, then the question is which pattern should be applied.

If agreed that the sensitive names should be unique then it may make sense to implement the basic part which could be extended in future.

In the example the <code>Asterisk</code> pattern implies replacement of characters with the asterisk string of the same length. <code>LastFour</code> does not hide the last four characters.
<pre><code class="language-cs">public class SanitizedAttribute : Attribute{ }

public class ReplaceWithAttribute : Attribute
{
    public SanitizePatternType Pattern { get; set; }
}

public enum SanitizePatternType
{
    Asterisk,
    LastFour
}

public class Sanitizer
{
    public string GetSanitizedValue(object value, SanitizePatternType patternType)
    {
        if (value == null) return String.Empty;

        string valueAsString = value.ToString();
        int length = valueAsString.Length;

        return patternType switch
        {
            SanitizePatternType.LastFour => $"{new String('*', length - 4)}" +
                                            $"{valueAsString.Substring(length - 4)}",
            _ => new String('*', length),
        };
    }
}</code></pre>

After the desired behavior defined the next step is implement the way how the CLR gets all meta information. The <code>SanitizeReflector</code> will be executing search on all passed assemblies. 

Obviously, this method should not run on each request due to performance reasons. In other words, the result should be stored in memory.
<pre><code class="language-cs">public class SanitizeReflector
{
    public Dictionary<string, SanitizePatternType> Collect(Assembly[] assemblies)
    {
        var dictionary = new Dictionary<string, SanitizePatternType>();

        foreach (Assembly assembly in AppDomain.CurrentDomain.GetAssemblies())
        {
            foreach (Type type in assembly.GetTypes())
            {
                var attributes = type.GetCustomAttributes(typeof(SanitizedAttribute), false);
                if (attributes.Length > 0)
                {
                    var properties = type.GetProperties()
                        .Where(prop => prop.IsDefined(typeof(ReplaceWithAttribute), false));

                    foreach (var property in properties)
                    {
                        var attribute = (ReplaceWithAttribute)
                            property.GetCustomAttributes(typeof(ReplaceWithAttribute), false)
                                .SingleOrDefault();

                        if (attribute != null && !dictionary.ContainsKey(property.Name))
                        {
                            dictionary.Add(property.Name, attribute.Pattern);
                        }
                    }
                }
            }
        }

        return dictionary;
    }
}</code></pre>

The <code>JsonSanitizeService</code>, based on regex, represents just one of the ways, probably not the fastest, of handling the specific json data.
<pre><code class="language-cs">public class JsonSanitizeService
{
    private const string JsonPatternTemplate = @"""{0}""\s*:\s*([""'])(?:(?=(\\?))\2.)*?\1";
    private const string JsonReplacementTemplate = @"""{0}"": ""{1}""";

    private readonly Dictionary<string, SanitizePatternType> _sanitizeDictionary;
    private readonly Sanitizer _sanitizer;

    public JsonSanitizeService(SanitizeReflector reflector, Sanitizer sanitizer)
    {
        _sanitizeDictionary = reflector.Collect(AppDomain.CurrentDomain.GetAssemblies());
        _sanitizer = sanitizer;
    }

    public string ReplaceSensitiveData(string value)
    {
        foreach (var token in _sanitizeDictionary)
        {
            string pattern = string.Format(JsonPatternTemplate, token.Key);
            Regex rgx = new Regex(pattern, RegexOptions.IgnoreCase);

            foreach (Match match in rgx.Matches(value))
            {
                int index = match.Value.LastIndexOfAny(
                    new[] { '\'', '"' },
                    match.Value.Length - 2);

                string replacementValue = _sanitizer.GetSanitizedValue(
                    match.Value.Substring(index + 1, match.Value.Length - index - 2),
                    token.Value);

                string replacement = string.Format(JsonReplacementTemplate, token.Key, replacementValue);
                value = Regex.Replace(value, pattern, replacement, RegexOptions.IgnoreCase);
            }
        }

        return value;
    }
}</code></pre>

Finally, the test code looks simple.
<pre><code class="language-cs">static SanitizeReflector _reflector = new SanitizeReflector();

static void Main(string[] args)
{
    var card = new Card
    {
        CardholderName = "Alex Kovanev",
        Number = "1234 5678 9012 3456",
        Month = 1,
        Year = 2050,
        Cvc = "555"
    };

    string json = JsonSerializer.Serialize(card);

    var sanitizer = new Sanitizer();
    var service = new JsonSanitizeService(_reflector, sanitizer);

    Console.WriteLine(service.ReplaceSensitiveData(json));
}</code></pre>

The output is.
<pre><code class="nohighlight">{"Number": "***************3456","CardholderName": "************","Year":2050,"Month":1,"Cvc": "***"}</code></pre>

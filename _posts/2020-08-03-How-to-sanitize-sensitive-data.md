---
layout: post
title: How to Sanitize Sensitive Data
category: blogs
tag: Sanitizing Data
---

As soon as it becomes necessary to store/log some user data, there can be a need to extract all *sensitive* information. 

In the example below the card request object is considered to be sanitized.
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

If it were possible to mark the `Card` with the `Sanitized` attribute so that replacement patterns were defined for all *sensitive* properties, the code then would look readable and enough flexible.

Unfortunately, there is no easy way to implement a universal solution supporting the attributes approach. Much depends on the input data and the format itself. For instance, if it is the json and the class represents the main request input, then the name of that class will not be sent. Therefore, if several requests have the same named property but with different patterns or even no pattern listed, then the question will be which pattern should be applied.

If agreed that the *sensitive* names should be unique then it would make sense to implement the basic part, which may be extended in future.

In the example the `Asterisk` pattern implies replacement of characters with the asterisk string of the same length as the property value has. The `LastFour` looks similar except that it does not hide the last four characters.
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

After the desired behavior defined, the next step is to implement how the CLR will get all meta information. For that the `SanitizeReflector` will be executing the search on all assemblies that may have classes marked with the `Sanitized` attribute. 

<pre><code class="language-cs">public class SanitizeReflector
{
    private readonly Lazy<Dictionary<string, SanitizePatternType>> _sanitizeDictionary;
    
    public SanitizeReflector(Assembly[] assemblies)
    {
        _sanitizeDictionary = new Lazy<Dictionary<string, SanitizePatternType>>(
            () => Collect(assemblies));
    }

    public Dictionary<string, SanitizePatternType> SanitizeDictionary => _sanitizeDictionary.Value;
    
    private Dictionary<string, SanitizePatternType> Collect(Assembly[] assemblies)
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

At last, the `JsonSanitizeService`, based on regex, represents just one of the ways, probably not the fastest, of handling the specific json data.
<pre><code class="language-cs">public class JsonSanitizeService
{
    private const string JsonPatternTemplate = @"""{0}""\s*:\s*([""'])(?:(?=(\\?))\2.)*?\1";
    private const string JsonReplacementTemplate = @"""{0}"": ""{1}""";

    private readonly Dictionary<string, SanitizePatternType> _sanitizeDictionary;
    private readonly Sanitizer _sanitizer;

    public JsonSanitizeService(SanitizeReflector reflector, Sanitizer sanitizer)
    {
        _sanitizeDictionary = reflector.SanitizeDictionary;
        _sanitizer = sanitizer;
    }

    public string ReplaceSensitiveData(string value)
    {
        foreach (var token in _sanitizeDictionary)
        {
            string pattern = string.Format(JsonPatternTemplate, token.Key);
            Regex regex = new Regex(pattern, RegexOptions.IgnoreCase);

            foreach (Match match in regex.Matches(value))
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
<pre><code class="language-cs">static

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

    var reflector = new SanitizeReflector(AppDomain.CurrentDomain.GetAssemblies());
    var sanitizer = new Sanitizer();
    var service = new JsonSanitizeService(reflector, sanitizer);

    Console.WriteLine(service.ReplaceSensitiveData(json));
}</code></pre>

The output is.
<pre><code class="nohighlight">{"Number": "***************3456","CardholderName": "************","Year":2050,"Month":1,"Cvc": "***"}</code></pre>

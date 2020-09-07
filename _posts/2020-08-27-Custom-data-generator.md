---
layout: post
title: Custom Data Generator 
category: blogs
tag: Data Generator 
---

In the previous post about <a href="/2020/08/26/Data-generator">Data Generator</a> I described how to get random test data.

Now I want to show how to extend the existing library with a custom generator. Let's call it uint.

<pre><code class="language-cs">"definitions": [
  //...
  {
    "name": "Product",
    "properties": [
      //...
      {
        "name": "absValue",
        "type": "uint",
        "failure": {
          "nullable": 0.1,
          "range": 0.1 
        }
      }
    ]
  },
  //...
]
</code></pre>

After updating the json input file I should write some C# code, having preliminarily included the dependency to the library.

<pre><code class="language-cs">public class UIntGenerator : GeneratorBase
{
    protected override object CreateImpl(PropertyObject propertyObject)
    {
        Random random = GetRandomInstance(propertyObject, nameof(CreateImpl));
        return random.GetInt(0, 1000);    
    }

    protected override object CreateRangeFailureImpl(PropertyObject propertyObject)
    {
        Random random = GetRandomInstance(propertyObject, nameof(CreateRangeFailureImpl));
        return random.GetInt(-100, -1);
    }
}

public class ExtendedGeneratorFactory : GeneratorFactory
{
    public override Dictionary<string, GeneratorBase> GetGeneratorDictionary()
    {
        Dictionary<string, GeneratorBase> generators = base.GetGeneratorDictionary();
        generators.Add("uint", new UIntGenerator());
        return generators;
    }
}

//In Main
var dg = new DG(new ExtendedGeneratorFactory());
Console.WriteLine(dg.Execute("data.json"));

</code></pre>

Using <code>GetRandomInstance</code> with passing <code>nameof(method)</code> into it will return the unique <code>Random</code> instance. If there was just one instance used in the application then it would not be possible to achieve correct random distribution for each property.

Just a note. It is not mandatory to use the <code>DG</code> runner. If you need just to generate data, then you can call the <code>CreateData</code> method of the <code>DataProcessor</code> class.
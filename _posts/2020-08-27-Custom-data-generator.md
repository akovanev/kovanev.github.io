---
layout: post
title: Custom Data Generator 
category: blogs
tag: Data Generator 
---

In the previous post about <a href="/blogs/2020/08/26/Data-generator">Data Generator</a> I described how to generate the random data.

If there is a need to add a custom type, there is a pretty simple way how to solve this issue.

Let add the `uint` type.

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

After having updated the json input, the next should be creating the `UIntGenerator` class. Do not forget to include the reference to the <a href="https://www.nuget.org/packages/Akov.DataGenerator/">Akov.DataGenerator</a>.

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

Calling the `GetRandomInstance` method, and passing `nameof(method)` as a parameter, should return the unique `Random` instance. Please do use it, rather than create your own `Random` object, as in this case, the randomness of the data will be lost at some extent.
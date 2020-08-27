---
layout: post
title: Custom data generator 
---

In the previous post about <a href="/2020/08/27/Data generator">Data generator</a> I described how to get random test data.

Now I want to show how to extend the existing library with a custom generator. Let's call it uint.

<pre><code class="language-cs">"templates": [
//...
{
  "type": "uint",
  "name": "uint1"
}]

//...

"definitions": [
  //...
  {
    "name": "Product",
    "properties": [
      //...
      {
        "name": "absValue",
        "template": "uint1",
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
    protected override object CreateImpl(Property property, Template template)
    {
        return GetRandom(0, 1000);
    }

    protected override object CreateRangeFailureImpl(Property property, Template template)
    {
        return GetRandom(-100, -1);
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

Just a note. It is not mandatory to use the <code>DG</code> runner. If you need just to generate data, then you can call the <code>CreateData</code> method of the <code>DataProcessor</code> class.
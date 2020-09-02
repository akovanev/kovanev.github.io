---
layout: post
title: Calculated Properties with Data Generator 
---

In the previous posts about <a href="/2020/08/26/Data-generator">Data Generator</a> and <a href="/2020/08/27/Custom-data-generator">Custom Data Generator</a> I described a couple of simple scenarios for random test data generation.

Since the version 1.2 the DG may contain logic for the calculated properties. It looks a little tricky and requires writing the custom code but the good thing (I believe) is that you can control what and how should be calculated.

So lets start from the definition. 

<pre><code class="language-cs">"definitions": [
  //...
"name": "Student",
"properties": [
  {
    "name": "id",
    "type": "guid",
    "pattern": "B",
    "failure": {
      "nullable": 0.1
    }
  },
  {
    "name": "firstname",
    "type": "file",
    "pattern": "firstnames.txt",
    "sequenceSeparator": ",",
    "failure": {
      "nullable": 0.075
    }
  },
  {
    "name": "lastname",
    "type": "file",
    "pattern": "lastnames.txt",
    "sequenceSeparator": ",",
    "failure": {
      "nullable": 0.1
    }
  },
  {
    "name": "fullname",
    "type": "calc"
  },
  //...
]
</code></pre>

The new type <code>file</code> allows to read data from a disk. The <code>pattern</code> specifies the file path. After data retrieved, the procees is equal to the <code>set</code> type.

The main goal is <code>fullname</code>, which I would like to be the calculated property that concatenates <code>firstname</code> and <code>lasttname</code>.

So I will create the <code>CalcGenerator</code> class that will derive from <code>CalcGeneratorBase</code>. 

<pre><code class="language-cs">public class CalcGenerator : CalcGeneratorBase
{
    protected override object CreateImpl(CalcPropertyObject propertyObject)
    {
        if (propertyObject.DefinitionName == "Student" && propertyObject.Property.Name == "fullname")
        {
            var val1 = propertyObject.Values.Single(v => v.Name == "firstname");
            var val2 = propertyObject.Values.Single(v => v.Name == "lastname");
            return $"{val1.Value} {val2.Value}";
        }

        throw new System.NotSupportedException("Not expected calculated property");
    }

    protected override object CreateRangeFailureImpl(CalcPropertyObject propertyObject)
    {
        throw new System.NotSupportedException("Range failure not supported");
    }
}
</code></pre>

The result should look similar to the the screenshot taken from the <a href="https://github.com/akovanev/DataGenerator/">JSON Editor Online</a> tool.

<img src="/public/calc.png">

The entire example of the definition part can be found on <a href="https://github.com/akovanev/DataGenerator/blob/master/Akov.DataGenerator.Console/data.json">Github</a>.
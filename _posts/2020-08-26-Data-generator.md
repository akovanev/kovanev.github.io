---
layout: post
title: Data Generator
category: blogs
tag: Data Generator 
---

The <a href="https://github.com/akovanev/DataGenerator/">Akov.DataGenerator</a> library helps to generate random data based on the <a href="https://github.com/akovanev/DataGenerator/blob/master/Akov.DataGenerator/Scheme/DataScheme.cs">DataScheme</a>.

There are two ways of how to populate the `DataScheme`. 
* Using json.
* Using the attributes from the `Akov.DataGenerator.Attributes` namespace.

In this article I will focus on the approach with json, while the attributes will be described later in <a href="/blogs/2020/09/07/Data-generator-attributes">Data Generator Attributes</a>. 

Lets suppose that there exists the couple of models.
<pre><code class="language-cs">class Product
{
    public string Name {get; set;}
    public Datetime LastUpdated {get; set;}
    public List&lt;Sku&gt; Skus {get; set;}
}

class Sku
{
    public string Name {get; set;}
    public decimal Price {get; set;}
}</code></pre>

As a developer, I want to generate one hundred products. It is expected that the `Product.Name`, is just like the `Sku.Name`, should contain only symbols `[a-z0-9]`. Lets supposet that there are some additional restictions on the length. The other properties, `LastUpdated` and `Price`, expect data in some ranges.

The main idea behind the <code>DataGenerator</code> is that it should be possible to generate not only the correct data. The several types of failures were added in the current version. 
* *null* result.
* *Out of range* result.
* Custom failure.

By default the result of custom failure is a string constant containing some error text. It can be overrided.

For each property that requires a failure, the probability in the range `(0,1)` should be specified.

Lets take a look at the example.

<pre><code class="language-cs">{
  "root": "Root",
  "definitions": [
    {
      "name": "Root",
      "properties": [
        {
          "name": "count",
          "type": "set",
          "pattern": "100"
        },
        {
          "name": "products",
          "type": "array",
          "pattern": "Product",
          "minLength" 100,
          "maxLength": 100
        }
      ]
    },
    {
      "name": "Product",
      "properties": [
        {
          "name": "name",
          "type": "string",
          "pattern": "abcdefghijklmnopqrstuvwxyz0123456789",
          "minLength": 10,
          "maxLength": 20,
          "minSpaceCount": 1,
          "maxSpaceCount": 2,
          "failure": {
            "nullable": 0.1,
            "custom": 0.1,
            "range": 0.05
          }
        },
        {
          "name": "lastUpdated",
          "type": "datetime",
          "pattern": "dd/MM/yy",
          "minValue": "20/10/19",
          "maxValue": "01/01/20",
          "failure": {
            "nullable": 0.1,
            "custom": 0.2,
            "range": 0.1
          }
        },
        {
          "name": "skus",
          "type": "array",
          "pattern": "Sku",
          "maxLength": 3
        }
      ]
    },
    {
      "name": "Sku",
      "properties": [
        {
          "name": "name",
          "type": "string",
          "pattern": "abcdefghijklmnopqrstuvwxyz0123456789",
          "minLength": 10,
          "maxLength": 50,
          "minSpaceCount": 1,
          "maxSpaceCount": 4,
          "failure": {
            "nullable": 0.25
          }
        },
        {
          "name": "price",
          "type": "double",
          "pattern": "0.00",
          "minValue": 0.0,
          "maxValue": 1999.99,
          "failure": {
            "nullable": 0.1,
            "custom": 0.05,
            "range": 0.15
          }
        }
      ]
    }
  ]
}</code></pre>

The `root` value `"Root"` specifies the `name` of the definition to entry. Every definition consists of a list of properties that may point to other definitions. There should not be circular references though. 

There are some attributes, like the `name` and `type`, that should be mandatory filled. For arrays and objects, and in some other cases, the `pattern` is also required. More information can be found on the project repository on <a href="https://github.com/akovanev/DataGenerator">GitHub</a>.

The code below shows the entire process.
<pre><code class="language-cs">var dg = new DG();
DataScheme scheme = dg.GetFromFile("data.json");
string output = dg.GenerateJson(scheme);
dg.SaveToFile("data.out.json", output);
</code></pre>

The result should be similar as on my screenshot taken from the <a href="https://jsoneditoronline.org/">JSON Editor Online</a> tool.

<img src="/public/datagen.png">
---
layout: post
title: Data Generator 
---

The idea behind this post is to show how specific test data can be easily prepared using the <a href="https://github.com/akovanev/DataGenerator/">DataGenerator</a>.

Here is a simple example of models.

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

Now I want to show how the data should be described in the <code>DataGenerator</code> input json. Basically the json consists of the <code>root</code> property and the <code>definitions</code>.  

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

The root value specifies the enrty definition name. Every definition may point to other ones, there should not be circular references though. 

Every property must have the <code>Type</code> value not empty. If the type is <code>array</code> or <code>object</code> then the <code>pattern</code> should be mandatory filled and reference to the definition in the file. For all the other types the default values are usually predefined. 

A property may have also a list of settings like <code>minValue</code> or <code>maxSpaceCount</code>. Apart of that, the <code>failure</code> object describes  probabilities of happening a specific failure. In this way you can easily control inconsitent data and their nature.

The result should look similar to the the screenshot taken from the <a href="https://github.com/akovanev/DataGenerator/">JSON Editor Online</a> tool.

<img src="/public/datagen.png">
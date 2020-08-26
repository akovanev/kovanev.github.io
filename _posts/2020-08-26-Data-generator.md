---
layout: post
title: Data generator 
---

The idea behind this post is to show how specific test data can be easily prepared using the <a href="https://github.com/akovanev/DataGenerator/">DataGenerator</a>.

Let's suppost that you have to import thousands of products from an external system. You wrote the code that retrieves the data by http, map them onto your object models and process them further. Before everything is ready to go in live, and also for the purpose of unit/integration tests, you want to run your import task on some mock data. You also know that, despite the expectations, some data are coming to you inconsistent. So you want to try the edge cases as well.

Here is a simple example of models that you expect to work with.

<pre><code class="language-cs">class Product
{
    public string Name {get; set;}
    public Datetime LastUpdated {get; set;}
    public List&lt;Sku&gt; Skus {get; set;}
}

class Sku
{
    public string SkuName {get; set;}
    public decimal Price {get; set;}
}</code></pre>

Now I want to show you how the data should be described in the DataGenerator input json. Basically the json consists of three sections: <code>templates</code>, <code>root</code> and <code>definitions</code>. The templates section is responsible for running a specific generator based on the property type and pattern. 

<pre><code class="language-cs">"templates": [
{
  "type": "object",
  "name": "root",
  "pattern": "Root"
},
{
  "type": "array",
  "name": "array_product_template",
  "pattern": "Product"
},
{
  "type": "string",
  "name": "string_template1",
  "pattern": "abcdefghijklmnopqrstuvwxyz0123456789"
},
{
  "type": "datetime",
  "name": "datetime_template1",
  "pattern": "dd/MM/yy"
},
{
  "type": "array",
  "name": "array_sku_template",
  "pattern": "Sku"
},
{
  "type": "double",
  "name": "price_template",
  "pattern": "0.00"
},
{
  "type": "set",
  "name": "count_template",
  "pattern": "100"
}]
</code></pre>

The section should contain the template for the root element. It is mandatory that the <code>type</code> must be <code>object</code>. In its turn the root element is considered to be an enrty point.

<pre><code class="language-cs">"root":
{
  "name": null,
  "template": "root"
},
</code></pre>

At last the definitions section describes the user defined objects, arrays and their properties.

<pre><code class="language-cs">"definitions": [
  {
    "name" : "Root",
    "properties":[
      {
        "name":"count",
        "template": "count_template"
      },
      {
        "name":"products",
        "template": "array_product_template",
        "maxLength": 100
      }
    ]
  },
  {
    "name": "Product",
    "properties": [
      {
        "name": "name",
        "template": "string_template1",
        "minLength": 10,
        "maxLength": 20,
        "minSpaceCount": 1,
        "maxSpaceCount": 2,
        "failure": {
          "nullable": 0.1,
          "format": 0.1,
          "range": 0.05 
        }
      },
      {
        "name": "lastUpdated",
        "template": "datetime_template1",
        "minValue": "20/10/19",
        "maxValue": "01/01/20",
        "failure": {
          "nullable": 0.1,
          "format": 0.2,
          "range": 0.1
        }
      },
      {
        "name": "skus",
        "template": "array_sku_template",
        "maxLength": 3
      },
    ]
  },
  {
    "name": "Sku",
    "properties": [
      {
        "name": "name",
        "template": "string_template1",
        "minLength": 10,
        "maxLength": 50,
        "minSpaceCount": 1,
        "maxSpaceCount": 4,
        "failure": {
          "nullable": 0.25,
        }
      },
      {
        "name": "price",
        "template": "price_template",
        "minValue": 0.0,
        "maxValue": 1999.99,
        "failure": {
          "nullable": 0.1,
          "format": 0.05,
          "range": 0.15
        }
      }
    ]
  }]
</code></pre>

A property type may have a list of settings like <code>minValue</code> or <code>maxSpaceCount</code>. Apart of that, every propery has the <code>failure</code> object with the values that define probabilities of happening a specific failure. In this way you can easy control the amount and types of inconsitent data.

After collecting all sections together and running the DataGenerator, the result should look like on the screenshot taken from the <a href="https://github.com/akovanev/DataGenerator/">JSON Editor Online</a> tool.

<img src="/public/datagen.png">
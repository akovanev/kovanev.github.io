---
layout: post
title: Data Generator Attributes
category: blogs 
tag: Data Generator 
---

In the fisrt post about <a href="/blogs/2020/08/26/Data-generator">Data Generator</a> I described the scenario when the `DataScheme` is populated from a json input. Another way to get the `DataScheme` is by creating new or updating the existing classes in some solution. This option is available since the generator version 1.3.

Let's suppose that we have a dto/model class called `Student`. It is accepatable to mark its properties with attributes from the `Akov.DataGenerator.Attributes` namespace. In this case the data generation process will be done for the type/class which also has another responsibility, being a dto/model object.

Pros:

* If the dto/model class has changed, the data generation process doesn't require additional changes. So you will be sure that everything is up to date.

Cons:

* The dto/model may look overloaded with the generator attributes.
* Any deserialization onto the dto/model, based on reflection, might be a little longer, as the amount of properties attributes is increased.

Another way is to create a separate class that will participate only in the data generation. I prefer to start naming such classes with the prefix `Dg`.

The `DgStudentCollection` represents a class that is supposed to have some random data generated. Despite you may see a lot of attributes above the properties, even if a class doesn't have any, it is still possible to generate the data for it using the defaults.  

<pre><code class="language-cs">public class DgStudentCollection
{
    [DgCalc]
    public int? Count { get; set; }

    [DgLength(Max = 100)]
    public &lt;DgStudent&gt;? Students { get; set; }
}

public class DgStudent
{
    [DgFailure(NullProbability = 0.2)]
    public Guid Id { get; set; }

    [DgSource("firstnames.txt")]
    [DgFailure(NullProbability = 0.1)]
    public string? FirstName { get; set; }

    [DgSource("lastnames.txt")]
    [DgFailure(NullProbability = 0.1)]
    public string? LastName { get; set; }

    [DgCalc] //supposed to be calculated
    public string? FullName { get; set; }

    [DgName("test_variant")]
    public Variant Variant { get; set; }

    [DgName("test_answers")]
    [DgRange(Min = 1, Max = 5)]
    [DgLength(Max = 5)]
    public int[]? TestAnswers { get; set; }

    [DgName("encoded_solution")]
    [DgPattern("abcdefghijklmnopqrstuvwxyz0123456789")]
    [DgLength(Min = 15, Max = 50)]
    [DgSpacesCount(Min = 1, Max = 3)]
    [DgFailure(
        NullProbability = 0.1,
        CustomProbability = 0.1,
        OutOfRangeProbability = 0.05)]
    [DgCustomFailure("####-####-####")]
    public string? EncodedSolution { get; set; }

    [DgName("last_updated")]
    [DgPattern("dd/MM/yy")]
    [DgRange(Min = "20/10/19", Max = "01/01/20")]
    [DgFailure(
        NullProbability = 0.2,
        CustomProbability = 0.2,
        OutOfRangeProbability = 0.1)]
    public DateTime? LastUpdated { get; set; }

    public List&lt;DgSubject&gt;? Subjects { get; set; }

    public DgSubject? Subject { get; set; }
}
</code></pre>

All `IEnumerable` collections are supported and will be handled in the same way as arrays. 

More information about the attributes you can find on the <a href="https://github.com/akovanev/DataGenerator/">Akov.DataGenerator</a> GitHub page.

The next code will demostrate the process of data generation. 
<pre><code class="language-cs">var dg = new DG(
    new StudentGeneratorFactory(), 
    new DataSchemeMapperConfig { UseCamelCase = true });

DataScheme scheme = dg.GetFromType&lt;DgStudentCollection&gt;();

string jsonData = dg.GenerateJson(scheme);
</code></pre>

The `StudentFactory` is a custom factory needed for populating calculated properties. Here is the <a href="https://github.com/akovanev/DataGenerator/blob/master/Akov.DataGenerator.Demo/StudentsSampleTests/Tests/Mocks/StudentGeneratorFactory.cs">example</a>.
---
layout: post
title:  "Why we use fsharp data in prod"
date:   2017-08-18 13:04:00 +0200
categories: fsharp data prod
---

I am currently working at a company that deal with hundreds of connectivities (apps protocol and domain adapters + big data). 

We have since 2014 used fsharp data progressively in production (I use fsharp in prod since 2012).

Today we have more than 10% and 1 of the top 3 (top business, high performance) in prod plus a custom big data solution in full fsharp (the company guarantee in case of sales/stockout risk for exchanged data thanks to zmq + 7z + Azure)

We use fsharp data into different stages of a project : estimation, proof of concept, dev.

After 3 years of production with fsharp, I would like to serialize my experience.

## Our context

A connectivity is an adapter to exchange data with a partner.
This partner use often a "standard". This standard [OTA](http://opentravel.org/#sthash.4w8LbhfH.dpbs) try to unify the communication between two partner. The bad news is : everything in OTA is optional.

> 70% of our partner try to use OTA standard and others use their own protocol

What's the problem ?

> The protocol is verbose (really hard to see with the key data in an editor, due to semantic issue, interval scope, ...)

> Every partner has his own set of required and optional data.

> Values are defined on an interval (another post could be dedicated to this part) and each partner has his own interval valued scope

> Some partner does not use OTA anymore or they don't know that OTA exists (due to historical reason, ...)

> Even if the partner has a poor interface, we have to integrate it has quick as possible

## Code analysis on our existing apps
When I was arrived in this company, I had ran [NDepend](http://www.ndepend.com/) and [Sonar](https://www.sonarqube.org/) to see : 
 - how modules are dependent (visualize and navigate to the dependency graph)
 - how many lines of codes depend on a module (+ the use of Resharper)
 
Each connector has approximatively 35% of codes that use ```System.Xml.Linq``` and 50% of tests are dedicated to serialization/deserialization part (test are focussed on serialization + domain adaptation without separation).

## Maintenability
When we have to fix an existing app, we have to add a test, run the test and see where there was a problem.
It is really boring and the partner will often break their compatibility.

When we build a new connector with xml linq, we have in the end a lot of method like this : 

```
private static Stuff BuildStuff(XDocument responseXml, XElement name)
```

Passing ```XElement```, ```XDocument```, any xml linq parameter does not supply the context : the what. We have constantly to dev with the sample to the right side because it is totally dynamic.

When I analyse this kind of code it is really hard to analyse the code without the sample and the big picture.

Here is a xml data sample (this is not a domain dependent sample) : 

```
<node>

	<item>
		<value>10</value>
	</item>
	<value>100</value>

</node>
```

the associated code that get all values: 

```
class Program
{
	private static IEnumerable<decimal> Values(XElement x)
	{
		return x.Descendants("value").Select(y => Decimal.Parse(y.Value));
	}
	static void Main(string[] args)
	{
		var xml = XDocument.Parse(@"<node><item><value>10</value></item><value>100</value></node>");

		var v1 = Values(xml.Root);
		var v2 = Values(xml.Descendants("item").First());

		Console.WriteLine("from root");
		v1.ToList().ForEach(x => Console.WriteLine("value : {0}", x));

		Console.WriteLine("from item");
		v2.ToList().ForEach(x => Console.WriteLine("value : {0}", x));

		Console.ReadLine();
	}
}
```

When the data is bigger, the number of this kind of method is also bigger you have to analyse the usage of the method to understand the context.

In the previous sample, the function does not return the same value depending of the context (because it force to use the metadata, the XElement instead of using the real data).

> Note : I try to use xsd.exe to generate the same C# code but there is some strange partner xml that not works correctly with xsd. When it works, the generated code is really hard to update.

> in 99% our partner does not supply an xsd. This solution works only for xml and not for json

Here is a version with the XmlProvider of [Fsharp.Data](http://fsharp.github.io/FSharp.Data/) : 
```
#r @"..\packages\FSharp.Data\lib\net40\FSharp.Data.dll"
#r "System.Xml.Linq"

open FSharp.Data

let [<Literal>] nodeSample = """<node>

	<item>
		<value>10</value>
	</item>
	<value>100</value>

</node>"""

type Node = XmlProvider< nodeSample >

let node = Node.Parse(nodeSample)

printfn "from root"
printfn "%i" (node.Value)
printfn "from item"
printfn "%i" (node.Item.Value)
```

> In this version, there is no need to build a function to extract and parse the value.
> There is no ambigous context between the item and root value

## The friend of estimation

In our team, we analyse complexity to dispatch work by using t-shirt size scale. 
I use fsharp to build a simple fsx with our domain, the data sample of the partner in order to understand their domain (It is not only protocol but domain and business adaptation over the time intervals).

FSharp.Data speed up the process because the porcelain and plumbing part (xml, json, ... all service layer) is done by FSharp.Data and we have a better complexity feedback.
The reason it that sometimes the partner protocol is really verbose and it is harder to navigate into a json or xml than an instanciated object in the interactive mode.

Before using fsharp.data, I was a System.Xml.Linq ninja :) for estimation purpose, but for now, each time there was a verbose/complex data to understand, I first use a fsharp script.

## Flexibility

When the partner change his protocol, we have to change the adapter with the new sample and it is really hard to make a diff between 2 sample where data are completely different.
So, we add a new test but due to the context problem, it is really hard to see if the change have same name but not the same structure and scope.

> If all test are done, there is only one test for the new protocol and other based on old protocol
> If there is a cardinality problems it is really hard to see how it impact the business part (price scope for example).

I prefer use fsharp because if there is a new scope (a new node for example), the code will not compile and we have to reason about the whole code. 

> How to support both new and old protocol in the same app without error ?

> We need a tool to challenge the existing code with the new protocol

Here is some example where fsharp data help us to increase the quality and the velocity of our process :

Like Xml Linq, FSharp.Data is tolerent : 

- What about changes that we don't need : 

```
#r @"..\packages\FSharp.Data\lib\net40\FSharp.Data.dll"
#r "System.Xml.Linq"

open FSharp.Data

type Node = XmlProvider< """<root><code>ABC</code><id>50</id></root>""" >

let node = Node.Parse("""<root><code>ABC</code></root>""")
```

> Note that the supplied xml to parse method does not have an id element (runtime and design time are different)

The interactive display : 
```
Code : ABC
```

- Breaking changes at runtime (when the partner break things directly in prod : data differ from the spec at runtime)

```
#r @"..\packages\FSharp.Data\lib\net40\FSharp.Data.dll"
#r "System.Xml.Linq"

open FSharp.Data

type Node = XmlProvider< """<root><code>ABC</code><id>50</id></root>""" >

let node = Node.Parse("""<root><code>ABC</code></root>""")

printfn "Id : %i" (node.Id)
```

The code compiles but there was a problem at runtime : 
```
System.Exception: XML mismatch: Expected exactly one 'id' child, got 0
```

> For this case, you should consider logging this error with the associated data.

- Breaking changes on legacy code (quick fix) : 

```
#r @"..\packages\FSharp.Data\lib\net40\FSharp.Data.dll"
#r "System.Xml.Linq"

open FSharp.Data

type Node = XmlProvider< """<root><code>ABC</code></root>""" >

let node = Node.Parse("""<root><code>ABC</code></root>""")

printfn "Id : %i" (node.Id)
```

> Note that the required Id (used by the printfn) has been removed from the given sample

There is an error at design time that we have to fix without adding a new unit test (may be we have to adapt the code without touching test and samples if you want to support both).

- When you have to support multiple version

```
#r @"..\packages\FSharp.Data\lib\net40\FSharp.Data.dll"
#r "System.Xml.Linq"
open FSharp.Data

type Node = XmlProvider< """
<samples>
<!-- version 1 -->
<root><code>ABC</code></root>
<!-- version 2 -->
<root><productcode>ABC</productcode></root>

</samples>""", SampleIsList=true >

let toResult error = function
    | Some x -> Ok x
    | None -> Error error

let (<|>) x y = 
    match x with
    | Ok x' -> Ok x'
    | Error e1 ->
        match y with 
        | Ok y' -> Ok y'
        | Error e2 -> sprintf "%s\r\nor%s" e1 e2 |> Error

let prettyprint (node:Node.Root) = 
    printfn "Code : %A" ((node.Code |> toResult "expected code element") <|> (node.Productcode |> toResult "expected Productcode element"))

let nodeLegacy = Node.Parse("""<root><code>ABC</code></root>""")
let nodeNew = Node.Parse("""<root><productcode>ABC</productcode></root>""")

prettyprint nodeLegacy
prettyprint nodeNew
```

> I have added the 2 versions in the samples of the XmlProvider by adding a new <samples> root node and the SampleIsList to true parameter. In that case, the XmlProvider skip the first element and consider many child elements to be processed.

> I have added 2 function : the toResult that add the errorMessage when the element is not present and the orElse ```(<|>)``` function to handle the 2 versions. The prettyprint function display the result

## Performance

FSharp.Data use an erased type provider, so the cost of this abstraction is paid at design time. 

When the application is built, all supplied types (like the type Node in our sample) are discarded.

So all the abstraction is inlined by a System.Xml.Linq call. The performance is equivalent to the System.Xml.Linq version.

Our most verbose app that use fsharp.data exchange near 3.000.000 rq/day.

## Conclusion

- We have first use a fsharp module to existing csharp apps in order to infer the partner domain
- The performance are equivalent to a hand made Xml Linq version
- The estimation part is quicker because it require less xml linq plumbing
- Quick fix is possible due to the move from unit test feedback to design time and compiler feedback
- There was more tests around the domain of our partner and not only on the serialization/deserialization part
- I have replaced my linqpad + fiddler by fsharp script
- When the breaking changes are too high, we completely rewrite the adapter by using fsharp and see that the LOC has really decreased (near 40%)
- The proof of concept code could be reused (better than extract linqpad code and import to existing + code after unit test)
- The language ML syntax provide a better way to avoid NRE (Null Reference Exception) when you only deal with fsharp. There is Option.ofObj adapters when object came from csharp.
- We have started another experience and made our adaptation of the interval valued in full fsharp [Outatime](https://github.com/cboudereau/Outatime)
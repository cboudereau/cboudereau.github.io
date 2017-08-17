---
layout: post
title:  "Why we use fsharp data in prod"
date:   2017-08-17 13:04:00 +0200
categories: fsharp data prod
---

Actually, I work for a company that deal with hundreds of connectivities. 

We have since 2014 used fsharp data progressively in production.

Today we have more than 10% and 1 of the top 3 (top business, high performance) in prod plus a custom big data solution in full fsharp (zmq + 7z + Azure)

We use fsharp data into different stages of a project : estimation, proof of concept, dev

## Our context

A connectivity is an adapter to exchange data with a partner.
This partner use often a "standard". This standard [OTA](http://opentravel.org/#sthash.4w8LbhfH.dpbs) try to unify the communication between two partner. The bad news is : everything in OTA is optional.

What's the problem ?

> Every partner has his own set of required and optional data.

> Values are defined on an interval (another post could be dedicated to this part) and each partner has his own interval valued scope

> Some partner does not use OTA anymore or they don't know that OTA exists (due to historical reason, ...)

> Even if the partner has a poor interface, we have to integrate it has quick as possible

## Code analysis on our existing apps
When I was arrived in this company, I had ran [NDepend](http://www.ndepend.com/) and [Sonar](https://www.sonarqube.org/) to see : 
 - how modules are dependent (visualize and navigate to the graph)
 - how many lines of codes depend on a module
 
Each connector has around 35% of codes that use ```System.Xml.Linq``` and 50% of tests are dedicated to serialization/deserialization part.

## Maintenability
When we have to fix an existing app, we have to add a test, run the test and see where there was a problem.
It is really boring and the partner will often break there compatibility.

When we build a new connector, with xml linq we have in the end a lot of method like this : 

```
private static Stuff BuildStuff(XDocument responseXml, XElement name)
```

Passing ```XElement```, ```XDocument```, any xml linq parameter does not supply the context : the what. We have constantly to dev with the sample to the right side because it is totally dynamic.

When I analyse this kind of code it is really hard to analyse the code without the sample and the big picture.

Here is an example : 

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

When the code is bigger you have to analyse the usage of the method to understand the context.

In the previous sample, the function does not return the same value depending of the context.

Here is a version with the XmlProvider of [Fsharp.Data](http://fsharp.github.io/FSharp.Data/) : 
```
#r @"..\packages\FSharp.Data\lib\net40\FSharp.Data.dll"
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
I use fsharp to build a simple fsx with our domain, the data sample of the partner in order to understand their domain (It is not only protocol but domain and business adaptation over the time intervals)
FSharp.Data speed up the process because the porcelain and plumbing part (xml, json, ... all service layer) is done by FSharp.Data and we have a better complexity feedback.
The reason it that sometimes, the partner protocol is really verbose and it is harder to navigate into a json or xml than an instanciated object in the interactive mode).

Before using fsharp.data, I was a System.Xml.Linq ninja :) for estimation purpose, but for now, each time there was a verbose/complex data to understand, I first use a fsharp script.

## Flexibility

When the partner change their protocol, we have to change the adapter with the new sample and it is really hard to make a diff between 2 sample where data is completely different.
So, we add a test like but due to the context problem, it is really hard to see if the change have same name but not the same structure and scope.

Like Xml Linq, FSharp.Data is tolerent : 

- What about changes that we don't need : 

```
#r @"..\packages\FSharp.Data\lib\net40\FSharp.Data.dll"
open FSharp.Data

type Node = XmlProvider< """<root><code>ABC</code><id>50</id></root>""" >

let node = Node.Parse("""<root><code>ABC</code></root>""")
```

> Note that the parse content does not have an id element

The interactive display : 
```
Code : ABC
```

- Breaking changes at runtime (when the partner break things directly in prod : data differ from the spec at runtime)

```
#r @"..\packages\FSharp.Data\lib\net40\FSharp.Data.dll"
open FSharp.Data

type Node = XmlProvider< """<root><code>ABC</code><id>50</id></root>""" >

let node = Node.Parse("""<root><code>ABC</code></root>""")

printfn "Id : %i" (node.Id)
```

The code compiles but there was a problem at runtime : 
```
System.Exception: XML mismatch: Expected exactly one 'id' child, got 0
```

- Breaking changes on legacy code (when a make a quick fix) : 

```
#r @"..\packages\FSharp.Data\lib\net40\FSharp.Data.dll"
open FSharp.Data

type Node = XmlProvider< """<root><code>ABC</code></root>""" >

let node = Node.Parse("""<root><code>ABC</code></root>""")

printfn "Id : %i" (node.Id)
```

> Note that the required Id (used by the printfn) has been removed from the given sample

There is an error at design time that we have to fix without adding a new unit test.

## Performance

FSharp.Data use an erased type provider, so the cost of this abstraction is paid at design time. 

When the application is built, all supplied types (like the type Node in our sample) are discarded.

So all the abstraction is inline by a System.Xml.Linq call. The performance is equivalent to the System.Xml.Linq version.

## Conclusion

- We have first use a fsharp module to existing csharp apps in order to infer the partner domain
- The performance are equivalent to a hand made Xml Linq version
- The estimation part is quicker because it require less xml linq plumbing
- Quick fix is possible due to the move from unit test feedback to design time and compiler feedback
- There was more test around the domain of our partner and not only on the serialization/deserialization part
- I have replaced my linqpad + fiddler by fsharp script
- When the breaking changes are too high, we completelty rewrite the adapter by using fsharp and see that the LOC has half 
- The proof of concept code could be reused (better than extract linqpad code and import to existing + code after unit test)
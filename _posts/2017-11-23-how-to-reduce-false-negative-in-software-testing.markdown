---
layout: post
title:  "How to reduce false negative in software testing!"
date:   2017-11-23 13:04:00 +0200
categories: fsharp qa scripting
---

As a QA, you have to write/maintain some code in order to automate your activities.
This activity is the same dev activity but... How do you test your code ?

![Inception]({{ "/images/inception.jpg" }})

## Use the write tools

Even if you use a tool, you have to integrate it to the existing QA environment. So you have to write code.
Some QA teams builds a real app, libraries or exe.

If you build a library, consider this activity like a dev and make unit test/build/ci.

If you make an exe you are into the wrong path! If you do that you also need a QA!

## QA of QA! What's wrong ??
The QA of QA comes when you have a codebase that evolves and produces false negative.
For example, your campaign is failing because you have introduced a bug into you setup script or whatever. 
This business part is not tested and should not be tested!

If the code is reusable and technical, it's ok to build a library (ie:scrapping lib, xml utils, ...)

What about plumbing part like setup/teardown functions ?
Generally this sort of code is hard to maintain and you have to run the entire test.
Running constantly the entire test to fix the setup/teardown part is wasting time! 
This kind of tests could ran into 10 or 30s (or even higher) and because they are too long you loose the focus of the last problem. 
So the number of iterations is higher than fixing a codebase with a little test.
Associated unit tests are hard to produce because it is side effect functions like executing sql query or enabling queue logging.
If you have this problem, try to use alpha test.

An alpha test is a little test built by the developper in order to check a very little part (not the whole QA test but one step of the setup/teardown part).
This test must be easy to write and easy to check.

## How do you run alpha test from existing code base ?
First of all, you have to use a language having scripting and REPL properties like fsharp.
In my experience, our QA team uses a csharp solution with specflow but there were no tests on the setup and teardown part.
Sometimes they have false negative due to regression on the setup/teardown part. After understanding the problem, the queue logging part was not ok.
The problem was too complex to fix into the existing codebase, so we have used the divide and conquer principle!

First we have extracted the code that activate the logging part into a fsharp script.
Side effects are often hard to check so we used alpha testing by calling the fsharp func making the side effects and checked that the log was activated.
Thanks to fsharp and its REPL + scripting properties executing a part of the setup/teardown is easier than the previous one plus the flow to produce the code force the QA dev to build very little function that do one thing at a time and packaging them into modules.

Here is for example hard code to test : 
```
let hello = printfn "hello %s"
```

You can say that testing the StandardOuput is ok, but you have to add a lots of plumbing into your code to have the benefits.
For this kind of function calling them directly and runnning into an interactive is very easy because only one test is ok to check the effect.

```
hello "QA"
```

It is very every to press Alt+Enter and check that the function display hello QA!

Here is what the interactive displays : 

```
val hello : (string -> unit)

>
>
hello QA
val it : unit = ()
```

## Conclusion
- As a QA, if you have constantly false negative, try to use alpha test before pushing the code.
- Alpha test, reduce the time to fix the problem by reducing the test time.
- Alpha test driven dev it is ok for not valuable automated test (the check is too complicated or the cost is too high).
- If you are a .Net QA, try to use fsharp in place of existing codebase (ie : the setup/teardown part).
- The flow to produce scripts is more efficient by always checking that the code is working.
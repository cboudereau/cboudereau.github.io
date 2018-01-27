---
layout: post
title:  "Parser Combinator in real life"
date:   2017-08-09 13:14:01 +0200
categories: fsharp, parser combinator
image: "/images/parsercombinator.png"
---

I had to write an application that handle 2200 msg per second. 
This application consist of indexing and compress data with 7z.

> I will write another post for the usage of the awesome [fszmq](https://github.com/zeromq/fszmq) and a custom interop of 7z in a later post.

## Disclaimer
> I attempted to use Linq to xml that load entirely the document element by element in memory causing LOH (on data that I don't want to index) problem when a big element is loaded.

> The XmlProvider is based on Linq so I had the same problem.

> Both are very usefull and the usage of parser combinator is overkill when there is no reason :)

## The Problem
All my documents consist of a xml document that contains information to store and organize for indexation process and other data to just forward (to the 7z zipper part).

2 elements of the xml contain logs of partners request/response data. 
Those 2 elements could have more than 10MB and I don't want to load this kind of message entirely due to LOH + GC problems (avg is near 1200msg/s :)).

When I check the linq to xml implementation, it load the document entirely element by element in memory with the XmlReader.

So, I first wrote a simple code that use the XmlReader but : 

- I don't know when this XmlReader read or not the data. 
- It is hard to have a clean context
- But it works!

After 2 weeks in production, I discover a new corner case like this : 

An Id could be represent as 

```
<root>
<Id>1</Id>
<Request>Big Data here !</Request>
<Response>Big Data here !</Response>
</root>
```
```
<root>
<Id>0</Id>
<items><item><Id>1</Id><Request>Big Data here !</Request><Response>Big Data here !</Response></item></items>
</root>
```

I remember how Linq and FSharp.Data were very usefull in this case but for the performance reason it is not ok!

After dealing with if/else/pattern matching approach, I tried Parser Combinator style.
Instead parsing char, the context used by the parser is the XmlReader.
I don't want to rewrite a full xml parser because parsing xml is too hard and XmlReader is ok for that point. 

## A Solution in Functional style

> I have to index a lots of document so compose a parser should be easy
> The solution should offer a maximum of flexibility to compose things together

I would like to express code like this : 

```
type Entry = { Id:string } 
let convert message = 
	use reader = XmlReader.Create(message:Stream)
	let entry id = { Id=id }
	let entryP = ffwd "Id" >>. element (id >> Ok) "Id"
	run reader entryP
```

The Parser type that wrap the parser function.
The aim of parser function is like an interface but instead it hide the context and error types

```
type Parser<'x, 'a> = Parser of ('x -> Result<'a, string list>) 
let run context (Parser p) = p context
```

Where 'x could be the XmlReader and the run function that just unwrap the parser function and apply the given context.

```>>.``` is an operator that takes 2 parser and return the result of the right parser. 
If you understand the meaning of the point you should intuitively understand that the ```.>>``` do the same except that it return the left result
```.>>.``` return both left and right result.

Quite Simple !

the full parser code based on XmlReader

```
type Parser<'x, 'a> = Parser of ('x -> Result<'a, string list>) 

//The code is based on the latest fsharp 4.1 (install vs2017 + fsharp 4.1 for eg)
open Result

let run context (Parser p) = p context
```

Define a map function that 
- Execute the parser x
- If is ok, the parser x return a x' result. Just apply x' to f and return Ok
- Stop if there is an error and do not apply f (show the error string list for exemple)

```
let map f x = 
    fun reader ->
        match run reader x with
        | Ok x' -> f x' |> Ok
        | Error e -> Error e
    |> Parser

let (<!>) = map
```

Define an apply function that do
- Execute the run on the reader context on f and x
- if parsers f and x in sequence return both ok then return Ok f' x'
- Like map stop on error

```
let apply f x = 
    fun reader ->
        match run reader f, run reader x with
        | Ok f', Ok x' -> Ok (f' x')
        | Error e, _ | _, Error e -> Error e
    |> Parser

let (<*>) = apply
```

The famous so called "combinator" function that

- Parse x and y in sequence and return one or both depending on the "." on operator

```
/// return y result if x and y are Ok in sequence
let (>>.) x y = 
    fun reader ->
        match run reader x with
        | Ok _ -> run reader y
        | Error e -> Error e
    |> Parser

/// return x result if x and y are Ok in sequence
let (.>>) x y = 
    fun reader ->
        match run reader x with
        | Ok x' -> 
            match run reader y with
            | Ok _ -> Ok x'
            | Error e -> Error e
        | Error e -> Error e
    |> Parser

/// return (x, y) as struct results if x and y are Ok in sequence
let (.>>.) x y = 
    fun reader ->
        match run reader x with
        | Ok x' -> 
            match run reader y with
            | Ok y' -> Ok (struct(x', y'))
            | Error e -> Error e
        | Error e -> Error e
    |> Parser
```

The aim of many function is like an unfold on List, 
so while there is no error aggregate result to a list and return the list on the first error

```
let many x = 
    fun reader ->
        List.unfold 
            <| fun reader -> match run reader x with Ok x' -> Some (x', reader) | Error _ -> None
            <| reader
        |> Ok

   |> Parser
```

The orElse combinator is like a branch condition, if x has an error then run y.
But if x is ok, just return x without running y
   
```
let orElse x y = 
    fun reader ->
        match run reader x with
        | Ok x' -> Ok x'
        | Error e1 -> 
            match run reader y with
            | Ok y' -> Ok y'
            | Error e2 -> Error (List.append e1 e2)
    |> Parser

let (<|>) = orElse
```

Ok now I have the abstraction to parse any context.

Remember that my context is the XmlReader

The Xml helper function



```
open System.Xml
```

Hey it is xml :)

I Have to deal with XmlReader that confuse technical error with business error with the same way...
So lets go to catch!
```
let tryF name f (reader:XmlReader) = 
    try
        f reader
    with ex -> Error [sprintf "expected %s got an error:\r\n%O" name ex]
```

The ffwd just fast forward the XmlReader to the given name

```
let ffwd name = 
    fun (reader:XmlReader) -> 
        while reader.Name <> name && not reader.EOF && reader.Read() do ()
        |> Ok
    |> Parser
```

The element function read the element with the given name, and apply the function f to the content

```
let element f name = 
    tryF name <| fun reader ->
        reader.ReadStartElement(name)
        if reader.NodeType = XmlNodeType.Text then
            let result = f (reader.ReadContentAsString())
            reader.ReadEndElement()
            result 
        else 
            let result = f ""
            if reader.NodeType = XmlNodeType.EndElement then reader.ReadEndElement()
            result
    |> Parser
```

The nested function treat the nested element case like : 
```
<item>...</item>
```

```
let nested name p = 
    tryF name <| fun reader ->
        reader.ReadStartElement(name)
        let result = run reader p
        reader.ReadEndElement()
        result
    |> Parser
```

The pIgnore skip an element and return unit (very usefull with .>> pIgnore or pIgnore >>.)

```
let pIgnore = 
    fun (reader:XmlReader) -> 
        reader.ReadStartElement()
        reader.Skip()
        reader.ReadEndElement()
        Ok ()
    |> Parser 
```

## Some Examples 
A sample parser implementation : 
```
open System 
open System.IO

type Entry = 
    { Keys: string Set
      Date : DateTime 
      Data: Stream }

let convert (message:Stream) = 
    let reader = XmlReader.Create(new StreamReader(message))
    let elt f x = element (f >> Ok) x
    let strE x = elt id x
    let ffwdE x f = ffwd x >>. element (f >> Ok) x
    let pIsSuccess = ffwdE "MessageType" ((<>) "Failed")

    let pHid = ffwdE "HotelId" int
    let pGid = elt int "GroupId"
    
    let parseDate s = 
        System.DateTime.Parse(s, Globalization.CultureInfo.InvariantCulture, Globalization.DateTimeStyles.AssumeUniversal).ToUniversalTime()

    let pDate = elt parseDate "Date"
    
    let pStartDate = ffwdE "Start" parseDate
    let pEndDate = elt parseDate "End"

    let pRoomCode = strE "RoomCode"
    let pRoomId = elt int "RoomId"
    
    let pRooms = 
        let zero = 
            fun _ -> Ok Set.empty
            |> Parser

        let nestedO name p = nested name p <|> zero  
        let pItem = pRoomCode .>>. pRoomId
        let pItems = set <!> (many (nested "item" pItem))
        ffwd "Rooms" >>. nestedO "Rooms" pItems
    
    let pRoom = Set.singleton <!> (ffwd "RoomCode" >>. pRoomCode .>>. pRoomId)

    let roomIdsP = 
        let defaultTo x y = 
            match y with _ when y |> Set.isEmpty -> x | _ -> y 

        defaultTo <!> pRoom <*> pRooms

    let logEntry (message:Stream) isSuccess hotelId groupId stamp startDate endDate rooms = 
        let ssnd (struct (_,x)) = x
        let roomIds = rooms |> Set.map ssnd
        let key = sprintf "%i/%i/%i"
        let keys = roomIds |> Set.map(key groupId hotelId)
        { Keys = keys
          Date = stamp
          Data = 
            message.Position <- 0L
            message }
    
    let rawXml (message:Stream) = 
        message.Position <- 0L
        let sr = new StreamReader(message)
        sr.ReadToEnd()

    try
        match run reader (logEntry message <!> pIsSuccess <*> pHid <*> pGid <*> pDate <*> pStartDate <*> pEndDate <*> roomIdsP) with
        | Ok logEntry -> Some logEntry
        | Error errors -> 
            let error = System.String.Join(System.Environment.NewLine, errors |> Array.ofList)
            printfn "[WARN]%s\r\nXml :\r\n%s" error (rawXml message)
            None
    with ex -> failwithf "Parsing failed with error %O\r\nXml:\r\n%s" ex (rawXml message)
```

I love this kind of little and composable declaration.
Parsing is now like a Lego game!

The associated tests : 
```
///Porcelain and plumbing
let convertFromString x = 
    use s = new MemoryStream()
    use sw = new StreamWriter(s)
    sw.Write(x:string)
    sw.Flush()
    s.Position <- 0L
    convert s

convertFromString """
<Transport><DestinationUri>dest</DestinationUri><SourceUri>source</SourceUri><ReplyUri /><MessageType>Update</MessageType><Message><HotelId>3116</HotelId><GroupId>9358</GroupId><Date>2017-02-20T16:19:12.4037606Z</Date><Request>Big Request</Request><Response>Big Response</Response><Start>2017-03-02</Start><End>2017-03-03</End>
    <RoomCode>26757</RoomCode>
    <RoomId>7066</RoomId>
    <Rooms /></Message></Transport>
"""

/// Do not consider the first Single RoomId if equal 0 and if there is no RoomIds
convertFromString """
<Transport><DestinationUri>dest</DestinationUri><SourceUri>source</SourceUri><ReplyUri /><MessageType>Update</MessageType><Message><HotelId>3116</HotelId><GroupId>9358</GroupId><Date>2017-02-20T16:19:12.4037606Z</Date><Request>Big Request</Request><Response>Big Response</Response><Start>2017-03-02</Start><End>2017-03-03</End>
    <RoomCode />
    <RoomId>0</RoomId>
    <Rooms>
        <item>
            <RoomCode>16239</RoomCode>
            <RoomId>3347</RoomId>
        </item>
        <item>
            <RoomCode>16241</RoomCode>
            <RoomId>3346</RoomId>
        </item>
   </Rooms></Message></Transport>
"""

/// Use the RoomId 0 = All Rooms when the is empty RoomIds
convertFromString """
<Transport><DestinationUri>dest</DestinationUri><SourceUri>source</SourceUri><ReplyUri /><MessageType>Update</MessageType><Message><HotelId>3116</HotelId><GroupId>9358</GroupId><Date>2017-02-20T16:19:12.4037606Z</Date><Request>Big Request</Request><Response>Big Response</Response><Start>2017-03-02</Start><End>2017-03-03</End>
    <RoomCode />
    <RoomId>0</RoomId>
    </Message></Transport>
"""
```

## The pros
- Easy to write
- Easy to maintain
- Hey it is composition, I can reuse parser and compose them to create new one!
- It fix a real issue (I start writing a code that parse xml with XmlReader with hard issue to solve in existing code)

## The cons
- It was my first attempt, so the first parser was harder to write but others were very easy to compose (The most part was to understand how works XmlReader and when the context has changed or not!)
- The use of operator and applicative separate the definition of the function from its execution. When you are debugging the run function you have to deal with a stack containing operator name (like GreaterGreaterEqual). Inlining the operator could help but it is more difficult to debug.
- For now, in my experience, I delivered multiple fix / multiple parsers composition and it is not a big deal to change my habits, finally it was easy to fix and maintain.

In [this post]({% post_url 2017-08-10-proto3-parser %}) I will introduce [FParsec](http://www.quanttec.com/fparsec/) in order to parse proto3 file.

## Links
- [Understanding Parser Combinators/Scott Wlaschin](https://fsharpforfunandprofit.com/posts/understanding-parser-combinators/)
- [Monad/Haskell wiki](https://wiki.haskell.org/Monad)
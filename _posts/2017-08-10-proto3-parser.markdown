---
layout: post
title:  "How to parse a proto3 message with FParsec"
date:   2017-08-10 13:13:00 +0200
categories: fsharp parser combinator fparsec proto3
---

I would like to try [FParsec](http://www.quanttec.com/fparsec/) fsharp parser to parse proto3 message.

I used this [FParsec](http://www.quanttec.com/fparsec/) [tutorial](http://www.quanttec.com/fparsec/tutorial.html#parsing-json) first in order to be more familiar with this library.

## Disclaimer
> This is not a production code
> proto3 specification is [here](https://developers.google.com/protocol-buffers/docs/proto3) if you want to continue with this script.

## The proto3 message sample
proto3 is a message format used by [protobuf-net](https://github.com/mgravell/protobuf-net) for example and could be export from type having some attributes indicating the mapping Id <-> properties and other metadata for the serialization configuration.

For more information about proto spec, read this [documentation](https://developers.google.com/protocol-buffers/docs/proto3).

To start, I will use this first sample and make step by step all necessary function to get the type informations in this structure : 

```
type Type = 
    | Scalar of string
    | Custom of string

type FieldSpec = 
    { Id : int
      Type: Type
      Name : string }

type Field = 
    | Required of FieldSpec
    | Optional of FieldSpec
    | Repeated of FieldSpec

type Message = 
    { Name : string
      Fields : Field list } 
```

Here is the sample that we will use all along this post : 

```
    message CalculateInfo {
    required string CalStarttime=1;
    optional string CalEndtime=2;
    required string Smiles=3;
    optional string CAS=4;
    optional string ChName=5;
    optional string EnName=6;
    required string Param=7;
    required bytes Result=8;
    required bool IsFinished=9;
    required bool IsFinished=9; }

    message GetAllCalulateResponse{
        required bool  isSuccessful = 1;
        required int32 Count=2;
        repeated CalculateInfo History=3; }
```

## Lets go !
The FParsec references (I will write with REPL style, so I will use a .fsx).

```
#I __SOURCE_DIRECTORY__
#r "packages/FParsec/lib/net40-client/FParsecCS.dll"
#r "packages/FParsec/lib/net40-client/FParsec.dll"
#r "System.Core.dll"
#r "System.dll"
#r "System.Drawing.dll"
#r "System.Numerics.dll"

open FParsec
```

## Step 1 : extracting the name of the message
In our case we want to extract the first message name : CalculateInfo

```
let msgSample = "message CalculateInfo {"

let pMessage = spaces >>. pstringCI "message" >>. spaces >>. (noneOf "{ " |> manyChars) 

run pMessage msgSample
```

The pMessage function will : 
- skip any spaces (including line break), 
- parse the world (case insensitive) "message", 
- skip any spaces 
- extract any chars (manyChars will convert chars to string) except (noneOf take a string and extract each char as blacklist) " " (whitespace) and "{".

The interactive will print this result : 

```
val it : ParserResult<string,unit> = Success: "CalculateInfo"
```

## Step 2 : Parse 1 field

```
let fieldSample = "required string CalStarttime=1"

let fieldSpec f t n i = f { FieldSpec.Id=i; Type=t; Name=n }
let pRequired = stringCIReturn "required" (fieldSpec Required)

run pRequired fieldSample
```

The pRequired will parse the work (case insensitive) required and return the fieldSpec with the parameter Required.
Let see the fieldSpec func : f parameter could be one of the choice defined into the type :

```
type Field = 
    | Required of FieldSpec
    | Optional of FieldSpec
    | Repeated of FieldSpec
```

When we look at the proto3 field definition we will parse in order : 

```
required string CalStarttime=1
```
1/ Field rule
2/ Type
3/ Name 
4/ Id

Look at the fieldSpec fun (defined in the previous code part) is exactly the adapter function that parsing after parsing will be applied to become in the end a Field instance.

Let just adapt the parser in order to parse the field rules optional or repeated : 

```
let ws = pchar ' ' |> manyChars |>> ignore

let fieldSample = "required string CalStarttime=1"

let fieldSpec f t n i = f { FieldSpec.Id=i; Type=t; Name=n }
let pRequired = stringCIReturn "required" (fieldSpec Required)
let pOptional = stringCIReturn "optional" (fieldSpec Optional)
let pRepeated = stringCIReturn "repeated" (fieldSpec Repeated)
let pField = ws >>. (pRequired <|> pOptional <|> pRepeated)

run pField "required string CalStarttime=1" //Success: <fun:pRequired@40-8>
run pField "repeated string CalStarttime=1" //Success: <fun:pRepeated@42-6>
run pField "optional string CalStarttime=1" //Success: <fun:pOptional@41-6>
```

We can see that the orElse operator (<|>) return the parser corresponding to the field rule (optional, required, repeated).

Now we have a function that take only the type, name and id.

Let's go to parse the type!

In proto, types are separated as scalar type and other types.

We will first parse the scalar type.
The code is a little but boring, the most interesting part is the parser function : 

```
module ScalarType = 
    let private mapping = 
        [ "double",  typeof<double>
          "float",   typeof<float>
          "int32",   typeof<int>
          "int64",   typeof<int64>
          "uint32",  typeof<uint32>
          "uint64",  typeof<uint64>
          "sint32",  typeof<int32>
          "sint64",  typeof<int64>
          "fixed32", typeof<uint32>
          "fixed64", typeof<uint64>
          "sfixed32",typeof<int32>
          "sfixed64",typeof<int64>
          "bool",    typeof<bool>
          "string",  typeof<string>
          "bytes",   typeof<byte[]> ]
        |> Map.ofList

    let parser = 
        mapping
        |> Map.toSeq
        |> Seq.map (fst >> pstring)
        |> Seq.fold (<|>) pzero
		
run ScalarType.parser "double" //ParserResult<string,unit> = Success: "double"
```

The parser function does : 
- take all scalar type as string into a seq.
- for each string, use a string parser
- use the orElse combinator and start with the pzero.

The pzero is really intesting because it will always return Error with empty error message. 
Combined with te orElse combinator it is like if you just force the second parser to be executed because the first parser will always failed.
This is exactly what we want when we would like to fold the sequence of parsers to one parser.

What is displayed if there was an error ?

```
run ScalarType.parser "hello" (*val it : ParserResult<string,unit> =
  Failure:
Error in Ln: 1 Col: 1
hello
^
Expecting: 'bool', 'bytes', 'double', 'fixed32', 'fixed64', 'float', 'int32',
'int64', 'sfixed32', 'sfixed64', 'sint32', 'sint64', 'string', 'uint32' or
'uint64'*)
```

Hey! not bad!

If it is not a scalar type, it is a custom type : 

```
let pWord = (<>) ' ' |> satisfy |> manyChars

let pType = 
    (ScalarType.parser |>> Scalar)
    <|> (ws >>. pWord |>> Custom)
	
run pType "hello" //ParserResult<Type,unit> = Success: Custom "hello"

```

There is only the name and identifier missed to build the full field parser, so here is the full code of the field parser : 
```
let (<*>) f x = f >>= fun f' -> x >>= fun x' -> preturn (f' x')

let pField = 
    let fieldSpec f t n i = f { Id=i; Type=t; Name=n }
    let pRequired = stringCIReturn "required" (fieldSpec Required)
    let pOptional = stringCIReturn "optional" (fieldSpec Optional)
    let pRepeated = stringCIReturn "repeated" (fieldSpec Repeated)
    let pField = ws >>. (pRequired <|> pOptional <|> pRepeated)
    
    //TODO: fallback scalar to custom type
    let pName = ws >>. (noneOf " =" |> manyChars)
    let pId = ws >>. pstring "=" >>. spaces >>. pint32

    pField <*> (ws >>. pType) <*> (ws >>. pName) <*> pId

run pField "required string CalStarttime=1" 
```

The result will be : 

```
val it : ParserResult<Field,unit> =
  Success: Required {Id = 1;
          Type = Scalar "string";
          Name = "CalStarttime";}
```

Now we have to build a fields parser that parse multiple field !

## Step 3 : Parsing n Fields
```
let pFields = spaces >>. sepEndBy pField (spaces >>. pchar ';' .>> spaces)

run pFields """required string CalStarttime=1;optional string CalEndtime=2"""
```

with more samples : 

```
run pFields """required string CalStarttime=1; optional string CalEndtime=2;"""
run pFields """required string CalStarttime=1 ; optional string CalEndtime=2;"""
run pFields """
required string CalStarttime=1 ; optional string CalEndtime=2;"""
run pFields """
 required string CalStarttime=1 ; optional string CalEndtime=2;"""
run pFields """
    required string CalStarttime=1 ; optional string CalEndtime=2;"""
run pFields """
    required string CalStarttime=1;
    optional string CalEndtime=2;"""
run pFields """
    required string CalStarttime=1;
    optional string CalEndtime=2;
    
    """
run pFields """


    required string CalStarttime=1;
    optional string CalEndtime=2;
    
    """
run pFields """


    required string CalStarttime=1;

    optional string CalEndtime=2;
    
    """
```

## Step final : the full parser
And now we have to combine pFields + pMessge together!

```
let message name fields = { Fields=fields; Name=name }

let pProtoMessage = pMessage |>> message .>> spaces .>> pchar '{' .>> spaces <*> pFields .>> spaces .>> pchar '}'

let pProtos = sepEndBy pProtoMessage spaces

run pProtos """
    message CalculateInfo {
    required string CalStarttime=1;
    optional string CalEndtime=2;
    required string Smiles=3;
    optional string CAS=4;
    optional string ChName=5;
    optional string EnName=6;
    required string Param=7;
    required bytes Result=8;
    required bool IsFinished=9;
    required bool IsFinished=9; }

    message GetAllCalulateResponse{
        required bool  isSuccessful = 1;
        required int32 Count=2;
        repeated CalculateInfo History=3; }
"""
```

And the final result of our parser!

```
val it : ParserResult<Field list,unit> =
  Success: [Required {Id = 1;
           Type = Scalar "string";
           Name = "CalStarttime";}; Optional {Id = 2;
                                              Type = Scalar "string";
                                              Name = "CalEndtime";}]
```

Finally, we see step by step how to deal with FParsec to build and compose parser together by builder a full proto message parser.
This parser could be composed with a enum parser. The enum parser could reuse the internal message parser, ...


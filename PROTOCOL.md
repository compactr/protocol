# Compactr Protocol
**for universal JSON serialization**

*Version* 2
*Revision* 1

## Table of content

---

## Glossary

### Payload

- The Object that is to be serialized.

ex:

```
{
  id: 2,
  first_name: 'John',
  last_name: 'Smith'
}
```

### Schema 

- The Object containing information about the payload model.

ex:

``` 
{
  id: 'number',
  first_name: 'string',
  last_name: 'string'
}
```

### Serializing

Serialization refers to the concept of translating data structures into an easily storable, or portable format. In most cases, this equates to interpreting application Objects as a string of characters or a binary buffer. 

Serialization happens constently in the context of web transactions. For your JSON Rest API to be able to send your Object over the line to the client, it needs to be transformed into something that HTTP, a text-based protocol, can easily chunk in packets. It this case, JSON stringification is used for serializing your payload.

Some other web protocols, like web sockets or HTTP2 are binary-ready protocols. Meaning that they allow the passing of binary buffers over the line. The browser gives us a naive but effective representation of buffer objects as an array of UINT8 values.

JSON serialization has many advantages. For one, it is included by default in may languages and runtimes - often making it a speedy and efficient process. Secondly, it creates readable, self-explanatory payloads. Decoding requires no additionnal components. The downside is that the resulting string is pretty large and in the case of realtime web applications, repeating the [schema](#lexicon) and the JSON markup elements wastes precious bytes.

Schema-based encoding aims to remove markup and unesserary [schema](#lexicon) key names, which results in *much* smaller `Buffer` outputs. Since the expected key names and data types are specified in a Schema that is shared between both parties, there is no ambiguity about the received data and less repetition. It has disadvantages too. The biggest one being the handling schema files, which we will explore in more details in the next section, and the other being that payloads are hardly inspectable.


### Schemas

Schema-based serialization, as the name suggests, requires that payloads be serialized against a schema, a data model of sorts. It tells the encoder what the expected properties are for that Object, their data types and other metadata info, like default values. 

If a property is to be encoded and it is not present in the Schema, it will be skipped.

Schema files need to be in sync between all parties, if there are discrepencies, data will get corrupted in the process.


## Preface

Before we dig into the subject, we need to take a look at existing solutions first.


### Protocol Buffers

One of the most used Schema-based serialization protocol right now is Google's Protocol Buffers. It is very reputable and and higly performant. My only critique of it is that it introduces a new type of asset that your application must care for: .proto files. These files contain the Schemas that Protocol Buffer uses. They are written in their own markdown and have data types that don't mean a lot for say, Web developpers. 

If you wanted to inspect or instrument these Schemas, you would have to build your own tools to parse them and load them into comprehensible and workable Objects.


Compactr is a serialization protocol that aims to solve this problem. Schemas being actual Data structures inside your app. Data types are also simplified to better suite web developement.

Data is structured like a JSON object would. It supports full recursiveness, lookups, compression with 100% consistency.


## Data types

Data types have been simplified in Compactr and try to mimic core Javscript types:

- Boolean
- Number
- Object
- String

In arrays:

- Array of Booleans
- Array of Numbers
- Array of Objects
- Array of Strings

When selecting number as a field data type, the encoder will automatically determine the best stategy for the value passed. It also offers the possibility to specify the strategy yourself:

- INT8
- INT16
- INT32
- DOUBLE

And also in arrays:

- Array of INT8
- Array of INT16
- Array of INT32
- Array of DOUBLES

Note that for decimals, only Double is a valid stategy, and most closely matches Javascript's Float64 definition.


## Mapping


# Compactr Protocol
**for universal serialization**

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

---

## Preface

Before we dig into the subject, we need to take a look at existing solutions first.


### Protocol Buffers

One of the most used Schema-based serialization protocol right now is Google's Protocol Buffers. It is very reputable and and higly performant. My only critique of it is that it introduces a new type of asset that your application must care for: .proto files. These files contain the Schemas that Protocol Buffer uses. They are written in their own markdown and have data types that don't mean a lot for say, Web developpers. 

If you wanted to inspect or instrument these Schemas, you would have to build your own tools to parse them and load them into comprehensible and workable Objects.


Compactr is a serialization protocol that aims to solve this problem. Schemas being actual Data structures inside your app. Data types are also simplified to better suite web developement.

Data is structured like a JSON object would. It supports full recursiveness, lookups, sending maps and content separately for even shorter payloads.

---

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

---

## Byte types

The bytes of a compactr payload are split into these categories

| Key | Description |
| --- | --- |
| KEY_COUNT | Holds the number of keys included in the payload |
| KEY_INDEX | Tells which Schema property is stored |
| LENGTH | Indicates the byte size of the data |
| DATA | The binary-encoded data |


LENGTH value, a UINT8 byte by default, can either be dynamic or static, depending on the data type. INT8, for example, will always be of length 01 (for one byte). Strings, can be of various lengths. When the length of your strings can exceed the max value of UINT8 (255 characters), it's important to change the configuration for that field in the encoding. This change has to be done by all parties to maintain data integrity.

It's also important to keep endian direction across parties.

---

## Sections

Compactr payloads are made up of two sections

| Section | Description |
| --- | --- |
|MAP | Includes KEY_COUNT, KEY_INDEX and LENGTH bytes. Helps to create a logical map of the data stored in the buffer |
|CONTENT | The actual binary-encoded DATA bytes, all concatenated |

---

## Mapping

A typical Compactr payload is built as follow (Shallow Object):

`[KEY_COUNT][KEY_INDEX A][LENGTH A][KEY_INDEX B][LENGTH B][DATA A][DATA B]`

Here's an example of how this translates for a Javascript Object:

```
// Data
{
    name: 'greg',
    age: 32
}

// Schema
{
    name: 'string',
    age: 'number'
}
```

```
-- MAP
  KEY_COUNT = 02

  (name) KEY_INDEX = 00
  (greg) LENGTH = 04

  (age) KEY_INDEX = 01
  (32) LENGTH = 01 // Gets cast as INT8

-- CONTENT
  (greg) DATA[0] = 67,72,65,67
  (32) DATA[1] = 32


-- Result:
<Buffer 02, 00, 04, 01, 01, 67, 72, 65, 67, 32>
```

If we compare the size from our compactr payload to what we would have had through JSON serialization:

- Compactr: 10 Bytes
- JSON: 24 Bytes

This also works with nested data, as deep as you need it! Maps are all aggregated in the first section. 


```
// Data
{
    users: [
    	{ name: 'greg', age: 32 },
    	{ name: 'steve', age: 26 },
    	{ name: 'patrick', age: 41 }
    ] 
}

// Schemas 
{
    name: 'string',
    age: 'number'
}

{
	users: {
	    type: 'schema_array',
	    items: User
	}
}
```

```
-- MAP
  KEY_COUNT = 01

  (users) KEY_INDEX = 00
  (users) LENGTH = 19

  (users[0]) KEY_COUNT = 02

  (users[0].name) KEY_INDEX = 00
  (users[0] greg) LENGTH = 04

  (users[0].age) KEY_INDEX = 01
  (users[0] 32) LENGTH = 01

  (users[1]) KEY_COUNT = 02

  (users[1].name) KEY_INDEX = 00
  (users[1] steve) LENGTH = 05

  (users[1].age) KEY_INDEX = 01
  (users[1] 26) LENGTH = 01

  (users[2]) KEY_COUNT = 02

  (users[2].name) KEY_INDEX = 00
  (users[2] patrick) LENGTH = 07

  (users[2].age) KEY_INDEX = 01
  (users[2] 41) LENGTH = 01

-- CONTENT
  (greg) DATA[0] = 67,72,65,67
  (32) DATA[1] = 32
  (steve) DATA[2] = 73,74,65,76,65
  (26) DATA[3] = 26
  (patrick) DATA[4] = 70,61,74,72,69,63,6b
  (41) DATA[5] = 41


<Buffer 01, 00, 19, 02, 00, 04, 01, 01, 02, 00, 05, 01, 01, 02, 00, 07, 01, 01, 67, 72, 65, 67, 32, 73, 74, 65, 76, 65, 26, 70, 61, 74, 72, 69, 63, 6b, 41>
```

- Compactr: 37 Bytes
- JSON: 90 Bytes

---

## Streaming

As mentionned above, one of Compactr's ability is also to be able to stream only the CONTENT section of the payload based on one map. Which works wonders if you just need to send booleans or numeric coordinates, say you're building a game, and only care about the x and y position of a character. 

This can be achieved like so:

```
const User = {
	x: 'int16',
	y: 'int16'
}

// Encode your data as usual
const UserMap = Compactr.map(User);

// Encode some data
let payload = UserMap.encode({ x: 12, y: -200 });

// Only prints out the content
// <Buffer 12, 00, 38, 00>
```

- Compactr: 4 Bytes
- JSON: 17 Bytes

This doesn't work as well for dynamic-length data:

```
// Make sure to include a config object to allocate space for dynamic length fields (strings, arrays, objects)
const UserMap = Compactr.map(User, {
	name: {
		size: 8	// Data will be truncated to 8 bytes
	}
});

// Encode some data
let payload = UserMap.encode({ name: 'greg', age: 32});

// Only prints out the content
// <Buffer 67, 72, 65, 67, 00, 00, 00, 00, 32>
```

- Compactr: 9 Bytes
- JSON: 24 Bytes

You may shave off a couple bytes, but you risk either truncating or over-allocating.

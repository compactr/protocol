# Compactr Protocol
**for universal serialization**

*Version* 2
*Revision* 1

## Table of Contents

---

## Glossary

### Payload

- The Object to be serialized.

ex:

```
{
  id: 2,
  first_name: 'John',
  last_name: 'Smith'
}
```

### Schema

- The Object containing the description of the payload model.

ex:

```
{
  id: 'number',
  first_name: 'string',
  last_name: 'string'
}
```

### Serialization

Serialization refers to the translation of data structures into an easily storable or portable format. In most cases, this equates to interpreting application Objects as a string of characters or a binary buffer. There are many ways to serialize data structures:

__JSON Stringification:__ Before a JSON REST API is able to send an Object over the line to the client, it needs to be translated into something that HTTP can easily chunk into packets. Since HTTP is a text-based protocol, the Object is translated into a string of characters, or *JSON stringification*.

Pros:

- Included by default in many languages and runtimes - often making serialization/de-serialization rapid.
- Creates human-readable, self-describing payloads.
- Decoding requires no additional components.

Cons:

- The resulting string is relatively large
- Bytes are wasted repeating the [schema](#lexicon) and JSON markup in every transfer
- In the case of realtime web applications, bytes are too precious to waste

__Binary-ready protocols:__ Some web protocols, like websockets or HTTP2, are not text-based protocols. They allow passing binary buffers over the line. The browser gives us a naïve but effective representation of buffer objects as an array of UINT8 values.

__Schema-based serialization__ aims to remove markup and unnecessary [schema](#lexicon) key names, which results in *much* smaller `Buffer` outputs. The expected key names and data types are specified in a Schema, which is shared between both parties beforehand.

Pros:

- There is no ambiguity about the received data.
- There is less repetition (key names and data types are transmitted once).

Cons:

- Handling schema files is a problem we will explore in more details in the next section.
- The payloads are difficult to inspect.

### Encoders, Decoders, Encoding, and Decoding

- The implementation inside your application which is responsible for serializing is called the *encoder*.
- The implementation inside your application which is responsible for de-serializing is called the *decoder*.
- The acts of serializing and de-serializing are referred to as *encoding* and *decoding*, respectively.

### Schemas

Schema-based serialization requires that payloads be serialized against a schema - a data model of sorts. It tells the encoder what the expected properties are for that Object, their data types and other metadata info, like default values.

If a property is to be encoded and it is not present in the Schema, it is excluded from serialization.

Schema files need to be synchronized between all parties. If there are discrepancies, data becomes corrupted during encoding/decoding.

---

## Preface

Before we dig into the subject, we need to take a look at existing solutions first.


### Protocol Buffers

One of the most used Schema-based serialization protocols right now is Google's Protocol Buffers. It is regarded by the industry as being reputable and highly performant. Here is my criticism:

- It introduces a new type of asset that your application must care for: `.proto` files.
- `.proto` files are written in their own special markdown syntax
- Protocol Buffers support data types that are not easily mapped to those used by web developers
- If you wanted to inspect or instrument these Schemas, you would have to build your own tools to parse them and load them into comprehensible and workable Objects.

Compactr is a serialization protocol that aims to solve this problem:

- Schemas are actual Data structures inside your app.
- Data types are simplified to better suit web development.
- Data is structured like a JSON object.
- It supports full recursion and lookups.
- It supports sending maps and content separately for even shorter payloads.

---

## Data types

Data types have been simplified in Compactr and try to mimic core Javascript types:

- Boolean
- Number
- Object
- String

In arrays:

- Array of Booleans
- Array of Numbers
- Array of Objects
- Array of Strings

When selecting `Number` as a field data type, the encoder will automatically determine the best strategy for the value passed. It also offers the possibility to specify the strategy yourself:

- INT8
- INT16
- INT32
- DOUBLE

And also in arrays:

- Array of INT8
- Array of INT16
- Array of INT32
- Array of DOUBLES

Note that for decimals, only `Double` is a valid strategy, and most closely matches Javascript's `Float64` definition.

---

## Byte types

The bytes of a Compactr payload are split into these categories:

| Key | Description |
| --- | --- |
| KEY_COUNT | Indicates the number of keys included in the payload |
| KEY_INDEX | Indicates which Schema property is stored |
| LENGTH | Indicates the byte size of the data |
| DATA | The binary-encoded data |


`LENGTH` value, a `UINT8` byte by default, can either be dynamic or static, depending on the data type. `INT8`, for example, will always be of length `01` (for one byte). Strings, can be of various lengths. When the length of your strings can exceed the max value of `UINT8` (255 characters), it's important to change the configuration for that field in the encoding. This change has to be done by all parties to maintain data integrity.

It's also important to keep endian direction across parties.

---

## Sections

Compactr payloads are made up of two sections

| Section | Description |
| --- | --- |
|MAP | Includes `KEY_COUNT`, `KEY_INDEX` and `LENGTH` bytes. Helps to create a logical map of the data stored in the buffer |
|CONTENT | The actual binary-encoded `DATA` bytes, all concatenated |

---

## Mapping

A typical Compactr payload is built as follows (Shallow Object):

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

If we compare the size of our Compactr payload to what we would have had through JSON stringification:

- Compactr: 10 Bytes
- JSON stringification: 24 Bytes

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
  users: {
    type: 'schema_array',
    items: {
      name: 'string',
      age: 'number'
    }
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

Say you're building a game, and you only care about the x and y position of a character. One of Compactr's abilities is to stream only the `CONTENT` section of the payload based on one map. This works wonders if you need to send booleans or numeric coordinates repetitively.

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

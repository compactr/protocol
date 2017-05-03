# Compactr Protocol
**for universal serialization**

*Version* 2
*Revision* 2

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

One of the most used Schema-based serialization protocols right now is Google's Protocol Buffers. It is regarded by the industry as being reputable and highly performant. Here our my criticism:

- Loss of tooling and social costs associated with the introduced `.proto` files.
- Protocol Buffers support data types and nested structures are not easily mapped to those used by web developers used to working with plain Objects (POJOs).
- Heavily engineered for extra-resilience and edge-case support limits performance and increases payload sizes.

Compactr is a serialization protocol that aims to solve these problem:

- Schemas are plain Object structures coded in your app's language or portable JSON.
- Data types and nested structures are simplified to better mimic JSON.
- It supports full recursion and lookups.
- It supports encoding payload headers and contents separately for even shorter payloads.
- Validation and coersing options are available- not enforced.

---

## Schema

A simple schema is an object that has some schema-keys defined.

### Schema-key

A schema-key is an object containing the options for the encoding and decoding of a given property.

Possible options include `type`, `count`, `size`, `schema` and `items`.
Where `schema`applies to *object* types and `items` to *array* types.

### Data types

The type of the data for the property. Here's the list of exposed data types.
It is important to note that *null* or *undefined* values are ignored by compactr. 

| Name | Description |
| --- | --- |
| boolean | A boolean value |
| number | ieee754 finite number value, does not support NaN or Infinity |
| int8 | 8Bit signed integer |
| int16 | 16Bit signed integer |
| int32 | 32Bit signed integer |
| double | ieee754 finite number value, does not support NaN or Infinity |
| string | 16bit character string |
| char8 | 8bit character string |
| char16 | 16bit character string |
| char32 | 32bit character string |
| object | recursive compactr schema processing (must be a valid compactr schema) |
| array | (TBD) |

### Count

The count property indicates how many bytes are required to display the content's size.
The byte count maps to the signed integers specs in data types.
For example, a count value of 1 supports data lengths of 0 to 127 bytes.
The default value for the count property is 1.

### Size

The size property overrides allocation size for the data. This is needed for strings, objects and other dynamic-size data types when streaming the header and content buffers separatly.

---

## Buffer structure

The resulting buffer is separated in two sections. The header and the content.

### Header

This sections includes the mapping information of the payload.

**Layout**

`[KEY_COUNT] each(key) { [KEY_ID][COUNT][LENGTH] }`

- KEY_COUNT: A single byte of type unsigned INT8 to display the number of keys in the payload
- KEY_ID: A single byte of type unsigned INT8 to indentify a given key in your schema
- LENGTH: A variable number of bytes that displays the byte length of the data for this key

**Example**

Schema: 
```
{
  a: { type: 'int8' },
  b: { type: 'int8' },
  c: { type: 'int8' }
}
```

Data: `{ a: 0, b: 1, c: 2 }`

- KEY_COUNT: 3 keys in the data, all matching keys in the schema -> 03
- KEY_ID(a): The first key in the schema -> 00
- LENGTH(a): No overrides were made to the count property, so this section is 1 byte (default) and the length of the data is -> 01
...

Header:

`<03, 00, 01, 01, 01, 02, 01>` 

### Content

The content section is simply the list of encoded data byte segments chained one after the other.

**Layout**

`each(data) { [DATA] }`

- DATA: The encoded data bytes for a given property

Schema: 
```
{
  a: { type: 'int8' },
  b: { type: 'int8' },
  c: { type: 'int8' }
}
```

Data: `{ a: 0, b: 1, c: 2 }`

- DATA(a): Encoded byte(s) -> 01
...

Content:

`<00, 01, 02>` 


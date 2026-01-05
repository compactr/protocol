# Compactr Format Specification v1.0

Authors:
- Frederic Charette <fredericcharette@gmail.com>
  
Date published: 2026-01-01

Last update: 2026-01-02

Keywords:
- serialization
- open-api

---

## Abstract

This document specifies the compactr format, a schema-based serialization protocol that reuses existing [[OAS]](#6-References)OpenAPI specifications as schemas.

## Status

The specification is Stable as of this publication's release. 

## Table of Contents

- [1. Background](#1-Background)

- [2. Design decisions](#2-Design-decisions)

  - [2.1 Byte-order](#21-Byte-order)

  - [2.2 Key limits](#22-Key-limits)

  - [2.3 Size limits](#23-Size-limits)

  - [2.4 Schema properties and Encoding order](#24-Schema-properties-and-Encoding-order)

  - [2.5 Versioning](#25-Versioning)

- [3. Schemas](#3-Schemas)

  - [3.1 Schema Source](#31-Schema-source)
 
  - [3.2 Required vs Optional Properties](#32-Required-vs-Optional-Properties)
 
  - [3.3 Walkthrough properties](#33-Walkthrough-properties)

- [4. Encoding](#4-Encoding)

  - [4.1 Variants](#41-Variants)

  - [4.2 Primitive types](#42-Primitive-types)
 
    - [4.2.1 Arrays](#421-arrays)
    
    - [4.2.2 Boolean](#422-boolean)
    
    - [4.2.3 Integers](#423-integers)
    
    - [4.2.4 Numbers](#424-numbers)
    
    - [4.2.5 Objects](#425-objects)
    
    - [4.2.6 Strings](#426-strings)
   
  - [4.3 Special formats](#43-special-formats)

    - [4.3.1 Binary](#431-binary)
   
    - [4.3.2 Date and DateTime](#432-date-and-datetime)
   
    - [4.3.3 IPV4 and IPV6](#433-ipv4-and-ipv6)
   
    - [4.3.4 UUID](#434-uuid)

- [5. Security considerations](#5-Security-considerations)

- [6. References](#6-References)

---

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [[RFC2119]](#6-References) [[RFC8174]](#6-References) when, and only when, they appear in all capitals, as shown here.

## 1. Background

Serialization in the context of Web APIs refers to the process of converting data structures into a format that can be easily transmitted over a network, typically in text-based formats (e.g., JSON, XML), or binary formats (e.g., files, [Protobuf](https://protobuf.dev/)), so that they can be understood and reconstructed by other systems.

A schema-based serialization approach enforces a predefined structure for data, ensuring consistency and validation, whereas a schema-less approach allows for more flexible and dynamic data representation, with fewer constraints on how data is organized.

Schema-based serialization protocols generally yield much smaller outputs, which is desirable to limit bandwidth and costs. The caveat to schema-based serialization is the cost of creating and maintaining schemas across multiple systems.

The initial concept for the compactr protocol was drafted in [2016](https://www.npmjs.com/package/compactr/v/0.0.1) with the goal of creating a schema-based serialization protocol that outputs minimal binary while using first party markdown or code structures as schemas.

While functional, the early versions would still require the knowledge of writing "compactr-style" schemas as Javascript Objects or JSON and limited adoption for languages outside of Javascript. As of compactr.js 3.0, release in 2025, the protocol moved to adopt OpenAPI 3.x as the base format for compactr schemas.

---

## 2. Design decisions

The primary objectives of the Compactr protocols are:

- First-party schema definitions, using [[OAS]](#6-References)OpenAPI specifications as base schemas.
- Optimized binary output
- Compatibility across runtimes
- Type safety

In order to meet these objectives, some key design decisions were made:

### 2.1 Byte-order

Compactr binary MUST follow Network Byte Order (NBO) big-endian format.

### 2.2 Key limits

Indices SHALL be assigned for properties and stored as an unsigned 8-bit integer. Thus limiting the number of properties per object to 255.

### 2.3 Size limits

Some primitive types (e.g., `Boolean`) have fixed sizes and therefore MUST NOT encode size bytes, while others (e.g.,: `String`) have variable sizes and MUST include between one and four size bytes.

Size bytes are represented by unsigned integers of varying sizes, which are described in the [primitives](#4-2-primitive-types) section of this document.


### 2.4 Schema properties and Encoding order

To maintain consistency across systems, the field index for each schema property is based on its alphabetical order, starting from 1.

The sorting function MUST be based on the numerical order of Unicode (UTF-16) character code values of the property names.

Example:

```json

{
  "type": "object",
  "properties": {
    "a": { "type": "boolean" },
    "c": { "type": "boolean" },
    "b": { "type": "boolean" }
  }
}

```
Will attribute index 0 to field `a`, index 1 to field `b` and 2 for `c`. Implementations of this protocol MUST follow this sorting rule to maintain consistency, even if properties are listed in differring orders across systems.

Encoding of values to generate the binary output SHOULD simply follow the order in which the properties are listed in the structure or object.

For example, serializing `{ c: true, a: true, b: true }` with the previous schema will output: `0x03 0x01 0x01 0x01 0x02 0x01`. 


## 2.5 Unsupported features

### 2.5.1 References

`$ref` references are supported, with constraints which MUST be enforced in client implementations:

- Circular references MUST be detected.
- Recursive schemas MAY be supported but implementations SHOULD impose depth limits.
- External $ref targets (remote URLs) MAY be supported but MUST be resolved prior to encoding.

### 2.5.2 Versioning

Compactr binaries do not include version flags and the protocol does not include versioning mechanisms.

Client implementations MAY elect to include integrity or versioning checks provided that the final encoded binary remains compatible with the Compactr protocol.

---

## 3. Schemas

### 3.1 Schema Source

Compactr schemas are derived from OpenAPI 3.0+ Schema Objects, as defined in [[OAS]](https://spec.openapis.org/oas/v3.1.2.html)OpenAPI specifications.

Only the following schema keywords are normative for Compactr encoding:

- `type`
- `format`
- `properties`
- `required`
- `items`
- `oneOf`
- `anyOf`
- `allOf`
- `nullable`
- `$ref`

All other OpenAPI keywords (e.g., description, example, deprecated) are ignored for encoding purposes.

### 3.2 Required vs Optional Properties

Properties listed in required MUST be present during encoding. Missing required properties MUST throw an encoding error. 

Optional properties MAY be omitted.

Missing optional properties are not encoded and do not occupy space.

Decoders MUST treat omitted optional properties as undefined (or language equivalent).

### 3.3 Walkthrough properties

Compactr walks through composition keywords `$ref`, `schema` `oneOf`, `allOf`, `anyOf` and only creates internal models for primitives.

---

## 4. Encoding

Properties are encoded with the matching schema index first (`i`), then an optional variant byte (`v`), optional size byte(s) (`s`), then the encoded value (`d`).

`[i][v?][s?...][d...]`

### 4.1 Variants

Encoded fields which have the `nullable` schema property and a `null` value have an extra byte that indicates the variant.

- `0x00` For null values
- `0x01` For non-null values

If the `nullable` property is not present in the schema, the variant byte is not encoded and `null` values are not encoded.

Fields with multiple definitions, as described in the schema with the `oneOf` or `anyOf` keywords use the variant byte to indicate which definition to use, starting with `0x01` for the first definition, and incrementing by `0x01` for each subsequent one.


### 4.2 Primitive types

Types are based on JSON Schema Validation Specification Draft 2020-12: `array`, `boolean`, `integer`, `number`, `object` or `string`.

#### 4.2.1 Array

Arrays MUST include an unsigned 32-bit integer to represent the whole size of the array. Individual elements are treated sequentially as their primitives defined in the schema.

Example:

```
// Schema
{
  type: 'object',
  properties: {
    foo: {
      type: 'array',
      items: { type: 'string' }
    }
  }
}

// Data
{ foo: [ 'hello', 'bye', 'bye' ] }
```

Results in this buffer: `0x01 0x00 0x00 0x0e 0x05 0x68 0x65 0x6c 0x6c 0x6f 0x03 0x62 0x79 0x65 0x03 0x62 0x79 0x65`.

## 4.2.2 Boolean

Fixed size of 1 byte, either 0x00 for false or 0x01 for true.

#### 4.2.3 Integers

Variable size based on the `format` attribute defined in the schema. Size byte SHOULD NOT be encoded. Decoding should take in account the `format` attribute to determine the size.

- `(null, undefined or language equivalent)`: unsigned 32-bit integer
- `int32`: unsigned 32-bit integer
- `int64`: unsigned 64-bit integer

#### 4.2.4 Numbers

Variable size based on the `format` attribute defined in the schema. Size byte SHOULD NOT be encoded. Decoding should take in account the `format` attribute to determine the size.

All floating-point arithmetic MUST adhere to [[IEEE 754-2019]](#6-References) 

- `(null, undefined or language equivalent)`: 32-bit floating point
- `float`: 32-bit floating point
- `double`: 64-bit floating point

#### 4.2.5 Objects

Objects are encoded recursively using the same scheme: `[i][v?][s?...][d...]`.

#### 4.2.6 Strings

Strings are encoded as UTF-8 Multi-byte Unicode characters. Most languages provide a UTF-8 encoding utility, which SHOULD be used to determine the size and generate the bytes to be appended.

### 4.3 Special formats

Compactr supports encoding of special formats to improve efficiency. Additional special encoding formats MAY be added.

#### 4.3.1 Binary

Variable length `string` format with 32-bit size bytes.

Buffers and UInt8Arrays MAY be encoded as-is, while `strings` MUST be Base64 encoded.

#### 4.3.2 Date and DateTime

Fixed length `string` formats with no size bytes.

`date` is represented as `[uint32][uint8][uint8]` to encode YYYY-MM-DD values.

`date-time` is represented as `[uint32][uint8][uint8][uint8][uint8][uint8][uint32]` to encode YYYY-MM-DDTHH:mm:ss.sssZ date strings with UTC time.

Values MUST be reconstructed as such by the decoder to fit [[ISO 8601]](#6-References) extended date.

Implementations SHOULD validate that the input string is a valid date string and SHOULD set time bytes to 0 if not explicitly set.

#### 4.3.3 IPV4 and IPV6

Fixed length `string` formats with no size bytes.

`ipv4` is represented as [uint8][uint8][uint8][uint8] and must be decoded to match [[RFC791]](#6-References) IPV4 format.

`ipv6` is represented as [uint32][uint32][uint32][uint32] and must be decoded to match [[RFC8200]](#6-References) IPV6 format.

#### 4.3.4 UUID

Fixed sized 16 bytes using raw UUID bytes (network-order) and must be decoded as standard [[RFC9562]](#6-References) UUID.

---

## 5. Security considerations

Implementations MUST guard against:

- Circular schemas
- Malformed UTF-8
- Integer overflow
- Maximum object keys
- Maximum object recursion depth
- Array total byte size
- Ensure schema formats match the appropriate schema type

Compactr does not provide encryption, authentication, or integrity guarantees.

---

## 6. References

- [OAS] OpenAPI Specification v3.1.2. The Linux foundation (2025). <https://spec.openapis.org/oas/v3.1.2.html>
- [RFC791] Internet protocol. DARPA Internet Program Protocol Specification. (1981). <https://www.rfc-editor.org/rfc/rfc791>
- [RFC2119] Key words for use in RFCs to Indicate Requirement Levels. S. Bradner. IETF. (1997). <https://www.rfc-editor.org/rfc/rfc2119>
- [RFC8174] Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words. B. Leiba. IETF. (2017). <https://www.rfc-editor.org/rfc/rfc8174>
- [RFC8200] Internet Protocol, Version 6 (IPv6) Specification. S. Deering. IETF. (2017). <https://datatracker.ietf.org/doc/html/rfc8200>
- [RFC9562] Universally Unique IDentifiers (UUIDs). K. Davis. IETF. (2024) <https://www.rfc-editor.org/rfc/rfc9562.html>
- [IEEE 754-2019] IEEE 754-2019: IEEE Standard for Floating-Point Arithmetic. Institute of Electrical and Electronic Engineers. (2019).

---

Licensed under Apache 2.0, 2026, Compactr, Frederic Charette

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

This document specifies the compactr format, a schema-based serialization protocol that reuses existing [[OAS]](https://spec.openapis.org/oas/v3.1.2.html)OpenAPI specifications as schemas.

## Status

The specification is Stable as of this publication's release. 

## Table of Contents

- [1. Background](#1-Background)

- [2. Design decisions](#2-Design-decisions)

  - [2.1 Byte-order](#2-1-Byte-order)

  - [2.2 Key limits](#2-2-Key-limits)

  - [2.3 Size limits](#2-3-Size-limits)

  - [2.4 Schema properties and Encoding order](#2-4-Schema-properties-and-Encoding-order)

  - [2.5 Versioning](#2-5-Versioning)

- [3. Schemas](#3-Schemas)

  - [3.1 Schema Source](#3-1-Schema-source)
 
  - [3.2 Required vs Optional Properties](#3-2-Required-vs-Optional-Properties)
 
  - [3.3 Walkthrough properties](#3-3-Walkthrough-properties)

- [4. Encoding](#4-Encoding)

  - [4.1 Variants](#4-1-Variants)

  - [4.2 Primitive types](#4-2-Primitive-types)
 
  - 

- [5. Complex schemas](#5-Complex-schemas)

- [6. Variants](#6-Variants)

- [7. Implementation considerations](#7-Implementation-considerations)

- [8. Security considerations](#8-Security-considerations)

- [9. References](#9-References)

---

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [[BCP 14]](https://tools.ietf.org/html/bcp14) [[RFC2119]](https://spec.openapis.org/oas/v3.1.2.html#bib-rfc2119) [[RFC8174]](https://spec.openapis.org/oas/v3.1.2.html#bib-rfc8174) when, and only when, they appear in all capitals, as shown here.

## 1. Background

Serialization in the context of Web APIs refers to the process of converting data structures into a format that can be easily transmitted over a network, typically in text-based formats (e.g., JSON, XML), or binary formats (e.g., files, [Protobuf](https://protobuf.dev/)), so that they can be understood and reconstructed by other systems.

A schema-based serialization approach enforces a predefined structure for data, ensuring consistency and validation, whereas a schema-less approach allows for more flexible and dynamic data representation, with fewer constraints on how data is organized.

Schema-based serialization protocols generally yield much smaller outputs, which is desirable to limit bandwidth and costs. The caveat to schema-based serialization is the cost of creating and maintaining schemas across multiple systems.

The initial concept for the compactr protocol was drafted in [2016](https://www.npmjs.com/package/compactr/v/0.0.1) with the goal of creating a schema-based serialization protocol that outputs minimal binary while using first party markdown or code structures as schemas.

While functional, the early versions would still require the knowledge of writing "compactr-style" schemas as Javascript Objects or JSON and limited adoption for languages outside of Javascript. As of compactr.js 3.0, release in 2025, the protocol moved to adopt OpenAPI 3.x as the base format for compactr schemas.

---

## 2. Design decisions

The primary objectives of the Compactr protocols are:

- First-party schema definitions, using [[OAS]](https://spec.openapis.org/oas/v3.1.2.html)OpenAPI specifications as base schemas.
- Optimized binary output
- Compatibility across runtimes
- Type safety

In order to meet these objectives, some key design decisions were made:

### 2.1 Byte-order

Compactr binary MUST follow Network Byte Order (NBO) big-endian format.

### 2.2 Key limits

Indices SHALL be assigned for properties and stored as an unsigned 8-bit integer. Thus limiting the number of properties per object to 255.

### 2.3 Size limits

Some primitive types (e.g., `Boolean`) have fixed sizes and therefore MUST NOT encode size bytes, while others (e.g.,: `String`) have dynamic sizes and MUST include between one and four size bytes.

Size bytes are represented by unsigned integers of varying sizes, which are described in the [primitives](#4-primitives) section of this document.


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

#### 4.2.2 Boolean

Fixed size of 1 byte, either 0x00 for false or 0x01 for true.

#### 4.2.3 Integers

Variable size based on the `format` attribute defined in the schema. Size byte SHOULD NOT be encoded. Decoding should take in account the `format` attribute to determine the size.

- `(null, undefined or language equivalent)`: unsigned 32-bit integer
- `int32`: unsigned 32-bit integer
- `int64`: unsigned 64-bit integer

#### 4.2.4 Numbers

Variable size based on the `format` attribute defined in the schema. Size byte SHOULD NOT be encoded. Decoding should take in account the `format` attribute to determine the size.

- `(null, undefined or language equivalent)`: 32-bit floating point
- `float`: 32-bit floating point
- `double`: 64-bit floating point

#### 4.2.5 Objects



#### 4.2.6 Strings

Strings are encoded as UTF-8 Multi-byte Unicode characters. Most languages provide a UTF-8 encoding utility, which SHOULD be used to determine the size and generate the bytes to be appended.


### 4.3 Special formats

Compactr supports encoding of special formats to improve efficiency. Additional special encoding formats MAY be added.

#### 4.3.1 Binary


### 4.

Mapped from OpenAPI:

type: string
format: binary


Maximum length: 4,294,967,295 bytes.

4.7 UUID

Size: 16 bytes

Encoding: Raw UUID bytes, network order

Mapped from:

type: string
format: uuid

4.8 Date

Size: 4 bytes

Encoding: Signed int32, days since Unix epoch (UTC)

Mapped from:

type: string
format: date

4.9 DateTime

Size: 9 bytes

Encoding: Component-based UTC timestamp

Mapped from:

type: string
format: date-time

## 5. Complex schemas

5.1 Object Encoding

Objects are encoded as:

[num_properties][property...]


Where each property is:

[index][size][value]


num_properties: u8

index: u8 (alphabetical index)

size: variable (see below)

5.3 Arrays

Arrays are encoded as a sequence of elements, without a global count:

[element_size][element_value]...


The end of the array is determined by the enclosing object’s size field.

This design allows streaming decoding.

5.4 Nested Objects

Nested objects follow the same encoding rules recursively.

There is no depth limit imposed by the protocol; implementations SHOULD impose practical limits.

## 6. Variants

6.1 Union Types (oneOf, anyOf)

Variants are encoded as:

[variant_index][value]


variant_index: u8, based on schema order

value: encoded according to the selected schema

Example:

oneOf:
  - type: string
  - type: int32


Encoding "abc":

0x00 [string encoding]

6.2 Nullable Values

If nullable: true, a null value is encoded as:

0xff


No further bytes follow.

This sentinel value is reserved and MUST NOT collide with valid indices.

### 6.3 Custom variants

## 7. Implementation considerations

### 7.1 Determinism

Encoders MUST:

Alphabetically sort schema properties

Preserve value insertion order

Use canonical size encodings

Failure to do so breaks binary compatibility.

## 8. Security considerations

Implementations MUST guard against:

Oversized length prefixes

Deeply nested schemas

Malformed UTF-8

Integer overflow during size calculations

Decoders SHOULD impose:

Maximum object size

Maximum recursion depth

Maximum array element count

Compactr does not provide encryption, authentication, or integrity guarantees.

## 9. References

[OAS] OpenAPI Specification, The OpenAPI initiative, <https://spec.openapis.org/oas/v3.1.2.html>






### 3.2 Supported OpenAPI Types

| Type | Format | Bytes | Description |
| --- | --- | --- | --- |
| boolean | - | 1 | Boolean value |
| integer | int32 | 4 | 32-bit integer |
| integer | int64 | 8 | 64-bit integer |
| number | float | 4 | 32-bit floating point |
| number | double | 8 | 64-bit floating point |
| string | - | variable | UTF-8 variable size encoding |
| string | uuid | 16 | UUID (compressed) |
| string | ipv4 | 4 | IPv4 address |
| string | ipv6 | 16 | IPv6 address |
| string | date | 4 | Date (YYYY-MM-DD) |
| string | date-time | 8 | ISO 8601 date-time |
| string | binary | variable | Base64 binary data |
| array | - | variable | Array of items |
| object | - | variable | Nested object |

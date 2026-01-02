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

- [4. Primitive types](#4-Primitive-types)

- [5. Complex schemas](#5-Complex-schemas)

- [6. Variants](#6-Variants)

- [7. Implementation considerations](#7-Implementation-considerations)

- [8. Security considerations](#8-Security-considerations)

- [9. References](#9-References)

---

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

Compactr binary follows Network Byte Order (NBO) big-endian format.

### 2.2 Key limits

Indices are assigned for properties and stored as an unsigned 8-bit integer. Thus limiting the number of properties per object to 255.

### 2.3 Size limits

Some primitive types (e.g., `Boolean`) have fixed sizes, thus not requiring size bytes to be encoded, while others (e.g.,: `String`) have dynamic sizes.

Dynamically-sized properties have size limits represented by unsigned integers of varying sizes, which are described in the [primitives](#4-primitives) section of this document.


### 2.4 Schema properties and Encoding order

To maintain consistency across systems, the field index for each schema property is based on its alphabetical order, starting from 1.

The sorting function must be based on the numerical order of Unicode (UTF-16) character code values of the property names.

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
Will attribute index 0 to field `a`, index 1 to field `b` and 2 for `c`. Implementations of this protocol must follow this sorting rule to maintain consistency, even if properties are listed in differring orders across systems.

Encoding of values to generate the binary output simply follows the order in which the properties are listed in the structure or object.

For example, serializing `{ c: true, a: true, b: true }` with the previous schema will output: `0x03 0x01 0x01 0x01 0x02 0x01`. 


## 2.5 Versioning

Compactr binaries do not include version flags and the protocol does not include versioning mechanisms.

---

## 3. Schemas

## 4. Primitive types

## 5. Complex schemas

## 6. Variants

## 7. Implementation considerations

## 8. Security considerations

## 9. References

[OAS] OpenAPI Specification, The OpenAPI initiative, <https://spec.openapis.org/oas/v3.1.2.html>



Type | Variant | Bytes | Limit
---
String | - | 2 | 0xFFFF
String | Binary | 4 | 0xFFFFFFFF
Array | - | 2 | 


## Key Characteristics

1. **Big-endian byte order** for all multi-byte integers
2. **UTF-8 encoding** for strings (2-byte length prefix + UTF-8 bytes)
3. **Interleaved structure** for objects (index, size, value, index, size, value, ...)
4. **Alphabetical property indexing** for deterministic encoding
5. **Value insertion order** preserved in encoded output

## Primitive Types

### Boolean (1 byte)
- `true` → `0x01`
- `false` → `0x00`

### Int32 (4 bytes, big-endian signed)
- `0` → `0x00 0x00 0x00 0x00`
- `42` → `0x00 0x00 0x00 0x2a`
- `-1` → `0xff 0xff 0xff 0xff`

### Int64 (8 bytes, IEEE 754 double)
**Note:** Due to JavaScript limitations, int64 values are encoded as IEEE 754 double (f64).
- `0` → `0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00`
- `42` → `0x40 0x45 0x00 0x00 0x00 0x00 0x00 0x00`
- `9007199254740991` (MAX_SAFE_INTEGER) → `0x43 0x3f 0xff 0xff 0xff 0xff 0xff 0xff`

### Float (4 bytes, big-endian IEEE 754)
- `0.0` → `0x00 0x00 0x00 0x00`
- `1.0` → `0x3f 0x80 0x00 0x00`
- `3.14` → `0x40 0x48 0xf5 0xc3`

### Double (8 bytes, big-endian IEEE 754)
- `0.0` → `0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00`
- `1.0` → `0x3f 0xf0 0x00 0x00 0x00 0x00 0x00 0x00`

### String (2 bytes length + UTF-8 bytes)
- Empty string `""` → `0x00 0x00`
- `"A"` → `0x00 0x01 0x41` (length=1, UTF-8 'A'=0x41)
- `"Hello"` → `0x00 0x05 0x48 0x65 0x6c 0x6c 0x6f`

**In object properties:** Strings are encoded as raw UTF-8 bytes (no length prefix).

## Special Formats

### UUID (16 bytes, raw bytes)
Standard UUID format, no hyphens:
- `550e8400-e29b-41d4-a716-446655440000` → 16 raw bytes

### DateTime (9 bytes: component format)
- 2 bytes: year (u16 big-endian)
- 1 byte: month (1-12)
- 1 byte: day (1-31)
- 1 byte: hour (0-23)
- 1 byte: minute (0-59)
- 1 byte: second (0-59)
- 2 bytes: milliseconds (u16 big-endian, 0-999)

Example: `2024-01-15T10:30:00.000Z` → `0x07 0xe8 0x01 0x0f 0x0a 0x1e 0x00 0x00 0x00`

### Date (4 bytes: days since Unix epoch, i32 big-endian)
- `1970-01-01` → `0x00 0x00 0x00 0x00`
- `2024-01-01` → `0x00 0x00 0x4e 0x94`

### IPv4 (4 bytes, network order)
- `192.168.1.1` → `0xc0 0xa8 0x01 0x01`

### IPv6 (16 bytes, network order)
- `::1` → `0x00 0x00 ... 0x00 0x01` (15 zeros + 1)

### Binary (4 bytes length + raw data)
- 4 bytes: length (u32 big-endian)
- N bytes: raw binary data

## Array Format

Arrays encode each element with a 1-byte size prefix:

```
[element1_size, element1_data, element2_size, element2_data, ...]
```

### Example: `[1, 2, 3]` (int32 array)
```
0x04 0x00 0x00 0x00 0x01    // size=4, value=1
0x04 0x00 0x00 0x00 0x02    // size=4, value=2
0x04 0x00 0x00 0x00 0x03    // size=4, value=3
```

## Object Format

Objects use an **interleaved structure** where each property is encoded as:
`[index, size, value]`

### Structure
```
[num_props, index0, size0, value0, index1, size1, value1, ...]
```

- **num_props** (1 byte): Number of properties present
- **index** (1 byte): Alphabetical index of property in schema
- **size** (variable): Size encoding depends on type
- **value**: Encoded property value

### Property Indexing

Properties are indexed **alphabetically by name** (not schema insertion order):
- Schema `{id: ..., name: ..., email: ...}` → alphabetically: `email=0, id=1, name=2`

### Size Encoding

Different types use different size encodings:

**Compound types (Array, Object):**
- Always use `0x00` prefix
- Then: single byte if size < 256, else u16 big-endian

**Primitive types:**
- Single byte if size < 256
- Else: `0x00` prefix + u16 big-endian

**Strings in objects:**
- Raw UTF-8 bytes (no length prefix)
- Size field indicates byte count

### Example: `{x: 10, y: 20}` with schema `{x: int32, y: int32}`

```
0x02              // 2 properties
0x00 0x04         // x (index 0), size 4
0x00 0x00 0x00 0x0a  // x = 10
0x01 0x04         // y (index 1), size 4
0x00 0x00 0x00 0x14  // y = 20
```

### Property Order

**Important:** Properties are encoded in the order they appear in the **value object** (insertion order), but use alphabetical indices from the schema.

Example:
```javascript
// Value: {email: "a@b.com", id: 1, name: "Alice"}
// Alphabetical indices: email=0, id=1, name=2
// Encoded order: email (idx 0), id (idx 1), name (idx 2) - follows value insertion
```

### Optional Properties

Missing optional properties are simply omitted from the encoding. Only present properties are encoded.

## Wrapper Format (v3.x)

In compactr.js v3.x, all top-level values are wrapped in objects:

```javascript
// To encode the number 42:
schema({ value: { type: 'int32' } }).write({ value: 42 })

// Result: Object with one property 'value' = 42
```

This is different from v2.x which allowed standalone primitives.

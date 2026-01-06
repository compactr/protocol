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

This document specifies the compactr format, a schema-based serialization protocol that reuses existing [[OAS]](#6-References) OpenAPI specifications as schemas.

## Status

The specification is Stable as of this publication's release. 

## Table of Contents

- [1. Background](#1-Background)

- [2. Design decisions](#2-Design-decisions)

  - [2.1 Byte-order](#21-Byte-order)

  - [2.2 Key limits](#22-Key-limits)

  - [2.3 Size limits](#23-Size-limits)

  - [2.4 Schema properties and Encoding order](#24-Schema-properties-and-Encoding-order)

  - [2.5 Schema References and Versioning](#25-Schema-References-and-Versioning)

- [3. Schemas](#3-Schemas)

  - [3.1 Schema Source](#31-Schema-source)
 
  - [3.2 Required vs Optional Properties](#32-Required-vs-Optional-Properties)

  - [3.3 Composition Keywords](#33-Composition-Keywords)

- [4. Encoding](#4-Encoding)

  - [4.1 Variants](#41-Variants)

  - [4.2 Primitive types](#42-Primitive-types)

    - [4.2.1 Arrays](#421-arrays)

    - [4.2.2 Boolean](#422-boolean)

    - [4.2.3 Integers](#423-integers)

    - [4.2.4 Numbers](#424-numbers)

    - [4.2.5 Objects](#425-objects)

    - [4.2.6 Strings](#426-strings)

  - [4.3 Size Encoding](#43-size-encoding)

  - [4.4 Edge Cases](#44-edge-cases)

  - [4.5 Special formats](#45-special-formats)

    - [4.5.1 Binary](#451-binary)

    - [4.5.2 Date and DateTime](#452-date-and-datetime)

    - [4.5.3 IPV4 and IPV6](#453-ipv4-and-ipv6)

    - [4.5.4 UUID](#454-uuid)

- [5. Security considerations](#5-Security-considerations)

- [6. References](#6-References)

---

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [[RFC2119]](#6-References) [[RFC8174]](#6-References) when, and only when, they appear in all capitals, as shown here.

## 1. Background

Serialization in the context of Web APIs refers to the process of converting data structures into a format that can be easily transmitted over a network, typically in text-based formats (e.g., JSON, XML), or binary formats (e.g., files, [Protobuf](https://protobuf.dev/)), so that they can be understood and reconstructed by other systems.

A schema-based serialization approach enforces a predefined structure for data, ensuring consistency and validation, whereas a schema-less approach allows for more flexible and dynamic data representation, with fewer constraints on how data is organized.

Schema-based serialization protocols generally yield much smaller outputs, which is desirable to limit bandwidth and costs. The caveat to schema-based serialization is the cost of creating and maintaining schemas across multiple systems.

The initial concept for the compactr protocol was drafted in [2016](https://www.npmjs.com/package/compactr/v/0.0.1) with the goal of creating a schema-based serialization protocol that outputs minimal binary while using first party markdown or code structures as schemas.

While functional, the early versions would still require the knowledge of writing "compactr-style" schemas as Javascript Objects or JSON and limited adoption for languages outside of Javascript. As of compactr.js 3.0, released in 2025, the protocol moved to adopt OpenAPI 3.x as the base format for compactr schemas.

---

## 2. Design decisions

The primary objectives of the Compactr protocols are:

- First-party schema definitions, using [[OAS]](#6-References) OpenAPI specifications as base schemas.
- Optimized binary output
- Compatibility across runtimes
- Type safety

In order to meet these objectives, some key design decisions were made:

### 2.1 Byte-order

Compactr binary MUST follow Network Byte Order (NBO) big-endian format.

### 2.2 Key limits

Indices SHALL be assigned for properties and stored as an unsigned 8-bit integer (range 0-255). Since indices start from 1 (as per Section 2.4), the maximum property index is 255, thus limiting the number of properties per object to 255.

### 2.3 Size limits

Some primitive types (e.g., `Boolean`) have fixed sizes and therefore MUST NOT encode size bytes, while others (e.g., `String`, `Array`) have variable sizes and MUST include size bytes.

Size bytes are represented by unsigned integers. The specific size encoding for each type is described in the [Size Encoding](#43-size-encoding) section of this document.


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
Will attribute index 1 to field `a`, index 2 to field `b` and index 3 to field `c`. Implementations of this protocol MUST follow this sorting rule to maintain consistency, even if properties are listed in differing orders across systems.

Encoding of values to generate the binary output SHOULD simply follow the order in which the properties are listed in the structure or object.

For example, serializing `{ c: true, a: true, b: true }` with the previous schema will output: `0x03 0x01 0x01 0x01 0x02 0x01`. 


## 2.5 Schema References and Versioning

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

Compactr schemas are derived from OpenAPI 3.0+ Schema Objects, as defined in [[OAS]](https://spec.openapis.org/oas/v3.1.2.html) OpenAPI specifications.

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

Properties listed in the schema's `required` array MUST be present during encoding. Missing required properties MUST throw an encoding error.

Optional properties (not listed in `required`) MAY be omitted from the data being encoded.

Missing optional properties are not encoded and do not occupy space in the binary output.

Decoders MUST treat omitted optional properties as undefined (or language equivalent).

Example:

```
// Schema
{
  type: 'object',
  properties: {
    name: { type: 'string' },
    age: { type: 'integer', format: 'int32' }
  },
  required: ['name']
}

// Data with both properties
{ name: 'Alice', age: 30 }
// Output: Both properties encoded

// Data with only required property
{ name: 'Alice' }
// Output: Only 'name' encoded, 'age' omitted entirely

// Data missing required property
{ age: 30 }
// Result: Encoding error
```

### 3.3 Composition Keywords

Compactr walks through composition and walkthrough keywords `$ref`, `schema`, `oneOf`, `allOf`, `anyOf` and only creates internal models for primitives.

**`allOf`**: Merges all schemas in the array. All properties from all schemas MUST be encoded. The merged schema is treated as a single object schema.

**`oneOf` and `anyOf`**: Require a variant byte (see Section 4.1) to indicate which schema definition is being encoded. The variant byte starts at `0x01` for the first schema in the array and increments for each subsequent schema.

**`$ref`**: Resolves to the referenced schema and encodes according to that schema's type.

**`schema`**: A walkthrough keyword that wraps a schema definition without affecting encoding.

---

## 4. Encoding

Properties are encoded with the matching schema index first (`i`), then an optional variant byte (`v`), optional size byte(s) (`s`), then the encoded value (`d`).

`[i][v?][s?...][d...]`

### 4.1 Variants

Encoded fields which have the `nullable` schema property MUST include a variant byte that indicates whether the value is null or not.

- `0x00` For null values (no data bytes follow)
- `0x01` For non-null values (data bytes follow as per the type specification)

If the `nullable` property is not present in the schema, the variant byte MUST NOT be encoded. Attempts to encode `null` values for non-nullable fields MUST result in an encoding error.

Fields with multiple definitions, as described in the schema with the `oneOf` or `anyOf` keywords use the variant byte to indicate which definition to use, starting with `0x01` for the first definition, and incrementing by `0x01` for each subsequent one.

Example of nullable field:

```
// Schema
{
  type: 'object',
  properties: {
    optionalValue: { type: 'string', nullable: true }
  }
}

// Data with null value
{ optionalValue: null }

// Output: 0x01 0x00
// Breakdown: [index: 1][variant: null]

// Data with non-null value
{ optionalValue: 'test' }

// Output: 0x01 0x01 0x04 0x74 0x65 0x73 0x74
// Breakdown: [index: 1][variant: non-null][size: 4]['test']
```

Example of oneOf variant:

```
// Schema
{
  type: 'object',
  properties: {
    value: {
      oneOf: [
        { type: 'string' },
        { type: 'integer', format: 'int32' }
      ]
    }
  }
}

// Data with first variant (string)
{ value: 'hello' }

// Output: 0x01 0x01 0x05 0x68 0x65 0x6c 0x6c 0x6f
// Breakdown: [index: 1][variant: 1][size: 5]['hello']

// Data with second variant (integer)
{ value: 42 }

// Output: 0x01 0x02 0x00 0x00 0x00 0x2a
// Breakdown: [index: 1][variant: 2][int32: 42]
```


### 4.2 Primitive types

Types are based on JSON Schema Validation Specification Draft 2020-12: `array`, `boolean`, `integer`, `number`, `object` or `string`.

#### 4.2.1 Arrays

Arrays MUST include an unsigned 32-bit integer to represent the total byte size of all encoded array elements combined. Individual elements are then treated sequentially as their primitives defined in the schema.

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

Results in this buffer: `0x01 0x00 0x00 0x00 0x0e 0x05 0x68 0x65 0x6c 0x6c 0x6f 0x03 0x62 0x79 0x65 0x03 0x62 0x79 0x65`.

Breakdown: `[index: 1][array size: 14 bytes as 32-bit int][string 'hello': size 5 + 5 bytes][string 'bye': size 3 + 3 bytes][string 'bye': size 3 + 3 bytes]`

#### 4.2.2 Boolean

Fixed size of 1 byte, either 0x00 for false or 0x01 for true.

Example:

```
// Schema
{
  type: 'object',
  properties: {
    isActive: { type: 'boolean' }
  }
}

// Data
{ isActive: true }

// Output: 0x01 0x01
// Breakdown: [index: 1][value: true]
```

#### 4.2.3 Integers

Variable size based on the `format` attribute defined in the schema. Size bytes MUST NOT be encoded. Decoders MUST use the `format` attribute to determine the byte size.

- `(null, undefined or language equivalent)`: unsigned 32-bit integer
- `int32`: signed 32-bit integer
- `int64`: signed 64-bit integer

Example:

```
// Schema
{
  type: 'object',
  properties: {
    count: { type: 'integer', format: 'int32' }
  }
}

// Data
{ count: 42 }

// Output: 0x01 0x00 0x00 0x00 0x2a
// Breakdown: [index: 1][int32 value: 42 in big-endian]
```

#### 4.2.4 Numbers

Variable size based on the `format` attribute defined in the schema. Size bytes MUST NOT be encoded. Decoders MUST use the `format` attribute to determine the byte size.

All floating-point arithmetic MUST adhere to [[IEEE 754-2019]](#6-References).

- `(null, undefined or language equivalent)`: 64-bit floating point (double precision)
- `float`: 32-bit floating point (single precision)
- `double`: 64-bit floating point (double precision)

Example:

```
// Schema
{
  type: 'object',
  properties: {
    price: { type: 'number', format: 'float' }
  }
}

// Data
{ price: 19.99 }

// Output: 0x01 0x41 0xa0 0x3d 0x71
// Breakdown: [index: 1][IEEE 754 single-precision float for 19.99]
```

#### 4.2.5 Objects

Objects are encoded recursively using the same scheme: `[i][v?][s?...][d...]`.

Example:

```
// Schema
{
  type: 'object',
  properties: {
    user: {
      type: 'object',
      properties: {
        name: { type: 'string' },
        age: { type: 'integer', format: 'int32' }
      }
    }
  }
}

// Data
{ user: { name: 'Alice', age: 30 } }

// Output: 0x01 0x01 0x00 0x00 0x00 0x1e 0x02 0x05 0x41 0x6c 0x69 0x63 0x65
// Breakdown: [user index: 1][nested age index: 1][age value: 30][nested name index: 2][name size: 5]['Alice']
```

#### 4.2.6 Strings

Strings are encoded as UTF-8 multi-byte Unicode characters. The byte size is encoded as an unsigned 8-bit integer (1 byte, supporting sizes 0-255) followed by the UTF-8 encoded bytes. Most languages provide a UTF-8 encoding utility, which SHOULD be used to determine the size and generate the bytes to be appended.

For strings exceeding 255 bytes, see Section 4.3 Size Encoding for implementation requirements.

Example:

```
// Schema
{
  type: 'object',
  properties: {
    message: { type: 'string' }
  }
}

// Data
{ message: 'Hello' }

// Output: 0x01 0x05 0x48 0x65 0x6c 0x6c 0x6f
// Breakdown: [index: 1][size: 5]['Hello' as UTF-8 bytes]
```

### 4.3 Size Encoding

Variable-length types (strings, arrays, binary) encode their size using unsigned integers. The number of bytes used for size encoding depends on the type:

- **Strings**: unsigned 8-bit integer (1 byte) for sizes 0-255
- **Arrays**: unsigned 32-bit integer (4 bytes) for total byte size
- **Binary**: unsigned 32-bit integer (4 bytes) for byte size

For strings, if the UTF-8 encoded byte size exceeds 255 bytes, implementations MUST either use a larger integer type or throw an encoding error. Future versions of this specification MAY support variable-length integer encoding for sizes.

### 4.4 Edge Cases

#### Empty Arrays and Empty Strings

Empty arrays MUST encode a size of `0x00 0x00 0x00 0x00` (4 bytes) followed by no element bytes.

Empty strings MUST encode a size of `0x00` (1 byte) followed by no character bytes.

#### Unknown Schema Keywords

Implementations MUST ignore schema keywords that are not normative for Compactr encoding (as listed in Section 3.1). Non-normative keywords (e.g., `description`, `example`, `deprecated`) SHOULD NOT affect encoding or decoding behavior.

#### Missing Required Properties

Encoders MUST throw an error if a required property (listed in the schema's `required` array) is missing from the data being encoded.

Decoders MUST throw an error if a required property is missing from the encoded binary when the schema specifies it as required.

### 4.5 Special formats

Compactr supports encoding of special formats to improve efficiency. Additional special encoding formats MAY be added.

#### 4.5.1 Binary

Variable length `string` format with 32-bit size bytes.

Binary data in Buffers and UInt8Arrays MAY be encoded as-is (raw bytes), while binary data represented as `strings` MUST be encoded to Base64 before binary encoding.

#### 4.5.2 Date and DateTime

Fixed length `string` formats with no size bytes.

`date` is represented as 6 bytes encoding YYYY-MM-DD:
- `[uint32]` Year (0-9999)
- `[uint8]` Month (1-12)
- `[uint8]` Day (1-31)

`date-time` is represented as 18 bytes encoding YYYY-MM-DDTHH:mm:ss.sssZ (UTC time):
- `[uint32]` Year (0-9999)
- `[uint8]` Month (1-12)
- `[uint8]` Day (1-31)
- `[uint8]` Hour (0-23)
- `[uint8]` Minute (0-59)
- `[uint8]` Second (0-59)
- `[uint32]` Milliseconds (0-999)

Decoders MUST reconstruct values to conform to [[ISO 8601]](#6-References) extended date format.

Encoders SHOULD validate that input strings are valid dates and SHOULD set time components to 0 when not explicitly specified.

#### 4.5.3 IPV4 and IPV6

Fixed length `string` formats with no size bytes.

`ipv4` is represented as 4 bytes encoding dotted-decimal notation (e.g., "192.168.1.1"):
- `[uint8]` First octet (0-255)
- `[uint8]` Second octet (0-255)
- `[uint8]` Third octet (0-255)
- `[uint8]` Fourth octet (0-255)

Decoders MUST reconstruct to dotted-decimal string format as specified in [[RFC791]](#6-References).

`ipv6` is represented as 16 bytes encoding the 128-bit IPv6 address (e.g., "2001:0db8:85a3:0000:0000:8a2e:0370:7334"):
- `[uint32]` First 32 bits
- `[uint32]` Second 32 bits
- `[uint32]` Third 32 bits
- `[uint32]` Fourth 32 bits

Decoders MUST reconstruct to standard IPv6 string format as specified in [[RFC8200]](#6-References).

#### 4.5.4 UUID

Fixed sized 16 bytes using raw UUID bytes (network-order) and MUST be decoded as standard [[RFC9562]](#6-References) UUID.

---

## 5. Security considerations

Implementations MUST guard against the following security vulnerabilities:

### 5.1 Schema Validation

**Circular schemas**: Detect and reject schemas with circular `$ref` references to prevent infinite loops during encoding/decoding.

**Format-type mismatches**: Validate that schema `format` attributes match their declared `type`. For example, a schema declaring `{ type: 'number', format: 'uuid' }` is invalid and MUST be rejected, as `uuid` format only applies to `string` types.

Example of valid format-type combinations:
- `{ type: 'string', format: 'date' }` - Valid
- `{ type: 'string', format: 'uuid' }` - Valid
- `{ type: 'integer', format: 'int32' }` - Valid
- `{ type: 'integer', format: 'date' }` - Invalid (must reject)

### 5.2 Input Validation

**Malformed UTF-8**: Validate all string inputs are valid UTF-8 before encoding. Reject or sanitize malformed UTF-8 sequences.

**Integer overflow**: Validate that integer values fit within their declared format's range (e.g., int32 values must be between -2,147,483,648 and 2,147,483,647).

### 5.3 Resource Limits

**Maximum object keys**: Enforce a maximum limit on the number of properties per object (255 as per Section 2.2) to prevent resource exhaustion.

**Maximum recursion depth**: Implement a maximum depth limit for nested objects to prevent stack overflow attacks. A reasonable limit is 100 levels of nesting.

**Array byte size limits**: Enforce maximum array byte sizes to prevent memory exhaustion attacks. Implementations SHOULD reject arrays exceeding a configured size limit (e.g., 100MB).

### 5.4 Cryptographic Considerations

Compactr does not provide encryption, authentication, or integrity guarantees. Applications requiring these properties MUST implement them at the transport or application layer.

---

## 6. References

- [OAS] OpenAPI Specification v3.1.2. The Linux foundation (2025). <https://spec.openapis.org/oas/v3.1.2.html>
- [RFC791] Internet protocol. DARPA Internet Program Protocol Specification. (1981). <https://www.rfc-editor.org/rfc/rfc791>
- [RFC2119] Key words for use in RFCs to Indicate Requirement Levels. S. Bradner. IETF. (1997). <https://www.rfc-editor.org/rfc/rfc2119>
- [RFC8174] Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words. B. Leiba. IETF. (2017). <https://www.rfc-editor.org/rfc/rfc8174>
- [RFC8200] Internet Protocol, Version 6 (IPv6) Specification. S. Deering. IETF. (2017). <https://datatracker.ietf.org/doc/html/rfc8200>
- [RFC9562] Universally Unique IDentifiers (UUIDs). K. Davis. IETF. (2024) <https://www.rfc-editor.org/rfc/rfc9562.html>
- [IEEE 754-2019] IEEE 754-2019: IEEE Standard for Floating-Point Arithmetic. Institute of Electrical and Electronic Engineers. (2019).
- [ISO 8601] Date and time format. International Organization for Standardization. <https://www.iso.org/iso-8601-date-and-time-format.html>

---

Licensed under Apache 2.0, 2026, Compactr, Frederic Charette

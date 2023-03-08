---
sidebar_position: 2
sidebar_label: ZNG
---

# ZNG Specification

## 1. Introduction

ZNG (pronounced "zing") is an efficient, sequence-oriented serialization format for any data
conforming to the Zed data model.

ZNG is "row oriented" and
analogous to [Apache Avro](https://avro.apache.org) but does not
require schema definitions as it instead utilizes the fine-grained type system
of the Zed data model.
This binary format is based on machine-readable data types with an
encoding methodology inspired by Avro,
[Parquet](https://en.wikipedia.org/wiki/Apache_Parquet), and
[Protocol Buffers](https://developers.google.com/protocol-buffers).

To this end, ZNG embeds all type information
in the stream itself while having a binary serialization format that
allows "lazy parsing" of fields such that
only the fields of interest in a stream need to be deserialized and interpreted.
Unlike Avro, ZNG embeds its "schemas" in the data stream as Zed types and thereby admits
an efficient multiplexing of heterogeneous data types by prepending to each
data value a simple integer identifier to reference its type.

Since no external schema definitions exist in ZNG, a "type context" is constructed
on the fly by composing dynamic type definitions embedded in the ZNG format.
ZNG can be readily adapted to systems like
[Apache Kafka](https://kafka.apache.org/) which utilize schema registries,
by having a connector translate the schemas implied in the
ZNG stream into registered schemas and vice versa.  Better still, Kafka could
be used natively with ZNG obviating the need for the schema registry.

Multiple ZNG streams with different type contexts are easily merged because the
serialization of values does not depend on the details of
the type context.  One or more streams can be merged by simply merging the
input contexts into an output context and adjusting the type reference of
each value in the output ZNG sequence.  The values need not be traversed
or otherwise rewritten to be merged in this fashion.

## 2. The ZNG Format

A ZNG stream comprises a sequence of frames where
each frame contains one of three types of data:
_types_, _values_, or externally-defined _control_.

A stream is punctuated by the end-of-stream value `0xff`.

Each frame header includes a length field
allowing an implementation to easily skip from frame to frame.

Each frame begins with a single-byte "frame code":
```
    7 6 5 4 3 2 1 0
   +-+-+-+-+-+-+-+-+
   |V|C|  T|      L|
   +-+-+-+-+-+-+-+-+

   V: 1 bit

     Version number.  Must be zero.

   C: 1 bit

     Indicates compressed frame data.

   T: 2 bits

     Type of frame data.

       00: Types
       01: Values
       10: Control
       11: End of stream

   L: 4 bits

     Low-order bits of frame length.
```

Bit 7 of the frame code must be zero as it defines version 0
of the ZNG stream format.  If a future version of ZNG
arises, bit 7 of future ZNG frames will be 1.
ZNG version 0 readers must ignore and skip over such frames using the
`len` field, which must survive future versions.
Any future versions of ZNG must be able to integrate version 0 frames
for backward compatibility.

Following the frame code is its encoded length followed by a "frame payload"
of bytes of said length:
```
<frame code><uvarint><frame payload>
```
The length encoding utilizes a variable-length unsigned integer called herein a `uvarint`:

> Inspired by Protocol Buffers,
> a `uvarint` is an unsigned, variable-length integer encoded as a sequence of
> bytes consisting of N-1 bytes with bit 7 clear and the Nth byte with bit 7 set,
> whose value is the base-128 number composed of the digits defined by the lower
> 7 bits of each byte from least-significant digit (byte 0) to
> most-significant digit (byte N-1).

The frame payload's length is equal to the value of the `uvarint` following the
frame code times 16 plus the low 4-bit integer value `L` field in the frame code.

If the `C` bit is set in the frame code, then the frame payload following the
frame length is compressed and has the form:
```
<format><size><compressed payload>
```
where
* `<format>` is a single byte indicating the compression format of the the compressed payload,
* `<size>` is a `uvarint` encoding the size of the uncompressed payload, and
* `<compressed payload>` is a bytes sequence whose length equals
the outer frame length less 1 byte for the compression format and the encoded length
of the `uvarint` size field.

The `compressed payload` is compressed according to the compression algorithm
specified by the `format` byte.  Each frame is compressed independently
such that the compression algorithm's state is not carried from frame to frame
(thereby enabling parallel decoding).

The `<size>` value is redundant with the compressed payload
but is useful to an implementation to deterministically
size decompression buffers in advance of decoding.

Values for the `format` byte are defined in the
ZNG compression format specification.

> This arrangement of frames separating types and values allows
> for efficient scanning and parallelization.  In general, values depend
> on type definitions but as long as all of the types are known when
> values are used, decoding can be done in parallel.  Likewise, since
> each block is independently compressed, the blocks can be decompressed
> in parallel.  Moreover, efficient filtering can be carried out over
> uncompressed data before it is deserialized into native data structures,
> e.g., allowing entire frames to be discarded based on
> heuristics, e.g., knowing a filtering predicate can't be true based on a
> quick scan of the data perhaps using the Boyer-Moore algorithm to determine
> that a comparison with a string constant would not work for any
> value in the buffer.

Whether the payload was originally uncompressed or was decompressed, it is
then interpreted according to the `T` bits of the frame code as a
* [types frame](#21-types-frame),
* [values frame](#22-values-frame), or
* [control frame](#23-control-frame).

### 2.1 Types Frame

A _types frame_ encodes a sequence of type definitions for complex Zed types
and establishes a "type ID" for each such definition.
Type IDs for the "primitive types"
are predefined with the IDs listed in the [Primitive Types](#3-primitive-types) table.

Each definition, or "typedef",
consists of a typedef code followed by its type-specific encoding as described below.
Each type must be decoded in sequence to find the start of the next type definition
as there is no framing to separate the typedefs.

The typedefs are numbered in the order encountered starting at 30
(as the largest primary type ID is 29).  Types refer to other types
by their type ID.  Note that the type ID of a typedef is implied by its
position in the sequence and is not explicitly encoded.

The typedef codes are defined as follows:

| Code | Complex Type             |
|------|--------------------------|
|   0  |  record type definition  |
|   1  |  array type definition   |
|   2  |  set type definition     |
|   3  |  map type definition     |
|   4  |  union type definition   |
|   5  |  enum type definition    |
|   6  |  error type definition   |
|   7  |  named type definition   |

Any references to a type ID in the body of a typedef are encoded as a `uvarint`,

#### 2.1.1 Record Typedef

A record typedef creates a new type ID equal to the next stream type ID
with the following structure:
```
---------------------------------------------------------
|0x00|<nfields>|<name1><type-id-1><name2><type-id-2>...|
---------------------------------------------------------
```
Record types consist of an ordered set of fields where each field consists of
a name and its type.  Unlike JSON, the ordering of the fields is significant
and must be preserved through any APIs that consume, process, and emit ZNG records.

A record type is encoded as a count of fields, i.e., `<nfields>` from above,
followed by the field definitions,
where a field definition is a field name followed by a type ID, i.e.,
`<name1>` followed by `<type-id-1>` etc. as indicated above.

The field names in a record must be unique.

The `<nfields>` value is encoded as a `uvarint`.

The field name is encoded as a UTF-8 string defining a "ZNG identifier".
The UTF-8 string
is further encoded as a "counted string", which is the `uvarint` encoding
of the length of the string followed by that many bytes of UTF-8 encoded
string data.

N.B.: As defined by [ZSON](zson.md), a field name can be any valid UTF-8 string much like JSON
objects can be indexed with arbitrary string keys (via index operator)
even if the field names available to the dot operator are restricted
by language syntax for identifiers.

The type ID follows the field name and is encoded as a `uvarint`.

#### 2.1.2 Array Typedef

An array type is encoded as simply the type code of the elements of
the array encoded as a `uvarint`:
```
----------------
|0x01|<type-id>|
----------------
```

#### 2.1.3 Set Typedef

A set type is encoded as the type ID of the
elements of the set, encoded as a `uvarint`:
```
----------------
|0x02|<type-id>|
----------------
```

#### 2.1.4 Map Typedef

A map type is encoded as the type code of the key
followed by the type code of the value.
```
--------------------------
|0x03|<type-id>|<type-id>|
--------------------------
```
Each `<type-id>` is encoded as `uvarint`.

#### 2.1.5 Union Typedef

A union typedef creates a new type ID equal to the next stream type ID
with the following structure:
```
-----------------------------------------
|0x04|<ntypes>|<type-id-1><type-id-2>...|
-----------------------------------------
```
A union type consists of an ordered set of types
encoded as a count of the number of types, i.e., `<ntypes>` from above,
followed by the type IDs comprising the types of the union.
The type IDs of a union must be unique.

The `<ntypes>` and the type IDs are all encoded as `uvarint`.

`<ntypes>` cannot be 0.

#### 2.1.6 Enum Typedef

An enum type is encoded as a `uvarint` representing the number of symbols
in the enumeration followed by the names of each symbol.
```
--------------------------------
|0x05|<nelem>|<name1><name2>...|
--------------------------------
```
`<nelem>` is encoded as `uvarint`.
The names have the same UTF-8 format as record field names and are encoded
as counted strings following the same convention as record field names.

#### 2.1.7 Error Typedef

An error type is encoded as follows:
```
----------------
|0x06|<type-id>|
----------------
```
which defines a new error type for error values that have the underlying type
indicated by `<type-id>`.

#### 2.1.8 Named Type Typedef

A named type defines a new type ID that binds a name to a previously existing type ID.  

A named type is encoded as follows:
```
----------------------
|0x07|<name><type-id>|
----------------------
```
where `<name>` is an identifier representing the new type name with a new type ID
allocated as the next available type ID in the stream that refers to the
existing type ID `<type-id>`.  `<type-id>` is encoded as a `uvarint` and `<name>`
is encoded as a `uvarint` representing the length of the name in bytes,
followed by that many bytes of UTF-8 string.

As indicated in the [data model](zed.md),
it is an error to define a type name that has the same name as a primitive type,
and it is permissible to redefine a previously defined type name with a
type that differs from the previous definition.

### 2.2 Values Frame

A _values frame_ is a sequence of Zed values each encoded as the value's type ID,
encoded as a `uvarint`, followed by its tag-encoded serialization as described below.

Since a single type ID encodes the entire value's structure, no additional
type information is needed.  Also, the value encoding follows the structure
of the type explicitly so the type is not needed to parse the structure of the
value, but rather only its semantics.

It is an error for a value to reference a type ID that has not been
previously defined by a typedef scoped to the stream in which the value
appears.

The value is encoded using a "tag-encoding" scheme
that captures the structure of both primitive types and the recursive
nature of complex types.  This structure is encoded
explicitly in every value and the boundaries of each value and its
recursive nesting can be parsed without knowledge of the type or types of
the underlying values.  This admits an efficient implementation
for traversing the values, inclusive of recursive traversal of complex values,
whereby the inner loop need not consult and interpret the type ID of each element.

#### 2.2.1 Tag-Encoding of Values

Each value is prefixed with a "tag" that defines:
* whether it is the null value, and
* its encoded length in bytes.

The tag is 0 for the null value and `length+1` for non-null values where
`length` is the encoded length of the value.  Note that this encoding
differentiates between a null value and a zero-length value.  Many data types
have a meaningful interpretation of a zero-length value, for example, an
empty array, the empty record, etc.

The tag itself is encoded as a `uvarint`.

#### 2.2.2 Tag-Encoded Body of Primitive Values

Following the tag encoding is the value encoded in N bytes as described above.
A typed value with a `value` of length `N` is interpreted as described in the
[Primitive Types](#3-primitive-types) table.  The type information needed to
interpret all of the value elements of a complex type are all implied by the
top-level type ID of the values frame.  For example, the type ID could indicate
a particular record type, which recursively provides the type information
for all of the elements within that record, including other complex types
embedded within the top-level record.

Note that because the tag indicates the length of the value, there is no need
to use varint encoding of integer values.  Instead, an integer value is encoded
using the full 8 bits of each byte in little-endian order.  Signed values,
before encoding, are shifted left one bit, and the sign bit stored as bit 0.
For negative numbers, the remaining bits are negated so that the upper bytes
tend to be zero-filled for small integers.

#### 2.2.3 Tag-Encoded Body of Complex Values

The body of a length-N container comprises zero or more tag-encoded values,
where the values are encoded as follows:

| Type     |          Value                          |
|----------|-----------------------------------------|
| `array`  | concatenation of elements               |
| `set`    | normalized concatenation of elements    |
| `record` | concatenation of elements               |
| `map`    | concatenation of key and value elements |
| `union`  | concatenation of tag and value     |
| `enum`   | position of enum element                |
| `error`  | wrapped element                         |

Since N, the byte length of any of these container values, is known,
there is no need to encode a count of the
elements present.  Also, since the type ID is implied by the typedef
of any complex type, each value is encoded without its type ID.

For sets, the concatenation of elements must be normalized so that the
sequence of bytes encoding each element's tag-counted value is
lexicographically greater than that of the preceding element.

A union value is encoded as a container with two elements. The first
element, called the tag, is the `uvarint` encoding of the
positional index determining the type of the value in reference to the
union's list of defined types, and the second element is the value
encoded according to that type.

An enumeration value is represented as the `uvarint` encoding of the
positional index of that value's symbol in reference to the enum's
list of defined symbols.

A map value is encoded as a container whose elements are alternating
tag-encoded keys and values, with keys and values encoded according to
the map's key type and value type, respectively.

The concatenation of elements must be normalized so that the
sequence of bytes encoding each tag-counted key (of the key/value pair) is
lexicographically greater than that of the preceding key (of the preceding
key/value pair).

### 2.3 Control Frame

A _control frame_ contains an application-defined control message.

Control frames are available to higher-layer protocols and are carried
in ZNG as a convenient signaling mechanism.  A ZNG implementation
may skip over all control frames and is guaranteed by
this specification to decode all of the data as described herein even if such
frames provide additional semantics on top of the base ZNG format.

The body of a control frame is a control message and may be JSON,
ZSON, ZNG, binary, or UTF-8 text.  The serialization of the control
frame body is independent of the ZNG stream containing the control
frame.

Any control message not known by a ZNG data receiver shall be ignored.

The delivery order of control messages with respect to the delivery
order of values of the ZNG stream should be preserved by an API implementing
ZNG serialization and deserialization.
In this way, system endpoints that communicate using ZNG can embed
protocol directives directly into the ZNG stream as control payloads
in an order-preserving semantics rather than defining additional
layers of encapsulation and synchronization between such layers.

A control frame has the following form:
```
-------------------------
|<encoding>|<len>|<body>|
-------------------------
```
where
* `<encoding>` is a single byte indicating whether the body is encoded
as ZNG (0), JSON (1), ZSON (2), an arbitrary UTF-8 string (3), or arbitrary binary data (4),
* `<len>` is a `uvarint` encoding the length in bytes of the body
(exclusive of the length 1 encoding byte), and
* `<body>` is a control message whose semantics are outside the scope of
the base ZNG specification.

If the encoding type is ZNG, the embedded ZNG data
starts and ends a single ZNG stream independent of the outer ZNG stream.

### 2.4 End of Stream

A ZNG stream must be terminated by an end-of-stream marker.
A new ZNG stream may begin immediately after an end-of-stream marker.
Each such stream has its own, independent type context.

In this way, the concatenation of ZNG streams (or ZNG files containing
ZNG streams) results in a valid ZNG data sequence.

For example, a large ZNG file can be arranged into multiple, smaller streams
to facilitate random access at stream boundaries.
This benefit comes at the cost of some additional overhead --
the space consumed by stream boundary markers and repeated type definitions.
Choosing an appropriate stream size that balances this overhead with the
benefit of enabling random access is left up to implementations.

End-of-stream markers are also useful in the context of sending ZNG over Kafka,
as a receiver can easily resynchronize with a live Kafka topic by
discarding incomplete frames until a frame is found that is terminated
by an end-of-stream marker (presuming the sender implementation aligns
the ZNG frames on Kafka message boundaries).

A end-of-stream marker is encoded as follows:
```
------
|0xff|
------
```

After this marker, all previously read
typedefs are invalidated and the "next available type ID" is reset to
the initial value of 30.  To represent subsequent values that use a
previously defined type, the appropriate typedef control code must
be re-emitted
(and note that the typedef may now be assigned a different ID).

## 3. Primitive Types

For each ZNG primitive type, the following table describes:
* its type ID, and
* the interpretation of a length `N` [value frame](#22-values-frame).

All fixed-size multi-byte sequences representing machine words
are serialized in little-endian format.

| Type         | ID |    N     |       ZNG Value Interpretation                 |
|--------------|---:|:--------:|------------------------------------------------|
| `uint8`      |  0 | variable | unsigned int of length N                       |
| `uint16`     |  1 | variable | unsigned int of length N                       |
| `uint32`     |  2 | variable | unsigned int of length N                       |
| `uint64`     |  3 | variable | unsigned int of length N                       |
| `uint128`    |  4 | variable | unsigned int of length N                       |
| `uint256`    |  5 | variable | unsigned int of length N                       |
| `int8`       |  6 | variable | signed int of length N                         |
| `int16`      |  7 | variable | signed int of length N                         |
| `int32`      |  8 | variable | signed int of length N                         |
| `int64`      |  9 | variable | signed int of length N                         |
| `int128`     | 10 | variable | signed int of length N                         |
| `int256`     | 11 | variable | signed int of length N                         |
| `duration`   | 12 | variable | signed int of length N as ns                   |
| `time`       | 13 | variable | signed int of length N as ns since epoch       |
| `float16`    | 14 |     2    | 2 bytes of IEEE 64-bit format                  |
| `float32`    | 15 |     4    | 4 bytes of IEEE 64-bit format                  |
| `float64`    | 16 |     8    | 8 bytes of IEEE 64-bit format                  |
| `float128`   | 17 |    16    | 16 bytes of IEEE 64-bit format                 |
| `float256`   | 18 |    32    | 32 bytes of IEEE 64-bit format                 |
| `decimal32`  | 19 |     4    | 4 bytes of IEEE decimal format                 |
| `decimal64`  | 20 |     8    | 8 bytes of IEEE decimal format                 |
| `decimal128` | 21 |    16    | 16 bytes of IEEE decimal format                |
| `decimal256` | 22 |    32    | 32 bytes of IEEE decimal format                |
| `bool`       | 23 |     1    | one byte 0 (false) or 1 (true)                 |
| `bytes`      | 24 | variable | N bytes of value                               |
| `string`     | 25 | variable | UTF-8 byte sequence                            |
| `ip`         | 26 | 4 or 16  | 4 or 16 bytes of IP address                    |
| `net`        | 27 | 8 or 32  | 8 or 32 bytes of IP prefix and subnet mask     |
| `type`       | 28 | variable | type value byte sequence [as defined below](#4-type-values) |
| `null`       | 29 |    0     | No value, always represents an undefined value |

## 4. Type Values

As the ZSON data model supports first-class types and because the ZNG design goals
require that value serializations cannot change across type contexts, type values
must be encoded in a fashion that is independent of the type context.
Thus, a serialized type value encodes the entire type in a canonical form
according to the recursive definition in this section.

The type value of a primitive type (include type `type`) is its primitive ID,
serialized as a single byte.

The type value of a complex type is serialized recursively according to the
complex type it represents as described below.

#### 4.1 Record Type Value

A record type value has the form:
```
---------------------------------------------------
|30|<nfields>|<name1><typeval><name2><typeval>...|
---------------------------------------------------
```
where `<nfields>` is the number of fields in the record encoded as a `uvarint`,
`<name1>` etc. are the field names encoded as in the
record typedef, and each `<typeval>` is a recursive encoding of a type value.

#### 4.2 Array Type Value

An array type value has the form:
```
--------------
|31|<typeval>|
--------------
```
where `<typeval>` is a recursive encoding of a type value.

#### 4.3 Set Type Value

An set type value has the form:
```
--------------
|32|<typeval>|
--------------
```
where `<typeval>` is a recursive encoding of a type value.

#### 4.4 Map Type Value

A map type value has the form:
```
--------------------------
|33|<key-type>|<val-type>|
--------------------------
```
where `<key-type>` and `<val-type>` are recursive encodings of type values.

#### 4.5 Union Type Value

A union type value has the form:
```
-----------------------------------
|34|<ntypes>|<typeval><typeval>...|
-----------------------------------
```
where `<ntypes>` is the number of types in the union encoded as a `uvarint`
and each `<typeval>` is a recursive definition of a type value.

#### 4.6 Enum Type Value

An enum type value has the form:
```
------------------------------
|35|<nelem>|<name1><name2>...|
------------------------------
```
where `<nelem>` and each symbol name is encoded as in an enum typedef.

#### 4.7 Error Type Value

An error type value has the form:
```
-----------
|36|<type>|
-----------
```
where `<type>` is the type value of the error.

#### 4.8 Named Type Type Value

A named type type value may appear either as a definition or a reference.
When a named type is referenced, it must have been previously
defined in the type value in accordance with a left-to-right depth-first-search (DFS)
traversal of the type.

A named type definition has the form:
```
--------------------
|37|<name><typeval>|
--------------------
```
where `<name>` is encoded as in an named type typedef
and `<typeval>` is a recursive encoding of a type value.  This creates
a binding between the given name and the indicated type value only within the
scope of the encoded value and does not affect the type context.
This binding may be changed by another named type definition
of the same name in the same type value according to the DFS order.

An named type reference has the form:
```
-----------
|38|<name>|
-----------
```
It is an error for an named type reference to appear in a type value with a name
that has not been previously defined according to the DFS order.

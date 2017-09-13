HBON
====

Hummingbird Object Notation

Latest version: v1.0.0

Objectives
----------

HBON aims to be a binary serialization format for passing objects between
different server and clients. HBON objects are self-describing, strongly-typed 
and easy to serialize to and parse from.

HBON was designed to replace a custom binary (and arbitrarily ordered) wire
format used in a mobile app to communicate between clients and servers. As the
application required high data throughput, we wanted a format that was 
lightweight yet performant to convert to and from. After considering various
alternatives, HBON was born.

While it is possible to be more thrifty with the byte allocation, HBON is 
designed with the byte (8 bits) boundary so we can inspect data over the 
wire without conversion. We think it's a nice to have.

"[Hummingbirds] are among the smallest of birds, most species measuring 7.5–13 cm (3–5 in) in length". ([Wikipedia](https://en.wikipedia.org/wiki/Hummingbird))

Table of contents
-----------------
- [Example](#user-content-example)
- [Spec](#user-content-spec)
- [Root object](#user-content-root-object)
- [Number](#user-content-number)
- [Key](#user-content-key)
- [Integer](#user-content-integer)
- [Floating points](#user-content-floating-point)
- [String](#user-content-string)
- [Boolean](#user-content-boolean)
- [Array](#user-content-array)
- [Map](#user-content-map)
- [GUID](#user-content-guid)
- [Limitations](#user-content-limitations)
- [Implementations](#user-content-implementations)

Example
-------

A very simple HBON object

```
0x0D 0x01 0x05 0x68 0x65 0x6C 0x6C 0x6F 
0x10 0x05 0x77 0x6F 0x72 0x6C 0x64
```

This breaks down to

|                            |                       |
| :------------------------- | :-------------------- |
| `0x0D`                     | Map type              |
| `0x01`                     | Number of keys        |
| `0x05`                     | Length of (first) key |
| `0x68 0x65 0x6C 0x6C 0x6F` | "Hello"               |
| `0x10`                     | Type of value: string |
| `0x05`                     | Length of value       |
| `0x77 0x6F 0x72 0x6C 0x64` | "World"               |

Spec
----
- The root of every HBON object is a **Map**.
- The byte order of all values are stored in Big Endian.
- HBON objects are data model agnostic.

Root object
-----------

The root of every HBON object is a **Map** ([see Map spec](#user-content-map)). As HBON is most often used to represent a bag of key/value pairs, the map
provides an easy default to represent these data types.

The recommendation for serializing non-map data types (such as a single array) is to wrap the data in a **Map** with a 0 key. This allows parsers and serializers to provide consistent and simple interfaces to get and set them.

When inspecting a HBON object, you'll see a binary stream like the following: 

```
+------+----------------+-----+-------+-----+-------+- - -
| 0x0D | Number of keys | Key | Value | Key | Value | ...
+------+----------------+-----+-------+-----+-------+- - -
```

Number
------

**Numbers** are used throughout HBON to indicate variable length items, such as the number of keys or the length of an item. Each **Number** has a variable number of bytes from one to 7, representing values between 0 to 4,294,967,295. Note that **Numbers** are distinct from value types such as integers or floating points.

**Numbers** can be single byte, three bytes or 7 bytes. The rule for constructing numbers is as follows:

- If the number is smaller than 255, represent it using a single byte.
- If the number is greater or equal to 255 but less than 65535, set the first byte to `0xFF` (255) and put the number in the next two bytes.
- If the number is greater or equal to 65535, set the first three bytes to `0xFF` and use the last four bytes to represent the number.
- This schema is technically infinitely extensible, but there is currently no need for very large numbers so the spec calls to parse at least a minimum of 7 bytes.

```
127:
+------+
| 0x7F |
+------+

255:
+------+------+------+
| 0xFF | 0x00 | 0xFF |
+------+------+------+

3,999:
+------+------+------+
| 0xFF | 0x9F | 0x0F |
+------+------+------+

1,000,000:
+------+------+------+------+------+------+------+
| 0xFF | 0xFF | 0xFF | 0x40 | 0x42 | 0x0F | 0x00 |
+------+------+------+------+------+------+------+

```


Key
---

The default key in Maps are **String** keys. Like the **String** value type, **String** keys is constructed with a **Number** indicating the number of bytes of the UTF-8 encoded string, and the UTF-8 string following it.

```
+------+------+------+------+------+------+
| Len  | "H"  | "e"  | "l"  | "l"  | "o"  |
+------+------+------+------+------+------+
| 0x05 | 0x48 | 0x65 | 0x6C | 0x6C | 0x6F |
+------+------+------+------+------+------+
```

### Short Key

To further help save bytes over the wire, the specification defines a short key format which allows keys to be substituted with a mutually agreed set of key/number pairings. Short keys are a single byte unsigned integer, with a minimum value of 0 and a maximum value of 255. The first byte is then set to the value of 0, and the second byte contains the value of the key.

For example, instead of sending "Hello" every time through the wire, both the server and client can define the integer 8 as the Short Key.

```
+------+------+
| Len  | S.K. |
+------+------+
| 0x00 | 0x08 |
+------+------+
```

This is designed such that from the user of a parser/serialization perspective, the data can be compressed without compromising on the readability of the object.

```
JSON:
{ "hello": "world" }

HBON:
+------+------+------+------+------+------+------+------+------+------+------+
| Type | Num  | Short Key   | Value                                          |
+------+------+------+------+------+------+------+------+------+------+------+
| 0x0D | 0x01 | 0x00 | 0x08 | 0x10 | 0x05 | 0x77 | 0x6F | 0x72 | 0x6C | 0x64 |
+------+------+------+------+------+------+------+------+------+------+------+
```

Value types
-----------
HBON is strongly typed, and as a result there are various value types defined in the spec.

Fundamentally, values are represented by the type indicator on the first byte followed by the specific format of the value.

```
53 (UInt8):
+------+-------+
| Ind. | Value |
+------+-------+
| 0x01 | 0x35  |
+------+-------+
```

Here is a list of all Value types supported in HBON

| Indicator | Type   | Data length |
|:----------|:-------|:------------|
| 0x01      | UInt8  | 1 byte      |
| 0x02      | Int16  | 2 bytes     |
| 0x03      | UInt16 | 2 bytes     |
| 0x04      | Int32  | 4 bytes     |
| 0x05      | UInt32 | 4 bytes     |
| 0x06      | Int64  | 8 bytes     |
| 0x07      | UInt64 | 8 bytes     |
| 0x08      | Double | 8 bytes     |
| 0x09      | Float  | 4 bytes     |
| 0x0A      | String | Variable    |
| 0x0B      | Bool   | 1 byte      |
| 0x0C      | Array  | Variable    |
| 0x0D      | Map    | Variable    |
| 0x0E      | GUID   | 16 bytes    |

Integer
-------
**Integer** types come in the one, two, four or 8 byte varieties in both the signed and unsigned forms.

| Indicator | Type   | Data length |
|:----------|:-------|:------------|
| 0x01      | UInt8  | 1 byte      |
| 0x02      | Int16  | 2 bytes     |
| 0x03      | UInt16 | 2 bytes     |
| 0x04      | Int32  | 4 bytes     |
| 0x05      | UInt32 | 4 bytes     |
| 0x06      | Int64  | 8 bytes     |
| 0x07      | UInt64 | 8 bytes     |

**Integer** typed values are formatted starting with the type indicator byte followed by the bytes of the value.

```
53 (UInt8):
+------+------+
| Ind. | Val. |
+------+------+
| 0x01 | 0x35 |
+------+------+

-2017 (Int16):
+------+------+------+
| Ind. | Value       |
+------+------+------+
| 0x02 | 0x1F | 0xF8 |
+------+------+------+

2017 (UInt16):
+------+------+------+
| Ind. | Value       |
+------+------+------+
| 0x03 | 0x07 | 0xE1 |
+------+------+------+

```

Floating point
--------------

HBON supports both single and double precision floating points.

| Indicator | Type   | Data length |
|:----------|:-------|:------------|
| 0x08      | Double | 8 bytes     |
| 0x09      | Float  | 4 bytes     |

**Floating point** typed values are formatted starting with the type indicator byte followed by the bytes of the value.

```
3.14159265359 (Double):
+------+------+------+------+------+------+------+------+------+
| Ind. | Value                                                 |
+------+------+------+------+------+------+------+------+------+
| 0x08 | 0xEA | 0x2E | 0x44 | 0x54 | 0xFB | 0x21 | 0x09 | 0x40 |
+------+------+------+------+------+------+------+------+------+

3.14159265359 (Float):
+------+------+------+------+------+
| Ind. | Value                     |
+------+------+------+------+------+
| 0x09 | 0xDB | 0x0F | 0x49 | 0x40 |
+------+------+------+------+------+
```

String
------

| Indicator | Type   | Data length |
|:----------|:-------|:------------|
| 0x0A      | String | Variable    |

**String** typed values are formatted starting with the type indicator byte, the number of bytes of the string, followed by the bytes of the UTF-8 encoded string.

```
❤️:
+------+------+------+------+------+------+------+------+
| Ind. | Len. | Value                                   |
+------+------+------+------+------+------+------+------+
| 0x0A | 0x06 | 0xE2 | 0x9D | 0xA4 | 0xEF | 0xB8 | 0x8F |
+------+------+------+------+------+------+------+------+
```

Boolean
-------
| Indicator | Type   | Data length |
|:----------|:-------|:------------|
| 0x0B      | Bool   | 1 byte      |

**Boolean** typed values are formatted starting with the type indicator byte, and followed with 0x01 for `true` and 0x00 for `false`.

```
true:
+------+------+
| Ind. | Val. |
+------+------+
| 0x0B | 0x01 |
+------+------+

false:
+------+------+
| Ind. | Val. |
+------+------+
| 0x0B | 0x00 |
+------+------+
```

Array
-----

| Indicator | Type   | Data length |
|:----------|:-------|:------------|
| 0x0C      | Array  | Variable    |

**Arrays** in HBON are fixed-length, ordered, and consists only of a single type. Construct an **array** starting with the type indicator byte, followed by the number of elements, the type of all elements, and the byte of the elements themselves.

```
[1, 1, 2, 3, 5]:
+------+------+------+------+------+------+------+------+
| Ind. | Len  | Type | El 1 | El 2 | El 3 | El 4 | El 5 |
+------+------+------+------+------+------+------+------+
| 0x0C | 0x05 | 0x01 | 0x01 | 0x01 | 0x02 | 0x03 | 0x05 |
+------+------+------+------+------+------+------+------+

["one", "one", "two"]:
+------+------+------+-----+----+----+-----+-----+----+----+----
| Ind. | Len  | Type | El 1                | El 2                 
+------+------+------+-----+----+----+-----+-----+----+----+----
| 0x0C | 0x03 | 0x0A | 0x03 0x6F 0x6E 0x65 | 0x03 0x6F 0x6E 0x65 
+------+------+------+-----+----+----+-----+-----+----+----+----

   -+-----+----+----+-----+
    | El 3                |
   -+-----+----+----+-----+
    | 0x03 0x74 0x77 0x6F |
   -+-----+----+----+-----+
```

Map
---
| Indicator | Type   | Data length |
|:----------|:-------|:------------|
| 0x0D      | Map    | Variable    |

**Maps** in HBON are fixed-length, unordered and can consists of multiple key/value pairs. Each of the key/value pair can be of a different type. Construct a **Map** starting with the type indicator, the **Number** of key/value pairs, followed by each of the key/value pair in sequence.

```
{
    "hello": "world",
    "pi": 3.14159 
}:
+------+------+-----+----+----+----+----+-----
| Ind. | Len. | Key 1
+------+------+-----+----+----+----+----+-----
| 0x0D | 0x02 | 0x05 0x68 0x65 0x6C 0x6C 0x6F 
+------+------+-----+----+----+----+----+-----

   -+-----+----+----+----+----+----+-----
    | Value 1
   -+-----+----+----+----+----+----+-----
    | 0x10 0x05 0x77 0x6F 0x72 0x6C 0x64 
   -+-----+----+----+----+----+----+-----

   -+-----+----+-----+-----+----+----+----+-----+
    | Key 2          | Value 2                  |
   -+-----+----+-----+-----+----+----+----+-----+
    | 0x02 0x70 0x69 | 0x09 0xD0 0x0F 0x49 0x40 |
   -+-----+----+-----+-----+----+----+----+-----+
```

GUID
----
| Indicator | Type   | Data length |
|:----------|:-------|:------------|
| 0x0E      | GUID   | 16 bytes     |

**GUID** typed values are formatted starting with the type indicator byte, and followed by the 16 bytes of the value.


```
(Base64) MMl4yW6f30m3uqMhOdc2kw==:
+------+------+------+------+------+------+------+------+------+
| Ind. | Value                                                 
+------+------+------+------+------+------+------+------+------+
| 0x0E | 0xC9 | 0x78 | 0xC9 | 0x30 | 0x9F | 0x6E | 0x49 | 0xDF |
+------+------+------+------+------+------+------+------+------+

   -+------+------+------+------+------+------+------+------+
     ...Value (cont.)                                       |
   -+------+------+------+------+------+------+------+------+
    | 0xB7 | 0xBA | 0xA3 | 0x21 | 0x39 | 0xD7 | 0x36 | 0x93 |
   -+------+------+------+------+------+------+------+------+
```

Limitations
-----------
- There is currently ambiguity when keys are omitted when sending over the wire. It is unclear if the omission means set the value to null or there is no change in the value. The current work around is to have specific unset commands outside the object (usually booleans) that shows up in the API.

Implementations
---------------
- [Hummingbird-JS](https://github.com/echlo/hummingbird-js)
- [Hummingbird-Swift](https://github.com/echlo/hummingbird-swift)



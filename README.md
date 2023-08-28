# MUS Serialization Format
The MUS (**M**arshal, **U**nmarshal, **S**ize) format, which by the way is a 
binary format, tries to be as simple as possible. You won't find field names or 
any other information in it besides values (with few exceptions, as 'nil' 
pointer flag and length for string, list and map data types). So, for example, 
an object of type `Foo`:
```
type Foo {
  a int
  b bool
  c string
}
```
in MUS format may look like this:
```
Foo{a: 300, b: true, c: "hi"}    MUS->    [216 4 1 4 104 105]
, where 
- [216 4] - is the value of the field a
- [1] - value of the field b
- [4] - length of the field  c
- [104 105] - value of the field c
```

Note that the object fields are encoded in order, first the first, then the 
second, then third, and so on.

# Motivation
This approach provides:
1. A simple serializer that can be implemented quite easily and quickly for any 
   programming language.
2. And as we know, simple products are much easier to maintain + they usually 
   have fewer bugs.
3. The small number of bytes required to encode data. This means we can send 
   fewer bytes over the network or store fewer bytes on disk. And that's great 
   because I/O is often a performance bottleneck.

# Format Features
- All `uint` (`uint64`, `uint32`, `uint16`, `uint8`, `uint`), `int`, 
  `float` data types can be encoded in one of two ways: using Varint or Raw 
  encoding. The last one uses all the bytes that make up a number (in 
  LittleEndian format), so Varint encoding gives an advantage in the number 
  of used bytes, but is a bit slower.
  For example, the number `40` of type `uint64` is encoded with only one
  byte:
  ```
  40    Varint->    [40]
  ```
  , the same number in Raw encoding will take as much as 8 bytes:
  ```
  40    Raw->    [40 0 0 0 0 0 0 0]
  ```
  In general, Raw encoding uses for:
  - `int64, uint64, float64` - 8 bytes
  - `int32, uint32, float32` - 4 bytes
  - `int16, uint16` - 2 bytes
  - `int8, uint8` - 1 byte

  And only for large numbers it becomes more profitable than Varint, both in 
  speed and in the number of used bytes. These "large numbers", for `uint` 
  types, are:
  - \> 2^56 - for `uint64`
  - \> 2^28 - for `uint32`
  - \> 2^14 - for `uint16`
  - absent - for `uint8`, both encodings use one byte for `uint8` numbers

- ZigZag encoding is used for signed to unsigned integer mapping.
- Strings and lists are encoded with length (`int` type) and values, maps -
  with length and key/value pairs.
  ```
  string = "hello world"    MUS->    [22 104 101 108 108 111 32 119 111 114 108 100]
  , where
  - [22] - is the length of the string
  - [104 101 108 108 111 32 119 111 114 108 100] - string
  ```
  ```
  list = {"hello", "world"}    MUS->    [4 10 104 101 108 108 111 10 119 111 114 108 100]
  , where
  - [4] - length of the list
  - [10] - length of the first elem
  - [104 101 108 108 111] - first elem
  -	[10] - length of the second elem
  - [119 111 114 108 100] - second elem
  ```
  ```
  map = {1: "hello", 2: "world"}    MUS->    [4 2 10 104 101 108 108 111 4 10 119 111 114 108 100]
  , where
  - [4] - length of the map
  - [2] - first key
  - [10] - length of the first value
  - [104 101 108 108 111] - first value
  - [4] - second key
  - [10] - length of the second value
  - [119 111 114 108 100] - second value
  ```
- Booleans and bytes are encoded by a single byte.
  ```
  true    MUS->    [1]
  false    MUS->    [0]
  ```
- Pointers are encoded with `nil` flag: `nil` pointer is encoded as `1`, not 
  `nil` pointer as `0` + pointer value.
  ```
  nil    MUS->    [1]
  , where
  - [1] - nil flag
  ```
  ```
  &"hello world"    MUS->    [0 22 104 101 108 108 111 32 119 111 114 108 100]
  , where
  - [0] - nil flag
  - [22] - length of the string
  - [104 101 108 108 111 32 119 111 114 108 100] - string
  ```

# Data Type Metadata (DTM)
You can place a data type metadata in front of the data. Like in the 
[Data Versioning](#data-versioning) section.

# Data Versioning
MUS format does not have explicit data versioning support. But you can always do 
next:
```
// In this case DTM defines both the type and its version.
const FooV1DTM = 1
const FooV2DTM = 2

type FooV1 {
  // ...
}

type FooV2 {
  // ...
}

dtm, err = UnmarshalDTM(buf)
// ...
// Check DTM before Unmarshal.
switch dtm {
  case FooV1DTM:
    fooV1, err = UnmarshalFooV1(buf)
    // ...
  case FooV2DTM:
    fooV2, err = UnmarshalFooV2(buf)
    // ...
  default:
    return ErrUnsupportedDTM
}
```
Moreover, it is highly recommended to use DTM. With it, you will always be ready
for changes in the type structure or MUS format.

Thus, the MUS format suggests to have only one "version" mark for the entire 
type, instead of having a separate "version" mark for each field. And again, 
with this approach, we can send fewer bytes over the network or store fewer 
bytes on disk.

# Streaming
MUS format is suitable for streaming, all we need to know for this is the data 
type on the receiving side.

# Serializers
- [mus-go](https://github.com/mus-format/mus-go)
- [mus-stream-go](https://github.com/mus-format/mus-stream-go)
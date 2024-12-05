# MUS Serialization Format
MUS (Marshal, Unmarshal, Size) is a binary serialization format that tries to 
be as simple as possible. It does not include field names, types, or any 
additional metadata, except for a few cases: pointers are preceded by a pointer 
flag, and variable-length data types are preceded by length. For example, an 
object of type `Foo`:
```
type Foo {
  a int
  b bool
  c string
}
```
in the MUS format may look like this:
```
Foo{a: 300, b: true, c: "hi"}    MUS->    [216 4 1 4 104 105]
, where 
- [216 4] - is the value of the field a
- [1] - value of the field b
- [4] - length of the field  c
- [104 105] - value of the field c
```
Note that object fields are encoded in order, starting with the first, followed 
by the second, third, and so on.

# Motivation
This approach provides:
1. A simple serializer that can be quickly implemented in any programming 
   language. Simple products, as we know, are much easier to maintain and 
   usually contain fewer bugs.
2. And most importantly, the small number of bytes required to encode data. 
   Thanks to this, we can send fewer bytes over the network or use less disk 
   space. That's great because I/O is often a performance bottleneck.

# Format Features
- All `uint` (`uint64`, `uint32`, `uint16`, `uint8`, `uint`), `int`, 
  `float` data types can be encoded in one of two ways: using Varint or Raw 
  encoding. The last one uses all the bytes that make up a number (in 
  LittleEndian format) and is a little bit faster than Varint.
  For example, the number `40` of type `uint64` is encoded in Raw with 8 bytes:
  ```
  40    Raw->    [40 0 0 0 0 0 0 0]
  ```
  , and with only 1 byte in Varint:
    ```
  40    Varint->    [40]
  ```
  In general, Raw encoding uses for:
  - `int64, uint64, float64` - 8 bytes.
  - `int32, uint32, float32` - 4 bytes.
  - `int16, uint16` - 2 bytes.
  - `int8, uint8` - 1 byte.
  Only for large numbers it becomes more profitable than Varint, both in speed 
  and in the number of used bytes. These "large numbers", for `uint` types, are:
  - \> 2^56 - for `uint64`.
  - \> 2^28 - for `uint32`.
  - \> 2^14 - for `uint16`.
  - Absent - both encodings use one byte for `uint8` numbers.
- ZigZag encoding is used for signed to unsigned integer mapping.
- Positive integers, such as the length of a string, can be Varint encoded 
  without using ZigZag.
- Strings and lists are encoded by length (`int` type with non-fixed encoding - 
  can be Varint or Raw) and their value, maps - by length and key/value pairs.
  ```
  string = "hello world"    MUS->    [22 104 101 108 108 111 32 119 111 114 108 100]
  , where
  - [22] - length of the string
  - [104 101 108 108 111 32 119 111 114 108 100] - string value
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
- Booleans and bytes are each encoded using a single byte.
  ```
  true    MUS->    [1]
  false    MUS->    [0]
  ```
- Pointers are encoded with the pointer flag: the `nil` pointer is represented 
  as `1`, not `nil` as `0` + pointer value.
  ```
  nil    MUS->    [1]
  , where
  - [1] - pointer flag
  ```
  ```
  &"hello world"    MUS->    [0 22 104 101 108 108 111 32 119 111 114 108 100]
  , where
  - [0] - pointer flag
  - [22] - length of the string
  - [104 101 108 108 111 32 119 111 114 108 100] - string value
  ```
  The value `2` of the pointer flag denotes the pointer mapping. In this case, 
  all occurrences of the pointer are encoded as `2` followed by the pointer ID 
  (Varint), with only the first occurrence containing the pointer value.
  ```
  ptr1 = &10
  ptr2 = &20
  list = {ptr1, ptr1, ptr2}     MUS->     [6 2 0 20 2 0 2 2 40]
  , where
  - [6] - length of the list
  - [2] - pointer flag
  - [0] - ptr1 ID
  - [20] - ptr1 value
  - [2] - pointer flag
  - [0] - ptr1 ID
  - [2] - pointer flag
  - [2] - ptr2 ID
  - [40] - ptr2 value
  ```

# Data Type Metadata (DTM)
To decode data, we need to know its type. To explicitly indicate it, use a DTM,
which can simply be a number:
```
DTM + data
```

# Data Versioning
The MUS format does not support data versioning. However, it can be achieved 
using a DTM:
```
// In this case DTM indicates both the data type and version.
const FooV1DTM = 1
const FooV2DTM = 2

type FooV1 { ... }

type FooV2 { ... }

dtm, err = UnmarshalDTM(buf)
...
// Check the DTM before unmarshaling the data.
switch dtm {
  case FooV1DTM:
    fooV1, err = UnmarshalFooV1(buf)
    ...
  case FooV2DTM:
    fooV2, err = UnmarshalFooV2(buf)
     ...
  default:
    return ErrUnexpectedDTM
}
```
Moreover, it is highly recommended to use a DTM. With it, you will always be 
prepared for changes in the type structure or the MUS format itself.

Thus, the MUS format suggests having only one "version" mark for the entire 
type, instead of separate "version" marks for each field.

# Streaming
MUS format is suitable for streaming, all we need to know for this is the data 
type on the receiving side.

# Serializers
- [mus-go](https://github.com/mus-format/mus-go)
- [mus-stream-go](https://github.com/mus-format/mus-stream-go)
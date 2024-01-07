# Rows of String Values (RSV Data Format) Specification

The RSV data format is a simple binary alternative to CSV.
This document describes the data structure behind RSV and defines the encoding used.
If you want to watch an explanation video about RSV, you can watch the following YouTube video:
[RSV - Rows of String Values - A Simple Binary Alternative to CSV](https://www.youtube.com/watch?v=tb_70o6ohMA)

## RSV Data Structure

An RSV document represents an array of arrays of nullable string values, also called a jagged array.

```
Array<Array<String|Null>>
```

It's main purpose is to store tabular data. But because it's a jagged array, it's not limited to that.
So, rows can contain the same number of values, but don't have to.

### Tabular Data

Here is an example of tabular data:

| FirstName | LastName | Age          | PlaceOfBirth  |
| --------- | -------- | ------------ | ------------- |
| William   | Smith    | 30           | Boston        |
| Olivia    | Jones    | 27           | San Francisco |
| Lucas     | Brown    | &lt;Null&gt; | Chicago       |

We have four columns, a header row, and three data rows.
There is one age value missing, so we have a null value.

Expressed as a JSON document, our example would look like this:

```json
[
  ["FirstName", "LastName", "Age", "PlaceOfBirth"],
  ["William",   "Smith",    "30",  "Boston"],
  ["Olivia",    "Jones",    "27",  "San Francisco"],
  ["Lucas",     "Brown",    null,  "Chicago"]
]
```

We can see the array of arrays, our string values, and the null value.

### Non-Tabular Data

Now let's have a look at an example of non-tabular data.

```
+------+
| 2D   |
+------+----+----+----+----+----+----+----+----+
| Pts  |  1 |  1 |  1 | -1 | -1 | -1 | -1 |  1 |
+------+----+----+----+----+----+----+----+----+
| Tris |  0 |  1 |  2 |  2 |  3 |  0 |
+------+----+----+----+----+----+----+
```

Here we have a small 2D vector graphics format, that describes a list of four points
and a list of two triangles indexing the previously defined points.
In this example we can clearly see, that the number of values per row can vary.

Again as JSON-document, the example would look like this:

```json
[
  ["2D"],
  ["Pts", "1", "1", "1", "-1", "-1", "-1", "-1", "1"],
  ["Tris", "0", "1", "2", "2", "3", "0"]
]
```
### Value Types

Notice in the previous example, that numbers are also represented as strings,
because RSV does not differentiate between different data types like boolean or numbers,
but instead has only string or null values.

```json
["Tris", "0", "1", "2", "2", "3", "0"]
```

The interpretation of these values is up to the program,
or depends on the custom data format, which was build upon RSV.

Not having special data types makes the format both simple and universal,
because you can represent every data type as string,
without worrying about precision (int32, int64, ...) or aspects like endianness (little/big).
It also makes writing code to read and write RSV documents really easy.

## RSV Encoding

Now that we have an idea about the type of data we can represent with RSV,
let's have a look at how an RSV document is actually encoded.

### Differences to CSV

A first difference to CSV is that RSV is a binary format
and not a textual format like CSV, which means an RSV file is not meant
to be opened with a text editor. Doing so will result in the display
of weird characters, and thus is not recommended.

This is a typical property of binary formats, which sacrifice
this sort of readability in order to offer other benefits
like better reading and writing performance,
or not having you to write a complex parser.
In case of RSV it has a nice advantage, which we will now have a look at.

In order for a textual data format
like CSV or TSV to indicate where
one value ends and another value begins,
a delimiter character must be defined.
In case of CSV this might be a comma,
or a semicolon, or in case of TSV
a simple tab character.

```
a,b,c
a;b;c
a→b→c 
```

Another delimiter is also needed, to indicate the end of one row,
and the beginning of the following row.
In case of CSV or TSV, this might be a single line break
character like a line feed (LF) or a combination using the carriage return character (CR).

```
a,b,c<LF>    vs.   a,b,c<CR><LF>
d,e                d,e
```

This works perfectly well, as long as the values themselves don't contain
these delimiters. But this is kind of fragile, because what happens when they do contain these delimiters?
This is refered to as [delimiter collision](https://en.wikipedia.org/wiki/Delimiter#Delimiter_collision).
And one solution for it is to use some sort of escape sequences, so that you can differentiate those cases.

For CSV a common approach is to put values containing commas or line breaks
inside of double quotes:

```
"a,a",b,c
```

But then again the double quote character itself becomes a delimiter, and you must handle the case
when a value contains it.

```
"a,""a",b,c
```

### Avoiding Delimiter Collision

This all comes with a processing cost, and raises the question,
if we could avoid delimiter collision at all?

And yes, with a binary format, we can. With a binary format we can simply write the length
of a string value before the actual data. But that approach might significantly
increase the size of the resulting file, if a fixed-width encoding is used
for the length values, or makes the format more complicated when you need to think about encoding
the length values with a variable-width encoding scheme (VarInts), to only use a minimal
number of bytes.

You also need to write a value that indicates how many values a row contains,
or how many bytes it will take. But this approach might limit the flexibility of appending
values to an existing file.

So that's why RSV does not follow this approach.

### RSV's Special Bytes

Instead of using length values prefixed, RSV uses special bytes
to indicate the end of a value (EOV), or the end of a row (EOR).

These terminating bytes are possible, because RSV strings are Unicode strings,
and are encoded using [UTF-8](https://en.wikipedia.org/wiki/UTF-8#Encoding).

| From    | To       | Byte 1   | Byte 2   | Byte 3   | Byte 4   |
| ---     | ---      | ---      | ---      | ---      | ---      |
| U+0000  | U+007F   | 0xxxxxxx | -        | -        | -        |
| U+0080  | U+07FF   | 110xxxxx | 10xxxxxx | -        | -        |
| U+0800  | U+FFFF   | 1110xxxx | 10xxxxxx | 10xxxxxx | -        |
| U+10000 | U+10FFFF | 11110xxx | 10xxxxxx | 10xxxxxx | 10xxxxxx |

With the UTF-8 encoding and it's specific
byte pattern, there is a range of bytes, that will never be produced by the encoding scheme.

| Binary       | Hex  | Decimal |
| ---          | ---  | ---     |
| 11111**111** | FF   | 255     |
| 11111**110** | FE   | 254     |
| 11111**101** | FD   | 253     |
| 11111**100** | FC   | 252     |
| 11111**011** | FB   | 251     |
| 11111**010** | FA   | 250     |
| 11111**001** | F9   | 249     |
| 11111**000** | F8   | 248     |

These bytes, which are invalid UTF-8 bytes,
invalid, because they should be rejected by a UTF-8 decoder,
can be used in a binary format to convey special meaning.

RSV uses three of these special bytes, which are the bytes 253, 254, and 255.

```
11111111  FF  255  =  Value Terminator Byte
11111110  FE  254  =  Null Value Byte
11111101  FD  253  =  Row Terminator Byte
```

Byte 255 is used to terminate a value, byte 254 signals a null value,
and byte 253 is used to terminate a row.
These three bytes will never collide with any byte-value of a UTF-8 encoded string value.
So we completely get rid of delimiter collision, and don't have to check our string values
for special characters and don't need to resort to any kind of escape sequences.

### Terminating vs. Separating

So we've learned that two bytes are terminating bytes.
This aspect of RSV is also a big difference to CSV or Comma-Separated Values,
where, like the name already tells us, values are separated by a delimiter,
instead of being terminated by. Having every row of an RSV file
being terminated by our special byte, has the nice advantage,
that RSV files can simply be concatenated. It also makes the special case
of an empty RSV file without any rows possible:


```json
[
]
```

## Simple RSV Example

Now that we know the basics of the RSV encoding, let's have a look
at a simple example:

```json
[
  ["Hello", "🌎"]
]
```

Here we have a single row, containing two strings. First the string 'hello' and then a second string,
simply containing a world emoji.

The encoding process would look like this. First, we write the UTF-8 bytes of
the hello string. All 5 characters are ASCII characters, so there is nothing special here:

```
72 101 108 108 111
```

We terminate the value with byte 255 and go on to the next string:

```
72 101 108 108 111|255
                   ^^^
```

The world emoji is a Unicode supplementary character that requires 4 bytes
when encoded using UTF-8:

```
72 101 108 108 111|255|240 159 140 142
                       ^^^ ^^^ ^^^ ^^^
```

Again we terminate the value with byte 255:

```
72 101 108 108 111|255|240 159 140 142|255
                                       ^^^
```

And because we've reached the end of the row, we terimate it using our special byte 253.
And this is our resulting byte sequence for our simple RSV document:

```
72 101 108 108 111|255|240 159 140 142|255|253
                                           ^^^
```

Now let's add two other rows to our document, the first being empty,
and the second containing a null value and an empty string value:

```json
[
  ["Hello", "🌎"],
  [],
  [null, ""]
]
```

The empty row would simply add another byte 253 to our byte sequence:

```
72 101 108 108 111|255|240 159 140 142|255|253|253
                                               ^^^
```

And the last row would encode like this. For the null value we add special byte 254,
and terminate it with byte 255:

```
72 101 108 108 111|255|240 159 140 142|255|253|253|254|255
                                                   ^^^ ^^^
```
The empty string value is simply our value terminating byte 255:

```
72 101 108 108 111|255|240 159 140 142|255|253|253|254|255|255
                                                           ^^^
```

Finally, we terminate the row with byte 253:
```
72 101 108 108 111|255|240 159 140 142|255|253|253|254|255|255|253
                                                               ^^^
```
And that's basically it:
```
72 101 108 108 111|255|240 159 140 142|255|253|253|254|255|255|253
H  e   l   l   o   EOV                 EOV EOR EOR NUL EOV EOV EOR
```

Really simple and straight forward, and easy to implement,
both for writing and reading.

## RSV Implementations

### RSV Encoder

A simple RSV encoder can be written in Python with only 9 lines of code:
```python
def encode_rsv(rows: list[list[str | None]]) -> bytes:
	parts: list[bytes] = []
	for row in rows:
		for value in row:
			if value is None: parts.append(b"\xFE")
			elif len(value) > 0: parts.append(value.encode())
			parts.append(b"\xFF")
		parts.append(b"\xFD")
	return b"".join(parts)
```
And be used like this, which will give us exactly
our example file, which we encoded previously by hand:

```python
def save_rsv(rows: list[list[str | None]], file_path: str):
	with open(file_path, "wb") as file:
		file.write(encode_rsv(rows))

rows = [
	["Hello", "🌎"],
	[],
	[None, ""]
]
save_rsv(rows, "Example.rsv")
```

### RSV Decoder

Decoding then simply is a matter of iterating over the encoded bytes
and detecting the special terminator bytes, as well as the special null value byte:
```python
def decode_rsv(bytes: bytes) -> list[list[str | None]]:
	if len(bytes) > 0 and bytes[-1] != 0xFD:
		raise Exception("Incomplete RSV document")
	result: list[list[str | None]] = []
	current_row: list[str | None] = []
	value_start_index = 0
	for i in range(len(bytes)):
		if bytes[i] == 0xFF:
			length = i - value_start_index
			if length == 0:
				current_row.append("")
			elif length == 1 and bytes[value_start_index] == 0xFE:
				current_row.append(None)
			else:
				value_bytes = bytes[value_start_index:i]
				current_row.append(value_bytes.decode())
			value_start_index = i + 1
		elif bytes[i] == 0xFD:
			if i > 0 and value_start_index != i:
				raise Exception("Incomplete RSV row")
			result.append(current_row)
			current_row = []
			value_start_index = i + 1
	return result
```

So loading an RSV file simply becomes this:

```python
def load_rsv(file_path: str) -> list[list[str | None]]:
	with open(file_path, "rb") as file:
		return decode_rsv(file.read())
	
loaded = load_rsv("Example.rsv")
print(loaded) 
```

### RSV-Challenge Repository

If you wanna try out RSV then you can checkout the [RSV-Challenge repository](https://github.com/Stenway/RSV-Challenge), where you can find the basic encoding and decoding functions,
as well as saving and loading functions ported to over 30 different programming languages.

If you think there is an important programming language missing,
which is not too esoteric, simply open an issue in the repository.

## Use Cases for RSV

You can use RSV as an alternative to CSV or TSV. You can also use it as a binary alternative
to streaming oriented formats like [JSON lines](https://jsonlines.org/),
where you could put multiple textual documents like JSON, YAML or XML documents
into a single file. A simple configuration file would also be a use case.
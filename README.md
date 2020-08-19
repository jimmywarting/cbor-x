# msgpackr

The msgpackr package is an extremely fast MessagePack NodeJS/JavaScript implementation. At the time of writing, it is several times faster than any other known implementations, faster than Avro (for JS), and generally even faster than native JSON.stringify/parse. It also includes an optional record extension (the `r` in msgpackr), for defining record structures that makes MessagePack even faster and more compact, often over twice as fast as even native JSON methods and many times faster than other JS implementations.

## Basic Usage

Install with:

```
npm install msgpackr
```
And import or require it for basic unpack and pack functions:
```
import { unpack, pack } from 'msgpackr';
let packdAsBuffer = pack(value);
let data = unpack(packdAsBuffer);
```
This `pack` function will generate standard MessagePack without any extensions that should be compatible with any standard MessagePack serializer. It will pack JavaScript objects as MessagePack maps by default. The `unpack` function will deserialize MessagePack maps as an `Object` with the properties from the map.

# Record / Object Structures
There is a critical difference between maps (or dictionaries) that hold an arbitrary set of keys and values (JavaScript `Map`s are optimal for this), and records or object structures that have a well-defined set of fields which may have many instances using that same structure (most objects in JS). By using the record extension, this distinction is preserved in MessagePack and the encoding can reuse structures and not only provides better type preservation, but yield signficantly more compact encodings and increase parsing/deserialization performance by 2-3x. Msgpackr automatically generates record definitions that are reused and referenced by objects with the same structure. There are a number of ways to use this to our advantage. For large object structures with repeating nested objects with similar structures, simply serializing with the record extension can yield benefits. To use the record structures extension, we create a new `Packr` instance. By default a new `Packr` instance will have the record extension enabled:
```
import { Packr } from 'msgpackr';
let packr = Packr();
packr.pack(myBigData);

```

Another way to further leverage the benefits of the msgpackr record structures is to use streams that naturally allow for data to reuse based on previous record structures. We can use the `PackrStream` instance that will have also the record extension enabled by default:

```
import { PackrStream } from 'msgpackr';
let stream = PackrStream();
stream.write(myData);

```
Or for a full example of sending and receiving data on a stream
```
import { PackrStream } from 'msgpackr';
let sendingStream = PackrStream();
let receivingStream = UnpackrStream();
// we just piping to our own stream, but normally you would send and
// receive over some type of inter-process or network connection.
sendingStream.pipe(receivingStream);
sendingStream.write(myData);
receivingStream.on('data', (data) => {
	// received data
});
```
When creating a new `Packr`, `PackrStream`, or `UnpackrStream` instance, we can enable or disable the record structure extension with the `objectsAsMaps` property. When this is `true`, the record structure extension will be disabled, and all objects will be serialized using MessageMap maps, and all maps will be deserialized to JS `Object`s as properties.

## Shared Record Structures
Another useful way of using msgpackr, and the record extension, is for storing data in a databases, files, or other storage systems. If a number of objects with common data structures are being stored, a shared structure can be used to greatly improve data storage and deserialization efficiency. We just need to provide a way to store the generated shared structure so it is available to deserialize stored data in the future:

```
import { Packr } from 'msgpackr';
let packr = Packr({
	getStructures() {
		// storing our data in file (but we could also store in a db or key-value store)
		return unpack(readFileSync('my-shared-structures.mp')) || [];
	},
	saveStructures(structures) {
		writeFileSync('my-shared-structures.mp', pack(structures))
	},
	structures: []
});

```

### resetMemory
During the serialization process, data is written to buffers. Allocating new buffers is a relatively expensive process, and the `resetMemory` method can help allow reuse of buffers that will further improve performance. The `resetMemory` method can be called when previously created buffer(s) are no longer needed. For example, if we serialized an object, and wrote it to a database, we could indicate that we are done:
```
let buffer = packr.pack(data);
writeToStorageSync(buffer);
// finished with buffer, we can reset the memory on our packr now:
packr.resetMemory()
// future serialization can now reuse memory for better performance
```
The use of `resetMemory` is never required, buffers will still be handled and cleaned up through GC if not used, it just provides a small performance boost.

## Performance
msgpackr is fast. Really fast. Here is comparison with the next fastest JS projects. It also includes comparison to V8 native JSON functionality, and JavaScript Avro (`avsc`, a very optimized Avro implementation):

operation                                                  |   op   |   ms  |  op/s
---------------------------------------------------------- | ------: | ----: | -----:
buf = Buffer(JSON.stringify(obj));                         |   82000 |  5004 |  16386
obj = JSON.parse(buf);                                     |   88600 |  5000 |  17720
require("msgpackr").pack(obj);                             |  161500 |  5002 |  32287
require("msgpackr").unpack(buf);                           |   94600 |  5004 |  18904
msgpackr w/ shared structures: packr.pack(obj);            |  178400 |  5002 |  35665
msgpackr w/ shared structures: packr.unpack(buf);          |  376700 |  5000 |  75340
buf = require("msgpack-lite").encode(obj);                 |   30100 |  5012 |   6005
obj = require("msgpack-lite").decode(buf);                 |   16200 |  5001 |   3239
buf = require("notepack").encode(obj);                     |   62600 |  5005 |  12507
obj = require("notepack").decode(buf);                     |   32400 |  5007 |   6470
require("what-the-pack")... encoder.encode(obj);           |   63500 |  5002 |  12694
require("what-the-pack")... encoder.decode(buf);           |   32000 |  5001 |   6398
require("avsc")...make schema/type...type.toBuffer(obj);   |   84600 |  5003 |  16909
require("avsc")...make schema/type...type.toBuffer(obj);   |   99300 |  5001 |  19856

(`avsc` is schema-based and more comparable in style to msgpackr with shared structures).

### Record Structure Extension Definition
The record struction extension uses extension id 0x72 ("r") to declare the use of this functionality. The extension "data" byte (or bytes) identifies the byte or bytes used to identify the start of a record in the subsequent MessagePack block or stream. The identifier byte (or the first byte in a sequence) must be from 0x40 - 0x7f (and therefore replaces one byte representations of positive integers 64 - 127). The extension decaration must be immediately follow by an MessagePack array that defines the field names of the record structure.

Once a record identifier and record field names have been defined, the parser/decoder should proceed to read the next value. Any subsequent use of the record identifier as a value in the block or stream should parsed as a record instance, and the next n values, where is n is the number of fields (as defined in the array of field names), should be read as the values of the fields. For example, here we have defined a structure with fields "foo" and "bar", with the record identifier 0x40, and then read a record instance that defines the field values of 4 and 2, respectively:
```
+--------+--------+--------+~~~~~~~~~~~~~~~~~~~~~~~~~+--------+--------+--------+
|  0xd4  |  0x72  |  0x40  | array: [ "foo", "bar" ] |  0x40  |  0x04  |  0x02  |
+--------+--------+--------+~~~~~~~~~~~~~~~~~~~~~~~~~+--------+--------+--------+
```
Which should generate an object that would correspond to JSON:
```
{ "name" : 4, "bar": 2}
```


### License

MIT!
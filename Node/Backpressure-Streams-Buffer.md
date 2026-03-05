https://www.geeksforgeeks.org/node-js/node-js-streams/
https://betterstack.com/community/guides/scaling-nodejs/nodejs-streams/
https://dev.to/ayako_yk/understanding-streams-in-nodejs-from-piping-to-backpressure-1ncf

1. What is a Buffer in Node.js?

A Buffer in Node.js is a raw memory allocation outside the V8 heap used to store binary data.

A Buffer in Node.js is a fixed-size memory allocation used to handle binary data directly outside the V8 heap, enabling efficient processing of streams, files, and network protocols.

JavaScript normally handles UTF-16 strings, but many real-world operations deal with binary streams, such as:

files
network packets
TCP streams
encryption keys
images
video
database binary data

Buffers allow Node.js to efficiently manipulate binary data without converting it to strings.

2. Why Buffers Exist in Node.js
Node.js was built for I/O heavy applications.
Most I/O operations deal with streams of bytes, not JavaScript objects.

| Operation       | Data Type |
| --------------- | --------- |
| Reading file    | bytes     |
| HTTP response   | bytes     |
| TCP socket      | bytes     |
| Image upload    | binary    |
| encryption      | binary    |
| video streaming | binary    |

Without Buffers, Node would need to convert everything to strings, which is slow and memory heavy.

Buffers allow:
zero-copy memory usage
efficient streaming
low-level networking

3. Where Buffers Are Used Internally in Node.js
Buffers are used in many Node.js core modules:

| Node Module | Buffer Usage          |
| ----------- | --------------------- |
| **fs**      | reading files         |
| **net**     | TCP sockets           |
| **http**    | request/response body |
| **tls**     | encryption            |
| **crypto**  | hashing/encryption    |
| **stream**  | chunked data          |
| **zlib**    | compression           |


Example:
fs.readFile("file.txt", (err, data) => {
  console.log(data); 
});

Output:
<Buffer 48 65 6c 6c 6f>

4. Buffer vs Array vs TypedArray
| Feature           | Buffer | Array | TypedArray |
| ----------------- | ------ | ----- | ---------- |
| Stores binary     | yes    | no    | yes        |
| fixed size        | yes    | no    | yes        |
| optimized for I/O | yes    | no    | yes        |
| outside V8 heap   | yes    | no    | no         |
| Node specific     | yes    | no    | no         |

Buffer extends:
Uint8Array

5. Buffer Memory Model
Node uses:
C++ memory allocation
Buffers are allocated in native memory.

Benefits:
faster I/O
avoids garbage collector overhead
efficient streaming

But misuse can cause memory leaks.

6. Creating Buffers
1. Buffer.alloc()
Safe allocation.

const buf = Buffer.alloc(10);
Creates a buffer filled with zeros.

2. Buffer.allocUnsafe()
Fast allocation but not initialized.

const buf = Buffer.allocUnsafe(10);
Memory may contain old data.

3. Buffer.from()
Create buffer from existing data.

Buffer.from("hello");
Buffer.from([1,2,3]);
Buffer.from(arrayBuffer);

7. Buffer Example
const buf = Buffer.from("hello");

console.log(buf);
Output
<Buffer 68 65 6c 6c 6f>

Each byte represents ASCII value.

8. Encoding and Decoding
Buffers can convert between:
binary
strings
base64
hex

Example:
const buf = Buffer.from("hello");
console.log(buf.toString());

Output
hello

9. Important Buffer Encodings
| Encoding | Use           |
| -------- | ------------- |
| utf8     | default       |
| ascii    | old protocols |
| base64   | APIs          |
| hex      | crypto        |
| binary   | raw data      |

12. Buffers in Streams

Buffers are heavily used in streams.

Example:
const fs = require("fs");

const stream = fs.createReadStream("file.txt");

stream.on("data", chunk => {
  console.log(chunk);
});

chunk is a Buffer.

5. Buffers in File Uploads
Example:
Uploading images.

Client → HTTP request → Node server
File chunks are buffers.
Libraries:
multer
busboy

6. Buffers in Banking Applications
Buffers are used in:

1. Encryption
Sensitive data encryption.

Example:

Account numbers
Passwords
Tokens

Node crypto uses buffers.

2. Payment Gateway APIs
Binary signing.

HMAC
RSA
SHA256

Buffers hold encrypted payloads.

3. ISO 8583 Banking Protocol
Bank messages are binary packets.
Buffers parse them.

Example structure:

Header
Bitmap
Data fields

4. Secure File Transfers
Banks transfer files via:

SFTP
SWIFT
ACH

Buffers process file streams.

17. Buffers in Insurance Systems

Used in:
Document storage
PDF uploads.

Policy documents
Claims images
Medical scans

Handled as buffers.
19. Performance Benefits

| Benefit              | Explanation          |
| -------------------- | -------------------- |
| Zero copy            | avoid copying memory |
| Faster streams       | chunk processing     |
| lower GC overhead    | outside heap         |
| efficient networking | raw packets          |

20. Buffer Best Practices
Use alloc instead of allocUnsafe
Unsafe may expose old memory.

Buffer.alloc()

23. Advanced Concept – Buffer Pool
Node internally maintains buffer pool.
Small buffers come from shared pool.

Benefits:
faster allocation
less memory fragmentation

1. UTF-16 Strings
Definition
UTF-16 is the internal character encoding used by the JavaScript engine (V8) to represent strings.

In JavaScript:
const str = "hello";

Internally this is stored in UTF-16 encoding, not ASCII or UTF-8.

Why UTF-16?
Because JavaScript must support Unicode characters like:

emojis
Chinese characters
Arabic
special symbols

UTF-16 uses 2 bytes per character (sometimes 4 for surrogate pairs).

Character storage in memory:
| Character | UTF-16 bytes |
| --------- | ------------ |
| h         | 2 bytes      |
| e         | 2 bytes      |
| l         | 2 bytes      |
| l         | 2 bytes      |
| o         | 2 bytes      |

So:
"hello" = 10 bytes
But in UTF-8
"hello" = 5 bytes

Why this matters in Node.js
Most network protocols and files use UTF-8 or binary, not UTF-16.
So Node must convert between UTF-16 and binary.

Example:
Buffer.from("hello")

UTF-16 string
      ↓
converted to
      ↓
UTF-8 binary
      ↓
stored in Buffer

JavaScript stores strings in UTF-16 internally, but most I/O operations use UTF-8 or binary formats. Node.js Buffers allow efficient conversion between these formats when handling files, streams, and network packets.

2. Binary Streams
Definition
A binary stream is a continuous flow of raw bytes instead of structured objects or strings.

Examples:
video streaming
file downloads
TCP packets
image uploads
audio streaming

These streams consist of chunks of binary data.

3. V8 Heap
Definition
The V8 heap is the memory region used by the V8 JavaScript engine to store JavaScript objects.

Examples stored in heap:
objects
arrays
strings
functions
closures

The V8 heap is the memory space used by the JavaScript engine to store JavaScript objects. Node.js Buffers allocate memory outside the V8 heap to efficiently handle large binary data without affecting garbage collection performance.

A stream pipeline is a chain of stream operations that process data incrementally in chunks instead of loading everything into memory.

Source → Transform → Compress → Encrypt → Destination
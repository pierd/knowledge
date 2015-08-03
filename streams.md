# Streams
Streams are a control flow abstraction native to node to read, transform and
write data. Streams are composable, efficient and _easy_! Yes, really. As a
stream consumer, you can consider streams to be the simpler counterpart to
Promises. This section is meant to give you a pragmatic introduction to all
aspects of streams. If you're looking for a guide that covers _all_ aspects,
check out the [stream handbook](https://github.com/substack/stream-handbook) by
substack.

- [basics](#basics)
- [read-transform-write](#read-transform-write)
- [modules](#modules)
- [internals](#internals)
- [creating streams](#creating-streams)
- [testing streams](#testing-streams)
- [resources](#see-also)

## basics
There are 4 types of streams:
- __read__: data can be read from (e.g. `fs.createReadStream`)
- __write__: data can be written to (e.g. `fs.createWriteStream`)
- __duplex__: data can be read from and data can be written to (e.g. websockets)
- __transform__: data can be written to, transformed and then read from (e.g.
  `gzip`)

Streams originate from shell, where commands can be piped together using the
`|` (pipe) character. Take this copy function:
```sh
$ cat ./my-file > ./other-file
```

Now using node streams:
```js
const fs = require('fs')

fs.createReadStream('./my-file')
  .pipe(fs.createWriteStream('./other-file'))
```

The stream reads from a file (`./my-file`) using a read stream and writes to a
file (`./other-file`) using a write stream. The speed is throttled to the
slowest operation (probably harddisk writes), and data is only requested when
the harddisk is ready to write a new chunk making it super efficient.

### memory
Using streams reduces the amount of memory required by a node program, making
the program faster and less prone to crash. Using the regular `fs.readFile` /
`fs.readFileSync` api's with `'utf8'` will devour resources and eventually
crash on large files.

## read-transform-write
In shell / unix the most common operation is to read data from a source,
transform it, and then write it back out. In shell:
```sh
# read file, find lines containing `foo`, echo to stdout
$ cat ./my-file | grep 'foo'
```

The same program using streams:
```js
const through = require('through2')
const fs = require('fs')

fs.createReadStream('./my-file')
  .pipe(through(grep(/foo/)))
  .pipe(process.stdout)

// grep `transform` stream
// regex -> (buf, str, fn) -> null
function grep (regex) {
  return (chunk, enc, cb) => {
    if (regex.test(chunk.toString())) this.push(chunk)
    cb()
  }
}
```

This program does the same as the bash function. The reason why the code looks
more verbose is because:

1. we're importing dependencies explicitely rather than relying on globals
2. we implemented the `grep` function from scratch

To make sure you understand what's going on, let's break down the code:

1. we import the `fs` and `through2` modules (more on stream modules later)
2. we create a read stream from `./my-file`
3. we connect the read stream to a transform stream that filters our data
4. we connect our filtered data to `stdout`

The `grep` function works as follows:

1. `grep` takes a regex and returns a function that can be consumed by
   `through2`
2. `through2` creates a transform stream from a function
3. the function in `through2` gets a chunk (buffer), encoding (string) and
   callback (function) passed in
4. if the stringified buffer (`.toString()`) passed the regex check, push it to
   transform's stream read stream (e.g. to the next `.pipe()` function)
5. call the callback once done so the next chunk can be processed

And that's it! You should now understand what a read stream is, how to create
transform streams, and how to pipe data back out to either `stdout` or a file.

### modules
By design Node only exposes bare essentials, leaving much of the staple
functionality to live in userland (npm). If you're working with streams, these
are probably the packages you want to be using.

_These descriptions were extracted from
[mississippi](https://github.com/maxogden/mississippi), a convenience wrapper
for common streaming functions._

### [pump](https://github.com/mafintosh/pump)
#### `pump(stream1, stream2, stream3, ..., cb)`
Pipes streams together and destroys all of them if one of them closes. Calls
`cb` with `(error)` if there was an error in any of the streams.

When using standard `source.pipe(destination)` the source will _not_ be
destroyed if the destination emits close or error. You are also not able to
provide a callback to tell when then pipe has finished.

`pump` does these two things for you, ensuring you handle stream errors
100% of the time (unhandled errors are probably the most common bug in most
node streams code)

```js
// lets do a simple file copy
var fs = require('fs')

var read = fs.createReadStream('./original.zip')
var write = fs.createWriteStream('./copy.zip')

// use pump instead of read.pipe(write)
pump(read, write, function (err) {
  if (err) return console.error('Copy error!', err)
  console.log('Copied successfully')
})
```

### [pumpify](https://github.com/mafintosh/pumpify)
#### `pipeline = pumpify(stream1, stream2, stream3, ...)`
Builds a pipeline from all the transform streams passed in as arguments by
piping them together and returning a single stream object that lets you write
to the first stream and read from the last stream

If any of the streams in the pipeline emits an error or gets destroyed, or you
destroy the stream it returns, all of the streams will be destroyed and
cleaned up for you

```js
// first create some transform streams (note: these two modules are fictional)
const imageResize = require('image-resizer-stream')({width: 400})
const pngOptimizer = require('png-optimizer-stream')({quality: 60})

// instead of doing a.pipe(b), use pipeline
const resizeAndOptimize = pumpify(imageResize, pngOptimizer)
// `resizeAndOptimize` is a transform stream. when you write to it, it writes
// to `imageResize`. when you read from it, it reads from `pngOptimizer`.
// it handles piping all the streams together for you

// use it like any other transform stream
const fs = require('fs')

const read = fs.createReadStream('./image.png')
const write = fs.createWriteStream('./resized-and-optimized.png')

pump(read, resizeAndOptimize, write, function (err) {
  if (err) return console.error('Image processing error!', err)
  console.log('Image processed successfully')
})
```

### [duplexify](https://github.com/mafintosh/duplexify)
#### `duplex = duplexify([writable, readable, opts])`
Take two separate streams, a writable and a readable, and turn them into a
single duplex (readable and writable) stream

The returned stream will emit data from the readable, and when you write to it,
it writes to the readable

You can either choose to supply the writable and the readable at the time you
create the stream, or you can do it later using the `.setWritable` and
`.setReadable` methods, and data written to the stream in the meantime will be
buffered for you

```js
const stdio = duplexify(process.stdout, process.stdin)
```

### [through2](https://github.com/rvagg/through2)
#### `transform = through2(fn(chunk, enc, cb))`
`through2` is a module that allows you to create a transform stream from a
function, rather than a full prototype chain. It can operate on either buffers
or objects if `opt.objectMode = true`.
```js
const through = require('through2')
const csv = require('csv-parser')
const fs = require('fs')

fs.createReadStream('data.csv')
  .pipe(csv())
  .pipe(through.obj(throughFn))
  .pipe(process.stdout)

function throughFn (chunk, enc, cb) {
  const data = {
    name: chunk.name,
    address: chunk.address
  }
  cb(null, data)
}
```

### [concat-stream](https://github.com/maxogden/concat-stream)
#### `file = concatStream(fn(data))`
Sometimes you need to operate on a full file rather than individual chunks.
`concat-stream` concatenates the full content of a stream and then calls the
callback. It is not a transform stream, more of a stream sink.

```js
const concatStream = require('concat-stream')
const fs = require('fs')

fs.createReadStream('./cat.jpg')
  .pipe(concatStream((data) => {
      // data is `cat.jpg` as a buffer
    }))
```

## internals
[tbi]

## creating streams
[tbi]

## testing streams
[tbi]

## See Also
- [stream handbook](https://github.com/substack/stream-handbook) - stream guide by substack
- [mississippi](https://github.com/maxogden/mississippi) - a collection of useful stream utility modules
- [iojs/api/streams](https://iojs.org/api/stream.htm) - iojs streams documentation
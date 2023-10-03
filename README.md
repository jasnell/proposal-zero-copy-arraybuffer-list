# Zero-copy ArrayBuffer lists

A TC39 proposal for creating a concatenated `ArrayBuffer` instance from a list of other `ArrayBuffer` instances without copy or resize.

Stage: Not-yet-submitted

## Table of Contents

## Authors

* [James M Snell](https://github.com/jasnell)

## Participate

* [GitHub repository](https://github.com/jasnell/proposal-zero-copy-arraybuffer-list)
* [Issue tracker](https://github.com/jasnell/proposal-zero-copy-arraybuffer-list/issues?q=is%3Aissue+is%3Aopen+sort%3Aupdated-desc)

## Motivation

The `BufferList` ([bl](https://www.npmjs.com/package/bl)) module on npm is downloaded 30 million times a week with thousands of dependencies. It's purpose is to be

> a storage object for collections of Node Buffers, exposing them with the main Buffer readable API.

Its key benefit is performant zero-copy concatenation of `Buffer` instances that can be collectively used as if they were a single `Buffer`, which provides a massive performance improvement by avoiding copies.

Unfortunately, however, while `BufferList` in the bl module implements the basic `Buffer` API, it is not usable as a `Uint8Array` the way Node.js `Buffer` is. This makes it a bit difficult to use effectively in scenarios where we need an `ArrayBuffer` or `TypedArray`.

What we want is the ability to create an `ArrayBuffer` that is a concatenation of one or more other `ArrayBuffer` instances without the need to copy, and with the ability to wrap a `TypedArray` around it.

## Proposal

Here we propose a similar language level mechanism that would allow an `ArrayBuffer` to be composed to multiple source `ArrayBuffer` instances, without copying, that can be still be used just like any other type of `ArrayBuffer` (at least at the JavaScript level)

```js
const ab1 = new ArrayBuffer(10);
const ab2 = new ArrayBuffer(10);

const combined = ArrayBuffer.of(ab1, ab2);

console.log(combined instanceof ArrayBuffer);  // true
console.log(combined.byteLength); // 20

const u8 = new Uint8Array(combined);

const ab3 = new ArrayBuffer(20);
const combined2 = ArrayBuffer.of(combined, ab3); // ab1, ab2, and ab3
console.log(combined2.byteLength); // 30
const combinedu8 = new Uint8Array(combined2);  // works!

// importantly, the original source ArrayBuffers are still usable
const view1 = new Uint8Array(ab1);  // still works
view1[0] = 1;
const view2 = new Uint8Array(ab2);
view2[0] = 1;

console.log(u8[0]);  // 1
console.log(u8[10]);  // 1
console.log(u8); // [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0]
```

Importantly, the `ArrayBuffer` instance returned by `ArrayBuffer.of(...)` is really just a collection of the `ArrayBuffer` instances that are passed in.

For simplicity, resizable/growable `ArrayBuffer`s cannot be added to the list.

If any of the source `ArrayBuffer` instances become detached, the combined `ArrayBuffer` is detached.

```js
const ab1 = new ArrayBuffer(10);
const ab2 = new ArrayBuffer(10);
const combined = ArrayBuffer.of(ab1, ab2);
ab1.transfer();
console.log(combined.byteLength);  // 0
console.log(combined.detached);  // true
```

## Use Case: Buffering TransformStream

Imagine a scenario where we want to easily coallesce writes in a stream without copying.

```js
const ts = new TransformStream({
  start() {
    this.buffer = new ArrayBuffer(0);
  },
  transform(chunk, controller) {
    const ab = getArrayBufferFromChunk(chunk);
    // zero-copy concatenation...
    this.buffer = ArrayBuffer.of(this.buffer, ab.transfer());
    if (this.buffer.byteLength >= 4096) {
      controller.enqueue(this.buffer);
      this.buffer = new ArrayBuffer(0);
    }
  },
  terminate(controller) {
    controller.enqueue(this.buffer);
  }
});

async consume(readable) {
  for await (const chunk of readable) {
    // sees >=4096 sized chunks concatenated without copy, smaller only if the stream ended
  }
}

consume(ts.readable);

const writer = ts.writable.getWriter();
while (true) {
  writer.write(new ArrayBuffer(randomSize()));
}
await writer.close();
```

WHATWG streams do not currently have any built in mechanism similar to the Node.js streams `writev(...)` that allows multiple chunks to be buffered and written as one unit. While a `WritableStream` can be implemented specifically to assume that the `chunk` written is an array, it is difficult to retroactively update existing `WritableStream` implementations to support such a construction. The `ArrayBuffer.of(...)` mechanism would, at least for `WritableStream` instances that support bytes, provide an equivalent mechanism without the need to modify existing implementations of the streams API, as illustrated in the example above.

### `ArrayBuffer.of(arrayBuffers...) : ArrayBuffer`

Returns an `ArrayBuffer` that is a zero-copy concatenation of the given array of `ArrayBuffer`s.

The method will throw if:

* Any of the arguments is not an `ArrayBuffer` or `SharedArrayBuffer`
* Any of the argument `ArrayBuffer`s are resizable
* Any of the argument `ArrayBuffer`s are detached (zero-length `ArrayBuffers` are acceptable)

### `arrayBuffer.slice(...) : ArrayBuffer`

Calling `arrayBuffer.slice(...)` on an `ArrayBuffer` returned by `ArrayBuffer.of(...)` will return a new instance that is a *copy* of the combined contents. This is necessary to remain consistent with the current `ArrayBuffer.prototype.slice()` which returns a copy. Conveniently, this provides a mechanism for flattening the collection of member buffers into a single copy when needed.

### `arrayBuffer.transfer(...) : ArrayBuffer`

Calling `arrayBuffer.transfer()` will return a new `ArrayBuffer` that uses the same underlying collection of `ArrayBuffers`, then detaches this buffer.

### `arrayBuffer.resizable : boolean`

Will always be `false` for these `ArrayBuffer` instances. They will never be resizable, and the source `ArrayBuffer` instances are not resizable.

### `arrayBuffer.subarray(...) : ArrayBuffer`

This would be an new alternative to `arrayBuffer.slice(...)` that returns a view over the current `ArrayBuffer` without copying. This is added to better support the zero-copy use case. Such `ArrayBuffer`s become detached when their source `ArrayBuffer`s are detached.

As an example of how this new method would be useful, consider an example where I have two `TypedArrays` that I want to zero-copy concatenate, taking the appropriate `byteOffset` and `byteLength` into consideration:

```js
const u8 = new Uint8Array(100);
const u8a = u8.subarray(0, 10);  // take a view of the first 10 bytes
const u8b = u8.subarray(90, 100); // take a view of the last 10 bytes

// Now create a concatenation of those two ranges...
const combined = ArrayBuffer.of(u8a.buffer.subarray(u8a.byteOffset, u8a.byteLength),
                                u8b.buffer.subarray(u8b.byteOffset, u8b.byteLength));
```

### `SharedArrayBuffer.of(sharedArrayBuffers...) : SharedArrayBuffer`

The equivalent of `ArrayBuffer.of(...)` but specifically for `SharedArrayBuffer`. All of the arguments given must be non-growable `SharedArrayBuffer` instances. Effectively the same zero-copy characteristics.

### `SharedArrayBuffer.subarray(...) : SharedArrayBuffer`

The equivalent of `arrayBuffer.subarray(...)` but specifically for `SharedArrayBuffer`.

### `sharedArrayBuffer.growable : boolean`

Will always be `false` for these `SharedArrayBuffer` instances. They will never be growable, and the source `SharedArrayBuffer` instances are not growable.

## Questions

### Do `arrayBuffer.subarray(...)` / `sharedArrayBuffer.subarray(...)` belong here?

Many users may not know that `arrayBuffer.slice()` actually copies the data. To better support the zero-copy use case, and to avoid unintended footguns, `arrayBuffer.subarray(...)` is introduced here. That said, an argument could be made that it is out of place in this proposal. Should it remain? Should it be removed?

An example of intended use:

```js
const ab1 = new ArrayBuffer(100);
const ab2 = new ArrayBuffer(10);
const u8 = new Uint8Array(ab1, 0, 10);

const combined = ArrayBuffer.of(u8.buffer.subarray(u8.byteOffset, u8.byteLength), ab2);
```

Alternatively, `transfer(length)` could be used,

```
const combined = ArrayBuffer.of(u8.buffer.transfer(10), ab2);
```

But `arrayBuffer.subarray(...)` provides the ability to create an offset view without additional tricks or copies.

### What about the host view of such objects?

Host runtimes will likely need to handle these kinds of `ArrayBuffer` instances specially. For instance, in v8, a `v8::ArrayBuffer` is expected to be backed by a single `v8::BackingStore` whose memory is always contiguous. It is likely that a new underlying type would need to be created (hypothetically like a `v8::ArrayBufferList`) that provides access to the collection of individual `v8::BackingStore` instances that have been collected. Alternatively (or additionally) runtimes could support coercion of this kind of `ArrayBufferList` into an `ArrayBuffer` that forces a copy into a single, contiguous `BackingStore`.

```c++
v8::Local<v8::Value> value;

if (value->IsArrayBufferList()) {
  v8::Local<v8::ArrayBufferList> abl = value.As<v8::ArrayBufferList>();
  for (auto n = 0; n < abl.Length(); n++) {
    auto backingStore = abl.GetBackingStore(n);
    // ...
  }

  // And/Or ...
  v8::Local<v8::ArrayBuffer> ab = abl->ToArrayBuffer();  // copies and flattens
}
```

### Something to consider separately: `arrayBuffer.transferAsShared(...)`

One thing about `SharedArrayBuffer` that came up as part of the consideration for this, but somewhat unrelated to the zero-copy use case, is how to go from an `ArrayBuffer` to a `SharedArrayBuffer`. As a separate proposal we might consider if an `arrayBuffer.transferAsShared(...)` would be generally useful.

```js
const ab = new ArrayBuffer(10);
const sab = ab.transferAsShared();
ab.detached; // true
```

Noting this here only so it's not forgotten.


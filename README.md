# JetKV

_JetKV_ is a key-value store written in _Zig_ intended for use in development web servers.

_JetKV_ can be used for:

* Background job queuing
* Persistent data storage
* Cache

_JetKV_ is used by the [Jetzig Web Framework](https://jetzig.dev/) to provide a zero-setup, in-process key-value store for all of the above.

## Checklist

* :white_check_mark: In-memory storage.
* :white_check_mark: String value storage.
* :white_check_mark: Array value storage.
* :white_check_mark: Array pop/queue implementation.
* :white_check_mark: On-disk storage.
* :x: Key expiry.
* :x: Shared memory.

## Usage

### Memory Allocator

```zig
var kv = try JetKV.init(allocator, .{ .backend = .memory });
```

### File Allocator

The file allocator receives an allocator but does not perform any allocations. It is therefore possible to pass `undefined` instead of an allocator when using the file allocator.

The file passed as the `path` field is locked on startup.

```zig
var kv = try JetKV.init(
    allocator,
    .{
        .backend = .file,
        .file_backend_options = .{
            // Path to storage file (JetKV stores all data in a single, platform-agnostic file)
            .path = "/path/to/jetkv.db",
            // Set to `true` to clear the store on each launch.
            .truncate = false,
            // Set the size of the on-disk hash table (each address is currently 4 bytes)
            // Use `jetkv.addressSpaceSize` to guarantee a valid size if address size changes in future
            .address_space_size = jetkv.addressSpaceSize(4096),
        },
    },
);
```

### Key-Value Operations

All operations are identical for `.file` and `.memory` backends.

All operations are _O(1)_ complexity for both backends.

```zig
// Put some strings into the KV store
try kv.put("foo", "baz");
try kv.put("bar", "qux");

// `append` and `prepend` create a new array if one does not already exist
try kv.append("example_array", "quux");
try kv.prepend("example_array", "corge");

if (try kv.get(allocator, "foo")) |value| {
    // "baz"
    allocator.free(value);
}

if (try kv.fetchRemove(allocator, "bar")) |value| {
    // "qux"
    allocator.free(value);
}

try kv.remove("foo");

if (kv.pop(allocator, "example_array")) |value| {
    // "quux"
    allocator.free(value);
}

if (kv.popFirst(allocator, "example_array")) |value| {
    // "corge"
    allocator.free(value);
}
```

## Implementation

### Memory

The memory backend uses a _Zig_ `std.StringHashMap` of `[]const u8` for string storage and `std.DoublyLinkedList([]const u8)` for array storage.

### File

The file backend implements a fixed-sized hash table at the beginning of the file.

Hash collisions are resolved as singly-linked lists. Arrays are implemented as doubly-linked lists.

Each index in the hash table references a location in the file which provides address information:

* Value type
* Next linked item (for collision resolution)
* Next array item
* Previous array item
* End array item
* Key length
* Initial key length
* Value length
* Initial value length
* Key
* Value

Values are inserted with a relative amount of over-allocation to allow re-use of space when replacing values.

Keys have a maximum length of `1024` bytes in order to allow key comparison to operate exclusively on the stack.

Reference counting is used to allow truncating the file when the store becomes empty.

## License

[MIT](LICENSE)

# Why `localhost:3490` Shows “File not found!”

The problem is in `request.zig`’s parsing order.

A browser sends this request line:

```http
GET / HTTP/1.1
```

But `parse_request()` assigns the tokens like this:

```zig
const method = try Method.init(iterator.next().?); // GET
const version = iterator.next().?;                 // /
const uri = iterator.next().?;                     // HTTP/1.1
```

So `request.uri` becomes `"HTTP/1.1\r"` rather than `"/"`.

Then `main.zig` checks:

```zig
if (std.mem.eql(u8, request.uri, "/")) {
    try Response.send_200(connection, io);
} else {
    try Response.send_404(connection, io);
}
```

Because `"HTTP/1.1\r"` does not equal `"/"`, it calls `send_404()`. That function in `response.zig` explicitly returns:

```html
<h1>File not found!</h1>
```

So this is not an actual missing-file error—the server does not try to open a file at all. It displays that message because the URI and HTTP version are parsed in the wrong order. The request format is **method, URI, version**, whereas the code reads it as **method, version, URI**.

Separately, the response status line says `404 OK`, which is contradictory—it would conventionally be `404 Not Found`—but that is not what causes this behavior.

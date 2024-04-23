# Networking

[1) Understanding Networking Layers](https://github.com/fabio-miguel/tcp-nodejs-networking)

[2) Implementing TCP](https://github.com/fabio-miguel/chat-nodejs-app-networking)

[3) Implementing DNS](https://github.com/fabio-miguel/dns-nodejs-networking)

[4) Implementing TCP (_Again_) - **this repo**](https://github.com/fabio-miguel/tcp-nodejs-networking)

## About Uploader Application - Node.js

An uploader app, using only Node.js modules (no external libraries). The app takes in a file path on the client side, `node client.js /desktop/documents/my-file.txt` and stores it in the storage folder deployed on a server. Integrated a loading functionaity to disoplay percent upload completed.

## Issues

This is a TCP application that doesn’t have any encryption. It’s the same if you have just a HTTP application. You wouldn’t / shouldn’t use these own their own. You would run an encryption on top of top of HTTP. This is usually TLS. When you run TLS on top of HTTP, it will become HTTPS. Many secure protocols use TLS.

Right now, any hacker can sniff the network and understand what is going on in this network like what files are being uploaded and its contents. This isn’t something that should be done in production. Thus this isn't production ready nor was it intended to be!

## Notes on server.js file

A TCP server instance was created using `net.createServer`. This allows us to listen to TCP connections on a specified port and hostName. In the code of this repository, the host is set to the local host / loopback address. The `server.on()` method was used to listen for this connection event and trigger the callback when a connection has been established with a client. The callback provides access to the socket object, which is a duplex stream, meaning it is able to read and write data to the client. It represents the server endpoint. Then, we log that a connection has been made. We define variables for the `fileHandle` and `filewriteStream` outside the scope of the data reading as every new file upload will require these. Then, we use the `socket.on("data", () => {})` method to begin reading data from the data being receiced from the client. This "data" event is emitted implicitly.

---

**A sidenote about implicit triggering of data event on read stream:**
_When you listen for the 'data' event on a readable stream in Node.js, such as a file stream created using `fs.createReadStream()` in the client.js file, the 'data' event is emitted as soon as the stream starts reading from its source._

_So, by attaching a listener to the 'data' event using the `fileStream.on('data', …)` method, you're essentially indicating to Node.js that you want to start reading data from the file and handle it as soon as it becomes available. As soon as data is available, the 'data' event is emitted, triggering the callback function provided to the `on()` method. Therefore, the 'data' event is implicitly triggered as soon as you listen for it using the readable stream is correct. It marks the beginning of data transmission from the source (in this case, the file) through the stream._

---

If the `fileHandle` is false, meaning it is not open, we pause receiving data from the client. We then extract the fileName, which is a custom header I made on the client side, from the first packet received from the TCP connection. _More on this below._ We then open / create the file using the `fs.open()` method using the `"w"` flag to indicate that we will write to this file reference. Next, using the file reference we create a **writable** stream to that file. Now, it becomes a matter of writing to that file using the writable stream created. However, rather than writing all the data, we ensure that we only write the data after the headers as to not iclude them and only the file data. As we write to the internal buffer of the write stream, it will return false when full. This means to avoid backpressure where any further reading is buffered in memory, we only resume reading data when the internal write stream buffer emits a `"drain"` event to indicate that it has emptied its internal buffer. So, once the `fileWriteStream` (i.e. the writable stream) has been drained, we can resume reading from the client again by using `socket.resume()`. Lastly, when the client ends their connection and the server receives it FIN packet, the variables will be reset to handle other uploads and close the file to avoid server-side memory issues.

## Getting the File Names Dynamically

We’re using TCP, and in TCP packets are numbered. This means that we’re going to receive the packets in order. So, in the first write, we intend to send the fileName. So, in the `server.js`, the first chunk / first packet that we are going to get will include the fileName.

```
  socket.write(`fileName: ${fileName}-------`);
```

_**Side note:** using UDP, there is no guarantee that this would be the first packet._

In `server.js`, we then grab the fileName variable by using the `subarray()` method, which starts at the variable and ends at the start of the divider.

```
const fileName = data.subarray(10, indexOfDivider).toString("utf-8");
```

There are 10 characters from the start of the template literal up until the variable (including the space). And, in our buffer, because we are using uff-8, we are representing each character using 8 bits / 1 byte. So, we are ignoring the first 80 bits / 8 bytes to get to the variable. The `subarray()` method returns a buffer, so we use the `toString()` method with encoding to get the filename as a string.

The index is a static index of 10, which is more rigid in its implementation. HTTP actually builds on top of TCP with a more complex approach with regard to its calculations, their substrings / grabbing a portion out of a buffer etc.

## Creating and Displaying an Upload Progress Bar

Every time we read a chunk of data from…

```
fileReadStream.on("data", async (data) => {
    if (!socket.write(data)) {
      fileReadStream.pause();
    }
})
```

…we would like to figure out how much of the file we have read. Then, we can do some math and convert that number to a percentage and just log that on the screen.

First, we need the file size of the file.

```
const fileSize = (await fileHandle.stat()).size;

```

Each file has its own tags associated with it and the `stat()` method simply returns stats like, its last modified, last opened etc. We use await as the stat method returns a promise and it’s also why the whole method is wrapped in parentheses. Then, we grab the size using the `size()` method. It returns how many bits the file is.

## Other Notes

As soon as we uploaded the file, we want the connection to be ended.
This is how HTTP 1.0 works. In HTTP v1 there’s a header called connection. If you specify the header to be closed, that means for each HTTP request, you will open a TCP connection. You will then send that request and receive a response. Once you receive the response you’re going to kill that TCP connection. However, that’s not the case anymore with HTTP 2.0.

Now, you would usually set the header connection to be “keep alive,” which means that we’re going to open up a TCP connection and then we’re going to send a request, receive a response and then use the exact same TCP connection for any following / more requests. Perhaps after a few seconds of inactivity the connection may be killed. For example, there might be a timeout that says, if after five seconds nothing travels in this TCP connection, then go ahead and kill the TCP connection. So, HTTP v2 keeps the connection Alice to reuse the same connection so that performance improves, instead of having to establish a new TCP connection for every single request, which increases latency and has more overhead for managing multiple TCP connections. This is called multiplexing multiple requests and responses over a single TCP connection. Multiplexing simply means being able to carry out multiple streams over a single connection. Note: each stream will have its own stream ID.

But, to reiterate, this application operates like HTTP 1.0. We open up a TCP connection, send the request, send that file and then close the TCP connection once we’re done, rather than leaving it open. Alternatively, we could keep the connection open in case we want to upload another file.

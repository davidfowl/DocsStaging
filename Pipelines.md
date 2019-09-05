# System.IO.Pipelines

- [What problem does it solve?](#)
- [Creating a PipeReader/PipeWriter](#)
- [Reading from a PipeReader](#)
- [Scenarios](#)
- [Gotchas](#)
- [Using PipeReader with Stream APIs](#)

System.IO.Pipelines is a new library that is designed to make it easier to do high performance IO in .NET. It’s a library targeting .NET Standard that works on all .NET implementations.

## What problem does it solve?
Correctly parsing streaming data is dominated by boilerplate code and has many corner cases, leading to complex code that is difficult to maintain.
Achieving high performance and being correct, while also dealing with this complexity is difficult. Pipelines aims to solve this complexity.

Let's start with a simple problem. We want to write a TCP server that receives line-delimited messages (delimited by n) from a client.

The typical code you would write in .NET before pipelines looks something like this:

```C#
async Task ProcessLinesAsync(NetworkStream stream)
{
    var buffer = new byte[1024];
    await stream.ReadAsync(buffer, 0, buffer.Length);
    
    // Process a single line from the buffer
    ProcessLine(buffer);
}
```

This code might work when testing locally but it's has several errors:

- The entire message (end of line) may not have been received in a single call to ReadAsync.
- It's ignoring the result of stream.ReadAsync() which returns how much data was actually filled into the buffer.
- It doesn't handle the case where multiple lines come back in a single ReadAsync call.

These are some of the common pitfalls when reading streaming data. To account for this we need to make a few changes:

- We need to buffer the incoming data until we have found a new line.
- We need to parse all of the lines returned in the buffer.
- It's possible that the line is bigger than 1KiB (1024 bytes) so we need to resize the input buffer until we have found a new line.
    - If we re-size the buffer it results in more buffer copies as longer lines appear in the input. 
    - To reduce wasted space, we also need to compact the buffer used for reading lines.

```C#
async Task ProcessLinesAsync(NetworkStream stream)
{
    byte[] buffer = ArrayPool<byte>.Shared.Rent(1024);
    var bytesBuffered = 0;
    var bytesConsumed = 0;

    while (true)
    {
        // Calculate the amount of bytes remaining in the buffer
        var bytesRemaining = buffer.Length - bytesBuffered;

        if (bytesRemaining == 0)
        {
            // Double the buffer size and copy the previously buffered data into the new buffer
            var newBuffer = ArrayPool<byte>.Shared.Rent(buffer.Length * 2);
            Buffer.BlockCopy(buffer, 0, newBuffer, 0, buffer.Length);
            // Return the old buffer to the pool
            ArrayPool<byte>.Shared.Return(buffer);
            buffer = newBuffer;
            bytesRemaining = buffer.Length - bytesBuffered;
        }

        var bytesRead = await stream.ReadAsync(buffer, bytesBuffered, bytesRemaining);
        if (bytesRead == 0)
        {
            // EOF
            break;
        }
        
        // Keep track of the amount of buffered bytes
        bytesBuffered += bytesRead;
        
        do
        {
            // Look for a EOL in the buffered data
            linePosition = Array.IndexOf(buffer, (byte)'\n', bytesConsumed, bytesBuffered - bytesConsumed);

            if (linePosition >= 0)
            {
                // Calculate the length of the line based on the offset
                var lineLength = linePosition - bytesConsumed;

                // Process the line
                ProcessLine(buffer, bytesConsumed, lineLength);

                // Move the bytesConsumed to skip past the line we consumed (including \n)
                bytesConsumed += lineLength + 1;
            }
        }
        while (linePosition >= 0);
    }
}
```

## Creating a PipeReader/PipeWriter

There are several ways to create a `PipeReader/PipeWriter`:
- Use the [`Pipe`](https://docs.microsoft.com/en-us/dotnet/api/system.io.pipelines.pipe?view=dotnet-plat-ext-2.1) class to create a `PipeWriter/PipeReader` pair. All data written into the `PipeWriter` will be available in the `PipeReader`.
- Use `PipeReader.Create`/`PipeWriter.Create` to create a `PipeReader` or `PipeWriter` from an existing `Stream`.
- Implement your own. Both `PipeReader` and `PipeWriter` are abstract classes.

```C#
var pipe = new Pipe();
PipeReader reader = pipe.Reader;
PipeWriter reader = pipe.Writer;
```

## Reading from a [PipeReader](https://docs.microsoft.com/en-us/dotnet/api/system.io.pipelines.pipereader?view=dotnet-plat-ext-2.1)

The `PipeReader` manages memory on the callers behalf, because of this it's important to **always** call `PipeReader.AdvanceTo` after calling `PipeReader.ReadAsync`. This lets the `PipeReader` know when the caller is done with the memory so that it can be tracked appropriately. The `ReadOnlySequence<byte>` returned from `PipeReader.ReadAsync` is only valid until the call the `PipeReader.AdvanceTo`. This means that it's illegal to use it after calling `PipeReader.AdvanceTo` and doing so may result in invalid/undocumented/broken behavior. 

`PipeReader.AdvanceTo` takes 2 `SequencePosition(s)`, the first one determines how much memory was consumed and the second determines how much of the buffer was observed. Marking data as consumed means the pipe can clean the memory up (return it to the underlying buffer pool etc), while marking data as observed controls what the next call to `PipeReader.ReadAsync` will do. Marking everything observed means the next call to `PipeReader.ReadAsync` will not return until there's more data. Anything other value will make the next call to `PipeReader.ReadAsync` return immediately with the unobserved data.

### Scenarios

There are a couple of typical patterns that emerge when trying to read streaming data:
- Given a stream of data, parse a single message
- Given a stream of data, parse all available messages

The following examples will use the following method for parsing messages from a `ReadOnlySequence<byte>`. This method will parse a single message and update the input buffer to trim the parsed message from the buffer.

```C#
bool TryParseMessage(ref ReadOnlySequence<byte> buffer, out Message message);
```

#### Reading a single message

The code below reads a single message from a `PipeReader` and returns it to the caller.

```C#
async ValueTask<Message> ReadSingleMessageAsync(PipeReader reader, CancellationToken cancellationToken = default)
{
    while (true)
    {
        ReadResult result = await reader.ReadAsync(cancellationToken);
        ReadOnlySequence<byte> buffer = result.Buffer;
        
        // In the event that we don't parse any message successfully, mark consumed as nothing
        // and examined as the entire buffer.
        SequencePosition consumed = buffer.Start;
        SequencePosition examined = buffer.End;
        
        try
        {
            if (TryParseMessage(ref buffer, out Message message))
            {
                // We successfully parsed a single message so mark the start as the parsed buffer as consumed
                // TryParseMessage trims the buffer to point to the data after the message was parsed
                consumed = buffer.Start;
                
                // Examined is marked the same as consumed here so that the next call to ReadSingleMessageAsync
                // will process the next message if there is one
                examined = consumed;

                return message;
            }
            
            // There's no more data to be processed
            if (result.IsCompleted)
            {
                if (buffer.Length > 0)
                {
                    // We have an incomplete message and there's no more data to process
                    throw new InvalidDataException("Incomplete message!");
                }
                
                break;
            }
        }
        finally
        {
            reader.AdvanceTo(consumed, examined);
        }
    }
    
    return null;
}
```

The code above parses a single message and updates the consumed and examined `SequencePosition`s to point to start of the trimmed input buffer. This is because `TryParseMessage` removes the parsed message from the input buffer. Generally, when parsing a single message from the buffer, the examined position should be the end of the message or the end of the received buffer if no message was found.

The single message case has the most potential for errors. Passing the wrong values to examined can result in an OOM or infinite loop (see the [gotchas](#) section below).

#### Reading multiple messages

The code below reads all messages from a `PipeReader` and calls `ProcessMessageAsync` on each.

```C#
async Task ProcessMessagesAsync(PipeReader reader, CancellationToken cancellationToken = default)
{
    try
    {
        while (true)
        {
            ReadResult result = await reader.ReadAsync(cancellationToken);
            ReadOnlySequence<byte> buffer = result.Buffer;

            // In the event that we don't parse any message successfully, mark consumed as nothing
            // and examined as the entire buffer.
            SequencePosition consumed = buffer.Start;
            SequencePosition examined = buffer.End;

            try
            {
                while (TryParseMessage(ref buffer, out Message message))
                {
                    // We successfully parsed a single message so mark the start as the parsed buffer as consumed
                    // TryParseMessage trims the buffer to point to the data after the message was parsed
                    consumed = buffer.Start;

                    // Examined is marked the same as consumed here so that the next call to ReadSingleMessageAsync
                    // will process the next message if there is one
                    examined = consumed;

                    await ProcessMessageAsync(message);
                }

                // There's no more data to be processed
                if (result.IsCompleted)
                {
                    if (buffer.Length > 0)
                    {
                        // We have an incomplete message and there's no more data to process
                        throw new InvalidDataException("Incomplete message!");
                    }
                    break;
                }
            }
            finally
            {
                reader.AdvanceTo(consumed, examined);
            }
        }
    }
    finally
    {
        await reader.CompleteAsync();
    }
}
```

### Gotchas

- Passing the wrong values to consumed/examined may result reading already read data (for e.g. passing a `SequencePosition` that was already processed)
- Passing `buffer.End` as examined may result in missing/stalled data or hangs. (for e.g. `PipeReader.AdvanceTo(position, buffer.End)` when processing a single message at a time from the buffer.)
- Passing the wrong values to consumed/examined may result in an infinite loop. (for e.g. `PipeReader.AdvanceTo(buffer.Start)` if `buffer.Start` hasn't changed will cause the next call to `PipeReader.ReadAsync` to return immediately before new data arrives)
- Passing the wrong values to consumed/examined may result in inifinite buffering (eventual OOM). (for e.g.  `PipeReader.AdvanceTo(buffer.Start, buffer.End)` unconditionally when processing a single message at a time from the buffer)
- Failing to call `PipeReader.Complete/CompleteAsync` may result in a memory leak.
- Checking `ReadResult.IsCompleted` and exiting the reading logic before processing the buffer will result in data loss. The loop exit condition should be based on `ReadResult.Buffer.IsEmpty` and `ReadResult.IsCompleted`. Doing this in the wrong order could result in an infinite loop.

#### Code samples

These code samples will result in data loss, hangs, potential security issues (depending on where they are used) and should **NOT** be copied. They exists solely for illustration of the gotchas mentioned above.

❌ **Data loss**

Breaking early from the loop will result in data loss. `ReadResult.IsCompleted` can be set to true even if the reader hasn't already processed all of the data. This is possible because unprocessed data is buffered by the `PipeReader`.

```C#
// These code samples will result in data loss, hangs, security issues and should **NOT** be copied. They exists solely for illustration of the gotchas mentioned above.
Environment.FailFast("This code is terrible, don't use it!");
while (true)
{
    ReadResult result = await reader.ReadAsync(cancellationToken);
    ReadOnlySequence<byte> buffer = result.Buffer;
    
    if (result.IsCompleted)
    {
        break;
    }
    
    Process(ref buffer, out Message message);
    
    reader.AdvanceTo(buffer.Start, buffer.End);
}
// These code samples will result in data loss, hangs, security issues and should **NOT** be copied. They exists solely for illustration of the gotchas mentioned above.
```

❌ **Infitnite loop**

The below logic may result in an infinite loop if the `Result.IsCompleted` but there's never a complete message in the buffer. 

```C#
// These code samples will result in data loss, hangs, security issues and should **NOT** be copied. They exists solely for illustration of the gotchas mentioned above.
Environment.FailFast("This code is terrible, don't use it!");
while (true)
{
    ReadResult result = await reader.ReadAsync(cancellationToken);
    ReadOnlySequence<byte> buffer = result.Buffer;
    if (result.IsCompleted && buffer.IsEmpty)
    {
        break;
    }
    
    Process(ref buffer, out Message message);
    
    reader.AdvanceTo(buffer.Start, buffer.End);
}
// These code samples will result in data loss, hangs, security issues and should **NOT** be copied. They exists solely for illustration of the gotchas mentioned above.
```

❌ **Unexpected Hang**

Unconditionally calling `PipeReader.AdvanceTo` with `buffer.End` in the examined position may result in hangs when parsing a single message. The next call to `PipeReader.AdvanceTo` will not return until there's more data than previous examined.

```C#
// These code samples will result in data loss, hangs, security issues and should **NOT** be copied. They exists solely for illustration of the gotchas mentioned above.
Environment.FailFast("This code is terrible, don't use it!");
while (true)
{    
    ReadResult result = await reader.ReadAsync(cancellationToken);
    ReadOnlySequence<byte> buffer = result.Buffer;
    
    Process(ref buffer, out Message message);
    
    if (result.IsCompleted)
    {
        break;
    }
    
    reader.AdvanceTo(buffer.Start, buffer.End);
    
    if (message != null)
    {
        return message;
    }
}
// These code samples will result in data loss, hangs, security issues and should **NOT** be copied. They exists solely for illustration of the gotchas mentioned above.
```

❌ **Out of Memory**

If there's no maximum message size and the data returned from the `PipeReader` does not make a complete message (because the other side is writing a large message e.g 4GB) the logic below will keep buffering until an `OutOfMemoryException` occurs.

```C#
// These code samples will result in data loss, hangs, security issues and should **NOT** be copied. They exists solely for illustration of the gotchas mentioned above.
Environment.FailFast("This code is terrible, don't use it!");
while (true)
{
    ReadResult result = await reader.ReadAsync(cancellationToken);
    ReadOnlySequence<byte> buffer = result.Buffer;
    
    Process(ref buffer, out Message message);
    
    if (result.IsCompleted)
    {
        break;
    }
    
    reader.AdvanceTo(buffer.Start, buffer.End);
    
    if (message != null)
    {
        return message;
    }
}
```

### Using PipeReader with Stream APIs

When reading streaming data it is very common to read data using a de-serializer or write data using a serializer. Most of these APIs take `Stream` today. In order to make it easier to integrate with these existing APIs `PipeReader` and `PipeWriter` expose an `AsStream` which will return a `Stream` implementation around the `PipeReader` or `PipeWriter`. 


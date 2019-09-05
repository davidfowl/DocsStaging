# System.IO.Pipelines

*Pre-requisite: Read https://devblogs.microsoft.com/dotnet/system-io-pipelines-high-performance-io-in-net/ for details on what Pipes are and why you would want to use them*

- [Creating a PipeReader/PipeWriter](#)
- [Reading from a PipeReader](#)
- [Scenarios](#)
- [Gotchas](#)
- [Using PipeReader with Stream APIs](#)

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
- Passing buffer.End as examined may result in missing/stalled data or hangs. (for e.g. `PipeReader.AdvanceTo(position, buffer.End)` when processing a single message at a time from the buffer.)
- Passing the wrong values to consumed/examined may result in an infinite loop. (for e.g. `PipeReader.AdvanceTo(buffer.Start)` if `buffer.Start` hasn't changed will cause the next call to `PipeReader.ReadAsync` to return immediately before new data arrives)
- Passing the wrong values to consumed/examined may result in inifinite buffering (eventual OOM). (for e.g.  `PipeReader.AdvanceTo(buffer.Start, buffer.End)` unconditionally when processing a single message at a time from the buffer)
- Failing to call `PipeReader.Complete/CompleteAsync` may result in a memory leak.
- Incorrect handling of `ReadResult.Buffer.IsEmpty` and `ReadResult.IsCompleted` may cause incorrect parsing of data.

### Using PipeReader with Stream APIs

When reading streaming data it is very common to read data using a de-serializer or write data using a serializer. Most of these APIs take `Stream` today. In order to make it easier to integrate with these existing APIs `PipeReader` and `PipeWriter` expose an `AsStream` which will return a `Stream` implementation around the `PipeReader` or `PipeWriter`.

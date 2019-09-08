# System.IO.Pipelines

- [What problem does it solve?](#what-problem-does-it-solve)
- [Pipe](#pipe)
    - [Basic usage](#basic-usage)
    - [Backpressure and flow control](#backpressure-and-flow-control)
    - [PipeScheduler](#pipescheduler)
- [PipeReader](#pipereader)
    - [Scenarios](#scenarios)
    - [Cancellation](#cancellation)
    - [Gotchas](#gotchas)
- [PipeWriter](#pipewriter)
    - [Scenarios](#)
    - [Gotchas](#)
- [Streams](#streams)

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
- We may want to use buffer pooling to avoid allocating memory repeatedly.

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

The complexity has gone through the roof (and we haven't even covered all of the cases). High performance networking usually means writing very complex code in order to eke out more performance from the system. The goal of **System.IO.Pipelines** is to make writing this type of code easier.

## Pipe

The [`Pipe`](https://docs.microsoft.com/en-us/dotnet/api/system.io.pipelines.pipe?view=dotnet-plat-ext-2.1) class can be uesd to create a `PipeWriter/PipeReader` pair. All data written into the `PipeWriter` will be available in the `PipeReader`.

```C#
var pipe = new Pipe();
PipeReader reader = pipe.Reader;
PipeWriter reader = pipe.Writer;
```

### Basic usage

```C#
async Task ProcessLinesAsync(Socket socket)
{
    var pipe = new Pipe();
    Task writing = FillPipeAsync(socket, pipe.Writer);
    Task reading = ReadPipeAsync(pipe.Reader);

    return Task.WhenAll(reading, writing);
}

async Task FillPipeAsync(Socket socket, PipeWriter writer)
{
    const int minimumBufferSize = 512;

    while (true)
    {
        // Allocate at least 512 bytes from the PipeWriter
        Memory<byte> memory = writer.GetMemory(minimumBufferSize);
        try 
        {
            int bytesRead = await socket.ReceiveAsync(memory, SocketFlags.None);
            if (bytesRead == 0)
            {
                break;
            }
            // Tell the PipeWriter how much was read from the Socket
            writer.Advance(bytesRead);
        }
        catch (Exception ex)
        {
            LogError(ex);
            break;
        }

        // Make the data available to the PipeReader
        FlushResult result = await writer.FlushAsync();

        if (result.IsCompleted)
        {
            break;
        }
    }

    // Tell the PipeReader that there's no more data coming
    writer.Complete();
}

async Task ReadPipeAsync(PipeReader reader)
{
    while (true)
    {
        ReadResult result = await reader.ReadAsync();
        ReadOnlySequence<byte> buffer = result.Buffer;

        while (TryReadLine(ref buffer, out ReadOnlySequence<byte> line))
        {
            // Process the line
            ProcessLine(line);
        }

        // Tell the PipeReader how much of the buffer we have consumed
        reader.AdvanceTo(buffer.Start, buffer.End);

        // Stop reading if there's no more data coming
        if (result.IsCompleted)
        {
            break;
        }
    }

    // Mark the PipeReader as complete
    reader.Complete();
}

bool TryReadLine(ref ReadOnlySequence<byte> buffer, out ReadOnlySequence<byte> buffer)
{
    // Look for a EOL in the buffer
    SequencePosition? position = buffer.PositionOf((byte)'\n');

    if (position == null)
    {
        buffer = default;
        return false;
    }
    
    // Skip the line + the \n
    buffer = buffer.Slice(buffer.GetPosition(1, position.Value));
    return true;
}
```

There are 2 loops:

- `FillPipeAsync` reads from the `Socket` and writes into the `PipeWriter`.
- `ReadPipeAsync` reads from the `PipeReader` and parses incoming lines.

There are no explicit buffers allocated anywhere. All buffer management is delegated to the `PipeReader`/`PipeWriter` implementations. This makes it easier for consuming code to focus solely on the business logic instead of complex buffer management.

In the first loop, `PipeWriter.GetMemory(int)` is called to get some memory from the underlying writer; then we call `PipeWriter.Advance(int)` to tell the `PipeWriter` how much data was written to the buffer. `PipeWriter.FlushAsync()` is called to make the data available to the `PipeReader`.

In the second loop, the `PipeReader` is consuming the buffers written to the`PipeWriter` which ultimately comes from the Socket. The call to `PipeReader.ReadAsync()` returns a `ReadResult` which contains 2 important pieces of information, the data that was read in the form of `ReadOnlySequence<byte>` and a boolean `IsCompleted` that indicates if we've reached the end of data (EOF). After finding the end of line (EOL) delimiter and parsing the line, the logic slice the buffer to skip what we've already processed and then we call `PipeReader.AdvanceTo` to tell the `PipeReader` how much data we have consumed and examined.

At the end of each of the loops, we complete both the reader and the writer. This lets the underlying Pipe release all of the memory it allocated.

### Backpressure and flow control

In a perfect world, reading & parsing work as a team: the writing thread consumes the data from the network and puts it in buffers while the parsing thread is responsible for constructing the appropriate data structures. Normally, parsing will take more time than just copying blocks of data from the network. As a result, the reading thread can easily overwhelm the parsing thread. The result is that the reading thread will have to either slow down or allocate more memory to store the data for the parsing thread. For optimal performance, there is a balance between frequent pauses and allocating more memory.

To solve this problem, the `Pipe` has two settings to control the flow of data, the `PauseWriterThreshold` and the `ResumeWriterThreshold`. The `PauseWriterThreshold` determines how much data should be buffered before calls to PipeWriter.FlushAsync pauses. The `ResumeWriterThreshold` controls how much the reader has to consume before writing can resume.

![image](https://user-images.githubusercontent.com/95136/64408565-ee3af580-d03b-11e9-9e8a-5b9bc56d592b.png)

`PipeWriter.FlushAsync` returns an incomplete `ValueTask<FlushResult>` when the amount of data in the Pipe crosses PauseWriterThreshold and completes said task when it becomes lower than ResumeWriterThreshold. Two values are used to prevent thrashing around the limit.

### PipeScheduler

Usually when using async/await, asynchronous code resumes on either on a `TaskScheduler` or on the current `SynchronizationContext`.

When doing IO it's very important to have fine-grained control over where that IO is performed so that one can take advantage of CPU caches more effectively, which is critical for high-performance applications like web servers. The `PipeScheduler` gives users control over where asynchronous callbacks run. By default, the current `SynchronizationContext` will be respected and if there is none, it will use the thread pool to run callbacks.

#### Examples

```C#

public static void Main(string[] args)
{
    var writeScheduler = new SingleThreadPipeScheduler();
    var readScheduler = new SingleThreadPipeScheduler();

    // Tell the Pipe what schedulers to use, we also disable the SynchronizationContext 
    var options = new PipeOptions(readerScheduler: readScheduler, writerScheduler: writeScheduler, useSynchronizationContext: false);
    var pipe = new Pipe(options);
}

// This is a sample scheduler that async callbacks on a single dedicated thread
public class SingleThreadPipeScheduler : PipeScheduler
{
    private readonly BlockingCollection<(Action<object> Action, object State)> _queue = new BlockingCollection<(Action<object> Action, object State)>();
    private readonly Thread _thread;

    public SingleThreadPipeScheduler()
    {
        _thread = new Thread(DoWork);
        _thread.Start();
    }

    private void DoWork()
    {
        foreach (var item in _queue.GetConsumingEnumerable())
        {
            item.Action(item.State);
        }
    }

    public override void Schedule(Action<object> action, object state)
    {
        _queue.Add((action, state));
    }
}
```

## [PipeReader](https://docs.microsoft.com/en-us/dotnet/api/system.io.pipelines.pipereader?view=dotnet-plat-ext-2.1)

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

            try
            {
                // Process all messages from the buffer
                while (TryParseMessage(ref buffer, out Message message))
                {
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
                // Since we're processing all messages in the buffer, we can use the remaining buffer's Start and End
                // position to determine consumed and examined
                reader.AdvanceTo(buffer.Start, buffer.End);
            }
        }
    }
    finally
    {
        await reader.CompleteAsync();
    }
}
```

### Cancellation

### Gotchas

- Passing the wrong values to consumed/examined may result reading already read data (for e.g. passing a `SequencePosition` that was already processed)
- Passing `buffer.End` as examined may result in missing/stalled data or hangs. (for e.g. `PipeReader.AdvanceTo(position, buffer.End)` when processing a single message at a time from the buffer.)
- Passing the wrong values to consumed/examined may result in an infinite loop. (for e.g. `PipeReader.AdvanceTo(buffer.Start)` if `buffer.Start` hasn't changed will cause the next call to `PipeReader.ReadAsync` to return immediately before new data arrives)
- Passing the wrong values to consumed/examined may result in inifinite buffering (eventual OOM). (for e.g.  `PipeReader.AdvanceTo(buffer.Start, buffer.End)` unconditionally when processing a single message at a time from the buffer)
- Using the `ReadOnlySequence<byte>` after calling `PipeReader.AdvanceTo` may result in memory corruption (use after free).
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
    ReadOnlySequence<byte> dataLossBuffer = result.Buffer;
    
    if (result.IsCompleted)
    {
        break;
    }
    
    Process(ref dataLossBuffer, out Message message);
    
    reader.AdvanceTo(dataLossBuffer.Start, dataLossBuffer.End);
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
    ReadOnlySequence<byte> infiniteLoopBuffer = result.Buffer;
    if (result.IsCompleted && infiniteLoopBuffer.IsEmpty)
    {
        break;
    }
    
    Process(ref infiniteLoopBuffer, out Message message);
    
    reader.AdvanceTo(infiniteLoopBuffer.Start, infiniteLoopBuffer.End);
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
    ReadOnlySequence<byte> hangBuffer = result.Buffer;
    
    Process(ref hangBuffer, out Message message);
    
    if (result.IsCompleted)
    {
        break;
    }
    
    reader.AdvanceTo(hangBuffer.Start, hangBuffer.End);
    
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
    ReadOnlySequence<byte> thisWillOOM = result.Buffer;
    
    Process(ref thisWillOOM, out Message message);
    
    if (result.IsCompleted)
    {
        break;
    }
    
    reader.AdvanceTo(thisWillOOM.Start, thisWillOOM.End);
    
    if (message != null)
    {
        return message;
    }
}
```

❌ **Memory Corruption**

When writing helpers that read the buffer any returned payload should be copied before calling Advance. The example below will return memory that the `Pipe` has discarded and may reuse for the next operation (read/write).

```C#
public class Message
{
   public ReadOnlySequence<byte> Payload { get; set; }
}

// These code samples will result in data loss, hangs, security issues and should **NOT** be copied. They exists solely for illustration of the gotchas mentioned above.
Environment.FailFast("This code is terrible, don't use it!");
while (true)
{
    ReadResult result = await reader.ReadAsync(cancellationToken);
    ReadOnlySequence<byte> buffer = result.Buffer;
    
    ReadHeader(ref buffer, out int length);
    
    if (length > 0)
    {
        message = new Message 
        {
            // Slice the payload from the existing buffer
            Payload = buffer.Slice(0, length);
        };
        
        buffer = buffer.Slice(length);
    }
    
    if (result.IsCompleted)
    {
        break;
    }
    
    reader.AdvanceTo(buffer.Start, buffer.End);
    
    if (message != null)
    {
        // This code is broken since we called reader.AdvanceTo() with a position *after* the buffer we captured
        return message;
    }
}
```

## [PipeWriter](https://docs.microsoft.com/en-us/dotnet/api/system.io.pipelines.pipewriter?view=dotnet-plat-ext-2.1)

## Streams

When reading streaming data it is very common to read data using a de-serializer or write data using a serializer. Most of these APIs take `Stream` today. In order to make it easier to integrate with these existing APIs `PipeReader` and `PipeWriter` expose an `AsStream` which will return a `Stream` implementation around the `PipeReader` or `PipeWriter`. 


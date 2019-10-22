# Distributed Tracing Overview

Distributed tracing is the equivalent of call stacks for modern cloud and microservices architectures. It helps debugging failures and investigating performance issues.

# Activity

## What is Activity

Activity is .NET concept similar to [OpenTracing Span](https://opentracing.io/docs/overview/spans/). It represents operations such as incoming or outgoing requests or just logical calls within a service.
Activity carries tracing context that uniquely identifies `Activity` and also points to the end-to-end operation this `Activity` is part of. Check out more about context in [W3C Trace-Context](#w3c-trace-context). Besides trace context, activity carries timestamp, duration and arbitrary property bag for additional details.

```csharp
var activity = new Activity("foo")
     .SetIdFormat(ActivityIdFormat.W3C)
     .Start();

activity.Stop();

LogActivity(activity);

void LogActivity(Activity activity)
{
     Console.WriteLine("[{0:o}] Activity '{1}': duration={2:F3} ms, TraceId='{3}', SpanId='{4}', ParentSpanId={5}",
          activity.StartTimeUtc,
          activity.OperationName,
          activity.Duration.TotalMilliseconds,
          activity.TraceId.ToHexString(),
          activity.SpanId.ToHexString(),
          activity.ParentSpanId.ToHexString());
}
```

Activity is ambient - it flows with async calls and you can always access `Activity.Current` to get or put some information.

```csharp
var activity = new Activity("foo")
     .SetIdFormat(ActivityIdFormat.W3C)
     .Start();

await DoWork();

activity.Stop();

LogActivity(activity);

async Task DoWork()
{
     try
     {
          // do some work
          throw new Exception("123");
     }
     catch (Exception e)
     {
          Console.WriteLine("{0}: TraceId='{1}', SpanId={2}", e,
              Activity.Current.TraceId.ToHexString(),
              Activity.Current.SpanId.ToHexString());
          // process exception
     }
}

```

New activity should be created for each unit of work.

For incoming HTTP requests new `Activity` is created and parent context comes from the headers. This is done automatically in [ASP.NET Core](#asp.net-core).

When you start a new `Activity`, it automatically captures parent information if there is one.
So when outgoing request starts, you only need to start new `Activity` without providing any parent information. For outgoing HTTP requests context needs to be propagated and [HttpClient in .NET Core](#httpclient) does it automatically.

## Built in support 
### ASP.NET Core
### HttpClient
### gRPC

## DiagnosticSource

`Activity` carries and propagates context and applications expect to log events about incoming and outgoing requests additionally. `DiagnosticSource` is used by ASP.NET Core. HttpClient and other libraries to notify interested in-process listeners when activities initiated by the library start or stop.
`DiagnosticSource` allows to subscribe to interesting sources and events, each event may carry rich payloads containing raw objects like HTTP requests and responses.

Check out [DiagnosticSource User Guide](https://github.com/dotnet/corefx/blob/master/src/System.Diagnostics.DiagnosticSource/src/DiagnosticSourceUsersGuide.md) for more info.

# Tools

Tools offer distribute tracing data collection, analysis and visualizations. [Zipkin](#zipkin) and [Jaeger](#jaeger) are the most popular open source tools you host and manage yourself. [Azure Monitor](#azure-monitor) is Microsoft tool that provides observability into applications and Azure infrastructure capabilities. There are multiple offerings from cloud providers and independent vendors with distributed tracing support.

## OpenTelemetry

[OpenTelemetry](https://opentelemetry.io/) provides a single set of libraries, collectors, agents to collect distributed traces and metrics while allowing to use any tool on the backend to visualize and store the data. OpenTelemetry is the result of merge between [OpenTracing](https://opentracing.io/) and [OpenCensus](https://opencensus.io/)

## Azure Monitor

Azure Monitor (aka Application Insights) [provides SDKs](https://docs.microsoft.com/azure/azure-monitor/app/asp-net-core) for telemetry collection as well as rich capabilities for distributed tracing such as [Application Map](https://docs.microsoft.com/azure/azure-monitor/app/app-map) and [Transactions diagnostics](https://docs.microsoft.com/en-us/azure/azure-monitor/app/transaction-diagnostics). Beyond distributed tracing, Azure Monitor supports metrics, logs, exceptions, and custom events.

## Jaeger

[Jaeger](https://www.jaegertracing.io/) provides visualizations for distributed transaction. Jaeger also provides collection libraries that implement [OpenTracing API](https://opentracing.io).

## Zipkin

[Zipkin](https://zipkin.io/) is open source tool to visualize distributed transactions. Zipkin community provides set of [collection libraries](https://zipkin.io/pages/tracers_instrumentation.html) for multiple languages.

# W3C Trace-Context

[Trace-Context](https://www.w3.org/TR/trace-context/) is W3C standard proposal in Candidate Recommendation (CR) status. It's supported by Microsoft, Google, Dynatrace, New Relic and other active tracing providers. Historically every team or tracing tool implemented their ways to correlate telemetry within distributed system boundaries or tool users.

The Trace-Context describes what is the context and how to propagate it over HTTP.

Example of headers:

```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b9c7c989f97918e1-01
tracestate: congo=ucfJifl5GOE,rojo=00f067aa0ba902b7
```

**traceparent** is required header that contains:
- *protocol version* - `00`
- *trace-id* - globally unique identifier that persist for sub-requests and sub-operations. Populated as 16 bytes long, hex encoded string - `0af7651916cd43dd8448eb211c80319c` 
- *parent-id* - unique identifier of outgoing request within the trace. Populated as 8 bytes long, hex encoded string - `b9c7c989f97918e1`
- *trace flags* - control flags such as sampling decision. 1 hex encoded byte - `01`.

**tracestate** is optional and tracing-system specific context.

## W3C Trace-Context and Activity

`Activity` supports generating identifiers in Trace-Context format and (de)serializing `traceparent` to/from string into set of identifiers. `Activity` blindly propagates `tracestate` if it was populated.

Set `ActivityIdFormat.W3C` to create Activity in Trace-Context format

```csharp
var activity = new Activity("foo")
     .SetIdFormat(ActivityIdFormat.W3C)
     .Start();
```

Use `SetParentId` to populate Activity with context obtained from the wire if there is no [built-in](Built-in-support) support. No need to populate `ActivityIdFormat.W3C` if your parent follows Trace-Context format.

```csharp
var activity = new Activity("process message");

// read traceparent from the wire
if (queueMessage.TryGetTraceparent(out string traceParent))
{
    activity.SetParentId(parentId)
}

activity.Start();
```

If you use binary protocols, you could find it more efficient to propagate trace context identifiers as byte arrays (e.g. following [binary Trace-Context draft](https://w3c.github.io/trace-context-binary/)).

```csharp
var activity = new Activity("process message");

// read traceparent from the wire
SetParent(activity)

activity.Start();

void SetParent(Activity activity)
{
     if (queueMessage.TryGetTraceParent(out ReadOnlySpan<byte> parentId)
     {
          activity.SetParentId(
               ActivityTraceId.CreateFromBytes(parentId.Slice(2, 16)),
               ActivitySpanId.CreateFromBytes(parentId.Slice(19, 8)),
               (ActivityTraceFlags)parentId[28]);
     }
}

// process message ...

activity.Stop();
```

If you want to propagate context into outgoing requests and there is no [built-in](Built-in-support) support for it, use `Activity.Id` to propagate `traceparent` as string

```csharp
var activity = new Activity("foo")
     .SetIdFormat(ActivityIdFormat.W3C)
     .Start();

queueMessage.AddProperty("traceparent", activity.Id);

// send message ...

activity.Stop();
```

or you can optimize for binary protocols by using [binary Trace-Context draft](https://w3c.github.io/trace-context-binary/).

```csharp
var activity = new Activity("foo")
     .SetIdFormat(ActivityIdFormat.W3C)
     .Start();

SetTraceParent(activity, queueMessage);

// send message ...

activity.Stop();

void SetTraceParent(Activity activity, QueueMessage queueMessage)
{
     Span<byte> traceparent = stackalloc byte[29];
     traceparent[0] = 0;
     traceparent[1] = 0;
     activity.TraceId.CopyTo(traceparent.Slice(2, 16));
     traceparent[18] = 1;
     activity.SpanId.CopyTo(traceparent.Slice(19, 8));
     traceparent[27] = 2;
     traceparent[28] = (byte)activity.ActivityTraceFlags;

     queueMessage.AddProperty("traceparent", activity.Id);
}
```

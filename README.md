# OpenCensus Specs

This is a high level design of the OpenCensus library, some of the API examples may be written 
(linked) in C++, Java or Go but every language should translate/modify them based on language 
specific patterns/idioms. Our goal is that all libraries have a consistent "look and feel".

This repository uses terminology (MUST, SHOULD, etc) from [RFC 2119][RFC2119].

## Overview
Today, distributed tracing systems and stats collection tend to use unique protocols and 
specifications for propagating context and sending diagnostic data to backend processing systems.
This is true amongst all the large vendors, and we aim to provide a reliable implementations in 
service of frameworks and agents. We do that by standardizing APIs and data models.

OpenCensus provides a tested set of application performance management (APM) libraries, such as
Metrics and Tracing, under a friendly OSS license. We acknowlege the polyglot nature of modern 
applications, and provide implementations in all main programming languages including C/C++, 
Java, Go, Ruby, PHP, Python, C#, Node.js, Objective-C and Erlang.

## Ecosystem Design

![alt text][EcosystemLayers]

### Layers

#### Service Exporters
Each backend service SHOULD implement this API to export data to their services.

#### OpenCensus Library
This is what we design/describe in the current document.

#### Manually instrumented Frameworks
We are going to instrument some of the most popular frameworks for each language using the 
OpenCensus library to allow users to get traces/stats when they use these frameworks.

#### Tools for automatic instrumentation
Some of the languages may support libraries for automatic instrumentation. Java for example can use
byte-code manipulation (monkey patching) to provide an agent that automatically instruments a 
binary. Note: not all the languages support this.

#### Application
This is customer's application/binary.

## Library Design

### Namespace and Package
* For details about the library package names structure see [Namespace and Package][NamespaceAndPackage].

### Components
This section focuses on the important components that each OpenCensus library must have to 
support all the functionalities.

Here is a layering structure of the proposed OpenCensus library:

![alt text][LibraryComponents]


#### Context
Some of the features like tracing (distributed tracing) and tagging (possibly others) need a way 
to propagate a specific context (trace, tags) in-process (possibly between threads) and function
calls.

The key elements of the Context Support are:
* Every implementation MUST offer an explicit or implicit generic Context propagation mechanism 
that allows different sub-contexts to be propagated.
* Languages that already have this support, like [Go][goContext] or C# (ExecutionContext), MUST 
use the language supported generic context instead of building their own.
* For an implicit generic context implementation you can look at the Java [io.grpc.Context][gRPCContext].


#### Trace
Trace component is designed to support distributed tracing (see [dapper paper][DapperPaper]).
Census allows functionality beyond data collection and export, for example, tracking active and  
keeping local samples for interesting requests.

The key elements of the API can be broken down as:
* A Span represents a single operation within a trace. Spans can be nested to form a trace tree. 
* Libraries must allow users to record tracing events for a span (attributes, annotations, links, 
etc.).
* Spans are carried in the Context. Libraries MUST provide a way of getting, manipulating,
and replacing the Span in the current context.
* Libraries SHOULD provide a means of dynamically controlling the trace global config at runtime (e
.g. trace sampling rate/probability).
* Libraries SHOULD keep track of active spans and in memory samples based on latency/errors and 
offer ways to access the data.
* Because context must also be propagated across processes, library MUST offer the functionality 
that allows any transport (e.g RPC, HTTP, etc.) systems to encode/decode the “trace context” for 
placement on the wire.

##### Links
* Trace API is described [here][TraceAPI].
* Data model is defined [here][TraceDataModel].

#### Tags
Tags are values propagated through the Context subsystem inside a process and among processes by 
any transport (e.g RPC, HTTP, etc.). For example tags are used by the Stats component to break 
down measurements by arbitrary metadata set in the current process or propagated from a remote 
caller.

The key elements of the Tags component are:
* A tag: this is a key-value pair, where the key is a string, and the value can be one of a 64-bit
integer, a boolean, or a string. The API allows for creating, modifying and querying objects 
representing a tag value.
* A set of tags (with unique keys) carried in the Context. Libraries MUST provide a means 
of manipulating the tags in the context, including adding new tags, replacing tag values, deleting
tags, and querying the current value for a given tag key. 
* Because tags must also be propagated across processes, library MUST offer the functionality that 
allows RPC systems to encode/decode the set of tags for placement on the wire.

##### Links
* TODO: Add links to API definition and data model.

#### Stats
The Stats component is designed to record measurements, dynamically break them down by 
application-defined tags, and aggregate those measurements in user-defined ways. It is designed 
to offer multiple types of aggregation (e.g. distributions) and be efficient (all measurement 
processing is done as a background activity); aggregating data enables reducing the overhead of 
uploading data, while also allowing applications direct access to stats.

The key elements the API MUST provide are:
* Defining what is to be measured (the types of data collected, and their meaning), and how data 
will be aggregated (e.g. into a distribution, cumulative aggregation vs. deltas, etc.). Libraries
must offer ways for customer to define Metrics that make sense for their application, and support
a canonical set for RPC/HTTP systems.
* Recording data - API's for recording measured values. This data is then broken down by tags 
carried in the context (e.g. a tag can have a value that describes the current RPC service/method
name; when RPC latency is recorded, this can be made in a generic call, without having to specify
the exact method), and aggregated as needed (e.g. a histogram of all latency values).
* Accessing the aggregated data. This can be filtered by data type, resource name, etc. This 
allows applications to easily get access to their own data in-process.

##### Links
* TODO: Add links to API definition and data model.

### Supported propagation formats
* Library MUST support the [TraceContext][TraceContextSpecs] format for Trace and Tags components.
* Binary encoding is defined [here](https://github.com/census-instrumentation/opencensus-specs/blob/master/encodings/BinaryEncoding.md)

[EcosystemLayers]: /drawings/EcosystemLayers.png "Ecosystem Layer"
[DapperPaper]: https://research.google.com/pubs/pub36356.html
[goContext]: https://golang.org/pkg/context
[gRPCContext]: https://github.com/grpc/grpc-java/blob/master/context/src/main/java/io/grpc/Context.java
[LibraryComponents]: /drawings/LibraryComponents.png "OpenCensus Library Components"
[NamespaceAndPackage]: https://github.com/census-instrumentation/opencensus-specs/blob/master/NamespaceAndPackage.md
[RFC2119]: https://www.ietf.org/rfc/rfc2119.txt
[TraceAPI]: https://github.com/census-instrumentation/opencensus-specs/blob/master/trace/README.md
[TraceContextSpecs]: https://github.com/TraceContext/tracecontext-spec
[TraceDataModel]: https://github.com/census-instrumentation/opencensus-proto/blob/master/trace/trace.proto

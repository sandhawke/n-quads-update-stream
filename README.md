# n-quads-update-stream

**Protocol for mirroring RDF Datasets via Server-Sent Events**

## Motivation

Sometimes there's an RDF data source (eg a JSON-LD file offered by a
web server) which changes from time to time, and clients would like
track the changes, hopefully in real time, from a browser, and without
much code.

Discarded option 1: Clients could poll. Using [If-None-Match](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-None-Match) this can be fairly efficient.  If a propagation delay of hours is okay, this is probably fine.  Maybe even minutes.  Probably not viable if you want to be synchronized within seconds, though.

Discarded option 2: Using [WebSub](https://www.w3.org/TR/websub/) (standard pub-sub via webhooks) gets rid of the propagation delay.  Unfortunately, requires the client to be available on the web as a server.  Also, while fairly simple, it's not trivial to implement.

Discarded option 3: Using [WebSockets](https://en.wikipedia.org/wiki/WebSocket) would work, but is slightly more complex than necessary.

Discarded option 4: Use an existing RDF patch format.  These are unnecessarily complicated (as simple as some of them are), adding another layer when combined with server-sent-events.

Options 1 and 2 require retransmitting the full contents with every change. Option 3 would require defining an update-description syntax comparable to this spec. Option 4 would require defining a connection to a protocol.

## Design Background

We use [Server-Sent Events (SSE)](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events), a very simple streaming HTTP(S) protocol implemented by browsers as [EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource).  SSE is [standardized by W3C](https://www.w3.org/TR/eventsource/), maintained as [part of the HTML Living Standard](https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events), [available in non-MS browsers](https://caniuse.com/#search=eventsource), and implemented by [various polyfills](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events#Tools) for older browsers.

This protocol operates in terms of [N-Quads](https://www.w3.org/TR/n-quads/) a very simple W3C standard data serialization format, suitable for graphs as well as datasets include named graphs / quads.  N-Quads is verbose, without including support for prefix expansion, but http-level gzip should solve that.  (For example, see [nginx gzip](http://nginx.org/en/docs/http/ngx_http_gzip_module.html).

## Example

Imagine a (somewhat absurd) resource `<http://example.org/curtime>`
which provides an RDF graph of the current time in selected locations,
updated every second. It's available in various RDF syntaxes (JSON-LD,
Turtle, etc), with a preference for Turtle.

```
$ curl http://example.org/curtime
PREFIX <:> <http://example.org/vocab#>
PREFIX xs: <http://www.w3.org/2001/XMLSchema#>
:Boston :timeIs "2018-12-13T13:14:13-05:00^^xs:dateTimeStamp;
        :label "Boston, Massachusetts".
$ curl http://example.org/curtime
PREFIX <:> <http://example.org/vocab#>
PREFIX xs: <http://www.w3.org/2001/XMLSchema#>
:Boston :timeIs "2018-12-13T13:16:40-05:00"^^xs:dateTimeStamp;
        :label "Boston, Massachusetts".
```

Let's look at the N-Quads version of that data:

```
$ curl http://example.org/curtime.nq
<http://example.org/vocab#Boston> <http://example.org/vocab#timeIs> "2018-12-13T13:16:40-05:00"^^<http://www.w3.org/2001/XMLSchema#dateTimeStamp>
<http://example.org/vocab#Boston> <label> "Boston, Massachusetts".
```

Now, the server makes available an n-quads-update-stream of that resource:

```
$ curl http://example.org/curtime.nqupdates
event: add
data: <http://example.org/vocab#Boston> <http://example.org/vocab#timeIs> "2018-12-13T13:16:40-05:00"^^<http://www.w3.org/2001/XMLSchema#dateTimeStamp>
data: <http://example.org/vocab#Boston> <label> "Boston, Massachusetts".

event: remove
<http://example.org/vocab#Boston> <http://example.org/vocab#timeIs> *

event: add
data: <http://example.org/vocab#Boston> <http://example.org/vocab#timeIs> "2018-12-13T13:16:41-05:00"^^<http://www.w3.org/2001/XMLSchema#dateTimeStamp>

event: remove
<http://example.org/vocab#Boston> <http://example.org/vocab#timeIs> *

event: add
data: <http://example.org/vocab#Boston> <http://example.org/vocab#timeIs> "2018-12-13T13:16:42-05:00"^^<http://www.w3.org/2001/XMLSchema#dateTimeStamp>

event: remove
<http://example.org/vocab#Boston> <http://example.org/vocab#timeIs> *

event: add
data: <http://example.org/vocab#Boston> <http://example.org/vocab#timeIs> "2018-12-13T13:16:43-05:00"^^<http://www.w3.org/2001/XMLSchema#dateTimeStamp>
^C
```

That event stream will continue until interrupted.

## Event Types

### add

Add quads.  Each "data" element is an N-Quads line (aka an N-Quads statement).

N-Quads comments are not allowed.

For example:

```
event: add
data: <http://example.org/vocab#Boston> <http://example.org/vocab#timeIs> "2018-12-13T13:16:40-05:00"^^<http://www.w3.org/2001/XMLSchema#dateTimeStamp>
data: <http://example.org/vocab#Boston> <label> "Boston, Massachusetts".
```

### remove

Each "data" element is an N-quads line **pattern**, which is an
N-Quads statement where each of the four parts could be a
wildcard. That is:

```
statementPattern ::= (subject | "*") (predicate | "*") (object | "*") (graphLabel | "*")? "."
```

The wildcard matches any value.  A wildcard in the graphLabel position
also matches the default graph.

For example:

```
event: remove
data: _:x * * *
```

would remove from the default graph and every named graph any RDF
statements which have the blank node identified by "_:x" as their
subject.

### check-sha256-hex (optional)

Available for error detection, to confirm the state is correct.
Checksum should be as it would be on a file containing only the
non-removed triples, in the order added. N-Quads has some flexibility
about newlines; for computing this hash, each statement must be
treated as followed by exactly one newline (\n, 0x0a).

For reference, this can be computed using `openssl dgst -sha256`.

Issue: Are there characters in strings that could be escaped different
ways? Is unicode norminalization an issue.

Note that this definition means clients need to remember the order
quads arrived if they want to check this hash.  If this turns out to
be burdensome we could define another hash based on quads in a sorted
order or an XOR of per-quad hashes.

The hash of the empty dataset:

```
event: check-sha256-hex
data: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

### check-count (optional)

Used for error detection, to confirm the state is correct.

The count of the empty dataset:

```
event: check-count
data: 0
```

### set-self (optional)

A way to indicate which node is used for self-reference, to allow for
more clear metadata.

```
event: set-self
data: _:me
```

[Editor's note: Do we need this?  Is there some better way?  See [How
can you embed metadata in an N-Quads file and have it survive the file
being copied/moved/proxied to a different
URL?](https://www.quora.com/unanswered/How-can-you-embed-metadata-in-an-N-Quads-file-and-have-it-survive-the-file-being-copied-moved-proxied-to-a-different-URL).
Maybe we could use a response header, eg rel=canonical.]

### update-response-headers (optional)

Allows the server to inform the client about various metadata it would
have gotten during a GET.

```
event: update-response-headers
data: last-modified: Tue, 11 Dec 2018 10:40:12 GMT
data: etag: "8809-57cbcb4c41b00;89-3f26bd17a2f00"
```

## Blank Nodes

[Blank node
identifiers](https://www.w3.org/TR/rdf11-concepts/#section-blank-nodes)
by definition are "locally scoped to the file or RDF store".  In this
case, the scope is declared to be at least the update stream resource
and the associated n-quads resource, if any. (The scope may be larger,
but that issue is out of scope for this specification.)

This means: blank node identifers are bound to the same blank nodes
throughout an entire n-quads-update-stream, as well as every other
stream obtained from the same stream resource.  In addition, if there
is an associated n-quads resource, the same blank node identifiers
apply there.

Once a blank node no longer appears in any quads, it does not matter
if readers and/or writers remember or reuse the identifier.

## Finding the update stream

Given the URL of some RDF resource, how do you find the
n-quads-update-stream for it?

TBD.  For now, try suffix .nqupdates.  More clean architecture would
use "link rel", with some new relation.  Another option is metadata
within the RDF.

Unfortunately, we can't use content negotiation because SSE mandates
"text/event-stream", and there's no reason to think this will be the
only update stream format for RDF dataset resources.

## See Also

* [TurtlePatch](https://www.w3.org/2001/sw/wiki/TurtlePatch)
* [RDF Patch](https://afs.github.io/rdf-patch/)
* [Linked Data Patch Format](https://www.w3.org/TR/ldpatch/)
* [Delta: Design Issues Diff](https://www.w3.org/DesignIssues/Diff)
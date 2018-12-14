# n-quads-update-stream

**Protocol for mirroring RDF Datasets via Server-Sent Events**

Status: Just a proposal at the moment. 

## Motivation

Sometimes there's an RDF data source (eg a JSON-LD file offered by a web server) which changes from time to time, and clients would like track the changes, hopefully in real time, from a browser, and without much code.

## Design Discussion

Some general design options:

* Clients could poll. Using [If-None-Match](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-None-Match) this can be fairly efficient.  If a propagation delay of hours is okay, this is probably fine.  Maybe even minutes.  Probably not viable if you want to be synchronized within seconds, though.  Requires transmitting the full content with each change, which could also be wasteful if there are many small changes.

* Using [WebSub](https://www.w3.org/TR/websub/) (standard pub-sub via webhooks) gets rid of the propagation delay.  Unfortunately, it requires the client to be available on the web as a server.  Also, while fairly simple, it's not trivial to implement.  Requires transmitting full content with each change, until/unless WebSub is extended to support patches (which would fit the architecture, but is not currently standardized).

* For getting rapid change propagation while working in the browser, we need a "server push" technology.  The main options are [Long Poll](https://en.wikipedia.org/wiki/Push_technology#Long_polling), [Server-Sent Events (SSE)](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events), and [WebSockets](https://en.wikipedia.org/wiki/WebSocket).  At this point, Long Poll appears to be obsolete due to SSE and WebSockets. Since this application only requires one-pay communication, SSE seems like the best option.

SSE is a very simple streaming HTTP(S) protocol, implementable with a few lines of server code, and [supported](https://caniuse.com/#search=eventsource) all modern browsers (except Edge) as [EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource).  ([Polyfills](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events#Tools) exist for Edge and older browsers.)  It was [standardized by W3C](https://www.w3.org/TR/eventsource/) and is now maintained as [part of the HTML Living Standard](https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events).

On the question of what to send:

* [Linked Data Patch Format](https://www.w3.org/TR/ldpatch/) and [the Delta Ontology](https://www.w3.org/DesignIssues/Diff) are unnecessarily complicated, having many additional features.
* [TurtlePatch](https://www.w3.org/2001/sw/wiki/TurtlePatch) and [RDF Patch](https://afs.github.io/rdf-patch/) are much simpler, if we want to simple package general RDF Dataset patch format in an SSE event stream.
* [JSON Patch](http://jsonpatch.com/) could be used to send updates to some JSON-like model of the Dataset, such as one of the forms of JSON-LD. It would require agreeing on a canonical JSON form, and understanding how changes to that JSON structure should impact the dataset's internal representation.  This also seems more complicated than necessary.
* A text-based patch format could be used, like unix diffs, or [jpatch](https://www.npmjs.com/package/jpatch), but this requires keeping the text around on the client, and either very clever algorithms or full re-parsing with every change

Of these options, RDF Patch seems the best existing format, but also has some unnecesary features, especially when layering it on top of SSE.

The selected design is (like RDF Patch) based on [N-Quads](https://www.w3.org/TR/n-quads/), with the basic operations being to add and remove lines of N-Quads text, each of which is a statement representing a triple or quad. N-Quads is verbose, without including support for prefix expansion, but http-level gzip should solve that.  (For example, see [nginx gzip](http://nginx.org/en/docs/http/ngx_http_gzip_module.html).  Some quick testing shows >80% compression on this format.)

To signal the addition of lines (each representing a triple/quad), we can simply use an event-stream like this:

```
event: add-lines
data: <http://example.org/vocab#Boston> <http://example.org/vocab#label> "Boston, Massachusetts".
data: <http://example.org/vocab#Boston> <http://example.org/vocab#population> "625000"^^<http://www.w3.org/2001/XMLSchema#integer>.
```

There are some design options for how to remove lines.
1. Send the full text, as it was added. This can only remove one line at a time
2. Send some of the text, and use a wildcard or regex mechanism, allowing shorter deletion messages and removing multiple lines at once, under certain conditions. This includes the useful feature of being able to delete all quads to start again, when the state of the store is unknown. It is not clear how to generate wildcard of regex expressions matching actually changes made by some opaque application.
3. Send line numbers, with optional ranges. This is easy to implement and terse, and allows removing all quads, but requires the client maintain some kind of numbering.  There are two variations on this: sequence numbers (this was the 47th quad added) and index numbers (this is the 47th quad currently held, in the order they were added).

The current design goes with the last option, index numbers, counting from zero.  So, to change the boston population from 625000 to 685000, we could send these two events:

```
event: remove-lines
data: 1

event: add-lines
data: <http://example.org/vocab#Boston> <http://example.org/vocab#population> "685000"^^<http://www.w3.org/2001/XMLSchema#integer>.
```

We note that there is nothing N-Quads specific about these two events. We can consider this a generic patch format for line-oriented content, which is useful to us because of the line orientation of N-Quads. Implementations are *not* expected to need to retain the text of, however, just the quads.

## Example

Imagine a (somewhat absurd) resource `<http://example.org/curtime>` which provides an RDF graph of the current time in selected locations, updated every second. It's available in various RDF syntaxes (JSON-LD, Turtle, etc), with a preference for Turtle.

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
<http://example.org/vocab#Boston> <http://example.org/vocab#label> "Boston, Massachusetts".
```

Now, imagine the server makes available an n-quads-update-stream of that resource:

```
$ curl http://example.org/curtime.nqupdates
event: add-lines
data: <http://example.org/vocab#Boston> <http://example.org/vocab#timeIs> "2018-12-13T13:16:40-05:00"^^<http://www.w3.org/2001/XMLSchema#dateTimeStamp>
data: <http://example.org/vocab#Boston> <http://example.org/vocab#label> "Boston, Massachusetts".

event: remove-lines
<http://example.org/vocab#Boston> <http://example.org/vocab#timeIs> *

event: add-lines
data: <http://example.org/vocab#Boston> <http://example.org/vocab#timeIs> "2018-12-13T13:16:41-05:00"^^<http://www.w3.org/2001/XMLSchema#dateTimeStamp>

event: remove-lines
<http://example.org/vocab#Boston> <http://example.org/vocab#timeIs> *

event: add-lines
data: <http://example.org/vocab#Boston> <http://example.org/vocab#timeIs> "2018-12-13T13:16:42-05:00"^^<http://www.w3.org/2001/XMLSchema#dateTimeStamp>

event: remove-lines
<http://example.org/vocab#Boston> <http://example.org/vocab#timeIs> *

event: add-lines
data: <http://example.org/vocab#Boston> <http://example.org/vocab#timeIs> "2018-12-13T13:16:43-05:00"^^<http://www.w3.org/2001/XMLSchema#dateTimeStamp>
^C
```

That event stream will continue until interrupted, adding another 'add' and 'remove' event each second.

## Event Types

### add-lines

Add quads.  Each "data" element is an N-Quads line (aka an N-Quads statement).

N-Quads comments, text starting with '#' outside of a term, MUST be ignored, and like blank lines, they do not count for the index (which is counting quads, not lines).  Comments MUST NOT be included in a content-hash, which means N-Quads documents which include comments cannot have their hashes computed by generic text tools.

Each line must end with a period (alas). 

For example:

```
event: add-lines
data: <http://example.org/vocab#Boston> <http://example.org/vocab#timeIs> "2018-12-13T13:16:40-05:00"^^<http://www.w3.org/2001/XMLSchema#dateTimeStamp>.
data: <http://example.org/vocab#Boston> <http://example.org/vocab#label> "Boston, Massachusetts".
```

### remove-lines

Each "data" element is either a decimal index number of a quad in the current store to remove, or a hyphenated pair of two such numbers, which means to remove both of them, and every quad in between.

For example:

```
event: remove-lines
data: 3-6
```

means to remove the 4th, 5th, 6th, and 7th quads from the current
list, considering all adds and removes up until this point.

A single remove even may have multiple such numbers and ranges, separated by a newline, like this:

```
event: remove-lines
data: 3-6
data: 9
data: 0
```

Each line is processed in order, so the deletion of the first element, at the end, does not affect the numbering of the earlier entries.

As a special case, the keyword "end" stands for the highest current index.

### update-response-headers (optional)

This allows the server to inform the client about various metadata it would have gotten during a GET of the N-Quads file. It can be used at the start, to provide metadata that's doesn't apply to the event stream, and also as data changes.

For example:

```
event: update-response-headers
data: last-modified: Tue, 11 Dec 2018 10:40:12 GMT
data: etag: "8809-57cbcb4c41b00;89-3f26bd17a2f00"
```

These header fields MUST be the same as a client would have received doing a GET on the associated N-Quads resource at approximately the same point in time.

Some additional header fields are suggested for use in this context which are not standard.  They will be proposed to IETF if/when they prove somewhat useful:

### Lines

* "Lines" is the count of lines in the current state, for error detection. (This field is already defined, from NNTP, but deprecated.) For example:

```
event: remove-lines
data: 0-end

event: update-response-headers
data: Lines: 0

event: add-lines
data: <https://example.org/#a> <https://example.org/#b> <https://example.org/#c>.

event: update-response-headers
data: Lines: 1
```

### Link rel=self

* "Link" with "rel=self" (originally from [atom](https://tools.ietf.org/html/rfc4287)) is used to indicate the URL by which the Dataset refers to itself, for its metadata.  For example:

```
event: update-response-headers
data: Link: <http://example.org/ds1>; rel=self

event: add-lines
data: <http://example.org/ds1> <http://purl.org/dc/terms/creator> "Alice Example".
```

Here, the dataset is telling us the name of its creator.

Editor's Note: this use of rel=self needs to be tested and more widely considered.  See [How can you embed metadata in an N-Quads file and have it survive the file being copied/moved/proxied to a different URL?](https://www.quora.com/unanswered/How-can-you-embed-metadata-in-an-N-Quads-file-and-have-it-survive-the-file-being-copied-moved-proxied-to-a-different-URL)

### Version-Integrity

* "Version-Integrity" allows specifying a secure hash of what would be the N-Quads file representing the current state (with any comments removed), computed as per (Subresource Integrity)[https://www.w3.org/TR/SRI/] and [Version Integrity](https://github.com/sandhawke/version-integrity).

For example:

```
event: remove-lines
data: 0-end

event: update-response-headers
data: Version-Integrity: sha256-47DEQpj8HBSa-_TImW-5JCeuQeRkm5NMpJWZG3hSuFU=

event: add-lines
data: <https://example.org/#a> <https://example.org/#b> <https://example.org/#c>.

event: update-response-headers
data: Version-Integrity: sha256-ejVcS-5rqOG7TXp8VZ7wRKLuLEmbOvp4HyT1YULD1fg=
```

These strings can be used to make sure changes are being processed properly, and potentially to resume an event string from a given state.

Design alternatives:
* We could assume if the etag looks like a resource-integrity string (starting with "shaNNN"), it is one.  It seems unlikely IETF would like this header, after removing Content-MD5.
* We could use rel=canonical with a version-integrity URL.

Issue: Are there characters in strings that could be escaped different ways? Like, will clients have to remember the form the string arrived in?  Is unicode norminalization an issue?  Maybe clients should confirm their re-serialization of each quad matches the input they parsed, and if they do not match, then remember an exception for this quad.

## Blank Nodes

[Blank node
identifiers](https://www.w3.org/TR/rdf11-concepts/#section-blank-nodes) by definition are "locally scoped to the file or RDF store".  In this case, the scope is declared to be at least the update stream resource and the associated n-quads resource. (The scope may be larger, but that issue is out of scope for this specification.)

This means: blank node identifers are bound to the same blank nodes throughout an entire n-quads-update-stream, as well as every other stream obtained from the same stream resource.  In addition, if there is an associated n-quads resource, the same blank node identifiers apply there.

Once a blank node no longer appears in any quads, it does not matter if readers and/or writers remember or reuse the identifier.

## Finding the update stream

Given the URL of some RDF resource, how do you find the n-quads-update-stream for it?

TBD.  For now, try suffix .nqupdates.  More clean architecture would use "link rel", with some new relation.  Another option is metadata within the RDF.

Unfortunately, we can't use content negotiation because SSE mandates "text/event-stream", and there's no reason to think this will be the only update stream format for RDF dataset resources.

## Introduction
HTTP protocol is a request/response protocol. HTTP communication is initiated by a *user agent* and consists of a request to be applied to a resource on some *origin server*. This is a more simpler situation. In case of more complicated example, one or more intermediaries such as:
- Proxy: forwarding agent, receiving requests for a URI in its absolute form, rewriting all or part of the message, and forwarding the reformatted request toward the server identified by the URI
- Gateway: receiving agent, acting as a layer above some other server(s) and, if necessary, translating the requests to the underlying server's protocol
- Tunnel: acts as a relay point between two connections without changing the messages; tunnels are used when the communication needs to pass through an intermediary (such as a firewall) even when the intermediary cannot understand the contents of the messages

Depending upon the number of intermediaries, number of connections existing would increase/descrease. Some HTTP communication options may apply only to the connection with the nearest, non-tunnel neighbor, only to the end-points of the chain, or to all connections along the chain.

Any party to the communication which is not acting as a tunnel may employ an internal cache for handling requests.

HTTP communication usually takes place over TCP/IP connections. The default port is TCP 80, but other ports can be used. This does not preclude HTTP from being implemented on top of any other protocol.

## Protocol Parameters
### Version
Defined as `HTTP/<major_version>.<minor_version>`

### URI
Uniform Resource Identifiers are simply formatted strings which identify--via name, location, or any other characteristic--a resource. URIs in HTTP can be represented in absolute form or relative to some known base URI. Absolute URIs always begin with a scheme name followed by a colon.

The HTTP protocol does not place any limit on the length of a URI. It is upto server to have capability to accept long length URIs. Server should return 414 (Request-URI Too Long) status if a URI is longer than the server can handle.

HTTP URL has syntax: `http://host [ :port ] [ abs_path [ ?query ]]`. If port is not specified, port 80 is assumed. An empty abs_path is equivalent to an abs_path of "/".

### Content Coding
Indicates an encoding transformation that has been or can be applied to an entity. Content codings are primarily used to allow a document to be compressed or otherwise usefully transformed without losing the identity of its underlying media type and without loss of information. Frequently, the entity is stored in   coded form, transmitted directly, and only decoded by the recipient.

HTTP/1.1 uses content-coding values in the `Accept-Encoding` and `Content-Encoding` header fields. The values can be:
- gzip: encoding format produced by the file compression program "gzip"
- compress: encoding format produced by the common UNIX file compression program "compress"
- deflate: "zlib" format in combination with "deflate" compression mechanism
- br: using "Brottli" algorithm created by Google
- identity: no transformation whatsoever. This content-coding is used only in the `Accept-Encoding` header, and should not be used in the `Content-Encoding` header.

### Media Types
HTTP uses Internet Media Types in the `Content-Type` and `Accept` header fields in order to provide open and extensible data typing and type negotiation. The value has the format: `type "/" subtype *( ";" parameter )`

### Quality Values
HTTP content negotiation uses short "floating point" numbers (lying between 0 and 1) to indicate the relative importance ("weight") of various negotiable parameters. If a parameter has a quality value of 0, then content with this parameter is not acceptable for the client.

## HTTP Message
An HTTP message is either a HTTP request or a response. In both HTTP request or response, the format is something like:
```
start-line
*(message-header CRLF)
CRLF
[ message-body ]
```
Start line is either a request line or a status line. Example message:
```
GET / HTTP/1.1
Host: google.com
User-Agent: curl/7.83.1
Accept: */*
```
Or
```
HTTP/1.1 301 Moved Permanently
Location: http://www.google.com/
Content-Type: text/html; charset=UTF-8
Date: Wed, 07 Dec 2022 18:21:01 GMT
Expires: Fri, 06 Jan 2023 18:21:01 GMT
Cache-Control: public, max-age=2592000
Server: gws
Content-Length: 219
X-XSS-Protection: 0
X-Frame-Options: SAMEORIGIN

<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```

### Message Header
Message header is a "field-name: field-value" pair, where the field-name is case-insensitive. The order in which header fields with differing field names are   received is not significant. Multiple message-header fields with the same field-name MAY be present in a message if and only if the entire field-value for that header field is defined as a comma-separated list.

### Message Body
The rules for when a message-body is allowed in a message differ for requests and responses:
**Requests:** A message-body MUST NOT be included in a request if the specification of the request method does not allow sending an entity-body in requests. From server's perspective, message body should be ignored if the request method does not include defined semantics for an entity-body.

**Response:** whether or not a message-body is included with a message is dependent on both the request method and the response status code.
- Response to request with method HEAD MUST NOT contain message body
- All 1xx (informational), 204 (no content), and 304 (not modified) responses MUST NOT include a message-body.

## Request
The request message format is:
```
Request-Line
*(( general-header
    | request-header
    | entity-header ) CRLF)
CRLF
[ message-body ] 
```
Where the Request Line is: `Method<space>Request-URI<space>HTTP-Version CRLF`. 

### Request Method
Can be either of
- OPTIONS: this method represents a request for information about the communication options available on the request/response chain identified by the Request-URI. Responses to this method are not cacheable. Example:
```
OPTIONS / HTTP/1.1
Host: example.com
User-Agent: curl/7.83.1
Accept: */*
```
Response:
```
HTTP/1.1 200 OK
Allow: OPTIONS, GET, HEAD, POST
Cache-Control: max-age=604800
Content-Type: text/html; charset=UTF-8
Date: Wed, 07 Dec 2022 18:49:22 GMT
Expires: Wed, 14 Dec 2022 18:49:22 GMT
Server: EOS (vny/0453)
Content-Length: 0
```


- GET: method means retrieve whatever information (in the form of an entity) is identified by the Request-URI. The semantics of the GET method change to a  "partial GET" if the request message includes a Range header field. A partial GET requests that only part of the entity be transferred.

- HEAD: identical to GET except that the server MUST NOT return a message-body in the response. This method is often used for testing hypertext links for validity, accessibility, and recent modification.

- POST: method is used to request that the origin server accept the entity enclosed in the request as a new subordinate of the resource identified by the Request-URI in the Request-Line (eg: a file is subordinate to a directory containing it, a news article is subordinate to a newsgroup to which it is posted, or a record is subordinate to a database). It convers the following functions:
    - Annotation of existing resources;
    - Posting a message to a bulletin board, newsgroup, mailing list, or similar group of articles
    - Providing a block of data, such as the result of submitting a form, to a data-handling process
    - Extending a database through an append operation.
    
    The result of a POST request:
    - May not result in a resource that can be identified by a URI. In this case, both 200(OK) and 204(No Content) is valid response type depending upon whether there is a message body created or not
    - If a resource has been created on the origin server, the respons eshould be 201(Created) and contain an entity which describes the status of the request and refers to the new resource, and a `Location` header.
    
- PUT: this method is often touted as being similar to POST with difference between POST and PUT is that PUT requests are idempotent. That is, calling the same PUT request multiple times will always produce the same result. However, the usage of PUT is more limited as compared to POST. The PUT method requests that the enclosed entity be stored under the supplied Request-URI. If the Request-URI refers to an already existing resource, the enclosed entity SHOULD be considered as a modified version of the one residing on the origin server. If a new resource is created, the origin server MUST inform the user agent via the 201(Created) response. If an existing resource is modified, either the 200(OK) or 204(No Content) response codes SHOULD be sent to indicate successful completion of the request.

- DELETE: this method requests that the origin server delete the resource identified by the Request-URI. A successful response SHOULD be 200(OK) if the response includes an entity describing the status, 202(Accepted) if the action has not yet been enacted, or 204(No Content) if the action has been enacted but the response does not include an entity.

- TRACE
- CONNECT

**Idempotency:** Methods can also have the property of "idempotence" in that (aside from error or expiration issues) the side-effects of $N > 0$ identical requests is the same as for a single request. The methods GET, HEAD, PUT and DELETE share this property. Also, the methods OPTIONS and TRACE SHOULD NOT have side effects, and so are inherently idempotent. Idempotency doesn't mean the response will always be the same, it means that the state of the system will remain unchanged.

### Request URI
The Request-URI can be either of "*", "absolute_uri", "abs_path" or "authority"
- The asterisk "*" means that the request does not apply to a particular resource, but to the server itself. Example: `OPTIONS * HTTP/1.1`
- The absolute_uri form is REQUIRED when the request is being made to a proxy. The proxy is requested to forward the request or service it from a valid cache, and return the response. Example: `GET http://www.w3.org/pub/WWW/TheProject.html HTTP/1.1`. In this case any `Host` header field value in the request MUST be ignored.
- The abs_path is the most common one, where the path locates some resource. In this case, the network location of the URI (authority) MUST be transmitted in a `Host` header field.
```
GET /pub/WWW/TheProject.html HTTP/1.1
Host: www.w3.org
```

## Response
The response message format is:
```
Status-Line
*(( general-header
    | response-header
    | entity-header ) CRLF)
CRLF
[ message-body ]
```
Where the Status-Line is `HTTP-Version<space>Status-Code<space>Reason-Phrase CRLF`. The Reason-Phrase is intended to give a short textual description of the Status-Code.

### Status Code
It is a 3-digit integer result code of the attempt to understand and satisfy the request. The first digit of the Status-Code defines the class of response:
- 1xx: Informational - Request received, continuing process
- 2xx: Success - The action was successfully received, understood, and accepted
- 3xx: Redirection - Further action must be taken in order to complete the request
- 4xx: Client Error - The request contains bad syntax or cannot be fulfilled
- 5xx: Server Error - The server failed to fulfill an apparently valid request

Status codes:
- 100: Continue
- 101: Switching Protocols
- 200: OK
- 201: Created
- 202: Accepted
- 203: Non-Authoritative Information
- 204: No Content
- 205: Reset Content
- 206: Partial Content
- 300: Multiple Choices
- 301: Moved Permanently
- 302: Found
- 303: See Other
- 304: Not Modified
- 305: Use Proxy
- 307: Temporary Redirect
- 400: Bad Request
- 401: Unauthorized
- 402: Payment Required
- 403: Forbidden
- 404: Not Found
- 405: Method Not Allowed
- 406: Not Acceptable
- 407: Proxy Authentication Required
- 408: Request Time-out
- 409: Conflict
- 410: Gone
- 411: Length Required
- 412: Precondition Failed
- 413: Request Entity Too Large
- 414: Request-URI Too Large
- 415: Unsupported Media Type
- 416: Requested range not satisfiable
- 417: Expectation Failed
- 500: Internal Server Error
- 501: Not Implemented
- 502: Bad Gateway
- 503: Service Unavailable
- 504: Gateway Time-out
- 505: HTTP Version not supported

HTTP status codes are extensible.

## Connections
Prior to persistent connections, a separate TCP connection was established to fetch each URL, increasing the load on HTTP servers and causing congestion on the Internet. A significant difference between HTTP/1.1 and earlier versions of HTTP is that persistent connections are the default behavior of any HTTP connection. Persistent connections provide a mechanism by which a client and a server can signal the close of a TCP connection. This signaling takes place using the Connection header field.

An HTTP/1.1 server MAY assume that a HTTP/1.1 client intends to maintain a persistent connection unless a `Connection` header including value "close" was sent in the request. If the server chooses to close the connection immediately after sending the response, it SHOULD send a `Connection` header including thevalue "close". If either the client or the server sends the close token in the `Connection` header, that request becomes the last one for the connection.

### Pipelining
A client that supports persistent connections MAY "pipeline" its requests (i.e., send multiple requests without waiting for each response). A server MUST send its responses to those requests in the same order that the requests were received. Clients SHOULD NOT pipeline requests using non-idempotent methods or non-idempotent sequences of methods (see section 9.1.2). Otherwise, a premature termination of the transport connection could lead to indeterminate results.

### Practical Considerations
Servers will usually have some time-out value beyond which they will no longer maintain an inactive connection. Proxy servers might make this a higher value since it is likely that the client will be making more connections through the same server. The use of persistent connections places no requirements on the length (or existence) of this time-out for either the client or the server.

Clients that use persistent connections SHOULD limit the number of simultaneous connections that they maintain to a given server. A single-user client SHOULD NOT maintain more than 2 connections with any server or proxy. A proxy SHOULD use up to $2N$ connections to another server or proxy, where N is the number of simultaneously active users. These guidelines are intended to improve HTTP response times and avoid congestion.

## Content Negotiation
HTTP has provisions for several mechanisms for "content negotiation" -- the process of selecting the best representation for a given response when there are multiple representations available. Any response containing an entity-body MAY be subject to negotiation, including error responses.

There are two kinds of content negotiation which are possible in HTTP: server-driven and agent-driven negotiation.

### Server Driven Negotiation
If the selection of the best representation for a response is made by an algorithm located at the server, it is called server-driven negotiation. HTTP/1.1 includes the following request-header fields for enabling server-driven negotiation through description of user agent capabilities and user preferences: `Accept`, `Accept-Charset` , `Accept-Encoding`, `Accept-Language`, and `User-Agent`. However, an origin server is not limited to these dimensions and MAY vary the response based on any aspect of the request, including information outside the request-header fields or within extension header fields not defined by this specification.

### Agent-driven Negotiation
With agent-driven negotiation, selection of the best representation for a response is performed by the user agent after receiving an initial response from the origin server. Selection is based on a list of the available representations of the response included within the header fields or entity-body of the initial response, with each representation identified by its own URI. Agent-driven negotiation suffers from the disadvantage of needing a second request to obtain the best alternate representation.

## Caching
?

## HTTP Headers
Common HTTP headers:

### Connection
The Connection HTTP header controls whether the current network connection remains open after the transaction. 
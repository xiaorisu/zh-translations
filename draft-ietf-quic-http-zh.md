---
title: Hypertext Transfer Protocol Version 3 (HTTP/3)
abbrev: HTTP/3
docname: draft-ietf-quic-http-latest
date: {DATE}
category: std
ipr: trust200902
area: Transport
workgroup: QUIC

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping]

author:
-
    ins: M. Bishop
    name: Mike Bishop
    org: Akamai
    email: mbishop@evequefou.be
    role: editor

normative:

  QUIC-TRANSPORT:
    title: "QUIC: A UDP-Based Multiplexed and Secure Transport"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-transport-latest
    author:
      -
        ins: J. Iyengar
        name: Jana Iyengar
        org: Fastly
        role: editor
      -
        ins: M. Thomson
        name: Martin Thomson
        org: Mozilla
        role: editor

  QPACK:
    title: "QPACK: Header Compression for HTTP over QUIC"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-qpack-latest
    author:
      -
          ins: C. Krasic
          name: Charles 'Buck' Krasic
          org: Google, Inc
      -
          ins: M. Bishop
          name: Mike Bishop
          org: Akamai Technologies
      -
          ins: A. Frindell
          name: Alan Frindell
          org: Facebook
          role: editor


informative:


--- abstract

The QUIC transport protocol has several features that are desirable in a
transport for HTTP, such as stream multiplexing, per-stream flow control, and
low-latency connection establishment.  This document describes a mapping of HTTP
semantics over QUIC.  This document also identifies HTTP/2 features that are
subsumed by QUIC, and describes how HTTP/2 extensions can be ported to HTTP/3.

--- note_Note_to_Readers

Discussion of this draft takes place on the QUIC working group mailing list
(quic@ietf.org), which is archived at
<https://mailarchive.ietf.org/arch/search/?email_list=quic>.

Working Group information can be found at <https://github.com/quicwg>; source
code and issues list for this draft can be found at
<https://github.com/quicwg/base-drafts/labels/-http>.


--- middle


# Introduction

HTTP semantics are used for a broad range of services on the Internet. These
semantics have commonly been used with two different TCP mappings, HTTP/1.1 and
HTTP/2.  HTTP/2 introduced a framing and multiplexing layer to improve latency
without modifying the transport layer.  However, TCP's lack of visibility into
parallel requests in both mappings limited the possible performance gains.

The QUIC transport protocol incorporates stream multiplexing and per-stream flow
control, similar to that provided by the HTTP/2 framing layer. By providing
reliability at the stream level and congestion control across the entire
connection, it has the capability to improve the performance of HTTP compared to
a TCP mapping.  QUIC also incorporates TLS 1.3 at the transport layer, offering
comparable security to running TLS over TCP, but with improved connection setup
latency (unless TCP Fast Open {{?RFC7413}}} is used).

This document defines a mapping of HTTP semantics over the QUIC transport
protocol, drawing heavily on the design of HTTP/2. This document identifies
HTTP/2 features that are subsumed by QUIC, and describes how the other features
can be implemented atop QUIC.

QUIC is described in {{QUIC-TRANSPORT}}.  For a full description of HTTP/2, see
{{!RFC7540}}.


## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

Field definitions are given in Augmented Backus-Naur Form (ABNF), as defined in
{{!RFC5234}}.

This document uses the variable-length integer encoding from
{{QUIC-TRANSPORT}}.

Protocol elements called "frames" exist in both this document and
{{QUIC-TRANSPORT}}. Where frames from {{QUIC-TRANSPORT}} are referenced, the
frame name will be prefaced with "QUIC."  For example, "QUIC CONNECTION_CLOSE
frames."  References without this preface refer to frames defined in {{frames}}.


# Connection Setup and Management

## Draft Version Identification

> **RFC Editor's Note:**  Please remove this section prior to publication of a
> final version of this document.

HTTP/3 uses the token "h3" to identify itself in ALPN and Alt-Svc.  Only
implementations of the final, published RFC can identify themselves as "h3".
Until such an RFC exists, implementations MUST NOT identify themselves using
this string.

Implementations of draft versions of the protocol MUST add the string "-" and
the corresponding draft number to the identifier. For example,
draft-ietf-quic-http-01 is identified using the string "h3-01".

Non-compatible experiments that are based on these draft versions MUST append
the string "-" and an experiment name to the identifier. For example, an
experimental implementation based on draft-ietf-quic-http-09 which reserves an
extra stream for unsolicited transmission of 1980s pop music might identify
itself as "h3-09-rickroll". Note that any label MUST conform to the "token"
syntax defined in Section 3.2.6 of [RFC7230]. Experimenters are encouraged to
coordinate their experiments on the quic@ietf.org mailing list.

## Discovering an HTTP/3 Endpoint

An HTTP origin advertises the availability of an equivalent HTTP/3 endpoint via
the Alt-Svc HTTP response header field or the HTTP/2 ALTSVC frame
({{!ALTSVC=RFC7838}}), using the ALPN token defined in
{{connection-establishment}}.

For example, an origin could indicate in an HTTP response that HTTP/3 was
available on UDP port 50781 at the same hostname by including the following
header field:

~~~ example
Alt-Svc: h3=":50781"
~~~

On receipt of an Alt-Svc record indicating HTTP/3 support, a client MAY attempt
to establish a QUIC connection to the indicated host and port and, if
successful, send HTTP requests using the mapping described in this document.

Connectivity problems (e.g. firewall blocking UDP) can result in QUIC connection
establishment failure, in which case the client SHOULD continue using the
existing connection or try another alternative endpoint offered by the origin.

Servers MAY serve HTTP/3 on any UDP port, since an alternative always includes
an explicit port.

### QUIC Version Hints {#alt-svc-version-hint}

This document defines the "quic" parameter for Alt-Svc, which MAY be used to
provide version-negotiation hints to HTTP/3 clients. QUIC versions are four-byte
sequences with no additional constraints on format. Leading zeros SHOULD be
omitted for brevity.

Syntax:

~~~ abnf
quic = DQUOTE version-number [ "," version-number ] * DQUOTE
version-number = 1*8HEXDIG; hex-encoded QUIC version
~~~

Where multiple versions are listed, the order of the values reflects the
server's preference (with the first value being the most preferred version).
Reserved versions MAY be listed, but unreserved versions which are not supported
by the alternative SHOULD NOT be present in the list. Origins MAY omit supported
versions for any reason.

Clients MUST ignore any included versions which they do not support.  The "quic"
parameter MUST NOT occur more than once; clients SHOULD process only the first
occurrence.

For example, suppose a server supported both version 0x00000001 and the version
rendered in ASCII as "Q034".  If it also opted to include the reserved version
(from Section 15 of {{QUIC-TRANSPORT}}) 0x1abadaba, it could specify the
following header field:

~~~ example
Alt-Svc: h3=":49288";quic="1,1abadaba,51303334"
~~~

A client acting on this header field would drop the reserved version (not
supported), then attempt to connect to the alternative using the first version
in the list which it does support, if any.

## Connection Establishment {#connection-establishment}

HTTP/3 relies on QUIC as the underlying transport.  The QUIC version being used
MUST use TLS version 1.3 or greater as its handshake protocol.  HTTP/3 clients
MUST indicate the target domain name during the TLS handshake. This may be done
using the Server Name Indication (SNI) {{!RFC6066}} extension to TLS or using
some other mechanism.

QUIC connections are established as described in {{QUIC-TRANSPORT}}. During
connection establishment, HTTP/3 support is indicated by selecting the ALPN
token "h3" in the TLS handshake.  Support for other application-layer protocols
MAY be offered in the same handshake.

While connection-level options pertaining to the core QUIC protocol are set in
the initial crypto handshake, HTTP/3-specific settings are conveyed in the
SETTINGS frame. After the QUIC connection is established, a SETTINGS frame
({{frame-settings}}) MUST be sent by each endpoint as the initial frame of their
respective HTTP control stream (see {{control-streams}}).

## Connection Reuse

Once a connection exists to a server endpoint, this connection MAY be reused for
requests with multiple different URI authority components.  The client MAY send
any requests for which the client considers the server authoritative.

An authoritative HTTP/3 endpoint is typically discovered because the client has
received an Alt-Svc record from the request's origin which nominates the
endpoint as a valid HTTP Alternative Service for that origin.  As required by
{{RFC7838}}, clients MUST check that the nominated server can present a valid
certificate for the origin before considering it authoritative. Clients MUST NOT
assume that an HTTP/3 endpoint is authoritative for other origins without an
explicit signal.

A server that does not wish clients to reuse connections for a particular origin
can indicate that it is not authoritative for a request by sending a 421
(Misdirected Request) status code in response to the request (see Section 9.1.2
of {{!RFC7540}}).

The considerations discussed in Section 9.1 of {{?RFC7540}} also apply to the
management of HTTP/3 connections.

# Stream Mapping and Usage {#stream-mapping}

A QUIC stream provides reliable in-order delivery of bytes, but makes no
guarantees about order of delivery with regard to bytes on other streams. On the
wire, data is framed into QUIC STREAM frames, but this framing is invisible to
the HTTP framing layer. The transport layer buffers and orders received QUIC
STREAM frames, exposing the data contained within as a reliable byte stream to
the application. Although QUIC permits out-of-order delivery within a stream
HTTP/3 does not make use of this feature.

QUIC streams can be either unidirectional, carrying data only from initiator to
receiver, or bidirectional.  Streams can be initiated by either the client or
the server.  For more detail on QUIC streams, see Section 2 of
{{QUIC-TRANSPORT}}.

When HTTP headers and data are sent over QUIC, the QUIC layer handles most of
the stream management.  HTTP does not need to do any separate multiplexing when
using QUIC - data sent over a QUIC stream always maps to a particular HTTP
transaction or connection context.

## Bidirectional Streams

All client-initiated bidirectional streams are used for HTTP requests and
responses.  A bidirectional stream ensures that the response can be readily
correlated with the request. This means that the client's first request occurs
on QUIC stream 0, with subsequent requests on stream 4, 8, and so on. In order
to permit these streams to open, an HTTP/3 client SHOULD send non-zero values
for the QUIC transport parameters `initial_max_stream_data_bidi_local`. An
HTTP/3 server SHOULD send non-zero values for the QUIC transport parameters
`initial_max_stream_data_bidi_remote` and `initial_max_bidi_streams`. It is
recommended that `initial_max_bidi_streams` be no smaller than 100, so as to not
unnecessarily limit parallelism.

These streams carry frames related to the request/response (see
{{request-response}}). When a stream terminates cleanly, if the last frame on
the stream was truncated, this MUST be treated as a connection error (see
HTTP_MALFORMED_FRAME in {{http-error-codes}}).  Streams which terminate abruptly
may be reset at any point in the frame.

HTTP/3 does not use server-initiated bidirectional streams; clients MUST omit or
specify a value of zero for the QUIC transport parameter
`initial_max_bidi_streams`.


## Unidirectional Streams

Unidirectional streams, in either direction, are used for a range of purposes.
The purpose is indicated by a stream type, which is sent as a variable-length
integer at the start of the stream. The format and structure of data that
follows this integer is determined by the stream type.

~~~~~~~~~~ drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Stream Type (i)                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-stream-header title="Unidirectional Stream Header"}

Some stream types are reserved ({{stream-grease}}).  Two stream types are
defined in this document: control streams ({{control-streams}}) and push streams
({{push-streams}}).  Other stream types can be defined by extensions to HTTP/3;
see {{extensions}} for more details.

Both clients and servers SHOULD send a value of three or greater for the QUIC
transport parameter `initial_max_uni_streams`.

If the stream header indicates a stream type which is not supported by the
recipient, the remainder of the stream cannot be consumed as the semantics are
unknown. Recipients of unknown stream types MAY trigger a QUIC STOP_SENDING
frame with an error code of HTTP_UNKNOWN_STREAM_TYPE, but MUST NOT consider such
streams to be an error of any kind.

Implementations MAY send stream types before knowing whether the peer supports
them.  However, stream types which could modify the state or semantics of
existing protocol components, including QPACK or other extensions, MUST NOT be
sent until the peer is known to support them.

A sender can close or reset a unidirectional stream unless otherwise specified.
A receiver MUST tolerate unidirectional streams being closed or reset prior to
the reception of the unidirectional stream header.

###  Control Streams

A control stream is indicated by a stream type of `0x00`.  Data on this stream
consists of HTTP/3 frames, as defined in {{frames}}.

Each side MUST initiate a single control stream at the beginning of the
connection and send its SETTINGS frame as the first frame on this stream.  If
the first frame of the control stream is any other frame type, this MUST be
treated as a connection error of type HTTP_MISSING_SETTINGS. Only one control
stream per peer is permitted; receipt of a second stream which claims to be a
control stream MUST be treated as a connection error of type
HTTP_WRONG_STREAM_COUNT.  The sender MUST NOT close the control stream.  If the
control stream is closed at any point, this MUST be treated as a connection
error of type HTTP_CLOSED_CRITICAL_STREAM.

A pair of unidirectional streams is used rather than a single bidirectional
stream.  This allows either peer to send data as soon they are able.  Depending
on whether 0-RTT is enabled on the connection, either client or server might be
able to send stream data first after the cryptographic handshake completes.

### Push Streams

A push stream is indicated by a stream type of `0x01`, followed by the Push ID
of the promise that it fulfills, encoded as a variable-length integer. The
remaining data on this stream consists of HTTP/3 frames, as defined in
{{frames}}, and fulfills a promised server push.  Server push and Push IDs are
described in {{server-push}}.

Only servers can push; if a server receives a client-initiated push stream, this
MUST be treated as a stream error of type HTTP_WRONG_STREAM_DIRECTION.

~~~~~~~~~~ drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           0x01 (i)                          ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Push ID (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-push-stream-header title="Push Stream Header"}

Each Push ID MUST only be used once in a push stream header. If a push stream
header includes a Push ID that was used in another push stream header, the
client MUST treat this as a connection error of type HTTP_DUPLICATE_PUSH.

### Reserved Stream Types {#stream-grease}

Stream types of the format `0x1f * N + 0x21` for integer values of N are
reserved to exercise the requirement that unknown types be ignored. These
streams have no semantics, and can be sent when application-layer padding is
desired. They MAY also be sent on connections where no data is currently being
transferred. Endpoints MUST NOT consider these streams to have any meaning upon
receipt.

The payload and length of the stream are selected in any manner the
implementation chooses.


# HTTP帧层(HTTP Framing Layer) {#http-framing-layer}


HTTP帧在QUIC流上传输，如{{stream-mapping}}所述。HTTP/3定义了三种流类型：控制流、请求流和推送流。
本节介绍HTTP/3帧格式及其允许的流类型；有关概述，请参见{{stream-frame-mapping}}。{{h2-frames}}
提供了HTTP/2和HTTP/3帧的比较。

| 帧             | 控制流         | 请求流          | 推送流      | 章节                     |
| -------------- | -------------- | -------------- | ----------- | ------------------------ |
| DATA           | No             | Yes            | Yes         | {{frame-data}}           |
| HEADERS        | No             | Yes            | Yes         | {{frame-headers}}        |
| PRIORITY       | Yes            | Yes (1)        | No          | {{frame-priority}}       |
| CANCEL_PUSH    | Yes            | No             | No          | {{frame-cancel-push}}    |
| SETTINGS       | Yes (1)        | No             | No          | {{frame-settings}}       |
| PUSH_PROMISE   | No             | Yes            | No          | {{frame-push-promise}}   |
| GOAWAY         | Yes            | No             | No          | {{frame-goaway}}         |
| MAX_PUSH_ID    | Yes            | No             | No          | {{frame-max-push-id}}    |
| DUPLICATE_PUSH | No             | Yes            | No          | {{frame-duplicate-push}} |
{: #stream-frame-mapping title="HTTP/3帧和流类型概述"}

某些帧只能作为特定流类型的第一帧出现；这些帧在{{stream-frame-mapping}}中用(1)表示。
相关章节提供了具体指导。

## 帧布局(Frame Layout)

所有帧都具有以下格式：

~~~~~~~~~~ drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           类型 (i)                          ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           长度 (i)                          ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          帧荷载 (*)                         ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-frame title="HTTP/3帧格式"}

帧包括以下字段：

  类型:
  : 标识帧类型的可变长度整数。

  长度:
  : 描述帧有效载荷长度的可变长度整数。

  帧荷载:
  : 有效荷载，其语义由Type字段确定。

每个帧的有效荷载**必须**精确地包含在其描述中标识的字段。在标识字段之后包含其他字节
的帧有效荷载或在标识字段结束之前终止的帧荷载**必须**被视为类型为
HTTP_MALFORM_FRAME的连接错误。

## 帧定义(Frame Definitions) {#frames}

### DATA帧(DATA) {#frame-data}

DATA帧(类型为0x0)传递与HTTP请求或响应有效荷载相关联的任意可变长度的字节序列。

DATA帧**必须**与HTTP请求或响应相关联。如果在任一控制流上接收到数据帧，则接收方
**必须**响应类型为HTTP_WRONG_STREAM的连接错误({{errors}})。

~~~~~~~~~~ drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         有效荷载 (*)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-data title="DATA帧有效荷载"}

### HEADERS帧(HEADERS) {#frame-headers}

HEADERS帧(类型为0x1)用于承载头信息块，并使用QPACK进行压缩。有关详细信息，请参见[QPACK]。

~~~~~~~~~~  drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       头信息块 (*)                         ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-headers title="HEADERS帧有效荷载"}

HEADERS帧只能在请求流与推送流发出。

### 优先级帧(PRIORITY) {#frame-priority}


PRIORITY帧(类型=0x02)指定请求、服务器推送或占位符的客户端建议的优先级。

PRIORITY帧标识要优先排序的元素以及它所依赖的元素。优先级ID或依赖ID使用
相应的流ID标识客户端发起的请求，使用推送ID标识服务器推送(请参见
{{frame-push-promise}})，或使用占位符ID标识占位符(请参见{{placeholders}})。

当客户端发起请求时，**可以**将PRIORITY帧作为流的第一帧发送，从而创建对现有
元素的依赖。为了确保以一致的顺序处理优先级，该请求的任何后续PRIORITY帧都
**必须**在控制流上发送。在请求流上的其他帧之后接收到的PRIORITY帧**必须**被视为
HTTP_UNEXPECTED_FRAME类型的流错误。

如果在打开新的请求流时，已通过控制流接收到其优先级信息，则必须忽略在请求
流上发送的PRIORITY帧。

~~~~~~~~~~  drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|PT |DT | 空 |               [优先级元素ID(i)]             ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         [元素依赖ID(i)]                     ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  权重 (8)  |
+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-priority title="PRIORITY frame payload"}

PRIORITY帧有效负载具有以下字段：

  PT (优先元素类型):
  : 一个两位字段，指示被优先处理的元素的类型(参见
    {{prioritized-element-types}}). 在请求流中发送时，
    **必须**将其设置为`11`。在控制流上发送时，**禁止**将其设置为`11`。

  DT (元素依赖类型):
  : 指示所依赖的元素类型的两位字段(参见
    {{element-dependency-types}})。

  空:
  : 发送时**必须**为零且在收到时**必须**忽略的四位字段。

  优先元素ID:
  : 标识要优先处理的元素的可变长度整数。根据优先类型的值，它包含请求流的流ID、
  承诺资源的推送ID、占位符的占位符ID或缺失。

  元素依赖ID:
  : 一个可变长度整数，用于标识表示依赖项的元素。根据依赖类型的值，它包含请求流的
  流ID、承诺资源的推送ID、占位符的占位符ID或缺失。有关依赖项的详细信息，请参见
  {{priority}}和{{!RFC7540}}, 第5.3节.

  权重:
  : 一个无符号8位整数，表示优先元素的优先权重(参见 {{!RFC7540}},第5.3节)。
  向值中添加一个以获得介于1和256之间的权重。

优先元素类型({{prioritized-element-types}})和元素依赖类型({{element-dependency-types}})
的值隐含着关联元素ID字段的解释。

| PT 位 | 类型描述 | 优先元素ID内容 |
| ------- | ---------------- | ------------------------------- |
| 00      | 请求流  | 流ID                      |
| 01      | 推送流      | 推送ID                       |
| 10      | 占位符      | 占位符ID                 |
| 11      | 当前流   | 缺失                         |
{: #prioritized-element-types title="Prioritized Element Types"}

| DT 位   | 类型描述 | 元素依赖ID内容 |
| ------- | ---------------- | ------------------------------ |
| 00      | 请求流   | 流ID                    |
| 01      | 推送流     | 推送ID                   |
| 10      | 占位符      | 占位符ID                |
| 11      | 树的根节点 | 缺失                         |
{: #element-dependency-types title="Element Dependency Types"}

请注意，与{{!RFC7540}}不同，不能使用为0的流ID引用树的根节点，因为
QUIC流0包含有效的HTTP请求。树的根节点不能被重新排序。在优先级元素类型
设置为“11”以外的任何值的请求流上发送的PRIORITY帧，或者在具有大于当前流的
流ID的请求流上表达依赖关系的PRIORITY帧**必须**被视为HTTP_MALFORMED_FRAME
类型的流错误。必须将当前流视为类型为HTTP_MERFORM_FRAME的流错误。
同样，在优先级元素类型设置为`11`的控制流上发送的优先级帧**必须**被视
为HTTP_MALFORMED_FRAME类型的连接错误。

当PRIORITY帧声称引用请求时，关联的ID**必须**标识客户端启动的双向流。
服务器**必须**将接收到标识任何其他类型的流的PRIORITY帧视为类型为
HTTP_MALFORMED_FRAME的连接错误。

引用不存在的推送ID、超过服务器限制的占位符ID或客户端尚不允许打开
的流ID的PRIORITY帧**必须**视为HTTP_LIMIT_EXCEEDED错误。

在请求或控制流以外的任何流上接收到的PRIORITY帧**必须**被视为类型为
HTTP_WRONG_STREAM的连接错误。

客户端接收到的PRIORITY帧**必须**被视为HTTP_UNEXPECTED_FRAME类型
的流错误。

### CANCEL_PUSH帧 {#frame-cancel-push}

CANCEL_PUSH帧(type=0x3)用于在接收到
推送流之前请求取消服务器的推送。
CANCEL_PUSH帧通过推送ID标识服务器的
推送(请参阅{{frame-push-promise}})，
推送ID被编码为可变长度整数。

当服务器接收到此帧时，它将中止发送
帧标识的服务器推送流的响应。
如果服务器尚未开始发送服务器推送，
则可以通过接收CANCEL_PUSH帧来避免
打开推送流。如果服务器已打开推送流，
则服务器应在该流上发送QUIC RESET_STREAM帧
并停止响应的传输。

服务器可以在创建推送流之前发送CANCEL_PUSH帧
以表明它不会创建推送流。
一旦推送流建立后，再发送CANCEL_PUSH将不会
对推送流的状态产生影响。
**应该**改用QUIC RESET_STREAM帧中止服务器推送响应的传输。

CANCEL_PUSH帧是在控制流上发送的。
在控制流以外的流上接收到CANCEL_PUSH帧时，
必须将其视为类型为HTTP_OWRY_STREAM的流错误。

~~~~~~~~~~  drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Push ID (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-cancel-push title="CANCEL_PUSH frame payload"}

CANCEL_PUSH帧携带一个可变长度整数编码的推送ID，
推送ID用于标识被取消的
服务器推送(请参阅{{frame-push-promise}})。

如果客户端接收到CANCEL_PUSH帧，
则该帧可能标识PUSH_PROMISE帧
尚未提及的推送ID。


### SETTINGS帧 {#frame-settings}

SETTINGS帧(type=0x4)用于传递影响端点
通信方式的配置参数，
例如对端行为的默认项和约束。
单独地，SETTINGS参数也可以称为“设置”；
每个设置参数的标识符和值可以称为
“设置标识符”和“设置值”。

SETTINGS帧始终应用于连接，而不是单个流。
每个对端**必须**将SETTINGS帧作为每个
控制流(请参见{{control-streams}})的第一个帧
发送，并且**禁止**在非流的开始时或
在任何其他流上发送SETTINGS帧。
如果终端接收到同个链接不同流上的SETTINGS帧，
则该终端**必须**响应类型为HTTP_OWRY_STREAM的连接错误。
如果终端接收到第二个设置帧，
则该终端**必须**响应类型为HTTP_UNIRECTION_FRAME的连接错误。

SETTINGS参数不是通过协商确立的；
它们描述了接收端可以支持的发送端的特性。
但是，协商可以通过使用SETTINGS来暗中进行-
每个对端使用SETTINGS来通告一组受支持的值。
设置的定义将描述每个对端如何组合这两个集，
以确定将使用哪个选项。SETTINGS
没有提供标识选择何时生效的机制。

每个对端可以通告同一参数的不同值。
例如，客户端可能愿意使用非常大的响应头，
而服务器则对请求大小更加谨慎。

参数**禁止**在SETTINGS帧中出现多次。
接收方**可以**将同一参数的多次出现
视为类型为HTTP_MALMALFORM_FRAME的
连接错误。

SETTINGS帧的有效负载由零个或多个参数组成。
每个参数由一个设置标识符和一个值组成，
两者都编码为QUIC可变长度整数。

~~~~~~~~~~~~~~~  drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Identifier (i)                       ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Value (i)                         ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~
{: #fig-ext-settings title="SETTINGS parameter format"}

实现**必须**忽略它不理解的任何SETTINGS标识符的内容。


#### Defined SETTINGS Parameters {#settings-parameters}

The following settings are defined in HTTP/3:

  SETTINGS_MAX_HEADER_LIST_SIZE (0x6):
  : The default value is unlimited.  See {{header-formatting}} for usage.

  SETTINGS_NUM_PLACEHOLDERS (0x8):
  : The default value is 0.  However, this value SHOULD be set to a non-zero
    value by servers.  See {{placeholders}} for usage.

Setting identifiers of the format `0x1f * N + 0x21` for integer values of N are
reserved to exercise the requirement that unknown identifiers be ignored.  Such
settings have no defined meaning. Endpoints SHOULD include at least one such
setting in their SETTINGS frame. Endpoints MUST NOT consider such settings to
have any meaning upon receipt.

Because the setting has no defined meaning, the value of the setting can be any
value the implementation selects.

Additional settings can be defined by extensions to HTTP/3; see {{extensions}}
for more details.

#### Initialization

An HTTP implementation MUST NOT send frames or requests which would be invalid
based on its current understanding of the peer's settings.  All settings begin
at an initial value, and are updated upon receipt of a SETTINGS frame.  For
servers, the initial value of each client setting is the default value.

For clients using a 1-RTT QUIC connection, the initial value of each server
setting is the default value. When a 0-RTT QUIC connection is being used, the
initial value of each server setting is the value used in the previous session.
Clients MUST store the settings the server provided in the session being resumed
and MUST comply with stored settings until the current server settings are
received.

A server can remember the settings that it advertised, or store an
integrity-protected copy of the values in the ticket and recover the information
when accepting 0-RTT data. A server uses the HTTP/3 settings values in
determining whether to accept 0-RTT data.

A server MAY accept 0-RTT and subsequently provide different settings in its
SETTINGS frame. If 0-RTT data is accepted by the server, its SETTINGS frame MUST
NOT reduce any limits or alter any values that might be violated by the client
with its 0-RTT data.


### PUSH_PROMISE {#frame-push-promise}

The PUSH_PROMISE frame (type=0x05) is used to carry a promised request header
set from server to client on a request stream, as in HTTP/2.

~~~~~~~~~~  drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Push ID (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Header Block (*)                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-push-promise title="PUSH_PROMISE frame payload"}

The payload consists of:

Push ID:
: A variable-length integer that identifies the server push operation.  A Push
  ID is used in push stream headers ({{server-push}}), CANCEL_PUSH frames
  ({{frame-cancel-push}}), DUPLICATE_PUSH frames ({{frame-duplicate-push}}), and
  PRIORITY frames ({{frame-priority}}).

Header Block:
: QPACK-compressed request header fields for the promised response.  See [QPACK]
  for more details.

A server MUST NOT use a Push ID that is larger than the client has provided in a
MAX_PUSH_ID frame ({{frame-max-push-id}}) and MUST NOT use the same Push ID in
multiple PUSH_PROMISE frames.  A client MUST treat receipt of a PUSH_PROMISE
that contains a larger Push ID than the client has advertised or a Push ID which
has already been promised as a connection error of type HTTP_MALFORMED_FRAME.

If a PUSH_PROMISE frame is received on either control stream, the recipient MUST
respond with a connection error ({{errors}}) of type HTTP_WRONG_STREAM.

See {{server-push}} for a description of the overall server push mechanism.

### GOAWAY {#frame-goaway}

The GOAWAY frame (type=0x7) is used to initiate graceful shutdown of a
connection by a server.  GOAWAY allows a server to stop accepting new requests
while still finishing processing of previously received requests.  This enables
administrative actions, like server maintenance.  GOAWAY by itself does not
close a connection.

~~~~~~~~~~  drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Stream ID (i)                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-goaway title="GOAWAY frame payload"}

The GOAWAY frame is always sent on the control stream. It carries a QUIC Stream
ID for a client-initiated bidirectional stream encoded as a variable-length
integer.  A client MUST treat receipt of a GOAWAY frame containing a Stream ID
of any other type as a connection error of type HTTP_WRONG_STREAM.

Clients do not need to send GOAWAY to initiate a graceful shutdown; they simply
stop making new requests.  A server MUST treat receipt of a GOAWAY frame on any
stream as a connection error ({{errors}}) of type HTTP_UNEXPECTED_FRAME.

The GOAWAY frame applies to the connection, not a specific stream.  A client
MUST treat a GOAWAY frame on a stream other than the control stream as a
connection error ({{errors}}) of type HTTP_UNEXPECTED_FRAME.

See {{connection-shutdown}} for more information on the use of the GOAWAY frame.

### MAX_PUSH_ID {#frame-max-push-id}

The MAX_PUSH_ID frame (type=0xD) is used by clients to control the number of
server pushes that the server can initiate.  This sets the maximum value for a
Push ID that the server can use in a PUSH_PROMISE frame.  Consequently, this
also limits the number of push streams that the server can initiate in addition
to the limit set by the QUIC MAX_STREAM_ID frame.

The MAX_PUSH_ID frame is always sent on the control stream.  Receipt of a
MAX_PUSH_ID frame on any other stream MUST be treated as a connection error of
type HTTP_WRONG_STREAM.

A server MUST NOT send a MAX_PUSH_ID frame.  A client MUST treat the receipt of
a MAX_PUSH_ID frame as a connection error of type HTTP_UNEXPECTED_FRAME.

The maximum Push ID is unset when a connection is created, meaning that a server
cannot push until it receives a MAX_PUSH_ID frame.  A client that wishes to
manage the number of promised server pushes can increase the maximum Push ID by
sending MAX_PUSH_ID frames as the server fulfills or cancels server pushes.

~~~~~~~~~~  drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Push ID (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-max-push title="MAX_PUSH_ID frame payload"}

The MAX_PUSH_ID frame carries a single variable-length integer that identifies
the maximum value for a Push ID that the server can use (see
{{frame-push-promise}}).  A MAX_PUSH_ID frame cannot reduce the maximum Push ID;
receipt of a MAX_PUSH_ID that contains a smaller value than previously received
MUST be treated as a connection error of type HTTP_MALFORMED_FRAME.

### DUPLICATE_PUSH {#frame-duplicate-push}

The DUPLICATE_PUSH frame (type=0xE) is used by servers to indicate that an
existing pushed resource is related to multiple client requests.

The DUPLICATE_PUSH frame is always sent on a request stream.  Receipt of a
DUPLICATE_PUSH frame on any other stream MUST be treated as a connection error
of type HTTP_WRONG_STREAM.

A client MUST NOT send a DUPLICATE_PUSH frame.  A server MUST treat the receipt
of a DUPLICATE_PUSH frame as a connection error of type HTTP_MALFORMED_FRAME.

~~~~~~~~~~  drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Push ID (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-duplicate-push title="DUPLICATE_PUSH frame payload"}

The DUPLICATE_PUSH frame carries a single variable-length integer that
identifies the Push ID of a resource that the server has previously promised
(see {{frame-push-promise}}).

This frame allows the server to use the same server push in response to multiple
concurrent requests.  Referencing the same server push ensures that a promise
can be made in relation to every response in which server push might be needed
without duplicating request headers or pushed responses.

Allowing duplicate references to the same Push ID is primarily to reduce
duplication caused by concurrent requests.  A server SHOULD avoid reusing a Push
ID over a long period.  Clients are likely to consume server push responses and
not retain them for reuse over time.  Clients that see a DUPLICATE_PUSH that
uses a Push ID that they have since consumed and discarded are forced to ignore
the DUPLICATE_PUSH.


### 保留帧类型(Reserved Frame Types) {#frame-grease}

帧类型是以`0x1f * N + 0x21`格式的，其中整数值 N 被保留用于满足未知种类被忽略的需求 ({{extensions}})。
这些帧没有语义，而且可以当需要应用层填充的时候发送。
他们也**可能**在当前没有数据传输的连接上发送。
终端在收到时**禁止**认为这些帧有任何有意义。

载荷和帧的长度是通过实现的某种规则选择的。


# HTTP请求生命周期(HTTP Request Lifecycle)

## HTTP消息交换(HTTP Message Exchanges) {#request-response}

客户端通过一个客户端建立的双向 QUIC 流发送一个 HTTP 请求。
客户端在给定的流上**必须**仅发送一个请求。
服务端在和请求发起的同一个流上发送一个或多个 HTTP 回复，详见下文描述。

一条 HTTP 消息(请求或回复)包含:

1. 消息头(详见 {{!RFC7230}}, 章节 3.2)，作为单个 HEADERS 帧发送(详见{{frame-headers}})。

2. 载荷消息体(详见{{!RFC7230}}, 章节 3.3)，作为一系列 DATA 帧发送(详见{{frame-data}})。

3. (可选的)一个包含包尾部分的 HEADERS 帧，如果存在(详见{{!RFC7230}} , 章节 4.1.2)。

服务端**可能**在回复消息的帧中插入一个或更多 PUSH_PROMISE 帧(详见 {{frame-push-promise}})。
这些 PUSH_PROMISE 帧不是回复的一部分；查看{{server-push}}获得更多细节。

在{{!RFC7230}}章节4.1中定义的"大块"的传输编码**禁止**使用。

紧接着载荷消息体之后的预告头字段通过一个附加的 HEADERS 帧携带。
发送者在一个预告部分中**必须**仅发送一个 HEADERS 帧。
接收者**必须**丢弃任何随后的 HEADERS 帧。

回复当且仅当一个或更多的信息性回复(1xx, 详见{{!RFC7231}}, 章节 6.2)先于同一个请求的最终回复时**可能**包含多条消息。
非最终回复不包含载荷消息体或者预告。

一个 HTTP 请求/回复交换完全的利用了一个双向 QUIC 流。
在发送一个请求之后，客户端**必须**关闭发送用的流。
除非使用 CONNECT 方法(详见{{the-connect-method}})，客户端**禁止**使得流的关闭与否依赖于接收他们请求的回复。
在发送一个最终回复之后，服务端**必须**关闭发送用的流。
在这种角度上，QUIC 流被完全关闭了。

当一个流被关闭了，这预示着一个 HTTP 消息的终结。
因为一些消息过大或者无限，终端当已经收到可供处理的足够的消息时**应该**开始处理仅收到部分的 HTTP 消息。
如果一个客户端流没有收到足够的 HTTP 消息来提供一个完整的回复就关闭了，
服务端**应该**以错误码HTTP_INCOMPLETE_REQUEST放弃它的回复。

如果回复不依赖于任何还没有被发送或者还没有接收到的部分，服务端可以发送一个完整的回复优先级给发送一个完整请求的客户端。
当这出现了，服务端**可能**通过触发一个带有 HTTP_EARLY_RESPONSE
错误码的 QUIC STOP_SENDING 帧请求客户端不带错误的放弃传输请求，
发送一个完整的回复，并且干净的关闭流。
客户端**禁止**因为他们的请求被突然终结而丢弃完整的回复，尽管客户端总是可以以任何其他理由自由的丢弃回复。


### 头格式化与压缩(Header Formatting and Compression) {#header-formatting}

HTTP 消息头以一系列键值对的形式来携带信息，称呼为头字段。
关于列举注册过的 HTTP 头字段，详见"消息头字段(Message Header Field)"
维护于<https://www.iana.org/assignments/message-headers>的仓库。

和之前版本的 HTTP 一样, 头字段名是 ASCII 字符的字符串，以不区分大小写的方式进行比较。
HTTP 头字段名字和值的属性在{{!RFC7230}}章节 3.2 中有详细的讨论，尽管网络渲染在 HTTP/3 中有所不同。
同 HTTP/2 中，头字段名**必须**被转化为小写来进行编码。
请求或者回复包含大写的头字段名**必须**被认为是畸形的处理。

如 HTTP/2 中，HTTP/3 使用了以":"字符(ASCII 0x3a)开头的特殊的假头部字段来表达 URI，请求的方法，以及返回的状态码。
这些假头部字段定义于{{!RFC7540}}章节8.1.2.3和8.1.2.4中。
假头部字段不是 HTTP 头字段。
终端**禁止**生成除了定义在{{!RFC7540}}中的那些之外的假头部字段。
在{{!RFC7540}}章节8.1.2.1中描述的使用假头部字段的使用上的限制也适用于 HTTP/3。

HTTP/3 使用 QPACK 如在[QPACK]中描述头压缩，这是一种允许灵活避免头压缩导致的队头阻塞的HPACK 的变体。
详见文档获取附加细节。

HTTP/3 的实现**可能**强制要求一个它在单个 HTTP 消息上会接受的头部的最大大小的限制值;
收到一个更大的消息头部**应该**以`HTTP_EXCESSIVE_LOAD`类型流错误进行处理。
如果实现是希望建议对端这个限制值，它可以在`SETTINGS_MAX_HEADER_LIST_SIZE`参数中以字节数的形式传递给对端。
头部列表的大小基于未压缩的头字段的大小进行计算，包含名字和值以字节为单位的长度，再加上每个头字段 32 字节的开销。


### 请求取消与拒绝(Request Cancellation and Rejection) {#request-cancellation}

客户端可以通过以HTTP_REQUEST_CANCELLED ({{http-error-codes}})
错误码来终止流取消请求(QUIC RESET_STREAM 以及/或者 STOP_SENDING 帧，如果合适的话)。
当客户端取消一个回复，它表达了这个回复已经不再感兴趣。
实现**应该**通过终止一个流的两个方向来取消请求。

当服务器在没有执行任何应用程序处理的情况下拒绝请求时，它**应该**以错误码 HTTP_REQUEST_REJECTED 中止响应流。
在这种情况下，“已处理”意味着流中的一些数据被传递到软件的更高层，这些软件可能因此采取了一些操作。
客户端可以将被服务器拒绝的请求当作根本没有发送过的请求来处理，从而允许以后在新连接上重试这些请求。
服务器**禁止**对已部分或完全处理的请求使用HTTP_REQUEST_REJECTED错误代码。
当服务器在部分处理后放弃响应时，它**应该**以错误码 HTTP_REQUEST_CANCELLED 中止响应流。

当客户端发送带有 HTTP_REQUEST_CANCELLED 的 STOP_SENDING 帧时，
如果未执行任何处理，服务器**可能**会在相应的 RESET_STREAM 中发送 HTTP_REQUEST_REBJECT 错误码。
除非响应包含相同错误码的 QUIC STOP_SENDING 帧，否则客户端**禁止**使用 HTTP_REQUEST_REJECTED 错误码来重置流。

如果流在接收到完整响应后被取消，客户端**可能**忽略该取消并使用得到的响应。
但是，如果流在接收到部分响应后被取消，则**不应该**使用所得到的响应。
除非另有允许，否则不可能自动重试此类请求(例如，诸如 GET、PUT 或 DELETE 等幂等操作)。


## 连接方法(The CONNECT Method)

伪方法连接(pseudo-method CONNECT)(({{!RFC7231}}章节4.3.6)主要与 HTTP
代理一起使用，用于与源服务器建立 TLS 会话，以便与“https”资源进行交互。
在 HTTP/1.x 中，CONNECT 用于将整个 HTTP 连接转换为到远程主机的隧道。
在 HTTP/2 中，CONNECT 方法用于通过单个 HTTP/2 流建立到远程主机的隧道，以实现类似的目的。

HTTP/3 中的连接(CONNECT)请求的行为与 HTTP/2 中相同。
请求的格式**必须**如{{!RFC7540}}章节8.3所述。
不符合这些限制的连接请求格式是不正确的。
请求流**禁止**在请求结束时关闭。

支持 CONNECT 的代理与在权威伪头部字段中认证过的服务器建立 TCP 连接({{!RFC0793}})。
成功建立此连接后，代理将包含 2xx 系列状态代码的 HEADERS 帧发送到客户端，如{{!RFC7231}}章节4.3.6定义。

流中的所有数据(DATA)帧都与 TCP 连接上发送或接收的数据相对应。
客户端发送的任何数据(DATA)帧都由代理发送到 TCP 服务器；
从TCP服务器接收的数据由代理打包成数据(DATA)帧。
注意，TCP 包的大小和数量不保证可以通过可预测的方式映射到 HTTP 数据(DATA)或 QUIC STEAM 帧的大小和数量。

TCP 连接可以由任一对端关闭。
当客户端结束请求流(即代理处的接收流进入“DataRecvd”状态)时，代理将在其与TCP服务器的连接上设置 FIN 位。
当代理收到设置了 FIN 位的数据包时，它将终止发送到客户端的发送流。
保持单向半关闭状态的TCP连接不是不可用的，但服务器通常处理得很差，因此客户端**不应该**在仍希望从连接(CONNECT)的目标接收数据时关闭用于发送的流。

TCP 连接错误通过 QUIC RESET_STREAM 帧发出信号。
代理将 TCP 连接中的任何错误(包括接收设置了RST位的TCP数据段)视为类型为
HTTP_CONNECT_ERROR({{http-error-codes}})的流错误。
相应地，代理如果检测到流或 QUIC 连接的错误**必须**发送一个带 RST 位的 TCP 包。


## 优先级(Prioritization) {#priority}

HTTP/3 使用的优先级规则与{{!RFC7540}}章节5.3 中描述的方案类似。
在这个优先级方案中，一个给定的元素可以被指定为依赖于另一个元素。
此信息在标识元素和依赖项的 PRIORITY 帧{{frame-priority}}中表示。
可以确定优先顺序的元素有：

- 请求，通过请求流的 ID 标识
- 推送，通过承诺的源({{frame-push-promise}})的推送 ID 标识
- 占位符，通过占位符 ID 标识

总之，连接中所有优先级元素之间的依赖关系形成依赖关系树。
一个元素可以依赖于另一个元素，也可以依赖于树的根。
对不再存在于树中的元素的引用被视为对树的根的引用。
当 PRIORITY 帧修改带优先级的元素之间的依赖关系时，依赖关系树的结构也会发生变化。

由于流之间的重新排序，尚未在树中的元素也可以设置优先级。
这些元素以请求的优先级添加到树中。

首次创建优先级元素时，它的默认初始权重为16，并且具有默认依赖项。
请求和占位符依赖于优先级树的根；推送依赖于发送 PUSH_PROMISE 帧的客户端请求。

请求可以通过在流的开头包含 PRIORTIY 帧(详见{{frame-priority}})来覆盖默认初始值。
可以通过在控制流上发送 PRIORITY 帧来更新这些优先级。


### 占位符(Placeholders) {#placeholders}

在 HTTP/2 中，某些实现在描述请求的相对优先级时使用已关闭或未使用的流作为占位符。
这造成了混乱，因为服务器无法可靠地确定可以安全丢弃优先级树的哪些元素。
客户端可能会在服务器丢弃状态很长时间后引用关闭的流，从而导致服务端得到了与客户端试图表示的优先级的不同的优先级树。

在 HTTP/3 中，服务端使用`SETTINGS_NUM_PLACEHOLDERS`设置显式允许许多占位符。
因为服务器承诺维护优先级树中的这些占位符，所以客户端可以放心地使用它们，因为服务器不会丢弃该状态。
客户端**禁止**发送`SETTINGS_NUM_PLACEHOLDERS`设置；
服务器接收到此设置**必须**被以类型为`HTTP_WRONG_SETTING_DIRECTION`的连接错误来处理。

占位符由零到服务端允许的占位符数量小一的 ID 标识。

和流一样，占位符也有与其关联的优先级。


### Priority Tree Maintenance

Because placeholders will be used to "root" any persistent structure of the tree
which the client cares about retaining, servers can aggressively prune inactive
regions from the priority tree. For prioritization purposes, a node in the tree
is considered "inactive" when the corresponding stream has been closed for at
least two round-trip times (using any reasonable estimate available on the
server).  This delay helps mitigate race conditions where the server has pruned
a node the client believed was still active and used as a Stream Dependency.

Specifically, the server MAY at any time:

- Identify and discard branches of the tree containing only inactive nodes
  (i.e. a node with only other inactive nodes as descendants, along with those
  descendants)
- Identify and condense interior regions of the tree containing only inactive
  nodes, allocating weight appropriately

~~~~~~~~~~  drawing
    x                x                 x
    |                |                 |
    P                P                 P
   / \               |                 |
  I   I     ==>      I      ==>        A
     / \             |                 |
    A   I            A                 A
    |                |
    A                A
~~~~~~~~~~
{: #fig-pruning title="Example of Priority Tree Pruning"}

In the example in {{fig-pruning}}, `P` represents a Placeholder, `A` represents
an active node, and `I` represents an inactive node.  In the first step, the
server discards two inactive branches (each a single node).  In the second step,
the server condenses an interior inactive node.  Note that these transformations
will result in no change in the resources allocated to a particular active
stream.

Clients SHOULD assume the server is actively performing such pruning and SHOULD
NOT declare a dependency on a stream it knows to have been closed.

## Server Push

HTTP/3 server push is similar to what is described in HTTP/2 {{!RFC7540}}, but
uses different mechanisms.

Each server push is identified by a unique Push ID. This Push ID is used in a
single PUSH_PROMISE frame (see {{frame-push-promise}}) which carries the request
headers, possibly included in one or more DUPLICATE_PUSH frames (see
{{frame-duplicate-push}}), then included with the push stream which ultimately
fulfills those promises.

Server push is only enabled on a connection when a client sends a MAX_PUSH_ID
frame (see {{frame-max-push-id}}). A server cannot use server push until it
receives a MAX_PUSH_ID frame. A client sends additional MAX_PUSH_ID frames to
control the number of pushes that a server can promise. A server SHOULD use Push
IDs sequentially, starting at 0. A client MUST treat receipt of a push stream
with a Push ID that is greater than the maximum Push ID as a connection error of
type HTTP_LIMIT_EXCEEDED.

The header of the request message is carried by a PUSH_PROMISE frame (see
{{frame-push-promise}}) on the request stream which generated the push. This
allows the server push to be associated with a client request. Ordering of a
PUSH_PROMISE in relation to certain parts of the response is important (see
Section 8.2.1 of {{!RFC7540}}).  Promised requests MUST conform to the
requirements in Section 8.2 of {{!RFC7540}}.

The same server push can be associated with additional client requests using a
DUPLICATE_PUSH frame (see {{frame-duplicate-push}}).  Ordering of a
DUPLICATE_PUSH in relation to certain parts of the response is similarly
important.  Due to reordering, DUPLICATE_PUSH frames can arrive before the
corresponding PUSH_PROMISE frame, in which case the request headers of the push
would not be immediately available.  Clients which receive a DUPLICATE_PUSH
frame for an as-yet-unknown Push ID can either delay generating new requests for
content referenced following the DUPLICATE_PUSH frame until the request headers
become available, or can initiate requests for discovered resources and cancel
the requests if the requested resource is already being pushed.

When a server later fulfills a promise, the server push response is conveyed on
a push stream (see {{push-streams}}). The push stream identifies the Push ID of
the promise that it fulfills, then contains a response to the promised request
using the same format described for responses in {{request-response}}.

If a promised server push is not needed by the client, the client SHOULD send a
CANCEL_PUSH frame. If the push stream is already open or opens after sending the
CANCEL_PUSH frame, a QUIC STOP_SENDING frame with an appropriate error code can
also be used (e.g., HTTP_PUSH_REFUSED, HTTP_PUSH_ALREADY_IN_CACHE; see
{{errors}}). This asks the server not to transfer additional data and indicates
that it will be discarded upon receipt.

# Connection Closure

Once established, an HTTP/3 connection can be used for many requests and
responses over time until the connection is closed.  Connection closure can
happen in any of several different ways.

## Idle Connections

Each QUIC endpoint declares an idle timeout during the handshake.  If the
connection remains idle (no packets received) for longer than this duration, the
peer will assume that the connection has been closed.  HTTP/3 implementations
will need to open a new connection for new requests if the existing connection
has been idle for longer than the server's advertised idle timeout, and SHOULD
do so if approaching the idle timeout.

HTTP clients are expected to request that the transport keep connections open
while there are responses outstanding for requests or server pushes, as
described in Section 19.2 of {{QUIC-TRANSPORT}}. If the client is not expecting
a response from the server, allowing an idle connection to time out is preferred
over expending effort maintaining a connection that might not be needed.  A
gateway MAY maintain connections in anticipation of need rather than incur the
latency cost of connection establishment to servers. Servers SHOULD NOT actively
keep connections open.

## 连接关闭（Connection Shutdown）

即使连接不是空闲的，任何一个终端都可以决定停止使用连接，并让连接正常关闭。由于客户端
主导请求生成，因此客户端通过不在连接上发送其他请求来执行连接关闭；与先前请求相关联的
响应和推送响应将继续完成。服务器通过与客户端通信来执行相同的功能。

服务器通过发送GOAWAY帧({{frame-goaway}})启动连接关闭。GOAWAY帧指示在较低流ID上的
客户端发起的请求在此连接中被处理或可能被处理，而在指示的流ID和更大流ID上的请求被拒绝。
这使客户端和服务器能够在连接关闭之前就哪些请求被接受达成一致。此标识符**可能**低于
QUIC MAX_STREAM_ID 帧标识的流限制，如果没有请求已被处理，则该标识符**可能**为零。
发送GOAWAY帧后，服务器**不应**增加QUIC MAX_STREAM_ID限制。

GOAWAY发送后，服务器**必须**拒绝使用标识符大于或等于最后一个流ID的流发送的请求。
客户端在收到GOAWAY后**禁止**在连接上发送新的请求，尽管请求可能已经在传输中。可以
为新请求建立新连接。

如果客户端在流ID大于或等于GOAWAY帧中指示的流ID的流上发送请求，则这些请求被视为
被拒绝({{request-cancellation}})。客户端**应该**取消对此ID以上的流的任何请求。
如果未处理这些请求，服务器还**可能**拒绝ID低于指示ID的流上的请求。

流ID小于GOAWAY帧中流ID的请求可能已被处理；只有在成功完成、单独重置或连接终止之前，
才能知道它们的状态。

当预先知道连接关闭时，服务器应该发送GOAWAY帧，即使提前通知很小，这样远程对端
就可以知道请求是否已被部分处理。例如，如果HTTP客户端在服务器关闭QUIC连接的同时
发送POST请求，则如果服务器未发送GOAWAY帧以指示它可能对哪些流执行了操作，那么
客户端无法知道服务器是否开始处理该POST请求。

当服务器关闭连接时，无法重试请求的客户端将丢失正在运行的所有请求。服务器**可能**会
发送多个GOAWAY帧，表示不同的流ID，但**禁止**增加它们在最后一个流ID中发送的值，
因为客户端可能已经在另一个连接上重试了未处理的请求。尝试正常关闭连接的服务器**应该**
发送初始GOAWAY帧，且最后一个流ID设置为QUIC的MAX_STREAM_ID的当前值，此后**不应**
增加MAX_STREAM_ID。这向客户端发出信号，表明即将关闭，并且禁止进一步的请求。
在为任何传输中请求留出时间(至少一个往返时间)后，服务器**可能**会发送另一个更新后
且是最后的流ID的GOAWAY帧。这可以确保连接可以在不丢失请求的情况下干净地关闭。

一旦所有已接受的请求都得到处理，服务器就可以允许连接变得空闲，或者**可以**启动连接
的立即关闭。完成正常关闭的端点**应该**在关闭连接时使用HTTP_NO_ERROR代码。

## 立即关闭应用程序

一个HTTP/3的实现可以随时立即关闭QUIC连接，这会发送QUIC CONNECTION_CLOSE帧到对端；
此帧中的错误代码告诉对端为什么要关闭连接。有关关闭连接时可以使用的错误代码，
请参见{{errors}}。

在关闭连接之前，可能会发送GOAWAY帧以允许客户端重试某些请求。将GOAWAY帧包含在QUIC
CONNECTION_CLOSE帧所在的数据包中可以提高客户端接收该帧的机会。

## 传输关闭

由于各种原因，QUIC传输可能会向应用层表明连接已终止。这可能是由于对端显式关闭、传输层
错误或网络拓扑更改中断连接所致。

如果连接在没有GOAWAY帧的情况下终止，客户端**必须**假设发送的任何请求(无论是全部还是部分)
都可能已被处理。

# HTTP/3的扩展(Extensions to HTTP/3) {#extensions}

HTTP/3允许对协议进行扩展。在本节描述的限制范围内，协议扩展可用于提供附加服务或更改
协议的任何方面。扩展仅在单个HTTP/3连接的范围内有效。

这适用于本文档中定义的协议元素。这不影响现有的HTTP扩展选项，例如定义新方法、状态码
或标头字段。

扩展允许使用新的帧类型({{frames}})、新的设置({{settings-parameters}})、新的
错误码({{errors}})或新的单向流类型({{unidirectional-streams}})。建立了用于
管理这些扩展点的注册表：帧类型({{iana-frames}})、设置({{iana-settings}})、
错误码({{iana-error-codes}})和流类型({{iana-stream-types}})。

实现**必须**忽略所有可扩展协议元素中的未知或不受支持的值。实现**必须**丢弃具有
未知或不支持类型的帧和单向流。这意味着在没有事先安排或协商的情况下，扩展可以安全
地使用这些扩展点中的任何一个。

在使用之前，必须协商可以更改现有协议组件语义的扩展。例如，在对端发出可接受的正面
信号之前，不能使用更改HEADERS帧布局的扩展。在这种情况下，也可能有必要在修改后的布局
生效时进行协调。

本文档并不要求使用特定的方法来协商扩展的使用，但提到可以使用设置({{settings-parameters}})
来实现此目的。如果两个对端都设置了表示愿意使用扩展的值，则可以使用扩展。如果将设置用于扩展协商
则**必须**以这样的方式定义默认值：如果省略该设置，则禁用扩展。


# 错误处理（Error Handling） {#errors}

QUIC允许应用程序在遇到错误时突然终止(重置)单个流或整个连接。这些错误称为“流错误”或
“连接错误”，在{{QUIC-TRANSPORT}}中有详细说明。端点**可以**选择将流错误视为连接错误。

本节介绍特定于HTTP/3的错误代码，这些代码可用于表示连接或流错误的原因。

## HTTP/3 错误码 {#http-error-codes}

以下错误码被定义用于HTTP/3协议下的
QUIC RESET_STREAM帧、STOP_SESSING帧和
CONNECTION_CLOSE帧。

HTTP_NO_ERROR (0x00):
: 没有错误。当需要关闭连接或者流，
 但没有错误要发送的时候使用。

HTTP_WRONG_SETTING_DIRECTION (0x01):
: 一个客户端限定的设置项由服务端发送，
 或者服务端限定的设置项由客户端发送时使用。

HTTP_PUSH_REFUSED (0x02):
: 服务端尝试在该连接中推送客户端不能接受的内容。

HTTP_INTERNAL_ERROR (0x03):
: 在HTTP栈内部发生错误。

HTTP_PUSH_ALREADY_IN_CACHE (0x04):
: 服务端尝试推送的内容已经在客户端缓存。

HTTP_REQUEST_CANCELLED (0x05):
: 请求或者请求的响应被取消

HTTP_INCOMPLETE_REQUEST (0x06):
: 客户端流在没有完成完整格式的请求的情况下终止。

HTTP_CONNECT_ERROR (0x07):
: 响应CONNECT请求而建立的连接被重置或异常关闭。

HTTP_EXCESSIVE_LOAD (0x08):
: 终端检测到其对等端正在表现出可能会产生
 过多负载的行为。

HTTP_VERSION_FALLBACK (0x09):
: 请求的操作无法通过HTTP/3提供。对端应通过HTTP/1.1重试。

HTTP_WRONG_STREAM (0x0A):
: 从流中接收到一个帧，而该流不允许发送这个帧。

HTTP_LIMIT_EXCEEDED (0x0B):
: 引用的流ID、推送ID或占位符ID大于该标识符的当前最大值。

HTTP_DUPLICATE_PUSH (0x0C):
: 在两个不同的流头部中引用了相同的推送ID。

HTTP_UNKNOWN_STREAM_TYPE (0x0D):
: 单向流标头包含未知的流类型。

HTTP_WRONG_STREAM_COUNT (0x0E):
: 单向流类型的使用次数超过了该类型所允许的次数。

HTTP_CLOSED_CRITICAL_STREAM (0x0F):
: 连接所需的流已关闭或重置。

HTTP_WRONG_STREAM_DIRECTION (0x0010):
: A unidirectional stream type was used by a peer which is not permitted to do
  so.
: 对端使用了被禁止的单向流类型。

HTTP_EARLY_RESPONSE (0x0011):
: 客户端请求的其余部分不需要生成响应。仅用于STOP_SENDING。

HTTP_MISSING_SETTINGS (0x0012):
: 在控制流的开始处未接收到任何设置帧。

HTTP_UNEXPECTED_FRAME (0x0013):
: 接收到当前状态下不允许接收的帧。

HTTP_REQUEST_REJECTED (0x0014):
: 服务端在未执行任何应用程序处理的情况下拒绝了请求。

HTTP_GENERAL_PROTOCOL_ERROR (0x00FF):
: 对端违反了协议要求
并且未明确定义错误码，
或者终端拒绝使用更明确的错误码。

HTTP_MALFORMED_FRAME (0x01XX):
: 特定帧类型中的错误。如果帧类型
为‘0xfe`或更小，则该类型
将包含在错误代码的最后一个字节。
例如，MAX_Push_ID帧中的
错误将用代码(0x10D)指示。
最后一个字节`0xff`用于表示
任何大于`0xfe`的帧类型。


# 安全注意事项(Security Considerations)

HTTP/3的安全注意事项应与具有TLS的
HTTP/2的安全注意事项类似。
请注意，如果HTTP/2在其他帧
中使用PADDING帧和填充字段来使连接
更不受流量分析的影响，则HTTP/3可以
依赖QUIC PADDING帧或使用在{{frame-grease}}和
{{stream-grease}}中讨论的保留帧和流类型。

当HTTP替代服务被用于发现HTTP/3终端时，
{{!ALTSVC}}的安全注意事项也适用。

几个协议要素包含嵌套的长度要素，
通常以帧中带有包含可变长度整数
显式长度形式出现。这可能会给
一个鲁莽的实现者带来安全风险。
实现**必须**确保帧的长度与
其包含的字段的长度完全匹配。

某些HTTP实现使用客户端地址
进行日志记录或访问控制。
由于QUIC客户端的地址可能
在连接过程中发生更改
(未来版本可能支持同时使用多个地址)，
因此此类实现需要主动检索客户端的
一个或多个与当前地址相关的地址，
或者明确接受原始地址可能会更改。


# IANA注意事项(IANA Considerations)

## 注册HTTP/3标识字符串(Registration of HTTP/3 Identification String)

本文档在{{?RFC7301}}中
建立的“应用层协议协商(ALPN)协议ID”注册表
中新注册了HTTP/3标识

“h3”字符串标识HTTP/3:

  Protocol:
  : HTTP/3

  Identification Sequence:
  : 0x68 0x33 ("h3")

  Specification:
  : This document

## Registration of QUIC Version Hint Alt-Svc Parameter

This document creates a new registration for version-negotiation hints in the
"Hypertext Transfer Protocol (HTTP) Alt-Svc Parameter" registry established in
{{!RFC7838}}.

  Parameter:
  : "quic"

  Specification:
  : This document, {{alt-svc-version-hint}}

## Frame Types {#iana-frames}

This document establishes a registry for HTTP/3 frame type codes. The "HTTP/3
Frame Type" registry governs a 62-bit space. This space is split into three
spaces that are governed by different policies. Values between `0x00` and `0x3f`
(in hexadecimal) are assigned via the Standards Action or IESG Review policies
{{!RFC8126}}. Values from `0x40` to `0x3fff` operate on the Specification
Required policy {{!RFC8126}}. All other values are assigned to Private Use
{{!RFC8126}}.

While this registry is separate from the "HTTP/2 Frame Type" registry defined in
{{RFC7540}}, it is preferable that the assignments parallel each other where the
code spaces overlap.  If an entry is present in only one registry, every effort
SHOULD be made to avoid assigning the corresponding value to an unrelated
operation.

New entries in this registry require the following information:

Frame Type:
: A name or label for the frame type.

Code:
: The 62-bit code assigned to the frame type.

Specification:
: A reference to a specification that includes a description of the frame layout
  and its semantics, including any parts of the frame that are conditionally
  present.

The entries in the following table are registered by this document.

| ---------------- | ------ | -------------------------- |
| Frame Type       |  Code  | Specification              |
| ---------------- | :----: | -------------------------- |
| DATA             |  0x0   | {{frame-data}}             |
| HEADERS          |  0x1   | {{frame-headers}}          |
| PRIORITY         |  0x2   | {{frame-priority}}         |
| CANCEL_PUSH      |  0x3   | {{frame-cancel-push}}      |
| SETTINGS         |  0x4   | {{frame-settings}}         |
| PUSH_PROMISE     |  0x5   | {{frame-push-promise}}     |
| Reserved         |  0x6   | N/A                        |
| GOAWAY           |  0x7   | {{frame-goaway}}           |
| Reserved         |  0x8   | N/A                        |
| Reserved         |  0x9   | N/A                        |
| MAX_PUSH_ID      |  0xD   | {{frame-max-push-id}}      |
| DUPLICATE_PUSH   |  0xE   | {{frame-duplicate-push}}   |
| ---------------- | ------ | -------------------------- |

Additionally, each code of the format `0x1f * N + 0x21` for integer values of N
(that is, `0x21`, `0x40`, ..., through `0x‭3FFFFFFFFFFFFFFE‬`) MUST NOT be
assigned by IANA.

## Settings Parameters {#iana-settings}

This document establishes a registry for HTTP/3 settings.  The "HTTP/3 Settings"
registry governs a 62-bit space. This space is split into three spaces that are
governed by different policies. Values between `0x00` and `0x3f` (in
hexadecimal) are assigned via the Standards Action or IESG Review policies
{{!RFC8126}}. Values from `0x40` to `0x3fff` operate on the Specification
Required policy {{!RFC8126}}. All other values are assigned to Private Use
{{!RFC8126}}.  The designated experts are the same as those for the "HTTP/2
Settings" registry defined in {{RFC7540}}.

While this registry is separate from the "HTTP/2 Settings" registry defined in
{{RFC7540}}, it is preferable that the assignments parallel each other.  If an
entry is present in only one registry, every effort SHOULD be made to avoid
assigning the corresponding value to an unrelated operation.

New registrations are advised to provide the following information:

Name:
: A symbolic name for the setting.  Specifying a setting name is optional.

Code:
: The 62-bit code assigned to the setting.

Specification:
: An optional reference to a specification that describes the use of the
  setting.

The entries in the following table are registered by this document.

| ---------------------------- | ------ | ------------------------- |
| Setting Name                 |  Code  | Specification             |
| ---------------------------- | :----: | ------------------------- |
| Reserved                     |  0x2   | N/A                       |
| Reserved                     |  0x3   | N/A                       |
| Reserved                     |  0x4   | N/A                       |
| Reserved                     |  0x5   | N/A                       |
| MAX_HEADER_LIST_SIZE         |  0x6   | {{settings-parameters}}   |
| NUM_PLACEHOLDERS             |  0x8   | {{settings-parameters}}   |
| ---------------------------- | ------ | ------------------------- |

Additionally, each code of the format `0x1f * N + 0x21` for integer values of N
(that is, `0x21`, `0x40`, ..., through `0x‭3FFFFFFFFFFFFFFE‬`) MUST NOT be
assigned by IANA.

## Error Codes {#iana-error-codes}

This document establishes a registry for HTTP/3 error codes. The "HTTP/3 Error
Code" registry manages a 16-bit space.  The "HTTP/3 Error Code" registry
operates under the "Expert Review" policy {{?RFC8126}}.

Registrations for error codes are required to include a description
of the error code.  An expert reviewer is advised to examine new
registrations for possible duplication with existing error codes.
Use of existing registrations is to be encouraged, but not mandated.

New registrations are advised to provide the following information:

Name:
: A name for the error code.  Specifying an error code name is optional.

Code:
: The 16-bit error code value.

Description:
: A brief description of the error code semantics, longer if no detailed
  specification is provided.

Specification:
: An optional reference for a specification that defines the error code.

The entries in the following table are registered by this document.

| ----------------------------------- | ---------- | ---------------------------------------- | ---------------------- |
| Name                                | Code       | Description                              | Specification          |
| ----------------------------------- | ---------- | ---------------------------------------- | ---------------------- |
| HTTP_NO_ERROR                       | 0x0000     | No error                                 | {{http-error-codes}}   |
| HTTP_WRONG_SETTING_DIRECTION        | 0x0001     | Setting sent in wrong direction          | {{http-error-codes}}   |
| HTTP_PUSH_REFUSED                   | 0x0002     | Client refused pushed content            | {{http-error-codes}}   |
| HTTP_INTERNAL_ERROR                 | 0x0003     | Internal error                           | {{http-error-codes}}   |
| HTTP_PUSH_ALREADY_IN_CACHE          | 0x0004     | Pushed content already cached            | {{http-error-codes}}   |
| HTTP_REQUEST_CANCELLED              | 0x0005     | Data no longer needed                    | {{http-error-codes}}   |
| HTTP_INCOMPLETE_REQUEST             | 0x0006     | Stream terminated early                  | {{http-error-codes}}   |
| HTTP_CONNECT_ERROR                  | 0x0007     | TCP reset or error on CONNECT request    | {{http-error-codes}}   |
| HTTP_EXCESSIVE_LOAD                 | 0x0008     | Peer generating excessive load           | {{http-error-codes}}   |
| HTTP_VERSION_FALLBACK               | 0x0009     | Retry over HTTP/1.1                      | {{http-error-codes}}   |
| HTTP_WRONG_STREAM                   | 0x000A     | A frame was sent on the wrong stream     | {{http-error-codes}}   |
| HTTP_LIMIT_EXCEEDED                 | 0x000B     | An identifier limit was exceeded         | {{http-error-codes}}   |
| HTTP_DUPLICATE_PUSH                 | 0x000C     | Push ID was fulfilled multiple times     | {{http-error-codes}}   |
| HTTP_UNKNOWN_STREAM_TYPE            | 0x000D     | Unknown unidirectional stream type       | {{http-error-codes}}   |
| HTTP_WRONG_STREAM_COUNT             | 0x000E     | Too many unidirectional streams          | {{http-error-codes}}   |
| HTTP_CLOSED_CRITICAL_STREAM         | 0x000F     | Critical stream was closed               | {{http-error-codes}}   |
| HTTP_WRONG_STREAM_DIRECTION         | 0x0010     | Unidirectional stream in wrong direction | {{http-error-codes}}   |
| HTTP_EARLY_RESPONSE                 | 0x0011     | Remainder of request not needed          | {{http-error-codes}}   |
| HTTP_MISSING_SETTINGS               | 0x0012     | No SETTINGS frame received               | {{http-error-codes}}   |
| HTTP_UNEXPECTED_FRAME               | 0x0013     | Frame not permitted in the current state | {{http-error-codes}}   |
| HTTP_REQUEST_REJECTED               | 0x0014     | Request not processed                    | {{http-error-codes}}   |
| HTTP_MALFORMED_FRAME                | 0x01XX     | Error in frame formatting                | {{http-error-codes}}   |
| ----------------------------------- | ---------- | ---------------------------------------- | ---------------------- |

## 流类型(Stream Types) {#iana-stream-types}

本文档为 HTTP/3 单向流类型建立了注册表。
"HTTP/3 Stream Type"注册表掌握了62位空间。
此空间可分为三个由不同策略管理的空间。
`0x00`和`0x3f`(十六进制)之间的值是通过标准操作或IESG审核策略{{!RFC8126}}分配的。
从`0x40`到`0x3fff`的值对按照规范要求策略{{!RFC8126}}分配。
所有其他值都分配给私有使用{{!RFC8126}}。

此注册表中的新条目需要以下信息:

流类型(Stream Type):
: 一个流类型的名字或者标签。

编码号(Code):
: 赋予给此类型的62位编码值。

规范(Specification):
: 对规范的引用，该规范包括流类型的描述，以及其载荷的布局语义。

发送者(Sender):
: 可能会建立此种类流的连接的端。值可以是"客户端(Client)", "服务端(Server)", "均可(Both)"。

下表中的条目已注册在这篇文档中。

| ---------------- | ------ | -------------------------- | ------ |
| Stream Type      |  Code  | Specification              | Sender |
| ---------------- | :----: | -------------------------- | ------ |
| Control Stream   |  0x00  | {{control-streams}}        | Both   |
| Push Stream      |  0x01  | {{server-push}}            | Server |
| ---------------- | ------ | -------------------------- | ------ |

额外的，每一个形如 `0x1f * N + 0x21` 的整数值N
(意思是`0x21`, `0x40`, ..., 直到 `0x3FFFFFFFFFFFFFFE`)**禁止**被 IANA 赋值。

--- back

# 对于HTTP/2中事务的思考(Considerations for Transitioning from HTTP/2)

HTTP/3 承袭于 HTTP/2，并且有许多相似之处。
本节描述了设计 HTTP/3 所采用的方法，指出了与 HTTP/2 的重要区别，并描述了如何将 HTTP/2 扩展映射到 HTTP/3。

HTTP/3 始于这样一个前提，即与 HTTP/2 的相似的设计更可取，但并不是一个硬性的要求。
HTTP/3 主要在必要时脱离HTTP/2，以适应 QUIC 和 TCP 之间的行为差异(缺乏排序，对流的支持)。
我们打算避免无端的更改，因为它会使得构建同时适用于两种协议具有相同语义的扩展变得困难或不可能。

本节中指出了这些差异。

## 流(Streams) {#h2-streams}

HTTP/3 允许使用比 HTTP/2 更多的流(2^62-1)。
有关流标识符空间耗尽的注意事项仍适用，尽管空间要大得多，防止可能出现的 QUIC 中的其他限制首先达到，例如连接流控制窗口的限制。

## HTTP帧类型(HTTP Frame Types) {#h2-frames}

在 QUIC 上可以省略 HTTP/2 中的许多帧概念，因为传输已经处理了这些概念。
因为帧已经在流上，所以它们可以省略流编号。
由于帧不会阻塞多路复用(QUIC的多载荷路复用发生在此层以下)，因此可以删除对可变最大长度数据包的支持。
由于流终止由 QUIC 处理，因此不需要 END_STREAM 标志。
这允许了我们从通用框架布局中删除“标志”字段。

帧载荷主要来自{{!RFC7540}}。然而，QUIC 包含了 HTTP/2 中也存在的许多特性(例如，流控制)。
在这些情况下，HTTP 映射不会重新实现它们。
因此，HTTP/3 中不需要几种 HTTP/2 帧类型。
如果不再使用 HTTP/2 定义的帧，则会保留帧ID，以便最大限度地提高 HTTP/2 和 HTTP/3 实现之间的可移植性。
然而，即使两个映射之间的等效帧也不完全相同。

许多差异源于 HTTP/2 跨所有流提供帧之间的绝对排序，而 QUIC 仅在每个流上提供这种保证。
因此，如果帧类型假设仍将按发送顺序接收来自不同流的帧，HTTP/3 将破坏这些假设。

例如，HTTP/2 优先级方案中隐含的是按顺序递送优先级改变(即，依赖关系树突变)的概念：
由于对依赖关系树的操作(例如，重生子树)不是可交换的，
因此发送方和接收方必须以相同的顺序应用它们，以确保双方都具有流依赖关系树的一致视图。
HTTP/2 在优先级帧(PRIORITY)和(可选)报头帧(HEADERS)中指定优先级分配。
为了在 HTTP/3 中按顺序传递优先级更改，在控制流上发送优先级帧(PRIORITY)，并删除独占优先级排序。

同样，HPACK 的设计假设是按顺序交付。
编码的报头块序列必须以与它们被编码的顺序相同的顺序到达(并被解码)到对端。
这可确保两个对端的动态状态保持同步。
因此，HTTP/3 使用 HPACK 的修改版本，如[QPACK]中所述。

HTTP/3 中的帧类型定义通常使用 QUIC 可变长度整数编码。
特别地，流ID使用这种编码，这允许比 HTTP/2 中使用的编码有更大的可能值范围。
HTTP/3 中的一些帧使用标识符而不是流ID(例如，优先级帧中的推送ID)。
如果编码包括流ID，则可能需要重新定义扩展帧类型的编码。

由于“标志”字段不存在于通用 HTTP/3 帧中，因此依赖于标志存在的那些帧需要为标志分配空间作为其帧载荷的一部分。

除了此问题之外，帧类型 HTTP/2 扩展通常只需将 HTTP/2 中的流 0
替换为 HTTP/3 中的控制流即可移植到 QUIC。
HTTP/3 扩展不会承诺有序，但不会因有序受损，并且将以相同的方式可移植到 HTTP/2。

下面列出了每种HTTP/2帧类型的映射方式:

DATA (0x0):
: 占位符在 HTTP/3 帧中未定义。详见{{frame-data}}.

HEADERS (0x1):
: 如上描述，优先级种类的报头不支持。**必须**使用单独的 PRIORITY 帧。占位符在 HTTP/3 帧中未定义。详见{{frame-headers}}.

PRIORITY (0x2):
: 如上描述，PRIORITY 帧在控制流上发送，并且可以引用各种标识。详见{{frame-priority}}.

RST_STREAM (0x3):
: RST_STREAM 帧不再存在，因为 QUIC 提供了流生命周期管理。
相同码点用于CANCEL_PUSH 帧({{frame-cancel-push}})。

SETTINGS (0x4):
: SETTINGS 帧仅当连接开始的时候发送，详见{{frame-settings}} 和 {{h2-settings}}.

PUSH_PROMISE (0x5):
: PUSH_PROMISE 不再引用一个流，相反，推送流通过 Push ID 来引用 PUSH_PROMISE 帧。
详见{{frame-push-promise}}.

PING (0x6):
: PING 帧不再存在, 因为 QUIC 提供了等同的功能。

GOAWAY (0x7):
: GOAWAY 仅从服务端到客户端发送并且不包含错误码。详见{{frame-goaway}}.

WINDOW_UPDATE (0x8):
: WINDOW_UPDATE 帧不再存在，因为QUIC 提供了流量控制。

CONTINUATION (0x9):
: CONTINUATION 帧不再存在; 相反, 可以拥有相较 HTTP/2 更大的 HEADERS/PUSH_PROMISE 帧.

HTTP/2 扩展定义的帧类型需要单独注册 HTTP/3(如果仍然适用)。
为简单起见，已保留在 {{!RFC7540}}中定义的帧的ID。
请注意，HTTP/3 中的帧类型空间实质上更大(62位对8位)，
因此许多 HTTP/3 帧类型没有等效的 HTTP/2 代码点。详见 {{iana-frames}}。

## HTTP/2设置参数(HTTP/2 SETTINGS Parameters) {#h2-settings}

与 HTTP/2 的一个重要区别是，设置在连接开始时只发送一次，此后不能更改。
这消除了许多围绕更改同步的边界情况。

HTTP/2 通过设置帧指定的某些传输级别选项将被 HTTP/3 中的 QUIC 传输参数取代。
HTTP/3 中保留的 HTTP 级别选项的值与 HTTP/2 中的相同。

下面列出了每个 HTTP/2 设置参数的映射方式:

SETTINGS_HEADER_TABLE_SIZE:
: 详见 [QPACK].

SETTINGS_ENABLE_PUSH:
: 已移除，MAX_PUSH_ID 提供了对服务器推送的更细粒度的控制。

SETTINGS_MAX_CONCURRENT_STREAMS:
: 作为其流控制逻辑的一部分，QUIC 控制最大的开放流ID。
在 SETTINGS 帧中指定SETTINGS_MAX_CONCURRENT_STREAMS是错误的。

SETTINGS_INITIAL_WINDOW_SIZE:
: QUIC要求在初始传输握手中指定流和连接流控制窗口大小。
在 SETTINGS 帧中指定SETTINGS_INITIAL_WINDOW_SIZE是错误的。

SETTINGS_MAX_FRAME_SIZE:
: 此设置在 HTTP/3 中没有等价物。在 SETTINGS 帧中指定它是错误的。

SETTINGS_MAX_HEADER_LIST_SIZE:
: 详见 {{settings-parameters}}.

在 HTTP/3 中，设置值是可变长度整数(6、14、30或62位长)，而不是 HTTP/2 中的固定长度32位字段。
这通常会产生较短的编码，但可以为使用完整32位空间的设置生成较长的编码。
从 HTTP/2 移植的设置可能会选择重新定义其设置的格式，以避免使用62位编码。

需要分别为 HTTP/2 和 HTTP/3 定义设置。
为简单起见，已保留在{{!RFC7540}}中定义的设置ID。
请注意，HTTP/3 中的设置标识符空间非常大(62位对16位)，
因此许多 HTTP/3 设置没有等效的 HTTP/2 代码点。详见{{iana-settings}}。


## HTTP/2错误码(HTTP/2 Error Codes)

QUIC 具有 HTTP/2 提供的“流”和“连接”错误的相同概念。
但是，HTTP/2 错误代码没有直接可移植性。

{{!RFC7540}}第7节中定义的 HTTP/2 错误代码映射到 HTTP/3 错误代码，如下所示：

NO_ERROR (0x0):
: {{http-error-codes}}中的 HTTP_NO_ERROR.

PROTOCOL_ERROR (0x1):
: 没有单独映射。详见定义在 {{http-error-codes}}中新的 HTTP_MALFORMED_FRAME 错误码。

INTERNAL_ERROR (0x2):
: {{http-error-codes}}中的HTTP_INTERNAL_ERROR。

FLOW_CONTROL_ERROR (0x3):
: 不适用，因为 QUIC 处理了流量控制。会从 QUIC 层触发QUIC_FLOW_CONTROL_RECEIVED_TOO_MUCH_DATA。

SETTINGS_TIMEOUT (0x4):
: 不适用，因为对SETTINGS 的确认未定义。

STREAM_CLOSED (0x5):
: 不适用，因为 QUIC 处理了流管理。会从 QUIC 层触发 QUIC_STREAM_DATA_AFTER_TERMINATION。

FRAME_SIZE_ERROR (0x6):
: HTTP_MALFORMED_FRAME 错误码定义于{{http-error-codes}}.

REFUSED_STREAM (0x7):
: HTTP_REQUEST_REJECTED (在{{http-error-codes}}中)用于指示未处理请求。
否则不适用，因为 QUIC 处理了流管理。QUIC 层的 STREAM_ID_ERROR用于不正确打开的流。

CANCEL (0x8):
: {{http-error-codes}}中 HTTP_REQUEST_CANCELLED。

COMPRESSION_ERROR (0x9):
: 多个错误码定义于[QPACK].

CONNECT_ERROR (0xa):
: {{http-error-codes}}中 HTTP_CONNECT_ERROR。

ENHANCE_YOUR_CALM (0xb):
: {{http-error-codes}}中 HTTP_EXCESSIVE_LOAD。

INADEQUATE_SECURITY (0xc):
: 不适用，因为 QUIC 假设已经为所有连接提供了足够的安全保证。

HTTP_1_1_REQUIRED (0xd):
: {{http-error-codes}}中 HTTP_VERSION_FALLBACK。

需要分别为HTTP/2和HTTP/3定义错误代码。详见{{iana-error-codes}}.

# Change Log

> **RFC Editor's Note:**  Please remove this section prior to publication of a
> final version of this document.

## Since draft-ietf-quic-http-17

- HTTP_REQUEST_REJECTED is used to indicate a request can be retried (#2106,
  #2325)
- Changed error code for GOAWAY on the wrong stream (#2231, #2343)


## Since draft-ietf-quic-http-16

- Rename "HTTP/QUIC" to "HTTP/3" (#1973)
- Changes to PRIORITY frame (#1865, #2075)
  - Permitted as first frame of request streams
  - Remove exclusive reprioritization
  - Changes to Prioritized Element Type bits
- Define DUPLICATE_PUSH frame to refer to another PUSH_PROMISE (#2072)
- Set defaults for settings, allow request before receiving SETTINGS (#1809,
  #1846, #2038)
- Clarify message processing rules for streams that aren't closed (#1972, #2003)
- Removed reservation of error code 0 and moved HTTP_NO_ERROR to this value
  (#1922)
- Removed prohibition of zero-length DATA frames (#2098)


## Since draft-ietf-quic-http-15

Substantial editorial reorganization; no technical changes.

## Since draft-ietf-quic-http-14

- Recommend sensible values for QUIC transport parameters (#1720,#1806)
- Define error for missing SETTINGS frame (#1697,#1808)
- Setting values are variable-length integers (#1556,#1807) and do not have
  separate maximum values (#1820)
- Expanded discussion of connection closure (#1599,#1717,#1712)
- HTTP_VERSION_FALLBACK falls back to HTTP/1.1 (#1677,#1685)

## Since draft-ietf-quic-http-13

- Reserved some frame types for grease (#1333, #1446)
- Unknown unidirectional stream types are tolerated, not errors; some reserved
  for grease (#1490, #1525)
- Require settings to be remembered for 0-RTT, prohibit reductions (#1541,
  #1641)
- Specify behavior for truncated requests (#1596, #1643)

## Since draft-ietf-quic-http-12

- TLS SNI extension isn't mandatory if an alternative method is used (#1459,
  #1462, #1466)
- Removed flags from HTTP/3 frames (#1388, #1398)
- Reserved frame types and settings for use in preserving extensibility (#1333,
  #1446)
- Added general error code (#1391, #1397)
- Unidirectional streams carry a type byte and are extensible (#910,#1359)
- Priority mechanism now uses explicit placeholders to enable persistent
  structure in the tree (#441,#1421,#1422)

## Since draft-ietf-quic-http-11

- Moved QPACK table updates and acknowledgments to dedicated streams (#1121,
  #1122, #1238)

## Since draft-ietf-quic-http-10

- Settings need to be remembered when attempting and accepting 0-RTT (#1157,
  #1207)

## Since draft-ietf-quic-http-09

- Selected QCRAM for header compression (#228, #1117)
- The server_name TLS extension is now mandatory (#296, #495)
- Specified handling of unsupported versions in Alt-Svc (#1093, #1097)

## Since draft-ietf-quic-http-08

- Clarified connection coalescing rules (#940, #1024)

## Since draft-ietf-quic-http-07

- Changes for integer encodings in QUIC (#595,#905)
- Use unidirectional streams as appropriate (#515, #240, #281, #886)
- Improvement to the description of GOAWAY (#604, #898)
- Improve description of server push usage (#947, #950, #957)

## Since draft-ietf-quic-http-06

- Track changes in QUIC error code usage (#485)

## Since draft-ietf-quic-http-05

- Made push ID sequential, add MAX_PUSH_ID, remove SETTINGS_ENABLE_PUSH (#709)
- Guidance about keep-alive and QUIC PINGs (#729)
- Expanded text on GOAWAY and cancellation (#757)

## Since draft-ietf-quic-http-04

- Cite RFC 5234 (#404)
- Return to a single stream per request (#245,#557)
- Use separate frame type and settings registries from HTTP/2 (#81)
- SETTINGS_ENABLE_PUSH instead of SETTINGS_DISABLE_PUSH (#477)
- Restored GOAWAY (#696)
- Identify server push using Push ID rather than a stream ID (#702,#281)
- DATA frames cannot be empty (#700)

## Since draft-ietf-quic-http-03

None.

## Since draft-ietf-quic-http-02

- Track changes in transport draft

## Since draft-ietf-quic-http-01

- SETTINGS changes (#181):
    - SETTINGS can be sent only once at the start of a connection;
      no changes thereafter
    - SETTINGS_ACK removed
    - Settings can only occur in the SETTINGS frame a single time
    - Boolean format updated

- Alt-Svc parameter changed from "v" to "quic"; format updated (#229)
- Closing the connection control stream or any message control stream is a
  fatal error (#176)
- HPACK Sequence counter can wrap (#173)
- 0-RTT guidance added
- Guide to differences from HTTP/2 and porting HTTP/2 extensions added
  (#127,#242)

## Since draft-ietf-quic-http-00

- Changed "HTTP/2-over-QUIC" to "HTTP/QUIC" throughout (#11,#29)
- Changed from using HTTP/2 framing within Stream 3 to new framing format and
  two-stream-per-request model (#71,#72,#73)
- Adopted SETTINGS format from draft-bishop-httpbis-extended-settings-01
- Reworked SETTINGS_ACK to account for indeterminate inter-stream order (#75)
- Described CONNECT pseudo-method (#95)
- Updated ALPN token and Alt-Svc guidance (#13,#87)
- Application-layer-defined error codes (#19,#74)


## Since draft-shade-quic-http2-mapping-00

- Adopted as base for draft-ietf-quic-http
- Updated authors/editors list

# Acknowledgements
{:numbered="false"}

The original authors of this specification were Robbie Shade and Mike Warres.

A substantial portion of Mike's contribution was supported by Microsoft during
his employment there.

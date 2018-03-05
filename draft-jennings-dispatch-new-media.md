%%%
    title = "New Media Stack"
    abbrev = "new-media"
    category = "std"
    docName = "draft-jennings-dispatch-new-media"
    ipr = "trust200902"

    [pi]
    symrefs = "yes"
    sortrefs = "yes"
    toc = "yes"

    [[author]]
    initials = "C."
    surname = "Jennings"
    fullname = "Cullen Jennings"
    organization = "Cisco"
      [author.address]
      email = "fluffy@iii.ca"

%%%


.# Abstract

A sketch of a proposal for a new meida stack for interactive
communications.

{mainmatter}


# Introduction

This draft proposes a new media stack to replace the existing stack
RTP, DTLS-SRTP, and SDP Offer Answer.  The key parts of this stack are
connectivity layer, the transport layer, the media layer, a control
API, and the signalling layer.

The connectivity layer uses a simplified version of ICE, called
snowflake [@I-D.jennings-dispatch-snowflake], to find connectivity
between endpoints and change the connectivity from one address to
another as different networks become available or disappear. It is
based on ideas from [@I-D.jennings-mmusic-ice-fix].

The transport layer uses QUIC to provide a hop by hop encrypted,
congestion controlled transport of media. Although QUIC does not
currently have all of the partial reliability mechanisms to make this
work, this draft assumes that they will be added to QUIC.

The media layer uses existing codecs and packages them along with
extra header information to provide information about, when the
sequence needs to be played back, which camera it came from, and media
streams to be synchronized.

The control API is an abstract API that provides a way for the media
stack to report it capabilities and features and a way for the an
application tell the media stack how it should be
configured. Configuration includes what codec to use, size and frame
rate of video, and where to send the media.

The signalling layer is based on an advertisement and proposal
model. Each endpoint can create an advertisement that describes what
it supports including things like supported codecs and maximum
bitrates. A proposal can be sent to an endpoint that tells the
endpoint exactly what media to send and receive and where to send
it. The endpoint can accept or reject this proposal in total but
cannot change any part of it.


# Terminology

* media stream: Stream of information from a single sensor. For example,
  a video stream from a single camera. A stream may have multiple
  encodings for example video at different resolutions.

* encoding: A encoded version of a stream. A given stream may have
  several encodings at different resolutions. One encoding may depend
  on other encodings such as forward error corrections or in the case
  of scalable video codecs.

* flow: A logical transport between two computers. Many media strams
  can be transported over a single flow. The actually IP address and
  ports used to transport data in the flow may change over time as
  connectivity changes.

* message: some data or media that to be sent across the network along
  with metadata about it. Similar to an RTP packet.

* media source: a camera, microphone or other source of data on an
  endpoint

* media sink: a speaker, screen, or other destination for data on an
  endpoint

* TLV: Tag Length Value. When used in the draft, the Tag, Length, and
  any integer values are coded as variable length integers similar to
  how this is done in CBOR.


# Connectivity Layer

## Snowflake - New ICE

All that is needed to discover the connectivity is way to:

* Gather some IP/ports that may work using TURN relay, STUN, and local
  addresses.

* A controller, which might be running in the cloud, to inform a
  client to send a STUN packet from a given source IP/port to a given
  destination IP/port.
  
* The receiver notifies the controller about infomation on received
  STUN packets.
  
* The controller can tell the sender the secret that was in the packet
  to prove consent of the receiver to receive data then the sending
  client can allow media to flow over that connection.
  
The actually algorithm used to decide on what pairs of addresses are
tested and in what order does not need to be agreed on by both the
sides of the call - only the controller needs to know it. This allows
the controller to use machine learning, past history, and heurstics to
find an optimal connection much faster than something like ICE.

The details of this aproach are described in
[@I-D.jennings-dispatch-snowflake].

## New Stun

The speed of setting up a new media flow is often determined by how
many STUN checks need to be done. If the STUN packets are smaller,
then the stun checks can be done faster without risk of causing
congestion. The STUN server and client share a secret that they use
for authentication and encryption. When talking to a public STUN
server this secret is the empty string.

### Stun Request

A STUN request consists of the following TLVs:

* a magic number that uniquely identifies this as a STUN request
  packet with minimal risk of collision when multiplexing.

* a transaction ID that uniquely identifies this request and does not
  change in retransmissions of the same request.

* an optional sender secret that can be used by the receiver to prove
  that it received the request. In WebRTC the browser would create the
  secret but the JavaScript on the sending side would know the value.

The packet is encrypted by using the secret and an AEAD crypto to
create a STUN packet where the first two fields are the magic number
and transaction ID which are only authenticated followed by the rest
of the fields that are authenticated and encrypted followed by the
AEAD authentication data.

The STUN requests are transmitted with the same retransmission and
congestion algorithms as STUN in WebRTC 1.0

### Stun Response

A STUN response consists of the following TLVs:

* a magic number that uniquely identifies this as a STUN response
  packet with minimal risk of collision when multiplexing.

* the transaction ID from the request.

* the IP address and port the request was received from.

The packet is encrypted where the first two fields are the magic
number and transaction ID which are only authenticated followed by the
rest of the fields that are authenticated and encrypted followed by
the AEAD authentication data.

## New TURN 

Out of band, the client tells the TURN server the fingerprint of the
cert it uses to authenticate with and the TURN server gives the client
two public IP:port address pairs. One is called inbound and other
called outbound. The client connects to the outbound port and
authenticates to TURN server using the TLS domain name of server. The
TURN server authenticates the client using mutual TLS with fingerprint
of cert provided by the client. Any time a message or stun packet is
received on the matched inbound port, the TURN server forwards it to
the client(s) connected to the outbound port.

A single  TURN connection can be used for mutliple differnt calls
or session at the same time and a cleint could choose to allocate the
TURN connection at the time that it started up. It does not need to be
done on a per session basis.

The client can not send from the TURN server. 

~~~
        Client A      Turn Server     Client B
    (Media Receiver)                (Media Sender) 
          |              |              |
          |              |              |
          |              |              |
          |(1) OnInit Register (A's fingerprint)
          |------------->|              |
          |              |              |
          |              |              |
          |(2) Register  Response (Port Pair (L,R))
          |<-------------|              |
          |              |              |
          |              |              |
          | L(left of Server), R(Right of Server)
          |              |              |
          |              |              |
          |              |              |
          |(3) Setup TLS Connection (L port)
          |..............|              |
          |              |              |
          |              |              |
          |              |              | B send's media to A
          |              |              |
          |              |              |
          |              |              |
          |              |(4) Media Tx (Received on Port R)
          |              |<-------------|
          |              |              |
          |              |              |
          |(5) Media Tx (Sent from Port L)
          |<-------------|              |
          |              |              |
          |              |              |
~~~

# Transport Layer

The responsibility of the transport layer is to provide an end to end
crypto layer equivalent to DTLS and they must ensure adequate
congestion control. The transport layer brings up a flow between two
comptuers. This flow can be used by multiple media streams. 

The MTI transport layer is QUIC with packets sent in an unreliable
mode.
 
This is secured by checking the fingerprints of the DTLS connection
match the fingerprints provided at the control layer or by checking
the names of the certificates match what was provided at control
layer.

The transport layer needs to be able to set the DSCP values in
transmitting packets as specified by the control layer.

The transport MAY provide a compression mode to remove the redundancy
of the non-encrypted portion of the media messages such as
GlobalEncodingID. For example, a GlobalEncodingID could be mapped to a
QUIC channel and then it could be removed before sending the message
and added back on the receiving side.

The transport need to be able to ensure that it has a very small
chance of being confused with the STUN traffic it will be multiplexed
with.


# Media Layer - New RTP

Each message consist of a set of TLV headers with metadata about the
packet, followed by payload data such as the output of audio or video
codec.

There are several message headers that help the receiver understand
what to do with the media. The TLV header are the follow:

* Conference ID: Integer that will be globally unique identifier for
  the for all applications using a common call signalling system. This
  is set by the proposal.

* Endpoint ID: Integer to uniquely identify the endpoint with within
  scope of conference ID. This is set by the proposal.

* Source ID: integer to uniquely identify the input source within the
  scope a endpoint ID. A source could be a specific camera or a
  microphone. This is set by the endpoint and included in the
  advertisement.

* Sink ID: integer to uniquely identify the sink within the scope a
  endpoint ID. A sink could be a speaker or screen. This is set by the
  endpoint and included in the advertisement.

* Encoding ID: integer to uniquely identify the encoding of the stream
  within the scope of the stream ID. Note there may be multiple
  encodings of data from the same source. This is set by the proposal.

* Salt : salt to use for forming the initialization vector for
  AEAD. The salt shall be sent as part of the packet and need not be
  sent in all
the packets. This is created by the endpoint sending the message.  

* GlobalEncodingID: 64 bit hash of concatenation of conference ID,
  endpoint ID, stream ID, encoding ID

* Capture time: Time when the first sample in the message was
  captured. It is a NTP time in ms with the high order bits
  discarded. The number of bits in the capture time needs to be large
  enough that it does not wrap in for the lifetime of this
  stream. This is set by the endpoint sending the message.

* Sequence ID: When the data captured for a single point in time is
  too large to fit in a single message, it can be split into multiple
  chunks which are sequentially numbered starting at 0 corresponding
  to the first chunk of the message. This is set by the endpoint
  sending the message.

* GlobalMessageID: 64 bit hash of concatenation of conference ID,
  endpoint ID, encoding ID, sequence ID

* Active level: this is a number from 0 to 100 indicates the level
  that the sender of this media wishes it to be considered active
  media. For example if it was voice, it would be 100 if the person
  was clearly speaking, and 0 if not, and perhaps a value in the
  middle if it was uncertain. This allows an media switch to select
  the active speaker in the in a conference call.

* Room location: get left right screens correct for video and allow
  for spatial audio

* Reference Frame : bool to indicate if this message is part of a
  reference frame

* DSCP : DSCP to use on transmissions of this message and future
  messages on this GlobalEncodingID

* Layer ID : Indication which layer is for scalable video codecs. SFU
  may use this to selectively drop a frame.

The keys used for the AEAD are unique to a given conference ID and
endpoint ID.

If the message has any of the following headers, they must occur in
the following order followed by all other headers:

1.  GlobalEncodingID,
2. GlobalMessageID,
3. conference ID,
4. endpoint ID,
5. encoding ID, 
6. sequence ID,
7. active level,
8. DSCP

Every second there much be at least one message in each encoding that
contains:

* conference ID,
* endpoint ID,
* encoding ID,
* salt,
* and sequence ID headers 

but they are not needed in every packet. 

If none of these are available, then the GlobalEncodingID must be
included.

The sequence ID or GlobalMessageID is required in every message and
periodically there should be message with the capture time.


## Securing the messages

The whole message is end to end secured with AEAD. The headers are
authenticated while the payload data is authenticated and
encrypted. Similar to how the IV for AES-GCM is calculated in SRTP, in
this case the IV is computed by xor'ing the salt with the
concatenation of the GlobalEncodingID and low 64 bits of sequence
ID. The message consists of the authenticated data, followed by the
encrypted data , then the authentication tag.

## Sender requests

The control layer supports requesting retransmission of a particular
media message identified by IDs and capture time it would contain.

The control layer supports requesting a maximum rate for each given
encoding ID.

## Data Codecs

Data messages including raw bytes, xml, senml can all be sent just
like media by selecting an appropriate codec and a software based
source or sink. An additional parameter to the codec can indicate if
reliably delivery is needed and if in order delivery is needed.


## Forward Error Correction

A new Reed-Solomon based FEC scheme based on
draft-ietf-payload-flexible-fec-scheme that provides FEC over messages
needs to be defined.

## MTI Codecs

Implementation MUST support at least G711, Opus, H.264 and AOM-1

Video codecs use square pixels.

Video codecs MUST support any aspect ratio within the limits of their
max width and height.

Video codecs MUST support a min width and min height of 1. 

All video on the wire is oriented such that the first scan line in the
frame is up and first pixel in the scan line is on the left. 

T.38 fax and DTMF are not supported. Fax can be sent as a TIFF imager
over a data channel and DTFM can be done as an application specific
information over a data channel.

## Message Key Agreement

The secret for encrypting messages can be provided in the proposal by
value or by a reference. The reference approach allows the client to
get it from a messaging system where the server creating the proposal
may not have access to the the secret. For example, it might come from
a system like [@I-D.barnes-mls-protocol].

# Control Layer

The control layer needs an API to find out what the capabilities of
the device are, and then a way to set up sending and receiving
stream. All media flow are only in one direction. The controll is
broken intto controll of conenctiviuty and transpotrs, and controll of
media streams.

## Media Capabilities API

Send and receive codecs are consider seprate codecs and can have
seprate capabilities though the default to the same if not specified seperately. 

For each send or receive audio codec, an API to learn:

* codec name 
* the max sample rate
* the max sample size 
* the max bitrate
* max number of streams to decode 

For each send or receive video codec, an API to learn:

* codec name 
* the max width
* the max height
* the max frame rate 
* the max pixel  depth 
* the max bitrate
* the max pixel rate ( pixels / second )
* max number of streams to decode
* max number of stream to encode 

## Transport Capabilties API


An API to get information for remote connectivity inlcuidng:

*  set the IP, port, and credential for each TURN server 
* can return the IP, port tuple for the remote side to send to TURN 
server
* gather local IP, port, protcol tuples for receiving media
* report SHA256 fingrerpinrt of local TLS certificate 


## Transport Configuration API


To create a new flow, the information that can be configured is:

* turn server to use 
* list of IP, Port, Protcol tuples to try connecting to
* TLS fingerprint of far side 

An api to allow modificaiton of the follow atributes of a flow:

* total max bandwidwith for flow
* forward error correection scheme for flow
* retransmition scheme for flow 
* adition IP, Port, Protcol pairs to send to that may inmprove connectivity 

## Media Configuration API 

For all streams:

* set confernce ID
* set endpoint ID
* set encoding ID
* salt and secret for AEAD 

For each transmitted audio steam, a way to set the:

* audio codec to use 
* media source to connect 
* max encoded bitrate
* sample rate
* sample size
* number of channels to encode
* packetization time
* process as one of : automatically set, raw, speach, music
* DSCP value to use 

For each transmitted video stream, a way to set

* video codec to use
* media source to connect to 
* max width and max height 
* max encoded bitrate
* sample rate
* sample size
* process as one of :  automatically set, rapidly changing video, fine detail video 
* DSCP value to use
* for layered codec, a layer ID and set of layers IDs this depends on 

For each transmitted video stream, a way to tell it to:

* encode the next frame as an intra frame

For each received audo stream:

* audio codec to use 
* media sink to connect to
* lip sync flag 

For each reveived video stream:

* video codec to use 
* media sink to connect to
* lip sync flag 

Note on lip sync: For any streams that have the lip sync flag set to
true, the render attempts to syncronize thier play back. 

## Flow Metrics API

For each flow, report:

* report connectivity state
* report bits sent 
* report packets lost
* report estimated RTT
* report SHA256 fingerprint for certificate of far side

## Stream Metrics API

For sending streams:

* Bits sent
* packets lost 

For receiving streams:

* capture time of most recently receives packet 
* endpoint ID of more recently received packet
* bits reeceived
* packets losts

For video streams (send & receive):

* current encoded width and height
* current encoded frame rate

# Call Signalling 

Call signalling is out of scope for usages like WebRTC but other 
usages may want a common REST API they can use. It works be having the 
client connect to a server when it starts up and send its current 
advertisement and open a web socket to receive proposals from the 
server. A client can make a rest call indicating the parties(s) it 
wishes to connect to and the server will then send propels to all 
clients that connect them. The signalling is based on the the 
advertimsment proposal ideas from [@I-D.peterson-sipcore-advprop]. 


## Switched Forwarding Unit (SFU) 

When several clients are in conference call, the SFU can forward 
packets based on looking at which clients needs a given 
GlobalEncodingID. By looking at the "active level", the SFU can figure 
out which endpoints are the active speaker and forward only those. The 
SFU never changes anything in the message. 


# Signalling Examples

## Simple Audio Example

### simple audio advertisement

~~~
{
  "receiveAt":[
    {
      "relay":"2001:db8::10:443",
      "stunSecret":"s8i739dk8",
      "tlsFingerprintSHA256":"1283938"
    },
    {
      "stun":"203.0.113.10:43210",
      "stunSecret":"s8i739dk8",
      "tlsFingerprintSHA256":"1283938"
    },
    {
      "local":"192.168.0.2:443",
      "stunSecret":"s8i739dk8",
      "tlsFingerprintSHA256":"1283938"
    }
  ],
  "sources":[
    {
      "sourceID":1,
      "sourceType":"audio",
      "codecs":[
        {
          "codecName":"opus",
          "maxBitrate":128000
        },
        {
          "codecName":"g711"
        }
      ]
    }
  ],
  "sinks":[
    {
      "sinkID":1,
      "sourceType":"audio",
      "codecs":[
        {
          "codecName":"opus",
          "maxBitrate":256000
        },
        {
          "codecName":"g711"
        }
      ]
    }
  ]
}
~~~

### simple audio proposal

~~~
{
  "receiveAt":[
    {
      "relay":"2001:db8::10:443",
      "stunSecret":"s8i739dk8"
    },
    {
      "stun":"203.0.113.10:43210",
      "stunSecret":"s8i739dk8"
    },
    {
      "local":"192.168.0.10:443",
      "stunSecret":"s8i739dk8"
    }
  ],
  "sendTo":[
    {
      "relay":"2001:db8::20:443",
      "stunSecret":"20kdiu83kd8",
      "tlsFingerprintSHA256":"9389739"
    },
    {
      "stun":"203.0.113.20:43210",
      "stunSecret":"20kdiu83kd8",
      "tlsFingerprintSHA256":"9389739"
    },
    {
      "local":"192.168.0.20:443",
      "stunSecret":"20kdiu83kd8",
      "tlsFingerprintSHA256":"9389739"
    }
  ],
  "sendStreams":[
    {
      "conferenceID":4638572387,
      "endpointID":23,
      "sourceID":1,
      "encodingID":1,
      "codecName":"opus",
      "AEAD":"AES128-GCM",
      "secret":"xy34",
      "maxBitrate":24000,
      "packetTime":20
    }
  ],
  "receiveStreams":[
    {
      "conferenceID":4638572387,
      "endpointID":23,
      "sinkID":1,
      "encodingID":1,
      "codecName":"opus",
      "AEAD":"AES128-GCM",
      "secret":"xy34"
    }
  ]
}
~~~

## Simple Video Example

Advertisement for simple send only camera with no audio

~~~
{
  "sources":[
    {
      "sourceID":1,
      "sourceType":"video",
      "codecs":[
        {
          "codecName":"aom1",
          "maxBitrate":20000000,
          "maxWidth":3840,
          "maxHeight":2160,
          "maxFrameRate":120,
          "maxPixelRate":248832000,
          "maxPixelDepth":8
        }
      ]
    }
  ]
}
~~~

### Proposal sent to camera

~~~
{
  "sendTo":[
    {
      "relay":"2001:db8::20:443",
      "stunSecret":"20kdiu83kd8",
      "tlsFingerprintSHA256":"9389739"
    }
  ],
  "sendStreams":[
    {
      "conferenceID":0,
      "endpointID":0,
      "sourceID":0,
      "encodingID":0,
      "codecName":"aom1",
      "AEAD":"NULL",
      "width":640,
      "height":480,
      "frameRate":30
    }
  ]
}
~~~

## Simulcast Video Example

Advertisement same as simple camera above but proposal has two
streams with different encodingID.


~~~
{
  "sendTo":[
    {
      "relay":"2001:db8::20:443",
      "stunSecret":"20kdiu83kd8",
      "tlsFingerprintSHA256":"9389739"
    }
  ],
  "sendStreams":[
    {
      "conferenceID":0,
      "endpointID":0,
      "sourceID":0,
      "encodingID":1,
      "codecName":"aom1",
      "AEAD":"NULL",
      "width":1920,
      "height":1080,
      "frameRate":30
    },
    {
      "conferenceID":0,
      "endpointID":0,
      "sourceID":0,
      "encodingID":2,
      "codecName":"aom1",
      "AEAD":"NULL",
      "width":240,
      "height":240,
      "frameRate":15
    }
  ]
}
~~~

## FEC Example

### Advertisement includes a FEC codec.  
~~~
{
  "sources":[
    {
      "sourceID":1,
      "sourceType":"video",
      "codecs":[
        {
          "codecName":"aom1",
          "maxBitrate":20000000,
          "maxWidth":3840,
          "maxHeight":2160,
          "maxFrameRate":120,
          "maxPixelRate":248832000,
          "maxPixelDepth":8
        },
        {
          "codecName":"flex-fec-rs"
        }
      ]
    }
  ]
}
~~~

###  Proposal sent to camera
~~~
{
  "sendTo":[
    {
      "relay":"2001:db8::20:443",
      "stunSecret":"20kdiu83kd8",
      "tlsFingerprintSHA256":"9389739"
    }
  ],
  "sendStreams":[
    {
      "conferenceID":0,
      "endpointID":0,
      "sourceID":0,
      "encodingID":1,
      "codecName":"aom1",
      "AEAD":"NULL",
      "width":640,
      "height":480,
      "frameRate":30
    },
    {
      "conferenceID":0,
      "endpointID":0,
      "sourceID":0,
      "encodingID":2,
      "AEAD":"NULL",
      "codecName":"flex-fec-rs",
      "fecRepairWindow":200,
      "fecRepairEncodingIDs":[
        1
      ]
    }
  ]
}
~~~

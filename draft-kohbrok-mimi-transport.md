---
title: MIMI Transport
abbrev: MT
docname: draft-kohbrok-mimi-transport-latest
category: info

ipr: trust200902
area: Security
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -  ins: K. Kohbrok
    name: Konrad Kohbrok
    organization: Phoenix R&D
    email: konrad.kohbrok@datashrine.de
 -  ins: R. Robert
    name: Raphael Robert
    organization: Phoenix R&D
    email: ietf@raphaelrobert.com

--- abstract

This document defines an HTTPS based transport layer for use with the MIMI
Protocol.

--- middle

# Introduction

This document describes an HTTP-based transport layer protocol for use with all
MIMI sub-protocols. The MIMI transport protocol provides an envelope for for
MIMI event types and takes care of message authentication.

This document also describes the endpoints that are served by the individual
MIMI functionalities.

# Authentication

The MIMI transport protocol uses the mutually authenticated mode of TLS to
provide authentication for server-to-server communication.

TODO: More information specific to how TLS should be used, i.e. mandate best
practices that make sense in a mutually authenticated scenario that involves two
WebPKI based certificates.

# Framing

The MIMI transport protocol uses a simple framing structure that includes the
event, as well as a small header that signals the protocol version, as well as
the event type contained in the envelope.

~~~
enum {
    reserved(0),
    mimi10(1),
    (65535)
} ProtocolVersion;

// See the "MIMI Event Types" IANA registry for values
// e.g. "mimi.delivery-service.add"
opaque EventType;

struct {
  ProtocolVersion version = mimi10;
  EventType event_type;
  opaque serialized_event<V>;
} EventEnvelope;
~~~

# Endpoint Discovery

A messaging provider that wants to query the endpoint of another messaging
provider first has to discover the fully qualified domain name under which
Delivery Service of that provider can be reached. It does so by performing a GET
request to `provider.com/.well-known/mimi/ds-domain`. provider.com could for
example answer by providing the domain `ds.provider.com` (assuming that this is
where it responds to the REST endpoints defined below).

# REST Endpoints

The following REST endpoints can then be used to access the different
functionalities of the Delivery Service.

As the Delivery Service relies on TLS encoded structs, all requests to endpoints
described below should be marked as `Content-type: application/octet-stream`.

All structs and concepts referred to below are defined in
draft-robert-mimi-delivery-service, where their underlying functionality is
defined in more detail.

## Get Public Signature Key

~~~
GET /signature_key
Content-type: application/octet-stream

Body
TLS serialized DSRequest

Response
TLS serialized DSResponse
~~~

TODO: We will likely want to respond with more info than just the public key,
i.e. lifetime, signature scheme, etc.


## Process Group Message

~~~
POST /group_operation
Content-type: application/octet-stream

Body
TLS serialized DSRequest

Response
TLS serialized DSResponse
~~~

This REST endpoint provides access to all operations associated with an existing
MLS group on the Delivery Service such as delivering application messages,
adding group members, removing group members, updating key material, etc. The
payloads for this endpoint are generally provided (and signed) by a member of
the corresponding group rather than the service provider of that member. The
exact operation, as well as the target group ID is determined by the payload
itself rather than an HTTP header, the path or any other query parameter.

## Welcome Information

~~~
GET /welcome_information
Content-type: application/octet-stream

Body
TLS serialized DSRequest

Response
TLS serialized DSResponse
~~~

Through this endpoint, a provider can obtain information required to join the
group for clients that have already received a Welcome message. The DS responds
with the group’s RatchetTree, as well as authentication information of existing
group members.

## External Commit Information

~~~
GET /external_commit_information
Content-type: application/octet-stream

Body
TLS serialized DSRequest

Response
TLS serialized DSResponse
~~~

Guest providers can use this endpoint to obtain information that allows a client
to join a group without a Welcome message from an existing group member.

## Verification Key

~~~
GET /verification_key
Content-type: application/octet-stream

Body
TLS serialized VerificationKeyRequest

Response
TLS serialized VerificationKeyResponse
~~~

This allows guest providers to obtain the verification key of this provider.
This allows other providers to authenticate queries originating from this
provider.

## Deliver Connection Request

~~~
POST /connection_request
Content-type: application/octet-stream

Body
TLS serialized QueueingServiceRequest

Response
TLS serialized QueueingServiceResponse
~~~

This endpoint lets other providers deliver connection establishment request to
clients of this provider.

## Deliver Message

~~~
POST /deliver_message
Content-type: application/octet-stream

Body
TLS serialized QueueingServiceRequest

Response
TLS serialized QueueingServiceResponse
~~~

An owning provider can deliver messages from one of its owned groups to this
endpoint, if one of the group’s clients is associated with this provider.

## Connection KeyPackage Retrieval

~~~
POST /connection_key_packages
Content-type: application/octet-stream

Body
TLS serialized ConnectionKeyPackageRequest

Response
TLS serialized ConnectionKeyPackageResponse
~~~

Allows another provider to retrieve KeyPackages for use during the connection
establishment process between two users.

## Group KeyPackage Retrieval

~~~
POST /group_key_packages
Content-type: application/octet-stream

Body
TLS serialized GroupKeyPackageRequest

Response
TLS serialized GroupKeyPackageResponse
~~~

Allows another provider to retrieve KeyPackages that can be used to add another
user or one of its clients to an existing group.

# Rate-limiting

The MIMI transport protocol itself doesn’t include any rate-limiting measures.
However, traditional rate-limiting (e.g. based on sender IP) can be applied, as
well as rate-limiting based on information in the message body such as Group ID
(e.g. in the case of the `/welcome_information` endpoint) or group member (in
the case of the `/group_operation` endpoint). More fine-grained rate-limiting
can be applied through the use of the emerging Privacy Pass protocol
(draft-ietf-privacypass-auth-scheme).

# IANA Considerations

IANA has created the following registries:
* Event Types

## Event Types

An event type denotes the nature of a given payload in the context of the MIMI
protocol. The event type is a string that is composed of substrings separated by
dots.

The first substring is "mimi", followed by the document that defines the
corresponding event payload, which in turn is followed by the name of the event.
The MIMI event specified in the MIMI DS document that signals the addition of a
client would for example be denoted by the string
~~~
"mimi.delivery-service.add"
~~~

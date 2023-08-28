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
MIMI sub-protocols. The MIMI transport protocol provides a unifying framing
structure for all MIMI event types and takes care of message authentication.

This document also describes the endpoints that are served by the individual
MIMI functionalities.

# Authentication

The MIMI transport protocol provides mutual authentication for server-to-server
communication, as well as additional sender-authentication for queries
originating from clients.

## Authentication key material

There are three types of key material used in this document. Recipient
authentication key material, sender authentication key material for server, and
sender authentication key material for clients.

* Recipient authentication key material consists of an X.509 certificate chain
  that each server uses to authenticate itself towards senders of HTTPS queries
  to all endpoints used in this document.
* Sender authentication key material of servers consists of a public/private key
  pair (server signing keys), which servers use to sign queries they make to
  endpoints specified in this document and that the recipients can use to
  authenticate the sender. Senders make the public key available for recipients
  to fetch via one of the endpoints defined in {{XXX}}.
* Sender authentication key material of clients is provided by the MIMI DS layer
  of the MIMI protocol.

TODO: Sender authentication key material needs to be specified further,
especially how it can be end-to-end authenticated to the client's AS.

## Server-to-server authentication

For all HTTP queries to endpoints defined in this document, the sender first
establishes a TLS connection with version at least 1.3 to protect
confidentiality and to unilaterally authenticate the recipient of the query.

In addition, the sending server signs its query using its server signing keys.

To prevent forwarding attacks, the payload of each query includes both sender
and recipient.

TODO: We should improve this in the future to allow for mutually authenticated
channels or at least batching.

## Client-to-server authentication

Some events contained in queries to the endpoints defined in this document
originate from clients and are signed by the client using its client specific
key material. The receiving server (or client) can authenticate such events by
verifying the sender's signature.

# Framing

The framing structure somewhat mimicks that of MLS.

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
  EventType event_type;
  opaque event_payload;
} Event;

struct {
    ProtocolVersion version = mimi10;
    opaque sender_domain;
    opaque recipient_domain; // Only one for now

    Event event;

    opaque signature;
} S2SMessage;
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
---
title: MIMI over HTTPS
abbrev: MoH
docname: draft-kohbrok-mimi-moh-latest
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

This document an HTTPS based transport layer for use with the MIMI Delivery Service.

--- middle

# Introduction

This document describes an HTTP-based transport layer protocol for use with the
Delivery Service protocol specified in draft-robert-mimi-delivery-service, as
well as other MIMI-relevant components such as contact discovery.

## Transport Security and Authentication

All HTTP queries described in this document MUST use TLS with version 1.3 or
higher to protect confidentiality and authenticity of the payloads. Delivery
Service payloads are typically authenticated by the sender through the use of
signatures and rely on HTTPS to authenticate the recipient. To prevent
forwarding attacks, the payloads of the Delivery Service include both sender and
recipient. The provider should thus always verify that the recipient corresponds
to its own provider name.

## Endpoint Discovery

A messaging provider that wants to query the endpoint of another messaging
provider first has to discover the fully qualified domain name under which
Delivery Service of that provider can be reached. It does so by performing a GET
request to `[provider.com](http://provider.com)/.well-known/mimi/ds-domain`.
provider.com could for example answer by providing the domain
`[ds.provider.com](http://ds.provider.com)` (assuming that this is where it
responds to the REST endpoints defined below).

## REST Endpoints

The following REST endpoints can then be used to access the different
functionalities of the Delivery Service.

As the Delivery Service relies on TLS encoded structs, all requests to endpoints
described below should be marked as `Content-type: application/octet-stream`.

All structs and concepts referred to below are defined in
draft-robert-mimi-delivery-service, where their underlying functionality is
defined in more detail.

### Process Group Message

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

### Welcome Information

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

### External Commit Information

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

### Verification Key

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

### Deliver Connection Request

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

### Deliver Message

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

### Connection KeyPackage Retrieval

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

### Group KeyPackage Retrieval

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

## Rate-limiting

The MoH protocol itself doesn’t include any rate-limiting measures. However,
traditional rate-limiting (e.g. based on sender IP) can be applied, as well as
rate-limiting based on information in the message body such as Group ID (e.g. in
the case of the `/welcome_information` endpoint) or group member (in the case of
the `/group_operation` endpoint). More fine-grained rate-limiting can be applied
through the use of the emerging Privacy Pass protocol
(draft-ietf-privacypass-auth-scheme

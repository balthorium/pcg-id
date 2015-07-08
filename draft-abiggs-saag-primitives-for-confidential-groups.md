---
title: Primitives for Confidential Group Communications
abbrev: primitives-for-confidential-group-communications
docname: draft-abiggs-saag-primitives-for-confidential-groups-00
date: 2015-07-03
category: std
ipr: trust200902
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: A. Biggs
    name: Andrew Biggs
    organization: Cisco Systems
    email: adb@cisco.com
 -
    ins: S. Cooley 
    name: Shaun Cooley
    organization: Cisco Systems
    email: shcooley@cisco.com

normative:

  RFC2119:

  RFC4949:
  
  RFC7565:

  RFC7515:
  
  RFC7516:
  
  RFC7517:
  
  RFC7518:  

  I-D.miller-saag-key-discovery:
  
  I-D.newton-json-content-rules:

informative:
  
--- abstract

This specification defines generalized primitives for use in establishing end-to-end confidentiality in multiparty communications.  These primitives address three essential elements of secure group communications: authentication, group membership, and secure key exchange.  A core objective of this specification is to define these primitives as common constructs for private group communications and in a manner that allows for flexibility in how they are integrated and deployed as part of new and existing group communications applications. 

--- middle

# Introduction

This specification defines application agnostic primitives for use in establishing authorization and end-to-end confidentiality in multiparty communications.  These primitives address three essential elements of secure group communications:

 * Authentication
 * Group Membership
 * Secure Key Exchange 
  
Authentication is based on the identification of interoperating entities by acct URI and proof of possession of the private counterpart of a public key discoverable through a key discovery service {{I-D.miller-saag-key-discovery}} available from a well-known URL.

Authorization is based on the group membership classification of authenticated entities, as represented in the form of a Group Membership Block Chain (GMBC) structure defined by this specification.  Critical properties of the GMBC are tamper-resistance, efficient mutability, broad deployability, and integrity in the context of distributed handling.

The secure exchange of keys is based on existing key wrapping standards which allow for the sharing of encrypted key material with multiple explicitly identified recipients.  This strategy builds on the GMBC based authorization model by taking advantage of reliable group membership classification when addressing wrapped keys to other members.

A goal of this specification is to define these primitives in such a way as they may be suitable for both centralized and distributed management.  It is also a goal to describe these primitives in terms of accepted standards for cryptographic processing and infrastructure.

A non-goal of this specification is to define the means by which these primitives are exchanged among interoperating entities involved in group communications.  Rather these are building blocks for extending group confidentiality to both new and existing communications and content sharing protocols.  With that in mind, however, this specification does advance the notion of recognizing two general classes of deployment for these primitives: "centralized" and "decentralized".  

An additional non-goal of this specification is the establishment of mechanisms for the authentication of sender identity for encrypted data exchanged over a confidential group communications resource.  Care is taken to ensure that only authenticated members of a group may decipher secured communications, however the means by which group members may reliably identify the originator of some particular communications data is out of scope for this specification. 

## Terminology {#terminology}

curator

> An entity optionally identified in the genesis block of a GMBC as a permanent member of a group that performs the function of accepting and distributing GMBC updates and GKs among group members.

entity

> An entity is a user or automated agent that is uniquely identifiable by an acct URI {{RFC7565}} and for which there exists a key discovery service {{I-D.miller-saag-key-discovery}} through which public keys may be obtained for that URI.

genesis block

> The first block in a group membership block chain.

group

> A group is a set of entities whose membership wish to engage in secure and authenticated multiparty communications over some group communications resource.

group communications resource

> A group communications resource is any uniquely identifiable streamed or discrete data path that represents an exchange of personal communications between two or more entities.

group key (GK)

> A group key is an encrypted object containing symmetric key material and associated metadata secured by the public key(s) of other group members.

group membership block chain (GMBC)

> A group membership block chain is a primitive defined by this specification for the purpose of providing an effective means for defining, updating, sharing, and verifying the membership of a group.

This document uses the terminology from {{RFC7515}}, {{RFC7516}}, {{RFC7517}}, and {{RFC7518}} when discussing JOSE technologies.  Most security-related terms in this document are to be understood in the sense defined in {{RFC4949}}.

## Notational Conventions

In this document, the key words "MUST", "MUST NOT", "REQUIRED",
"SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY",
and "OPTIONAL" are to be interpreted as described in BCP 14, RFC 2119
{{RFC2119}}.

# Overview

This section provides a general overview of three basic constructs for enabling authentication, authorization, and secure key exchange in confidential group communications.

## Authentication by Public Key Discovery

In the context of this specification, entity authentication is defined as the demonstration of possession of the private component of an asymmetric key pair.  Specifically, an entity uniquely identified by an acct URI {{RFC7565}} may be authenticated by demonstrating possession of the private counterpart of one or more public keys as may be discovered using that acct URI and the mechanisms described in {{I-D.miller-saag-key-discovery}}.  

## Authorization by Group Membership Block Chain

In the context of this specification, authorization is defined as the classification of any given entity as either a "member" or "non-member" with respect to a group.  A member of the group is by definition authorized to receive keying material used to encrypt group communications, and likewise a non-member is not.  A member is also entitled to alter the membership of the group.  The means by which group membership classification established, updated, and validated is through operations on a Group Membership Block Chain (GMBC).  

A GMBC is an ordered list of data blocks representing a tamper-resistant chronological account of group membership updates.  The first block in the GMBC defines the initial set of group members and each subsequent block represents an addition/removal of one or more other entities to/from the group.  Any entity can create a GMBC, but only members can update it by appending new blocks.

Each block consists of a JSON object signed (as a JWS {{RFC7515}}) with the private key of the entity that created that block within the chain.  That JSON object includes attributes representing the following:

  * the acct URI {{RFC7565}} of the entity that created the block,
  * an array of group membership update operations, 
  * a timestamp indicating the date and time of the block's creation, and
  * a hash of the preceding block in the membership chain (if any).

A group membership update operation is a JSON object with two fields:

  * a tag indicating the operation type ("add" or "remove"), and
  * the acct URI of the entity being either added to or removed from the group.

In addition to the above attributes the first block of the chain, or genesis block, also includes the following attributes:

  * a URI that uniquely identifies the group communications resource, 
  * the acct URI {{RFC7565}} of the group's curator (optional), and
  * a nonce.

The genesis block must also include at least one "add" operation, though it need not necessarily represent the addition of the entity that created it (i.e. entities may create new groups within which they are not themselves members).

The membership of the group is implicit and may be determined by processing the GMBC in chronological order.  At any given point in time the membership of the group is defined as that set of entities for which there exists, for each entity, a previously introduced block containing an "add" operation for which there does not exist a subsequent but also previously introduced block containing a "remove" operation.

To protect against unauthorized tampering the GMBC is validated by verifying the signatures of each block, verifying that each non-genesis block contains a valid hash of the preceding block, and verifying that each block is created by an entity that is among the group's membership as determined by the segment of chain preceding that block.  Block signature verification is made possible through the key discovery mechanisms defined in {{I-D.miller-saag-key-discovery}} and the knowledge of each member's acct URI {{RFC7565}}.

## Key Distribution by Group Keys

A Group Key (GK) is signed object containing encrypted symmetric key material with associated metadata.  A GK exists for the purpose of sharing the contained symmetric key with other members of the group.  The exclusivity of access to the key material is achieved by encrypting the key material in such a way that it may be decrypted with the private entity key(s) of any one of the other group members.  The creating entity signs the GK such that the authenticity of the associated metadata may be verified by its recipients. 

More specifically, the payload of the GK is a JSON object including attributes representing the following:

  * a URI that uniquely identifies the GK,
  * the acct URI {{RFC7565}} of the creator of the GK,
  * a hash of the GMBC tail block at the time this key was created,
  * an encrypted {{RFC7517}} that represents the symmetric key material, and
  * a timestamp indicating the date and time the GK was created.

The JWK attribute value is encrypted in a JWE {{RFC7516}} JSON serialization with one or more recipients.  In decentralized groups the resulting JSON serialization must include each other member of the group as determined by the current and validated GMBC.  In centralized groups the resulting JSON serialization may include as a recipient just the curator (e.g. when an entity shares a new GK) or just one member (e.g. when the curator shares a GK with a member that has requested it).  The full JSON payload of the GK is signed as a JWS {{RFC7515}} using the creator's private entity key.

GKs may be created by members and non-members alike.  A non-member may generate a GK as described above and use it to encrypt its own communications to the group.  This can be a useful property as it provides for a confidential "write only" capability to the group communications resource.  

A group may have any number of GKs associated with it.  Where practical, it is recommended that each member of a group use its own GK for purposes of encryption and share this GK with the remainder of the group for purposes of decryption.  A member must not re-use the keying material of a GK created by another entity to encrypt it's own communications unless it has verified that GK is signed by a current member of the group as defined by the GMBC.

Upon receiving and validating an update to the GMBC, each entity must discard their encryption GK and produce a new encryption GK for which the recipients reflect the updated GMBC membership.  This is necessary to ensure that new members are able to decrypt subsequent communications but not prior communications.  Perhaps more importantly, this also ensures that former members are not able to decrypt subsequent group communications.  In centralized groups the curator may implement a policy where it permits new group members to request previously created GKs.

It is recommended that all entities that share encrypted content over the group communications resource rotate their GKs regularly so as to mitigate against vulnerabilities that are exacerbated by the extended use of individual keys.

# Group Models

While this specification provides definition for constructs that enable confidential group communications, the means by which these objects may be exchanged among group members is intentionally omitted as this is regarded as out of scope for this specification.  It remains worthwhile, however, to discuss two general classes of confidential group communications and how the primitives defined by this specification may be leveraged in each.  These classes may be described as "centralized groups" and "decentralized groups".

## Decentralized Groups

A decentralized group is characterized by the absence of a curator attribute in the GMBC genesis block and therefore the absence of a permanent member within the group through which GMBC and GK objects may be exchanged.  In a decentralized group these objects may instead be exchanged either in-band through the group communications resource itself, or through in-band references to external repositories within which GMBC and GK objects are stored.  Both the GMBC and GK objects are designed to be hardened against tampering and protect sensitive data such that they may be reasonably exchanged through either public or private transports and stores.

## Centralized Groups

A centralized group is characterized by the presence of a curator attribute in the GMBC genesis block.  The curator attribute identifies an entity by its acct URI {{RFC7565}} and declares that entity as a permanent member of the group which will serve as a facilitator for the exchange of GMBC and GK objects among all group members.  In particular, a curator will respond to the following types of requests from other entities.  Note that the means by which these operations are carried out between members and the curator is out of scope for this document.

GMBC Post

> When adding or removing members from a group, a member will submit a new GMBC block to the curator representing that change.  The curator will verify that the block is signed by a member of the group and that the hash attribute of the block represents the hash of the current tail end of the chain.  If both checks succeed then the curator will make the new block a permanent part of the GMBC and indicate to the requesting entity that the update was successful.

GMBC Get

> Entities may request all or part of the current GMBC from the curator by providing to the GMBC a hash of the last GMBC block of which they are aware (or 0x0 if they are requesting the entire chain).

GMBC Notify (optional)

> In some deployments it may be desirable for a curator to immediately multicast GMBC updates to all current members of the group.  This may be based on either an explicit or automatic/implicit subscription model.

GK Post

> Entities may inform the curator of new GKs which they have generated for the purpose of encrypting data they emit to the group communications resource.  The curator will store a received GKs such that it may subsequently service requests for that GK from other members that wish to decrypt these communications.

GK Get

> Members may request from the curator GKs which have been generated by other entities.  In doing so, an entity must indicate the URI of the requested GK and the curator must verify that the requesting entity was a member of the group at the time the GK was created by processing the GMBC from the genesis block up to and including the block whose hash is indicated in the metadata of the requested GK.  A successful confirmation of the requesting entity as a member of the group at that point in time will result in the curator generating a new GK which is in every way identical to the requested GK except that the key material is re-encrypted using the public key of the requesting entity and the GK itself is signed using the curator's own public entity key.

# Use Cases

The following are non-normative examples of how the primitives described in this specification may be employed to facilitate confidential group communications.

## Decentralized Group File Sharing 

This use case describes an application that utilizes some third party file sharing service to store confidential information and employs this specification as part of a scheme to secure that confidentiality among the members of self defined group.

1. User A generates a symmetric key to be used for file encryption.

2. User A encrypts a file using the symmetric key and posts it to the file sharing service.

3. User A generates a new GMBC by creating a genesis block.  In that block user A includes a reference to the URL of the encrypted file on the file sharing service.  User A also adds to the genesis block three group membership "add" operations: one for itself, one for user B, and one for user C.

4. User A creates a GK that includes a hash of the GMBC genesis block, and encrypts the key material portion of the GK using a multi-recipient JWE JSON serialization that indicates users B and C as recipients.

5. User A posts the GMBC and GK as text files to the same file sharing service, and sends the URLs of the encrypted content file, the GMBC, and GK to users B and C.

6. User B recognizes user A's acct URI as the identity of a trusted correspondent, retrieves the GMBC and GK from the file service, and verifies the signatures on the GMBC genesis block and GK by discovering and retrieving user A's public entity key through {{I-D.miller-saag-key-discovery}}.

7. User B uses its own private entity key to decrypt the key material contained within the GK, downloads the file indicated by the resource URL in the GMBC genesis block, and decrypts the contents of that file.

8. User C repeats the procedure outlined for user B in steps 6 and 7.

## Centralized Group Instant Messaging

This use case describes an application that utilizes some instant messaging service to exchange confidential messages among a group of users and employs this specification as part of a scheme to secure that confidentiality among the members as a centralized group.

1. User A establishes a messaging thread on the messaging service that includes users B and C.  We presume it can be associated with some unique URI for purposes of identification.

2. User A generates a new GMBC by creating a genesis block.  In that block user A includes a reference to the URI of the messaging thread created in step 2, and three group membership "add" operations: one for itself, one for user B, and one for user C.  User A also identifies itself as the curator of the group by provisioning the genesis block with its own acct URI in the curator field.

3. User A creates a GK that includes a hash of the GMBC genesis block.

4. User A encrypts a message using the keying material of the GK and sends it over the instant messaging service.  As metadata within that message it also includes the URI of the GK it used to encrypt the message and the current GMBC.

5. User B recognizes user A's acct URI as the identity of a trusted correspondent and validates the GMBC as originating from user A by discovering and retrieving user A's public entity key through {{I-D.miller-saag-key-discovery}}.

6. User B observes that the GMBC indicates user A as the curator for this group and sends a request (perhaps as an in-band extension to the instant messaging protocol) to user A for the GK used to encrypt the message sent in step 3.

7. User A receives the GK request from user B, validates that user B was a member of the GMBC at the time the requested GK was created, and returns a copy of the GK created in step 3 with the keying material portion encrypted using the public entity key of user B.

8. User C repeats the procedure outlined for user B in steps 4 through 6.

# Primitive Specifications

This section provides normative definition for the objects introduced in this document.

## Entity

An entity is identified by an acct URI {{RFC7565}}, its associated public entity key is discovered through key discovery as described in {{I-D.miller-saag-key-discovery}}, and that key is represented as a JWK {{RFC7517}} also as defined in that specification.

## Group Membership Block Chain

A GMBC is composed of JSON encoded blocks, each signed with the private key of the entity that introduced that block to the chain.  Signing is performed in conformance with the JWS {{RFC7515}} specification and the block is communicated between entities in the form of a JWS compact serialization.

The payload of a standard GMBC block is defined as follows, using JSON content rules notation {{I-D.newton-json-content-rules}}.

~~~
operation "Operation" {
    "entity": uri,                ; acct URI of the entity added/removed
    "optype": < "add" "remove" >  ; tag indicating the type of operation
}

gmbc-block {
    "creator": uri,               ; acct URI of the creator of the block
    "created": date-time,         ; the date and time of block creation
    "antecedent": string,         ; SHA-256 hash of the preceeding block
    "operations" [ *: operation ] ; membership update operations
}

root gmbc-block
~~~

A GMBC genesis block has the same specification as a standard block but with three additional payload fields, as defined below.

~~~
gmbc-genesis-block {
    "resource": uri,              ; URI of the group comms. resource
    "curator": ?uri,              ; (optional) acct URI of the curator
    "nonce": integer,             ; a random one-time numeric value
    gmbc-block                    ; include standard block attributes
}

root gmbc-genesis-block
~~~

## Group Key

A GK is composed of JSON encoded blocks, each signed with the private key of the entity that created it (or the curator when servicing GK requests in centralized groups).  Signing is performed in conformance with the JWS {{RFC7515}} specification and the block is communicated between entities in the form of a JWS compact serialization.

The payload of a GK is defined as follows.

~~~
group-key {
    "uri": uri,                   ; URI to identify the GK itself
    "creator": uri,               ; acct URI of the creator of the GK
    "created": date-time,         ; the date and time of GK creation
    "block": string,              ; SHA-256 hash of assoc. GMBC block
    "key": wrapped-key            ; encrypted symmetric key material
}

root group-key
~~~

The "key" attribute's value is a JWE JSON serialization as defined in {{RFC7516}} with one or more recipients (either one recipient for each member of the group at the time the key was created, or the group curator).  The payload of that JWE is a JWK {{RFC7517}} representing a symmetric key.

# Security Considerations

Security considerations are discussed throughout this document.  Additional considerations may be added here as needed.

# Appendix A. Acknowledgments

This specification is the work of several contributors. In particular, the following individuals contributed ideas, feedback, and wording that influenced this specification:

Matt Miller

# Appendix B. Document History

\-00

  * Initial draft.

---
v: 3

title: "Trustworthy Workload Identities WIMSE Profile for Replica Workloads"
abbrev: "TWI WIMSE-CREDS Profile"
docname: draft-novak-rats-wimse-creds-twi-profile-latest
category: std
consensus: true
submissionType: IETF

ipr: trust200902
area: "Security"
workgroup: "Remote ATtestation ProcedureS"
keyword: [ evidence, attestation results, endorsements, reference values ]

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 - name: Mark Novak
   org: J.P. Morgan Chase
   email: mark.f.novak@jpmchase.com
 - name: Henk Birkholz
   organization: Fraunhofer SIT
   email: henk.birkholz@ietf.contact

normative:
  RFC9334: rats-arch
  I-D.draft-ietf-wimse-arch: wimse-arch
  I-D.draft-ietf-wimse-workload-creds: wimse-creds
  TWISIGCharter:
    -: TWISIGCharter
    target: https://github.com/confidential-computing/governance/blob/main/SIGs/TWI/TWI_Charter.md
    title: Trustworthy Workload Identity (TWI) Special Interest Group — Charter
    author:
      org: Confidential Computing Consortium Trustworthy Workload Identity SIG
  TWISIGReq:
    -: TWISIGReq
    target: https://github.com/confidential-computing/twi/blob/main/TWI_Requirements.md
    title: Trustworthy Workload Identity (TWI) Special Interest Group — Requirements
    author:
      org: Confidential Computing Consortium Trustworthy Workload Identity SIG
  BCP26:
    -: ianacons
    =: RFC8126

informative:

  key-management: DOI.10.6028/NIST.SP.800-57pt2r1

entity:
  SELF: "RFCthis"

--- abstract

The focus of this document is "replica workloads"--meaning workloads that are functionally indistinguishable from the point of view of their clients and their relying parties.
Such workloads share code, security-sensitive configuration and other compliance-relevant attributes, such as hardware location.
This document specifies how replica workloads running inside Trusted Execution Environments can obtain WIMSE-compatible credentials (WIC) in a manner that satisfies the requirements of Trustworthy Workload Identities and fits both within the IETF WIMSE and RATS architectures.

--- middle

# Introduction

The WIMSE Reference Architecture {{-wimse-arch}} allows workloads to request credentials from an agent.
The agent runs on a node and generates RATS evidence {{-rats-arch}} that includes credentials for all workloads running on this node.
The evidence is conveyed to a WIMSE Server to obtain the node's crendentails in a trustworthy and authentic manner.

This procedure can be a challenge for confidential workloads (workloads running inside Trusted Execution Environments (TEE)).
Confidential workloads do not necessarily trust the machines on which they are extectued and running.
Hence, a mechanism is required by which confidential workloads can obtain WIMSE credentials contingent on successfully performing remote attestation to a Verifier, while at the same time being compatible with the WIMSE architechture {{-wimse-arch}} and RATS architectures {{-rats-arch}}.

This document specifies such a mechanism.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The reader is assumed to be familiar with the vocabulary and concepts
defined in {{-wimse-arch}} and {{-rats-arch}}.

# Actors and Services

The proposal is composed of interactions between the following actors and services:

Workload:

: One of a collection of identical (in terms of code and configuration) instances of software running inside Trusted Execution Environments

Workload Owner:

: The entity that owns the Workload and decides which identity it is to have

Key Store:

: A Hardware Security Module (HSM) or equivalent trusted service generates certain cryptographic keys and keeps them safe from disclosure and tampering while allowing certain operations to be performed using these keys

Credential Authority:

: A trusted issuer of Workload Credentials

Verifier:

: A trusted instance of a RATS Verifier used in Remote Attestation of Workloads

# Cryptographic Keys

This proposal uses the following three types of keys:

1. Credential Signing Key (CSK): the asymmetric signing key corresponding to the Workload Credential.
The public portion is part of the Credential itself, and the private portion is used for proof-of-possession of the Credential.
The private portion is only given to authorized Workloads and is known exclusively by these Workloads and the Key Store where it is generated.

2. CSK Wrapping Key (CWK): the symmetric encryption key used to encrypt the CSK. The Workload obtains the CWK contingent on successfully passing remote attestation. The CWK is generated inside the Key Store just like the CSK, and is uniquely bound to it so that no two CSKs can have the same CWK.

3. CWK Delivery Key (CDK): the asymmetric encryption key generated and attested by each Workload instance. Used to securely deliver the CWK to each authorized Workload.

Under the proposal outlined below, the Workload obtains the Workload Credential (e.g., either an X.509 certificate or a Workload Identity Token (WIT)), as well as the encrypted CSK from the Control Plane, and the CWK from a Key Store after performing successful Remote Attestation.
The Workload then uses the CWK to decrypt (unwrap) the private CSK inside its TEE.

# Credential Acquisition Mechanism

## Two Phases of the Credential Acquisition Mechanism

The proposed credential acquisition mechanism is split into two phases: the first performed by the Control Plane, and the second by the Workload itself.

The first phase, called the Provisioning phase, is performed by the Control Plane (i.e., untrusted entities outside of the Workload itself).
Inside a secure key store, the Control Plane generates 1) a CSK, and 2) a corresponding CWK--without learning either.
The Control Plane then invokes the Credential Authority to generate the associated public credential containing the public key of the CSK pair, and it uses the key store to generate the CWK and wrap (encrypt) the private key of the CSK pair with the CWK.
It then makes both the public credential and the wrapped CSK available to the workload instance.

The second phase is the Acquisition phase. In this phase, the Workload instance launches, performs remote attestation, and uses the Attestation Results from the Verifier to obtain the CWK key from the key store.
With the CWK key in its possession, the Workload instance can decrypt the wrapped private key of the CSK pair, which enables it to use the Credential.

### Provisioning Phase

~~~~ aasvg
+----------------+ +-----------+ +----------------------+ +----------+
| Workload Owner | | Key Store | | Credential Authority | | Workload |
+--+-------------+ +---------+-+ +-------------------+--+ +----+-----+
   |                         |                       |         |
 +-+-------------------------+-----------------------+-+       |
 |          Credential Provisioning Phase              |       |
 +-+-------------------------+-----------------------+-+       |
   | +---------------+       |                       |         |
   | | Invoke Claims |       |                       |         |
   | | Mapper        |       |                       |         |
   | +---------------+       |                       |         |
   | Create Workload         |                       |         |
   | Identity                |                       |         |
   +--(1)-------+            |                       |         |
   +<-----------+            |                       |         |
   | Precompute RATS AR      |                       |         |
   +--(2)-------+            |                       |         |
   +<-----------+            |                       |         |
   |                         |                       |         |
   | Establish authen-       |                       |         |
   | ticated secure channel  |                       |         |
   +<-(3)------------------->+                       |         |
   | Provision key and       |                       |         |
   | request CSR (WIC + AR)  |                       |         |
   +--(4)------------------->+                       |         |
   |                         | CWK Gen and set CWK   |         |
   |                         | access policy to AR   |         |
   |                         +--(5)-------+          |         |
   |                         +<-----------+          |         |
   |                         | CSK Gen and Store     |         |
   |                         +--(6)-------+          |         |
   |                         +<-----------+          |         |
   |                         | Encrypt privCSK       |         |
   |                         | with CWK              |         |
   |                         +--(7)-------+          |         |
   |                         +<-----------+          |         |
   |                         | CSR Gen with privCSK  |         |
   |                         | based on WIC + pubCSK |         |
   |                         +--(8)-------+          |         |
   |                         +<-----------+          |         |
   |                         |                       |         |
   | Return CSR and          |                       |         |
   | wrapped privCSK         |                       |         |
   +<--(9)-------------------+                       |         |
   |                         |                       |         |
   | Credential Request      |                       |         |
   +--(10)-------------------+---------------------->+         |
   |                         |          Validate CSR |         |
   |                         |           +-----(11)--+         |
   |                         |           +---------->+         |
   |                         |        Credential Gen |         |
   |                         |           +-----(12)--+         |
   |                         |           +---------->+         |
   |                         |     Return Credential |         |
   +<------------------------+-----------------(13)--+         |
   |                         |                       |         |
   | Provision Credential    |                       |         |
   | and wrapped CSK         |                       |         |
   +--(14)-------------------+-----------------------+-------->+
   |                         |                       |         |
~~~~

| Step | Notes                                                       |
|-----:|:------------------------------------------------------------|
|    1 | The Workload Owner computes the intended Workload Identity. This Identity can be expressed as an X.509 subject alternative name, a WIMSE SVID, etc. The Identity MUST include all attributes of the Workload that the Relying  Party would need to consider in its authorization policy. For example, if the Relying Party cares about the country where the Workload would be executing, that information  must be reflected in the Identity. The mechanism by which the Workload Owner decides on which identity to assign to the Workload is outside of scope of this document. |
|------|-------------------------------------------------------------|
|    2 | The Workload Owner precomputes the Attestation Results that the Workload would present to the Key Store in exchange for the CWK during the Acquisition phase. The mechanism for doing so is outside the scope of this document. |
|------|-------------------------------------------------------------|
|    3 | The Workload Owner and the Key Store establish a mutually authenticated secure channel. |
|------|-------------------------------------------------------------|
|    4 | The Workload Owner sends the Workload Identity from Step 1 and the Attestation Results from Step 2 to the Key Store. It expects to receive back from the Key Store (in Step 10, below) 1) a wrapped CWK and 2) a Credential Signing Request (CSR) that it will subsequently use to request the Workload Credential from the Credential Authority. |
|------|-------------------------------------------------------------|
|    5 | The Key Store generates and stores a new CWK, and sets the CWK access policy to the value of Attestation Results it has received in Step 4. For security reasons explained in the Threat Model section in the Security Considerations section later in the document, the CWK access policy MUST NOT be changed once set and MUST NOT allow retrieving or modifying the CWK. |
|------|-------------------------------------------------------------|
|    6 | The Key Store generates and stores the CSK Pair. The CSK pair has a one-to-one relationship with the CWK that lasts for the lifetime of both keys. The CSK Pair is stored in case the Workload Owner might subsequently ask for the Credential to be renewed without changing the CSK. |
|------|-------------------------------------------------------------|
|    7 | The Key Store encrypts the Private CSK with the CWK.        |
|------|-------------------------------------------------------------|
|    8 | The Key Store uses the Workload Identity supplied in Step 4 and the CSK Pair generated in Step 6 to create a Credential Signing Request (CSR) and signs it with the Private CSK. |
|------|-------------------------------------------------------------|
|    9 | The Key Store returns to the Workload Owner the wrapped Private CSK computed in Step 7 and the Credential Signing Request generated in Step 8. |
|------|-------------------------------------------------------------|
|   10 | The Workload Owner forwards the CSR received from the Key Store in Step 9 to the Credential Authority to request the Workload Credential. |
|------|-------------------------------------------------------------|
|   11 | The Credential Authority validates that the CSR has originated from the appropriate Workload Owner and/or Key Store (details in the threat model below). |
|------|-------------------------------------------------------------|
|   12 | The Credential Authority generates and signs (with its own CA issuer signing key, trusted by the Relying Party) the Workload Credential based on the CSR it has received in Step 10. |
|------|-------------------------------------------------------------|
|   13 | The Credential Authority returns the Workload Credential to the Workload Owner. |
|------|-------------------------------------------------------------|
|   14 | The Workload Owner makes the wrapped Private CSK and the Workload Credential available to the Workload. |
{: #tbl-provisioning title="Provisioning Phase"}

### Acquisiton Phase

Diagram TBD


| Step | Notes                                                       |
|-----:|:------------------------------------------------------------|
|    1 | The Workload Owner supplies the Workload with the plaintext Workload Credential and the wrapped Private CSK. This is the same as Step 14 in the previous diagram. |
|------|-------------------------------------------------------------|
|    2 | Inside its TEE, the Workload generates a CDK asymmetric encryption key pair. |
|------|-------------------------------------------------------------|
|    3 | The Workload generates Evidence about itself and its hosting environment for Remote Attestation. The Evidence MUST include the public portion of the CDK generated in Step 2. |
|------|-------------------------------------------------------------|
|    4 | The Workload contacts the Verifier and presents it with Evidence generated in Step 3. |
|------|-------------------------------------------------------------|
|  5/6 | The Verifier appraises the Evidence received from the Workload in Step 4 and computes Attestation Results based on this Evidence. |
|------|-------------------------------------------------------------|
|    7 | The Verifier returns Attestation Results from Step 6 back to the Workload. The Attestation Results MUST contain the public portion of the CDK from the Evidence. |
|------|-------------------------------------------------------------|
|    8 | The Workload forwards the Attestation Results received from the Verifier in Step 7 to the Key Store, requesting the unwrapped CSK. |
|------|-------------------------------------------------------------|
|    9 | The Key Store validates the Attestation Results against the CWK access policy. |
|------|-------------------------------------------------------------|
|   10 | The Key Store encrypts the CWK with the Public CDK in the Attestation Results. |
|------|-------------------------------------------------------------|
|   11 | The Key Store returns the encrypted CWK to the Workload.    |
|------|-------------------------------------------------------------|
|   12 | The Workload decrypts the CWK with its Private CDK.         |
|------|-------------------------------------------------------------|
|   13 | The Workload decrypts the wrapped Private CSK with the CWK. It is now in possession of both the plaintext Credential (supplied in Step 1) and the Private CSK needed to use it with proof-of-possession (decrypted in Step 12). |
{: #tbl-acquisition title="Acquisition Phase"}

## Discussion

### Fitness for Purpose

The mechanism just described is particularly well suited to “horizontally scaling” and “serverless” Workloads.
The defining characteristic of these Workloads (typically, but not exclusively, Docker containers and Lambda functions) is that they are likely to manifest as large numbers of identical, and frequently short-lived, instances.
This makes it optimal for Workloads that reuse credentials and credential keys.
Longer lived Workloads that may want to generate their own individual-to-each-Workload signing keys and obtain Workload Credentials for those are outside the scope of this document.

### Deployment Considerations

#### Flexible Runtime Integrity Measurements

While bigger than the scope of this document, this warrants mentioning: it is common for Evidence generated during Workload Remote Attestation to change without materially affecting the security posture of the Workload.
That can happen, for instance, when the Cloud Service Provider upgrades the firmware of the hardware on which the Workload runs, or when a minor (non-security-sensitive) code change is made to the Workload.
In either case, the mapping of Evidence to Attestation Results SHOULD be tolerant of the change and not cause Remote Attestation to fail.
How such flexibility will be achieved is still TBD and needs to be decided by the Attestation SIG at the CCC and the RATS WG at the IETF.

#### Credential Partitioning/Sharding

A break-one-break-all threat is inherent in this proposal: if any Workload instance (or the Workload’s Key Store) leaks the private CSK or the CWK, all its peers get compromised.
One way of mitigating this threat is to utilize different signing keys for groups of Workloads sharing the same Identity.
For instance, having a separate Key Store and thus separate CSK/CWK for Workload instances running in different Availability zones in the same cloud region would be an example of such a mitigation.

### Performance Optimizations

Many Confidential Computing platforms provide a “key derivation” ABI that enables a running Workload to derive encryption keys for its own use (that functionality is sometimes referred to as “sealing to the platform”).
This encryption key, if used correctly, i.e., meaning that it utilizes both the Workload’s code and its security-sensitive configuration, can allow a Workload to store the Private CSK securely across a restart or power outage.
A new Workload Instance could re-derive the same encryption key and unseal the previously sealed Private CSK without having to contact a Verifier.

### Credential Rotations

Credentials are issued for a period of time: that could mean a fixed time period, a number of uses or until a notable change in the threat environment occurs (see cryptoperiod {{key-management}}]).
For a Credential expiration to be extended without changing the underlying CSK, the Workload Owner can obtain a refreshed Credential from the CA after requesting a new CSR from the Key Store.
For that the Key Store has to store the Private CSK.

# Security Considerations

## CWK Acces Policy

It would be dangerous for any outside party to directly obtain a plaintext CWK from the Key Store as the CWK encrypts the Private CSK.
Once generated, the CWK is accessed only indirectly and in only the following circumstances:

1. By the Key Store itself in order to wrap the Private CSK (in Step 7 of the Provisioning phase)
1. CSK/CWK deletion by the Workload Owner
1. By the Workload in order to retrieve a CWK encrypted specifically for it using its Public CDK (in Steps 10 and 11 of the Acquisition Phase)

If the plaintext CWK leaks due to, e.g., a Key Store vulnerability, that immediately leads to the compromise of the corresponding Private CSK (see the following section).

## CSK Compromise Threats

In this proposal all Workloads matching the CWK access policy share the same Private CSK and thus can utilize the same Workload Credential.
This means that if any Workload in this authorized set is compromised, or if the Key Store releases the CWK to an authorized party, the corresponding Credential MUST be revoked and reissued.
Standard rules about limiting the lifetime of a Credential by periodically rotating the CSK apply. Another potential mitigation would be limiting how many Workload instances can share the same CSK.

## CSK-CWK One-to-One Coupling

Each CWK protects exactly one Private CSK. If the CSK is updated, it is likely due to the need to revoke the previous CSK, and if the same CWK were to be used to wrap the new Private CSK, that would enable continued unauthorized use of those older CSKs.
Therefore, for each newly generated CSK there has to exist a different CWK.

## Threat Model

The following best security practices are implied in the analysis below and are not called out in the individual protocol step analysis. Among other things this means that:

* Proper cryptography hygiene is maintained:
    * All sources of randomness used to generate cryptographic keys are of high quality
    * All cryptographic keys are generated with appropriate strength in terms of key size
    * Correct cryptographic algorithms and modes are used with those cryptographic keys, and that the latest known best practices are used when developing all code.
* Appropriate and up-to-date code development practices are followed
* All deployed software matches expectations and follows best “chain of custody” rules from the point of creation to deployment and throughout the software’s deployed lifetime
* All services listed (Key Store, Credential Authority, Verifier) are in the TCB of the Workload and thus must properly safeguard their code and assets in accordance with appropriate guidelines.
* Consider running all sensitive code inside a TEE wherever possible.

The point-to-point message exchanges in this specification do not require the use of transport level security or authentication between the parties, unless noted otherwise.

All identified threats are represented by their STRIDE abbreviations, as follows:

* S: Spoofing
* T: Tampering
* R: Repudiation
* ID: Information Disclosure
* DoS: Denial of Service
* EoP: Elevation of Privilege

## Provisioning Phase Threats and Mitigations

| Step | Type | Threat                  | Mitigation               |
|-----:|:-----|:------------------------|:-------------------------|
|  1/2 | S    | The Workload Owner and/or Identity generation logic are tricked into assigning the wrong Identity or Attestation Results to the future Workload. Generated Identity and Attestation Results are mismatched. | Safeguard the Identity and Attestation Results generation logic and policies. Generate Identity and Attestation Results as a pair rather than separately. Log and verify all generated outputs.  |
|      | T    |  |  |
|      | EoP  |  |  |
|------|------|-------------------------|--------------------------|
|    3 | Implement standard practices for securing communications  |
|      | between two entities (mutual authentication,              |
|      | data-in-transit protection)                               |
|------|------|-------------------------+--------------------------|
|    4 | S    | An unauthorized entity  | The Key Store MUST       |
|      | EoP  | places a request to the | authenticate and         |
|      |      | Key Store resulting in  | authorize the Workload   |
|      |      | Identity and            | Owner in Step 3 to       |
|      |      | Attestation Results of  | ensure that only         |
|      |      | an attacker's choosing  | authorized entities are  |
|      |      | being provisioned to    | allowed to perform this  |
|      |      | Workloads.              | operation on the         |
|      |      |                         | Workload Identity in     |
|      |      |                         | question.                |
|      |------|-------------------------|--------------------------|
|      | T    | MITM intercepts and     |                          |
|      | EoP  | changes the Workload    |                          |
|      |      | Identity communicated   |                          |
|      |      | to the Key Store,       |                          |
|      |      | leading to Credentials  |                          |
|      |      | being issued for the    |                          |
|      |      | wrong Identity.         |                          |
|      |      | MITM intercepts and     |                          |
|      |      | changes the Attestation |                          |
|      |      | Results communicated to |                          |
|      |      | the Key Store, leading  |                          |
|      |      | to the CWK access       |                          |
|      |      | policy being incorrect  |                          |
|      |      | and resulting in        |                          |
|      |      | Credentials being       |                          |
|      |      | issued to vulnerable or |                          |
|      |      | malicious Workloads.    |                          |
|      |------|-------------------------|--------------------------|
|      | ID   | Attestation Results     |                          |
|      |      | might leak and provide  |                          |
|      |      | vulnerability           |                          |
|      |      | intelligence to an      |                          |
|      |      | attacker.               |                          |
|      |------|-------------------------|--------------------------|
|      | S    | The Workload Owner      |                          |
|      |      | talks to the wrong Key  |                          |
|      |      | Store which ends up     |                          |
|      |      | being malicious.        |                          |
|------|------|-------------------------|--------------------------|
|    5 | ID   | CWK access policy is    | The Key Store is         |
|      | EoP  | stored in a manner that | expected to implement    |
|      |      | allows subsequent       | this functionality       |
|      |      | tampering, or retrieval | correctly. The generated |
|      |      | of the plaintext CWK.   | Access Policy SHOULD be  |
|      |      |                         | logged and audited.      |
|------|------|-------------------------|--------------------------|
|    6 |      | CSK Pair is stored by   |                          |
|      |      | the Key Store in a      |                          |
|      |      | manner that allows      |                          |
|      |      | unauthorized retrieval  |                          |
|      |      | or access.              |                          |
|------|------+-------------------------+--------------------------|
|    7 | N/A                                                       |
|    8 |                                                           |
|------|------+-------------------------+--------------------------|
|    9 | DoS  | The Credential Request  | These are not security-  |
|   10 | T    | and/or the wrapped CSK  | sensitive: 1) the        |
|      |      | may end up in the wrong | Private CSK is wrapped   |
|      |      | hands, or delivery may  | and requires the         |
|      |      | be prevented, or        | Workload to obtain the   |
|      |      | contents might be       | CWK, which means it has  |
|      |      | modified in transit.    | to be the authorized     |
|      |      |                         | Workload. 2) The CSR can |
|      |      |                         | only be used to obtain a |
|      |      |                         | Credential, which itself |
|      |      |                         | cannot be used without   |
|      |      |                         | the Private CSK.         |
|      |      |                         | Tampering with either    |
|      |      |                         | would render them        |
|      |      |                         | useless.                 |
|------|------|-------------------------|--------------------------|
|   10 | S    | A CSR from an           | Credential Authority     |
|   11 |      | unauthorized Workload   | MUST do either or both   |
|      |      | Owner or Key Store is   | of the following:        |
|      |      | sent to the Credential  | 1. Authenticate the      |
|      |      | Authority, resulting in | Workload Owner and       |
|      |      | the Credential          | establish a secure       |
|      |      | Authority issuing a     | channel with it before   |
|      |      | Credential to the wrong | trusting the CSR.        |
|      |      | Identity and/or signing | 2. Authenticate the Key  |
|      |      | key.                    | Store by having it       |
|      |      |                         | counter-sign the CSR     |
|      |      |                         | with its own private     |
|      |      |                         | signing key.             |
|------|------+-------------------------+--------------------------|
|   12 | N/A                                                       |
|------|------+-------------------------+--------------------------|
|   13 | DoS  | Sending the Workload    | Both Workload Credential |
|   14 | T    | Credential or the       | and wrapped Private CSK  |
|      |      | wrapped Private CSK to  | are safe to disclose,    |
|      |      | the wrong Workload or   | and Credential cannot be |
|      |      | preventing delivery, or | used without unwrapping  |
|      |      | contents might be       | the Private CSK first.   |
|      |      | modified in transit.    | Tampering with either    |
|      |      |                         | would render them        |
|      |      |                         | useless.                 |
{: #tbl-prov-threats title="Provisioning Phase Threats and Mitigations"}

## Acquisition Phase Threats and Mitigations

| Step | Type | Threat                  | Mitigation               |
|-----:|:-----|:------------------------|:-------------------------|
|    1 | DoS  | Identical to Step 14    |                          |
|      | T    | under Provisioning      |                          |
|------|------|-------------------------|--------------------------|
|    2 | N/A  |                         |                          |
|    3 |      |                         |                          |
|------|------|-------------------------|--------------------------|
|    4 | DoS  | Attestation requests    | Attestation flows are    |
|      | T    | might be prevented from | assumed to be designed   |
|      |      | being delivered or      | to be tamper-resistant.  |
|      |      | tampered with in        |                          |
|      |      | transit.                |                          |
|      |------|-------------------------|--------------------------|
|      | ID   | Attestation request     | Workload SHOULD          |
|      |      | sent to the wrong       | authenticate the         |
|      |      | Verifier or over an     | Verifier and SHOULD      |
|      |      | unencrypted channel     | establish a secure       |
|      |      | discloses information   | transport channel with   |
|      |      | about Workload's        | the Verifier prior to    |
|      |      | composition to an       | sharing the Attestation  |
|      |      | unauthorized party.     | Results.                 |
|------|------|-------------------------|--------------------------|
|    5 | EoP  | Wrong appraisal policy  | Secure the Appraisal     |
|    6 |      | for Evidence on         | Policy for Evidence and  |
|      |      | Verifier leads to       | Reference Values. Log    |
|      |      | incorrect appraisal     | and audit all requests   |
|      |      | decision.               | and responses.           |
|------|------|-------------------------|--------------------------|
|    7 | ID   | Lack of secure channel  | Utilizing the same       |
|      |      | between the Workload    | secure channel           |
|      |      | and the Verifier may    | established in Step 4    |
|      |      | lead to leaking the     | will mitigate this       |
|      |      | Attestation Results to  | threat.                  |
|      |      | unauthorized parties.   |                          |
|------|------|-------------------------|--------------------------|
|    7 | DoS  | Attestation Results     | Flows involving          |
|    8 | T    | might be prevented from | Attestation Results are  |
|      |      | being delivered or      | assumed to be designed   |
|      |      | tampered with in        | to be tamper-resistant.  |
|      |      | transit. Same applies   |                          |
|      |      | to the request for the  |                          |
|      |      | CWK sent to the Key     |                          |
|      |      | Store.                  |                          |
|------|------|-------------------------|--------------------------|
|    8 | EoP  | There is a              | Verifier signs or        |
|      |      | vulnerability in the    | timestamps the           |
|      |      | Verifier that causes it | Attestation Results and  |
|      |      | to produce wrong        | the Key Store validates  |
|      |      | Attestation Results.    | this before releasing    |
|      |      | The Workload caches     | the CWK. Alternatively,  |
|      |      | these Attestation       | the Key Store engages in |
|      |      | Results and             | a proof-of-liveness      |
|      |      | accidentally or         | exchange with the        |
|      |      | deliberately causes the | Verifier.                |
|      |      | Key Store to            |                          |
|      |      | incorrectly release the |                          |
|      |      | CWK.                    |                          |
|------|------|-------------------------|--------------------------|
|    9 | N/A  | Incorrect decision made | Security at this step    |
|      |      | on whether to allow the | assumes that the correct |
|      |      | caller to retrieve the  | CWK access policy has    |
|      |      | CWK.                    | been set during the      |
|      |      |                         | Provisioning flow.       |
|------|------+-------------------------+--------------------------|
|   10 | N/A                                                       |
|------|------+-------------------------+--------------------------|
|   11 | DoS  | Encrypted CWK might be  | Tampered CWK is          |
|      | T    | tampered with in        | unusable.                |
|      |      | transit or prevented    |                          |
|      |      | from being delivered.   |                          |
|------|------+-------------------------+--------------------------|
|   12 | N/A                                                       |
|   13 |                                                           |
{: #tbl-acq-threats title="Acquisition Phase Threats and Mitigations"}

--- back

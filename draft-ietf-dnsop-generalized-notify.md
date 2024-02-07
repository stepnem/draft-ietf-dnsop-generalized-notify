---
title: Generalized DNS Notifications
abbrev: Generalized Notifications
docname: draft-ietf-dnsop-generalized-notify-latest
date: {DATE}
category: std

ipr: trust200902
area: Internet
workgroup: DNSOP Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: J. Stenstam
    name: Johan Stenstam
    organization: The Swedish Internet Foundation
    email: johan.stenstam@internetstiftelsen.se
 -
    ins: P. Thomassen
    name: Peter Thomassen
    org: deSEC, Secure Systems Engineering
    email: peter@desec.io
 -
    ins: J. Levine
    name: John Levine
    org: Standcore LLC
    email: standards@standcore.com

normative:

informative:

--- abstract

This document extends the use of DNS NOTIFY ({{!RFC1996}} beyond conventional
zone transfer hints, bringing the benefits of ad-hoc notifications to DNS
delegation maintenance in general.  Use cases include DNSSEC key rollovers
hints, and quicker changes to a delegation's NS record set.

To enable this functionality, a method for discovering the receiver endpoint
for such notification message is introduced, via the new NOTIFY record type.

TO BE REMOVED: This document is being collaborated on in Github at:
[https://github.com/peterthomassen/draft-ietf-dnsop-generalized-notify](https://github.com/peterthomassen/draft-ietf-dnsop-generalized-notify).
The most recent working version of the document, open issues, etc. should all be
available there.  The authors (gratefully) accept pull requests.

--- middle

# Introduction

Traditional DNS notifications {{!RFC1996}}, which are here referred to as
"NOTIFY(SOA)", are sent from a primary server to a secondary server to
minimize the latter's convergence time to a new version of the
zone. This mechanism successfully addresses a significant inefficiency
in the original protocol.

Today similar inefficiencies occur in new use cases, in particular delegation
maintenance (DS and NS record updates). Just as in the NOTIFY(SOA) case, a new
set of notification types will have a major positive benefit by
allowing the DNS infrastructure to completely sidestep these
inefficiencies. For additional context, see {{context}}.

No DNS protocol changes are introduced by this document. The mechanism
instead makes use of a wider range of DNS messages allowed by the protocol.
Future extension for further use cases (such as multi-signer key exchange)
is possible.

Readers are expected to be familiar with DNSSEC, including {{!RFC4033}},
{{!RFC4034}}, {{!RFC4035}}, {{!RFC6781}}, {{!RFC7344}}, {{!RFC7477}},
{{!RFC7583}}, and {{!RFC8901}}.

## Requirements Notation

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**", "**SHALL
NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**", "**NOT
RECOMMENDED**", "**MAY**", and "**OPTIONAL**" in this document are to be
interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}} when, and only
when, they appear in all capitals, as shown here.

## Terminology

In the text below there are two different uses of the term
"NOTIFY". One refers to the NOTIFY message, sent from a DNSSEC signer
or name server to a notification target (for subsequent
processing). We refer to this message as NOTIFY(RRtype) where the
RRtype indicates the type of NOTIFY message (CDS or CSYNC).

The second is a proposed new DNS record type, with the suggested
mnemonic "NOTIFY". This record is used to publish the location of the
notification target. We refer to this as the "NOTIFY record".

# CDS/CDNSKEY and CSYNC Notifications

The {{!RFC1996}} NOTIFY message sends a SOA record in the Query
Section. We refer to this as a NOTIFY(SOA).
By generalizing the concept of DNS NOTIFY it is possible to address
not only the original inefficiency (primary name server to secondary
nameserver convergence) but also the problems with CDS/CDNSKEY and CSYNC
scanning for zone updates {{!RFC7344}} {{!RFC7477}}.

The CDS/CDNSKEY inefficiency may be addressed by the child sending a
NOTIFY(CDS) to an address where the parent listens for such notifications.

While the receiving side will often be a scanning service provided by
the registry itself, it is expected that in the ICANN RRR model, some
registries will prefer registrars to conduct CDS/CDNSKEY scans.
For such parents, the receiving service should notify the appropriate
registrar, over any communication channel deemed suitable between
registry and registrar (such as EPP, or possibly by forwarding the
actual NOTIFY(CDS) or NOTIFY(CSYNC) directly).
Such internal processing is inconsequential from the perspective of
the child: the NOTIFY packet is simply sent to the notification address.

To address the CDS/CDNSKEY dichotomy, NOTIFY(CDS) is defined to indicate
any child-side changes pertaining to a upcoming update of DS records.
Upon receipt of NOTIFY(CDS), the parent SHOULD initiate the same scan
that would otherwise be triggered based on a timer.

The CSYNC inefficiency may similarly be addressed by the child sending a
NOTIFY(CSYNC) to an address where the parent is listening to CSYNC
notifications.

In both cases the notification will speed up the CDS and CSYNC
scanning by providing the parent with a hint that a particular child
zone has published new CDS, CDNSKEY and/or CSYNC records.

The DNS protocol already allows these new notification types, so no
protocol change is required. The only thing needed is specification of
where to send the new types of Notifies and how to interpret them in
the receiving end.

## Where to send CDS and CSYNC Notifications

In the case of NOTIFY(CDS) and NOTIFY(CSYNC) the ultimate recipient of
the notification is the parent, to improve the speed and efficiency of
the parent's CDS/CSYNC scanning. Because the name of the parent zone
is known, and because the parent obviously controls the contents of
its own zone the simplest solution is for the parent to publish the
address where it prefers to have notifications sent.

However, there may exist cases where this scheme (sending the
notification to the parent) is not sufficient and a more general
design is needed. At the same time, it is strongly desireable that the
child is able to figure out where to send the NOTIFY via a single
query.

By adding the following to the zone `parent.`:

    parent.   IN NOTIFY CDS   scheme port scanner.parent.
    parent.   IN NOTIFY CSYNC scheme port scanner.parent.

where the only scheme defined here is scheme=1 with the interpretation
that when a new CDS (or CDNSKEY or CSYNC) is published, a NOTIFY(CDS)
or NOTIFY(CSYNC) should be sent to the address and port listed in the
corresponding NOTIFY RRset.

Example:

    parent.   IN NOTIFY CDS   1 5359 cds-scanner.parent.
    parent.   IN NOTIFY CSYNC 1 5360 csync-scanner.parent.

Other schemes are possible, but are out of scope for this document.

The suggested mnemonic for the new record type is "NOTIFY" and it is
further described below.

## How to Interpret CDS and CSYNC Notifications

Upon receipt of a NOTIFY(CDS) for a particular child zone at the
published address for CDS notifications, the parent has two options:

  1. Schedule an immediate check of the CDS and CDNSKEY RRsets as
     published by that particular child zone.

     If the check finds that the CDS/CDNSKEY RRset has indeed changed,
     the parent MAY reset the scanning timer for children for which
     NOTIFY(CDS) is received, or reduce the periodic scanning frequency
     accordingly (e.g. to every two weeks).
     This will decrease the scanning effort for the parent.
     If a CDS/CDNSKEY change is then detected (without having received
     a notification), the parent SHOULD clear that state and revert to
     the default scanning schedule.

     Parents introducing CDS/CDNSKEY scanning support at the same time
     as NOTIFY(CDS) support are not in danger of breaking children's
     scanning assumption, and MAY therefore use a low-frequency
     scanning schedule in default mode.

  2. Ignore the notification, in which case the system works exactly
     as before.

If the parent implements the first option, the convergence time (time
between publication of a new CDS and propagation of the resulting DS)
will decrease significantly, thereby providing improved service to the
child zone.

If the parent, in addition to scheduling an immediate check for the
child zone of the notification, also choses to modify the scanning
schedule (to be less frequent), the cost of providing the scanning
service will be reduced.

Upon receipt of a NOTIFY(CSYNC) to the published address for CSYNC
notifications, the parent has exactly the same options to choose among
as for the NOTIFY(CDS).

# Who Should Send the Notifications?

Because of the security model where a notification by itself never
causes a change (it can only speed up the time until the next
check for the same thing), the sender's identity is not crucial.

This opens up the possibility of having an arbitrary party (e.g., a
side-car service) send the notifications to the parent,
thereby enabling this new functionality even before the emergence of
support for generalized DNS notifications in the name servers used for
signing.

# Timing Considerations

When a primary name server publishes a new CDS RRset there will be a
time delay until all copies of the zone world-wide will have been
updated. Usually this delay is short (on the order of seconds), but it
is larger than zero. If the primary sends a NOTIFY(CDS) at the exact
time of publication of the new zone there is a potential for the
parent to schedule the CDS check that the notification triggers faster
than the zone propagates to all secondaries. In this case the parent
may draw the wrong conclusion (“the CDS RRset has not been updated”).

Having a delay between the publication of the new data and the check
for the new data would alleviate this issue. However, as the parent
has no way of knowing how quickly the child zone propagates, the
appropriate amount of delay is uncertain.
It is therefore RECOMMENDED that the child delays sending NOTIFY
packets to the parent until a consistent public view of the pertinent
records is ensured.

NOTIFY(CSYNC) has the same timing consideration as NOTIFY(CDS).

# The Format of the NOTIFY Record

Here is the format of the NOTIFY RR, whose DNS type code is not yet
defined.

        Name TTL Class NOTIFY RRtype Scheme Port Target

Name
        Name of the zone to which the notification's hint pertains (i.e.,
        the zone in which the change will be made, if accepted).
        For NOTIFY(CDS) and NOTIFY(CSYNC), this is the name of the parent zone.

TTL
        Standard DNS meaning {{!RFC1035}}.

Class
        Standard DNS meaning {{!RFC1035}}. NOTIFY records occur in the IN
        Class.

RRtype
        The type of generalized NOTIFY that this NOTIFY RR defines
        the desired target address for. Currently only the types CDS and
        CSYNC are suggested to be supported, but there is
        no protocol issue should a use case for additional types of
        notifications arise in the future.

Scheme
        The scheme for locating the desired notification address.
        The range is 0-255. This is a 16 bit unsigned integer in
        network byte order. The value 0 is an error. The value 1 is
        described in this document and all other values are currently
        unspecified.

Port
        The port on the target host of the notification service. The
        range is 0-65535. This is a 16 bit unsigned integer in network
        byte order.

Target
        The domain name of the target host providing the service of
        listening for generalized notifications of the specified type.
        This name MUST resolve to one or more address records.

# Open Questions

## Rationale for a new record type, "NOTIFY"

It is technically possible to store the same information in an SRV
record as in the proposed NOTIFY record. This would, however, require
name space pollution (indicating the RRtype via a label in the owner
name) and also changing the semantics of one of the integer fields of
the SRV record.

Overloading semantics on a single record type has not been a good idea
in the past. Furthermore, as the generalized notifications are a new
proposal with no prior deployment on the public Internet there is an
opportunity to avoid repeating previous mistakes.

In the case of the "vertical" NOTIFY(CDS) and NOTIFY(CSYNC), no
special processing needs to be applied by the authoritative nameserver
upon insertion of the record indicating the notification target.
The nameserver can be "unaware"; a conventional SRV record would
therefore suffice from a processing point of view.

However, future use cases (such as for multi-signer key exchange) may
require the nameserver to trigger special operations, for example when
a NOTIFY record is inserted during onboarding of a new signer.

A new record type would therefore make it possible to more easily
associate the special processing with the record's insertion. The
NOTIFY record type also provides a cleaner solution for bundling all
the new types of notification signaling in an RRset, like:

    parent.         IN NOTIFY  CDS     1  59   scanner.parent.
    parent.         IN NOTIFY  CSYNC   1  59   scanner.parent.

## Rationale for Keeping DNS Message Format and Transport

In the most common cases of using generalized notifications the
recipient is expected to not be a nameserver, but rather some other
type of service, like a CDS/CSYNC scanner.

However, this will likely not always be true. In particular it seems
likely that in cases where the parent is not a large
delegation-centric zone like a TLD, but rather a smaller zone with a
small number of delegations there will not be separate services for
everything and the recipient of the NOTIFY(CDS) or NOTIFY(CSYNC) will
be an authoritative nameserver for the parent zone.

For this reason it seems most reasonable to stay within the the well
documented and already supported message format specified in RFC 1996
and delivered over normal DNS transport, although not necessarily to
port 53.

# Out of Scope

To accommodate ICANN's RRR model, the parent's designated notification
target may forward NOTIFY(CDS) messages to the registrar, e.g. via EPP
or by forwarding the NOTIFY(CDS) message directly. The same is true
also for NOTIFY(CSYNC).

From the perspective of this protocol, the NOTIFY(CDS) packet is simply
sent to the parent's listener address. However, should this turn out
not to be sufficient, it is possible to define a new "scheme" that
specifies alternative logic for dealing with such requirements.
Description of internal processing in the recipient end or for
locating the recipient are out of scope of this document.

While use of the NOTIFY mechanism for coordinating the key exchange in
multi-signer setups {{!I-D.wisser-dnssec-automation}} is conceivable,
the detailed specification is left for future work.

# Security Considerations

The original NOTIFY specification sidesteps most security issues by not
relying on the information in the NOTIFY message in any way, and instead
only using it to "enter the state it would if the zone's refresh timer
had expired" ({{Section 4.7 of !RFC1996}}).

It therefore seems impossible to affect the behaviour of the recipient of
the NOTIFY other than by hastening the timing for when different checks
are initiated. This security model is reused for generalized NOTIFY messages.

The receipt of a notification message will, in general, cause the
receiving party to cause an outbound query for the records of interest
(for example, NOTIFY(CDS) will cause CDS/CDNSKEY queries). When performed
via port 53, the size of these queries is comparable to that of the
notification messages themselves, rendering any amplfication attempts
futile.

However, when the outgoing query occurs via encrypted transport, some
amplification is possible, both with respect to bandwidth and computational
resources. Receivers therefore MUST implement rate limiting for notification
processing. It is RECOMMENDED to configure rate limiting independently for
both the notification's source IP address and the name of the zone that is
conveyed in the notification message (e.g., the child zone).


# IANA Considerations

Per {{!RFC8552}}, IANA is requested to create a new registry on the
"Domain Name System (DNS) Parameters" IANA web page as follows:

Name: Generalized DNS Notifications

Assignment Policy: Expert Review

Reference: (this document)

| NOTIFY type | Scheme | Location | Reference       |
| ----------- | ------ | -------- | --------------- |
| CDS         |      1 | parent.  | (this document) |
| CSYNC       |      1 | parent.  | (this document) |

# Acknowledgements

Joe Abley, Mark Andrews, Christian Elmerot, Ólafur Guðmundsson, Paul
Wouters, Brian Dickson

--- back


# Efficiency and Convergence Issues in DNS Scanning {#context}

{{!RFC1996}} introduced the concept of a DNS Notify message which was used
to improve the convergence time for secondary servers when a DNS zone
had been updated in the primary. The basic idea was to augment the
traditional "pull" mechanism (a periodic SOA query) with a "push"
mechanism (a Notify) for a common case that was otherwise very
inefficient (due to either slow convergence or wasteful overly
frequent scanning of the primary for changes).

While it is possible to indicate how frequently checks should occur
(via the SOA Refresh parameter), these checks did not allow catching
zone changes that fall between checkpoints. {{!RFC1996}} addressed the
optimization of the time-and-cost trade-off between a seceondary checking
frequently for new versions of a zone, and infrequent checking, by
replacing scheduled scanning with the more efficient NOTIFY mechanism.

Today, we have similar issues with slow updates of DNS data in spite of
the data having been published. The two most obvious cases are CDS and
CSYNC scanners deployed in a growing number of TLD registries. Because of
the large number of child delegations, scanning for CDS and CSYNC records
is rather slow (as in infrequent).

It is only a very small number of the delegations that will have updated
CDS or CDNSKEY record in between two scanning runs. However, frequent
scanning for CDS and CDNSKEY records is costly, and infrequent scanning
causes slower convergence (i.e., delay until the DS RRset is updated).

Unlike in the original case, where the primary is able to suggest the
scanning interval via the SOA Refresh parameter, an equivalent mechanism
does not exist for DS-related scanning.

All of this above also applies to parents that offer automated NS and glue
record maintenance via CSYNC scanning {{!RFC7477}}. Again, given that CSYNC
records change only rarely, frequent scanning of a large number of
delegations seems disproportionately costly, while infrequent scanning
causes slower convergence (delay until the delegation is updated).


# Change History (to be removed before publication)

* draft-ietf-dnsop-generalized-notify-01

> More discussion on amplification risks

> Clean-up, remove multi-signer use-case

* draft-ietf-dnsop-generalized-notify-00

> Revision after adoption.

* draft-thomassen-dnsop-generalized-dns-notify-02

> Add rationale for staying in band

> Add John as an author

* draft-thomassen-dnsop-generalized-dns-notify-01

> Mention Ry-to-Rr forwarding to accommodate RRR model

> Add port number flexiblity

> Add scheme parameter

> Drop SRV-based alternative in favour of new NOTIFY RR

> Editorial improvements

* draft-thomassen-dnsop-generalized-dns-notify-00

> Initial public draft.

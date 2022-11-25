---
title: Generalized DNS Notifications
abbrev: Generalized Notifications
docname: draft-thomassen-dnsop-generalized-dns-notifications-latest
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

normative:

informative:

--- abstract

This document specifies generalized DNS notifications.

TO BE REMOVED: This document is being collaborated on in Github at:
[https://github.com/peterthomassen/draft-thomassen-dnsop-generalized-dns-notifications](https://github.com/peterthomassen/draft-thomassen-dnsop-generalized-dns-notifications).
The most recent working version of the document, open issues, etc. should all be
available there.  The authors (gratefully) accept pull requests.

--- middle

# Introduction

This document introduces a generalization of the DNS NOTIFY mechanism
described in RFC 1996.

Traditional DNS notifications, which are here referred to as
”NOTIFY(SOA)”, are sent from a primary server to a secondary server to
minimize the latter’s convergence time to a new version of the
zone. This mechanism successfully addresses a significant inefficiency
in the original protocol.

Today there are several new cases where similar inefficiencies cause
significant problems. However, just as in the NOTIFY(SOA) case, a new
set of notification types will have a major positive benefit by
allowing the DNS infrastructure to completely sidestep these
inefficiencies.

There are no DNS protocol changes introduced by this document.

There are, however, proposals for how to interpret a wider range of
DNS messages that are already allowed (but not used) by the protocol.


Readers are expected to be familiar with DNSSEC, including {{!RFC4033}},
{{!RFC4034}}, {{!RFC4035}}, {{!RFC6781}}, {{!RFC7344}}, {{!RFC7477}},
{{!RFC7583}}, and {{!RFC8901}}.

## Requirements Notation

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**", "**SHALL
NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**", "**NOT
RECOMMENDED**", "**MAY**", and "**OPTIONAL**" in this document are to be
interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}} when, and only
when, they appear in all capitals, as shown here.


# Costs and Dangers of Slow Convergence

RFC 1996 introduced the concept of a DNS Notify message which was used
to improve the convergence time for secondary servers when a DNS zone
had been updated in the primary. The basic idea was to augment the
traditional “pull” mechanism (a periodic SOA query) with a “push”
mechanism (a Notify) for a common case that was otherwise very
inefficient (due to either slow convergence or wasteful overly
frequent scanning of the primary for changes).

Today we have similar issues with slow updates of DNS data in spite of
the data having been published. The two most obvious cases are the
scalability issues of modern CDS and CSYNC scanners that are beginning
to be deployed in a growing number of TLD registries. Because of the
large number of child delegations, scanning for CDS and CSYNC records
is rather slow (as in infrequent).

Another use case is for so-called “multi-signer” setups where there
are multiple, semi-independent, “signers” that each sign the same
zone, but with individual sets of keys. One requirement for
multi-signer setups to work is the ability to insert additional
DNSKEY records in each signer’s version of the zone. This is not the
only multi-signer requirement, but it is the one that matters
here. The problem here is that modern signers will commonly roll
DNSSEC Zone Signing Keys automatically and without informing anyone,
assuming a single-signer setup where ZSK rollovers are a local matter
(as opposed to KSK rollovers that involve the parent).

However, when the ZSKs of one signer are rolled the other signers will
be unaware of this event. Having the individual signers sign DNSKEY
RRsets where the ZSKs are not the same create a possibly broken
signature chain and is therefore a dangerous situation. Eventually the
signers will catch up, as multi-signer operators import each other's
DNSKEY public key(s) to be able to compute the combined RRsets that
should be published, but the period between the ZSK rollover in one
signer and the other signers becoming aware constitutes a period of
potential brokenness.

# Efficiency Issues with DNS Scanning vs. Notification

The original problem that RFC 1996 addressed was the problem of
optimization between frequent checking for new versions of a zone by a
secondary and infrequent checking. Both are inefficient. NOTIFY
transforms this into an efficient mechanism.

In modern DNS infrastructure there are several new scanning based
inefficiencies that did not exist at the time of the writing of RFC
1996.

Not every zone runs a CDS or CSYNC scanner (today). However, for those
that do, the problem is actually worse than in the traditional primary
to secondary case. The reason is that in the original case the primary
is able to indicate frequent changes are to be expected (via the SOA
Refresh parameter). No equivalent mechanism exist for the new scanning
services.

## CDS and CDNSKEY Scanners

An increasing number of TLD registries are implementing periodic
scanning for CDS and CDNSKEY records in child zones. Because of the
large number of delegations this is rather costly, especially as it is
only a very small number of the signed delegations that will have
updated the CDS or CDNSKEY record between two scanning runs.

Frequent scanning is costly. Infrequent scanning causes slower
convergence (i.e. delay until the DS in the parent is updated).

## CSYNC Scanners

This is basically the same problem as for the CDS / CDNSKEY scanners:
large number of delegations vs infrequent updates to the CSYNC record
in the child zones.

Frequent scanning is costly. Infrequent scanning causes slower convergence
(i.e. updated delegation information in the parent).

## Multi-Signer Setups

{{!I-D.wisser-dnssec-automation}} describes processes for managing signed
zones using multiple semi-independent “signers” (i.e. services that
take an unsigned zone, sign it using unique DNSKEYs and publish the
zone on the public Internet). The setup most commonly discussed for
multi-signer uses a “multi-signer controller”, which is a separate
service responsible for keeping the signing and delegation information
in sync across multiple signers.

To keep information in sync, the signers need to be scanned for
current information about the DNSKEY RRset in each signer. The problem
is that modern “signers” will typically do DNSKEY rollovers
(especially ZSK rollovers) automatically without informing anyone
else, because a single-signer setup is often assume (in which case the
ZSK rollover is a matter completely internal to the signer).

In the multi-signer case, this is not a correct assumption. It is
therefore necessary to run frequent polls frequently to minimize
the time window between one signer changing its version of the DNSKEY
RRset and the controller noticing and re-synchronizing all signers.

Frequent scanning: costly. Infrequent scanning: slower convergence
(i.e. longer potential inconsistency across signers).

# CDS/CDNSKEY and CSYNC Notifications

The RFC 1996 NOTIFY message sends a SOA record in the Query
Section. We refer to this as a NOTIFY(SOA).
By generalizing the concept of DNS NOTIFY it is possible to address
not only the original inefficiency (primary name server to secondary
nameserver convergence) but also the problems with CDS/CDNSKEY, CSYNC
and Multi-Signer scanning for zone updates.

The CDS/CDNSKEY inefficiency may be addressed by the child sending a
NOTIFY(CDS) to an address where the parent listens for such notifications.
To address the CDS/CDNSKEY dichotomy, NOTIFY(CDS) is defined to indicate
any child-side changes pertaining to a upcoming update of DS records.
Upon receipt of NOTIFY(CDS), the parent SHOULD initiate the same scan
that would other be triggered based on a timer.

The CSYNC inefficiency may similarly be addressed by the child sinding a
NOTIFY(CSYNC) to an address where the parent is listening to CSYNC
notifications.

In both cases the notification will speed up the CDS and CSYNC
scanning by providing the parent with a hint that a particular child
zone has published new CDS, CDNSKEY and/or CSYNC records.

The DNS protocol already allows these new notification types, so no
protocol change is required, the only thing needed is specification of
where to send the new types of Notifies and how to interpret them in
the receiving end.

## Where to send CDS and CSYNC Notifications

In the case of NOTIFY(CDS) and NOTIFY(CSYNC) the ultimate recipient of
the notification is the parent, to improve the speed and efficiency of
the parent's CDS/CSYNC scanning. Because the name of the parent zone
is known, and because the parent obviously controls the contents of
its own zone the simplest solution is for the parent to publish the
address where it prefers to have notifications sent. To be slightly
more general than just using an address record we propose to use SRV
records, as they are well suited to this.

In the zone `parent.`:

    _cds-notifications.parent.   IN SRV n m  53 scanner.parent.
    _csync-notifications.parent. IN SRV n m  53 scanner.parent.

## How to Interpret CDS and CSYNC Notifications

Upon receipt of a NOTIFY(CDS) for a particular child zone at the
published address for CDS notifications, the parent, roughly
speaking, has three options:

  1. Schedule an immediate check of the CDS and CDNSKEY RRsets as
     published by that particular child zone.

     The parent MAY keep state about children which are known to send
     NOTIFY(CDS) messages, and reduce the periodic scanning frequency
     accordingly (e.g. to every two weeks).
     If a CDS/CDNSKEY change is then detected, the parent SHOULD clear
     that state and revert to the default scanning schedule.
     Parents introducing CDS/CDNSKEY scanning support at the same time
     as NOTIFY(CDS) support are not in danger of breaking children's
     scanning assumption, and MAY therefore use a low-frequency
     scanning schedule in default mode.

  2. Schedule an immediate check of the CDS and CDNSKEY RRsets as
     published by that particular child zone.

     Choose to reduce the scheduled scanning frequency for that
     particular child zone, at least not for some time, as that child is
     apparently able to send CDS notifications when it is rolling its
     KSK.  This will decrease the scanning effort for the parent.

  3. Ignore the notification, in which case the system works exactly
     as before.

If the parent implements one of the first two options the convergence
time (time between publication of a new CDS and propagation of the
resulting DS) will decrease significantly, thereby providing improved
service to the child zone.

If the parent, in addition to scheduling an immediate check for the
child zone of the notification, also choses to modify the scanning
schedule (to be less frequent) the cost of providing the scanning
service will be reduced.

Upon receipt of a NOTIFY(CSYNC) to the published address for CSYNC
notifications, the parent has exactly the same options to choose among
as for the NOTIFY(CDS).

# DNSKEY Notifications

In the multi-signer case the problem is slightly different, because it
is not the parent that should be notified, but rather a third party.

## Where to send DNSKEY Notifications

The question is how the multi-signer controller should inform the
signer about where to send the notification. In the NOTIFY(CDS) and
NOTIFY(CSYNC) cases the child (the service that signs the child zone)
knows the name of the parent zone and can look up the notification
address there. In the NOTIFY(DNSKEY) case this is not
possible. However, what is possible is to look up the notification
address in the child zone itself. This works out as a requirement for
multi-signer setups to work is that at least one external party (the
multi-signer controller) has the ability to insert or modify RRsets in
the child zone. Therefore the controller should be able to insert a
record that documents the wanted address for DNSKEY notifications.

In analogy with the other cases, but here in the zone `child.parent.`:

    _dnskey-notification.child.parent. IN SRV n m 53 scanner.signerA.
    _dnskey-notification.child.parent. IN SRV n m 53 scanner.signerB.

should act as a trigger for the signer that whenever there are changes
to the DNSKEY RRset it should send a corresponding NOTIFY(DNSKEY) to
all signers that have signaled interest via insertion of SRV records
in the zone.

## How to Interpret DNSKEY Notifications

The receipt of a NOTIFY(DNSKEY) is a hint that the DNSKEY RRset at the
sending signer has been updated. If the child zone is not part of a
multi-signer setup this should be a no-op that does not cause any
action. In a multi-signer setup, though, this hint provides the
recipient of the notification with the same three options as in the
NOTIFY(CDS) and NOTIFY(CSYNC) cases: schedule an immediate check,
delay regular checks and ignore.

# Who Should Send the Notifications?

Because of the security model where a notification by itself never
causes a change (it can only speed up the time until the next
scheduled check for the same thing) who the sender is is not crucial.

This opens up the possibility of having the controller in a
multi-signer setup send the CDS and CSYNC notifications to the parent,
thereby enabling this new functionality even before the emergence of
support for generalized DNS notifications in the name servers used for
signing.

However, as multi-signer is strongly dependent on DNSKEY notifications
(which can only be generated by the signing software) this does not
alleviate the need for modifications also to signing software to
support NOTIFY(DNSKEY).

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
has no way of knowing how quickly the child zone propagates, any such
intentional delay should be implemented in the sending end, i.e. at
the server generating the NOTIFY(CDS), not in the receiving end.

NOTIFY(CSYNC) has the same timing consideration as NOTIFY(CDS) while
NOTIFY(DNSKEY) does not.

# Open Questions

## SRV vs new record type

In the case of the “vertical” Notify(CDS) and NOTIFY(CSYNC) nothing
needs to be done by the nameservers serving the parent zone. So
therefore the SRV records appear to be a near perfect match. In the
case of the more “horizontal” NOTIFY(DNSKEY), however, the nameserver
will have to act on the insertion of a NOTIFY(DNSKEY) instruction.

It may therefore be preferable to use a new record type, to make it
possible to more easily associate the special processing needed with
the new type rather than the standard SRV RRtype that occurs in
completely other scenarios than are described here. If that is the
case it could be that a new record type would provide a cleaner
solution to all the new types of notification signaling. Eg.:

    parent.         IN NOTIFY  CDS     53   scanner.parent.
    parent.         IN NOTIFY  CSYNC   53   scanner.parent.
    child.parent.   IN NOTIFY  DNSKEY  5300 music.service.provider.

## Open Question For DNSKEY Notifications

In a multi-signer setup there are multiple signers. How will the
multi-signer controller know which signer sent the notification? As
the number of signers for a particular zone is low (typically 2 or
possibly 3), there is no major problem caused by not knowing which
signer sent the notification and instead always check all the signers
for updates to the DNSKEY RRset.

# Out of Scope

In the multi-signer case, where the same zone is signed by multiple
semi-independent “signers” it is obviously necessary to devise means
of keeping the non-DNSSEC contents of the zone in sync. That is out of
scope of the multi-signer work and also out of scope of this document.

A corner case of this scoping regards making changes to the NS
RRset. Individual signers (as in DNS service providers) may want to be
able to make such changes independently of the other contents of the
zone. See previous section.

# Security Considerations

The original NOTIFY specification (RFC 1996) sidesteps most security
issues by not relying on the information in the NOTIFY message in any
way but instead only use it to set the timer until the next scheduled
check (for the SOA record) to zero. Therefore it is impossible to
affect the behaviour of the recipient of the NOTIFY other than by
changing the timing for when different checks are initiated.

This mechanism and security model is reused for all the generalized
NOTIFY messages.

Another consideration is whether generalized DNS
Notifications can be used as an amplification attack. The answer seems
to be “NO”, because the size of the generalized NOTIFY messages is
mostly equal to the size of the response and it is also mostly equal
to the size of the resulting outbound query (for the child zone’s CDS,
CSYNC or DNSKEY RRset).

Hence the amplification attack potential of generalized Notifications
is the same as for the original NOTIFY(SOA), which has never been
found to be useful for amplification attacks.

In any case, NOTIFY consumers MAY configure rate limits to address
concerns about the impact of unsolicited NOTIFY messages.

--- back

# Change History (to be removed before publication)

* draft-thomassen-dnsop-generalized-dns-notifications-00

> Initial public draft.

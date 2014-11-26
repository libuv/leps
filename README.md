
# Libuv Enhancement Proposals

## Overview

This repository contains the Libuv Enhancement Proposal (LEP) collection.
A LEP is a document describing an enhancement proposal for inclusion in libuv.

LEPs will be used when the proposed feature is too broad or intrusive, or it
would modify APIs heavily. Minor changes do not require writing a LEP. What
is and isn't minor is subjective, so as a general rule, users should discuss
the proposal briefly by other means (issue tracker, mailing list or IRC) and
write a LEP when requested by a libuv core team member.

## Rationale

The idea behind the LEP process is to keep track of what ideas will be worked on
and which ones where discarded and why. This should help everyone (those closely involved
with the project and newcomers) have a clear picture of where libuv stands and
where it want to be in the future.

## Format

LEP documents don't follow a given format (other than being writen in MarkDown).
It is, however, required, that all LEPs include the following information at
the top of the file:

* Title
* Name of author
* Status (more on statuses later, new documents must be submitted with the
'draft' status
* Date

The document file name must conform to the following format: "XXX-title-ish.md"
and it must be added to the document index in 000=index.md.

## Content

LEP documents should be as detailed as possible. Any type of media which helps
clarify what it tries to describe is more than welcome, be that an ASCII diagram,
psudocode or actual C code.

## Licensing

All LEP documents must be MIT licensed.

## Progress of a LEP

All LEPs will be comitted to the repository regardless of their acceptance. The
initial status shall be "draft".

If the document is uncontroversial and agreenment is reached quickly it might be
comitted directly with the "accepted" status. Likewise, if the proposal is rejected the
status shall be "rejected". When a document is rejected a member of the core team
should append a section describing the reasons for rejection.

A document shall also be comitted in "draft" status. This means consensus has not
been reached yet.

The author of a LEP is expected to actually pursue the proposal with code.

